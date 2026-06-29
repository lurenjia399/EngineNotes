# 1 UMassZoneGraphLocationInitializer
1 是ObserverProcessor
```cpp
ObservedType = FMassZoneGraphLaneLocationFragment::StaticStruct();
Operation = EMassObservedOperation::Add;
```
2 具体查找
```cpp
if (ZoneGraphSubsystem.FindNearestLane(QueryBounds, NavigationParams.LaneFilter, NearestLane, NearestLaneDistSqr))
{
	const FZoneGraphStorage* ZoneGraphStorage = ZoneGraphSubsystem.GetZoneGraphStorage(NearestLane.LaneHandle.DataHandle);
	check(ZoneGraphStorage); // Assume valid storage since we just got result.

	LaneLocation.LaneHandle = NearestLane.LaneHandle;
	LaneLocation.DistanceAlongLane = NearestLane.DistanceAlongLane;
	UE::ZoneGraph::Query::GetLaneLength(*ZoneGraphStorage, LaneLocation.LaneHandle, LaneLocation.LaneLength);
	
	MoveTarget.Center = AgentLocation;
	MoveTarget.Forward = NearestLane.Tangent;
}
1 就是通过FindNearestLane接口，将自己手中的QueryBound，根据BVTree查找相交的叶子节点，找到后遍历Zone中的Lanes，找到里QueryCenter最近的Lane，然后返回数据
```

# 2 UMassZoneGraphPathFollowProcessor
1 
```cpp
1 UMassSteeringTrait 需要添加这个Trait，来添加FMassMoveTargetFragment这个
2 UMassZoneGraphNavigationTrait 需要添加这个Trait，来添加FMassZoneGraphShortPathFragment这个和FMassZoneGraphLaneLocationFragment
```
2 
```cpp
// 遍历查询到的entity，FMassZoneGraphShortPathFragment用来记录entity行走的相关数据

// ProgressDistance代表走过的距离，<=0就是刚开始走
if (ShortPath.ProgressDistance <= 0.0f)
{
	// 
	LaneLocation.DistanceAlongLane = ShortPath.Points[0].DistanceAlongLane.Get();
	
	MoveTarget.Center = ShortPath.Points[0].Position;
	MoveTarget.Forward = ShortPath.Points[0].Tangent.GetVector();
	MoveTarget.DistanceToGoal = ShortPath.Points[LastPointIndex].Distance.Get();
	MoveTarget.bOffBoundaries = ShortPath.Points[0].bOffLane;
}
// 没有走完这段路程
else if (ShortPath.ProgressDistance <= 
		ShortPath.Points[LastPointIndex].Distance.Get())
{
	// 计算出当前移动的距离在两个点之间的插值比例T
	const FMassZoneGraphPathPoint& CurrPoint = ShortPath.Points[PointIndex];
	const FMassZoneGraphPathPoint& NextPoint = ShortPath.Points[PointIndex + 1];
	const float T = (ShortPath.ProgressDistance - CurrPoint.Distance.Get()) 
					/ (NextPoint.Distance.Get() - CurrPoint.Distance.Get());
	// 根据插值T，将沿着Lane移动的距离cha'zh
	LaneLocation.DistanceAlongLane = FMath::Min(FMath::Lerp(CurrPoint.DistanceAlongLane.Get(), NextPoint.DistanceAlongLane.Get(), T), LaneLocation.LaneLength);

	MoveTarget.Center = FMath::Lerp(CurrPoint.Position, NextPoint.Position, T);
	MoveTarget.Forward = FMath::Lerp(CurrPoint.Tangent.GetVector(), NextPoint.Tangent.GetVector(), T).GetSafeNormal();
	MoveTarget.DistanceToGoal = ShortPath.Points[LastPointIndex].Distance.Get() - FMath::Lerp(CurrPoint.Distance.Get(), NextPoint.Distance.Get(), T);
	MoveTarget.bOffBoundaries = CurrPoint.bOffLane || NextPoint.bOffLane;

	UE_CVLOG(bDisplayDebug, LogOwner, LogMassNavigation, Verbose, TEXT("Entity [%s] along lane %s at distance %.1f. Distance to goal: %.1f. Off Boundaries: %s"),
		*Entity.DebugGetDescription(),
		*LaneLocation.LaneHandle.ToString(),
		LaneLocation.DistanceAlongLane,
		MoveTarget.DistanceToGoal,
		*LexToString((bool)MoveTarget.bOffBoundaries));
}
```