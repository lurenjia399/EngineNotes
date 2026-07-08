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
		
	for (const FNavigationObstacleHashGrid2D::ItemIDType OtherEntity 
		: CloseEntities)
	{
		
	}
}
```