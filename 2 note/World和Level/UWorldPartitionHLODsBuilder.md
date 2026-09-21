# SetupHLODActors
``` cpp
bool UWorldPartitionHLODsBuilder::SetupHLODActors()
{
	// 不是独立HLODWorld的WP关卡会执行
	if (WorldPartition && !WorldPartition->IsStandaloneHLODWorld())
	{
		WorldPartition->SetupHLODActors(SetupHLODActorsParams);
	}
	
}
```