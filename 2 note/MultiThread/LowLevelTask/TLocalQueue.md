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

# 总结
1 FLocalQueueType 作为任务队列，其中有头指针和尾指针，头指针支持本地线程添加移除任务，而尾指针支持其他线程偷取任务。
2 队列没有扩容机制，不会重新分配内存是定长数组，如果数组满了还要添加元素，就会放在Registry全局管理的溢出队列上，溢出队列是链表的形式实现的。
3 因为有任务的优先级的，所以LocalQueue是一个二维数组，每个优先级都有FLocalQueueType。其中溢出队列也是支持优先级的。
4 每个线程都会有自己的LocalQueue，每个LocalQueue会按照每个优先级都创建一个任务队列。LocalQueueRegistry也会按照每个优先级都创建一个溢出队列。