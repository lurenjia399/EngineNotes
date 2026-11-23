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
	uint32 RecursionGuard;
	bool QuitForReturn;
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
1 每个NamedTaskThread中有两个工作队列，一个MainQueue，一个LocalQueue
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
	// 
	while (!Queue(QueueIndex).QuitForReturn)
	{
		const bool bIsRenderThreadAndPolling = bIsRenderThreadMainQueue && (GRenderThreadPollPeriodMs >= 0);
		const bool bStallQueueAllowStall = bAllowStall && !bIsRenderThreadAndPolling;
		FBaseGraphTask* Task = Queue(QueueIndex).StallQueue.Pop(0, bStallQueueAllowStall);
		TestRandomizedThreads();
		if (!Task)
		{
			if (bAllowStall)
			{
				TRACE_CPUPROFILER_EVENT_SCOPE(WaitForTasks);
				{
					FScopeCycleCounter Scope(StallStatId, EStatFlags::Verbose);
					Queue(QueueIndex).StallRestartEvent->Wait(bIsRenderThreadAndPolling ? GRenderThreadPollPeriodMs : MAX_uint32, bCountAsStall);
					if (Queue(QueueIndex).QuitForShutdown)
					{
						return ProcessedTasks;
					}
					TestRandomizedThreads();
				}
				continue;
			}
			else
			{
				break; // we were asked to quit
			}
		}
		else
		{
			Task->Execute(NewTasks, ENamedThreads::Type(ThreadId | (QueueIndex << ENamedThreads::QueueIndexShift)), true);
			ProcessedTasks++;
			TestRandomizedThreads();
		}
	}
	return ProcessedTasks;
}
```
# ProcessThreadUntilIdle

```cpp
virtual uint64 ProcessTasksUntilIdle(int32 QueueIndex) override
{
	Queue(QueueIndex).QuitForReturn = false;
	verify(++Queue(QueueIndex).RecursionGuard == 1);
	uint64 ProcessedTasks = ProcessTasksNamedThread(QueueIndex, false);
	verify(!--Queue(QueueIndex).RecursionGuard);
	return ProcessedTasks;
}
```

