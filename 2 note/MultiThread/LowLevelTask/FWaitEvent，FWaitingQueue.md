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
	1 [1, 14]位，StackMask表示等待栈栈顶节点在 `NodesArray` 中的索引
	2 [15, 28]位 WaiterMask表示当前有多少个线程PrepareWait
	3 [29, 42]位，SignalMask表示当前有多少唤醒信号
	4 [43, 64]位，Epoch表示State改变的次数每次改变都会增加，用于ABA问题
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
		int32 WaitCycles) // 自旋的时候短暂暂停的时间
	{
		// 自旋的循环等待，如果一直是没有信号唤醒的状态，就自旋等待
		for (int Spin = 0; Spin < SpinCycles; ++Spin)
		{
			if (Node->State.load(std::memory_order_relaxed) 
				== EWaitState::NotSignaled)
			{
				// 短暂暂停的时间
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
			/*
			1 直接赋值Signaled，然后拿旧值和EWaitState::Waiting比较
			2 如果相等，说明Event已经自旋和CAS都结束了，所以需要Trigger来唤醒Event
			3 如果不相等，旧值是NotSignaled。如果在自旋，新值Signaled就会退出自旋。如果在CAS操作中，新值Signaled也会退出CAS。
			*/
			if (Node->State.exchange(EWaitState::Signaled, 
				std::memory_order_relaxed) == EWaitState::Waiting)
			{
				Node->Event->Trigger();
			}
			else{}
		}
		return UnparkedCount;
	}
	// 设置准备等待状态。当前线程没有工作时，会首先执行，就是将计数器+1。表示即将进入等待的线程数量+1。
	void FWaitingQueue::PrepareWait(FWaitEvent* Node)
	{
		State.fetch_add(WaiterInc, std::memory_order_relaxed);
	}
	// 设置真正等待状态。当前线程没有工作时，在PrepareWait执行后的第二个循环里执行，
	bool FWaitingQueue::CommitWait(FWaitEvent* Node, FOutOfWork& OutOfWork, int32 SpinCycles, int32 WaitCycles)
	{
		{
			// 进入NotSignaled状态，就是自旋等待的状态
			Node->State.store(EWaitState::NotSignaled, std::memory_order_relaxed);
			// 如果唤醒信号不为0，说明被唤醒了。就需要减少等待计数，减少新号计数，增加状态修改次数。更新新状态
			if ((LocalState & SignalMask) != 0)
			{
				NewState = LocalState - WaiterInc - SignalInc + EpochInc;
			}
			// 如果唤醒新号为0，说明依然没有唤醒。
			else
			{
				NewState = 
					// 只保留准备等待计数和State改变次数
					((LocalState & (WaiterMask | EpochMask))
					// 减少等待计数，因为要进入真正等待了
					- WaiterInc 
					// 增加State改变次数，因为改变了就要增加
					+ EpochInc) 
					// 添加Node在NodesArray中的索引，表示把Node放到等待队列的第一个
				    | (Node - &NodesArray[0]);
				// 把等待队列的第一个挪到Node的下一个
				Node->Next.store(LocalState & StackMask, 
					std::memory_order_relaxed);
			}
			// 比较交换的操作，如果比较没有改变，就会设置新状态
			if (State.compare_exchange_weak(LocalState, NewState, 
				std::memory_order_acq_rel, std::memory_order_relaxed))
			{
				// 新状态后，如果依然没有唤醒信号，就执行Park了
				if ((LocalState & SignalMask) == 0)
				{}
				// 新状态后，如果有唤醒信号，就不用Park了，直接返回
				else
				{
					return true;
				}
			}
		}
		// 阻塞等待，里面会自旋一下，然后真正开始阻塞
		Park(Node, OutOfWork, SpinCycles, WaitCycles);
		return true;
	}
	
	// 和Commit基本一致，对称着看
	bool FWaitingQueue::CancelWait(FWaitEvent* Node)
	{
		...
	}
	// 
	int32 FWaitingQueue::NotifyInternal(int32 Count)
	{
		
	}
}
```

