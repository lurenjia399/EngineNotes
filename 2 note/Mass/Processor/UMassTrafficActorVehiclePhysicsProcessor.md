1 
2 
```cpp
void UMassTrafficActorVehiclePhysicsProcessor::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	// 只有高模Actor才会执行，其余的不执行
	if (RepresentationFragments[EntityIt].CurrentRepresentation 
		!= EMassRepresentationType::HighResSpawnedActor)
	{
		continue;
	}
	
	// 如果Actor已经报废
	AActor* Actor = ActorFragment.GetMutable();
	if (Actor != nullptr && Actor->Implements<UMassTrafficVehicleControlInterface>())
	{
		if (VehicleDamageFragments[EntityIt].VehicleDamageState 
		>	= EMassTrafficVehicleDamageState::Totaled)
		{
			Context.Defer().PushCommand<FMassDeferredSetCommand>(
				[Actor](FMassEntityManager&)
				{
					IMassTrafficVehicleControlInterface::
						Execute_SetVehicleInputs(
						Actor, 0.0f, 1.0f, false, 0.0f, false);
				});
		}
}
```