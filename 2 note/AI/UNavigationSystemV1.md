# Build
```cpp
void UNavigationSystemV1::Build()
{
	/*
	1 清掉当前World上所有Level的NavDataChunk。
	2 ARecastNavMesh::OnNavMeshGenerationFinished 在编辑器下的这个方法中会创建出NavDataChunk。
	3 在ARecastNavMesh::OnStreamingLevelAdded和ARecastNavMesh::OnStreamingLevelRemoved这个方法中会往NavDataChunk中填充数据
	*/
	FNavigationSystem::DiscardNavigationDataChunks(*World);
	/*
	1 判断是否有导航区域，就是NavVolume框住的范围
	2 没有导航区域就返回，不Build
	*/
	const bool bHasWork = IsThereAnywhereToBuildNavigation();
	if (!bHasWork)
	{
		return;
	}
	
	/*
	1 开启
	*/
	const double BuildStartTime = FPlatformTime::Seconds();
}
```

```cpp
 ┌─────────┬───────────────────────────────────┬─────────────────┐
  │  方式   │               位置                │      触发       │
  ├─────────┼───────────────────────────────────┼─────────────────┤
  │ 控制台  │ NavigationSystem.cpp:5877         │ 控制台输入 Rebu │
  │ 命令    │                                   │ ildNavigation   │
  ├─────────┼───────────────────────────────────┼─────────────────┤
  │ 编辑器  │                                   │ Build → Build   │
  │ 菜单    │ EditorBuildUtils.cpp:979          │ Paths / Build   │
  │         │                                   │ All             │
  ├─────────┼───────────────────────────────────┼─────────────────┤
  │ WP      │ WorldPartitionNavigationDataBuild │ World Partition │
  │ 构建器  │ er.cpp:418                        │  分块导航构建   │
  ├─────────┼───────────────────────────────────┼─────────────────┤
  │         │                                   │ -run=ResavePack │
  │ 命令行  │ ContentCommandlets.cpp:2272       │ ages -BuildNavi │
  │         │                                   │ gationData      │
  └─────────┴───────────────────────────────────┴─────────────────┘

```
# AddNavigationBoundsUpdateRequest
``` cpp
void UNavigationSystemV1::PerformNavigationBoundsUpdate(const TArray<FNavigationBoundsUpdateRequest>& UpdateRequests)
{
	// 1 会根据参数请求的类型，将请求中的NavBounds从RegisteredNavBounds成员变量中添加或者移除
	TArray<FBox> UpdatedAreas;
	for (const FNavigationBoundsUpdateRequest& Request : UpdateRequests)
	{
		switch (Request.UpdateRequest)
		{
		
		}
	}
	// 2 更新了NavBounds，也需要告诉ANavigationData，NavBounds改变了。因为NavBounds改变所需的最大tile是否超标，如果超标了就需要重建DetourMesh
	if (UpdatedAreas.Num())
	{
		for (ANavigationData* NavData : NavDataSet)
		{
			if (NavData)
			{
				NavData->OnNavigationBoundsChanged();	
			}
		}
	}
}
```

# GetNavigationBoundsForNavData

```cpp
// 获取特定导航数据对象（如某个NavMesh）应该使用的所有导航边界（Navigation Bounds）
int UNavigationSystemV1::GetNavigationBoundsForNavData(
	const ANavigationData& NavData, // 导航数据
	TArray<FBox>& OutBounds, // 输出符合条件的导航范围
	ULevel* InLevel) const // 只获取特定关卡的导航范围
{
	// 
	const int InitialBoundsCount = OutBounds.Num();
	OutBounds.Reserve(InitialBoundsCount + RegisteredNavBounds.Num());
	
	// 2 通过导航数据获取AgentIndex,AgentIndex表示不同尺寸的角色
	const int32 AgentIndex = GetSupportedAgentIndex(&NavData);
	// 3 SystemV1没配SupportedAgents就会是默认的，这里AgentIndex默认回是0
	if (AgentIndex != INDEX_NONE)
	{
		// 4 判断NavBounds中配的SupportedAgents是否满足，满足的话就输出导航区域
		for (const FNavigationBounds& NavigationBounds : RegisteredNavBounds)
		{
			if ((InLevel == nullptr || NavigationBounds.Level == InLevel)
				&& NavigationBounds.SupportedAgents.Contains(AgentIndex))
			{
				OutBounds.Add(NavigationBounds.AreaBox);
			}
		}
	}

	return OutBounds.Num() - InitialBoundsCount;
}
```
