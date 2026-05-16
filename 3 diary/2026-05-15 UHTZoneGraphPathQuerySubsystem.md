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
RebuildForAllGraphs 这个方法就是监听UE::ZoneGraphDelegates::OnZoneGraphDataBuildDone这个事件，在ZoneGraphData构建完成后，就会执行。内容就是创建出AZoneGraphPathQuerySaveActor这个Actor
# OnRebuildForAllGraphsSucceed
```cpp
void AZoneGraphPathQuerySaveActor::OnRebuildForAllGraphsSucceed(const FZoneGraphBuildData& BuildData)
{
#if WITH_EDITOR
	// Build数据完成 这里重新缓存路口连接数据
	++DataGenerated;
	
	LastGenerateTimeText = FDateTime::Now().ToFormattedString(TEXT("%Y-%m-%d %H:%M:%S"));

	// 错误计数
	ZoneGraphBuildErrorNum = 0;

	FMessageLogModule& MessageLogModule = FModuleManager::LoadModuleChecked<FMessageLogModule>("MessageLog");

	FMessageLog("ZoneGraphBuild").Info(FText::FromString(TEXT("OnRebuildForAllGraphsSucceed Data Build Begin")));
	
	UWorld* World = GetWorld();
	if (!World)
	{
		return;
	}
	
	const UMassTrafficSettings* TrafficSettings = GetDefault<UMassTrafficSettings>();
	if (!TrafficSettings)
	{
		return;
	}

	UZoneGraphSubsystem* ZoneGraphSubsystem = UWorld::GetSubsystem<UZoneGraphSubsystem>(World);
	if (!ZoneGraphSubsystem)
	{
		return;
	}
	
	FScopedSlowTask SlowTask(100.f, FText::FromString(TEXT("ZoneGraphPathQuerySaveActor Cache Dataing...")));
	SlowTask.MakeDialog();
	
	// 1、构造火车路线
	{
		SlowTask.EnterProgressFrame(20.f, FText::FromString(TEXT("ZoneGraphPathQuerySaveActor Cache 1. Build Train Lane.")));
		// 火车路线
		OnRebuildForAllGraphsSucceedTrainRoute(BuildData);
	}

	// 2、构造公交车路线
	{
		SlowTask.EnterProgressFrame(20.f, FText::FromString(TEXT("ZoneGraphPathQuerySaveActor Cache 2. Build Bus Lane.")));
		// bus
		OnRebuildForAllGraphSucceedBusRoute(BuildData);
	}

	// 3、构造车道连通步行路网的路线
	{
		SlowTask.EnterProgressFrame(20.f, FText::FromString(TEXT("ZoneGraphPathQuerySaveActor Cache 3. Build Vehicle link to Pedestrian Intersection.")));
		OnRebuildExtraLaneData(BuildData);
	}
	
	MarkPackageDirty();

	if (ZoneGraphBuildErrorNum == 0)
	{
		FString FailMsg = FString::Printf(TEXT("HTZoneGraphPathQuerySubsystem Build Success!"));
		UHTUtil::EditorShowSimpleDialogMsg(FailMsg);
	}
	else
	{
		FString FailMsg = FString::Printf(TEXT("HTZoneGraphPathQuerySubsystem Build Fail, %d errors Occured, Check Log~"), ZoneGraphBuildErrorNum);
		UHTUtil::EditorShowSimpleDialogMsg(FailMsg);
		
		MessageLogModule.OpenMessageLog("ZoneGraphBuild");
	}

	// 结束记录
	FMessageLog("ZoneGraphBuild").Info(FText::FromString(TEXT("OnRebuildForAllGraphsSucceed Data Build End")));
#endif
}
```
# 3 OnRebuildForAllGraphsSucceedTrainRoute
```cpp

```
构建火车路线的方法，
1 遍历ZoneShpeComp上的额外车道信息ExtraLaneInfos，如果车道满足火车车道的tag，就记录了到UnOrderTrainRoutes数组中
2 处理火车变轨车站，AHTTrainStationChangeLaneActor就是变轨actor，在变轨Actor上配置可以变轨的车站Actor（AHTTrainStationMarkerActor），把可变轨车站Actor记录在NeedChangeLaneStations数组中
3 遍历所有车站，构建路线名称-车站Actor的TMap，如果路线是变轨车站