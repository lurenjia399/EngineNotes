1 
```cpp
1 如果车已经停不下来了，说明已经在NextLane了，不用重新选择NextLane
2 判断是否到达了计算NextLane的距离，这个距离至少是两倍半径
3 如果NextLane有值，说明已经选择了，不用重新选择
4 如果CurrentLane.NextLanes.Num() == 1，说明只有一条Nextlane，直接选择
5 如果下一条车道不是交叉路口，就选择密度最小的那一条，
```
2 
```cpp
// 
/**/
void UMassTrafficChooseNextLaneProcessor::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	// 1 第一步，判断是否到达了计算NextLane的距离
	{
		float ChooseNextLaneDistanceFromLaneEnd = FMath::Max(VehicleControlFragment.Speed * ChooseNextLaneTime, ChooseNextLaneMinDistance);
		const float AssumedVehicleSpeed = FMath::Max(VehicleControlFragment.Speed, VehicleControlFragment.CurrentLaneConstData.SpeedLimit * 0.25);
		ChooseNextLaneDistanceFromLaneEnd = FMath::Max(
			MassTrafficSettings->StopSignBrakingTime * AssumedVehicleSpeed,
			ChooseNextLaneDistanceFromLaneEnd);
		const float VehicleLength = 2.0f * AgentRadiusFragment.Radius;
		if (ChooseNextLaneDistanceFromLaneEnd < VehicleLength)
		{
			ChooseNextLaneDistanceFromLaneEnd = VehicleLength;
		}
		if (LaneLocationFragment.DistanceAlongLane < (LaneLocationFragment.LaneLength - ChooseNextLaneDistanceFromLaneEnd))
		{
			continue;
		}
	}
}
```