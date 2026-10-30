# FAsyncTaskBase

```cpp
// 这个FAsyncTaskBase是继承IQueuedWork的，封装了Work，包含一些同步执行，任务完成回调等接口
class FAsyncTaskBase
	: private UE::FInheritedContextBase
	, private IQueuedWork
{
	FThreadSafeCounter	WorkNotFinishedCounter;// 线程安全的计数器，每开始一个工作就增加
	FEvent*				DoneEvent = nullptr;// 工作完成触发的Event
	FQueuedThreadPool*	QueuedPool = nullptr;// 执行工作的线程池
	EQueuedWorkPriority Priority = EQueuedWorkPriority::Normal;// 工作优先级

	// 开始Work的方法，初始化一些参数
	void Start(
		bool bForceSynchronous, 
		FQueuedThreadPool* InQueuedPool, 
		EQueuedWorkPriority InQueuedWorkPriority, 
		EQueuedWorkFlags InQueuedWorkFlags, 
		int64 InRequiredMemory, 
		const TCHAR * InDebugName)
	{
		WorkNotFinishedCounter.Increment();
		QueuedPool = InQueuedPool;
		Priority = InQueuedWorkPriority;
		Flags = InQueuedWorkFlags;
		if (bForceSynchronous)
		{
			QueuedPool = 0;
		}
		// 如果需要异步执行，就创建完成Event。然后把自己添加到线程池的工作列表中
		if (QueuedPool)
		{
			if (!DoneEvent)
			{
				DoneEvent = FPlatformProcess::GetSynchEventFromPool(true);
			}
			DoneEvent->Reset();
			QueuedPool->AddQueuedWork(this, InQueuedWorkPriority);
		}
		// 如果是同步执行，就销毁掉完成Event，直接调用DoWork
		else 
		{
			DestroyEvent();
			DoWork();
		}
	}
	
	// 同步执行
	void StartSynchronousTask(...)
	{
		Start(true, GThreadPool, InQueuedWorkPriority, 
			InQueuedWorkFlags, InRequiredMemory, InDebugName);
	}
	// 异步执行
	void StartBackgroundTask(...)
	{
		Start(false, InQueuedPool, InQueuedWorkPriority, 
			InQueuedWorkFlags, InRequiredMemory, InDebugName);
	}
	// 确保任务完成。
	void EnsureCompletion(
		bool bDoWorkOnThisThreadIfNotStarted = true,
		bool bIsLatencySensitive = false)// 这参数没用
	{
		bool DoSyncCompletion = true;
		// 如果任务还没开始就在当前线程同步执行
		if (bDoWorkOnThisThreadIfNotStarted)
		{
			// 如果线程池存在，就说明是异步执行的任务。这时就需要先从线程池中回收掉任务，然后在同步执行任务
			if (QueuedPool)
			{
				if (QueuedPool->RetractQueuedWork(this))
				{
					DoSyncCompletion = false;
					DoWork(); 
					FinishThreadedWork();
					QueuedPool = 0;
				}
			}
			// 如果线程池不存在，说明是同步执行的任务。同步执行的任务这里直接执行
			else if (WorkNotFinishedCounter.GetValue())
			{
				DoWork(); 
			}
		}
		// 这里面会阻塞当前线程，等待DoneEvent的唤醒。也就是会阻塞一直等待任务的完成
		if (DoSyncCompletion)
		{
			SyncCompletion(bIsLatencySensitive);
		}
	}
	// 重新分配线程池去执行任务
	bool Reschedule(
		FQueuedThreadPool* InQueuedPool = GThreadPool, 
		EQueuedWorkPriority InQueuedWorkPriority = EQueuedWorkPriority::Normal)
	{
		if (QueuedPool)
		{
			if (QueuedPool->RetractQueuedWork(this))
			{
				QueuedPool = InQueuedPool;
				Priority = InQueuedWorkPriority;
				QueuedPool->AddQueuedWork(this, InQueuedWorkPriority);
				
				return true;
			}
		}
		return false;
	}
}
```

# FAsyncTask
```cpp
// 更上层的封装，能自己传Task模板，自己定义工作内容
template<typename TTask>
class FAsyncTask
	: public FAsyncTaskBase
{
	TTask Task;
}
```

# FAutoDeleteAsyncTask
```cpp
// 也是一个上层的封装，在任务完成后，会执行delete this;
template<typename TTask>
class FAutoDeleteAsyncTask : 
	private UE::FInheritedContextBase, 
	private IQueuedWork
{
	
}
```