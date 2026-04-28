1 
```cpp
/*
	// 这个5是Grid的Level，4是每个level的cellsize需要乘4，也就是最大的cellsize就是cellsize 成4的4次方
	typedef THierarchicalHashGrid2D<5, 4> FZoneGraphBuilderHashGrid2D;
*/
```
2 CalcCellLocation 根据Bounds找cell
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
			// 设置Location，当前Bounds的中心点位于哪个grid里就赋值哪个，就是用中心点的位置 / 每个Level下的CellSize，然后向下取整
			Location.X = ClampInt32(
				FMath::FloorToInt(Center.X * InvCellSize[Location.Level]));
			Location.Y = ClampInt32(
				FMath::FloorToInt(Center.Y * InvCellSize[Location.Level]));
			break;
		}
	}

	// 如果所有的grid都没有找到，就设成初始值
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
3 Add 添加使用
```cpp
/*
void Add(const ItemIDType ID, const FCellLocation& Location)
{
	// 在稀疏数组中添加一项
	const int32 Idx = Items.AddUninitialized().Index;
	FItem& Item = Items[Idx];
	// 赋值新添加这一项的ID
	Item.ID = ID;

	if (Location.Level == INDEX_NONE)
	{
		// Could not fit into any of the grids, add to spill list.
		Item.Next = SpillList;
		SpillList = Idx;
	}
	else
	{
		// 根据CellLocation找到Cell
		FCell& Cell = FindOrAddCell(Location.X, Location.Y, Location.Level);
		// 赋值新添加这一项的Next，指向Cell的First，如果是新添加的Cell就是INDEX_NONE，不是心
		Item.Next = Cell.First;
		Cell.First = Idx;

		// Update per level counts
		LevelItemCount[Location.Level]++;

		// Update child counts
		FCellLocation ParentLocation = Location;
		while (ParentLocation.Level < NumLevels - 1)
		{
			ParentLocation.X = LevelDown(ParentLocation.X);
			ParentLocation.Y = LevelDown(ParentLocation.Y);
			ParentLocation.Level++;
			FCell& ParentCell = FindOrAddCell(ParentLocation.X, ParentLocation.Y, ParentLocation.Level);
			ParentCell.ChildItemCount++;
		}
	}
}
*/
```
