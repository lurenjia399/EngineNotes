# FTaskBase
```cpp
class FTaskBase : private UE::FInheritedContextBase
{
	// 执行的标记，如果任务开始执行了，会将ExecutionFlag + 1设置到NumLocks上
	static constexpr uint32 ExecutionFlag = 0x80000000;
	// 一个计数器，表示自己有多少个先决Task。为0说明自己可以执行了，为ExecutionFlag说明已经执行完了
	static constexpr uint32 NumInitialLocks = 1;
	std::atomic<uint32> NumLocks{ NumInitialLocks };
	// 一个计数器，表示自己是多少个Task的先决Task。为0了说明自己可以销毁了
	std::atomic<uint32> RefCount;
	// 后续数组
	FSubsequents<TInlineAllocator<1>> Subsequents;

	FPipe* Pipe{ nullptr };

	FEventCount StateChangeEvent;

	// 标识Task，是TaskEvent还是普通Task，还是Inline，还是NamedTask
	EExtendedTaskPriority ExtendedPriority;
	
	// 执行自己的线程
	std::atomic<uint32> ExecutingThreadId = FThread::InvalidThreadId;
	
	
}
```

# AddPrerequisites
```cpp
// 添加先决Task，先决Task执行完自己才能执行
bool AddPrerequisites(FTaskBase& Prerequisite)
{
	// 增加锁定计数，自己每有一个先决Task旧需要增加一个锁定计数
	uint32 PrevNumLocks = NumLocks.fetch_add(1, std::memory_order_relaxed);
	// 如果Task已经执行了，说明PrevNumLocks等于ExecutionFlag，如果PrevNumLocks + 1 > ExecutionFlag说明执行过程中添加了新的依赖，不允许直接崩掉
	checkf(PrevNumLocks + 1 < ExecutionFlag, 
		TEXT("Max number of task prerequisites reached: %d"), ExecutionFlag);

	// 把自己添加到先决Task的Subsequent数组中。线程安全的，会上锁
	if (!Prerequisite.AddSubsequent(*this))
	{
		// 添加后续失败了，回退
		NumLocks.fetch_sub(1, std::memory_order_relaxed);
		return false;
	}
	// 增加先决Task的引用。线程安全的，原子操作
	Prerequisite.AddRef();
	// 把先决Task添加到数组中。线程安全的，会上锁
	Prerequisites.Push(&Prerequisite);
	
	return true;
}
```

# AddNested
```cpp
// 添加嵌套Task，嵌套Task执行完自己才能完成。详情可见DontCompleteUntil方法
void AddNested(FTaskBase& Nested)
{
	// 增加锁定计数，等待执行完的Task数量
	uint32 PrevNumLocks = NumLocks.fetch_add(1, std::memory_order_relaxed);
	// NumLocks必须大于ExecutionFlag，因为只能在任务执行过程中添加内嵌任务
	checkf(PrevNumLocks > ExecutionFlag, 
		TEXT("Internal error: nested tasks can be added only during parent's execution (%u)"), PrevNumLocks);

	// 把自己添加到嵌套Task的后续数组中
	if (Nested.AddSubsequent(*this))
	{
		// 增加嵌套Task的引用计数。表示排在嵌套Task之后的数量
		Nested.AddRef();
		// 把嵌套Task添加到先决数组中
		Prerequisites.Push(&Nested);
	}
	else
	{
		// 如果添加后续数组失败了，就回退锁定数量
		NumLocks.fetch_sub(1, std::memory_order_relaxed);
	}
}
```

# TryLaunch
```cpp
// 外部的调用接口，尝试执行这个任务
bool TryLaunch(uint64 TaskSize)
{
	bool bWakeUpWorker = true;
	return TryUnlock(bWakeUpWorker);
}

bool TryUnlock(bool& bWakeUpWorker)
{
	FPipe* LocalPipe = GetPipe();
	// LockNum减一，每一次尝试解锁都会减少计数
	uint32 PrevNumLocks = NumLocks.fetch_sub(1, std::memory_order_acq_rel);
	uint32 LocalNumLocks = PrevNumLocks - 1;
	// 如果小于ExecutionFlag，说明现在任务还没执行
	if (PrevNumLocks < ExecutionFlag)  
	{
		/*
		1 如果Pipe为空，NumLocks为0说明没有先决Task了。
		2 如果Pipe不为空，NumLocks<=1说明没有先决Task了。
		3 没有先决Task就意味着可以执行任务了
		*/
	    bool bPrerequisitesCompleted = LocalPipe == nullptr 
		    ? LocalNumLocks == 0 : LocalNumLocks <= 1;
	    if (!bPrerequisitesCompleted)  
	    {       
		    return false;  
	    }
	    
	    if (LocalPipe != nullptr)  
	    {  
			bool bFirstPipingAttempt = LocalNumLocks == 1;  
		    if (bFirstPipingAttempt)  
			{          
				FTaskBase* PrevPipedTask = TryPushIntoPipe();  
			    if (PrevPipedTask != nullptr) // the pipe is blocked  
				{
			        return false;  
			    }
	          NumLocks.store(0, std::memory_order_release); // release pipe's lock 
			}  
	    }
	    // 如果是InLine，说明在当前线程执行
	    if (ExtendedPriority == EExtendedTaskPriority::Inline)  
	    {
		    // 尝试执行Task任务
			TryExecuteTask();
			// Task任务手动执行完了就需要手动释放引用。如果把Task交给FScheduler执行就不需要手动释放，FScheduler在执行的时候会把Task右移到局部变量上，局部变量析构的时候自动释放引用。通过Task的Init方法中的deleter实现的。ReleaseInternalReference方法执行的和FShceduler执行的是一个方法，区别在于ReleaseInternalReference不会执行任务，只会右移到局部变量上，因为Cancel标志位。
			ReleaseInternalReference();
	    }
	    // 如果是TaskEvent，说明是空实现，空任务只是用来占位的，这里就直接修改相关标志位。详情可以看FGraphEventImpl
	    else if (ExtendedPriority == EExtendedTaskPriority::TaskEvent)  
	    {
			// 每次执行Task都需要设置LockNum为ExecutionFlag+1
			if (TrySetExecutionFlag())
			{
				// 释放先决Task记录的RefCount
				ReleasePrerequisites();
				// 结束自己，让后续Task执行TryUnlock
				Close();
				ReleaseInternalReference();
			}  
	    }
	    // 如果都不是，就调度执行，把任务放到FScheduler中调度执行
	    else  
	    {  
	       Schedule(bWakeUpWorker);
	    }
	    // 如果执行了就返回true
	    return true;  
	}
	// 
	
	/*
	1 如果PrevNumLocks>=ExecutionFlag，说明现在任务已经执行完了，但是没有释放资源，因为在执行任务的过程中添加了NestedTask，必须等NestedTask执行完了才能释放资源
	2 如果LocalNumLocks != ExecutionFlag，说明有多个NestedTask，有的NestedTask还没执行完，执行完了才能Close，Release
	*/
	if (LocalNumLocks != ExecutionFlag) 
	{
		return false;
	}
	// 自己执行完了，尝试执行后续Task
	Close();
	// 释放自己的引用计数
	Release();
	return true;
}
```

# TryExecuteTask
```cpp
bool TryExecuteTask()
{
	// 抢占执行权，通过CAS的操作，抢占成功了就设置NumLocks为ExecutionFlag+1。抢占失败了就说明被其他线程抢占了
	if (!TrySetExecutionFlag())
	{
		return false;
	}
	
	// 增加引用计数，如果在执行的过程中添加了NestedTask，为了让自己存活到NestedTask执行完成才销毁，所以这里增加了这个AddRef操作。原子操作
	AddRef();
	// 释放先决Task的引用计数
	ReleasePrerequisites();
	// 设置当前要执行的Task为自己
	FTaskBase* PrevTask = ExchangeCurrentTask(this);
	// 保存执行this->Task的线程
	ExecutingThreadId.store(FPlatformTLS::GetCurrentThreadId(), 
		std::memory_order_relaxed);
	// 如果是Pipe的task，就开始执行Pipe
	if (GetPipe() != nullptr)
	{
		StartPipeExecution();
	}
	// 执行任务的方法
	ExecuteTask();
	// 结束执行Pipe
	if (GetPipe() != nullptr)
	{
		FinishPipeExecution();
	}
	// 清掉执行this->Task的线程
	ExecutingThreadId.store(FThread::InvalidThreadId, std::memory_order_relaxed);
	// 设置当前要执行的Task为空
	ExchangeCurrentTask(PrevTask);
	// 减少Numlocks。如果减少后为ExecutionFlag，或
	/*
	1 减少Numlocks
	2 如果减少后等于ExecutionFlag，说明this正常执行完成了，就执行Close通知后续Task执行TryUnlock，然后释放刚才AddRef添加的引用
	3 如果减少后不等于ExecutionFlag，说明在执行的过程中this添加了NestedTask，如果有NestedTask就不算执行完，需要等NestedTask执行完，NestedTask执行完后会执行自己的Tryunlock方法，在里面释放刚才AddRef添加的引用
	*/
	uint32 LocalNumLocks = NumLocks.fetch_sub(1, std::memory_order_acq_rel) - 1; 
	if (LocalNumLocks == ExecutionFlag)
	{
		Close();
		Release();
	}
	return true;
}
```
1 任务执行的方法，没有通过FScheduler调度执行，在当前线程主动执行
2 首先第一步抢占执行权，设置NumLocks为ExecutionFlag+1，在执行完后会减一
3 然后就是AddRef，目的是延长自己的生命周期，如果在执行的过程中给自己添加了NestedTask，自己的销毁就得在所有的NestedTask执行完成后才能销毁，所以这里添加引用

# Schedule

```cpp
void FTaskBase::Schedule(bool& bWakeUpWorker)
{
	if (IsNamedThreadTask())
	{
		FTaskGraphInterface::Get().QueueTask(static_cast<FBaseGraphTask*>(this), true, TranslatePriority(ExtendedPriority));
		return;
	}
	
	StateChangeEvent.Notify();
	bWakeUpWorker |= 
		LowLevelTasks::FScheduler::Get().TryLaunch(LowLevelTask, 
		bWakeUpWorker ? LowLevelTasks::EQueuePreference::GlobalQueuePreference 
		: LowLevelTasks::EQueuePreference::LocalQueuePreference, bWakeUpWorker);
}
```

1 NamedThreadTask执行路径不一样，NamedTask就包括GameThread，Render，RHI这些线程的任务，他们是直接把任务塞到FTaskGraphCompatibilityImplementation类中执行，这个类里面缓存了这三个NamedThread，把任务放到他们的任务队列中。
2 普通Task执行就是通过Scheduler系统了，把任务放到对应的工作队列中，等线程去取或者偷，然后执行

