# 
```cpp
1 FBox TileBB;// 表示Tile的边界盒子，就是一个立方体盒子。在SetUp方法中根据Tile的xy坐标合TileSize以及TotalNavVolume的高度组成的盒子
2 FBox TileBBExpandedForAgent;// 由于每个Tile在边界盒子的基础上扩展的盒子，就是需要考虑Agent半径，把TileBB盒子扩展，防止Tile边缘被认为是不能行走
3 
```

# GatherGeometry
```cpp
void FRecastTileGenerator::GatherGeometry(const FRecastNavMeshGenerator& ParentGenerator, bool bGeometryChanged)
{
	
}
```