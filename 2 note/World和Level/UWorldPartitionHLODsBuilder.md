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
bool UWorldPartitionHLODsBuilder::RunInternal(UWorld* InWorld, const FCellInfo& InCellInfo, FPackageSourceControlHelper& PackageHelper)
{
	
}
```