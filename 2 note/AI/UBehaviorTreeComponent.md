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
	1 
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