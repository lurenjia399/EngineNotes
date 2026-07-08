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
		
	}
}
```