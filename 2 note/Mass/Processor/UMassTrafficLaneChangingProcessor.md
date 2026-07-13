
```cpp
1 选择要变道的目标车道，左右的相邻车道
2 
```

# 1 TryStartingNewLaneChange
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
	// 根据变道上的位置，找到位置前的车和位置后的车，如果找了两百次都找不到位置前的车就添加变道cd不让变了
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
	// 如果变道位置前的车或者后的车在进行变道，就添加变道cd不让变了
	if ((LaneChangeFragment_Chosen_Ahead 
		&& LaneChangeFragment_Chosen_Ahead->IsLaneChangeInProgress()) 
		||
		(LaneChangeFragment_Chosen_Behind 
		&& LaneChangeFragment_Chosen_Behind->IsLaneChangeInProgress()))
	{
		// 设置变道冷却时间
		LaneChangeFragment_Current.SetLaneChangeCountdownSecondsToBeAtLeast
			(MassTrafficSettings, 
			EMassTrafficLaneChangeCountdownSeconds::AsRetryUsingSettings, 
			RandomStream);
		return; 
	}
	// 改变Fragment数据，从当前车道移除，在目标车道添加
	if (!TeleportVehicleToAnotherLane()
	{
		// 设置变道冷却时间
		LaneChangeFragment_Current.SetLaneChangeCountdownSecondsToBeAtLeast
			(MassTrafficSettings, 
			EMassTrafficLaneChangeCountdownSeconds::AsRetryUsingSettings, 
			RandomStream);
		return;		
	}
	// 如果数据设置完了，但是变成OFF了
	if (UE::MassLOD::GetLODFromArchetype(Context) == EMassLOD::Off)
	{
		// 设置变道冷却时间
		LaneChangeFragment_Current.SetLaneChangeCountdownSecondsToBeAtLeast
			(MassTrafficSettings, 
			EMassTrafficLaneChangeCountdownSeconds::AsRetryUsingSettings, 
			RandomStream);
		//
		InterpolatePositionAndOrientationAlongLane();
	}
	else
	{
		// 开始变道
		const bool bDidStartLaneChangeProgression = 
			LaneChangeFragment_Current.BeginLaneChangeProgression(
			//DebugLabel,
			LaneChangeSide,
			BeginDistanceAlongLaneForLaneChange_Chosen, 
			EndDistanceAlongLaneForLaneChange_Chosen,
			DistanceBetweenLanes,
			// Fragments..
			TransformFragment_Current,
			VehicleLightsFragment_Current,
			NextVehicleFragment_Current,
			ZoneGraphLaneLocationFragment_Current,
			Lane_Current/*initial*/, Lane_Chosen,
			// Other vehicles involved in lane change..
			Entity_Current,
			Entity_Current_Behind, Entity_Current_Ahead,
			Entity_Chosen_Behind, Entity_Chosen_Ahead,
			// Other..
			EntityManager);
		}
	}
}
```

# 2 ChooseLaneForLaneChange
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

# 3 TeleportVehicleToAnotherLane
```cpp
bool TeleportVehicleToAnotherLane()
{
	// 从当前车道上移除需要变道的车
	{
		if (Entity_Current_Behind.IsSet() && Entity_Current_Ahead.IsSet())
		{
			NextVehicleFragment_Current_Behind->SetNextVehicle(
				Entity_Current_Behind, Entity_Current_Ahead);
		}
		else if (Entity_Current_Behind.IsSet() && !Entity_Current_Ahead.IsSet())
		{
			NextVehicleFragment_Current_Behind->UnsetNextVehicle();
		}
		else if (!Entity_Current_Behind.IsSet() && Entity_Current_Ahead.IsSet())
		{
			TrafficLaneData_Current.TailVehicle = Entity_Current_Ahead;
		}
		else if (!Entity_Current_Behind.IsSet() && !Entity_Current_Ahead.IsSet())
		{
			TrafficLaneData_Current.TailVehicle = FMassEntityHandle();
		}
	}
	// 在变道的车道上添加需要变道的车
	{
		if (Entity_Chosen_Behind.IsSet() && Entity_Chosen_Ahead.IsSet())
		{
			NextVehicleFragment_Current.SetNextVehicle(
				Entity_Current, Entity_Chosen_Ahead);

			NextVehicleFragment_Chosen_Behind->SetNextVehicle(
				Entity_Chosen_Behind, Entity_Current);				
		}
		else if (Entity_Chosen_Behind.IsSet() && !Entity_Chosen_Ahead.IsSet())
		{
			NextVehicleFragment_Current.UnsetNextVehicle();
			NextVehicleFragment_Chosen_Behind->SetNextVehicle(
				Entity_Chosen_Behind, Entity_Current);				
		}
		else if (!Entity_Chosen_Behind.IsSet() && Entity_Chosen_Ahead.IsSet())
		{
			NextVehicleFragment_Current.SetNextVehicle(
				Entity_Current, Entity_Chosen_Ahead);
			Lane_Chosen.TailVehicle = Entity_Current;
		}
		else if (!Entity_Chosen_Behind.IsSet() && !Entity_Chosen_Ahead.IsSet())
		{
			NextVehicleFragment_Current.UnsetNextVehicle();
			Lane_Chosen.TailVehicle = Entity_Current;
		}
	}
	// 给当前车到上移除车，向变道车道上添加车
	{
		const float SpaceTakenByVehicle_Current = GetSpaceTakenByVehicleOnLane(
			RadiusFragment_Current.Radius, 
			RandomFractionFragment_Current.RandomFraction, 
			MassTrafficSettings.MinimumDistanceToNextVehicleRange);
		TrafficLaneData_Current.RemoveVehicleOccupancy(
			SpaceTakenByVehicle_Current);
		Lane_Chosen.AddVehicleOccupancy(SpaceTakenByVehicle_Current);
	}
	// 重新设置当前车身上的Fragemtn信息
	VehicleControlFragment_Current.CurrentLaneConstData = Lane_Chosen.ConstData;
	VehicleControlFragment_Current.PreviousLaneIndex = INDEX_NONE; 
	LaneLocationFragment_Current.LaneHandle = Lane_Chosen.LaneHandle;
	LaneLocationFragment_Current.DistanceAlongLane = DistanceAlongLane_Chosen;
	LaneLocationFragment_Current.LaneLength = Lane_Chosen.Length;
	// 设置Nextlane，和MoveVehicleToNextLane的操作保持一致
	if (Lane_Chosen.NextLanes.Num() == 1)
	{
		VehicleControlFragment_Current.NextLane = Lane_Chosen.NextLanes[0];
		++VehicleControlFragment_Current.NextLane->NumVehiclesApproachingLane;
		if (!NextVehicleFragment_Current.HasNextVehicle())
		{
			NextVehicleFragment_Current.SetNextVehicle(Entity_Current, VehicleControlFragment_Current.NextLane->TailVehicle);
		}
	}
	else
	{
		VehicleControlFragment_Current.NextLane = nullptr;
	}
}
```