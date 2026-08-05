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
	*/
	if (NavDataGenerator.IsValid())
	{
#if WITH_EDITOR		
		if (!IsBuildingOnLoad())
		{
			MarkPackageDirty();
		}
#endif

		NavDataGenerator->RebuildAll();
	}
}
```

# ARecastNavMesh
# OnNavigationBoundsChanged
```cpp
1 
```