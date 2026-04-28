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
	
	// Bounds的zho
	const FVector Center = Bounds.GetCenter();
	
	const FVector::FReal Diameter = 
		FMath::Max(Bounds.Max.X - Bounds.Min.X, Bounds.Max.Y - Bounds.Min.Y);
	for (Location.Level = 0; Location.Level < NumLevels; Location.Level++)
	{
		const int32 DiameterCells = ClampInt32(FMath::CeilToInt(Diameter * InvCellSize[Location.Level]));
		// note that it's fine for DiameterCells to equal 0 - that would happen for 0-sized items (valid location, no extent).
		if (DiameterCells <= 1)
		{
			Location.X = ClampInt32(FMath::FloorToInt(Center.X * InvCellSize[Location.Level]));
			Location.Y = ClampInt32(FMath::FloorToInt(Center.Y * InvCellSize[Location.Level]));
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
