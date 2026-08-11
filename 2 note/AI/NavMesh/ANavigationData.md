# 构建导航数据

```cpp
1 从 UNavigationSystemV1::Build 方法开始，可以通过RebuildNavigation命令调用过来
2 Build中会调用GatherNavigationBounds方法注册NavVolume。然后找到场景中的RecastNavMesh执行其RebuildAll方法。然后还要执行RecastNavMesh的EnsureBuildCompletion方法，最终会执行FRecastNavMeshGenerator::EnsureBuildCompletion的方法
3 ARecastNavMesh::RebuildAll方法中会创建出FRecastNavMeshGenerator，然后执行Generator的RebuildAll方法。Generator的RebuildAll方法中会初始化dtNavMesh和一些数据。
4 FRecastNavMeshGenerator::EnsureBuildCompletion方法，会创建出异步Task，为每个Task创建出TileGenerator，创建的时候会执行SetUp方法，Task执行的时候就会调用TileGenerator的DoWork方法。在Task执行完成后，把生成的NavMeshData导航数据通过AddTile方法添加到DtNaMeshzh
5 SetUp方法和DoWork方法都在FRecastTileGenerator文件中详细描述过，这里简化下。SetUp中就是收集自己Tile所覆盖的几何体，为体素化做准备。DoWork中就是执行管线，体素化->过滤可行走span->分水岭划分Layer->压缩Layer成CompassedLayer。然后是解压缩Layer->对每个Layer分水岭出区域->生成区域轮廓->三角化生成PolyMesh->进一步生成DetailMesh->生成NavMeshData导航数据。
```
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
	2 
	*/
	if (NavDataGenerator.IsValid())
	{
		NavDataGenerator->RebuildAll();
	}
}
```