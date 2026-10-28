# FQueuedThread

```cpp
class FQueuedThread: public FRunnable
{
	FEvent* DoWorkEvent = nullptr;
	TAtomic<bool> TimeToDie { false };
	IQueuedWork* volatile QueuedWork = nullptr;
	class FQueuedThreadPoolBase* OwningThreadPool = nullptr;
	FRunnableThread* Thread = nullptr;
	
	uint32 FQueuedThread::Run()
	{
		/*
		enum class EMemoryOrder
		{
		    Relaxed,// 保证原子性，不额外建立对其他内存访问的同步关系
		    SequentiallyConsistent,// 顺序一致性：提供更强的排序保证，所有采用该内存序的原子操作具有统一的全局顺序
		    Count//枚举数量标记，不用于指定内存序
		};
		
		*/
		while (!TimeToDie.Load(EMemoryOrder::Relaxed))
		{
			SET_DWORD_STAT(STAT_ThreadPoolDummyCounter, 0);
			bool bContinueWaiting = true;
			if (bContinueWaiting)
			{
				DoWorkEvent->Wait();
			}
	
			IQueuedWork* LocalQueuedWork = QueuedWork;
			QueuedWork = nullptr;
			FPlatformMisc::MemoryBarrier();
			while (LocalQueuedWork)
			{
				// Tell the object to do the work
				LocalQueuedWork->DoThreadedWork();
				// Let the object cleanup before we remove our ref to it
				LocalQueuedWork = OwningThreadPool->ReturnToPoolOrGetNextJob(this);
			}
		}
		return 0;
		}
}
```