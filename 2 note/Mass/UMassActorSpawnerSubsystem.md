1 在RepresentationProcessor中，通过UMassRepresentationActorManagement来创建FMassActorSpawnRequest一个请求，将请求添加到ActorSpawnerSubsystem中的SpawnRequests请求数组中，并返回请求Handle
2 处理请求是在EMassProcessingPhase::PrePhysics这个阶段tick中
3 具体的处理请求方法是ProcessSpawnRequest
```cpp
ESpawnRequestStatus UMassActorSpawnerSubsystem::ProcessSpawnRequest(
	const FMassActorSpawnRequestHandle SpawnRequestHandle, 
	FStructView SpawnRequestView, 
	FMassActorSpawnRequest& SpawnRequest)
{
	// 改变状态，正在Spawn中
	// Do the spawning
	SpawnRequest.SpawnStatus = ESpawnRequestStatus::Processing;
	// 广播Spawn前的Delegate
	// Call the pre spawn delegate on the spawn request
	if (SpawnRequest.ActorPreSpawnDelegate.IsBound())
	{
		SpawnRequest.ActorPreSpawnDelegate.Execute(SpawnRequestHandle, SpawnRequestView);
	}
	// 真正的Spawn处理，会首先在ActorPool中找，找不到就SpawnActor
	SpawnRequest.SpawnStatus = SpawnOrRetrieveFromPool(SpawnRequestView, SpawnRequest.SpawnedActor);

	if (SpawnRequest.IsFinished())
	{
		// 如果是Spawn成功了，就设置Puppet，用AgentComp上的Entity接管当前的行人Entity
		if (SpawnRequest.SpawnStatus == ESpawnRequestStatus::Succeeded && IsValid(SpawnRequest.SpawnedActor))
		{
			if (UMassAgentComponent* AgentComp = SpawnRequest.SpawnedActor->FindComponentByClass<UMassAgentComponent>())
			{
				AgentComp->SetPuppetHandle(SpawnRequest.MassAgent);
			}
		}

		EMassActorSpawnRequestAction PostAction = EMassActorSpawnRequestAction::Remove;

		// Call the post spawn delegate on the spawn request
		if (SpawnRequest.ActorPostSpawnDelegate.IsBound())
		{
			PostAction = SpawnRequest.ActorPostSpawnDelegate.Execute(SpawnRequestHandle, SpawnRequestView);
		}

		if (PostAction == EMassActorSpawnRequestAction::Remove)
		{
			// If notified, remove the spawning request
			ensureMsgf(SpawnRequestHandleManager.RemoveHandle(SpawnRequestHandle), TEXT("When providing a delegate, the spawn request gets automatically removed, no need to remove it on your side"));
		}
	}
	else
	{
		// lower priority
		SpawnRequest.SpawnStatus = ESpawnRequestStatus::RetryPending;
	}

	return SpawnRequest.SpawnStatus;
}
```
