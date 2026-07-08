1
```cpp
1 用FMassLaneCacheBoundaryFragment来标记此次计算edge的位置，就是标记以哪个位置为基准计算的Edge
2 FMassNavigationEdgesFragment用来缓存计算出的AvoidanceEdge
```

2 
```cpp
void UMassZoneGraphLaneCacheBoundaryProcessor::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	// 记录此次计算Edge的基准
	LaneCacheBoundary.LastUpdatePosition = MovementTarget.Center;
	LaneCacheBoundary.LastUpdateCacheID = CachedLane.CacheID;
	const float HalfWidth = 0.5f * CachedLane.LaneWidth.Get();

	// 确定要找多少个点，首先找到entity位于那个片段中，结合片段的前一个和后一个计算出di
	static const int32 MaxPoints = 4;
	FVector Points[MaxPoints];
	FVector SegmentDirections[MaxPoints];
	FVector SegmentNormals[MaxPoints];
	FVector MiterDirections[MaxPoints];
	const int32 CurrentSegment = 
		CachedLane.FindSegmentIndexAtDistance(LaneLocation.DistanceAlongLane);
	const int32 FirstSegment = FMath::Max(0, CurrentSegment - 1);
	const int32 LastSegment = 
		FMath::Min(CurrentSegment + 1, (int32)CachedLane.NumPoints - 2);
	const int32 NumPoints = (LastSegment - FirstSegment + 1) + 1;
}
```