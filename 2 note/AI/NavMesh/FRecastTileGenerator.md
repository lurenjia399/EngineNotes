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
	1 构建提速数据
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