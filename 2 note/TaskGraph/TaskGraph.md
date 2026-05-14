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
}
```