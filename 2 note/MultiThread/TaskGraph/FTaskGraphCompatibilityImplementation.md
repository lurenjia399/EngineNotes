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
		// 拆除执行线程的哪个队列，MainQueue还是LocalQueue
		int32 QueueToExecuteOn = ENamedThreads::GetQueueIndex(InThreadToExecuteOn);
		// 拆出是哪个线程，这三个Game,Render,RHI其中一个
		InThreadToExecuteOn = ENamedThreads::GetThreadIndex(InThreadToExecuteOn);
		FTaskThreadBase* Target = &Thread(InThreadToExecuteOn);
		// 判断是否跨线程，通过不同方法，把任务添加到队列中。两个方法的添加队列没区别，区别在于唤醒线程，如果不跨线程说明线程在运行不用唤醒
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

# ProcessThreadUntilIdle
```cpp
uint64 ProcessThreadUntilIdle(ENamedThreads::Type CurrentThread) final override
{
	int32 QueueIndex = ENamedThreads::GetQueueIndex(CurrentThread);
	CurrentThread = ENamedThreads::GetThreadIndex(CurrentThread);
	return Thread(CurrentThread).ProcessTasksUntilIdle(QueueIndex);
}
```
1 执行一次任务，从QueueIndex工作队列中取任务执行

# ProcessThreadUntilRequestReturn
```cpp
void ProcessThreadUntilRequestReturn(
	ENamedThreads::Type CurrentThread) final override
{
	int32 QueueIndex = ENamedThreads::GetQueueIndex(CurrentThread);
	CurrentThread = ENamedThreads::GetThreadIndex(CurrentThread);
	check(CurrentThread >= 0 && CurrentThread < NumNamedThreads);
	check(CurrentThread == GetCurrentThread());
	Thread(CurrentThread).ProcessTasksUntilQuit(QueueIndex);
}
```
1 死循环一直执行任务，取不到任务就等待

# RequestReturn
```cpp
void RequestReturn(ENamedThreads::Type CurrentThread) final override
{
	int32 QueueIndex = ENamedThreads::GetQueueIndex(CurrentThread);
	CurrentThread = ENamedThreads::GetThreadIndex(CurrentThread);
	check(CurrentThread != ENamedThreads::AnyThread);
	Thread(CurrentThread).RequestQuit(QueueIndex);
}
```
1 请求退出执行任务的死循环

# WaitUntilTasksComplete
```cpp
void WaitUntilTasksComplete(const FGraphEventArray& Tasks, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread) final override
{
	
}
```