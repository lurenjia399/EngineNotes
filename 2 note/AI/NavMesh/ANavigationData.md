# 构建导航数据

```cpp
1 从 UNavigationSystemV1::Build 方法开始，可以通过RebuildNavigation命令调用过来
2 Build中会调用GatherNavigationBounds方法注册NavVolume。然后找到场景中的RecastNavMesh执行其RebuildAll方法。然后还要执行RecastNavMesh的EnsureBuildCompletion方法，最终会执行FRecastNavMeshGenerator::EnsureBuildCompletion的方法
3 ARecastNavMesh::RebuildAll方法中会创建出FRecastNavMeshGenerator，然后执行Generator的RebuildAll方法。Generator的RebuildAll方法中会初始化dtNavMesh和一些数据。
4 FRecastNavMeshGenerator::EnsureBuildCompletion方法，会创建出异步Task，为每个Task创建出TileGenerator，创建的时候会执行SetUp方法，Task执行的时候就会调用TileGenerator的DoWork方法。在Task执行完成后，把生成的NavMeshData导航数据通过AddTile方法添加到DtNavMesh中。
5 SetUp方法和DoWork方法都在FRecastTileGenerator文件中详细描述过，这里简化下。SetUp中就是收集自己Tile所覆盖的几何体，为体素化做准备。DoWork中就是执行管线，体素化->过滤可行走span->分水岭划分Layer->压缩Layer成CompassedLayer。然后是解压缩Layer->对每个Layer分水岭出区域->生成区域轮廓->三角化生成PolyMesh->进一步生成DetailMesh->生成NavMeshData导航数据。
```

```cpp
// 基类的方法是这个，但是默认生成的是ARecastNavMesh这个NavData，会重写一部分方法
void ANavigationData::RebuildAll()
{
	/*
	1 ARecastNavMesh重写了，在非WP的情况下，会阻塞加载所有的LevelInstance
	2 目的是在Rebuild前处理一些东西
	*/
	LoadBeforeGeneratorRebuild();
	
	/*
	1 ARecastNavMesh重写了
	2 取消注册旧的UBaseGeneratedNavLinksProxy，创建新的并通过RegisterCustomLink方法注册到NavSystemV1中的CustomNavLinksMap这个数组里
	3 NavLinkJumpDownConfig.LinkProxyClass这个配置了才会生成
	*/
	PostLoadPreRebuild();
	/*
	1 ARecastNavMesh重写了
	2 重置NavDataGenerator，创建新的FRecastNavMeshGenerator，并初始化
	*/
	ConditionalConstructGenerator(); 
	/*
	1 如果NavDataGenerator创建成功了，这里会执行RebuildAll
	*/
	if (NavDataGenerator.IsValid())
	{
		NavDataGenerator->RebuildAll();
	}
}
```

# 序列化

# 写数据

```cpp
void ARecastNavMesh::Serialize( FArchive& Ar )
{
	/*
	1 占位
	2 缓存写数据之前的位置RecastNavMeshSizePos
	3 把0写进入去
	*/
	uint32 RecastNavMeshSizeBytes = 0;  
	int64 RecastNavMeshSizePos = Ar.Tell();  
	{  
	    Ar << RecastNavMeshSizeBytes;  
	}
	
	if (Ar.IsLoading())
	{
		// 读数据的不看
	}
	else
	{
		/*
		1 写入数据的具体方法
		2 执行RecastNavMeshImpl::Serialize方法，后面看下
		*/
		SerializeRecastNavMesh(Ar, RecastNavMeshImpl, NavMeshVersion);
		/*
		1 回填真实尺寸
		2 获取写入数据后的位置CurPos
		3 计算数据所占的大小RecastNavMeshSizeBytes = CurPos - RecastNavMeshSizePos
		4 回到占位的位置，把数据大小写入，在回到现在的位置
		*/
		if (Ar.IsSaving())
		{
			int64 CurPos = Ar.Tell();
			RecastNavMeshSizeBytes = IntCastChecked<uint32>(
				CurPos - RecastNavMeshSizePos);
			Ar.Seek(RecastNavMeshSizePos);
			Ar << RecastNavMeshSizeBytes;
			Ar.Seek(CurPos);
		}
	}
}
```

```cpp
void FPImplRecastNavMesh::Serialize( FArchive& Ar, int32 NavMeshVersion )
{
	if (Ar.IsSaving()) 
	{
		TilesToSave.Reserve(DetourNavMesh->getMaxTiles());
		/*
		1 如果是WP的NavMesh，就不用处理
		2 Tile中的导航数据都会由NavigationDataChunkActor序列化
		*/
		if (NavMeshOwner->bIsWorldPartitioned)
		{
		}
		/*
		1 如果不是WP的NavMesh
		2 如果是Static或者是Modifer类型的NavMesh，就获取NavMesh所在关卡的NavVolume所覆盖的Tile，把这些Tile记录到 TilesToSave 数组中
		3 如果是Dynamic类型的NavMesh，就遍历所有的Tile，把所有的Tile都记录到 TilesToSave 数组中
		*/
		else
		{
			if (NavMeshOwner->SupportsStreaming() 
				&& NavSys && !IsRunningCommandlet())
			{
				GetNavMeshTilesIn(NavMeshOwner->
					GetNavigableBoundsInLevel(NavMeshOwner->GetLevel()),
					TilesToSave);
			}
			else
			{
				dtNavMesh const* ConstNavMesh = DetourNavMesh;
				for (int i = 0; i < ConstNavMesh->getMaxTiles(); ++i)
				{
					const dtMeshTile* Tile = ConstNavMesh->getTile(i);
					if (Tile != NULL && Tile->header != NULL && Tile->dataSize > 0)
					{
						FNavTileRef TileRef(ConstNavMesh->getTileRef(Tile));
						TilesToSave.Add(TileRef);
					}
				}
			}
		}
		// 得到记录TilesToSave的长度，NumTiles
		NumTiles = TilesToSave.Num();
	}
	// 把需要序列化Tile的数量，Tile的原点，Tile的长度，Tile的高度，最大的Tile数量，Poly的数量等各种参数都序列化进去
	Ar << NumTiles;
	dtNavMeshParams Params = *DetourNavMesh->getParams();
	Ar << Params.orig[0] << Params.orig[1] << Params.orig[2];
	Ar << Params.tileWidth;
	Ar << Params.tileHeight;
	Ar << Params.maxTiles;
	Ar << Params.maxPolys;
	Ar << Params.walkableHeight;
	Ar << Params.walkableRadius;
	Ar << Params.walkableClimb;
	if (Ar.IsLoading())
	{
		// 读数据的不看
	}
	/*
	1 遍历需要序列化的Tile
	2 首先将每个Tile的索引，Tile所包含的数据大小序列化进去
	3 然后通过SerializeRecastMeshTile把Tile具体的数据序列化进去，如果是非Static的NavMesh，还需要将CompressedLayer上记录的数据也序列化进去
	*/
	else
	{
		for (FNavTileRef TileRefToSave : TilesToSave)
		{
			const dtMeshTile* Tile = ConstNavMesh->getTileByRef(static_cast<dtTileRef>(TileRefToSave));
			dtTileRef TileRef = ConstNavMesh->getTileRef(Tile);
			int32 TileDataSize = Tile->dataSize;
			Ar << TileRef << TileDataSize;
			unsigned char* TileData = Tile->data;
			SerializeRecastMeshTile(Ar, NavMeshVersion, TileData, TileDataSize);
			
			{
				FNavMeshTileData TileCacheLayer;
				uint8* CompressedData = nullptr;
				int32 CompressedDataSize = 0;
				if (bSupportsRuntimeGeneration)
				{
					TileCacheLayer = GetTileCacheLayer(Tile->header->x, 
						Tile->header->y, Tile->header->layer);
					CompressedData = TileCacheLayer.GetDataSafe();
					CompressedDataSize = TileCacheLayer.DataSize;
				}
				
				SerializeCompressedTileCacheData(Ar, NavMeshVersion, 
					CompressedData, CompressedDataSize);
			}
		}
	}
}
```
# 读数据

```cpp
void ARecastNavMesh::Serialize( FArchive& Ar )
{
	/*
	0 这些都和写数据一致。
	1 占位
	2 缓存写数据之前的位置RecastNavMeshSizePos
	3 把0写进入去
	*/
	uint32 RecastNavMeshSizeBytes = 0;
	int64 RecastNavMeshSizePos = Ar.Tell();
	{
		Ar << RecastNavMeshSizeBytes;
	}
	/*
	1 读数据的具体方法
	2 和写数据一样都是执行RecastNavMeshImpl::Serialize方法，后面看下
	*/
	if (Ar.IsLoading())
	{
		SerializeRecastNavMesh(Ar, RecastNavMeshImpl, NavMeshVersion);
	}
	else
	{
		// 写数据不看
	}
}
```

```cpp
void FPImplRecastNavMesh::Serialize( FArchive& Ar, int32 NavMeshVersion )
{
	// 如果有DetourNavMesh，就先释放然后创建新的
	if (Ar.IsLoading())
	{
		ReleaseDetourNavMesh();
		DetourNavMesh = dtAllocNavMesh();
	}
	// 把需要Tile的数量，Tile的原点，Tile的长度，Tile的高度，最大的Tile数量，Poly的数量等各种参数都从资源中读出来
	Ar << NumTiles;
	dtNavMeshParams Params = *DetourNavMesh->getParams();
	Ar << Params.orig[0] << Params.orig[1] << Params.orig[2];
	Ar << Params.tileWidth;
	Ar << Params.tileHeight;
	Ar << Params.maxTiles;
	Ar << Params.maxPolys;
	Ar << Params.walkableHeight;
	Ar << Params.walkableRadius;
	Ar << Params.walkableClimb;
	/*
	1 根据序列化出来的参数，首先初始化DetourNavMesh
	2 遍历所有的Tile，把TileData从资源中读出来
	3 然后通过DetourNavMesh->addTile方法，把TileData添加到DtourNavMesh中。然后紧接着读CompressedLayer的数据
	*/
	if (Ar.IsLoading())
	{
		dtStatus Status = DetourNavMesh->init(&Params);
		for (int i = 0; i < NumTiles; ++i)
		{
			dtTileRef TileRef = MAX_uint64;
			int32 TileDataSize = 0;
			Ar << TileRef << TileDataSize;
			SerializeRecastMeshTile(Ar, NavMeshVersion, TileData, TileDataSize);

			if (TileData != NULL)
			{
				dtMeshHeader* const TileHeader = (dtMeshHeader*)TileData;
				Status = DetourNavMesh->addTile(TileData, 
					TileDataSize, DT_TILE_FREE_DATA, TileRef, NULL);
				
				uint8* ComressedTileData = nullptr;
				int32 CompressedTileDataSize = 0;
				SerializeCompressedTileCacheData(Ar, NavMeshVersion,
					ComressedTileData, CompressedTileDataSize);
				
				if (CompressedTileDataSize > 0)
				{
					AddTileCacheLayer(TileHeader->x, 
						TileHeader->y, TileHeader->layer,
						FNavMeshTileData(ComressedTileData, 
							CompressedTileDataSize, TileHeader->layer, 
							Recast2UnrealBox(TileHeader->bmin, TileHeader->bmax)));
				}
			}
		}
	}
	else
	{
		// 写数据不看
	}
}
```

# 寻路

# FindPath
```cpp
FPImplRecastNavMesh::FindPath(
	const FVector& StartLoc, // 寻路的起点
	const FVector& EndLoc, // 寻路的终点
	const FVector::FReal CostLimit, // 添加到OpenList的代价
	const bool bRequireNavigableEndLocation, // 判断寻路终点是否在导航上
	FNavMeshPath& Path, // 寻路的结果
	const FNavigationQueryFilter& InQueryFilter,//UNavigationQueryFilter中过滤条件
	const UObject* Owner)// 谁发起的寻路，AIController
{
	const bool bCanSearch = InitPathfinding(StartLoc, EndLoc, 
		NavQuery, QueryFilter, RecastStartPos, StartPolyID, 
		RecastEndPos, EndPolyID, &Path);
	if (!bCanSearch)
	{
		return ENavigationQueryResult::Error;
	}
	const dtStatus FindPathStatus = NavQuery.findPath(StartPolyID, EndPolyID, 
		&RecastStartPos.X, &RecastEndPos.X, CostLimit, QueryFilter, PathResult, 0);
		
	return PostProcessPathInternal(FindPathStatus, Path, NavQuery, 
		QueryFilter, StartPolyID, EndPolyID, RecastStartPos, 
		RecastEndPos, PathResult);

}
```

```cpp
dtStatus dtNavMeshQuery::findPath(
	dtPolyRef startRef, dtPolyRef endRef,// 起点和终点所在的Poly
	const dtReal* startPos, const dtReal* endPos,//起点和终点
	const dtReal costLimit, const dtQueryFilter* filter, //限制的最大花销和过滤条件
	dtQueryResult& result, dtReal* totalCost) const// 结果和总代价
{
	// 起点Poly不存在或者是终点Poly不存在，就是起点和终点不在导航中
	if (!startRef || (m_query.requireNavigableEndLocation && !endRef))
		return DT_FAILURE | DT_INVALID_PARAM;
	// 起点Poly和终点Poly在一个Poly中
	if (startRef == endRef)
	{
		result.addItem(startRef, 0.0f, 0, 0);
		return DT_SUCCESS;
	}
	/*
	1 从节点池中申请一个起始节点
	2 把起始节点放到OpenList中
	*/
	dtNode* startNode = m_nodePool->getNode(startRef);
	dtVcopy(startNode->pos, startPos);
	startNode->pidx = 0;// 父节点的索引
	startNode->cost = 0;// gn = 从起点到当前节点的代价
	startNode->total = dtVdist(startPos, endPos) * H_SCALE;//fn = gn + hn
	startNode->id = startRef;//节点所代表的Poly
	startNode->flags = DT_NODE_OPEN;//节点是否处理过
	m_openList->push(startNode);//把节点放到OpenList中
	m_queryNodes++;//遍历到的节点数量
	// 遍历OpenList
	while (!m_openList->empty())
	{
		// 取出OpenList中节点，作为当前节点，并标记处理过
		dtNode* bestNode = m_openList->pop();
		bestNode->flags &= ~DT_NODE_OPEN;
		bestNode->flags |= DT_NODE_CLOSED;
		// 当前节点所代表的Poly就是终点Poly，直接结束循环
		if (bestNode->id == endRef)
		{
			lastBestNode = bestNode;
			break;
		}
		// 防止无限循环，循环次数够了就结束循环
		loopCounter++;
		if (loopCounter >= loopLimit * 4)
		{
			break;
		}
		// 遍历当前节点的相邻节点
		unsigned int i = bestPoly->firstLink;
		while (i != DT_NULL_LINK)
		{
			dtNode* neighbourNode = m_nodePool->getNode(neighbourRef);
			// 取相邻节点Poly和当前节点Poly共享边的中点作为相邻节点的位置
			dtReal neiPos[3] = { 0.0f, 0.0f, 0.0f };
			if (H_SCALE <= 1.0f || neighbourNode->flags == 0)
			{
				getEdgeMidPoint(bestRef, bestPoly, bestTile,
					neighbourRef, neighbourPoly, neighbourTile,
					neiPos);
			}
			else
			{
				dtVcopy(neiPos, neighbourNode->pos);
			}
			// 计算代价，
			/*
			1 计算代价
			2 CurCost是当前节点到相邻节点的代价
			3 cost是从起始节点到相邻节点的代价
			4 heuristic是相邻节点到终点的代价
			*/
			if (neighbourRef != endRef)
			{
				curCost = filter->getCost(bestNode->pos, neiPos, parentRef, 
					parentTile, parentPoly, bestRef, bestTile, 
					bestPoly, neighbourRef, neighbourTile, neighbourPoly);
				cost = bestNode->cost + curCost;
				heuristic = dtVdist(neiPos, endPos)*H_SCALE;
			}
			// fn = gn + hn
			const dtReal total = cost + heuristic;
			// 相邻节点已经在OpenList 并且 从起始点到相邻节点的代价更差，就跳过
			if ((neighbourNode->flags & DT_NODE_OPEN) 
				&& total >= neighbourNode->total)
			{
				continue;
			}
			// 相邻节点已经在OpenList 并且 从起始点到相邻节点的代价更差，就跳过
			if ((neighbourNode->flags & DT_NODE_CLOSED) &
				& total >= neighbourNode->total)
			{
				continue;
			}
			/*
			1 记录相邻节点的父节点是当前节点
			2 记录相邻节点表示的Poly
			3 记录当前当前节点访问过
			4 记录从起始点到相邻节点的代价
			5 记录起始点到相邻节点的总代价，fn = gn + hn
			6 记录相邻节点的位置
			*/
			neighbourNode->pidx = m_nodePool->getNodeIdx(bestNode);
			neighbourNode->id = neighbourRef;
			neighbourNode->flags = (neighbourNode->flags & ~DT_NODE_CLOSED);
			neighbourNode->cost = cost;
			neighbourNode->total = total;
			dtVcopy(neighbourNode->pos, neiPos);
			// 如果相邻节点已经在OpenList了就更新，如果不在就放到OpenList
			if (neighbourNode->flags & DT_NODE_OPEN)
			{
				m_openList->modify(neighbourNode);
			}
			else
			{
				neighbourNode->flags |= DT_NODE_OPEN;
				m_openList->push(neighbourNode);
				m_queryNodes++;
			}
			// 反转链表
			dtNode* prev = 0;
			dtNode* node = lastBestNode;
			int n = 1;
			do
			{
				dtNode* next = m_nodePool->getNodeAtIdx(node->pidx);
				node->pidx = m_nodePool->getNodeIdx(prev);
				prev = node;
				node = next;
			}
			while (node && ++n < loopLimit);
			// 记录结果，结果记录了从起始Poly到终点Poly所经过的Poly和相应的每一步的代价
			do
			{
				result.addItem(node->id, node->cost - prevCost, 0, 0);
				prevCost = node->cost;
		
				node = m_nodePool->getNodeAtIdx(node->pidx);
			}
			while (node && --n > 0);
		}
	}
}
```