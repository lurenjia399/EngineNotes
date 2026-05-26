```cpp
/*
注册 RegisterEntityTemplates
*/
void AMassSpawner::PostRegisterAllComponents()
{
	Super::PostRegisterAllComponents();

	if (HasAnyFlags(RF_ClassDefaultObject) == false)
	{
		UWorld* World = GetWorld();
		if (GEngine->GetNetMode(GetWorld()) == NM_Client)
		{
			UMassSpawnerSubsystem* MassSpawnerSubsystem = UWorld::GetSubsystem<UMassSpawnerSubsystem>(World);
			if (MassSpawnerSubsystem)
			{
				RegisterEntityTemplates();
			}
			else
			{
				FWorldDelegates::OnPostWorldInitialization.AddUObject(this, &AMassSpawner::OnPostWorldInit);
			}
		}
	}
}
```

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

# UMassEntitySpawnDataGeneratorBase
生成器
```cpp
void UHTMassTrainSpawnDataGenerator::Generate(
	UObject& QueryOwner, 
	TConstArrayView<FMassSpawnedEntityType> EntityTypes, 
	int32 Count,
                                              FFinishedGeneratingSpawnDataSignature& FinishedGeneratingSpawnPointsDelegate) const
{
	UWorld* World = GetWorld();
	if (!World)
	{
		return;
	}
	
	AHTMassTrainSpawner* OwnerTrainSpawner = Cast<AHTMassTrainSpawner>(&QueryOwner);
	if (!OwnerTrainSpawner)
	{
		return;
	}
	
	UHTZoneGraphPathQuerySubsystem* ZoneGraphPathQuerySubsystem = 
		World->GetSubsystem<UHTZoneGraphPathQuerySubsystem>();
	if (!ZoneGraphPathQuerySubsystem)
	{
		return;
	}

	TArray<FMassEntitySpawnDataGeneratorResult> Results;
	
	FName SkipRouteName = OwnerTrainSpawner->GetSkipSpawnRouteName();
	
	for (const auto& SpawnConfigPairVal:
		OwnerTrainSpawner->TrainEnityConfigOverride)
	{
		if (!SkipRouteName.IsNone() && 
			SpawnConfigPairVal.Key.IsEqual(SkipRouteName))
		{
			
			continue;
		}
		
		TArray<FZoneGraphLaneLocation> SpawnPoints = ZoneGraphPathQuerySubsystem->GetTrainSpawnLocation(SpawnConfigPairVal.Key);

		int32 EntityTemplateIndex = INDEX_NONE;
		for (int32 i = 0; i < EntityTypes.Num(); i++)
		{
			if (SpawnConfigPairVal.Value.EntityConfig.ToSoftObjectPath() == EntityTypes[i].EntityConfig.ToSoftObjectPath())
			{
				EntityTemplateIndex = i;
				break;
			}
		}
		
#if !UE_BUILD_SHIPPING || HOTTA_INNER_ENV
		for (const FZoneGraphLaneLocation &LaneLocation: SpawnPoints)
		{
			UE_LOG(LogHTTrain, Log, TEXT("Train Log Will Spawn [%s] Location:%s, Distance:%f"), *SpawnConfigPairVal.Key.ToString(), *LaneLocation.LaneHandle.ToString(), LaneLocation.DistanceAlongLane);
		}
#endif
		
		if (SpawnPoints.Num() > 0 && EntityTemplateIndex >= 0)
		{
			FMassEntitySpawnDataGeneratorResult& Res = Results.AddDefaulted_GetRef();
			Res.NumEntities = SpawnPoints.Num();

			Res.EntityConfigIndex = EntityTemplateIndex;
			
			Res.SpawnDataProcessor = UMassTrafficInitTrafficVehiclesProcessor::StaticClass();
			//Res.PostSpawnProcessors.Add(UMassTrafficFindNextVehicleProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(UMassTrafficVisualLoggingFieldOperationProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(UMassTrafficUpdateDistanceToNearestObstacleProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(UMassTrafficChooseNextLaneProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(UMassTrafficInitTrafficVehicleSpeedProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(UMassTrafficInitInterpolationProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(UMassTrafficUpdateVelocityProcessor::StaticClass());
			
			Res.PostSpawnProcessors.Add(UHTMassTrafficMarkVehicleInitProcessor::StaticClass());

			Res.SpawnData.InitializeAs<FMassTrafficVehiclesSpawnData>();
			Res.SpawnData.GetMutable<FMassTrafficVehiclesSpawnData>().LaneLocations = SpawnPoints;
		}
	}

	FinishedGeneratingSpawnPointsDelegate.Execute(Results);
}
```

