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
3 添加到PendingAgentEntities 这个数组后，处理zhe'ge逻辑
```cpp
void UMassAgentSubsystem::OnProcessingPhaseStarted(const float DeltaSeconds, const EMassProcessingPhase Phase)
{
	switch (Phase)
	{
	case EMassProcessingPhase::PrePhysics:
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