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
3 Add 添加
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
		// 赋值新添加这一项的Next，指向Cell的First，如果是新添加的Cell就是INDEX_NONE，不是新添加的Cell就是Items的索引
		Item.Next = Cell.First;
		// 把Items的索引保存到Cell中
		Cell.First = Idx;

		// 静态数组的用法，固定大小的数组，增加Level上的Item数量
		LevelItemCount[Location.Level]++;

		// 记录ChildItemCount，根据当前的CellLocation，找上一层Level的Cell，然后记录上一层Cell中的ChildItemCount++
		FCellLocation ParentLocation = Location;
		while (ParentLocation.Level < NumLevels - 1)
		{
			// LevelDown是用当前坐标，转换成上层Level的坐标
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
4 Query 查询
```cpp
/*
// 层级网格的深度优先遍历查询
void Query(const FBox& Bounds, OutT& OutResults) const
{
	// 用来存储Bounds最小值在哪个格子里，最大值在哪个格子里
	FCellRect Rects[NumLevels];
	FCellRectIterator Iters[NumLevels];
	int32 IterIdx = 0;

	// 
	for (int32 Level = 0; Level < NumLevels; Level++)
	{
		Rects[Level] = CalcQueryBounds(Bounds, Level);
	}
*/
```

5 
```cpp
THierarchicalHashGrid2D 最适合:
  - 大量、体量相近、持续移动的 agent
  在近似平面的世界里做邻域/重叠查询——这正是它在 Mass 里的用途(你这个文件 HTMassHashGridSubsystem 就是 MassCrowd 用的)。典型:人群避让、感知邻居查询、SmartObject 就近检索、AI 群体的空间索引。
  - 更新极其频繁(每帧成千上万个体在动)、需要 O(1) 增删移。
  - 世界很大甚至无界、不想预设 bounds。
  - Z 方向不重要(角色基本贴地)。
  - 只需要 broad-phase 候选集,narrow-phase 自己算。

OctTree 更适合:
  - 真正的 3D 查询:飞行单位、体积/空域、垂直分层明显的场景。
  - 静态或半静态几何:渲染剔除(视锥/遮挡)、光照、碰撞 broad-phase、点云( LidarPointCloudOctree)、网格绘制(TMeshPaintOctree)。
  - 空间密度高度不均:大片空旷 + 局部极密,自适应细分能显著减少查询访问的节点。
  - 需要射线求交、范围查询且希望假阳性更少。

```

# 2 OctTree
## 1

