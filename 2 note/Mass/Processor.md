
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

```cpp
void UMassStateTreeActivationProcessor::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& ExecutionContext)
{
	// 
	UMassSignalSubsystem& SignalSubsystem = ExecutionContext.
		GetMutableSubsystemChecked<UMassSignalSubsystem>();

	const UMassBehaviorSettings* BehaviorSettings = 
		GetDefault<UMassBehaviorSettings>();
	const double TimeInSeconds = EntityManager.GetWorld()
		->GetTimeSeconds();

	TArray<FMassEntityHandle> EntitiesToSignal;
	int32 ActivationCounts[EMassLOD::Max] {0,0,0,0};
	
	EntityQuery.ForEachEntityChunk(ExecutionContext,
		[&EntitiesToSignal, 
			&ActivationCounts, 
			MaxActivationsPerLOD = BehaviorSettings
				->MaxActivationsPerLOD, 
			TimeInSeconds]
		(FMassExecutionContext& Context)
		{
			UMassStateTreeSubsystem& MassStateTreeSubsystem = Context
				.GetMutableSubsystemChecked<UMassStateTreeSubsystem>();
			// 遍历到的Entities刷零
			const int32 NumEntities = Context.GetNumEntities();
			
			// 限制每帧激活statetree的数量
			const EMassLOD::Type ChunkLOD = FMassSimulationVariableTickChunkFragment::GetChunkLOD(Context);
			if (ActivationCounts[ChunkLOD] > MaxActivationsPerLOD[ChunkLOD])
			{
				return;
			}
			ActivationCounts[ChunkLOD] += NumEntities;
			
			// 拿到FMassStateTreeInstanceFragment和FMassStateTreeSharedFragment数组
			const TArrayView<FMassStateTreeInstanceFragment> StateTreeInstanceList = Context.GetMutableFragmentView<FMassStateTreeInstanceFragment>();
			const FMassStateTreeSharedFragment& SharedStateTree = Context.GetConstSharedFragment<FMassStateTreeSharedFragment>();

			// 修改FMassStateTreeInstanceFragment这个Fragment，为Entity创建UMassStateTreeProcessor这个Proscessor，只要是不同的StateTree，就会创建一个Processor
			for (FMassExecutionContext::FEntityIterator EntityIt = Context.CreateEntityIterator(); EntityIt; ++EntityIt)
			{
				FMassStateTreeInstanceFragment& StateTreeInstance = StateTreeInstanceList[EntityIt];
				StateTreeInstance.InstanceHandle = MassStateTreeSubsystem.AllocateInstanceData(SharedStateTree.StateTree);
			}
			
			// 运行StateTree
			// Start StateTree. This may do substantial amount of work, as we select and enter the first state.
			UE::MassBehavior::ForEachEntityInChunk(Context, MassStateTreeSubsystem,
				[TimeInSeconds](FMassStateTreeExecutionContext& StateTreeExecutionContext, FMassStateTreeInstanceFragment& StateTreeFragment)
				{
					// Start the tree instance
					StateTreeExecutionContext.Start();
					StateTreeFragment.LastUpdateTimeInSeconds = TimeInSeconds;
				});

			// 通过CommondBuffer给遍历到的Entity添加FMassStateTreeActivatedTag，在所有的Processor都执行完成，在主线程执行FlushCommond
			// Adding a tag on each entities to remember we have sent the state tree initialization signal
			EntitiesToSignal.Reserve(EntitiesToSignal.Num() + NumEntities);
			for (FMassExecutionContext::FEntityIterator EntityIt = Context.CreateEntityIterator(); EntityIt; ++EntityIt)
			{
				const FMassStateTreeInstanceFragment& StateTreeInstance = StateTreeInstanceList[EntityIt];
				if (StateTreeInstance.InstanceHandle.IsValid())
				{
					const FMassEntityHandle Entity = Context.GetEntity(EntityIt);
					Context.Defer().AddTag<FMassStateTreeActivatedTag>(Entity);
					EntitiesToSignal.Add(Entity);
				}
			}
		});
	
	// 把StateTreeActivate的信号发给这些Entities。
	// Signal all entities inside the consolidated list
	if (EntitiesToSignal.Num())
	{
		SignalSubsystem.SignalEntities(
			UE::Mass::Signals::StateTreeActivate, EntitiesToSignal);
	}
}
```