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
}
```