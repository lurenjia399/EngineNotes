# FThread
```cpp
FThread::FThread(
	TCHAR const* ThreadName,
	TUniqueFunction<void()>&& ThreadFunction,
	TUniqueFunction<void()>&& SingleThreadTickFunction,
	uint32 StackSize,
	EThreadPriority ThreadPriority,
	FThreadAffinity ThreadAffinity,
	EForkable IsForkable
)
	: Impl(MakeShared<FThreadImpl, ESPMode::ThreadSafe>(
		ThreadName, 
		MoveTemp(ThreadFunction), 
		MoveTemp(SingleThreadTickFunction), 
		StackSize, ThreadPriority, ThreadAffinity, IsForkable))
{
	// 在构造函数中初始化Impl，Impl是继承FRunable的Share智能指针
	Impl->Initialize(Impl);
}

void FThread::Join()
{
	// zhix
	Impl->Join();
	Impl.Reset();
}
```