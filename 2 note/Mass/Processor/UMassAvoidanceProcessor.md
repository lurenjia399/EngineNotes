# UMassMovingAvoidanceProcessor
1 
```cpp
1 通过THierarchicalHashGrid2D来空间存储障碍物信息，UMassNavigationObstacleGridProcessor这个来将障碍物添加到SubSystem中存储
2 Execute执行的时候通过查找范围获取到entity附近的障碍物
3 施加分离力，是由障碍物指向entity方向里的力，力的大小默认是250N/cm
4 施加CPA预测分离力，将预测接近点也认为是障碍物计算出分离力，使entity提前躲避
5 在计算分离力的过程中，对不会避让的障碍物（没有MoveTarget的entity）,检测它与Edge形成的狭窄夹缝,用 ForcedNormal 引导 entity 主动绕开而非往缝里钻
```
2 
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
		// 标记entity和障碍物之间的侵入量，就是理想的间距 - 当前的实际的间距
		const FVector::FReal PenSep = 
			(SeparationAgentRadius + Collider.Radius + 
			MovingAvoidanceParams.ObstacleSeparationDistance) - ConDist;
		// 侵入量 / 理想的间距，然后平方，得到一个反比例函数，如果侵入量小说明不用分离力，如果侵入量大就需要更大的比例来施加分离力
		const FVector::FReal SeparationMag = 
			FMath::Square(FMath::Clamp(PenSep /
			MovingAvoidanceParams.ObstacleSeparationDistance, 0., 1.));
		// 将分离力添加到SteeringForce中
		SteeringForce += SeparationForce;
		
		// 计算CPA，根据双方相对位置和相对速度,预测在未来PredictiveAvoidanceTime 时间窗口内最接近的那个时刻(一个 0 到  PredictiveAvoidanceTime 之间的时间值)。
		const FVector::FReal CPA =
			UE::MassAvoidance::ComputeClosestPointOfApproach(
			RelPos, RelVel, PredictiveAvoidanceAgentRadius 
			+ Collider.Radius, MovingAvoidanceParams.PredictiveAvoidanceTime);
		// 根据CPA，计算出预测最接近点，最接近点指向entity位置作为预测避障力
		const FVector AvoidRelPos = RelPos + RelVel * CPA;
		const FVector::FReal AvoidDist = AvoidRelPos.Size();
		const FVector AvoidConNormal = 
			AvoidDist > UE_KINDA_SMALL_NUMBER 
			? (AvoidRelPos / AvoidDist) : FVector::ForwardVector;
		
		/*
		 1. AvoidMag:未来最接近点的侵入深度(越深越强,平方曲线)。
		  2. AvoidMagDist:碰撞发生得越早(CPA越小),力越大;越晚越弱。给近在眼前的威胁更高优先级。
		*/
		const FVector AvoidForce = 
			AvoidNormal * AvoidMag * AvoidMagDist * 
			MovingAvoidanceParams.ObstaclePredictiveAvoidanceStiffness *
			StandingScaling;
			SteeringForce += AvoidForce;
		// 限制力的最大值
		Force.Value = UE::MassNavigation::ClampVector(
			SteeringForce, MaxSteerAccel);
	}
}
```

# UMassStandingAvoidanceProcessor
1
```cpp
1 初始化Ghost是在UMassSteerToMoveTargetProcessor中，如果moveTarget是Stand的，才会初始化Ghost
2 如果障碍物也有Ghost
3 如果障碍物没有Ghost，也就是移动障碍物，
2 计算目标力，把 ghost 拉回MoveTarget ghost 朝 MoveTarget.Center 转向,离得越近速度衰减(GhostStandSlowdownRadius)
3 计算分离力，这里有两类:
  - 对方也是站立、也有有效Ghost:→ 做 ghost-vs-ghost 分离。施加一个OtherGhost指向当前Ghost的力
  - 对方在移动 / 没 Ghost:→ 把对方当成一个朝前突出的 2D 胶囊线段(OtherPersonalSpacePosition 沿其 forward 延伸),ghost躲这个胶囊。
4 积分Ghost速度，位置等信息
```
2
```cpp
void UMassStandingAvoidanceProcessor::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	// 只考虑Stand状态的
	const FMassMoveTargetFragment& MoveTarget = MoveTargetList[EntityIt];
	if (MoveTarget.GetCurrentAction() != EMassMovementAction::Stand)
	{
		continue;
	}
	// 计算目标力，把Ghost拉回MoveTarget.Center
	const FVector::FReal SteerK = 1. / StandingParams.GhostSteeringReactionTime;
	constexpr FVector::FReal SteeringMinDistance = 1.;
	FVector SteerDirection = FVector::ZeroVector;
	FVector Delta = MoveTarget.Center - Ghost.Location;
	Delta.Z = 0.;
	const FVector::FReal Distance = Delta.Size();
	FVector::FReal SpeedFade = 0.;
	if (Distance > SteeringMinDistance)
	{
		SteerDirection = Delta / Distance;
		SpeedFade = FMath::Clamp(
			Distance / FMath::Max(KINDA_SMALL_NUMBER, 
			StandingParams.GhostStandSlowdownRadius), 0., 1.);
	}
	const FVector GhostDesiredVelocity = SteerDirection 
		* StandingParams.GhostMaxSpeed * SpeedFade;
	FVector GhostSteeringForce = SteerK * (GhostDesiredVelocity - Ghost.Velocity);
	
	// 找到entity周围的其他entity
	const FNavigationObstacleHashGrid2D& AvoidanceObstacleGrid = 
		NavigationSubsystem->GetObstacleGridMutable();
	UE::MassAvoidance::FindCloseObstacles(
		AgentLocation, 
		MovingAvoidanceParams.ObstacleDetectionDistance,
		AvoidanceObstacleGrid, CloseEntities, 
		UE::MassAvoidance::MaxObstacleResults);
	// 遍历找到的CloseGhose,计算分离力，就是其他Ghost指向当前Ghost的力
	for (int32 Index = 0; Index < NumCloseObstacles; Index++)
	{
		const FVector::FReal OtherDistanceToGoal = FVector::Distance(OtherGhost->Location, OtherMoveTarget->Center);
		const FVector::FReal OtherSteerFade = FMath::Clamp(OtherDistanceToGoal / StandingParams.GhostToTargetMaxDeviation, 0., 1.);
		const FVector::FReal SeparationStiffness = FMath::Lerp(GhostSeparationStiffness, MovingSeparationStiffness, OtherSteerFade);
		FVector RelPos = Ghost.Location - OtherGhost->Location;
		RelPos.Z = 0.; // we assume we work on a flat plane for now
		const FVector::FReal ConDist = RelPos.Size();
		const FVector ConNorm = ConDist > 0. ? RelPos / ConDist : FVector::ForwardVector;
		const FVector::FReal PenSep = (TotalRadius + GhostSeparationDistance) - ConDist;
		const FVector::FReal SeparationMag = UE::MassNavigation::Smooth(FMath::Clamp(PenSep / GhostSeparationDistance, 0., 1.));
		const FVector SeparationForce = ConNorm * SeparationStiffness * SeparationMag;
		GhostSteeringForce += SeparationForce;
	}
	// 积分速度
	Ghost.Velocity += GhostSteeringForce * DeltaTime;
	Ghost.Velocity.Z = 0.;
	Ghost.Location += Ghost.Velocity * DeltaTime;
	// 范围限制，不能离MoveTarget点太远
	const FVector DirToCenter = Ghost.Location - MoveTarget.Center;
	const FVector::FReal DistToCenter = DirToCenter.Length();
	if (DistToCenter > StandingParams.GhostToTargetMaxDeviation)
	{
		Ghost.Location = MoveTarget.Center + DirToCenter 
			* (StandingParams.GhostToTargetMaxDeviation / DistToCenter);
	}
}
```