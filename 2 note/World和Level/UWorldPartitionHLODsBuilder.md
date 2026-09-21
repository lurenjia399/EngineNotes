# SetupHLODActors
``` cpp
bool UWorldPartitionHLODsBuilder::SetupHLODActors()
{
	// 不是独立HLODWorld的WP
	if (WorldPartition && !WorldPartition->IsStandaloneHLODWorld())
	{
		
	}
	WorldPartition->SetupHLODActors(SetupHLODActorsParams);
}
```