# 创建

```cpp
1 FTaskGraphCompatibilityImplementation是一个静态全局的对象，继承FTaskGraphInterface
2 在引擎的初始化PreInitPreStartupScreen方法中，创建出来
3 这个类构造函数中会通过FScheduler来创建出工作线程，紧接着就会创建出3个FNamedTaskThread，分别是RHIThread，GameThread，RenderThread。其中NamedTaskThread是继承FRunnable的
```

# QueueTask
```cpp
void QueueTask(
	class FBaseGraphTask* Task, // 工作任务
	bool bWakeUpWorker, // 是否唤醒线程
	ENamedThreads::Type InThreadToExecuteOn, // 任务需要在哪个NamedThread上执行
	ENamedThreads::Type InCurrentThreadIfKnown) override // 在哪个线程执行的这个方法
{
	// 确定当前是哪个线程执行的QueueTask这个方法，把结果赋值到CurrentThreadIfKnown
	ENamedThreads::Type CurrentThreadIfKnown;
	if (ENamedThreads::GetThreadIndex(InCurrentThreadIfKnown) 
		== ENamedThreads::AnyThread)
	{
		CurrentThreadIfKnown = GetCurrentThread();
	}
	else
	{
		CurrentThreadIfKnown = 
			ENamedThreads::GetThreadIndex(InCurrentThreadIfKnown);
	}
	
	{
		// 需要执行任务的线程
		int32 QueueToExecuteOn = ENamedThreads::GetQueueIndex(InThreadToExecuteOn);
		InThreadToExecuteOn = ENamedThreads::GetThreadIndex(InThreadToExecuteOn);
		// 拿到执行任务的线程Target
		FTaskThreadBase* Target = &Thread(InThreadToExecuteOn);
		// 如果如果执行线程等于当前线程，就把任务添加到
		if (InThreadToExecuteOn == 
			ENamedThreads::GetThreadIndex(CurrentThreadIfKnown))
		{
			Target->EnqueueFromThisThread(QueueToExecuteOn, Task);
		}
		else
		{
			Target->EnqueueFromOtherThread(QueueToExecuteOn, Task);
		}
	}
}
```