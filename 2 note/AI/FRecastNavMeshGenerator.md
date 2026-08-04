# OnNavigationBoundsChanged
```cpp
void FRecastNavMeshGenerator::OnNavigationBoundsChanged()
{
	/*
	1 记录InclusionBounds，表示Navdata影响的范围
	2 记录TotalNavBounds，表示NavDatda影响多少个Box
	*/
	UpdateNavigationBounds();
}
```