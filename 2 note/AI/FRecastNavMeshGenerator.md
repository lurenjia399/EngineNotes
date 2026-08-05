# Init
```cpp
void FRecastNavMeshGenerator::Init()
{
	/*
	1 读取ARecastNavMesh中的配置，输出到Config成员变量中
	*/
	ConfigureBuildProperties(Config);
	/*
	1 
	*/
	AdditionalCachedData = FRecastNavMeshCachedData::Construct(DestNavMesh);
	ResolveGeneratedLinkAreas(Config);
	/*
	1 
	*/
	UpdateNavigationBounds();
	/*
	
	*/
	if (bRecreateNavmesh)
	{
		ConstructTiledNavMesh();
		if (NavSys && NavSys->IsActiveTilesGenerationEnabled() == false)
		{
			MarkNavBoundsDirty();
		}
	}
	else
	{
		Config.MaxPolysPerTile = MaxPolysPerTile;
		NumActiveTiles = GetTilesCountHelper(DestNavMesh->GetRecastNavMeshImpl()->DetourNavMesh);
	}
}
```

# ConstructTiledNavMesh

# OnNavigationBoundsChanged
```cpp
void FRecastNavMeshGenerator::OnNavigationBoundsChanged()
{
	/*
	1 记录InclusionBounds，表示Navdata影响的Box
	2 记录TotalNavBounds，表示NavDatda影响多少个Box
	*/
	UpdateNavigationBounds();
	
	/*
	1 获取Detour数据类
	*/
	dtNavMesh* DetourMesh = DestNavMesh->GetRecastNavMeshImpl() 
		? DestNavMesh->GetRecastNavMeshImpl()->GetRecastMesh() : nullptr;
	/*
	1 不能是静态NavMesh。静态 NavMesh = 游戏运行时不可修改的烘焙数据，动态 NavMesh = 可以在编辑器/运行时重建
	2 NavMesh 标记为可调整大小。 某些固定大小的 NavMesh 不允许改变瓦片数量
	3 DetourMesh 存在
	*/
	if (!IsGameStaticNavMesh(DestNavMesh) && DestNavMesh->IsResizable() && DetourMesh)
	{
		/*
		1  // 锁定场景：- 正在进行批量操作（如关卡流送）- 用户临时禁用自动重建。在没有锁定的情况下才会继续执行
		*/
		const UNavigationSystemV1* NavSys = FNavigationSystem::
			GetCurrent<UNavigationSystemV1>(GetWorld());
		if (NavSys && !NavSys->IsNavigationBuildingLocked())
		{
			// 计算边界范围里能容纳的Tile数量，数量最后在乘上平均层数就是MaxRequestedTiles
			const int32 MaxRequestedTiles = UE::NavMesh::Private::
				CalculateMaxTilesCount(
				InclusionBounds, // 导航应用的边界，就是NavVolume所框选的范围
				Config.GetTileSizeUU(), //配置的Tile大小
				AvgLayersPerTile, //每个Tile的平均层数
				DestNavMesh->NavMeshVersion);// 使用NavMesh的版本号
			/*
			1 如果DetourMesh中最大的Tile数量和需要的Tile不一致，就需要重建Detour
			2 为什么需要销毁重建？
				  // Detour NavMesh 在创建时指定最大瓦片数
				  // 这个数量是固定的，不能动态调整
				  // 要改变容量，必须：
				  //   1. 销毁旧的 dtNavMesh
				  //   2. 创建新的 dtNavMesh（新容量）
				  //   3. 重新生成所有瓦片数据
			*/
			if (DetourMesh->getMaxTiles() != MaxRequestedTiles)
			{
				// 销毁当前的DetourMesh
				DestNavMesh->GetRecastNavMeshImpl()->SetRecastMesh(nullptr);

				/*
				 // 1. 将所有边界标记为"脏区域"
				  // 2. 使用 NavigationBounds 标志（表示是边界变化，不是几何体变化）
				  // 3. 调用 RebuildDirtyAreas 触发重建
				*/
				if (InclusionBounds.Num() > 0)
				{
					TArray<FNavigationDirtyArea> AsDirtyAreas;
					AsDirtyAreas.Reserve(InclusionBounds.Num());
					for (const FBox& BBox : InclusionBounds)
					{
						AsDirtyAreas.Add(FNavigationDirtyArea(
						BBox, ENavigationDirtyFlag::NavigationBounds));
					}
					RebuildDirtyAreas(AsDirtyAreas);
				}
			}
		}
	}
}
```