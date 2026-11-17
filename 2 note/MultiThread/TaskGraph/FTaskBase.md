# FTaskBase
```cpp
class FTaskBase : private UE::FInheritedContextBase
{
	// 一个计数器，表示自己有多少个先决Task
	static constexpr uint32 NumInitialLocks = 1;
	std::atomic<uint32> NumLocks{ NumInitialLocks };
	// 一个计数器，表示自己是多少个Task的先决Task。
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
bool AddPrerequisites(FTaskBase& Prerequisite)
{
	// 增加锁定计数，自己每有一个先决Task旧需要增加一个锁定计数
	uint32 PrevNumLocks = NumLocks.fetch_add(1, std::memory_order_relaxed);
	// 把自己添加到先决Task的Subsequent数组中。线程安全的，会上锁
	if (!Prerequisite.AddSubsequent(*this))
	{
		// 
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