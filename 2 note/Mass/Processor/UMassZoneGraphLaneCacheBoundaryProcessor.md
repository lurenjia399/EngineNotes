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

	// 确定要找多少个点，首先找到entity位于那个片段中，结合片段的前一个和后一个计算出点的数量
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
	
	// 从cacheLane中拿到点的位置
	for (int32 Index = 0; Index < NumPoints; Index++)
	{
		Points[Index] = CachedLane.LanePoints[Index];
	}
	// 根据点的位置，计算出片段的向量和左向量
	for (int32 Index = 0; Index < NumPoints - 1; Index++)
	{
		SegmentDirections[Index] = (Points[Index + 1] - Points[Index]).GetSafeNormal();
		SegmentNormals[Index] = UE::MassNavigation::GetLeftDirection(
			SegmentDirections[Index], FVector::UpVector);
	}
	SegmentDirections[NumPoints - 1] = SegmentDirections[NumPoints - 2];
	SegmentNormals[NumPoints - 1] = SegmentNormals[NumPoints - 2];
	
	// 计算出中间向量，如果是拐角na'zhong
	MiterDirections[0] = SegmentNormals[0];
	MiterDirections[NumPoints - 1] = SegmentNormals[NumPoints - 1];
	for (int32 Index = 1; Index < NumPoints - 1; Index++)
	{
		MiterDirections[Index] = 
			UE::MassNavigation::ComputeMiterNormal(
			SegmentNormals[Index - 1], SegmentNormals[Index]);
	}
}
```