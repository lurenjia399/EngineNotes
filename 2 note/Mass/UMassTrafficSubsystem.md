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
	// 遍历ZoneGraphStorage中所有车道
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
		// 3 往TrafficLaneDataArray数组中添加数据，记录TrafficLane的CenterLocation，Radius
		FZoneGraphTrafficLaneData& TrafficLaneData = 
			TrafficZoneGraphData.TrafficLaneDataArray.AddDefaulted_GetRef();
		TrafficLaneData.LaneHandle = FZoneGraphLaneHandle(
			LaneIndex, TrafficZoneGraphData.DataHandle);
		const FVector MidPoint = UE::MassTraffic::GetLaneMidPoint(
			LaneIndex, ZoneGraphStorage);
		TrafficLaneData.CenterLocation = MidPoint; 
		TrafficLaneData.Radius.Set(FVector::Distance(MidPoint, 
			UE::MassTraffic::GetLaneBeginPoint(LaneIndex, ZoneGraphStorage)));
		// 
}
```