1 
```cpp
// 
void UMassTrafficChooseNextLaneProcessor::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	{
		// Get distance threshold to choose next lane based on current speed.
		float ChooseNextLaneDistanceFromLaneEnd = FMath::Max(VehicleControlFragment.Speed * ChooseNextLaneTime, ChooseNextLaneMinDistance);

		// Also make sure we choose a lane with enough time to come to a stop if it's closed.
		const float AssumedVehicleSpeed = FMath::Max(VehicleControlFragment.Speed, VehicleControlFragment.CurrentLaneConstData.SpeedLimit * 0.25);
		ChooseNextLaneDistanceFromLaneEnd = FMath::Max(
			MassTrafficSettings->StopSignBrakingTime * AssumedVehicleSpeed,
			ChooseNextLaneDistanceFromLaneEnd);

		// Make sure we have chosen a lane if we're very close to the end of the lane.
		const float VehicleLength = 2.0f * AgentRadiusFragment.Radius;
		if (ChooseNextLaneDistanceFromLaneEnd < VehicleLength)
		{
			ChooseNextLaneDistanceFromLaneEnd = VehicleLength;
		}

		// We only need to choose a lane when we get near the end.
		if (LaneLocationFragment.DistanceAlongLane < (LaneLocationFragment.LaneLength - ChooseNextLaneDistanceFromLaneEnd))
		{
			continue;
		}
	}
}
```