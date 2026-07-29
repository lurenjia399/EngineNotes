# Start
```cpp
EStateTreeRunStatus FStateTreeExecutionContext::Start(FStartParameters Parameters)
{
	// 1 从InstanceStorage中获取树的执行状态。InstanceStorage是每个Entity独有的在UMassStateTreeActivationProcessor这个里创建出的数据空间，并且是通过MakeShare出来的数据存放在UMassStateTreeSubsystem的InstanceDataArray数组中，通过SubSystem一直保持持有不析构
	FStateTreeExecutionState& Exec = GetExecState();
	// 2 判断树的执行状态，如果还在运行就Stop。
	if (Exec.TreeRunStatus == EStateTreeRunStatus::Running)
	{
		Stop();
	}
	//3 InstanceData就是每个Entity所独有的数据。这里重置因为是Start方法
	InstanceData.Reset();
	// 4 通过移动语义把参数中ExecutionExtension移动到树的执行状态中，并把EventQueue放到InstanceData中
	Exec.ExecutionExtension = MoveTemp(Parameters.ExecutionExtension);
	if (Parameters.SharedEventQueue)
	{
		InstanceData.SetSharedEventQueue(
			Parameters.SharedEventQueue.ToSharedRef());
	}
	// 5 如果参数中没有全局Parameters，就往执行上下文中设置默认的
	if (!Parameters.GlobalParameters || 
		!SetGlobalParameters(*Parameters.GlobalParameters))
	{
		SetGlobalParameters(RootStateTree.GetDefaultParameters());
	}
	// 6 保护bAllowedToScheduleNextTick值，在Start方法结束后恢复。表示从这里开始不能执行Rick
	TGuardValue<bool> ScheduledNextTickScope(bAllowedToScheduleNextTick, false);
	/*
		7.1 向树的执行状态中新加一个执行帧
		7.2 赋值执行帧的FrameID，是一个从0递增的计数器，计数器存在实例中，也就是Entity独有
		7.3 赋值StateTree，是一个指针，就是StateTree的指针
		7.4 赋值根节点RootState
		7.5 赋值ActiveStates，初始化下
		7.6 标记bIsGlobalFrame为true，表示这一个执行帧需要执行GlobalTask
	*/
	FStateTreeExecutionFrame& InitFrame = Exec.ActiveFrames.AddDefaulted_GetRef();
	InitFrame.FrameID = UE::StateTree::FActiveFrameID(Storage.GenerateUniqueId());
	InitFrame.StateTree = &RootStateTree;
	InitFrame.RootState = FStateTreeStateHandle::Root;
	InitFrame.ActiveStates = {};
	InitFrame.bIsGlobalFrame = true;
	// 8 在StateTree的Frames数组中寻找根Frame，也就是寻找根索引是Root的主树，因为子树的跟索引是节点索引，并返回主树的Frame
	const FCompactStateTreeFrame* FrameInfo = 
		RootStateTree.GetFrameFromHandle(FStateTreeStateHandle::Root);
	// 9 赋值执行帧的ActiveTasksStatus，表示在stateTree中激活的是哪个Frame
	InitFrame.ActiveTasksStatus = FrameInfo ? 
		FStateTreeTasksCompletionStatus(*FrameInfo) 
			: FStateTreeTasksCompletionStatus();
	// 10 更新每个Entity身上的InstanceData数据，把StateTree中的GobalTask，Evaluator中记录的InstanceData结构体都填充到Entity身上初始化占位。因为这里是Start方法所以没有激活的State，先不用添加State中的Task
	UpdateInstanceData({}, Exec.ActiveFrames);
	// 11 在数的执行状态中初始化一个随机种子
	Exec.RandomStream.Initialize(Parameters.RandomSeed.IsSet() 
		? Parameters.RandomSeed.GetValue() : FPlatformTime::Cycles());
	// 12 将数的执行状态中的CurrentPhase变量设置成StartTree。表示所处的当前状态是StartTree
	SetUpdatePhaseInExecutionState(Exec, EStateTreeUpdatePhase::StartTree);
	// 13 如果是GlobalFrame，就会执行Evaluator的TreeStart方法，执行GlobalTask的EnterState方法，并返回EnterState的结果作为GlobalTasksRunStatus
	const EStateTreeRunStatus GlobalTasksRunStatus = 
		StartEvaluatorsAndGlobalTasks(LastInitializedTaskIndex);
	// 14 如果GobalTask执行的结果是Running，或者是没有GobalTask
	if (GlobalTasksRunStatus == EStateTreeRunStatus::Running)
	{
		// 15 Tick一次Evaluator，但是不TickGlobalTask
		constexpr bool bTickGlobalTasks = false;
		TickEvaluatorsAndGlobalTasks(0.0f, bTickGlobalTasks);
		// 16 设置数的执行状态中的其他变量，表示数正在运行，上一次的Tick结果是UnSet
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

# TickTask
```cpp
EStateTreeRunStatus FStateTreeExecutionContext::TickTasks(const float DeltaTime)
{
	if (TickArgs.Frame->bIsGlobalFrame)
	{
		// TickEvaluatorsAndGlobalTasksForFrame
		constexpr bool bTickGlobalTasks = true;
		const EStateTreeRunStatus FrameResult = 
			TickEvaluatorsAndGlobalTasksForFrame(DeltaTime, bTickGlobalTasks,
				FrameIndex, TickArgs.ParentFrame, TickArgs.Frame);
		if (FrameResult != EStateTreeRunStatus::Running)
		{
			TickArgs.bShouldTickTasks = false;
			break;
		}
	}
	// 遍历state链，依次Tick
	for (int32 StateIndex = 0; StateIndex < 
		TickArgs.Frame->ActiveStates.Num(); ++StateIndex)
	{
		if (bCopyBoundPropertiesOnNonTickedTask || 
			CurrentState.ShouldTickTasks(bHasEvents))
		{
			const FTickTaskResult TickTasksResult = TickTasks(TickArgs);
		}
	}
}
```

# SelectState
```cpp
bool FStateTreeExecutionContext::SelectState(
	const FStateTreeExecutionFrame& CurrentFrame,// 从CurrentFrame选择State，Frame就代表一棵树
	const FStateTreeStateHandle NextState,// 开始遍历的NextState
	FStateSelectionResult& OutSelectionResult,
	const FStateTreeSharedEvent* TransitionEvent,
	const EStateTreeSelectionFallback Fallback)
{
	// 1 遍历的NextState链路，缓存从根节点出来到NextState的路径。CurrState表示根节点
	TArray<FStateTreeStateHandle, 
		TInlineAllocator<FStateTreeActiveStates::MaxStates>> PathToNextState;
	FStateTreeStateHandle CurrState = NextState;
	while (CurrState.IsValid())
	{
		if (PathToNextState.Num() == FStateTreeActiveStates::MaxStates)
		{
			return false;
		}
		PathToNextState.Push(CurrState);
		CurrState = CurrentFrame.StateTree->States[CurrState.Index].Parent;
	}
	Algo::Reverse(PathToNextState);
	// 2 赋值CurrentStateTreeIndex，表示激活帧中和参数使用的同一个statetree资源的帧索引。赋值CurrentFrameIndex，表示同一资源帧中相同的rootState的帧索引。
	int32 CurrentFrameIndex = INDEX_NONE;
	int32 CurrentStateTreeIndex = INDEX_NONE;
	for (int32 FrameIndex = Exec.ActiveFrames.Num() - 1; 
		FrameIndex >= 0; FrameIndex--)
	{
		const FStateTreeExecutionFrame& Frame = Exec.ActiveFrames[FrameIndex]; 
		if (Frame.StateTree == NextStateTree)
		{
			CurrentStateTreeIndex = FrameIndex;
			if (Frame.RootState == NextRootState)
			{
				CurrentFrameIndex = FrameIndex;
				break;
			}
		}
	}
	// 3 赋值CurrentFrameInActiveFrames，优先选CurrentFrameIndex，如果使用同一资源并且相同RootState的激活帧。次选使用同一资源的激活帧。
	const FStateTreeExecutionFrame* CurrentFrameInActiveFrames  = nullptr;
	if (CurrentFrameIndex != INDEX_NONE)
	{
		const int32 NumCommonFrames = CurrentFrameIndex + 1;
		OutSelectionResult = FStateSelectionResult(MakeArrayView(Exec.ActiveFrames.GetData(), NumCommonFrames));
		CurrentFrameInActiveFrames  = &Exec.ActiveFrames[CurrentFrameIndex];
	}
	else if (CurrentStateTreeIndex != INDEX_NONE)
	{
		const int32 NumCommonFrames = CurrentStateTreeIndex + 1;
		OutSelectionResult = FStateSelectionResult(MakeArrayView(Exec.ActiveFrames.GetData(), NumCommonFrames));
		CurrentFrameInActiveFrames  = &Exec.ActiveFrames[CurrentStateTreeIndex];
	}
	// 4 赋值FirstNewStateIndex，表示当前激活帧中激活链和目标节点链，第一次分叉的节点是哪个
	int32 FirstNewStateIndex = 0;
	if (CurrentFrameIndex != INDEX_NONE)
	{
		FirstNewStateIndex = FMath::Max(0, FMath::Min(PathToNextState.Num(), LastFrame.ActiveStates.Num()) - 1);
		for (int32 Index = 0; Index < FMath::Min(PathToNextState.Num(), LastFrame.ActiveStates.Num()); ++Index)
		{
			if (LastFrame.ActiveStates[Index] != PathToNextState[Index])
			{
				FirstNewStateIndex = Index;
				break;
			}
		}
	}
	/*
	5.1 执行SelectStateInternal方法，dfs遍历，遍历到叶子节点，构建选择链，
	*/
	if (SelectStateInternal(
		CurrentParentFrame, 
		OutSelectionResult.GetSelectedFrames()[LastFrameIndex],
		CurrentFrameInActiveFrames, 
		NewStatesPathToNextState, 
		OutSelectionResult, 
		TransitionEvent))
	{
		return true;
	}
}
```
# TriggerTransitions
```cpp
bool FStateTreeExecutionContext::TriggerTransitions()
{
	// 遍历state链上的所有Transition
	for (const FTransitionHandler& Handler : TransitionHandlers)
	{
		for (uint8 TransitionCounter = 0; 
			TransitionCounter < State.TransitionsNum; ++TransitionCounter)
		{
			// 如果触发类型是OnEvent，如果队列有所需的Event，就添加
			if (Transition.Trigger == EStateTreeTransitionTrigger::OnEvent)
			{
				TConstArrayView<FStateTreeSharedEvent> EventsQueue = 
					GetEventsToProcessView();
				for (const FStateTreeSharedEvent& Event : EventsQueue)
				{
					if (Transition.RequiredEvent.DoesEventMatchDesc(*Event))
					{
						TransitionEvents.Emplace(&Event);
					}
				}
			}
			// Tick触发的，直接添加
			else if (EnumHasAnyFlags(Transition.Trigger, 
				EStateTreeTransitionTrigger::OnTick))
			{
				TransitionEvents.Emplace(nullptr);
			}
			// delegate触发的，如果已经广播了直接添加
			else if (EnumHasAnyFlags(Transition.Trigger, 
				EStateTreeTransitionTrigger::OnDelegate))
			{
				if (Storage.
					IsDelegateBroadcasted(Transition.RequiredDelegateDispatcher))
				{
					TransitionEvents.Emplace(nullptr);
				}
			}
			// 遍历Trasnsition，依次执行RequestTransition
			for (const FStateTreeSharedEvent* TransitionEvent : TransitionEvents)
			{
				bool bPassed = false; 
				{
					bPassed = TestAllConditions(CurrentParentFrame, 
						CurrentFrame, Transition.ConditionsBegin, 
						Transition.ConditionsNum);
				}
				if (bPassed)
				{
					if (RequestTransition(
						CurrentFrame, Transition.State, 
						Transition.Priority, TransitionEvent,
						Transition.Fallback))
					{
						break;
					}
				}
			}
		}
	}
}
```

# Tick
```cpp
EStateTreeRunStatus FStateTreeExecutionContext::Tick(const float DeltaTime)
{
	// bAllowedToScheduleNextTick临时设置成false，在tick结束回滚
	TGuardValue<bool> ScheduledNextTickScope(bAllowedToScheduleNextTick, false);
	
	const EStateTreeRunStatus PreludeResult = TickPrelude();
	if (PreludeResult != EStateTreeRunStatus::Running)
	{
		return PreludeResult;
	}
	TickUpdateTasksInternal(DeltaTime);
	TickTriggerTransitionsInternal();
	return TickPostlude();
}
```


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