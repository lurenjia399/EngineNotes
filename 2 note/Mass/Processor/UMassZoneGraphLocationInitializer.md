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
1 就是通过FindNearestLane接口，将自己手中的QueryBound
```