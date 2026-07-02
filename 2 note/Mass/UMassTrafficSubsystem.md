1 构建道路数据BuildLaneData，是在ZoneGraph
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