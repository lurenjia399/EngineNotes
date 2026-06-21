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

```