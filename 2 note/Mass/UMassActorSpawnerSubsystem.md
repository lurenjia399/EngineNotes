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
		// 如果是Spawn成功了就设置Puppet，将AgentComp配置的Fragment动态填充到具体entity中
		if (SpawnRequest.SpawnStatus == ESpawnRequestStatus::Succeeded && IsValid(SpawnRequest.SpawnedActor))
		{
			if (UMassAgentComponent* AgentComp = SpawnRequest.SpawnedActor->FindComponentByClass<UMassAgentComponent>())
			{
				AgentComp->SetPuppetHandle(SpawnRequest.MassAgent);
			}
		}

	return SpawnRequest.SpawnStatus;
}
```
4 AgentComp上面会配置UHTMassAgentTransformSyncTrait这个，用来将mass位置同步到Actor位置

# PoolActor实现
1 
```cpp
TMap<TSubclassOf<AActor>, TArray<TObjectPtr<AActor>>> PooledActors;
// key是Actor的UClass
// 值就是Actor的数组
```
2 
