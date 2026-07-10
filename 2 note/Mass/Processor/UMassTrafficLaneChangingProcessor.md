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
	// 如果当前车前边堆满了要变道过来的车辆，当前车就不能变道走
	if (NextVehicleFragment_Current.NextVehicles_LaneChange.IsFull())
	{
		// 设置变道冷却时间
		LaneChangeFragment_Current.SetLaneChangeCountdownSecondsToBeAtLeast
			(MassTrafficSettings, 
			EMassTrafficLaneChangeCountdownSeconds::AsRetryUsingSettings, 
			RandomStream);
		return; 
	}
	// 根据当前车的位置，通过FindNearestLocationOnLane方法，找到变道上的位置
	{
		FZoneGraphLaneLocation ZoneGraphLocationOnLane_Current;
		UE::ZoneGraph::Query::CalculateLocationAlongLane(
			ZoneGraphStorage, Lane_Current->LaneHandle, 
			DistanceAlongLane_Current, ZoneGraphLocationOnLane_Current);
		if (!ZoneGraphLocationOnLane_Current.IsValid())
		{
			return;
		}
		Position_Current = ZoneGraphLocationOnLane_Current.Position;
		const float ZoneGraphLaneSearchDistance = 
		MassTrafficSettings.LaneChangeSearchDistanceScale * 
		GetMaxDistanceBetweenLanes(Lane_Current->LaneHandle.Index, 
			Lane_Chosen->LaneHandle.Index, ZoneGraphStorage); 
		float DistanceSquared;
		const FZoneGraphLaneLocation ZoneGraphLocationOnLane_Chosen = 
		GetClosestLocationOnLane(Position_Current, Lane_Chosen->LaneHandle.Index,
		ZoneGraphLaneSearchDistance, ZoneGraphStorage, &DistanceSquared);
		if (!ZoneGraphLocationOnLane_Chosen.IsValid())
		{
			return;
		}
		Position_Chosen = ZoneGraphLocationOnLane_Chosen.Position;
		DistanceAlongLane_Chosen = 
			ZoneGraphLocationOnLane_Chosen.DistanceAlongLane;
		DistanceBetweenLanes = FMath::Sqrt(DistanceSquared);
	}
	// 计算变道开始时在变道上的位置，和变道结束在变道上的位置，如果变道结束时的位置大于车道长度了，就添加变道cd不让变了
	{
		//省略了
	}
	// 根据变道上的位置，找到位置前的车和位置后的车，如果
	if (!FindNearbyVehiclesOnLane_RelativeToDistanceAlongLane(
		Lane_Chosen, DistanceAlongLane_Chosen, /*out*/Entity_Chosen_Behind,
		 /*out*/Entity_Chosen_Ahead, /*Coordinator,*/ EntityManager))
	{
	// 设置变道冷却时间
		LaneChangeFragment_Current.SetLaneChangeCountdownSecondsToBeAtLeast
			(MassTrafficSettings, 
			EMassTrafficLaneChangeCountdownSeconds::AsRetryUsingSettings, 
			RandomStream);
		return; 
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