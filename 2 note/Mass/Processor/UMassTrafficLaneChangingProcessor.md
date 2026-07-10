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
	// 如果当前车道不能变道，就是没有左右split,mergexiang'lin'che
	if (!TrafficLaneData_Initial->ConstData.bIsLaneChangingLane)
	{
		return;		
	}
	else if (!TrafficLaneData_Initial->SplittingLanes.IsEmpty() || !TrafficLaneData_Initial->MergingLanes.IsEmpty())
	{
		return;
	}
}
```