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

	// 相同entity之间共享的FMassRepresentationParameters
	bool bStaticMeshDescriptionValid = StaticMeshInstanceDesc.IsValid();
	if (bStaticMeshDescriptionValid)
	{
		if (bRegisterStaticMeshDesc && !BuildContext.IsInspectingData())
		{
			RepresentationFragment.StaticMeshDescHandle = RepresentationSubsystem->FindOrAddStaticMeshDesc(StaticMeshInstanceDesc);
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

	// 设置Lod的Fragment，设置成共享的，相同entity之间共享
	FConstSharedStruct LODParamsFragment = 
		EntityManager.GetOrCreateConstSharedFragment(LODParams);
	BuildContext.AddConstSharedFragment(LODParamsFragment);


	FSharedStruct LODSharedFragment = EntityManager.GetOrCreateSharedFragment<FMassVisualizationLODSharedFragment>(FConstStructView::Make(LODParams), LODParams);
	BuildContext.AddSharedFragment(LODSharedFragment);

	BuildContext.AddFragment<FMassRepresentationLODFragment>();
	BuildContext.AddTag<FMassVisibilityCulledByDistanceTag>();
	BuildContext.AddChunkFragment<FMassVisualizationChunkFragment>();

	BuildContext.AddTag<FMassVisualizationLODProcessorTag>();
	BuildContext.AddTag<FMassVisualizationProcessorTag>();
}
```