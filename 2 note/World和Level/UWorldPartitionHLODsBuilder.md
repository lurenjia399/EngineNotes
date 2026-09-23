# RunInternal
``` cpp
bool UWorldPartitionHLODsBuilder::RunInternal(UWorld* InWorld, const FCellInfo& InCellInfo, FPackageSourceControlHelper& PackageHelper)
{
	if (bRet && ShouldRunStep(EHLODBuildStep::HLOD_Setup))
	{
		bRet = SetupHLODActors();
	}
	if (bRet && ShouldRunStep(EHLODBuildStep::HLOD_Build))
	{
		bRet = BuildHLODActors();
	}

	if (bRet && ShouldRunStep(EHLODBuildStep::HLOD_Delete))
	{
		bRet = DeleteHLODActors();
	}

	if (bRet && ShouldRunStep(EHLODBuildStep::HLOD_Finalize))
	{
		bRet = SubmitHLODActors();
	}

	if (bRet && ShouldRunStep(EHLODBuildStep::HLOD_Stats))
	{
		bRet = DumpStats();
	}
}
```

## SetupHLODActors
内部实现参考 [[WorldPartition#SetupHLODActors]]

## BuildHLODActors
```cpp
bool UWorldPartitionHLODsBuilder::BuildHLODActors()
{
	if (WorldPartition)
	{
		// 1 获取需要Build的HLODActor
		TArray<FGuid> HLODActorsToBuild;
		if (!GetHLODActorsToBuild(HLODActorsToBuild))
		{
			return false;
		}
		// 2 
		for (int32 CurrentActor = ResumeBuildIndex; CurrentActor < HLODActorsToBuild.Num(); ++CurrentActor)
		{
			
		}
	}
}
```