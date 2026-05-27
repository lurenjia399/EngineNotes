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

DoSpawning之后生成数据构建完成
```cpp
void AMassSpawner::OnSpawnDataGenerationFinished(TConstArrayView<FMassEntitySpawnDataGeneratorResult> Results, FMassSpawnDataGenerator* FinishedGenerator)
{
	AllGeneratedResults.Append(Results.GetData(), Results.Num());
	SpawnGeneratedEntities(AllGeneratedResults);
	AllGeneratedResults.Reset();
}

void AMassSpawner::SpawnGeneratedEntities(
	TConstArrayView<FMassEntitySpawnDataGeneratorResult> Results)
{
	// 根据Result，执行SpawnEntities方法，创建Entities
	for (const FMassEntitySpawnDataGeneratorResult& Result : Results)
	{
		FSpawnedEntities& SpawnedEntities = 
			AllSpawnedEntities.AddDefaulted_GetRef();
		SpawnedEntities.TemplateID = EntityTemplate.GetTemplateID();
		SpawnerSystem->SpawnEntities(
			EntityTemplate.GetTemplateID(), 
			Result.NumEntities, 
			Result.SpawnData, 
			Result.SpawnDataProcessor, 
			SpawnedEntities.Entities);
		TotalNum += SpawnedEntities.Entities.Num();
	}
	
	// 创建PostSpawnProcessor
	for (const FMassEntitySpawnDataGeneratorResult& Result : Results)
	{
		for (const TSubclassOf<UMassProcessor>& ProcessorClass : Result.PostSpawnProcessors)
		{
			if (AddedProcessorClasses.Contains(ProcessorClass) == false)
			{
				if (UMassProcessor* Processor = 
					GetPostSpawnProcessor(ProcessorClass))
				{
					Processors.Add(Processor);
				}
				AddedProcessorClasses.Add(ProcessorClass);
			}
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
		// 跳过的车站，玩家在火车上，读取火车是哪个车站路线的，跳过生成
		if (!SkipRouteName.IsNone() && 
			SpawnConfigPairVal.Key.IsEqual(SkipRouteName))
		{
			continue;
		}
		
		// 获取这个路线火车的生成点，就是在车站之间的中点生成
		TArray<FZoneGraphLaneLocation> SpawnPoints = 
		ZoneGraphPathQuerySubsystem->GetTrainSpawnLocation(SpawnConfigPairVal.Key);

		int32 EntityTemplateIndex = INDEX_NONE;
		for (int32 i = 0; i < EntityTypes.Num(); i++)
		{
			if (SpawnConfigPairVal.Value.EntityConfig.ToSoftObjectPath() == EntityTypes[i].EntityConfig.ToSoftObjectPath())
			{
				EntityTemplateIndex = i;
				break;
			}
		}
		
		if (SpawnPoints.Num() > 0 && EntityTemplateIndex >= 0)
		{
			FMassEntitySpawnDataGeneratorResult& Res = Results.AddDefaulted_GetRef();
			Res.NumEntities = SpawnPoints.Num();

			Res.EntityConfigIndex = EntityTemplateIndex;
			
			Res.SpawnDataProcessor = 
				UMassTrafficInitTrafficVehiclesProcessor::StaticClass();
			Res.PostSpawnProcessors.Add(
				UMassTrafficVisualLoggingFieldOperationProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(
				UMassTrafficUpdateDistanceToNearestObstacleProcessor
				::StaticClass());
			Res.PostSpawnProcessors.Add(
				UMassTrafficChooseNextLaneProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(
				UMassTrafficInitTrafficVehicleSpeedProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(
				UMassTrafficInitInterpolationProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(
				UMassTrafficUpdateVelocityProcessor::StaticClass());
			Res.PostSpawnProcessors.Add(
				UHTMassTrafficMarkVehicleInitProcessor::StaticClass());

			Res.SpawnData.InitializeAs<FMassTrafficVehiclesSpawnData>();
			Res.SpawnData.GetMutable<
				FMassTrafficVehiclesSpawnData>().LaneLocations = SpawnPoints;
		}
	}

	FinishedGeneratingSpawnPointsDelegate.Execute(Results);
}
```

