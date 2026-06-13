
# UMassTrafficUpdateVelocityProcessor
```cpp
UMassTrafficUpdateVelocityProcessor::UMassTrafficUpdateVelocityProcessor()
	: EntityQuery_Conditional(*this)
{
	// 自动注册tick
	bAutoRegisterWithProcessingPhases = true;
	ExecutionOrder.ExecuteInGroup = 
		UE::MassTraffic::ProcessorGroupNames::VehicleBehavior;
	ExecutionOrder.ExecuteAfter.Add(
		UE::MassTraffic::ProcessorGroupNames::FrameStart);
	ExecutionOrder.ExecuteAfter.Add(
		UE::MassTraffic::ProcessorGroupNames::PreVehicleBehavior);
	ExecutionOrder.ExecuteAfter.Add(
		UE::MassTraffic::ProcessorGroupNames::VehicleSimulationLOD);
	ExecutionOrder.ExecuteAfter.Add(
		UMassTrafficInterpolationProcessor::StaticClass()->GetFName());
}
```
# UMassStateTreeActivationProcessor 
```cpp
void UMassStateTreeActivationProcessor::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
	// 包含FMassStateTreeInstanceFragment，并且要读写
	EntityQuery.AddRequirement<FMassStateTreeInstanceFragment>(EMassFragmentAccess::ReadWrite);
	// 包含FMassStateTreeSharedFragment，只读
	EntityQuery.AddConstSharedRequirement<
		FMassStateTreeSharedFragment>();
	// 不包含FMassStateTreeActivatedTag
	EntityQuery.AddTagRequirement<
		FMassStateTreeActivatedTag>(EMassFragmentPresence::None);
	// 可选FMassSimulationVariableTickChunkFragment，只读。如果这个方法只有Optional的，那么就会查询到包含这个Fragment的Entity
	EntityQuery.AddChunkRequirement<
		FMassSimulationVariableTickChunkFragment>
			(EMassFragmentAccess::ReadOnly, 
				MassFragmentPresence::Optional);
	// 读写UMassStateTreeSubsystem
	EntityQuery.AddSubsystemRequirement<
		UMassStateTreeSubsystem>(EMassFragmentAccess::ReadWrite);
	// 
	ProcessorRequirements.AddSubsystemRequirement<
		UMassSignalSubsystem>(EMassFragmentAccess::ReadWrite);
}
```