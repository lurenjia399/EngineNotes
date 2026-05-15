1 UHTZoneGraphPathQuerySubsystem
```cpp
#if WITH_EDITOR
void UHTZoneGraphPathQuerySubsystem::RebuildForAllGraphs(const FZoneGraphBuildData& BuildData)
{
	// 离线操作
	// build结束时缓存所有路口数据到对象
	UWorld* World = GetWorld();
	if (!World)
	{
		return;
	}

	if (ZoneGraphPathQuerySaveActor == nullptr)
	{
		// 这里先尝试遍历查找
		for (TActorIterator<AZoneGraphPathQuerySaveActor> ActorIter(World); ActorIter; ++ActorIter)
		{
			if (*ActorIter)
			{
				RegisterSaveDataActor(*ActorIter);
				break;
			}
		}
	}
	
	if (ZoneGraphPathQuerySaveActor == nullptr)
	{
		FActorSpawnParameters SpawnInfo;
		World->SpawnActor<AZoneGraphPathQuerySaveActor>(AZoneGraphPathQuerySaveActor::StaticClass(), SpawnInfo);
	}

	if (!ZoneGraphPathQuerySaveActor)
	{
		return;
	}

	ZoneGraphPathQuerySaveActor->OnRebuildForAllGraphsSucceed(BuildData);
}
#endif
```