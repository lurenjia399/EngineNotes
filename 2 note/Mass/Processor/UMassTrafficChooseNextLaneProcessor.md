1 
```cpp
// 
/**/
void UMassTrafficChooseNextLaneProcessor::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	// 1 第一步，判断是否到计算NextLane的距离，用
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