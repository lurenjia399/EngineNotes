# FScheduler

```cpp
class FScheduler final : public FSchedulerTls
{
	// 等待队列数组，一个是前台线程的等待队列，一个是后台线程的等待队列
	Private::FWaitingQueue WaitingQueue[2] = { 
		{ WorkerEvents, OversubscriptionLimitReachedEvent }, 
		{ WorkerEvents, OversubscriptionLimitReachedEvent } };
	// 全局的工作管理类LocalQueueRegistry，用来管理不同线程的工作队列LocalQueue
	FSchedulerTls::FQueueRegistry QueueRegistry;
	// 创建工作线程上的锁。在唤醒线程的时候，线程如果不够了就会创建线程，创建的时候不一定在游戏线程了，在加上创建新的线程要修改FScheduler上的变量，所以就加锁。
	UE::FPlatformRecursiveMutex WorkerCreationCS;
	UE::FPlatformRecursiveMutex WorkerThreadsCS;
	// 线程数组，所有创建出来的线程
	TArray<std::atomic<FThread*>> WorkerThreads;
	// 线程的工作队列数组，每个线程都有一个对应的LocalQueue
	TAlignedArray<FSchedulerTls::FLocalQueueType> WorkerLocalQueues;
	// 线程的等待Event数组，每个线程都一个FWaitEvent，用来唤醒，阻塞
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

# FScheduler::StartWorkers
```cpp
1 游戏线程执行StartWorkers这个方法，里面就是做一些初始化的操作，比如工作队列，等待Event队列等等之类的
2 如果开启了动态创建线程（pc是默认开的）的话，StartWorkers方法就不会创建线程，只做初始化的操作，有任务发到FScheduler结构中后，会判断线程是否够，不够就创建线程。
3 如果没开启动态创建线程的话，就会在StartWorkers方法里创建，创建两个前台线程和一堆后台线程。
```

# FScheduler::CreateWorker
```cpp
1 创建工作线程的方法，
```