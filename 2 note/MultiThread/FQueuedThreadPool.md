# FQueuedThreadPool
```cpp
class FQueuedThreadPool
{
	// 线程池创建线程的方法，不同的线程池重写
	virtual bool Create(
		uint32 InNumQueuedThreads, 
		uint32 StackSize = (32 * 1024), 
		EThreadPriority ThreadPriority = TPri_Normal, 
		const TCHAR* Name = TEXT("UnknownThreadPool")) = 0;
		
	// 线程池销毁线程的方法，不同的线程池重写
	virtual void Destroy() = 0;
	
	// 向线程池添加任务，如果有空闲的线程就执行任务，如果没有就放到优先级队列里
	virtual void AddQueuedWork( 
		IQueuedWork* InQueuedWork, 
		EQueuedWorkPriority InQueuedWorkPriority= EQueuedWorkPriority::Normal) = 0;
	
	// 线程池尝试回收任务
	virtual bool RetractQueuedWork(IQueuedWork* InQueuedWork) = 0;
	
	// 基类的静态分配方法，默认new一个普通的线程池
	static CORE_API FQueuedThreadPool* Allocate()  
	{  
	    return new FQueuedThreadPoolBase;  
	}
}
```
基类的实现，不同的线程池需要继承然后重写方法。
# FQueuedThreadPoolBase

```cpp
class FQueuedThreadPoolBase : public FQueuedThreadPool
{
	// 创建线程池，游戏中全局线程池是在FEngineLoop::PreInitPreStartupScreen方法中创建的
	bool CreateInternal(
		bool bForkable, // 创建的是能够配合 Fork 流程工作的线程。Fork是创建子进程，子进程继承父进程当时的内存状态，这种设计常见于 Linux 专用服务器：先在父进程加载资源，再 Fork 出多个服务器子进程，以共享部分内存、降低启动成本。
		uint32 InNumQueuedThreads, // 线程池创建的线程数量
		uint32 StackSize, // 每个线程栈的大小
		EThreadPriority ThreadPriority, // 线程的优先级
		const TCHAR* Name)// 线程池的名称
	{
		// 创建新的临界区，并在函数范围内上锁
		SynchQueue = new FCriticalSection();
		FScopeLock Lock(SynchQueue);
		// 重置线程池中的线程队列和工作队列
		QueuedThreads.Empty(InNumQueuedThreads);
		QueuedWork.Reset();
		// 遍历需要创建的线程，每个线程都new一个新的FRunnable干活的类。把成功创建的塞到线程队列中。
		for (uint32 Count = 0; Count < InNumQueuedThreads && bWasSuccessful == true; Count++)
		{
			// 在堆上new出新的FRunnable，所有线程都能读到。
			FQueuedThread* pThread = new FQueuedThread();
			const bool bCreateSuccess = bForkable ?
				pThread->CreateForkable(this, StackSize, ThreadPriority, *ThreadName) :
				pThread->Create(this, StackSize, ThreadPriority, *ThreadName);
			if (bCreateSuccess)
			{
				QueuedThreads.Add(pThread);
				AllThreads.Add(pThread);
			}
			else
			{
				bWasSuccessful = false;
				delete pThread;
			}
		}
		// 如果有一个失败了就清掉所有的。
		if (bWasSuccessful == false)  
		{  
		    Destroy();  
		}
		
		return bWasSuccessful;
	}
	// 销毁线程池的方法
	virtual void Destroy() override final
	{
		// 临界区存在
		if (SynchQueue)
		{
			{
				// 上锁临界区
				FScopeLock Lock(SynchQueue);
				// 标记销毁的标志位
				TimeToDie = 1;
				// 添加内存屏障，能够保证屏障前的操作（TimeToDie = 1;）先于屏障后的操作(清掉工作列表中的内容)。
				FPlatformMisc::MemoryBarrier();
				// 删掉工作列表中的内容并清空工作列表
				while (IQueuedWork * WorkItem = QueuedWork.Dequeue())
				{
					WorkItem->Abandon();
				}
				QueuedWork.Reset();
			}
			// 通过AllThreads.Num() == QueuedThreads.Num()判断来确定池中的线程都是空闲的，AllThreads数组表示这个池中所有的线程，QueuedThreads数组表示池中空闲的线程
			while (1)
			{
				{
					FScopeLock Lock(SynchQueue);
					if (AllThreads.Num() == QueuedThreads.Num())
					{
						break;
					}
				}
				FPlatformProcess::Sleep(0.0f);
			}
			// 删掉所有的线程
			{
				FScopeLock Lock(SynchQueue);
				for (int32 Index = 0; Index < AllThreads.Num(); Index++)
				{
					AllThreads[Index]->KillThread();
					delete AllThreads[Index];
				}
				QueuedThreads.Empty();
				AllThreads.Empty();
			}
			// 删掉临界区
			delete SynchQueue;
			SynchQueue = nullptr;
		}
	}
	
	void AddQueuedWork(
		IQueuedWork* InQueuedWork, // 往工作列表中添加的工作
		EQueuedWorkPriority InQueuedWorkPriority) override// 工作的优先级
	{
		// 池的销毁标记已经为true，就不能往池中新加工作了，直接返回
		if (TimeToDie)
		{
			InQueuedWork->Abandon();
			return;
		}
		
		{
			// 临界区上锁
			FScopeLock sl(SynchQueue);
			
			// 如果池中没有空闲的线程，就把工作放到工作队列中
			const int32 AvailableThreadCount = QueuedThreads.Num();
			if (AvailableThreadCount == 0)
			{
				QueuedWork.Enqueue(InQueuedWork, InQueuedWorkPriority);
				return;
			}
			// 如果池中有空闲的线程，就从空闲队列中取出线程
			const int32 ThreadIndex = AvailableThreadCount - 1;
			Thread = QueuedThreads[ThreadIndex];
			QueuedThreads.RemoveAt(ThreadIndex, EAllowShrinking::No);
		}

		// 用取出的线程执行工作
		Thread->DoWork(InQueuedWork);
	}
	// 池回收线程，并从工作队列中取出下一个工作
	IQueuedWork* ReturnToPoolOrGetNextJob(FQueuedThread* InQueuedThread)
	{
		IQueuedWork* Work = nullptr;
		// 上锁
		FScopeLock sl(SynchQueue);
		// 如果已经标记销毁了，工作队列中就不应该有工作了
		if (TimeToDie)
		{
			check(!QueuedWork.Num());
		}
		// 从工作队列中取出下一个工作
		Work = QueuedWork.Dequeue();
		if (!Work)
		{
			QueuedThreads.Add(InQueuedThread);
		}
		return Work;
	}
}
```