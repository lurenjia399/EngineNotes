# FQueuedThread

```cpp
class FQueuedThread: public FRunnable
{
	FEvent* DoWorkEvent = nullptr;
	TAtomic<bool> TimeToDie { false };// 源自标志位
	IQueuedWork* volatile QueuedWork = nullptr;
	class FQueuedThreadPoolBase* OwningThreadPool = nullptr;
	FRunnableThread* Thread = nullptr;
	
	uint32 FQueuedThread::Run()
	{
		/*
		enum class EMemoryOrder
		{
		    Relaxed,// 保证原子性，不额外建立对其他内存访问的同步关系，没有顺序保证
		    SequentiallyConsistent,// 顺序一致性：提供更强的排序保证，所有采用该内存序的原子操作具有统一的全局顺序
		    Count//没用，标记枚举数量
		};
		1 这里TimeToDie.Load的操作，只是读取TimeToDie标志位的值，不保证修改标志位之前写入数据的顺序
		2 没有销毁while循环才会执行
		*/
		while (!TimeToDie.Load(EMemoryOrder::Relaxed))
		{
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