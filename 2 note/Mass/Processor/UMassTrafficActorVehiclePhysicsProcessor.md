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
}
```