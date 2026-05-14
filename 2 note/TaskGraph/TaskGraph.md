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
}
```