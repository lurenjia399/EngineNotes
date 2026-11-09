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
	// 线程池中创建的线程数量
	uint32                         ThreadCount{ 0 };
	// 线程的最大数量，GTaskGraphOversubscriptionRatio来控制的
	uint32                         MaxThreadCount{ 0 };
	// 创建线程的函数指针
	TFunction<void()>              CreateThread;
	
	std::atomic<uint32>            Oversubscription{ 0 };
	/*
	状态位：
	1 [0，14] 表示
	*/
	std::atomic<uint64>            State;
	std::atomic<uint64>            StandbyState;
	// 控制的所有WaitEvent
	TAlignedArray<FWaitEvent>&     NodesArray;
	// 原子变量，正在销毁的标志位
	std::atomic<bool>              bIsShuttingDown{ false };
	// 广播代理
	FOversubscriptionLimitReached& OversubscriptionLimitReachedEvent;
}
```