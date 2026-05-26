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
3 遍历所有车站，构建路线名称-车站Actor的TMap（StationsByRoute），如果路线是变轨车站还需要额外添加到StationChangeLanesMap数组中
4 找车道起始位置，如果是变轨车道，通过FindNearestLocationOnLane接口来查询变轨车道上距离AHTTrainStationMarkerActor这个车站Actor最近的点（起始位置）。如果不是变轨车道，就直接通过FindNearestLane接口来查询最近车道上距离车站Actor最近的点（起始位置）。
5 构建火车路线，通过GetLinkedLanes接口获取当前车道相连的车道，把路线上所有车道添加到TrainRoute数组中

```cpp
TArray<FZoneGraphLaneLocation> UHTZoneGraphPathQuerySubsystem::GetTrainSpawnLocation(
	const FName& RouteName) const
{
	

	const TMap<int32, FTrainRouteCache>& TrainRouteCacheMap = ZoneGraphPathQuerySaveActor->TrainRouteCacheMap;

	const FTrainRouteCache* RouteCache = nullptr;
	int32 RouteIndex = INDEX_NONE;
    for (const auto& PairVal : TrainRouteCacheMap)
    {
    	if (PairVal.Value.RouteName == RouteName)
    	{
    		RouteCache = &PairVal.Value;
    		RouteIndex = PairVal.Key;
    		break;
    	}
    }

	if (!RouteCache)
	{
		return Res;
	}

	// 从火车路线中取出车站位置
	TArray<FZoneGraphLaneLocation> TrainStationLocations;
	for (int32 i = 0; i < RouteCache->TrainLanesInOrderArray.Num(); ++i)
	{
		if (RouteCache->TrainLanesInOrderArray[i].PossibleStationLoction.IsValid())
		{
			TrainStationLocations.Add(
				RouteCache->TrainLanesInOrderArray[i].
				PossibleStationLoction.GetLaneLocation());
		}
	}

	auto CheckLaneSuitableFunc = [](const FZoneGraphLaneHandle& LaneHandle, OUT bool& bFailedAndContinueNextLane)-> bool
	{
		// 没有道路限制，都允许
		return true;
	};

	// 遍历车站位置
	for (int32 i = 0; i < TrainStationLocations.Num(); ++i)
	{
	    if(!RouteCache->bLoopRoute && i == TrainStationLocations.Num() - 1)
	    {
	        // 如果非闭环，则不处理最后一个车站
	        break;
	    }
		
		// 当前车站位置
		FZoneGraphLaneLocation TrainStationLoc = TrainStationLocations[i];
		// 下一个车站位置，如果是环就从0开始
		FZoneGraphLaneLocation NextTrainStationLoc = 
			TrainStationLocations.IsValidIndex(i+1) ? 
			TrainStationLocations[i+1] : TrainStationLocations[0];
		
		float Dist = 0.f;
		// 从车站位置开始，到下一个车站位置的长度
		if (GetTrainRouteLocationDistacne(
			TrainStationLoc, RouteIndex, NextTrainStationLoc, Dist))
		{
			if (Dist <= 0.f)
			{
				continue;
			}

			FZoneGraphLaneLocation SpawnLoc;
			if (UHTUtil::AdvanceLaneLocationFullDist(
				ZoneGraphSubsystem, 
				TrainStationLoc, 
				Dist/2.f, 
				CheckLaneSuitableFunc, 
				SpawnLoc))
			{
				Res.Add(SpawnLoc);
			}
		}
	}
	
	return Res;
}
```