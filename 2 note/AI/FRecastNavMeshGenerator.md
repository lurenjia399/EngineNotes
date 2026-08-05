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
	1 如果是需要创建Mavmsh，就会通过ConstructTiledNavMesh方法创建出一个dtNavMesh，并初始化，但是还没有塞入Tile数据
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
```cpp
bool FRecastNavMeshGenerator::ConstructTiledNavMesh() 
{
	// 1 取消构建，确保没有残留的异步 tile 生成任务在跑，避免新旧网格并存导致数据竞争
	CancelBuild();
	/*
	2 创建新的dtNavMesh实例
	*/
	dtNavMesh* DetourMesh = dtAllocNavMesh();	
	if (DetourMesh)
	{
		// 3 初始化TiledMeshParameters数据
		dtNavMeshParams TiledMeshParameters;
		FMemory::Memzero(TiledMeshParameters);	
		// 4  是ARecastNavMesh中NavMeshOriginOffset，表示tile空间的原点，默认是0，0，0
		FVector NMOrigin = RcNavMeshOrigin;
		rcVcopy(TiledMeshParameters.orig, &NMOrigin.X);
		// 5 设置tile的宽高，计算就是tileSize * cs。tileSize是TileSizeUU / CellSize表示Tile的边长能容纳多少个Cell。cs就是CellSize，每个格子的大小。因为宽高一样所以是正方形
		TiledMeshParameters.tileWidth = Config.GetTileSizeUU();
		TiledMeshParameters.tileHeight = Config.GetTileSizeUU();
		// 6  Detour 的 dtPolyRef 设计是固定位宽的整数ID，不像现代数据库可以用变长ID。所以必须在初始化时就决定位分配方案，之后无法更改。 这个方法就是做这个决策：1. 看世界有多大（需要多少 tile） 2. 看硬限制允许多少（TileNumberHardLimit）3. 算出一个平衡的分配方案 4. 如果算出来的超过编码能力，报错并截断。一旦 dtNavMesh->init() 调用完成，这个导航网格的 maxTiles 和  maxPolys 就固化了，除非整个推倒重建（ConstructTiledNavMesh重来），否则无法扩容
		CalcNavMeshProperties(TiledMeshParameters.maxTiles, 
			TiledMeshParameters.maxPolys);
		Config.MaxPolysPerTile = TiledMeshParameters.maxPolys;
		// 7 设置Agent的攀爬高度，高度，身体半径
		TiledMeshParameters.walkableClimb = Config.AgentMaxClimb;
		TiledMeshParameters.walkableHeight = Config.AgentHeight;
		TiledMeshParameters.walkableRadius = Config.AgentRadius;
		// 8 赋值bvQuantFactor，就是CellSize的倒数
		TiledMeshParameters.resolutionParams[
			(uint8)ENavigationDataResolution::Low].bvQuantFactor = 
				1.f / DestNavMesh->GetCellSize(ENavigationDataResolution::Low);
		TiledMeshParameters.resolutionParams[
			(uint8)ENavigationDataResolution::Default].bvQuantFactor = 
				1.f / DestNavMesh->GetCellSize(ENavigationDataResolution::Default);
		TiledMeshParameters.resolutionParams[
			(uint8)ENavigationDataResolution::High].bvQuantFactor = 
				1.f / DestNavMesh->GetCellSize(ENavigationDataResolution::High);
		/*
		1 根据参数初始化dtNavMesh方法
		2 通过memcpy来将参数拷贝到m_params成员变量中
		3 缓存参数中的tileWidth，tileHeight，maxTiles等数据
		4 创建m_tiles单向列表，把所有的tile都串联起来
		5 创建m_posLookup快速查询，目的是通过tile的x,y坐标，快速从数组中找到tile
		6 此时 dtNavMesh 已完全初始化，但不含任何 tile 数据——所有 tile都在空闲链表里，_posLookup 全是 nullptr。后续通过 addTile() 逐个添加实际的导航数据
		*/
		const dtStatus status = DetourMesh->init(&TiledMeshParameters);
		// 10 记录当前有效的Tile的数量
		NumActiveTiles = GetTilesCountHelper(DetourMesh);
		// 11 把初始化号的dtnavMesh填充到RecastNamesh结构中
		DestNavMesh->GetRecastNavMeshImpl()->SetRecastMesh(DetourMesh);
	}
}
```

# ConfigureBuildProperties

```cpp
void FRecastNavMeshGenerator::ConfigureBuildProperties(FRecastBuildConfig& OutConfig)
{
	// @TODO those variables should be tweakable per navmesh actor
	const float CellSize = DestNavMesh->GetCellSize(ENavigationDataResolution::Default);
	ensure(CellSize != 0.f);
	const float CellHeight = DestNavMesh->GetCellHeight(ENavigationDataResolution::Default);
	const float AgentHeight = DestNavMesh->AgentHeight;
	const float AgentMaxSlope = DestNavMesh->AgentMaxSlope;
	const float AgentMaxClimb = DestNavMesh->GetAgentMaxStepHeight(ENavigationDataResolution::Default);
	const float AgentRadius = DestNavMesh->AgentRadius;

	OutConfig.Reset();

	OutConfig.cs = CellSize;
	OutConfig.ch = CellHeight;
	OutConfig.walkableSlopeAngle = AgentMaxSlope;
	OutConfig.walkableHeight = FMath::CeilToInt(AgentHeight / CellHeight);
	OutConfig.walkableClimb = FMath::CeilToInt(AgentMaxClimb / CellHeight);
	OutConfig.walkableRadius = FMath::CeilToInt(AgentRadius / CellSize);
	OutConfig.maxStepFromWalkableSlope = OutConfig.cs * FMath::Tan(FMath::DegreesToRadians(OutConfig.walkableSlopeAngle));
	
	// For each navmesh resolutions, validate that AgentMaxStepHeight is high enough for the AgentMaxSlope angle
	for (int32 Index = 0; Index < (uint8)ENavigationDataResolution::MAX; Index++)
	{
		const ENavigationDataResolution Resolution = (ENavigationDataResolution)Index;
		
		const float MaxStepHeight = DestNavMesh->GetAgentMaxStepHeight(Resolution);
		const float TempCellHeight = DestNavMesh->GetCellHeight(Resolution);
		const int WalkableClimbVx = FMath::CeilToInt(MaxStepHeight / TempCellHeight);

		// Compute the required climb to prevent direct neighbor filtering in rcFilterLedgeSpansImp (minh < -walkableClimb).
		// See comment: "The current span is close to a ledge if the drop to any neighbour span is less than the walkableClimb."
		const float RequiredClimb = DestNavMesh->GetCellSize(Resolution) * FMath::Tan(FMath::DegreesToRadians(AgentMaxSlope));
		const int RequiredClimbVx = FMath::CeilToInt(RequiredClimb / TempCellHeight);
		
		if (WalkableClimbVx < RequiredClimbVx)
		{
			// This is a log since we need to let the user decide which one of the parameters needs to be changed (if any).
			UE_LOG(LogNavigationDataBuild, Log, TEXT("%s: AgentMaxStepHeight (%f) for resolution %s is not high enough in steep slopes (AgentMaxSlope is %f). "
				"Use AgentMaxStepHeight bigger than %f or a smaller AgentMaxSlope to avoid undesirable navmesh holes in steep slopes. "
				"This can also be avoided by using smaller CellSize and CellHeight."),
				*GetNameSafe(DestNavMesh), MaxStepHeight,
				*UEnum::GetDisplayValueAsText(Resolution).ToString(), AgentMaxSlope, static_cast<float>(RequiredClimbVx-1)*TempCellHeight);
		}
	}

	if (IsGeneratingLinks())
	{
		// NavLink builder configuration
		const FNavLinkGenerationJumpDownConfig& JumpDown = DestNavMesh->NavLinkJumpDownConfig;
		JumpDown.CopyToDetourConfig(OutConfig.JumpDownConfig);

		const float JumpDownSpillDistance = JumpDown.bEnabled ? JumpDown.JumpLength - JumpDown.JumpDistanceFromEdge : 0.f;
		constexpr float JumpOverSpillDistance = 0.f; //JumpOver.bEnabled ? JumpOver.JumpDistanceFromGapCenter : 0.f;		// @todo: jump over config is not exposed for now
		OutConfig.LinkSpillDistance = FMath::Max(JumpDownSpillDistance, JumpOverSpillDistance);
	}
	
	// store original sizes
	OutConfig.AgentHeight = AgentHeight;
	OutConfig.AgentMaxClimb = AgentMaxClimb;
	OutConfig.AgentRadius = AgentRadius;

	UE::NavMesh::Private::ComputeConfigBorderSizes(DestNavMesh->bGenerateNavLinks, OutConfig);

	OutConfig.maxEdgeLen = (int32)(1200.0f / CellSize);

	// hardcoded, but can be overridden by RecastNavMesh params later
	OutConfig.minRegionArea = (int32)rcSqr(0);
	OutConfig.mergeRegionArea = (int32)rcSqr(20.f);

	OutConfig.maxVertsPerPoly = (int32)MAX_VERTS_PER_POLY;
	OutConfig.detailSampleDist = 600.0f;
	OutConfig.detailSampleMaxError = 1.0f;

	OutConfig.minRegionArea = (int32)rcSqr(DestNavMesh->MinRegionArea / CellSize);
	OutConfig.mergeRegionArea = (int32)rcSqr(DestNavMesh->MergeRegionSize / CellSize);
	OutConfig.maxSimplificationError = DestNavMesh->MaxSimplificationError;
	OutConfig.simplificationElevationRatio = DestNavMesh->SimplificationElevationRatio;
	OutConfig.bPerformVoxelFiltering = DestNavMesh->bPerformVoxelFiltering;
	OutConfig.bMarkLowHeightAreas = DestNavMesh->bMarkLowHeightAreas;
	OutConfig.bUseExtraTopCellWhenMarkingAreas = DestNavMesh->bUseExtraTopCellWhenMarkingAreas;
	OutConfig.bFilterLowSpanSequences = DestNavMesh->bFilterLowSpanSequences;
	OutConfig.bFilterLowSpanFromTileCache = DestNavMesh->bFilterLowSpanFromTileCache;
	if (DestNavMesh->bMarkLowHeightAreas)
	{
		OutConfig.walkableHeight = 1;
	}

	OutConfig.bGenerateLinks = IsGeneratingLinks();

	const UNavigationSystemV1* NavSys = FNavigationSystem::GetCurrent<UNavigationSystemV1>(GetWorld());
	OutConfig.AgentIndex = NavSys ? NavSys->GetSupportedAgentIndex(DestNavMesh) : 0;

	OutConfig.tileSize = FMath::Max(FMath::TruncToInt(DestNavMesh->TileSizeUU / CellSize), 1);
	UE_CLOG(OutConfig.tileSize == 1, LogNavigation, Error, TEXT("RecastNavMesh TileSize of 1 is highly discouraged. This occurence indicates an issue with RecastNavMesh\'s generation properties (specifically TileSizeUU: %f, CellSize: %f). Please ensure their correctness.")
		, DestNavMesh->TileSizeUU, CellSize);

	OutConfig.regionChunkSize = FMath::Max(1, OutConfig.tileSize / FMath::Max(1, DestNavMesh->LayerChunkSplits));
	OutConfig.TileCacheChunkSize = FMath::Max(1, OutConfig.tileSize / FMath::Max(1, DestNavMesh->RegionChunkSplits));
	OutConfig.LedgeSlopeFilterMode = DestNavMesh->LedgeSlopeFilterMode;
	OutConfig.regionPartitioning = DestNavMesh->LayerPartitioning;
	OutConfig.TileCachePartitionType = DestNavMesh->RegionPartitioning;
}
```
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