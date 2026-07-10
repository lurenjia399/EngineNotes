1 TryStartingNewLaneChange
```cpp
void TryStartingNewLaneChange()
{
	
}
```

2 ChooseLaneForLaneChange
```cpp
void ChooseLaneForLaneChange()
{
	// 如果当前车道不能变道，就是没有左右相邻车道
	if (!TrafficLaneData_Initial->ConstData.bIsLaneChangingLane)
	{
		return;		
	}
	// 如果当前车道有split，merge车道就不能变道
	else if (!TrafficLaneData_Initial->SplittingLanes.IsEmpty() 
		|| !TrafficLaneData_Initial->MergingLanes.IsEmpty())
	{
		return;
	}
	
	// 根据密度，选择一个左车道，右车道
	FZoneGraphTrafficLaneData* CandidateTrafficLaneData_Left = 
		TrafficLaneData_Initial->LeftLane;
	FZoneGraphTrafficLaneData* CandidateTrafficLaneData_Right = 
		TrafficLaneData_Initial->RightLane;
	CandidateTrafficLaneData_Left = FilterLaneForLaneChangeSuitability(
		CandidateTrafficLaneData_Left, *TrafficLaneData_Initial, 
		VehicleControlFragment, SpaceTakenByVehicleOnLane);
	CandidateTrafficLaneData_Right = FilterLaneForLaneChangeSuitability(
		CandidateTrafficLaneData_Right, *TrafficLaneData_Initial, 
		VehicleControlFragment, SpaceTakenByVehicleOnLane);
	// 
}
```