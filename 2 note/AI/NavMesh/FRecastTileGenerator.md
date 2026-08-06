# 
```cpp
1 FBox TileBB;// 表示Tile的边界盒子，就是一个立方体盒子。在SetUp方法中根据Tile的xy坐标合TileSize以及TotalNavVolume的高度组成的盒子
2 FBox TileBBExpandedForAgent;
```

# GatherGeometry
```cpp
void FRecastTileGenerator::GatherGeometry(const FRecastNavMeshGenerator& ParentGenerator, bool bGeometryChanged)
{
	
}
```