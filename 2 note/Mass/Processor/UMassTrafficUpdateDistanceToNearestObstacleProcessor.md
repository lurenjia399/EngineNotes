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
	// 如果有跟车对象，就计算DistanceToNext，是到跟车的距离
	if (NextVehicleFragment.HasNextVehicle())
	{
		CombineDistanceToNext(
			EMassTrafficCombineDistanceToNextType::Next, 
			NextTransformFragment, NextRadiusFragment);
	}
	// 遍历跟车后即将因为变道来的车，计算DistanceToNext，是到变道车的距离
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
	// 如果有因为Splitt的幽灵占位车，也是一样，计算DistanceToNext，是到幽灵车的距离
	if (NextVehicleFragment.NextVehicle_SplittingLaneGhost.IsSet())
	{
		if (EntityManager.IsEntityActive
			(NextVehicleFragment.NextVehicle_SplittingLaneGhost))
		{
			
			if (CanNextVehicleBeForgotten(
				NextSimulationParams, NextTransformFragment, 
				NextRadiusFragment, NextLaneChangeFragment))
			{
				NextVehicleFragment.NextVehicle_SplittingLaneGhost = 
					FMassEntityHandle();
			}
			else
			{
				CombineDistanceToNext(
					EMassTrafficCombineDistanceToNextType::LaneChangeNext,
					 NextTransformFragment, NextRadiusFragment);
			}
		}
		else
		{
			NextVehicleFragment.NextVehicle_SplittingLaneGhost = 
				FMassEntityHandle();
		}
	}
	// 如果有因为Merge的幽灵占位车，也是一样，计算DistanceToNext，是到幽灵车的距离
	if (NextVehicleFragment.NextVehicle_MergingLaneGhost.IsSet())
	{
		if (EntityManager.IsEntityActive(
			NextVehicleFragment.NextVehicle_MergingLaneGhost))
		{
			CombineDistanceToNext(
				EMassTrafficCombineDistanceToNextType::LaneChangeNext,
				 NextTransformFragment, NextRadiusFragment);
		}
		else
		{
			NextVehicleFragment.NextVehicle_MergingLaneGhost = FMassEntityHandle();
		}
	}
	// 如果障碍物列表中有障碍物
	if (!OptionalObstacleListFragments.IsEmpty())
	{
		// 计算理想速度 = 当前Forward * 当前车道的速度限制
		FVector IdealVelocity = 
			TransformFragment.GetTransform().GetRotation().GetForwardVector() 
			* VehicleControlFragment.CurrentLaneConstData.SpeedLimit.GetFloat();  
				
	}
}
```