```cpp
void UMassMovingAvoidanceProcessor::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	// zhao
	const FNavigationObstacleHashGrid2D& AvoidanceObstacleGrid = 
		NavigationSubsystem->GetObstacleGridMutable();
	UE::MassAvoidance::FindCloseObstacles(
		AgentLocation, 
		MovingAvoidanceParams.ObstacleDetectionDistance,
		AvoidanceObstacleGrid, CloseEntities, 
		UE::MassAvoidance::MaxObstacleResults);
}
```