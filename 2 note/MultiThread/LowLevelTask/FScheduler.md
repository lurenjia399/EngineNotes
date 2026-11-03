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