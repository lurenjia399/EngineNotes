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
FZoneGraphDataHandle UZoneGraphSubsystem::RegisterZoneGraphData(AZoneGraphData& InZoneGraphData)
{
	if (!IsValid(&InZoneGraphData))
	{
		return FZoneGraphDataHandle();
	}

	if (InZoneGraphData.IsRegistered())
	{
		UE_LOG(LogZoneGraph, Error, TEXT("Trying to register already registered ZoneGraphData \'%s\'"), *InZoneGraphData.GetName());
		return FZoneGraphDataHandle();
	}

	FScopeLock Lock(&DataRegistrationSection);

	if (RegisteredZoneGraphData.FindByPredicate([&InZoneGraphData](const FRegisteredZoneGraphData& Item) { return Item.bInUse && Item.ZoneGraphData == &InZoneGraphData; }) != nullptr)
	{
		UE_LOG(LogZoneGraph, Error, TEXT("ZoneGraphData \'%s\' already exists in RegisteredZoneGraphData."), *InZoneGraphData.GetName());
		return FZoneGraphDataHandle();
	}

	const int32 Index = (ZoneGraphDataFreeList.Num() > 0) ? ZoneGraphDataFreeList.Pop(EAllowShrinking::No) : RegisteredZoneGraphData.AddDefaulted();
	FRegisteredZoneGraphData& RegisteredData = RegisteredZoneGraphData[Index];
	RegisteredData.Reset(RegisteredData.Generation); // Do not change generation.
	RegisteredData.ZoneGraphData = &InZoneGraphData;
	RegisteredData.bInUse = true;
	check(Index < int32(MAX_uint16));
	const FZoneGraphDataHandle ResultHandle = FZoneGraphDataHandle(uint16(Index), uint16(RegisteredData.Generation));

	InZoneGraphData.OnRegistered(ResultHandle);

	UE::ZoneGraphDelegates::OnPostZoneGraphDataAdded.Broadcast(RegisteredData.ZoneGraphData);

	return ResultHandle;
}
*/
```