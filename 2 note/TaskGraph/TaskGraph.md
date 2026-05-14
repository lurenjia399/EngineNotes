# 基本概念
1 CAS
2 ABA
3 TLS


# 2 TLockFreeAllocOnceIndexedAllocator
```cpp

template<class T, unsigned int MaxTotalItems, unsigned int ItemsPerPage>
class TLockFreeAllocOnceIndexedAllocator
{
	// 计算Block的数量
	enum
	{
		MaxBlocks = (MaxTotalItems + ItemsPerPage - 1) / ItemsPerPage
	};
public:
	// 
	/*
		构造函数
		索引从1开始，保留0作为空指针标记
		所有页面初始化为 nullptr
	*/
	TLockFreeAllocOnceIndexedAllocator()
	{
		NextIndex.Increment(); // skip the null ptr
		for (uint32 Index = 0; Index < MaxBlocks; Index++)
		{
			Pages[Index] = nullptr;
		}
	}
	/*
		- 使用原子递增获取索引范围，无需加锁
		- 对每个新分配的位置调用placement new构造对象
		- 如果超出容量则调用 LockFreeLinksExhausted 报错
	*/
	FORCEINLINE uint32 Alloc(uint32 Count = 1)
	{
		uint32 FirstItem = NextIndex.Add(Count);
		if (FirstItem + Count > MaxTotalItems)
		{
			LockFreeLinksExhausted(MaxTotalItems);
		}
		for (uint32 CurrentItem = FirstItem; 
			CurrentItem < FirstItem + Count; CurrentItem++)
		{
			::new (GetRawItem(CurrentItem)) T();
		}
		return FirstItem;
	}
	/*
		- 通过除法和取模计算页面索引和页内偏移
		- 索引0返回 nullptr（空指针语义）
	*/
	FORCEINLINE T* GetItem(uint32 Index)
	{
		if (!Index)
		{
			return nullptr;
		}
		uint32 BlockIndex = Index / ItemsPerPage;
		uint32 SubIndex = Index % ItemsPerPage;
		checkLockFreePointerList(Index < (uint32)NextIndex.GetValue() && Index < MaxTotalItems && BlockIndex < MaxBlocks && Pages[BlockIndex]);
		return Pages[BlockIndex] + SubIndex;
	}
	/*
		- 双重检查：先检查页面是否存在
		- CAS操作：使用 InterlockedCompareExchangePointer
		原子地设置页面指针
		- 竞争处理：如果多个线程同时分配同一页面，只有一个成功，其他线程释
		  放多余的内存

	*/
	void* GetRawItem(uint32 Index)
	{
		uint32 BlockIndex = Index / ItemsPerPage;
		uint32 SubIndex = Index % ItemsPerPage;
		checkLockFreePointerList(Index && Index < (uint32)NextIndex.GetValue() && Index < MaxTotalItems && BlockIndex < MaxBlocks);
		if (!Pages[BlockIndex])
		{
			T* NewBlock = (T*)LockFreeAllocLinks(ItemsPerPage * sizeof(T));
			checkLockFreePointerList(IsAligned(NewBlock, alignof(T)));
			if (FPlatformAtomics::InterlockedCompareExchangePointer(
				(void**)&Pages[BlockIndex], NewBlock, nullptr) != nullptr)
			{
				// we lost discard block
				checkLockFreePointerList(Pages[BlockIndex] 
				&& Pages[BlockIndex] != NewBlock);
				LockFreeFreeLinks(ItemsPerPage * sizeof(T), NewBlock);
			}
			else
			{
				checkLockFreePointerList(Pages[BlockIndex]);
			}
		}
		return (void*)(Pages[BlockIndex] + SubIndex);
	}
	
	// 缓存行对齐避免伪共享（false sharing），提升多核性能。
	alignas(PLATFORM_CACHE_LINE_SIZE) FThreadSafeCounter NextIndex;
	alignas(PLATFORM_CACHE_LINE_SIZE) T* Pages[MaxBlocks];
}
```
# 3 LockFreeLinkAllocator_TLSCache
```cpp
// 全局单例，唯一一个回收池
static LockFreeLinkAllocator_TLSCache& GetLockFreeAllocator()
{
	// make memory that will not go away, a replacement for TLazySingleton, which will still get destructed
	alignas(LockFreeLinkAllocator_TLSCache) static unsigned char Data[sizeof(LockFreeLinkAllocator_TLSCache)];
	static bool bIsInitialized = false;
	if (!bIsInitialized)
	{
		::new((void*)Data)LockFreeLinkAllocator_TLSCache();
		bIsInitialized = true;
	}
	return *(LockFreeLinkAllocator_TLSCache*)Data;
}

```
这个LockFreeLickAllocator_TLSCache就是节点的回收池，本身定义了一个全局变量。容器在需要使用节点的时候，就会从这个全局变量上申请，而用完节点后会归还给这个回收池。回收池内部在开始的时候，本身也没有节点，就会向前面提到的TLockFreeAllocOnceIndexedAllocator去申请，这个Allocator前面也说了是一次分配一个Block，只增不减，而且内存连续的，所以效率很高。可能你也从回收池名字上看到了TLSCache，内部实现确实是使用TLS来管理节点的，每个线程在申请和归还节点时，都是从自己线程独有的节点包里取。内部向Allocator也是一次申请64个，这样同一线程内分配的节点，就有很大概率是连续的内存，在业务使用LockFreeList时，性能肯定也会很好。而且因为本身是TLS的，每个线程独占，所以在分配一块节点Cache时，也不用考虑多线程问题，不需要使用CAS以及回滚操作的写法