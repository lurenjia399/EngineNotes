# 1 构建ZoneGraph

```cpp
/*
// 编辑器每次点击Build之后执行 OnRequestRebuild 方法

void UZoneGraphSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
	Super::Initialize(Collection);

	// 编辑器下zhu'c
#if WITH_EDITOR
	OnActorMovedHandle = GEngine->OnActorMoved()
		.AddUObject(this, &UZoneGraphSubsystem::OnActorMoved);
	OnRequestRebuildHandle = UE::ZoneGraphDelegates::OnZoneGraphRequestRebuild
		.AddUObject(this, &UZoneGraphSubsystem::OnRequestRebuild);
#endif

	UE::Mass::Subsystems::RegisterSubsystemType(
	Collection, 
	GetClass(), 
	UE::Mass::FSubsystemTypeTraits::Make<UZoneGraphSubsystem>());

	bInitialized = true;
}
*/
```