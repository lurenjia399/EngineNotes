```cpp
void AMassSpawner::BeginPlay()
{
	Super::BeginPlay();
	const ENetMode NetMode = GEngine->GetNetMode(GetWorld());
	if (bAutoSpawnOnBeginPlay && NetMode != NM_Client)

	{
		const UMassSimulationSubsystem* MassSimulationSubsystem = 
			UWorld::GetSubsystem<UMassSimulationSubsystem>(GetWorld());
		if (MassSimulationSubsystem == nullptr 
			|| MassSimulationSubsystem->IsSimulationStarted())
		{
			DoSpawning();
		}
		else
		{
			SimulationStartedHandle = 
				UMassSimulationSubsystem::GetOnSimulationStarted().AddLambda(
				[this](UWorld* InWorld)
				{
					UWorld* World = GetWorld();
	
					if (World == InWorld)
					{
						DoSpawning();
					}
				});
		}
}
```

```cpp
void AMassSpawner::DoSpawning()
{
	
}
```

