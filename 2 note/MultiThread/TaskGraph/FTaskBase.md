# FTaskBase
```cpp
class FTaskBase : private UE::FInheritedContextBase
{
	static constexpr uint32 NumInitialLocks = 1;
	std::atomic<uint32> NumLocks{ NumInitialLocks };

	FPipe* Pipe{ nullptr };

	FEventCount StateChangeEvent;

	EExtendedTaskPriority ExtendedPriority; // internal priorities, if any

	std::atomic<uint32> ExecutingThreadId = FThread::InvalidThreadId;
	
	FSubsequents<TInlineAllocator<1>> Subsequents;
	
	// 一个计数器，表示有多少个Task依赖自己
	std::atomic<uint32> RefCount;
}
```

# AddPrerequisites

```cpp
bool AddPrerequisites(FTaskBase& Prerequisite)
{
	uint32 PrevNumLocks = NumLocks.fetch_add(1, std::memory_order_relaxed);
	if (!Prerequisite.AddSubsequent(*this))
	{
		NumLocks.fetch_sub(1, std::memory_order_relaxed);
		return false;
	}
	// 增加先决Task的引用。线程安全的，原子操作
	Prerequisite.AddRef();
	// 把先决Task添加到数组中。线程安全的，添加之前会上锁
	Prerequisites.Push(&Prerequisite);
	
	return true;
}
```