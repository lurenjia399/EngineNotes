```cpp
void UMassMovingAvoidanceProcessor::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	// 找到entity周围4米的其他entity
	const FNavigationObstacleHashGrid2D& AvoidanceObstacleGrid = 
		NavigationSubsystem->GetObstacleGridMutable();
	UE::MassAvoidance::FindCloseObstacles(
		AgentLocation, 
		MovingAvoidanceParams.ObstacleDetectionDistance,
		AvoidanceObstacleGrid, CloseEntities, 
		UE::MassAvoidance::MaxObstacleResults);
	// 遍历找到的CloseEntity
	for (const FNavigationObstacleHashGrid2D::ItemIDType OtherEntity 
		: CloseEntities)
	{
		// 被认为是障碍物，添加到障碍物数组中
		FSortedObstacle Obstacle;
		Obstacle.LocationCached = OtherLocation;
		Obstacle.Forward = Transform.GetRotation().GetForwardVector();
		Obstacle.ObstacleItem = OtherEntity;
		Obstacle.SqDist = SqDist;
		ClosestObstacles.Add(Obstacle);
	}
	// 遍历障碍物数组
	for (int32 Index = 0; Index < ClosestObstacles.Num(); Index++)
	{
		// 获取障碍物的速度
		const FMassVelocityFragment* OtherVelocityFragment = 
			OtherEntityView.GetFragmentDataPtr<FMassVelocityFragment>();
		const FVector OtherVelocity = OtherVelocityFragment != nullptr 
			? OtherVelocityFragment->Value : FVector::ZeroVector;
		// 如果障碍物有MoveTargetFragment，bCanAvoid为true
		const FMassMoveTargetFragment* OtherMoveTarget = 
			OtherEntityView.GetFragmentDataPtr<FMassMoveTargetFragment>();
		const bool bCanAvoid = OtherMoveTarget != nullptr;
		const bool bOtherIsMoving = OtherMoveTarget ? 
			OtherMoveTarget->GetCurrentAction() == EMassMovementAction::Move 
			: true;
		// 添加Collider数组
		FCollider& Collider = Colliders.Add_GetRef(FCollider{});
		Collider.Location = Obstacle.LocationCached;
		Collider.Velocity = OtherVelocity;
		Collider.Radius = 
			OtherEntityView.GetFragmentData<FAgentRadiusFragment>().Radius;
		Collider.bCanAvoid = bCanAvoid;
		Collider.bIsMoving = bOtherIsMoving;
	}
	// 遍历Colliders数组
	for (const FCollider& Collider : Colliders)
	{
		/*
		当对方不会主动避让(比如静止的障碍、玩家、不参与避障的实体)时,它和旁边的导航边界(墙 NavEdges)之间可能形成一条狭窄的缝隙(local minima)。如果 entity 试图从缝里挤过去会卡住,所以要主动绕开这条缝。
		*/
		if (Collider.bCanAvoid == false)
		{
			// 能安全通过的最小大小，就是entity的直径乘上个系数
			const FVector::FReal MinClearance = 2. * AgentRadius * 
				MovingAvoidanceParams.StaticObstacleClearanceScale;
			// 直接遍历Edge
			for (const FNavigationAvoidanceEdge& Edge : NavEdges.AvoidanceEdges)
			{
				// 根据障碍物位置，找到在Edge上的最近点
				const FVector Point = FMath::ClosestPointOnSegment(
					Collider.Location, Edge.Start, Edge.End);
				// 如果障碍物在Edge的背面，不用管这条Edge了
				const FVector Offset = Collider.Location - Point;
				if (FVector::DotProduct(Offset, Edge.LeftDir) < 0.)
				{
					continue;
				}
				// 障碍物和Edge之间缝隙的长度，ru
				const FVector::FReal OffsetLength = Offset.Length();
				const bool bTooNarrow = 
					(OffsetLength - Collider.Radius) < MinClearance; 
				if (bTooNarrow)
				{
					if (OffsetLength > MaxDist)
					{
						MaxDist = OffsetLength;
						ClosestPoint = Point;
					}
				}
			}

			if (MaxDist != -1.)
			{
				// Set up forced normal to avoid the gap between collider and edge.
				ForcedNormal = (Collider.Location - ClosestPoint).GetSafeNormal();
				bHasForcedNormal = true;
			}
		}
	}
}
```