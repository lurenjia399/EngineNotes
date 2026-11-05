# FScheduler

```cpp
class FScheduler final : public FSchedulerTls
{
	Private::FWaitingQueue WaitingQueue[2] = { 
		{ WorkerEvents, OversubscriptionLimitReachedEvent }, 
		{ WorkerEvents, OversubscriptionLimitReachedEvent } };
	FSchedulerTls::FQueueRegistry QueueRegistry;
	UE::FPlatformRecursiveMutex WorkerCreationCS;
	UE::FPlatformRecursiveMutex WorkerThreadsCS;
	TArray<std::atomic<FThread*>> WorkerThreads;
	TAlignedArray<FSchedulerTls::FLocalQueueType> WorkerLocalQueues;
	TAlignedArray<Private::FWaitEvent> WorkerEvents;
	TUniquePtr<FSchedulerTls::FLocalQueueType> GameThreadLocalQueue;
	std::atomic_uint ActiveWorkers { 0 };
	std::atomic_uint NextWorkerId { 0 };
	std::atomic<int32> ForegroundCreationIndex{ 0 };
	std::atomic<int32> BackgroundCreationIndex{ 0 };
	uint64 WorkerAffinity = 0;
	uint64 BackgroundAffinity = 0;
	// 前台Worker的线程优先级，前台Worker和后台Worker都是工作线程，只有两个线程是前台Worker。区分开的目的就是防止所有的Worker都被后台任务占用。
	EThreadPriority WorkerPriority = EThreadPriority::TPri_Normal;
	// 后台Worker的线程优先级
	EThreadPriority BackgroundPriority = EThreadPriority::TPri_BelowNormal;
	std::atomic_bool TemporaryShutdown{ false };
	FOversubscriptionLimitReached OversubscriptionLimitReachedEvent;
}
```

# TLocalQueueRegistry

```cpp
class TLocalQueueRegistry
{
private:
	/*
	1 溢出队列，是以链表的形式实现的
	2 每个LocalQueue是定长数组，如果LocalQueue满了，依然要存新的就会放在溢出队列上
	*/
	FOverflowQueueType        OverflowQueues[uint32(ETaskPriority::Count)];
	// MaxLocalQueues定长的数组，每个元素都是原子变量，是LocalQueue的指针
	std::atomic<TLocalQueue*> LocalQueues[MaxLocalQueues] { nullptr };
	// 原子变量，用来表示LocalQueue的数量
	std::atomic<uint32>       NumLocalQueues {0};
	
	// 线程安全的往LocalQueues数组中添加元素的方法
	void AddLocalQueue(TLocalQueue* QueueToAdd)
	{
		// 原子操作的增加计数。relaxed顺序表示只关心原子操作本身，不承载其他目的
		uint32 Index = NumLocalQueues.fetch_add(1, std::memory_order_relaxed);
		// 往LocalQueues数组中添加元素。release顺序表示发布，如果其他线程通过acquire捕获了发布，则发布前的操作都能读取
		LocalQueues[Index].store(QueueToAdd, std::memory_order_release);
	}
	
	
}
```
TLocalQueueRegistry 类似于一个全局的任务管理，里面会有一个LocalQueue的数组，这个数组用来存储所有的LocalQueue。

# TLocalQueue
```cpp
class TLocalQueue
{
	/*
	1 二维数组的任务队列。每个任务优先级都对应一个TWorkStealingQueueBase2
	2 TWorkStealingQueueBase2是由头索引，尾索引（原子变量）和一个数组（原子变量）组成
	3 TWorkStealingQueueBase2会有三个方法，Put,Get,Steal
	4 Put方法就是添加的方法，往数组里添加一个新的元素，然后头索引往后移动。不是线程安全的方法，因为移动的是头索引，不是原子变量。
	5 Get方法就是获取元素的方法，头索引往前移动。
	6 Steal方法也是获取元素的方法，但是它只获取尾索引的元素。这是线程安全的方法，通过acquire来获取尾指针指向的元素。因为和Get方法可能获取同一个元素，这里用CAS的方式来处理，用到了memory_order_acq_rel顺序操作，读取时acquire，写入时release。
	*/
	FLocalQueueType            LocalQueues[uint32(ETaskPriority::Count)];
	/*
	1 危险指针的队列。每个任务优先级都有对应一个危险指针。在初始化LocalQueue的时候把Registry全局管理上的溢出队列上的危险指针赋值。
	2 作用类似于引用计数的作用。
	3 线程A在读取Registry上的溢出队列的元素p，但是线程B移除了队列上的元素p。为了在销毁的时候不真正的删除p，在移除时会放到FHazardPointerCollection的回收队列里，等没有使用的时候在进行回收。
	*/
	DequeueHazard              DequeueHazards[uint32(ETaskPriority::Count)];
	// 这个就是LocalQueueRegistry全局管理，每个LocalQueue都会引用
	TLocalQueueRegistry*       Registry;
	// 在偷取Registry任务的时候，随机偷取的是哪个LocalQueue上的任务
	uint32                     CachedRandomIndex = InvalidIndex;
	// 在偷取Registry任务的时候，偷取的是哪个优先级的任务
	uint32                     CachedPriorityIndex = 0;
	// 前台线程还是后台线程。前台和后台只是名称不同，前台线程的任务优先级高于后台线程，两者都是工作线程
	ELocalQueueType            QueueType;
	
	// 构造函数，参数就是Registry全局管理和Type（前台还是后台）
	TLocalQueue(TLocalQueueRegistry& InRegistry, ELocalQueueType InQueueType) 
		: Registry(&InRegistry), QueueType(InQueueType)
	{
		// 把自己添加到Registry全局管理中
		Registry->AddLocalQueue(this);
		
		// 遍历任务优先级，将Registry全局管理中的溢出队列上的危险指针，存到自己身上
		for (int32 PriorityIndex = 0; PriorityIndex < int32(ETaskPriority::Count);
			 ++PriorityIndex)
		{
			DequeueHazards[PriorityIndex] = 
				Registry->OverflowQueues[PriorityIndex].getHeadHazard();
		}
	}
	// 添加任务的方法，如果LocalQueue满了，就会放到Registry全局管理的溢出队列上。不是线程安全的只能自己线程添加
	void Enqueue(FTask* Item, uint32 PriorityIndex)
	{
		if (!LocalQueues[PriorityIndex].Put(Item))
		{
			Registry->OverflowQueues[PriorityIndex].enqueue(Item);
		}
	}
	// 取出任务的方法，如果LocalQueue满了，就会从Registry全局管理的溢出队列上取。不是线程安全的
	FTask* Dequeue(bool GetBackGroundTasks)
	{
		for (int32 PriorityIndex = 0; PriorityIndex < MaxPriority; ++PriorityIndex)
		{
			FTask* Item;
			if (LocalQueues[PriorityIndex].Get(Item))
			{
				return Item;
			}

			Item = Registry->OverflowQueues[PriorityIndex].
				dequeue(DequeueHazards[PriorityIndex]);
			if (Item)
			{
				return Item;
			}
		}
		return nullptr;
	}
	// 偷取任务，线程安全的。会按优先级顺序偷，直到偷到为止，直接调用TWorkStealingQueueBase2的Steal方法。
	FTask* StealLocal(bool GetBackGroundTasks)
	{
		for (int32 PriorityIndex = 0; PriorityIndex < MaxPriority; ++PriorityIndex)
		{
			FTask* Item;
			if (LocalQueues[PriorityIndex].Steal(Item))
			{
				return Item;
			}
		}
		return nullptr;
	}
}
```