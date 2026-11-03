# FScheduler

```cpp
class FScheduler final : public FSchedulerTls
{
	Private::FWaitingQueue WaitingQueue[2] = { 
		{ WorkerEvents, OversubscriptionLimitReachedEvent }, 
		{ WorkerEvents, OversubscriptionLimitReachedEvent } };
	FSchedulerTls::FQueueRegistry QueueRegistry;
	UE::FPlatformRecursiveMutex WorkerCreationCS;
	UE::FPlatformRecursiveMutex WorkerThreadsCS;
	TArray<std::atomic<FThread*>> WorkerThreads;
	TAlignedArray<FSchedulerTls::FLocalQueueType> WorkerLocalQueues;
	TAlignedArray<Private::FWaitEvent> WorkerEvents;
	TUniquePtr<FSchedulerTls::FLocalQueueType> GameThreadLocalQueue;
	std::atomic_uint ActiveWorkers { 0 };
	std::atomic_uint NextWorkerId { 0 };
	std::atomic<int32> ForegroundCreationIndex{ 0 };
	std::atomic<int32> BackgroundCreationIndex{ 0 };
	uint64 WorkerAffinity = 0;
	uint64 BackgroundAffinity = 0;
	// 前台Worker的线程优先级，前台Worker和后台Worker都是工作线程，只有两个线程是前台Worker。区分开的目的就是防止所有的Worker都被后台任务占用。
	EThreadPriority WorkerPriority = EThreadPriority::TPri_Normal;
	// 后台Worker的线程优先级
	EThreadPriority BackgroundPriority = EThreadPriority::TPri_BelowNormal;
	std::atomic_bool TemporaryShutdown{ false };
	FOversubscriptionLimitReached OversubscriptionLimitReachedEvent;
}
```

# TLocalQueueRegistry

```cpp
class TLocalQueueRegistry
{
private:
	FOverflowQueueType        OverflowQueues[uint32(ETaskPriority::Count)];
	// MaxLocalQueues定长的数组，每个元素都是原子变量，是LocalQueue的指针
	std::atomic<TLocalQueue*> LocalQueues[MaxLocalQueues] { nullptr };
	// 原子变量，用来表示LocalQueue的数量
	std::atomic<uint32>       NumLocalQueues {0};
	
	// 线程安全的往LocalQueues数组中添加元素的方法
	void AddLocalQueue(TLocalQueue* QueueToAdd)
	{
		// 原子操作的增加计数。relaxed顺序表示只关心原子操作本身，不承载其他目的
		uint32 Index = NumLocalQueues.fetch_add(1, std::memory_order_relaxed);
		// 往LocalQueues数组中添加元素。release顺序表示发布，如果其他线程通过acquire捕获了发布，则发布前的操作都能读取
		LocalQueues[Index].store(QueueToAdd, std::memory_order_release);
	}
	
	
}
```