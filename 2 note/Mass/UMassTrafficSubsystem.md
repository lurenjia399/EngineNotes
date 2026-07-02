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
		
}
```