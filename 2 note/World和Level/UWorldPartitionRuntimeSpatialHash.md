# ForEachStreamingCellsSources
```cpp
void UWorldPartitionRuntimeSpatialHash::ForEachStreamingCellsSources(
	const TArray<FWorldPartitionStreamingSource>& Sources, // 流送的StreamingSource
	TFunctionRef<bool(const UWorldPartitionRuntimeCell*, 
		EStreamingSourceTargetState)> Func, // 
	const FWorldPartitionStreamingContext& InContext) const
{
	
}
```