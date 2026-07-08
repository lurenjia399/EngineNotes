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
				// 障碍物和Edge之间缝隙的长度，如果太窄不能通过，就记录最大缝隙长度和构成缝隙的点
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
			// 如果有缝隙通过不了，就改变力的方向
			if (MaxDist != -1.)
			{
				ForcedNormal = (Collider.Location - ClosestPoint).GetSafeNormal();
				bHasForcedNormal = true;
			}
		}
		// 障碍物指向entity的向量，在障碍物坐标系下，entity的位置
		FVector RelPos = AgentLocation - Collider.Location;
		RelPos.Z = 0.;
		// 障碍物坐标系下，entity的速度
		const FVector RelVel = DesVel - Collider.Velocity;
		// 障碍物和entity之间的距离
		const FVector::FReal ConDist = RelPos.Size();
		// 障碍物指向entity向量的归一化
		const FVector ConNorm = ConDist > 0. ? 
			RelPos / ConDist : FVector::ForwardVector;
		// 一般情况下分离力就是障碍物指向entity的向量
		FVector SeparationNormal = ConNorm;
		// 但如果有缝隙entity无法通过，就需要计算出插值，因为光靠一般分离力是无法让entity避开的，插值的系数就是在障碍物坐标系下entity的速度和entity位置的夹角，如果夹角小于90度，说明entity已经朝向分离力的方向移动了不用插值了，如果夹角大于90度，说明entity移动方向是撞向障碍物，就需要插值分离力来让entity避开
		if (bHasForcedNormal)
		{
			const FVector RelVelNorm = RelVel.GetSafeNormal();
			const FVector::FReal Blend = 
				FMath::Max(0., -FVector::DotProduct(ConNorm, RelVelNorm));
			SeparationNormal = FMath::Lerp
				(ConNorm, ForcedNormal, Blend).GetSafeNormal();
		}
		// 对静止的障碍物，施加的分离力可以更小
		const FVector::FReal StandingScaling = Collider.bIsMoving 
			? 1. : MovingAvoidanceParams.StandingObstacleAvoidanceScale;
		// 标记entity和障碍物之间的侵入量，就是当前的距离和理想之间
		const FVector::FReal PenSep = 
			(SeparationAgentRadius + Collider.Radius + 
			MovingAvoidanceParams.ObstacleSeparationDistance) - ConDist;
		const FVector::FReal SeparationMag = 
			FMath::Square(FMath::Clamp(PenSep /
			MovingAvoidanceParams.ObstacleSeparationDistance, 0., 1.));
	}
}
```