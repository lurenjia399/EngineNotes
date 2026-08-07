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
	1 创建高度场的数据结构，初始化Sp
	*/
	if (!CreateHeightField(BuildContext))
	{
		return false;
	}
}
```