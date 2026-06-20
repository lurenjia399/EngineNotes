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
	// 将TemplateID添加到Pending的Map中，
	FMassAgentInitializationQueue& AgentQueue = 
		PendingAgentEntities.FindOrAdd(EntityTemplate.GetTemplateID());
	AgentQueue.AgentComponents.Add(&AgentComp);
}
```