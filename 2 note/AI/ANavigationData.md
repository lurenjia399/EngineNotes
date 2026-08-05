# RebuildAll
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
	2 qu'xia
	*/
	PostLoadPreRebuild();
}
```

# ARecastNavMesh
# OnNavigationBoundsChanged
```cpp
1 
```