# 1创建
```cpp
1 由UWorldPartitionNavigationDataBuilder创建出来，在编辑器按下BuildPaths，会启动新的进程来执行Builder的Run方法，里面会执行GenerateNavigationData这个方法，根据WorldSetting中的NavigationDataChunkGridSize来找场景中和NavVolume重叠的Cell，为每一个Cell都创建一个NavigationDataChunkActor
2 创建的过程中会执行CollectNavData方法来收集导航数据，方法内部是执行的UNavigationSystemV1::FillNavigationDataChunkActor方法。
3 在FillNavigationDataChunkActor方法中就是会给NavigationDataChunkActor创建URecastNavMeshDataChunk导航数据，导航数据就是根据ChunkActor的大小去查询相交的Tile，把Tile中的数据拷贝到DataChunk中。
4 在运行时，通过AddNavigationDataChunkToWorld和RemoveNavigationDataChunkFromWorld方法来对导航数据进行添加移除
```

# 2 填充DataChunk数据
```cpp
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
	1 遍历相交的所有Tile
	*/
	for (const FNavTileRef TileRef : TileRefs)
	{
		if (Tile && Tile->header)
		{
			// Make our own copy of tile data
			uint8* RawTileData = nullptr;
			if (CopyMode & EGatherTilesCopyMode::CopyData)
			{
				RawTileData = DuplicateRecastRawData(Tile->data, Tile->dataSize);
			}

			// We need tile cache data only if navmesh supports any kind of runtime generation
			FNavMeshTileData TileCacheData;
			uint8* RawTileCacheData = nullptr;
			if (CopyMode & EGatherTilesCopyMode::CopyCacheData)
			{
				TileCacheData = NavMeshImpl->GetTileCacheLayer(Tile->header->x, Tile->header->y, Tile->header->layer);
				if (TileCacheData.IsValid())
				{
					// Make our own copy of tile cache data
					RawTileCacheData = DuplicateRecastRawData(TileCacheData.GetData(), TileCacheData.DataSize);
				}
			}

			FRecastTileData RecastTileData(Tile->dataSize, RawTileData, TileCacheData.DataSize, RawTileCacheData);
			RecastTileData.OriginalX = Tile->header->x;
			RecastTileData.OriginalY = Tile->header->y;
			RecastTileData.X = Tile->header->x;
			RecastTileData.Y = Tile->header->y;
			RecastTileData.Layer = Tile->header->layer;
			RecastTileData.bAttached = bMarkAsAttached;

			Tiles.Add(RecastTileData);
		}
	}
}
```