1 从完整的 ZoneGraph 数据中，提取并缓存 entity 即将用到的车道片段，避免每次都访问大型全局数据结构。
2 
```cpp
void FMassZoneGraphCachedLaneFragment::CacheLaneData(
	const FZoneGraphStorage& ZoneGraphStorage, // ZoneGraph中的所有数据
	const FZoneGraphLaneHandle CurrentLaneHandle,// entity当前所在的lane
	const float CurrentDistanceAlongLane, // 
	const float TargetDistanceAlongLane, 
	const float InflateDistance)
{
	
}
```