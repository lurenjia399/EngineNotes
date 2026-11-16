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
	Prerequisite.AddRef();
	Prerequisites.Push(&Prerequisite);
	return true;
}
```