# 1 构建ZoneGraph

```cpp
/*
// 编辑器每次点击Build之后执行 OnRequestRebuild 方法

void UZoneGraphSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
	Super::Initialize(Collection);

	// 编辑器下注册监听，每次点击Build之后会执行OnRequestRebuild这个方法重新构建
#if WITH_EDITOR
	OnActorMovedHandle = GEngine->OnActorMoved()
		.AddUObject(this, &UZoneGraphSubsystem::OnActorMoved);
	OnRequestRebuildHandle = UE::ZoneGraphDelegates::OnZoneGraphRequestRebuild
		.AddUObject(this, &UZoneGraphSubsystem::OnRequestRebuild);
#endif

	// 
	UE::Mass::Subsystems::RegisterSubsystemType(
	Collection, 
	GetClass(), 
	UE::Mass::FSubsystemTypeTraits::Make<UZoneGraphSubsystem>());

	bInitialized = true;
}
*/
```

# 2 RegisterZoneGraphData
```cpp
/*
0 在编辑器下重新BuildZoneGraph会遍历地图中所有的AZoneGraphData，先后响应注册进去。在运行时每个AZoneGraphData的PostLoad里也会注册（PostLoad会在）。
1 UZoneGraphSubsystem里的RegisteredZoneGraphData数组是存储场景中所有的ZoneGraphData
2 将地图中的ZoneGraphData和UZoneGraphSubsystem里的RegisteredZoneGraphData数组相关联，双方都存下互相的引用，通过FZoneGraphDataHandle来实现

FZoneGraphDataHandle UZoneGraphSubsystem::RegisterZoneGraphData(
	AZoneGraphData& InZoneGraphData)
{
	const int32 Index = 
	(ZoneGraphDataFreeList.Num() > 0) ? 
		ZoneGraphDataFreeList.Pop(EAllowShrinking::No) : 
		RegisteredZoneGraphData.AddDefaulted();
	FRegisteredZoneGraphData& RegisteredData = RegisteredZoneGraphData[Index];
	RegisteredData.Reset(RegisteredData.Generation); // Do not change generation.
	RegisteredData.ZoneGraphData = &InZoneGraphData;
	RegisteredData.bInUse = true;
	check(Index < int32(MAX_uint16));
	const FZoneGraphDataHandle ResultHandle = FZoneGraphDataHandle(uint16(Index), uint16(RegisteredData.Generation));

	// 
	InZoneGraphData.OnRegistered(ResultHandle);

	UE::ZoneGraphDelegates::OnPostZoneGraphDataAdded.Broadcast(
		RegisteredData.ZoneGraphData);

	return ResultHandle;
}
*/
```