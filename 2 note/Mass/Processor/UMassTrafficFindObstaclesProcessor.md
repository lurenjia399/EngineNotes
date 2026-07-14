1 
```cpp
1 障碍物的应该带有FMassTrafficObstacleTag的Tag，以障碍物为中心，构建一个查询Box，通过ZoneGraph的OverlapLane接口查询覆盖的lane都有哪些，通过遍历BVTree找到相交的叶子节点
2 遍历找到的Lane，
```
2 
```cpp
{
	// 1 通过BVTree来查询障碍物覆盖的Lanes
	TArray<FZoneGraphLaneHandle> NearbyLanes;
	FBox SearchBox = FBox::BuildAABB(
		TransformFragment.GetTransform().GetLocation(), 
		FVector(FVector2D(MassTrafficSettings->ObstacleSearchRadius),
		 MassTrafficSettings->ObstacleSearchHeight));
	{
		ZoneGraphSubsystem.FindOverlappingLanes(
			SearchBox, 
			GetDefault<UMassTrafficSettings>()->TrafficLaneFilter, 
			NearbyLanes);
	}
	// 2 遍历找到的Lane
	for (const FZoneGraphLaneHandle NearbyLane : NearbyLanes)
	{
		// 2.1 找到Lane上距离障碍物最近的点
		FZoneGraphLaneLocation NearestLocationOnLane;
		float DistanceSq;
		{
			ZoneGraphSubsystem.FindNearestLocationOnLane(
				NearbyLane, SearchBox, NearestLocationOnLane, DistanceSq);
		}
		if (NearestLocationOnLane.IsValid())
		{
			// 2.2 根据最近点找到Lane上在点前后的两辆车
			FMassEntityHandle PreviousVehicle;
			FMassEntityHandle NextVehicle;
			{
				UE::MassTraffic::FindNearestVehiclesInLane(
					EntityManager, 
					*NearbyTrafficLane, 
					NearestLocationOnLane.DistanceAlongLane, 
					PreviousVehicle, 
					NextVehicle);
			}
			if (PreviousVehicle.IsSet() && PreviousVehicle != ObstacleEntity)
			{
				
			}
		}
	}
}
```