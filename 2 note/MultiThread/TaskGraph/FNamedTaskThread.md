```cpp
class FNamedTaskThread : public FTaskThreadBase
{
	// 有两个MainQueue和LocalQueue工作队列
	FThreadTaskQueue Queues[ENamedThreads::NumQueues];
}

struct FThreadTaskQueue
{
	// 每个工作队列有一个个StallQueue
	FStallingTaskQueue<FBaseGraphTask, PLATFORM_CACHE_LINE_SIZE, 2> StallQueue;
	// 计数器，每次执行ProcessTasksNamedThread取任务方法，就会增加
	uint32 RecursionGuard;
	// 标志位，我们在退出执行的循环
	bool QuitForReturn;
	// 标志位，我们在退出执行的循环
	bool QuitForShutdown;
	// 系统事件，等待和唤醒功能
	FEvent*	StallRestartEvent;
}

class FStallingTaskQueue
{
	// 每个StallQueue有两个FLockFreePointerFIFOBase，一个是高优先级，一个是低优先级
	FLockFreePointerFIFOBase<T, TPaddingForCacheContention> 
		PriorityQueues[NumPriorities];
}
```
1 每个NamedTaskThread中有两个工作队列，一个MainQueue，一个LocalQueue。LocalQueue基本不用，在RenderThread中有用
2 每个工作队列有一个StallQueue，并且带有事件等待唤醒机制
3 每个StallQueue带有两个优先级队列，一个高优先级，一个低优先级

# ProcessTasksNamedThread
```cpp
uint64 ProcessTasksNamedThread(
	int32 QueueIndex, // 工作队列的索引
	bool bAllowStall) // 
{
	// 是否是渲染线程的MainQueue
	const bool bIsRenderThreadMainQueue = 
		(ENamedThreads::GetThreadIndex(ThreadId) == 
			ENamedThreads::ActualRenderingThread) 
			&& (QueueIndex == 0);
	while (!Queue(QueueIndex).QuitForReturn)
	{
		// 从StallQueue中取任务，但是取不到
		if (!Task)
		{
			// 如果没有任务但是允许暂停
			if (bAllowStall)
			{
				{
					// 执行StallQueue里的系统事件，开启等待
					Queue(QueueIndex).StallRestartEvent->Wait(
						bIsRenderThreadAndPolling ? 
							GRenderThreadPollPeriodMs : MAX_uint32, bCountAsStall);
					// 如果StallQueue里的QuitForShutdown标记为true，就return
					if (Queue(QueueIndex).QuitForShutdown)
					{
						return ProcessedTasks;
					}
				}
				continue;
			}
			// 如果没有任务还不允许暂停，就退出While循环，执行return了
			else
			{
				break;
			}
		}
		// 从StallQueue中取出任务，任务有效就执行任务
		else
		{
			// 任务执行
			Task->Execute(NewTasks, 
				ENamedThreads::Type(ThreadId | 
				(QueueIndex << ENamedThreads::QueueIndexShift)), true);
			// 每次执行了一个任务就增加的计数器
			ProcessedTasks++;
		}
	}
	// 返回任务执行计数器
	return ProcessedTasks;
}
```
1 有一个While死循环一直从工作队列中取任务
2 如果取到任务，就执行，然后下一次循环
3 如果没有取到任务，就判断参数是否可以暂停，如果可以就Wait，如果不能暂停就退出While系循环
# ProcessThreadUntilIdle

```cpp
virtual uint64 ProcessTasksUntilIdle(int32 QueueIndex) override
{
	// 重置标志位，让取任务的死循环能执行
	Queue(QueueIndex).QuitForReturn = false;
	// 增加计数，表示正在执行取任务
	verify(++Queue(QueueIndex).RecursionGuard == 1);
	// 调用取任务的方法，第一个参数是工作队列的索引，第二个参数是取不到任务时是否可以通过系统事件暂停线程
	uint64 ProcessedTasks = ProcessTasksNamedThread(QueueIndex, false);
	// 减少计数，表示结束取任务了
	verify(!--Queue(QueueIndex).RecursionGuard);
	return ProcessedTasks;
}
```

