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
				// 添加内存屏障，能够保证屏障前的操作（TimeToDie = 1;）先于屏障后的操作(遍历清掉所有的工作)。
				FPlatformMisc::MemoryBarrier();
				// Clean up all queued objects
				while (IQueuedWork * WorkItem = QueuedWork.Dequeue())
				{
					WorkItem->Abandon();
				}
				
				QueuedWork.Reset();
			}
			// wait for all threads to finish up
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
			// Delete all threads
			{
				FScopeLock Lock(SynchQueue);
				// Now tell each thread to die and delete those
				for (int32 Index = 0; Index < AllThreads.Num(); Index++)
				{
					AllThreads[Index]->KillThread();
					delete AllThreads[Index];
				}
				QueuedThreads.Empty();
				AllThreads.Empty();
			}
			delete SynchQueue;
			SynchQueue = nullptr;
		}
	}
}
```