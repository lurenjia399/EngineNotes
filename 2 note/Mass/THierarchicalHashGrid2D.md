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

```cpp
class TOctree2
{
	FOctreeNodeContext RootNodeContext;
	TArray<FNode> TreeNodes;//树的节点数组
	TArray<FNodeIndex> ParentLinks;
	TArray<ElementArrayType, TAlignedHeapAllocator<alignof(ElementArrayType)>> TreeElements;// 每个节点中包含的Elements
};
	
```
## 1 AddElement
```cpp
void AddElementInternal(
	FNodeIndex CurrentNodeIndex, // 添加的节点索引
	const FOctreeNodeContext& NodeContext, // octtree上下文信息
	const FBoxCenterAndExtent& ElementBounds, // 元素的Bounds
	typename TCallTraits<ElementType>::ConstReference Element,//元素
	ElementArrayType& TempElementStorage)//在细分节点过程中，临时存储节点中旧的元素
{
	/*
		增加树中当前节点的元素数量
	*/
	TreeNodes[CurrentNodeIndex].InclusiveNumElements++;
	/*
		如果向叶子节点中添加元素
	*/
	if (TreeNodes[CurrentNodeIndex].IsLeaf())
	{
		/*
			如果当前节点的元素数量+1超过了最大的数量，并且当前节点大小大于最小叶子节点大小，说明需要细分节点
		*/
		if (TreeElements[CurrentNodeIndex].Num() + 1 > OctreeSemantics::MaxElementsPerLeaf && NodeContext.Bounds.Extent.X > MinLeafExtent)
		{
			/*
				把当前节点中已有的元素右移到临时存储中
			*/
			TempElementStorage = MoveTemp(TreeElements[CurrentNodeIndex]);
			/*
				1 增加8个节点以及8个节点元素
				2 设置新增的8个节点的父节点为当前节点索引
				3 当前节点的孩子节点设为新增的8个节点最大的索引
				4 清掉当前节点的元素个数
			*/
			FNodeIndex ChildStartIndex = AllocateEightNodes();
			ParentLinks[(ChildStartIndex - 1) / 8] = CurrentNodeIndex;
			TreeNodes[CurrentNodeIndex].ChildNodes = ChildStartIndex;
			TreeNodes[CurrentNodeIndex].InclusiveNumElements = 0;
			/*
				把细分节点前的旧元素都沿着当前节点添加到树的节点中
			*/
			for (typename TCallTraits<ElementType>::ConstReference ChildElement : TempElementStorage)
			{
				const FBoxCenterAndExtent ChildElementBounds(OctreeSemantics::GetBoundingBox(ChildElement));
				AddElementInternal(CurrentNodeIndex, NodeContext, ChildElementBounds, ChildElement, TempElementStorage);
			}
			/*
				把新元素沿着当前节点添加到树的节点中
			*/
			TempElementStorage.Reset();
			AddElementInternal(CurrentNodeIndex, NodeContext, ElementBounds, Element, TempElementStorage);
			return;
		}
		/*
			如果不能细分节点了，就把新元素直接添加到当前节点中，并执行SetElementId回调
		*/
		else
		{
			int ElementIndex = TreeElements[CurrentNodeIndex].Emplace(Element);
			SetElementId(Element, FOctreeElementId2(CurrentNodeIndex, ElementIndex));	
			return;
		}
	}
	/*
		如果不是向叶子节点中添加元素
	*/
	else
	{
		/*
			判断新元素的Bounds能否被当前节点的8个子节点所容纳
		*/
		const FOctreeChildNodeRef ChildRef = 
			NodeContext.GetContainingChild(ElementBounds);
		/*
			如果当前节点的8个子节点都容纳不了，就放到当前节点中
		*/
		if (ChildRef.IsNULL())
		{
			int ElementIndex = TreeElements[CurrentNodeIndex].Emplace(Element);
			SetElementId(Element, FOctreeElementId2(CurrentNodeIndex, ElementIndex));
			return;
		}
		/*
			如果当前节点的某个子节点能容纳，就把新元素添加到子节点中
		*/
		else
		{
			FNodeIndex ChildNodeIndex = TreeNodes[CurrentNodeIndex].ChildNodes + ChildRef.Index;
			FOctreeNodeContext ChildNodeContext = NodeContext.GetChildContext(ChildRef);
			AddElementInternal(ChildNodeIndex, ChildNodeContext, ElementBounds, Element, TempElementStorage);
			return;
		}
	}
}
```


```cpp
inline void RemoveElement(FOctreeElementId2 ElementId)
	{
		checkSlow(ElementId.IsValidId());

		// Remove the element from the node's element list.
		TreeElements[ElementId.NodeIndex].RemoveAtSwap(ElementId.ElementIndex, EAllowShrinking::No);

		if (ElementId.ElementIndex < TreeElements[ElementId.NodeIndex].Num())
		{
			// Update the external element id for the element that was swapped into the vacated element index.
			SetElementId(TreeElements[ElementId.NodeIndex][ElementId.ElementIndex], ElementId);
		}

		FNodeIndex CollapseNodeIndex = INDEX_NONE;
		{
			// Update the inclusive element counts for the nodes between the element and the root node,
			// and find the largest node that is small enough to collapse.
			FNodeIndex NodeIndex = ElementId.NodeIndex;
			while (true)
			{
				TreeNodes[NodeIndex].InclusiveNumElements--;
				if (TreeNodes[NodeIndex].InclusiveNumElements < OctreeSemantics::MinInclusiveElementsPerNode)
				{
					CollapseNodeIndex = NodeIndex;
				}

				if (NodeIndex == 0)
				{
					break;
				}

				NodeIndex = ParentLinks[(NodeIndex - 1) / 8];			
			}
		}

		// Collapse the largest node that was pushed below the threshold for collapse by the removal.
		if (CollapseNodeIndex != INDEX_NONE && !TreeNodes[CollapseNodeIndex].IsLeaf())
		{
			if (TreeElements[CollapseNodeIndex].Num() < (int32)TreeNodes[CollapseNodeIndex].InclusiveNumElements)
			{
				ElementArrayType TempElementStorage;
				TempElementStorage.Reserve(TreeNodes[CollapseNodeIndex].InclusiveNumElements);
				// Gather the elements contained in this node and its children.
				CollapseNodesInternal(CollapseNodeIndex, TempElementStorage);
				TreeElements[CollapseNodeIndex] = MoveTemp(TempElementStorage);

				for (int ElementIndex = 0; ElementIndex < TreeElements[CollapseNodeIndex].Num(); ElementIndex++)
				{
					// Update the external element id for the element that's being collapsed.
					SetElementId(TreeElements[CollapseNodeIndex][ElementIndex], FOctreeElementId2(CollapseNodeIndex, ElementIndex));
				}
			}
		}
	}
```


