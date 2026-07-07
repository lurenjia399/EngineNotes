# 
# SelectionBehavior
```cpp
enum class EStateTreeStateSelectionBehavior : uint8
{
	/** The State cannot be directly selected. */
	None,
	
	// 直接进入这个节点，不管有没有子节点
	TryEnterState UMETA(DisplayName = "Try Enter"),
	// 按找子节点的顺序挑选进入
	TrySelectChildrenInOrder UMETA(DisplayName = "Try Select Children In Order"),
	// 将子节点随机选一个
	TrySelectChildrenAtRandom UMETA(DisplayName = "Try Select Children At Random"),
	// 从子节点中选择Utility计算出最高的
	TrySelectChildrenWithHighestUtility UMETA(DisplayName = "Try Select Children With Highest Utility"),
	// 根据效用分数加权随机选择子状态
	TrySelectChildrenAtRandomWeightedByUtility UMETA(DisplayName = "Try Select Children At Random Weighted By Utility"),

	// 触发节点切换，而不是选择子节点
	TryFollowTransitions UMETA(DisplayName = "Try Follow Transitions"),
};
```

# Start
```cpp
EStateTreeRunStatus FStateTreeExecutionContext::Start(FStartParameters Parameters)
{
	// 1 获取实例
	FStateTreeExecutionState& Exec = GetExecState();
	if (!Exec.CurrentPhase == EStateTreeUpdatePhase::Unset)
	{
		return EStateTreeRunStatus::Failed;
	}
	// 2 如果之前还在运行就停止
	if (Exec.TreeRunStatus == EStateTreeRunStatus::Running)
	{
		Stop();
	}
	// 3 添加一个ActiveFrame
	FStateTreeExecutionFrame& InitFrame = Exec.ActiveFrames.AddDefaulted_GetRef();
	InitFrame.FrameID = UE::StateTree::FActiveFrameID(Storage.GenerateUniqueId());
	InitFrame.StateTree = &RootStateTree;
	InitFrame.RootState = FStateTreeStateHandle::Root;
	InitFrame.ActiveStates = {};
	InitFrame.bIsGlobalFrame = true;
	// 4
	UpdateInstanceData({}, Exec.ActiveFrames);
	// 5 
	SetUpdatePhaseInExecutionState(Exec, EStateTreeUpdatePhase::StartTree);
	// 6 Evaluator执行TreeStart，GlobalTask执行EnterState
	const EStateTreeRunStatus GlobalTasksRunStatus = 
		StartEvaluatorsAndGlobalTasks(LastInitializedTaskIndex);
	if (GlobalTasksRunStatus == EStateTreeRunStatus::Running)
	{
		// 7 执行Evaluator的tick
		constexpr bool bTickGlobalTasks = false;
		TickEvaluatorsAndGlobalTasks(0.0f, bTickGlobalTasks);
		// 8 设置运行状态，上一次tick状态为Unset
		Exec.TreeRunStatus = EStateTreeRunStatus::Running;
		Exec.LastTickStatus = EStateTreeRunStatus::Unset;
		// 9 从根节点开始SelectState，dfs遍历，遍历到叶子节点，构建选择链，会调用TestCondition方法，判断能否进入选择链
		FStateSelectionResult StateSelectionResult;
		if (SelectState(InitFrame, RootState, StateSelectionResult))
		{
			// 如果叶子状态完成了，就标记statetree完成
			if (StateSelectionResult.GetSelectedFrames()
				.Last().ActiveStates.Last().IsCompletionState())
			{
				Exec.TreeRunStatus = 
					StateSelectionResult.GetSelectedFrames()
					.Last().ActiveStates.Last().ToCompletionStatus();
			}
			// 叶子状态没有完成，就进入叶子状态
			else
			{
				//
				/*
				1  执行EnterState，根据选择链依次进入state，如果state有enterConditions就执行Condition的EnterState
				2 for循环依次执行state中的Task,执行Task中的EnterState方法
				*/
				const EStateTreeRunStatus LastTickStatus = EnterState(Transition);
				Exec.LastTickStatus = LastTickStatus;
				// EnterState后不是Running，就完成statetree
				if (Exec.LastTickStatus != EStateTreeRunStatus::Running)
				{
					StateCompleted();
				}
			}
		}
	}
}
```