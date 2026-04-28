1 
```cpp
/*
	// 这个5是Grid的Level，4是每个level的cellsize需要乘4，也就是最大的cellsize就是cellsize 成4的4次方
	typedef THierarchicalHashGrid2D<5, 4> FZoneGraphBuilderHashGrid2D;
*/
```
2 根据Bounds找cell
```cpp
/*
FCellLocation CalcCellLocation(const FBox& Bounds) const
{
	FCellLocation Location(0, 0, 0);
	
	// Bounds的中心点
	const FVector Center = Bounds.GetCenter();
	// Bounds的最长边
	const FVector::FReal Diameter = 
		FMath::Max(Bounds.Max.X - Bounds.Min.X, Bounds.Max.Y - Bounds.Min.Y);
	// 找到能容纳Bounds最长边的Cell
	for (Location.Level = 0; Location.Level < NumLevels; Location.Level++)
	{
		// Bounds最长边 / 每个Level下的CellSize，然后向上取整
		const int32 DiameterCells = 
			ClampInt32(FMath::CeilToInt(Diameter * InvCellSize[Location.Level]));
		// 如果向上取整结果 <= 1，说明当前level的Cell能容纳此Bounds
		if (DiameterCells <= 1)
		{
			// 设置Location，当前Bounds的中心点位于哪个grid里就赋值哪个，就是用中心点的位置 / 每个Level下的Cell
			Location.X = ClampInt32(
				FMath::FloorToInt(Center.X * InvCellSize[Location.Level]));
			Location.Y = ClampInt32(
				FMath::FloorToInt(Center.Y * InvCellSize[Location.Level]));
			break;
		}
	}

	if (Location.Level == NumLevels)
	{
		// Could not fit into any of the levels.
		Location.X = 0;
		Location.Y = 0;
		Location.Level = INDEX_NONE;
	}

	return Location;
}
*/
```
