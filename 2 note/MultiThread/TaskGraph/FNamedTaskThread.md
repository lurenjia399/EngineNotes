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