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
	// 根据插值T，计算沿着道路移动的距离
	LaneLocation.DistanceAlongLane = 
		FMath::Min(FMath::Lerp(CurrPoint.DistanceAlongLane.Get(), 
			NextPoint.DistanceAlongLane.Get(), T), LaneLocation.LaneLength);
	// 根据插值T，赋值MoveTarget一些数据
	MoveTarget.Center = FMath::Lerp(CurrPoint.Position, NextPoint.Position, T);
	MoveTarget.Forward = FMath::Lerp(CurrPoint.Tangent.GetVector(), 
		NextPoint.Tangent.GetVector(), T).GetSafeNormal();
	MoveTarget.DistanceToGoal = ShortPath.Points[LastPointIndex].Distance.Get() 
		- FMath::Lerp(CurrPoint.Distance.Get(), NextPoint.Distance.Get(), T);
	MoveTarget.bOffBoundaries = CurrPoint.bOffLane || NextPoint.bOffLane;
}
else
{
	// 计算当前沿着道路移动的距离
	LaneLocation.DistanceAlongLane = 
		FMath::Min(ShortPath.Points[LastPointIndex].DistanceAlongLane.Get(), 
			LaneLocation.LaneLength);
	// 赋值MoveTarget一些数据，中心点取最后点的位置，朝向取最后点的切线，距离目标距离为0，是否移动到lane之外的点
	MoveTarget.Center = ShortPath.Points[LastPointIndex].Position;
	MoveTarget.Forward = ShortPath.Points[LastPointIndex].Tangent.GetVector();
	MoveTarget.DistanceToGoal = 0.0f;
	MoveTarget.bOffBoundaries = ShortPath.Points[LastPointIndex].bOffLane;

	// 一条道路走完了，如果还需要走下一条道路
	if (ShortPath.NextLaneHandle.IsValid())
	{
		const FZoneGraphStorage* ZoneGraphStorage = 
				ZoneGraphSubsystem.GetZoneGraphStorage(
				LaneLocation.LaneHandle.DataHandle);
		if (ZoneGraphStorage != nullptr)
		{
			// r
			if (ShortPath.NextExitLinkType == EZoneLaneLinkType::Outgoing)
			{
				float NewLaneLength = 0.f;
				UE::ZoneGraph::Query::GetLaneLength(*ZoneGraphStorage, ShortPath.NextLaneHandle, NewLaneLength);

				UE_CVLOG(bDisplayDebug, LogOwner, LogMassNavigation, Log, TEXT("Entity [%s] Switching to OUTGOING lane %s -> %s, new distance %f."),
					*Entity.DebugGetDescription(), *LaneLocation.LaneHandle.ToString(), *ShortPath.NextLaneHandle.ToString(), 0.f);

				// update lane location
				LaneLocation.LaneHandle = ShortPath.NextLaneHandle;
				LaneLocation.LaneLength = NewLaneLength;
				LaneLocation.DistanceAlongLane = 0.0f;
			}
			else if (ShortPath.NextExitLinkType == EZoneLaneLinkType::Incoming)
			{
				float NewLaneLength = 0.f;
				UE::ZoneGraph::Query::GetLaneLength(*ZoneGraphStorage, ShortPath.NextLaneHandle, NewLaneLength);

				UE_CVLOG(bDisplayDebug, LogOwner, LogMassNavigation, Log, TEXT("Entity [%s] Switching to INCOMING lane %s -> %s, new distance %f."),
					*Entity.DebugGetDescription(), *LaneLocation.LaneHandle.ToString(), *ShortPath.NextLaneHandle.ToString(), NewLaneLength);

				// update lane location
				LaneLocation.LaneHandle = ShortPath.NextLaneHandle;
				LaneLocation.LaneLength = NewLaneLength;
				LaneLocation.DistanceAlongLane = NewLaneLength;
			}
			else if (ShortPath.NextExitLinkType == EZoneLaneLinkType::Adjacent)
			{
				FZoneGraphLaneLocation NewLocation;
				float DistanceSqr;
				if (UE::ZoneGraph::Query::FindNearestLocationOnLane(*ZoneGraphStorage, ShortPath.NextLaneHandle, MoveTarget.Center, MAX_flt, NewLocation, DistanceSqr))
				{
					float NewLaneLength = 0.f;
					UE::ZoneGraph::Query::GetLaneLength(*ZoneGraphStorage, ShortPath.NextLaneHandle, NewLaneLength);

					UE_CVLOG(bDisplayDebug, LogOwner, LogMassNavigation, Log, TEXT("Entity [%s] Switching to ADJACENT lane %s -> %s, new distance %f."),
						*Entity.DebugGetDescription(), *LaneLocation.LaneHandle.ToString(), *ShortPath.NextLaneHandle.ToString(), NewLocation.DistanceAlongLane);

					// update lane location
					LaneLocation.LaneHandle = ShortPath.NextLaneHandle;
					LaneLocation.LaneLength = NewLaneLength;
					LaneLocation.DistanceAlongLane = NewLocation.DistanceAlongLane;

					MoveTarget.Forward = NewLocation.Tangent;
				}
				else
				{
					UE_CVLOG(bDisplayDebug, LogOwner, LogMassNavigation, Error, TEXT("Entity [%s] Failed to switch to ADJACENT lane %s -> %s."),
						*Entity.DebugGetDescription(), *LaneLocation.LaneHandle.ToString(), *ShortPath.NextLaneHandle.ToString());
				}
			}
			else
			{
				ensureMsgf(false, TEXT("Unhandle NextExitLinkType type %s"), *UEnum::GetValueAsString(ShortPath.NextExitLinkType));
			}

			// Signal lane changed.
			EntitiesToSignalLaneChanged.Add(Entity);
		}
		else
		{
			UE_CVLOG(bDisplayDebug, LogOwner, LogMassNavigation, Error, TEXT("Entity [%s] Could not find ZoneGraph storage for lane %s."),
				*Entity.DebugGetDescription(), *LaneLocation.LaneHandle.ToString());
		}
	}
	ShortPath.bDone = true;
```