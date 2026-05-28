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
	
	// 如果开启了异步线程，这里就变成同步的等异步线程结束
	if (WaitForAsyncUpdateStreamingState())
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(UWorldPartitionStreamingPolicy::UpdateTargetStateFromAsyncTask);

		check(AsyncUpdateStreamingStateTask.IsCompleted());
		PostUpdateStreamingStateInternal_GameThread(AsyncTaskTargetState);

		// 把TaskTargetState中的Cell放到TargetState里面
		for (const UWorldPartitionRuntimeCell* Cell : AsyncTaskTargetState.ToActivateCells)
		{
			if (!CurrentState.ActivatedCells.Contains(Cell))
			{
				TargetState.ToActivateCells.Add(Cell);
			}
		}
		for (const UWorldPartitionRuntimeCell* Cell : AsyncTaskTargetState.ToLoadCells)
		{
			if (!CurrentState.LoadedCells.Contains(Cell))
			{
				TargetState.ToLoadCells.Add(Cell);
			}
		}

		// 清掉数据
		// Reset everything related to last asynchronous tasks
		check(AsyncUpdateTaskState == EAsyncUpdateTaskState::Started);
		AsyncUpdateTaskState = EAsyncUpdateTaskState::None;
		AsyncUpdateStreamingStateTask = UE::Tasks::TTask<void>();
		AsyncTaskCurrentState.Reset();
		AsyncTaskTargetState.Reset();
	}
	// 一样的东西，依然更新StreamingSource
	UpdateStreamingSources(bCanOptimizeUpdate);
	
	// 开启了异步加载的话，就改变状态
	if (bCanUpdateAsync)
	{
		AsyncUpdateTaskState = EAsyncUpdateTaskState::Pending;
	}
	else
	{
		TargetState.Reset();
		UWorldPartitionStreamingPolicy::UpdateStreamingStateInternal(
			FUpdateStreamingStateParams(this, CurrentState), TargetState);
		PostUpdateStreamingStateInternal_GameThread(TargetState);
	}
}
```

# PostUpdateStreamingStateInternal_GameThread
```cpp
void UWorldPartitionStreamingPolicy::PostUpdateStreamingStateInternal_GameThread(FWorldPartitionUpdateStreamingTargetState& InOutTargetState)
{
	// 清掉没有加载的Cell
	if (!InOutTargetState.ToUnloadCells.IsEmpty())
	{
		SetCellsStateToUnloaded(InOutTargetState.ToUnloadCells);
		InOutTargetState.ToUnloadCells.Reset();
	}
	

}
```