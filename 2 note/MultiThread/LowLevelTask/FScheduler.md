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
	// 游戏线程的工作队列，在StartWorkers中赋值的
	TUniquePtr<FSchedulerTls::FLocalQueueType> GameThreadLocalQueue;
	// 当前Scheduler中有效的工作线程数量
	std::atomic_uint ActiveWorkers { 0 };
	std::atomic_uint NextWorkerId { 0 };
	std::atomic<int32> ForegroundCreationIndex{ 0 };
	std::atomic<int32> BackgroundCreationIndex{ 0 };
	// 线程亲和性
	uint64 WorkerAffinity = 0;
	uint64 BackgroundAffinity = 0;
	// 前台Worker的线程优先级，前台Worker和后台Worker都是工作线程，只有两个线程是前台Worker。区分开的目的就是防止所有的Worker都被后台任务占用。
	EThreadPriority WorkerPriority = EThreadPriority::TPri_Normal;
	// 后台Worker的线程优先级
	EThreadPriority BackgroundPriority = EThreadPriority::TPri_BelowNormal;
	// RestartWorkers的时候会使用，就是在重置过程中的标志位
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
1 创建工作线程的方法，会返回TUniquePtr<FThread>
2 线程的主循环就是WorkerMain方法
```

# FScheduler::WorkerMain
```cpp
void FScheduler::WorkerMain(
	Private::FWaitEvent* WorkerEvent, // FScheduler上等待Event队列中的一个，外部传进来
	FSchedulerTls::FLocalQueueType* WorkerLocalQueue, // FScheduler上工作队列中的一个
	uint32 WaitCycles, // 线程自旋等待的次数
	bool bPermitBackgroundWork) // 前台线程还是后台线程
{
	/*
	1 拿到当前线程的TLS
	2 struct FTlsValues : public TIntrusiveLinkedList<FTlsValues>
		{
			// 当前线程为哪个Scheduler工作
			FSchedulerTls* ActiveScheduler = nullptr;
			// 缓存线程的工作队列指针
			FLocalQueueType* LocalQueue = nullptr;
			// 线程的类型，前台线程还是后台线程
			EWorkerType WorkerType = EWorkerType::None;
			// 是否正在唤醒线程
			std::atomic<bool> bPendingWakeUp = false;
			// 是否是备用线程
			bool        bIsStandbyWorker = false;
		}
	*/
	FTlsValues& LocalTlsValues = GetTlsValuesRef();
	
	// 赋值TLS中的Scheduler
	LocalTlsValues.ActiveScheduler = this;
	// 申请TLS缓存。TLS缓存可以理解为在堆上取了一块内存区域，如果在线程中new对象，如果满足要求就会放到这块区域中。如果没有TLS缓存，new对象就直接在堆上申请了，那样不同线程间就会竞争，就需要加锁来限制，有了TLS缓存就不用加锁也能减少竞争。
	FMemory::SetupTLSCachesOnCurrentThread();
	// 赋值线程TLS中的线程类型，是前台线程还是后台线程
	LocalTlsValues.WorkerType = bPermitBackgroundWork ? 
		FSchedulerTls::EWorkerType::Background 
		: FSchedulerTls::EWorkerType::Foreground;
	// 赋值线程TLS，线程是否是备用线程
	LocalTlsValues.SetStandbyWorker(WorkerEvent->bIsStandby);
	// 赋值线程TLS，线程的工作队列
	LocalTlsValues.LocalQueue = WorkerLocalQueue;
	// 如果是备用线程，就走StandbyLoop循环，如果是普通工作线程就走WorkerLoop
	if (WorkerEvent->bIsStandby)
	{
		StandbyLoop(WorkerEvent, WorkerLocalQueue, 
			WaitCycles, bPermitBackgroundWork);
	}
	else
	{
		WorkerLoop(WorkerEvent, WorkerLocalQueue, 
			WaitCycles, bPermitBackgroundWork);
	}
	// 在线程销毁后，WorkerLoop才会退出循环，执行到后边的方法，这里就是清除数据的操作
	LocalTlsValues.LocalQueue = nullptr;
	LocalTlsValues.ActiveScheduler = nullptr;
	LocalTlsValues.SetStandbyWorker(false);
	LocalTlsValues.WorkerType = FSchedulerTls::EWorkerType::None;
	FMemory::ClearAndDisableTLSCachesOnCurrentThread();
}
```
1 WorkerMain就是线程执行的主方法，里面会填充TLS的数据，并申请TLS缓存。
2 主方法中会执行WorkerLoop的死循环，如果有工作就唤醒，没工作就阻塞，只有StopWorker才会退出死循环，结束这个线程

# FScheduler::WorkerLoop
```cpp
void FScheduler::WorkerLoop(
	Private::FWaitEvent* WorkerEvent, 
	FSchedulerTls::FLocalQueueType* WorkerLocalQueue, 
	uint32 WaitCycles, 
	bool bPermitBackgroundWork)
{
	Private::FOutOfWork OutOfWork;
	while (true)
	{
		bool bExecutedSomething = false;
		/*
		1 从队列中找任务，然后执行
		2 首先从GameThreadLocalQueue队列中StealLocal偷取任务，有任务就执行
		3 GameThreadLocalQueue里没任务就找自己的工作队列获取任务，有任务就执行
		4 如果自己的工作队列也没有任务，就从LocalQueueRegistry中偷其他线程工作队列中的任务，有任务就执行
		5 如果有任务，就更新bPreparingWait，bExecutedSomething标志位。
		6 等任务执行完，就进入下一次找任务的循环中
		*/
		while (TryExecuteTaskFrom<FSchedulerTls::FLocalQueueType, 
				&FSchedulerTls::FLocalQueueType::StealLocal, false>(WorkerEvent, 
					GameThreadLocalQueue.Get(), OutOfWork, bPermitBackgroundWork)
			|| TryExecuteTaskFrom<FSchedulerTls::FLocalQueueType, 
				&FSchedulerTls::FLocalQueueType::Dequeue, false>(WorkerEvent, 
					WorkerLocalQueue, OutOfWork, bPermitBackgroundWork)
			|| TryExecuteTaskFrom<FSchedulerTls::FLocalQueueType, 
				&FSchedulerTls::FLocalQueueType::DequeueSteal, false>(WorkerEvent,
					 WorkerLocalQueue, OutOfWork, bPermitBackgroundWork))
		{
			bPreparingWait = false;
			bExecutedSomething = true;
		}
		
		/*
		1 这里已经退出找任务的循环了，所以是找不到任务
		2 如果此时ActiveWorkers原子变量为0，说明这个FScheduler执行了StopWorkers，正在关闭
		3 所以需要撤销准备等待的标记，然后执行break，退出WorkerLoop的循环，结束线程循环
		*/
		if (ActiveWorkers.load(std::memory_order_relaxed) == 0)
		{
			if (OutOfWork.Stop())
			{
				WaitingQueue[bPermitBackgroundWork].CancelWait(WorkerEvent);
			}
			break;
		}
		
		/*
		1 如果这一轮循环有执行过任务就返回，如果没有执行过任务才处理
		2 如果这一轮循环线程没有进入准备等待的状态里，就进入准备等待状态
		3 如果这一轮循环线程在准备等待状态里，就进入等待状态，等待状态会调用Park方法，里面自旋等待一会，然后真正阻塞线程
		4 当线程被唤醒后，重置bPreparingWait标志位
		*/
		if (bExecutedSomething == false)
		{
			if (!bPreparingWait)
			{
				OutOfWork.Start();
				WaitingQueue[bPermitBackgroundWork].PrepareWait(WorkerEvent);
				bPreparingWait = true;
			}
			else if (WaitingQueue[bPermitBackgroundWork].CommitWait(WorkerEvent, OutOfWork, WorkerSpinCycles, WaitCycles))
			{
				bPreparingWait = false;
			}
		}
	}
}
```
1 WorkerLoop方法就是线程工作的主体
2 首先进入找任务死循环中，从GameThreadLocalQueue队列中偷取任务，如果有就执行
3 如果GameThreadLocalQueue队列中没有任务，就获取自己工作队列的任务，如果有就执行
4 如果自己工作队列中也没有任务，就偷取其他线程工作队列中的任务，如果有就执行
5 如果没有任务执行才会退出找任务死循环，退出找任务死循环后，判断线程是否有效，无效的话直接退出线程循环了，销毁线程
6 没任务会进入准备等待状态，下一次循环依然没任务就会从准备等待进入等待状态，等待状态就会执行park方法了，里面会自旋等待一会，然后执行阻塞线程的方法

# FScheduler::TryLaunch
```cpp
bool FScheduler::TryLaunch(
	FTask& Task, // 尝试执行的任务
	EQueuePreference QueuePreference, // 哪个工作队列来接收这个任务
	bool bWakeUpWorker)// 是否可以唤醒这个工作线程
{
	// 改变任务的状态为Scheduler，原子操作并且是Release的
	if(Task.TryPrepareLaunch())
	{
		LaunchInternal(Task, QueuePreference, bWakeUpWorker);
		return true;
	}
	return false;
}
```

```cpp
void FScheduler::LaunchInternal(
	FTask& Task, 
	EQueuePreference QueuePreference, 
	bool bWakeUpWorker)
{
	// 如果有工作线程或者是在重置工作线程的过程中，都会为true
	if (ActiveWorkers.load(std::memory_order_relaxed) 
		|| TemporaryShutdown.load(std::memory_order_acquire))
	{
		/*
		1 默认是本地队列接收任务，但如果是非后台线程提交后台任务，或者是备用线程提交的任务，就要改成全局队列接收任务，就是把任务放到LocalQueueRegistry中的溢出队列里面
		*/
		if ((bIsBackgroundTask && !bIsBackgroundWorker) || bIsStandbyWorker)
		{
			QueuePreference = EQueuePreference::GlobalQueuePreference;
		}
		// 备用线程或者是没有本地工作队列的线程，还需要强制唤醒线程
		bWakeUpWorker |= bIsStandbyWorker || CachedLocalQueue == nullptr;
		// 如果是游戏线程，任务必须放到游戏线程的工作队列中，并且必须唤醒其他线程，来让其他线程来执行游戏线程工作队列中的任务
		if (CachedLocalQueue && CachedLocalQueue == GameThreadLocalQueue.Get())
		{
			QueuePreference = EQueuePreference::LocalQueuePreference;
			bWakeUpWorker = true;
		}
		// 把任务塞进队列中的方法，按照判断是塞进本地队列还是全局队列
		if (CachedLocalQueue 
			&& QueuePreference != EQueuePreference::GlobalQueuePreference)
		{
			CachedLocalQueue->Enqueue(&Task, uint32(Task.GetPriority()));
		}
		else
		{
			QueueRegistry.Enqueue(&Task, uint32(Task.GetPriority()));
		}
		if (bWakeUpWorker)
		{
			const bool bExternalThread = LocalTlsValues.ActiveScheduler != this 
				|| LocalTlsValues.WorkerType == EWorkerType::None;
			if (bExternalThread)
			{
				LocalTlsValues.bPendingWakeUp.store(true, MemoryOrder);
				if (TemporaryShutdown.load(std::memory_order_acquire))
				{
					LocalTlsValues.bPendingWakeUp.store(false, MemoryOrder);
					return;
				}
			}
			if (!WakeUpWorker(bIsBackgroundTask) && !bIsBackgroundTask)
			{
				WakeUpWorker(true);
			}
			if (bExternalThread)
			{
				LocalTlsValues.bPendingWakeUp.store(false, MemoryOrder);
			}
		}
	}
	// 没有工作线程了，都销毁了。直接执行
	else
	{
		FTask* TaskPtr = &Task;
		while (TaskPtr)
		{
			if ((TaskPtr = ExecuteTask(TaskPtr)) != nullptr)
			{
				verifySlow(TaskPtr->TryPrepareLaunch());
			}
		}
	}
}
```