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
		// 17 从根节点开始SelectState，dfs遍历，遍历到叶子节点，构建选择链，会调用TestCondition方法，判断能否进入选择链
		FStateSelectionResult StateSelectionResult;
		if (SelectState(InitFrame, RootState, StateSelectionResult))
		{
			// 18 如果叶子状态完成了，就标记statetree完成
			if (StateSelectionResult.GetSelectedFrames()
				.Last().ActiveStates.Last().IsCompletionState())
			{
				Exec.TreeRunStatus = 
					StateSelectionResult.GetSelectedFrames()
					.Last().ActiveStates.Last().ToCompletionStatus();
			}
			// 19 叶子状态没有完成，就进入叶子状态
			else
			{
				/*
				19.1  执行EnterState，根据选择链依次进入state，如果state有enterConditions就执行Condition的EnterState
				19.2 for循环依次执行state中的Task,执行Task中的EnterState方法
				*/
				const EStateTreeRunStatus LastTickStatus = EnterState(Transition);
				Exec.LastTickStatus = LastTickStatus;
			}
		}
	}
}
```

# Tick
```cpp
EStateTreeRunStatus FStateTreeExecutionContext::Tick(const float DeltaTime)
{
	// 1 保护bAllowedToScheduleNextTick值，在Start方法结束后恢复。表示从这里开始不能执行Rick
	TGuardValue<bool> ScheduledNextTickScope(bAllowedToScheduleNextTick, false);
	// 2 
	TickUpdateTasksInternal(DeltaTime);
	TickTriggerTransitionsInternal();

	return TickPostlude();
}
```
## TickUpdateTasksInternal
```cpp
void FStateTreeExecutionContext::TickUpdateTasksInternal(float DeltaTime)
{
	//1 从树执行状态获取延迟触发的Transition，推进时间
	for (FStateTreeTransitionDelayedState& DelayedState : Exec.DelayedTransitions)
	{
		DelayedState.TimeLeft -= DeltaTime;
	}
	// 2 执行TickTask方法
	Exec.LastTickStatus = TickTasks(DeltaTime);
	if (Exec.LastTickStatus != EStateTreeRunStatus::Running 
		&& Exec.RequestedStop == EStateTreeRunStatus::Unset 
		&& PreviousTickStatus == EStateTreeRunStatus::Running)
	{
		StateCompleted();
	}
}

EStateTreeRunStatus FStateTreeExecutionContext::TickTasks(const float DeltaTime)
{
	// 1 获取树执行状态，如果当前没有激活帧就返回
	FStateTreeExecutionState& Exec = GetExecState();
	Exec.bHasPendingCompletedState = false;
	if (Exec.ActiveFrames.IsEmpty())
	{
		return EStateTreeRunStatus::Failed;
	}
	// 2 遍历激活帧
	for (int32 FrameIndex = 0; FrameIndex < Exec.ActiveFrames.Num(); ++FrameIndex)
	{
		// 2.1 如果是GlobalFrame，就会执行一次Evaluator和GlobalTask的Tick
		if (ExecutionContext::Private::bTickGlobalNodesFollowingTreeHierarchy)
		{
			if (TickArgs.Frame->bIsGlobalFrame)
			{
				constexpr bool bTickGlobalTasks = true;
				const EStateTreeRunStatus FrameResult = TickEvaluatorsAndGlobalTasksForFrame(DeltaTime, bTickGlobalTasks, FrameIndex, TickArgs.ParentFrame, TickArgs.Frame);
				if (FrameResult != EStateTreeRunStatus::Running)
				{
					if (ExecutionContext::Private::bGlobalTasksCompleteOwningFrame == false || FrameIndex == 0)
					{
						Exec.RequestedStop = ExecutionContext::GetPriorityRunStatus(Exec.RequestedStop, FrameResult);
					}
					TickArgs.bShouldTickTasks = false;
					break;
				}
			}
		}
		// 2.2 遍历激活帧当中的所有的激活状态，统计所有的状态上需要执行Task的数量。
		for (int32 StateIndex = 0; StateIndex < 
			TickArgs.Frame->ActiveStates.Num(); ++StateIndex)
		{
			const FStateTreeStateHandle CurrentHandle = TickArgs.Frame->ActiveStates[StateIndex];
			const FCompactStateTreeState& CurrentState = CurrentStateTree->States[CurrentHandle.Index];
			FTasksCompletionStatus CurrentCompletionStatus = TickArgs.Frame->ActiveTasksStatus.GetStatus(CurrentState);
			TickArgs.StateID = TickArgs.Frame->ActiveStates.StateIDs[StateIndex];
			TickArgs.TasksCompletionStatus = &CurrentCompletionStatus;
			FCurrentlyProcessedStateScope StateScope(*this, CurrentHandle);
			/*
			并且更新子树的上的参数，为什么要每帧做这件事：链接状态的子树/子资产在运行期间，其参数值可能依赖父树里会随时间变化的变量（比如黑板上的某个浮点数），所以不能只在进入状态（EnterState）时拷贝一次，需要在每次 Tick 时都重新同步一遍，保证子树看到的参数始终是最新值。这段逻辑发生在真正Tick 该状态下的任务，即参数总是先于任务被更新，保证任务读取到的是本帧最新的参数。
			*/
			if (CurrentState.Type == EStateTreeStateType::Linked || CurrentState.Type == EStateTreeStateType::LinkedAsset)
			{
				if (CurrentState.ParameterDataHandle.IsValid() && CurrentState.ParameterBindingsBatch.IsValid())
				{
					const FStateTreeDataView StateParamsDataView = GetDataView(TickArgs.ParentFrame, *TickArgs.Frame, CurrentState.ParameterDataHandle);
					CopyBatchOnActiveInstances(TickArgs.ParentFrame, *TickArgs.Frame, StateParamsDataView, CurrentState.ParameterBindingsBatch);
				}
			}
			/*
			如果需要执行TickTask，这里执行
			*/
			if (CurrentState.ShouldTickTasks(bHasEvents))
			{
				TickArgs.TasksBegin = CurrentState.TasksBegin;
				TickArgs.TasksNum = CurrentState.TasksNum;
				TickArgs.Indent = (FrameIndex + StateIndex + 1);
				const FTickTaskResult TickTasksResult = TickTasks(TickArgs);
				TickArgs.bShouldTickTasks = TickTasksResult.bShouldTickTasks
					&& !CurrentCompletionStatus.HasAnyFailed();
			}
			NumTotalEnabledTasks += CurrentState.EnabledTasksNum;
			if (bRequestLoopStop)
			{
				break;
			}
		}
	}
	// 3 更新状态，FirstFrameResult表示第一个激活帧中GlobalTask的执行状态，StateResult表示所有执行帧中每个节点的执行状态合，这个合会根据优先级排序
	EStateTreeRunStatus FirstFrameResult = EStateTreeRunStatus::Running;
	EStateTreeRunStatus FrameResult = EStateTreeRunStatus::Running;
	EStateTreeRunStatus StateResult = EStateTreeRunStatus::Running;
	for (int32 FrameIndex = 0; FrameIndex < Exec.ActiveFrames.Num(); ++FrameIndex)
	{
		using namespace UE::StateTree::ExecutionContext;

		const FStateTreeExecutionFrame& CurrentFrame = Exec.ActiveFrames[FrameIndex];
		const UStateTree* CurrentStateTree = CurrentFrame.StateTree;
		if (CurrentFrame.bIsGlobalFrame)
		{
			const ETaskCompletionStatus GlobalTasksStatus = CurrentFrame.ActiveTasksStatus.GetStatus(CurrentStateTree).GetCompletionStatus();
			if (FrameIndex == 0)
			{
				FirstFrameResult = CastToRunStatus(GlobalTasksStatus);
			}
			FrameResult = GetPriorityRunStatus(FrameResult, CastToRunStatus(GlobalTasksStatus));
		}

		for (int32 StateIndex = 0; StateIndex < CurrentFrame.ActiveStates.Num() && StateResult != EStateTreeRunStatus::Failed; ++StateIndex)
		{
			const FStateTreeStateHandle CurrentHandle = CurrentFrame.ActiveStates[StateIndex];
			const FCompactStateTreeState& State = CurrentStateTree->States[CurrentHandle.Index];
			const ETaskCompletionStatus StateTasksStatus = CurrentFrame.ActiveTasksStatus.GetStatus(State).GetCompletionStatus();
			StateResult = GetPriorityRunStatus(StateResult, CastToRunStatus(StateTasksStatus));
		}
	}
	// 4 最终判断节点是否即将完成并记录到树执行状态中，并返回树运行状态
	Exec.bHasPendingCompletedState = StateResult != EStateTreeRunStatus::Running
		 || FrameResult != EStateTreeRunStatus::Running;
	return StateResult;
}
```

## TickTriggerTransitionsInternal
```cpp
void FStateTreeExecutionContext::TickTriggerTransitionsInternal()
{
	// 1 获取树执行状态，如果当前没有激活帧就返回
	FStateTreeExecutionState& Exec = GetExecState();
	// 2 重置TriggerTransitionsFromFrameIndex数组
	TriggerTransitionsFromFrameIndex.Reset();
	// 3 for循环，循环5次
	static constexpr int32 MaxIterations = 5;
	for (int32 Iter = 0; Iter < MaxIterations; Iter++)
	{
		// 4 每次循环都清掉临时数据
		ON_SCOPE_EXIT{ InstanceData.ResetTemporaryInstances(); };
		// 5 
		if (TriggerTransitions())
		{
			UE_STATETREE_DEBUG_SCOPED_PHASE(this, EStateTreeUpdatePhase::ApplyTransitions);
			UE_STATETREE_DEBUG_TRANSITION_EVENT(this, NextTransitionSource, EStateTreeTraceEventType::OnTransition);
			NextTransitionSource.Reset();

			ExitState(NextTransition);

			// Tree succeeded or failed.
			if (NextTransition.TargetState.IsCompletionState())
			{
				// Transition to a terminal state (succeeded/failed), or default transition failed.
				Exec.TreeRunStatus = NextTransition.TargetState.ToCompletionStatus();

				// Stop evaluators and global tasks.
				StopEvaluatorsAndGlobalTasks(Exec.TreeRunStatus);

				// No active states or global tasks anymore, reset frames.
				Exec.ActiveFrames.Reset();

				RemoveAllDelegateListeners();

				break;
			}

			// Enter state tasks can fail/succeed, treat it same as tick.
			const EStateTreeRunStatus LastTickStatus = EnterState(NextTransition);

			NextTransition = FStateTreeTransitionResult();

			Exec.LastTickStatus = LastTickStatus;

			// Report state completed immediately.
			if (Exec.LastTickStatus != EStateTreeRunStatus::Running)
			{
				StateCompleted();
			}
		}

		// Stop as soon as have found a running state.
		if (Exec.LastTickStatus == EStateTreeRunStatus::Running)
		{
			break;
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
	// 1 遍历实例中的TransitionRequest
	for (const FStateTreeTransitionRequest& Request : 
		InstanceData.GetTransitionRequests())
	{
		const FStateTreeExecutionFrame* CurrentFrame = 
			Exec.FindActiveFrame(Request.SourceFrameID);
		if (CurrentFrame)
		{
			if (RequestTransition(*CurrentFrame, Request.TargetState, 
				Request.Priority, /*TransitionEvent*/nullptr, Request.Fallback))
			{
				NextTransitionSource = FStateTreeTransitionSource(
					CurrentFrame->StateTree, 
					EStateTreeTransitionSourceType::ExternalRequest, 
					Request.TargetState, Request.Priority);
			}
		}
	}
	// 2 处理完请求后就清空掉
	InstanceData.ResetTransitionRequests();
	
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