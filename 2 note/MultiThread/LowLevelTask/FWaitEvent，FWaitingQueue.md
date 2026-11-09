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
	
	// 阻塞等待的方法
	void FWaitingQueue::Park(
		FWaitEvent* Node, // 等待Event
		FOutOfWork& OutOfWork, // 
		int32 SpinCycles, // 自旋的次数
		int32 WaitCycles) // 每次自旋的时间
	{
		// 自旋的循环等待，如果一直是没有信号唤醒的状态，就自旋等待
		for (int Spin = 0; Spin < SpinCycles; ++Spin)
		{
			if (Node->State.load(std::memory_order_relaxed) 
				== EWaitState::NotSignaled)
			{
				FPlatformProcess::YieldCycles(WaitCycles);
			}
			else
			{
				return;
			}
		}
		
		Node->Event->Reset();
		EWaitState Target = EWaitState::NotSignaled;
		
		// 在自旋等待结束后，通过CAS的操作把自己的状态改成Waiting，真正的等待状态
		if (Node->State.compare_exchange_strong(Target, EWaitState::Waiting, 
			std::memory_order_relaxed, std::memory_order_relaxed))
		{
		}
		else
		{
			return;
		}
		// 实际让事件等待的方法，就是阻塞线程
		EnterWait(Node);
	}
	// 解除阻塞等待
	int32 FWaitingQueue::Unpark(FWaitEvent* Node)
	{
		int32 UnparkedCount = 0;
		for (FWaitEvent* Next; Node; Node = Next)
		{
			// 找WaitEvent链表的下一个Event
			uint64 NextNode = Node->Next.load(std::memory_order_relaxed) 
				& StackMask;
			Next = NextNode == StackMask ? nullptr : &NodesArray[(int)NextNode];
			
			// 解除阻塞的计数
			UnparkedCount++;
			
			// 交换
			if (Node->State.exchange(EWaitState::Signaled, 
				std::memory_order_relaxed) == EWaitState::Waiting)
			{
				// Always trace this one since signaling cost can be very expensive.
				WAITINGQUEUE_EVENT_SCOPE_ALWAYS(FWaitingQueue_Unpark_SignalWaitingThread);
				Node->Event->Trigger();
			}
			else
			{
				WAITINGQUEUE_EVENT_SCOPE(FWaitingQueue_Unpark_SignaledSpinningThread);
			}
		}

		return UnparkedCount;
	}
}
```

