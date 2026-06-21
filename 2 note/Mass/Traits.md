# 1 UMassStateTreeTrait
```cpp
void UMassStateTreeTrait::BuildTemplate(
	FMassEntityTemplateBuildContext& BuildContext, 
	const UWorld& World) const
{
	FMassEntityManager& EntityManager = 
		UE::Mass::Utils::GetEntityManagerChecked(World);

	UMassStateTreeSubsystem* MassStateTreeSubsystem = 
		World.GetSubsystem<UMassStateTreeSubsystem>();

	FMassStateTreeSharedFragment SharedStateTree;
	SharedStateTree.StateTree = StateTree;
	
	const FConstSharedStruct StateTreeFragment = EntityManager.
		GetOrCreateConstSharedFragment(SharedStateTree);
	
	// 添加这两个
	BuildContext.AddConstSharedFragment(StateTreeFragment);
	BuildContext.AddFragment<FMassStateTreeInstanceFragment>();
}
```

# 2 UMassVisualizationTrait

```cpp
void UMassVisualizationTrait::BuildTemplate(FMassEntityTemplateBuildContext& BuildContext, const UWorld& World) const
{
	// This should not be ran on NM_Server network mode
	if (World.IsNetMode(NM_DedicatedServer) && !bAllowServerSideVisualization 
		&& !BuildContext.IsInspectingData())
	{
		return;
	}
	// 需要这三个Fragment
	BuildContext.RequireFragment<FMassViewerInfoFragment>();
	BuildContext.RequireFragment<FTransformFragment>();
	BuildContext.RequireFragment<FMassActorFragment>();

	FMassEntityManager& EntityManager = 
		UE::Mass::Utils::GetEntityManagerChecked(World);

	// 获取配置的RepresentationSubSystem，并设置为共享的
	UMassRepresentationSubsystem* RepresentationSubsystem = 
		Cast<UMassRepresentationSubsystem>(World.
		GetSubsystemBase(RepresentationSubsystemClass));
	if (RepresentationSubsystem == nullptr && !BuildContext.IsInspectingData())
	{
		RepresentationSubsystem = UWorld::GetSubsystem<UMassRepresentationSubsystem>(&World);
		check(RepresentationSubsystem);
	}
	FMassRepresentationSubsystemSharedFragment SubsystemSharedFragment;
	SubsystemSharedFragment.RepresentationSubsystem = RepresentationSubsystem;
	FSharedStruct SubsystemFragment = EntityManager.GetOrCreateSharedFragment(SubsystemSharedFragment);
	BuildContext.AddSharedFragment(SubsystemFragment);
	
	// 将引用的Actor的UClass的索引，填充到FMassRepresentationFragment中
	FMassRepresentationFragment& RepresentationFragment = 
		BuildContext.AddFragment_GetRef<FMassRepresentationFragment>();
	if (LIKELY(BuildContext.IsInspectingData() == false))
	{
		RepresentationFragment.HighResTemplateActorIndex = 
			HighResTemplateActor.Get() ? RepresentationSubsystem
			->FindOrAddTemplateActor(HighResTemplateActor.Get()) : INDEX_NONE;
		RepresentationFragment.LowResTemplateActorIndex = 
			LowResTemplateActor.Get() ? RepresentationSubsystem
			->FindOrAddTemplateActor(LowResTemplateActor.Get()) : INDEX_NONE;
	}

	bool bStaticMeshDescriptionValid = StaticMeshInstanceDesc.IsValid();
	if (bStaticMeshDescriptionValid)
	{
		if (bRegisterStaticMeshDesc && !BuildContext.IsInspectingData())
		{
			RepresentationFragment.StaticMeshDescHandle = RepresentationSubsystem->FindOrAddStaticMeshDesc(StaticMeshInstanceDesc);
			ensureMsgf(RepresentationFragment.StaticMeshDescHandle.IsValid()
				, TEXT("Expected to get a valid StaticMeshDescHandle since we already checked that StaticMeshInstanceDesc is valid"));
			// if the unexpected happens and StaticMeshDescHandle is not valid we're going to treat it as if StaticMeshInstanceDesc
			// was not valid in the first place and handle it accordingly in a moment
			bStaticMeshDescriptionValid = RepresentationFragment.StaticMeshDescHandle.IsValid();
		}
	}

	FConstSharedStruct ParamsFragment;
	if (bStaticMeshDescriptionValid)
	{
		ParamsFragment = EntityManager.GetOrCreateConstSharedFragment(Params);
	}
	else
	{
		FMassRepresentationParameters ParamsCopy = Params;
		SanitizeParams(ParamsCopy, bStaticMeshDescriptionValid);
		ParamsFragment = EntityManager.GetOrCreateConstSharedFragment(ParamsCopy);
	}
	ParamsFragment.Get<const FMassRepresentationParameters>().ComputeCachedValues();
	BuildContext.AddConstSharedFragment(ParamsFragment);

#if HOTTA_ENGINE_MODIFY // add by wujingjing
	bool bIsMobilePlatform = false;
	
#if PLATFORM_IOS || PLATFORM_ANDROID || PLATFORM_OPENHARMONY
	bIsMobilePlatform = true;
#endif
	
#if WITH_EDITOR
	// 编辑器下预览移动端配置
	bool UseMobileConfig = IConsoleManager::Get().FindConsoleVariable(TEXT("HTMassSpawnerBase.GMUseMobileConfig"))->GetBool();
	bIsMobilePlatform = bIsMobilePlatform && UseMobileConfig;
#endif
	
	FConstSharedStruct LODParamsFragment = EntityManager.GetOrCreateConstSharedFragment(bIsMobilePlatform&&bUseSpecificLODParamsForMoilePlatform ? MobileLODParams : LODParams);
	BuildContext.AddConstSharedFragment(LODParamsFragment);
#else
	FConstSharedStruct LODParamsFragment = EntityManager.GetOrCreateConstSharedFragment(LODParams);
	BuildContext.AddConstSharedFragment(LODParamsFragment);
#endif

	FSharedStruct LODSharedFragment = EntityManager.GetOrCreateSharedFragment<FMassVisualizationLODSharedFragment>(FConstStructView::Make(LODParams), LODParams);
	BuildContext.AddSharedFragment(LODSharedFragment);

	BuildContext.AddFragment<FMassRepresentationLODFragment>();
	BuildContext.AddTag<FMassVisibilityCulledByDistanceTag>();
	BuildContext.AddChunkFragment<FMassVisualizationChunkFragment>();

	BuildContext.AddTag<FMassVisualizationLODProcessorTag>();
	BuildContext.AddTag<FMassVisualizationProcessorTag>();
}
```