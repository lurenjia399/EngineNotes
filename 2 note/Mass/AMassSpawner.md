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
	// 调用 FMassSpawnDataGenerator 来生成
	for (FMassSpawnDataGenerator& Generator : SpawnDataGenerators)
	{
		if (Generator.GeneratorInstance)
		{
			const float ProportionRatio = FMath::Min(
				Generator.Proportion / ProportionRemaining, 1.0f);
			const int32 SpawnCount = FMath::CeilToInt(
				static_cast<float>(SpawnCountRemaining) * ProportionRatio);
			
			FFinishedGeneratingSpawnDataSignature Delegate = 
				FFinishedGeneratingSpawnDataSignature::CreateUObject(
				this, &AMassSpawner::OnSpawnDataGenerationFinished, &Generator);
			Generator.GeneratorInstance->
				Generate(*this, EntityTypes, SpawnCount, Delegate);
			SpawnCountRemaining -= SpawnCount;
			ProportionRemaining -= Generator.Proportion;
		}
	}
}
```

