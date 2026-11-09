# FWaitEvent

```cpp
struct alignas(64) FWaitEvent
{
	std::atomic<uint64>     Next{ 0 };
	uint64                  Epoch{ 0 };
	std::atomic<EWaitState> State{ EWaitState::NotSignaled };
	FEventRef               Event{ EEventMode::ManualReset };
	bool                    bIsStandby{ false };
};
```