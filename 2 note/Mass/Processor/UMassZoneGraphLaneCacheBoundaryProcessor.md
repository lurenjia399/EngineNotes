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
	
}
```