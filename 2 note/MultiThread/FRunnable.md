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