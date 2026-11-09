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

# FWaitingQueue
```cpp
class FWaitingQueue
{
	uint32                         ThreadCount{ 0 };
	uint32                         MaxThreadCount{ 0 };
	TFunction<void()>              CreateThread;
	std::atomic<uint32>            Oversubscription{ 0 };
	std::atomic<uint64>            State;
	std::atomic<uint64>            StandbyState;
	TAlignedArray<FWaitEvent>&     NodesArray;
	std::atomic<bool>              bIsShuttingDown{ false };
	FOversubscriptionLimitReached& OversubscriptionLimitReachedEvent;
}
```