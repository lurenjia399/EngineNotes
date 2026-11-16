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
}
```