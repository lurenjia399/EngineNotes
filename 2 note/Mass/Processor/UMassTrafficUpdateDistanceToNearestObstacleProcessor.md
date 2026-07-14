1 
2 FMassTrafficObstacleAvoidanceFragment
```cpp
struct MASSTRAFFIC_API FMassTrafficObstacleAvoidanceFragment : public FMassFragment
{
	// 到跟车的最短距离，就是当前车的车头到前一辆车的屁股，还有多少距离就要跟前车撞上了
	float DistanceToNext = TNumericLimits<float>::Max();
	
	float TimeToCollidingObstacle = TNumericLimits<float>::Max();
	float DistanceToCollidingObstacle = TNumericLimits<float>::Max();
	FMassEntityHandle Obstacle;
};
```
3 
```cpp
void UMassTrafficUpdateDistanceToNearestObstacleProcessor::Execute()
{
	// 如果有跟车对象，就计算到跟车的最短距离
	if (NextVehicleFragment.HasNextVehicle())
	{
		CombineDistanceToNext(
			EMassTrafficCombineDistanceToNextType::Next, 
			NextTransformFragment, NextRadiusFragment);
	}
	// 遍历跟车后即将因为变道来的车
	for (FMassEntityHandle NextVehicle_LaneChange : 
		NextVehicleFragment.NextVehicles_LaneChange)
	{
		if (CanNextVehicleBeForgotten(
			NextSimulationParams, NextTransformFragment, 
			NextRadiusFragment, NextLaneChangeFragment))
		{
			NextVehicleFragment.RemoveLaneChangeNextVehicle(
				NextVehicle_LaneChange);
		}
		else
		{
			CombineDistanceToNext(
				EMassTrafficCombineDistanceToNextType::LaneChangeNext,
				 NextTransformFragment, NextRadiusFragment);
		}
	}
}
```