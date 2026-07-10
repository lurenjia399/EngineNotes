1 TryStartingNewLaneChange
```cpp
void TryStartingNewLaneChange()
{
	// 选择一个变道
	FMassTrafficLaneChangeRecommendation LaneChangeRecommendation;
	ChooseLaneForLaneChange();
	// 如果是不能变道
	if (LaneChangeRecommendation.Level == StayOnCurrentLane_RetryNormal)
	{
		// 设置变道冷却时间
		LaneChangeFragment_Current.SetLaneChangeCountdownSecondsToBeAtLeast
			(MassTrafficSettings, 
			EMassTrafficLaneChangeCountdownSeconds::AsRetryUsingSettings, 
			RandomStream);
		// 看情况设置，这条车道上不能变道
		LaneChangeFragment_Current.bBlockAllLaneChangesUntilNextLane = 
			LaneChangeRecommendation.bNoLaneChangesUntilNextLane;
		return;
	}
	// 添加少量变道cd，快速在次请求变道
	else if (LaneChangeRecommendation.Level == StayOnCurrentLane_RetrySoon)
	{
		// 设置变道冷却时间
		LaneChangeFragment_Current.SetLaneChangeCountdownSecondsToBeAtLeast
			(MassTrafficSettings, 
			EMassTrafficLaneChangeCountdownSeconds::AsRetryUsingSettings, 
			RandomStream);
		// 看情况设置，这条车道上不能变道
		LaneChangeFragment_Current.bBlockAllLaneChangesUntilNextLane = 
			LaneChangeRecommendation.bNoLaneChangesUntilNextLane;
		return;
	}
	// 剩下这些类型就是可以变道
	else if (LaneChangeRecommendation.Level == NormalLaneChange)
	{
	}
	else if (LaneChangeRecommendation.Level == TransversingLaneChange)
	{
	}
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
	// 左车道，右车道都没有，就返回
	if (!CandidateTrafficLaneData_Left && !CandidateTrafficLaneData_Right)
	{
		return;
	}
	// 如果有左车道，没有右车道，就选择左车道
	else if (CandidateTrafficLaneData_Left && !CandidateTrafficLaneData_Right)
	{
		OutRecommendation.Lane_Chosen = CandidateTrafficLaneData_Left;
		OutRecommendation.bChoseLaneOnLeft = true;
		OutRecommendation.Level = NormalLaneChange;			
		return;
	}
	// 如果没有左车道，有右车道，就选择右车道
	else if (!CandidateTrafficLaneData_Left && CandidateTrafficLaneData_Right)
	{
		OutRecommendation.Lane_Chosen = CandidateTrafficLaneData_Right;
		OutRecommendation.bChoseLaneOnRight = true;
		OutRecommendation.Level = NormalLaneChange;			
		return;
	}
	// 如果都有，挑选密度小的
	else if (CandidateTrafficLaneData_Left && CandidateTrafficLaneData_Right)
	{
		if (DownstreamFlowDensity_Candidate_Left < DownstreamFlowDensity_Candidate_Right)
		{
			OutRecommendation.Lane_Chosen = CandidateTrafficLaneData_Left;
			OutRecommendation.bChoseLaneOnLeft = true;
			OutRecommendation.Level = NormalLaneChange;
			return;				
		}
		else if (DownstreamFlowDensity_Candidate_Right < DownstreamFlowDensity_Candidate_Left) 
		{
			OutRecommendation.Lane_Chosen = CandidateTrafficLaneData_Right;
			OutRecommendation.bChoseLaneOnRight = true;
			OutRecommendation.Level = NormalLaneChange;
			return;
		}
		else
		{
			if (RandomStream.FRand() < 0.5f)
			{
				OutRecommendation.Lane_Chosen = CandidateTrafficLaneData_Left;
				OutRecommendation.bChoseLaneOnLeft = true;
				OutRecommendation.Level = NormalLaneChange;
				return;
			}
			else
			{
				OutRecommendation.Lane_Chosen = CandidateTrafficLaneData_Right;
				OutRecommendation.bChoseLaneOnRight = true;
				OutRecommendation.Level = NormalLaneChange;
				return;
			}
		}
	}
}
```