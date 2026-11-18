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

	FPipe* Pipe{ nullptr };

	FEventCount StateChangeEvent;

	EExtendedTaskPriority ExtendedPriority; // internal priorities, if any

	std::atomic<uint32> ExecutingThreadId = FThread::InvalidThreadId;
	
	FSubsequents<TInlineAllocator<1>> Subsequents;
}
```

# 依赖实现

```cpp
// 添加先决Task，先决Task执行完自己才能执行
bool AddPrerequisites(FTaskBase& Prerequisite)
{
	// 增加锁定计数，自己每有一个先决Task旧需要增加一个锁定计数
	uint32 PrevNumLocks = NumLocks.fetch_add(1, std::memory_order_relaxed);
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

```cpp
// 添加嵌套Task，嵌套Task执行完自己才能执行
void AddNested(FTaskBase& Nested)
{
	// 增加锁定计数，等待执行完的Task数量
	uint32 PrevNumLocks = NumLocks.fetch_add(1, std::memory_order_relaxed);
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
	    // 如果是TaskEvent，说明是空实现，空任务只是用来占位的，这里就直接修改相关标志位
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
	
	if (LocalNumLocks != ExecutionFlag) // still locked
	{
		return false;
	}
	
	Close();
	Release();
	return true;
}
```