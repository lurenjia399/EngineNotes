# 1创建
```cpp
1 由UWorldPartitionNavigationDataBuilder创建出来，在编辑器按下BuildPaths，会启动新的进程来执行Builder的Run方法，里面会执行GenerateNavigationData这个方法，根据WorldSetting中的NavigationDataChunkGridSize来找场景中和NavVolume重叠的Cell，为每一个Cell都创建一个NavigationDataChunkActor
2 创建的过程中会执行CollectNavData方法来收集导航数据，方法内部是执行的UNavigationSystemV1::FillNavigationDataChunkActor方法。在FillNavigationDataChunkActor方法中就是会给NavigationDataChunkActor创建URecastNavMeshDataChunk导航数据，导航数据就是根据ChunkActor的大小去查询相交的Tile，把Tile中的数据拷贝到DataChunk中。
3 在运行时，通过AddNavigationDataChunkToWorld和RemoveNavigationDataChunkFromWorld方法来对导航数据进行添加移除
```

# 2 收集导航数据
```cpp
/*
1 用DataChunkA
*/
void URecastNavMeshDataChunk::GetTiles(
	const FPImplRecastNavMesh* NavMeshImpl, // 场景中RecastNavMesh里的Impl
	const TArray<FNavTileRef>& TileRefs, // 和DataChunk的范围相交的Tile
	const EGatherTilesCopyMode CopyMode, // 枚举
	const bool bMarkAsAttached /*= true*/)
{
	/*
	1 清空Tiles数据
	*/
	Tiles.Empty(TileRefs.Num());
	/*
	2 遍历相交的所有Tile
	*/
	for (const FNavTileRef TileRef : TileRefs)
	{
		if (Tile && Tile->header)
		{
			// 把Tile中的数据拷贝到RawTileData中
			uint8* RawTileData = nullptr;
			if (CopyMode & EGatherTilesCopyMode::CopyData)
			{
				RawTileData = DuplicateRecastRawData(Tile->data, Tile->dataSize);
			}
			// 组装出新的临时变量
			FRecastTileData RecastTileData(Tile->dataSize, RawTileData, TileCacheData.DataSize, RawTileCacheData);
			RecastTileData.OriginalX = Tile->header->x;
			RecastTileData.OriginalY = Tile->header->y;
			RecastTileData.X = Tile->header->x;
			RecastTileData.Y = Tile->header->y;
			RecastTileData.Layer = Tile->header->layer;
			RecastTileData.bAttached = bMarkAsAttached;
			// 添加到Tiles数组中
			Tiles.Add(RecastTileData);
		}
	}
}
```

# 3 添加/移除导航数据
```cpp
void ANavigationDataChunkActor::BeginPlay()
{
	Super::BeginPlay();
	AddNavigationDataChunkToWorld();
}
void ANavigationDataChunkActor::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
	RemoveNavigationDataChunkFromWorld();
	Super::EndPlay(EndPlayReason);
}
```

```cpp
void ARecastNavMesh::OnStreamingNavDataAdded(ANavigationDataChunkActor& InActor)
{
	URecastNavMeshDataChunk* NavDataChunk = GetNavigationDataChunk(InActor);
	if (NavDataChunk)
	{
		AttachNavMeshDataChunk(*NavDataChunk);
	}
	/*
	1 运行时影响导航的情况
	2 直接通过OctTree查找DataChunkActor范围内的几何体，把几何体标记为脏的，等NavigationSystem的Tick中处理
	*/
	if (IsWorldPartitionedDynamicNavmesh())
	{
		UNavigationSystemV1* NavSys = FNavigationSystem::GetCurrent<UNavigationSystemV1>(GetWorld());
		if (NavSys)
		{
			FNavigationOctreeFilter Filter;
			Filter.bIncludeGeometry = true;
			Filter.bExcludeLoadedData = true;
		
			TArray<FNavigationOctreeElement> NavElements;
			NavSys->FindElementsInNavOctree(InActor.GetBounds(), Filter, NavElements);

			for (const FNavigationOctreeElement& NavElement : NavElements)
			{
				NavSys->AddDirtyArea(
					NavElement.Bounds.GetBox(),
					ENavigationDirtyFlag::All,
					[SourceElement = NavElement.Data->SourceElement]
					{
						return SourceElement;
					},
					"Streaming data added");
			}
		}
	}
}
```