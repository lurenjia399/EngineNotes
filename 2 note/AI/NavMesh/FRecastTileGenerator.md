# 
```cpp
1 RecastNavMeshGenerator的ProcessTileTasksAsyncAndGetUpdatedTiles方法触发，方法中会创建出Task，Task的执行的内容就是调用RecastTileGenerator的DoWork方法。
2 在创建Task的时候，传的参数是RecastTileGenerator，也就是会创建TileGenerator，就是会执行SetUp方法。
3 SetUp方法的内容就是，根据Tile的大小在NavOctTree中查到，查到的结果就是Tile所覆盖的Actor或者是Comp，然后遍历结果，将每个结果的Geometry都收集到RawGeometry数组中。也就是收集几何体数据为体素话做准备。
4 DoWork方法的内容就是GenerateTile，
```

# 
```cpp
1 FBox TileBB;// 表示Tile的边界盒子，就是一个立方体盒子。在SetUp方法中根据Tile的xy坐标合TileSize以及TotalNavVolume的高度组成的盒子
2 FBox TileBBExpandedForAgent;// 由于每个Tile在边界盒子的基础上扩展的盒子，就是需要考虑Agent半径，把TileBB盒子扩展，防止Tile边缘被认为是不能行走

3 TNavStatArray<FBox> InclusionBounds;// 存储的是和自己Tile相交的NavVolume
4 uint32 bFullyEncapsulatedByInclusionBounds;// InclusionBounds中记录的NavVolume是否完全包含自己的Tile

5 TArray<FNavMeshTileData> CompressedLayers;
```

# GatherGeometry
```cpp
void FRecastTileGenerator::GatherGeometry(const FRecastNavMeshGenerator& ParentGenerator, bool bGeometryChanged)
{
	// 拿到自己的Tile包含扩展的盒子大小
	const FBox NewBounds = ParentGenerator.GrowBoundingBox(TileBB, false);
	// 通过八叉树的缓存结构，根据Tile扩展盒子大小查找，找到相交的元素记录到RelevantDataArray中。最终找到的结果就是场景中的影响导航的Actor，Comp之类的
	NavigationOctree->FindElementsWithBoundsTest(NewBounds,[]()
	{
		RelevantDataArray.Add(Element.Data);
	});
	/*
	1 为找到的影响导航Actor，构建体素数据缓存，每个Tile会对应一个体素数据缓存
	2 遍历影响导航的Actor，对每一个Actor遍历Collision组成的三角形，将三角形投影到xz平面找到覆盖的体素列，每个体素列和三角形相交之间的体素就组成span
	3 生成Span，每个Span记录了是位于哪个体素列，从高度上数包含了哪些体素，以及是否可行走的标志
	*/
	for (TSharedRef<FNavigationRelevantData, ESPMode::ThreadSafe>& ElementData : RelevantDataArray)
	{
		GatherNavigationDataGeometry(
			ElementData, *NavSys, OwnerNavDataConfig, bGeometryChanged);

		const ENavigationDataResolution Resolution = ElementData->Modifiers.GetNavMeshResolution();
		if (Resolution != ENavigationDataResolution::Invalid)
		{
			HighestResolution = FMath::Max(HighestResolution, Resolution);
			bNewResolutionFound = true;
		}
	}
}
```

# GenerateTile
```cpp
bool FRecastTileGenerator::GenerateTile()
{
	if (bRegenerateCompressedLayers)
	{
		CompressedLayers.Reset();
		bSuccess = GenerateCompressedLayers(BuildContext, LinkBuiderData);
		if (bSuccess)
		{
			// Mark all layers as dirty
			DirtyLayers.Init(true, CompressedLayers.Num());
		}
	}
	
	if (bSuccess)
	{
		bSuccess = GenerateNavigationData(BuildContext, LinkBuiderData);
	}
	
	return bSuccess;
}
```
## GenerateCompressedLayers
```cpp
bool FRecastTileGenerator::GenerateCompressedLayers(
	FNavMeshBuildContext& BuildContext, const dtLinkBuilderData& InLinkBuilderData)
{
	/*
	1 创建高度场的数据结构，初始化Spans的内存池，这里只是初始化内存，内存中没有数据。最终在体素化之后会存下这个Tile里面所有的Span。
	*/
	if (!CreateHeightField(BuildContext))
	{
		return false;
	}
	/*
	1 计算是否要投影到地面标志位，遍历Tile中几何体的三角形的时候，如果是默认的就是不投影，会根据三角形位置生成span就是会悬空。如果几何体配置了FillCollisionUnderneathForNavmesh，就会向下投影到地面。如果NavModiferVolume勾选了这个标志位，就是Volume框选的位置不会投影到地面。
	2 比如一个房子的，我们的屋顶是默认的就是不投影到地面，房间内部会生成导航。
	3 如果不想让房间内部生成导航就可以设置屋顶是投影的，房间内部就不会生成导航。
	4 如果想让房间内部的走廊有导航其他地方没有导航，可以让屋顶投影让所有的地方都没有导航，再用标记MaskFillCollisionUnderneathForNavmesh的ModiferVolume包裹走廊部分的房顶，就可以去掉走廊部分的投影，让走廊部分有导航。ModiferVolume上的MaskFillCollisionUnderneathForNavmesh作用就是清楚包裹的Spans上的投影标记。
	*/
	ComputeRasterizationMasks(BuildContext, RasterContext);
	/*
	1 体素化，遍历Tile中包含几何体的三角形，找到其覆盖的体素组成Span,保存到高度场的Spans中。
	2 每个Span表示在一个体素列上的区间。这个区间是由一个或多个体素块组成。
	*/
	RasterizeTriangles(BuildContext, RasterContext);
	/*
	1 如果没有高度场
	2 如果高度场中的Spans数组是空的
	3 说明这个Tile中没有几何体，就提前返回掉
	*/
	if (!SolidHF || SolidHF->pools == 0)
	{
		return true;
	}
	/*
	1 一个过滤，过滤掉生成在Tile之外的Span，把这些Span标记成不可走
	2 执行过滤的条件一个是配置，是默认开启的，第二个条件是当前的NavVolume是否完全包含了Tile，只有每完全包含才需要执行过滤
	*/
	if (TileConfig.bPerformVoxelFiltering && !bFullyEncapsulatedByInclusionBounds)
	{
		ApplyVoxelFilter(SolidHF, TileConfig.walkableRadius);
	}
	/*
	1 过滤可行走表面
	2 rcFilterLowHangingWalkableObstacles 沿着每个体素列从下往上遍历Span，如果当前Span不能走但是上一个Span可以走，并且高度差小于walkableClimb,也把这个不能走的Span标记成可以走。
	3 rcFilterLedgeSpans 如果当前Span可以走，但是距离他邻居的高度差超过了walkableClimb，说明当前Span没法站人，需要标记成不可走。
	4 rcFilterWalkableLowHeightSpans 同一个体素列上当前Span的顶面距离下一个Span的地面的空间，这个空间站不下一个Agent，就要把当前Span标记成不可走。
	5 rcFilterWalkableLowHeightSpansSequences 如果空间站不下一个Agent，但是可以让Agent蹲下通过，也不会把Span标记成不可以走。
	*/
	GenerateRecastFilter(BuildContext);
	/*
	1 压缩高度场中的Spans，变成CompactSpan，重新组织Span记录的数据并记到rcCompactHeightfield中。
	2 rcCompactCell 是每一个体素列一个，其中Index表示最下边的Span在总的Spans数组中的索引，count表示这个体素列上有多少个Span。
	3 rcCompactSpan 是新的Span结构，其中y表示Span地面高度，h表示Span顶面高度 - Span地面高度，con表示相邻的CompactSpan，areas表示那个是否可行走的标志位。
	*/
	if (!BuildCompactHeightField(BuildContext))
	{
		return false;
	}
	/*
	1 收缩可行走区域，不能紧贴着边缘，如果紧贴着边缘就会导致Agent陷进墙里
	*/
	if (!RecastErodeWalkable(BuildContext))
	{
		return false;
	}
	/*
	1 构建分层，将体素列上的Span都按照层分开，就是将离散的Span按高度整合成Layer，用的是WATERSHED这个算法
	*/
	if (!RecastBuildLayers(BuildContext, RasterContext))
	{
		return false;
	}
	/*
	1 填充到CompressedLayers，把Layer在进行整合，把数据都存在CompressedLayers里
	*/
	return RecastBuildTileCache(BuildContext, RasterContext);
}
```