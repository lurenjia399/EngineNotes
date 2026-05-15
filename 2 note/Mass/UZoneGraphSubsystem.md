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
0 在编辑器下重新BuildZoneGraph会遍历地图中所有的AZoneGraphData，先后响应注册进去。在运行时每个AZoneGraphData的PostLoad里也会注册（PostLoad会从地图序列化出来的时候执行）。
1 UZoneGraphSubsystem里的RegisteredZoneGraphData数组是存储场景中所有的ZoneGraphData
2 将地图中的ZoneGraphData和UZoneGraphSubsystem里的RegisteredZoneGraphData数组相关联，双方都存下互相的引用，通过FZoneGraphDataHandle来实现
3 RegisteredZoneGraphData这个数组里的元素也是复用的，不会反复的创建删除。每次UnRegister会通过ZoneGraphDataFreeList来记录数组里闲着的位置，下次Register就会使用。其中GeneRation这个变量就表示复用的次数。

FZoneGraphDataHandle UZoneGraphSubsystem::RegisterZoneGraphData(
	AZoneGraphData& InZoneGraphData)
{
	// 空闲队列有就取，没有就在添加一个，返回他的索引
	const int32 Index = 
		(ZoneGraphDataFreeList.Num() > 0) ? 
		ZoneGraphDataFreeList.Pop(EAllowShrinking::No) : 
		RegisteredZoneGraphData.AddDefaulted();
	// 从索引中拿出一个
	FRegisteredZoneGraphData& RegisteredData = RegisteredZoneGraphData[Index];
	// 重置数据，Generation这个是使用次数
	RegisteredData.Reset(RegisteredData.Generation); // Do not change generation.
	RegisteredData.ZoneGraphData = &InZoneGraphData;
	RegisteredData.bInUse = true;
	// 组装handle，FZoneGraphDataHandle(索引，索引出的使用次数来标识唯一)
	const FZoneGraphDataHandle ResultHandle = 
	FZoneGraphDataHandle(uint16(Index), uint16(RegisteredData.Generation));
	// 注册，将handle设置到AZoneGraphData中
	InZoneGraphData.OnRegistered(ResultHandle);
	// 广播通知
	UE::ZoneGraphDelegates::OnPostZoneGraphDataAdded.Broadcast(
		RegisteredData.ZoneGraphData);

	return ResultHandle;
}


*/
```
# 3 OnRequestRebuild
```cpp
void UZoneGraphSubsystem::OnRequestRebuild()
{
	// 重新构建ZoneGraph
	RebuildGraph(true /*bForceRebuild*/);

	// Force update viewport
	if (GCurrentLevelEditingViewportClient)
	{
		GCurrentLevelEditingViewportClient->Invalidate();
	}
}

void UZoneGraphSubsystem::RebuildGraph(const bool bForceRebuild)
{
	
	UnregisterStaleZoneGraphDataInstances();
	// 这个里面就是遍历World里的所有ZoneGraphData
	RegisterZoneGraphDataInstances();
	SpawnMissingZoneGraphData();

	TArray<AZoneGraphData*> AllZoneGraphData;
	AllZoneGraphData.Reserve(RegisteredZoneGraphData.Num());

	for (const FRegisteredZoneGraphData& RegisteredData : RegisteredZoneGraphData)
	{
		if (RegisteredData.ZoneGraphData)
		{
			AllZoneGraphData.Add(RegisteredData.ZoneGraphData);
		}
	}

	Builder.BuildAll(AllZoneGraphData, bForceRebuild);
}
```
在编辑器点击BuildZoneGraph后，就是重新构建路网
1 会首先从world中找所有的 AZoneGraphData
2 然后会遍历所有的ZoneGraphData，把Builder里所有的ZoneShapeComp数据都存到ZoneGraphData里的ZoneStorage里
3 ZoneShapeComp的OnRegister方法里就会把自己注册到Builder里面