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
// 添加先决Task，先决Task执行完自己才能执行
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