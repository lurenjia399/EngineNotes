# StartTree

```cpp
void UBehaviorTreeComponent::StartTree(UBehaviorTree& Asset, EBTExecutionMode::Type ExecuteMode)
{
	/*
	1 StopTree
	*/
	StopTree(EBTStopMode::Safe);

	TreeStartInfo.Asset = &Asset;
	TreeStartInfo.ExecuteMode = ExecuteMode;
	TreeStartInfo.bPendingInitialize = true;
	/*
	1 核心初始化的操作
	*/
	ProcessPendingInitialize();
}
```

```cpp
bool UBehaviorTreeComponent::PushInstance(UBehaviorTree& TreeAsset)
{
	/*
	1 向 Manager 请求加载这棵树，拿到树的根节点 RootNode 和这棵树需要的总内存大小 InstanceMemorySize。
	2 每次Load相同的tree，会从缓存中找信息，当某个 AI 第一次跑某棵行为树，LoadTree 会把树"编译"一次（StaticDuplicateObject 复制出一套节点对象、算好每个节点的内存偏移），存进 LoadedTemplates。之后所有跑同一棵树的 AI，LoadTree 直接返回缓存里那同一个 Root 节点指针——不再复制。
	*/
	const bool bLoaded = BTManager->LoadTree(TreeAsset, RootNode, InstanceMemorySize);
	
	if (bLoaded)
	{
		/*
		1 添加一个BehaviorTreeInstance，填入InstanceStack数组中，表示一个行为树实例
		2 添加一个BehaviorTreeInstanceId，填入KnownInstances数组中，表示一个行为树子树实例。UpdateInstanceId 用"从根到当前节点的路径"生成一个唯一标识，用来复用同一子树在同一位置的历史内存。
		3 这个实例是在BehaviorTreeComp上的，每个AI一个
		*/
		FBehaviorTreeInstance& NewInstance = InstanceStack.AddDefaulted_GetRef();
		NewInstance.InstanceIdIndex = UpdateInstanceId(&TreeAsset, ActiveNode, InstanceStack.Num() - 1);
		NewInstance.RootNode = RootNode;
		NewInstance.ActiveNode = NULL;
		NewInstance.ActiveNodeType = EBTActiveNode::Composite;
		/*
		1 每个AI单独存储的行为树实例，赋值实例独有的内存空间InstanceMemory
		*/
		FBehaviorTreeInstanceId& InstanceInfo = 
			KnownInstances[NewInstance.InstanceIdIndex];
		int32 NodeInstanceIndex = InstanceInfo.FirstNodeInstance;
		const bool bFirstTime = (InstanceInfo.InstanceMemory.Num() != InstanceMemorySize);
		if (bFirstTime)
		{
			InstanceInfo.InstanceMemory.AddZeroed(InstanceMemorySize);
			InstanceInfo.RootNode = RootNode;
		}
		/*
		1 如果Node设置了bCreateNodeInstance，就需要创建NodeInstance，填入NodeInstances数组中
		*/
		NewInstance.SetInstanceMemory(InstanceInfo.InstanceMemory);
		NewInstance.Initialize(*this, *RootNode, NodeInstanceIndex, bFirstTime ? EBTMemoryInit::Initialize : EBTMemoryInit::RestoreSubtree);
	}
}
```

# tick
```cpp
void UBehaviorTreeComponent::TickComponent(float DeltaTime, const ELevelTick TickType, FActorComponentTickFunction *ThisTickFunction)
{
	/*
	1 减少下一次Tick执行时间
	*/
	NextTickDeltaTime -= DeltaTime;
	if (NextTickDeltaTime > 0.0f)
	{
		AccumulatedTickDeltaTime += DeltaTime;
		ScheduleNextTick(NextTickDeltaTime);
		return;
	}
	/*
	1 累计tick的间隔，就是距离上次tick经过的时间
	2 每次执行完tick，就清零这个时间
	*/
	AccumulatedTickDeltaTime += DeltaTime;
	ON_SCOPE_EXIT
	{
		AccumulatedTickDeltaTime = 0.0f;
	};
	DeltaTime = AccumulatedTickDeltaTime;
	/*
	1 记录这次tick是否是第一次tick
	*/
	const bool bWasTickedOnce = bTickedOnce;
	bTickedOnce = true;
	/*
	1 Tick 当前激活的辅助节点(Service / Decorator) 只遍历 当前处于激活状态 的 aux 节点(GetActiveAuxNodes),不是全树。Service 的定时逻辑、条件监听就在这里跑
	*/
	{
		FBTSuspendBranchActionsScoped ScopedSuspend(*this, EBTBranchAction::Changing_Topology_Actions);
		for (int32 InstanceIndex = 0; InstanceIndex < InstanceStack.Num(); InstanceIndex++)
		{
			FBehaviorTreeInstance& InstanceInfo = InstanceStack[InstanceIndex];
			InstanceInfo.ExecuteOnEachAuxNode([&InstanceInfo, this, &bDoneSomething, DeltaTime, &NextNeededDeltaTime](const UBTAuxiliaryNode& AuxNode)
				{
					uint8* NodeMemory = AuxNode.GetNodeMemory<uint8>(InstanceInfo);
					SCOPE_CYCLE_UOBJECT(AuxNode, &AuxNode);
					bDoneSomething |= AuxNode.WrappedTickNode(*this, NodeMemory, DeltaTime, NextNeededDeltaTime);
				});
		}
	}
	/*
	*/
	const bool bJustFinishedLatentAborts = TrackPendingLatentAborts();
	if (bJustFinishedLatentAborts)
	{
		if (bRequestedStop)
		{
			StopTree(EBTStopMode::Safe);
		}
		else
		{
			if (ExecutionRequest.ExecuteNode)
			{
				PendingExecution.Lock();

				if (ExecutionRequest.SearchEnd.IsSet())
				{
					ExecutionRequest.SearchEnd = FBTNodeIndex();
				}
			}

			ScheduleExecutionUpdate();
		}
	}
	/*
	*/
	bool bActiveAuxiliaryNodeDTDirty = false;
	if (bRequestedFlowUpdate)
	{
		ProcessExecutionRequest();
		bDoneSomething = true;

        // Since hierarchy might changed in the ProcessExecutionRequest, we need to go through all the active auxiliary nodes again to fetch new next DeltaTime
		bActiveAuxiliaryNodeDTDirty = true;
		NextNeededDeltaTime = UE::BehaviorTree::DisableTick;
	}
}
```