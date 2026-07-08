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
	
	// 计算出中间向量，目的是去掉拐角
	MiterDirections[0] = SegmentNormals[0];
	MiterDirections[NumPoints - 1] = SegmentNormals[NumPoints - 1];
	for (int32 Index = 1; Index < NumPoints - 1; Index++)
	{
		MiterDirections[Index] = 
			UE::MassNavigation::ComputeMiterNormal(
			SegmentNormals[Index - 1], SegmentNormals[Index]);
	}
	// 计算出道路的左边界和有边界
	const float LeftWidth = HalfWidth + CachedLane.LaneLeftSpace.Get();
	const float RightWidth = HalfWidth + CachedLane.LaneRightSpace.Get();
	FVector LeftPositions[MaxPoints];
	FVector RightPositions[MaxPoints];
	for (int32 Index = 0; Index < NumPoints; Index++)
	{
		const FVector MiterDir = MiterDirections[Index];
		LeftPositions[Index] = Points[Index] + LeftWidth * MiterDir;
		RightPositions[Index] = Points[Index] - RightWidth * MiterDir;
	}
	// 处理4点3段的情况，如果有两端之间有交叉，则删掉一个点
	if (NumPoints == 4)
	{
		FVector Intersection = FVector::ZeroVector;
		if (FMath::SegmentIntersection2D(
			LeftPositions[0], LeftPositions[1], 
			LeftPositions[2], LeftPositions[3], Intersection))
		{
			LeftPositions[1] = Intersection;
			LeftPositions[2] = LeftPositions[3];
			NumLeftPositions--;
		}

		Intersection = FVector::ZeroVector;
		if (FMath::SegmentIntersection2D(
			RightPositions[0], RightPositions[1], 
			RightPositions[2], RightPositions[3], Intersection))
		{
			RightPositions[1] = Intersection;
			RightPositions[2] = RightPositions[3];
			NumRightPositions--;
		}
	}
}
```