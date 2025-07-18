```cpp
class TMapBase
{
	typedef TSet<ElementType, KeyFuncs, SetAllocator> ElementSetType;
	// TMap中只有这一个成员变量，是TSet类型
	ElementSetType Pairs;
};

class TSet
{
	using ElementArrayType = TSparseArray<SetElementType, typename Allocator::SparseArrayAllocator>;
	ElementArrayType Elements;//TSet中的元素，是稀疏数组

	using HashType = typename Allocator::HashAllocator::template ForElementType<FSetElementId>;
	mutable HashType Hash;//TSet中Hash存储区，内存连续
	
	mutable int32	 HashSize;// 数量吧
};

class TSparseArray//稀疏数组和数组的区别就是元素可以不连续
{
	typedef TArray<FElementOrFreeListLink,typename Allocator::ElementAllocator> DataType;
	DataType Data;//稀疏数组中的数据，TArray

	typedef TBitArray<typename Allocator::BitArrayAllocator> AllocationBitArrayType;
	AllocationBitArrayType AllocationFlags;// TArray中是否有元素，

	int32 FirstFreeIndex;// 空闲的元素索引

	int32 NumFreeIndices;// 空闲的元素数量
};
```
1 tmap内部只有一个成员变量tset，tset的实现是通过hash表的形式现实的，有一个hash存储区和数据存储区，两者内存都是连续的，hash存储区存储hash值对应的数据存储区索引，而数据存储区是稀疏数组。
2 稀疏数组是内存连续的，但是不是每个元素都是有值的，这也是和tarray的区别。稀疏数组中的数据也是tarray存放的，还需要有一个数组来标记元素是否有值。

# TArray

https://www.unrealengine.com/zh-CN/blog/optimizing-tarray-usage-for-performance