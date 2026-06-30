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
}
```