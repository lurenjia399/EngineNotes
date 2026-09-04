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
	// 2 如果传进来的StreamingSource
	if (Sources.Num() == 0)
	{
	}
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
}
```