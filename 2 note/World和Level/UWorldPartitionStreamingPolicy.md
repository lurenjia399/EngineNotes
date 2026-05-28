# UpdateStreamingState
```cpp
void UWorldPartitionStreamingPolicy::UpdateStreamingState()
{
	// 增加更新的次数
	++UpdateStreamingStateCounter;
	
	// 重置变量
	ProcessedToLoadCells = 0;
	ProcessedToActivateCells = 0;
	TargetState.Reset();
}
```