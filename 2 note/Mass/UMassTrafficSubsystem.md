1 构建道路数据BuildLaneData，是在ZoneGraphData注册好后，会根据ZoneStorage构建
```cpp
void UMassTrafficSubsystem::RegisterZoneGraphData(const AZoneGraphData* ZoneGraphData)
{
	const FZoneGraphStorage& Storage = ZoneGraphData->GetStorage();
	const int32 Index = Storage.DataHandle.Index;
	while (Index >= RegisteredTrafficZoneGraphData.Num())
	{
		RegisteredTrafficZoneGraphData.Add(new FMassTrafficZoneGraphData());
	}

	FMassTrafficZoneGraphData& LaneData = RegisteredTrafficZoneGraphData[Index];
	if (LaneData.DataHandle != Storage.DataHandle)
	{
		BuildLaneData(LaneData, Storage);
	}
}
```


```cpp
void UMassTrafficSubsystem::BuildLaneData(
	FMassTrafficZoneGraphData& TrafficZoneGraphData, 
	const FZoneGraphStorage& ZoneGraphStorage)
{
	// 遍历ZoneGraphStorage中所有车道，向TrafficLaneDataArray数组中添加TrafficLane，并记录车道基础信息
	for (int32 LaneIndex = 0; 
		LaneIndex < ZoneGraphStorage.Lanes.Num(); ++LaneIndex)
	{
		// 1 过滤不是TrafficLane
		const FZoneLaneData& ZoneLaneData = ZoneGraphStorage.Lanes[LaneIndex];
		if (!MassTrafficSettings->TrafficLaneFilter.Pass(ZoneLaneData.Tags))
		{
			continue;
		}
		// 2 不是Intersection车道，如果是之字型车道，就continue，交通系统不模拟
		if (!MassTrafficSettings->IntersectionLaneFilter.Pass(ZoneLaneData.Tags))
		{
			int32 MergingLaneIndex = INDEX_NONE;
			int32 SplittingLaneIndex = INDEX_NONE;
			bool bSplittingRight = false;
			if (IsZigLagLane(ZoneGraphStorage, 
				LaneIndex, MergingLaneIndex, SplittingLaneIndex, bSplittingRight))
			{
				if (bSplittingRight)
				{
					LeftLaneOverrides.Add(SplittingLaneIndex, MergingLaneIndex);
				}
				else
				{
					RightLaneOverrides.Add(SplittingLaneIndex, MergingLaneIndex);
				}
				continue;
			}
		}
		// 3 往TrafficLaneDataArray数组中添加数据，记录TrafficLane的CenterLocation，Radius，SpeedLimit
		FZoneGraphTrafficLaneData& TrafficLaneData = 
			TrafficZoneGraphData.TrafficLaneDataArray.AddDefaulted_GetRef();
		TrafficLaneData.LaneHandle = FZoneGraphLaneHandle(
			LaneIndex, TrafficZoneGraphData.DataHandle);
		const FVector MidPoint = UE::MassTraffic::GetLaneMidPoint(
			LaneIndex, ZoneGraphStorage);
		TrafficLaneData.CenterLocation = MidPoint; 
		TrafficLaneData.Radius.Set(FVector::Distance(MidPoint, 
			UE::MassTraffic::GetLaneBeginPoint(LaneIndex, ZoneGraphStorage)));
		TrafficLaneData.ConstData.SpeedLimit = Chaos::MPHToCmS(SpeedLimitMPH);
		// 4 根据Setting，记录是否是交叉车道
		TrafficLaneData.ConstData.bIsIntersectionLane = 
			MassTrafficSettings->IntersectionLaneFilter.Pass(ZoneLaneData.Tags);
		// 5 是否是允许变道的车道
		TrafficLaneData.ConstData.bIsLaneChangingLane = 
			MassTrafficSettings->LaneChangingLaneFilter.Pass(ZoneLaneData.Tags)
		// 6 记录车道长度
		UE::ZoneGraph::Query::GetLaneLength(
			ZoneGraphStorage, TrafficLaneData.LaneHandle, TrafficLaneData.Length);
	}
	// 构建TrafficLaneDataLookup数组，快速查询，key是LaneIndex，value是TrafficLaneData
	TrafficZoneGraphData.TrafficLaneDataLookup.SetNumZeroed(
		ZoneGraphStorage.Lanes.Num());
	for (FZoneGraphTrafficLaneData& TrafficLaneData : 
		TrafficZoneGraphData.TrafficLaneDataArray)
	{
		TrafficZoneGraphData.TrafficLaneDataLookup
			[TrafficLaneData.LaneHandle.Index] = &TrafficLaneData;
	}
	//
}
```