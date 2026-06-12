# OnWorldBeginPlay
```cpp
void UMassSimulationSubsystem::OnWorldBeginPlay(UWorld& InWorld)
{
	Super::OnWorldBeginPlay(InWorld);

	// To evaluate the effective processors execution mode, we need to wait on OnWorldBeginPlay before calling
	// RebuildTickPipeline as we are sure by this time the network is setup correctly.
	RebuildTickPipeline();
	// note that since we're in this function we're tied to a game world. This means the StartSimulation in 
	// PostInitialize haven't been called.
	StartSimulation(InWorld);
}
```