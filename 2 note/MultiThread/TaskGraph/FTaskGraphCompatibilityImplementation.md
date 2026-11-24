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
1 执行一次任务，从QueueIndex工作队列中取任务执行，取不到任务就返回不取了

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
void WaitUntilTasksComplete(
	const FGraphEventArray& Tasks, 
		ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
{
	// 首先执行一遍需要执行的Task，如果全部执行完了，就返回
	bool bAllTasksCompleted = true;
	for (const FGraphEventRef& Task : Tasks)
	{
		if (Task.IsValid())
		{
			Task->TryRetractAndExecute(NeverTimeout);

			bAllTasksCompleted &= Task->IsCompleted();
		}
	}
	if (bAllTasksCompleted)
	{
		return;
	}
	
	// 确定当前线程是哪个，是NamedTask还是工作线程
	if (ENamedThreads::GetThreadIndex(CurrentThreadIfKnown) 
		== ENamedThreads::AnyThread)
	{
		CurrentThreadIfKnown = ENamedThreads::GetThreadIndex(GetCurrentThread());
		NamedThreadWithFlags = ENamedThreads::SetPriorities(
			CurrentThreadIfKnown, Priority, bIsHiPri);
	}
	else
	{
		CurrentThreadIfKnown = ENamedThreads::GetThreadIndex(CurrentThreadIfKnown);
	}
	
	WaitOnNamedThreadForTasks(Tasks, CurrentThreadIfKnown, NamedThreadWithFlags);

}
```

```cpp
void WaitOnNamedThreadForTasks(const FGraphEventArray& Tasks, ENamedThreads::Type BaseNamedThread, ENamedThreads::Type NamedThreadWithFlags)
{
	if (
		BaseNamedThread != ENamedThreads::AnyThread // 当前不是工作线程
		&& BaseNamedThread < NumNamedThreads // 当前是NamedThread
		&& !IsThreadProcessingTasks(NamedThreadWithFlags)) // 当前线程没有在处理任务
	{
		// 创建一个依赖所需任务执行完成的Task，直接尝试TryLaunch
		TGraphTask<FReturnGraphTask>::CreateTask(&Tasks, NamedThreadWithFlags)
			.ConstructAndDispatchWhenReady(NamedThreadWithFlags);
		// 当前NamedThread一直死循环执行任务，直到FReturnGraphTask执行完成，才会执行RequestReturn方法退出死循环。
		ProcessThreadUntilRequestReturn(NamedThreadWithFlags);
		return;
	}
	
	// 创建系统Event，在FScopedEvent析构的时候开始Wait
	FScopedEvent Event;
	TriggerEventWhenTasksComplete(Event.Get(), Tasks, BaseNamedThread);
}
```

```cpp
void TriggerEventWhenTasksComplete(
	FEvent* InEvent, // 系统事件，用来等待，唤醒
	const FGraphEventArray& Tasks, // 需要等待的Task
	ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread,//当前线程
	ENamedThreads::Type TriggerThread = ENamedThreads::AnyHiPriThreadHiPriTask)
{
	// 需要等待Task是否全部完成，true是没有，false是全部完成了
	bool bAnyPending = true;
	if (Tasks.Num() < 8)
	{
		bAnyPending = false;
		for (int32 Index = 0; Index < Tasks.Num(); Index++)
		{
			FGraphEvent* Task = Tasks[Index].GetReference();
			if (Task && !Task->IsComplete())
			{
				bAnyPending = true;
				break;
			}
		}
	}
	// 如果全部完成了，就触发系统事件，唤醒线程
	if (!bAnyPending)
	{
		TestRandomizedThreads();
		InEvent->Trigger();
		return;
	}
	
	// 如果没有全部完成，就创建一个Inline的任务TryUnlock，这个任务只用来唤醒Event
	// 为什么这里创建Inline的Task
	UE::Tasks::Launch(
			TEXT("TriggerEventWhenTaskComplete"),
			[InEvent]()
			{
				InEvent->Trigger();
			},
			Tasks,
			LowLevelTasks::ETaskPriority::Normal,
			UE::Tasks::EExtendedTaskPriority::Inline
		);
}

```
1 每次执行WaitUntilTasksComplete方法后，先尝试执行一遍需要等待的Task，如果Task完成了就结束
2 如果Task完不成，就判断当前执行的是哪种线程
3 如果是NamedThread，就创建一个新的FReturnGraphTask，其前置Task就是这些需要等待的Task，然后死循环从工作队列中取任务执行，没有任务执行就阻塞等待。最后一个前置Task执行完了就会把FReturnGraphTask塞到NamedThread工作队列中进而唤醒，唤醒完执行FReturnGraphTask的时候就会改变标志位进而退出死循环
4 如果不是NamedThread，就创建一个系统Event阻塞等待，然后创建一个Inline的新Task，新Task的前置Task也是这些需要等待的Task，等待新Task执行的时候在唤醒

# ProcessUntilTasksComplete
```cpp
bool ProcessUntilTasksComplete(
	const FGraphEventArray& Tasks, // 需要完成的Tasks
	ENamedThreads::Type CurrentThreadIfKnown, // 当前所在的线程
	const FProcessTasksUpdateCallback& IdleWorkUpdate) final override
{
	// 确认当前执行的线程是哪个，NamedThread还是工作线程，
	ENamedThreads::Type NamedThreadWithFlags = CurrentThreadIfKnown;
	if (ENamedThreads::GetThreadIndex(CurrentThreadIfKnown) 
		== ENamedThreads::AnyThread)
	{
		CurrentThreadIfKnown = ENamedThreads::GetThreadIndex(GetCurrentThread());
		NamedThreadWithFlags = ENamedThreads::SetPriorities(CurrentThreadIfKnown,
			 Priority, bIsHiPri);
	}
	else
	{
		CurrentThreadIfKnown = ENamedThreads::GetThreadIndex(CurrentThreadIfKnown);
	}
	
	// 初始化当前的操作类型为ProcessNamedThreadTasks，处理NamedThreadTask
	EProcessTasksOperation CurrentOperation = 
		EProcessTasksOperation::ProcessNamedThreadTasks;
	
	// 只要操作类型不是停止，就一直死循环
	while (CurrentOperation != EProcessTasksOperation::StopProcessing)
	{
		// 如果当前操作是处理NamedThreadTask，就执行ProcessThreadUntilIdle。从NamedThread的工作队列中取任务，取不到任务就返回
		if (CurrentOperation == EProcessTasksOperation::ProcessNamedThreadTasks)
		{
			ProcessThreadUntilIdle(NamedThreadWithFlags);
		}
		// 如果当前操作是ProcessAllOtherTasks或者ProcessOneOtherTask，就立即执行那些需要完成的任务
		else if (CurrentOperation == EProcessTasksOperation::ProcessAllOtherTasks
			 || CurrentOperation == EProcessTasksOperation::ProcessOneOtherTask)
		{
			bHasOtherTasks = false;
			for (int32 TaskIndex = 0; TaskIndex < RemainingTasks.Num(); 
				TaskIndex++)
			{
				FGraphEvent* Task = RemainingTasks[TaskIndex];
				if (!Task->IsNamedThreadTask())
				{
					bHasOtherTasks = true;
					if (Task->TryRetractAndExecute(NeverTimeout)&&
						CurrentOperation == 
							EProcessTasksOperation::ProcessOneOtherTask)
					{
						break;
					}
				}
			}
		}
		else if (CurrentOperation == EProcessTasksOperation::WaitUntilComplete)
		{
			FGraphEventArray TasksToWaitOn;
			for (FGraphEvent* TaskEvent : RemainingTasks)
			{
				TasksToWaitOn.Add(TaskEvent->GetCompletionEvent());
			}

			TaskTrace::FWaitingScope WaitingScope(GetTraceIds(TasksToWaitOn));
			WaitOnNamedThreadForTasks(TasksToWaitOn, CurrentThreadIfKnown, NamedThreadWithFlags);

			// All tasks must have succeeded
			return true;
		}

		// 更新需要完成任务数组，完成了一个任务就从数组中移除一个
		for (int32 TaskIndex = RemainingTasks.Num() - 1;TaskIndex >= 0;TaskIndex--)
		{
			if (RemainingTasks[TaskIndex]->IsCompleted())
			{
				RemainingTasks.RemoveAtSwap(TaskIndex, EAllowShrinking::No);
			}
		}
		// 如果所有需要完成的任务都完成了，就返回
		if (RemainingTasks.Num() == 0)
		{
			return true;
		}

		// 根据传进来的参数，确定操作类型。默认的是ProcessAllOtherTasks
		if (IdleWorkUpdate)
		{
			CurrentOperation = IdleWorkUpdate(RemainingTasks.Num());
		}
		else
		{
			CurrentOperation = EProcessTasksOperation::ProcessAllOtherTasks;
		}

		if ((CurrentOperation == EProcessTasksOperation::ProcessAllOtherTasks 
			|| CurrentOperation == EProcessTasksOperation::ProcessOneOtherTask)
			&& !bHasOtherTasks)
		{
			// If we have no more tasks to try and retract, just wait which will handle processing named thread tasks as well
			CurrentOperation = EProcessTasksOperation::WaitUntilComplete;
		}
	}
	return false;
}
```