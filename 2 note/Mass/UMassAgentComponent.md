1 entity的代理
2 在AgentComponent的Onregister方法中会将自己注册进MassAgentSubsystem中
```cpp
FMassEntityTemplateID UMassAgentSubsystem::RegisterAgentComponent(
	UMassAgentComponent& AgentComp)
{
	// 根据配置的EntityConfig创建出EntityTemplate
	const FMassEntityConfig& EntityConfig = AgentComp.GetEntityConfig();
	const FMassEntityTemplate& EntityTemplate = 
		EntityConfig.GetOrCreateEntityTemplate(*World);
	// 将Comp添加到Pending的Map中，key是TemplateID,value是Comp数组
	FMassAgentInitializationQueue& AgentQueue = 
		PendingAgentEntities.FindOrAdd(EntityTemplate.GetTemplateID());
	AgentQueue.AgentComponents.Add(&AgentComp);
	// 切换Comp身上的状态为EntityPendingCreation
	AgentComp.EntityCreationPending();
}
```
3 添加到PendingAgentEntities 这个数组后，处理这个数组的操作是在HandlePendingInitialization方法中
```cpp
void UMassAgentSubsystem::OnProcessingPhaseStarted(const float DeltaSeconds, const EMassProcessingPhase Phase)
{
	switch (Phase)
	{
	case EMassProcessingPhase::PrePhysics:
		// 是在PrePhysics的ProcessingPhase的tick开始的时候执行
		if (PendingAgentEntities.Num() > 0 || PendingPuppets.Num() > 0)
		{
			HandlePendingInitialization();
		}
		break;
	default:
		break;
	}
}
```

```cpp
void UMassAgentSubsystem::HandlePendingInitialization()
{
	for (TTuple<FMassEntityTemplateID, FMassAgentInitializationQueue>& Data : 
									PendingAgentEntities)
	{
		// 注册好的AgentComp
		TArray<TObjectPtr<UMassAgentComponent>>& AgentComponents = 
			Data.Get<1>().AgentComponents;
		// 有多少AgentComponent就说明要生成多少entity
		const int32 NewEntityCount = AgentComponents.Num();
		if (NewEntityCount <= 0)
		{
			continue;
		}
		// SpawnEntity
		SpawnerSystem->SpawnEntities(EntityTemplate, NewEntityCount, Entities);
		// 将创建好的EntityHandle设置回AgentComp中
		for (int AgentIndex = 0; AgentIndex < Entities.Num(); ++AgentIndex)
		{		
			AgentComponents[AgentIndex]->SetEntityHandle(Entities[AgentIndex]);
		}
	}
	// 清掉Pending数组
	PendingAgentEntities.Reset();
}
```

4 也可以设置entity为傀儡形态
```cpp
void UMassAgentSubsystem::MakePuppet(UMassAgentComponent& AgentComp)
{
	// 将我们的TemplateID添加到Puppet
	FMassAgentInitializationQueue& PuppetQueue = PendingPuppets.FindOrAdd(AgentComp.GetTemplateID());
	PuppetQueue.AgentComponents.Add(&AgentComp);

	AgentComp.PuppetInitializationPending();
}
```