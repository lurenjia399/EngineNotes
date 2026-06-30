1 从完整的 ZoneGraph 数据中，提取并缓存 entity 即将用到的车道片段，避免每次都访问大型全局数据结构。
2 
```cpp
void FMassZoneGraphCachedLaneFragment::CacheLaneData(
	const FZoneGraphStorage& ZoneGraphStorage, // ZoneGraph中的所有数据
	const FZoneGraphLaneHandle CurrentLaneHandle,// entity当前所在的lane
	const float CurrentDistanceAlongLane, // entity已经沿着lane行走了多少距离
	const float TargetDistanceAlongLane, // entity需要沿着lane行走多少距离到终点
	const float InflateDistance)//膨胀距离，0.2m
{
	// 当前lane数据
	const FZoneLaneData& Lane = ZoneGraphStorage.Lanes[CurrentLaneHandle.Index];
	// entity开始距离
	const float StartDistance = FMath::Min(
		CurrentDistanceAlongLane, TargetDistanceAlongLane);
	// entity结束距离
	const float EndDistance = FMath::Max(
		CurrentDistanceAlongLane, TargetDistanceAlongLane);
	// 拿到lane的长度
	const float CurrentLaneLength = 
		ZoneGraphStorage.LanePointProgressions[Lane.PointsEnd - 1];
	
	// 如果已经缓存过了，就提前返回
	if (LaneHandle == CurrentLaneHandle
		&& NumPoints > 0
		&& InflatedStartDistance >= LanePointProgressions[0].Get()
		&& InflatedEndDistance <= LanePointProgressions[NumPoints - 1].Get())
	{
		return;
	}
	// 没有缓存过，重置数据
	Reset();
	// 递增记录缓存
	CacheID++;
	LaneHandle = CurrentLaneHandle;
	LaneWidth = FMassInt16Real(Lane.Width);
	LaneLength = CurrentLaneLength;
	// 当前lane上点的数量
	const int32 LaneNumPoints = Lane.PointsEnd - Lane.PointsBegin;
	// 如果点数量 <= 5，说明可以直接缓存，直接把点的位置，切线，长度都缓存下来
	if (LaneNumPoints <= (int32)MaxPoints)
	{
		NumPoints = (uint8)LaneNumPoints;
		for (int32 Index = 0; Index < (int32)NumPoints; Index++)
		{
			LanePoints[Index] = 
				ZoneGraphStorage.LanePoints[Lane.PointsBegin + Index];
			LaneTangentVectors[Index] = FMassSnorm8Vector2D(
				FVector2D(
				ZoneGraphStorage.LaneTangentVectors[Lane.PointsBegin + Index]));
			LanePointProgressions[Index] = FMassInt16Real10(
				ZoneGraphStorage.LanePointProgressions
				[Lane.PointsBegin + Index]);
		}
	}
	else
	{
		// 根据当前位置，终点位置，找到
		int32 StartSegmentIndex = 0;
		int32 EndSegmentIndex = 0;
		UE::ZoneGraph::Query::CalculateLaneSegmentIndexAtDistance(
			ZoneGraphStorage, CurrentLaneHandle, StartDistance, StartSegmentIndex);
		UE::ZoneGraph::Query::CalculateLaneSegmentIndexAtDistance(
			ZoneGraphStorage, CurrentLaneHandle, EndDistance, EndSegmentIndex);

		// Expand if close to start of a segment start.
		if ((StartSegmentIndex - 1) >= Lane.PointsBegin && (StartDistance - InflateDistance) < ZoneGraphStorage.LanePointProgressions[StartSegmentIndex])
		{
			StartSegmentIndex--;
		}
		// Expand if close to end segment end.
		if ((EndSegmentIndex + 1) < (Lane.PointsEnd - 2) && (EndDistance + InflateDistance) > ZoneGraphStorage.LanePointProgressions[EndSegmentIndex + 1])
		{
			EndSegmentIndex++;
		}
	
		NumPoints = (uint8)FMath::Min((EndSegmentIndex - StartSegmentIndex) + 2, (int32)MaxPoints);

		for (int32 Index = 0; Index < (int32)NumPoints; Index++)
		{
			check((StartSegmentIndex + Index) >= Lane.PointsBegin && (StartSegmentIndex + Index) < Lane.PointsEnd);
			LanePoints[Index] = ZoneGraphStorage.LanePoints[StartSegmentIndex + Index];
			LaneTangentVectors[Index] = FMassSnorm8Vector2D(FVector2D(ZoneGraphStorage.LaneTangentVectors[StartSegmentIndex + Index]));
			LanePointProgressions[Index] = FMassInt16Real10(ZoneGraphStorage.LanePointProgressions[StartSegmentIndex + Index]);
		}
	}
}
```