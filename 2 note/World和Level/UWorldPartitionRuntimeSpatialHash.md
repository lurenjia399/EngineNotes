# ForEachStreamingCellsSources
```cpp
void UWorldPartitionRuntimeSpatialHash::ForEachStreamingCellsSources(
	const TArray<FWorldPartitionStreamingSource>& Sources, // 流送的StreamingSource
	TFunctionRef<bool(const UWorldPartitionRuntimeCell*, 
		EStreamingSourceTargetState)> Func, // 执行的回调
	const FWorldPartitionStreamingContext& InContext) const // 传进来的上下文，里面有数据层的激活信息
{
	// 1 如果上下文没有，就创建一个新的上下文
	const FWorldPartitionStreamingContext StackContext = 
		!InContext.IsValid() ? 
			FWorldPartitionStreamingContext::Create(GetTypedOuter<UWorld>()) 
			: FWorldPartitionStreamingContext();
	// 2 如果没有StreamingSource来流送
	if (Sources.Num() == 0)
	{
	}
	/*
	3 如果有StreamingSource来触发流送
	3.1 遍历WP上的所有Grid，每一个Gird都执行GetCells方法
	3.2 GetCells方法就是对Grid上的每个Level都于Source形状相交，得到相交的FGridCellCoord（也就是找到每个Level与Source相交的Cell的xy坐标），根据xy坐标从Level上找到RuntimeCell
	3.3 找到RuntimeCell之后，会从InContext上查找Cell配置的数据层运行时的状态，根据状态来填充到LoadStreamingSourceCells，和ActivateStreamingSourceCells数组中
	*/
	else
	{
		ForEachStreamingGrid([&](const FSpatialHashStreamingGrid& StreamingGrid)
		{
			if (IsCellRelevantFor(StreamingGrid.bClientOnlyVisible))
			{
				StreamingGrid.GetCells(Sources, ActivateStreamingSourceCells,
					 LoadStreamingSourceCells, 
					 GetEffectiveEnableZCulling(bEnableZCulling), Context);					
			}
		});
	}
	/*
	4 处理LoadStreamingSourceCells，和ActivateStreamingSourceCells数组
	*/
	for (const UWorldPartitionRuntimeCell* Cell : ActivateStreamingSourceCells.GetCells())
	{
		if (!Func(Cell, EStreamingSourceTargetState::Activated))
		{
			return;
		}
	}
	for (const UWorldPartitionRuntimeCell* Cell : LoadStreamingSourceCells.GetCells())
	{
		if (!Func(Cell, EStreamingSourceTargetState::Loaded))
		{
			return;
		}
	}
}
```

# SetupHLODActors
```cpp
bool UWorldPartitionRuntimeSpatialHash::SetupHLODActors(
	const IStreamingGenerationContext* StreamingGenerationContext, 
	const UWorldPartition::FSetupHLODActorsParams& Params) const
{
	// 1 遍历WP中所有的ActorDesc，找到配置的所有HLODLayer，没配就是默认的
	TMap<UHLODLayer*, int32> HLODLayersLevels = 
		GatherHLODLayers(StreamingGenerationContext, WorldPartition);
	TArray<UHLODLayer*> HLODLayers;
	HLODLayersLevels.GetKeys(HLODLayers);
	
	// 2 通过GCObject强引用HLODLayers，防止GC掉
	TGCObjectsScopeGuard<UHLODLayer> KeepHLODLayersAlive(HLODLayers);
	
	// 3 生成HLOD的RuntimeGrid，key是GridCell的名称，value是GridCell
	TMap<FName, FSpatialHashRuntimeGrid> HLODGrids = 
		CreateHLODGrids(HLODLayersLevels);
	
	// 4 缓存Grid的名称，key是Grid名称，vlaue是此Grid在Grid数组中索引
	TMap<FName, int32> GridsMapping;
	GridsMapping.Add(NAME_None, 0);
	for (int32 i = 0; i < Grids.Num(); i++)
	{
		const FSpatialHashRuntimeGrid& Grid = Grids[i];
		check(!GridsMapping.Contains(Grid.GridName));
		GridsMapping.Add(Grid.GridName, i);
	}
	
	/*
	5.1 核心方法，遍历Grid，对相应的ActorSet都执行生成HLOD方法
	5.2 生成方法是GenerateHLODActorsForGrid方法
	5.3 方法中首先判断是否能生成HLOD，
	*/
	for (int32 GridIndex = 0; GridIndex < Grids.Num(); GridIndex++)
	{
		GenerateHLODActors(Grids[GridIndex], 0, GridActorSetInstances[GridIndex]);
	}
}
```