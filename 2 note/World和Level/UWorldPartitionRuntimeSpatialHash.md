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
# GetStreamingPerformance
```cpp
EWorldPartitionStreamingPerformance 
	UWorldPartitionRuntimeHash::GetStreamingPerformance(
	const TSet<const UWorldPartitionRuntimeCell*>& CellsToActivate, 
	bool& bOutShouldBlock) const
{
	
}
```