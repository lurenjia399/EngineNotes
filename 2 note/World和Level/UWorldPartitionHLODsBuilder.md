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
		// 2 遍历HLODActor，每个HLODActor都执行BuildHLOD方法，方法最终执行FWorldPartitionHLODUtilities::BuildHLOD方法
		for (int32 CurrentActor = ResumeBuildIndex; CurrentActor < HLODActorsToBuild.Num(); ++CurrentActor)
		{
			AWorldPartitionHLOD* HLODActor = 
				CastChecked<AWorldPartitionHLOD>(ActorRef.GetActor());
			HLODActor->BuildHLOD(bForceBuild);
		}
	}
}
```

```cpp
uint32 FWorldPartitionHLODUtilities::BuildHLOD(AWorldPartitionHLOD* InHLODActor)
{
	// 1 收集当前HLODActor所包含的Actor。也就是加载HLODActor上的SourceActor，返回LevelStreaming类型。就是这个HLODActor包含的Actor集合，封装成了LevelStreaming。
	ULevelStreaming* LevelStreaming = nullptr;
	{
		LevelStreaming = LoadSourceActors(InHLODActor, bIsDirty);
	}
	// 2 收集包含Actor身上和HLOD相关的Comp，收集到HLODRelevantComponents数组中。
	TArray<UActorComponent*> HLODRelevantComponents;
	if (LevelStreaming->GetLoadedLevel())
	{
		HLODRelevantComponents = GatherHLODRelevantComponents(LevelStreaming->GetLoadedLevel()->Actors);
	}
	// 3 计算老的Hash和新的Hash，没有改变的话就不需要重新构建了
	uint32 OldHLODHash = bIsDirty ? 0 : InHLODActor->GetHLODHash();
	uint32 NewHLODHash = ComputeHLODHash(InHLODActor, HLODRelevantComponents);
	if (OldHLODHash == NewHLODHash)
	{
		return OldHLODHash;
	}
	/*
	4.1 根据配置，找到HLODBuilder的UClass并创建出HLODBuilder。
	4.2 调用UHLODBuilder::Build方法，自定义重写，然后返回构建出来的Comp
	4.3 对构建出来的Comp做一些操作，关掉导航，
	*/
	const UHLODLayer* HLODLayer = InHLODActor->GetSourceActors()->GetHLODLayer();
	TSubclassOf<UHLODBuilder> HLODBuilderClass = GetHLODBuilderClass(HLODLayer);
	if (HLODBuilderClass)
	{
		// 4.1
		UHLODBuilder* HLODBuilder = NewObject<UHLODBuilder>(
			GetTransientPackage(), HLODBuilderClass);
		// 4.2
		FHLODBuildResult BuildResult;
		{
			FAutoScopedDurationTimer BuildTimeScope;
			BuildResult = HLODBuilder->Build(HLODBuildContext);
			BuildTimeMS = FMath::RoundToInt(BuildTimeScope.GetTime() * 1000);
		}
		// 4.3 
		
		// 4.4
		InHLODActor->SetInputStats(BuildResult.InputStats);
		InHLODActor->SetHLODComponents(BuildResult.HLODComponents);
	}
}
```