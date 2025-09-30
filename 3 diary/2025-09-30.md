# UHTSilentCheckSubsystem
```cpp
struct HTGAME_API FInternalSilentCheckOctreeSemantics
{
	enum { MaxElementsPerLeaf = 16 };
	enum { MinInclusiveElementsPerNode = 7 };
	enum { MaxNodeDepth = 12 };

	typedef TInlineAllocator<MaxElementsPerLeaf> ElementAllocator;

	FORCEINLINE static FBoxSphereBounds GetBoundingBox(USilentElement OctreeEle);
	FORCEINLINE static void SetElementId(USilentElement OctreeEle, FOctreeElementId2 Id);
	FORCEINLINE static uint32 GetElementMask(USilentElement OctreeEle);
};
typedef TOctree2<USilentElement, FInternalSilentCheckOctreeSemantics> FSilentCheckComponentOctreeType;
```

1 RETURN_QUICK_DECLARE_CYCLE_STAT 这个东西如何使用