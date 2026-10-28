# FQueuedThread

```cpp
class FQueuedThread: public FRunnable
{
	FEvent* DoWorkEvent = nullptr;
	TAtomic<bool> TimeToDie { false };// 销毁的标志位，原子操作
	IQueuedWork* volatile QueuedWork = nullptr;
	class FQueuedThreadPoolBase* OwningThreadPool = nullptr;
	FRunnableThread* Thread = nullptr;
	
	// 重写父类的方法，线程运行时会执行
	uint32 FQueuedThread::Run()
	{
		/*
		enum class EMemoryOrder
		{
		    Relaxed,// 保证原子性，不额外建立对其他内存访问的同步关系，没有顺序保证
		    SequentiallyConsistent,// 顺序一致性，线程A在按照SC改变数据后，线程B按照SC读取数据。线程B只要能读取到新值，那么就能保证 “线程A在改变数据之前的写入” 都能读到。必须写入和读取双方都使用SC的形式才能起作用。
		    Count//没用，标记枚举数量
		};
		1 这里TimeToDie.Load的操作，只是读取TimeToDie标志位的值，不保证修改标志位之前写入数据的顺序
		2 没有销毁while循环才会执行
		*/
		while (!TimeToDie.Load(EMemoryOrder::Relaxed))
		{
			// 通过DoWorkEvent一直等待，等待Event触发后，才执行后续的代码
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
	
	// 唤醒线程的方法，把具体工作当作参数传进来
	void DoWork(IQueuedWork* InQueuedWork)
	{
		QueuedWork = InQueuedWork;
		DoWorkEvent->Trigger();
	}
}
```