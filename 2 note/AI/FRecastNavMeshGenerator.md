# OnNavigationBoundsChanged
```cpp
void FRecastNavMeshGenerator::OnNavigationBoundsChanged()
{
	/*
	1 记录InclusionBounds，表示Navdata影响的Box
	2 记录TotalNavBounds，表示NavDatda影响多少个Box
	*/
	UpdateNavigationBounds();
	
	/*
	1 获取Detour数据类
	*/
	dtNavMesh* DetourMesh = DestNavMesh->GetRecastNavMeshImpl() 
		? DestNavMesh->GetRecastNavMeshImpl()->GetRecastMesh() : nullptr;
	/*
	1 条件
	*/
	if (!IsGameStaticNavMesh(DestNavMesh) && DestNavMesh->IsResizable() && DetourMesh)
	{
		
	}
}
```