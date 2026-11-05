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
	// 添加任务，只能在本地线程添加，如果满了加不了了，就会添加到溢出队列上
	void Enqueue(FTask* Item, uint32 PriorityIndex)
	{
		if (!LocalQueues[PriorityIndex].Put(Item))
		{
			Registry->OverflowQueues[PriorityIndex].enqueue(Item);
		}
	}
	// 移除任务，只能在本地线程移除，如果本来就没东西了，就会从溢出队列上找一个移除
	FTask* Dequeue(bool GetBackGroundTasks)
	{
		for (int32 PriorityIndex = 0; PriorityIndex < MaxPriority; ++PriorityIndex)
		{
			FTask* Item;
			if (LocalQueues[PriorityIndex].Get(Item))
			{
				return Item;
			}
			Item = Registry->OverflowQueues[PriorityIndex].dequeue(
				DequeueHazards[PriorityIndex]);
			if (Item)
			{
				return Item;
			}
		}
		return nullptr;
	}
	// 偷取任务，线程安全的，可以由其他线程偷取本地线程上的任务
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
	// 通过LocalQueueRegistry偷取任务，可以任意线程间偷取
	FTask* DequeueSteal(bool GetBackGroundTasks)
	{
		if (CachedRandomIndex == InvalidIndex)
		{
			CachedRandomIndex = Rand();
		}
		FTask* Result = Registry->StealItem(CachedRandomIndex, CachedPriorityIndex, GetBackGroundTasks);
		if (Result)
		{
			return Result;
		}
		return nullptr;
	}
}
```
TLocalQueueRegistry 类似于一个全局的任务管理，里面会有一个LocalQueue的数组，这个数组用来存储所有的LocalQueue。