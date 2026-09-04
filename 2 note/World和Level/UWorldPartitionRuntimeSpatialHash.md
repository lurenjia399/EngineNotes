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
	if (Sources.Num() == 0)
	{
		// Get always loaded cells
		ForEachStreamingGrid([&] (const FSpatialHashStreamingGrid& StreamingGrid)
		{
#if HOTTA_ENGINE_MODIFY && UE_SERVER
			if (IsCellRelevantFor(StreamingGrid.bClientOnlyVisible, StreamingGrid.bHybridModeInvisible))
#else
			if (IsCellRelevantFor(StreamingGrid.bClientOnlyVisible))
#endif
			{
				StreamingGrid.GetNonSpatiallyLoadedCells(ActivateStreamingSourceCells.GetCells(), LoadStreamingSourceCells.GetCells(), Context);
			}
		});
	}
	else
	{
}
```