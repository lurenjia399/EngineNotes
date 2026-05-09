# 1 RegisterZoneShapeComponent
```c++
/*
void FZoneGraphBuilder::RegisterZoneShapeComponent(
	UZoneShapeComponent& ShapeComp)
{
	// 道理一样，获取数组中空闲的索引
	int32 Index = 
		ShapeComponentsFreeList.IsEmpty() ? 
		ShapeComponents.AddDefaulted() : ShapeComponentsFreeList.Pop();
	// 拿到索引中的BuilderComp对象
	FZoneGraphBuilderRegisteredComponent& Registered = ShapeComponents[Index];
	// 组装BuilderComp对象，缓存Comp指针
	Registered.Component = &ShapeComp;
	// 组装BuilderComp对象，计算Comp的Bounds
	const FBoxSphereBounds Bounds = 
		ShapeComp.CalcBounds(ShapeComp.GetComponentTransform());
	// 将Bounds放到2D的Grid平面上，计算出位于哪个格子中，将结果记录到BuilderComp对象里
	Registered.CellLoc = HashGrid.Add(uint32(Index), Bounds.GetBox());
	// 缓存Comp到数组索引的map，加速查询
	ShapeComponentToIndex.Add(&ShapeComp, Index);
	// 创建一个唯一的hash值，也能通过这个值来检测到BuilderComp对象是否改变
	Registered.ShapeHash = ShapeComp.GetShapeHash();
	// 标记为脏的
	bIsDirty = true;
}

void FZoneGraphBuilder::UnregisterZoneShapeComponent(
	UZoneShapeComponent& ShapeComp)
{
	// 从数组中找Comp
	if (int32* Index = ShapeComponentToIndex.Find(&ShapeComp))
	{
		// 拿到BuilderComp对象
		FZoneGraphBuilderRegisteredComponent& Registered = ShapeComponents[*Index];
		// 从HashGrid中移除掉
		HashGrid.Remove(uint32(*Index), Registered.CellLoc);
		// 从加速结构中移除掉
		ShapeComponentToIndex.Remove(Registered.Component);
		// 置空
		Registered.Component = nullptr;
		ShapeComponentsFreeList.Add(*Index);
	}

	bIsDirty = true;
}
*/
```
注册，每个UZoneShapeComponent的onRegister方法或者是onUnRegister就会执行。会把这个Component放到Builder的ShapeComponents数组中缓存起来。
# 2 Build
```cpp

void FZoneGraphBuilder::Build(AZoneGraphData& ZoneGraphData)
{
	
	ULevel* CurrentLevel = ZoneGraphData.GetLevel();
	ZoneGraphData.Modify();
	
	// 重置Storage中的所有数据
	FZoneGraphStorage& ZoneStorage = ZoneGraphData.GetStorageMutable();
	ZoneStorage.Reset();
	// 将注册的BuilderComp对象里数据，整理到Builder中,AppendShapeToZoneStorage主要是这个方法
	for (FZoneGraphBuilderRegisteredComponent& Registered : ShapeComponents)
	{
		if (Registered.Component && Registered.Component->GetComponentLevel() == CurrentLevel)
		{
			AppendShapeToZoneStorage(
				*Registered.Component, 
				Registered.Component->GetComponentTransform().ToMatrixWithScale(),
				ZoneStorage, 
				InternalLinks, 
				&BuildData);
		}
	}
	// Connect Lanes
	ConnectLanes(InternalLinks, ZoneStorage);

	// Update bounds for the whole data
	ZoneStorage.Bounds = FBox(ForceInit);
	for (FZoneData& Zone : ZoneStorage.Zones)
	{
		ZoneStorage.Bounds += Zone.Bounds;
	}

	// Build BV-tree for faster zone lookup.
	ZoneStorage.ZoneBVTree.Build(
		MakeStridedView(ZoneStorage.Zones, &FZoneData::Bounds));

	// Update combined hash
	const uint32 NewHash = CalculateCombinedShapeHash(ZoneGraphData);
	ZoneGraphData.SetCombinedShapeHash(NewHash);

	ZoneGraphData.UpdateDrawing();
}

```
## 2.1 AppendShapeToZoneStorage

```cpp
void FZoneGraphBuilder::AppendShapeToZoneStorage(
	const UZoneShapeComponent& ShapeComp, 
	const FMatrix& LocalToWorld,
	FZoneGraphStorage& OutZoneStorage,
	TArray<FZoneShapeLaneInternalLink>& OutInternalLinks,
	FZoneGraphBuildData* InBuildData)
 {
	// 获取来源ShapeComp上的Points，赋值到设置临时变量AdjustedPoints里
	TArray<FZoneShapePoint> AdjustedPoints(ShapeComp.GetPoints());
	// 获取来源ShapeComp上的Connector，这个Connector是根据ShapeComp的形状来的，表示可连接点
	TConstArrayView<FZoneShapeConnector> ShapeConnectors = 
		ShapeComp.GetShapeConnectors();
	// 获取来源ShapeComp上的Connetion，这个是记录Connector链接的目标上的信息
	TConstArrayView<FZoneShapeConnection> ConnectedShapes = 
		ShapeComp.GetConnectedShapes();
	// 如果来源ShapeComp上所有可连接点都链接了
	if (ConnectedShapes.Num() == ShapeConnectors.Num())
	{
		// Comp身上的Transform是CompToWorld的矩阵，取反就是WorldToComp的矩阵了
		FTransform WorldToSource = ShapeComp.GetComponentTransform().Inverse();
		// 遍历所有的来源连接点
		for (int32 i = 0; i < ShapeConnectors.Num(); i++)
		{
			// 当前遍历到的来源连接点
			const FZoneShapeConnector& SourceConnector = ShapeConnectors[i];
			// 来源ShapeComp的形状
			const FZoneShapeType SourceShapeType = ShapeComp.GetShapeType();
			// 当前遍历到的来源连接
			const FZoneShapeConnection& Connection = ConnectedShapes[i];
			// 根据来源连接取出连接目标的ShapeComp
			if (const UZoneShapeComponent* DestShapeComp = 
					Connection.ShapeComponent.Get())
			{
				// 目标ShapeComp身上所有的连接点
				TConstArrayView<FZoneShapeConnector> DestConnectors = 
					DestShapeComp->GetShapeConnectors();
				// 找到当前与来源连接点连接的目标连接点
				const FZoneShapeConnector& DestConnector = 
					DestConnectors[Connection.ConnectorIndex];
				// 目标ShapeComp的形状
				const FZoneShapeType DestShapeType = DestShapeComp->GetShapeType();

				// 根据来源ShapeComp和目标ShapeComp确定连接的两个点的混合因子
				float BlendFactor = 1.0f;
				if (SourceShapeType == FZoneShapeType::Spline)
				{
					if (DestShapeType == FZoneShapeType::Spline)
					{
						BlendFactor = 0.5f;
					}
					else // Polygon
					{
						BlendFactor = 0.0f;
					}
				}
				else // Polygon
				{
					if (DestShapeType == FZoneShapeType::Spline)
					{
						BlendFactor = 1.0f;
					}
					else // Polygon
					{
						BlendFactor = 0.5f;
					}
				}
				// 有混合因子，所以需要调整来源ShapeComp上的Point位置
				if (BlendFactor > 0.0f)
				{
					// 获取目标ShapeComp空间变换到来源ShapeComp的矩阵
					const FTransform DestToSource = 
						DestShapeComp->GetComponentTransform() * WorldToSource;
					// 将目标连接点位置转到来源ShapeComp空间下
					const FVector LocalDestPosition = 
						DestToSource.TransformPosition(DestConnector.Position);
					// 将目标连接点前向向量转到来源ShapeComp空间下
					const FVector LocalDestNormal = 
						DestToSource.TransformVector(DestConnector.Normal);
					// 将目标连接点Up向量转到来源ShapeComp空间下
					const FVector LocalDestUp = 
						DestToSource.TransformVector(DestConnector.Up);
					// 将来源连接点信息插值到目标来源点
					const FVector NewPosition = FMath::Lerp(
							SourceConnector.Position, 
							LocalDestPosition, 
							BlendFactor);
					const FVector NewNormal = FMath::Lerp(
						SourceConnector.Normal, 
						-LocalDestNormal, // 这里为啥是取负数？
						BlendFactor).GetSafeNormal();
					const FVector NewUp = FMath::Lerp(
						SourceConnector.Up, 
						LocalDestUp, 
						BlendFactor).GetSafeNormal();

					if (SourceShapeType == FZoneShapeType::Spline)
					{
						// Adjust spline extremity.
						FZoneShapePoint& Point = 
							AdjustedPoints[SourceConnector.PointIndex];
						Point.Position = NewPosition;
						const float FlipNormal =
							SourceConnector.PointIndex == 0 ? -1.0f : 1.0f;
						Point.SetRotationFromForwardAndUp(
							NewNormal * FlipNormal, NewUp);
					}
					else
					{
						// Adjust polygon lane segment.
						FZoneShapePoint& Point = 
							AdjustedPoints[SourceConnector.PointIndex];
						Point.Position = NewPosition;
						Point.SetRotationFromForwardAndUp(-NewNormal, NewUp);
					}
				}
			}
		}
	}
	int32 ZoneIndex = OutZoneStorage.Zones.Num();
	if (ShapeComp.GetShapeType() == FZoneShapeType::Spline)
	{
		// 获取样条线的车道
		FZoneLaneProfile SplineLaneProfile;
		ShapeComp.GetSplineLaneProfile(SplineLaneProfile);
		if (ShapeComp.IsLaneProfileReversed())
		{
			SplineLaneProfile.ReverseLanes();
		}
		// 将样条线细分成点的方法
		UE::ZoneShape::Utilities::TessellateSplineShape(
			AdjustedPoints, //上边我们Blend的来源连接点
			SplineLaneProfile, //车道配置
			ShapeComp.GetTags(), //来源ShapeComp身上的Tag
			LocalToWorld, //来源ShapeComp转换到世界的矩阵
			OutZoneStorage, 
			OutInternalLinks);
	}
	// 记录映射，方便查询
	// Build mapping data between ZoneShapeComponents and baked data.
	if (InBuildData && ZoneIndex != INDEX_NONE)
	{
		FZoneData& Zone = OutZoneStorage.Zones[ZoneIndex];

		FZoneShapeComponentBuildData ComponentBuildData;
		ComponentBuildData.ZoneIndex = ZoneIndex;
		for (int32 i = Zone.LanesBegin; i != Zone.LanesEnd; i++)
		{
			ComponentBuildData.Lanes.Add(
				FZoneGraphLaneHandle(i, OutZoneStorage.DataHandle));
		}

		InBuildData->ZoneShapeComponentBuildData.Add(
			&ShapeComp, ComponentBuildData);
	}
 }

```
## 2.2 void FZoneGraphBuilder::ConnectLanes 
```cpp
void FZoneGraphBuilder::ConnectLanes(
	TArray<FZoneShapeLaneInternalLink>& InternalLinks,// 离散化
	FZoneGraphStorage& ZoneStorage)
{
	enum class ELaneExtremity : uint8
	{
		Start,
		End,
	};

	struct FLanePointID
	{
		FLanePointID() = default;
		FLanePointID(const int32 InIndex, const ELaneExtremity InExtremity) 
		: Index(InIndex), Extremity(InExtremity) {}

		int32 Index;
		ELaneExtremity Extremity;
	};

	// 用于快速查询的缓存，key是车道在ZoneStorage里的索引，value是在InternalLink中索引
	TMap<int32, int32> FirstLinkByLane;
	// 根据Link中每个车道在ZoneStorage里索引排序
	InternalLinks.Sort();
	// 组装FirstLinkByLane这个结构
	int32 PrevLaneIndex = INDEX_NONE;
	for (int32 LinkIdx = 0; LinkIdx < InternalLinks.Num(); LinkIdx++)
	{
		const FZoneShapeLaneInternalLink& Link = InternalLinks[LinkIdx];
		if (Link.LaneIndex != PrevLaneIndex)
		{
			FirstLinkByLane.Add(Link.LaneIndex, LinkIdx);
			PrevLaneIndex = Link.LaneIndex;
		}
	}
	// 创建一个Grid2D结构，level为1，每层lelvel是上一层的1倍，每个格子的key是FLanePointID
	THierarchicalHashGrid2D<1, 1, FLanePointID> LinkGrid(100.0f);
	// 初始化静态局部变量，注意只会初始化一次
	static const float ConnectionTolerance = 2.0f;
	static const float ConnectionToleranceSqr = FMath::Square(ConnectionTolerance);
	static const FVector ConnectionToleranceExtent(ConnectionTolerance);
	// 每个车道取开始点和结束点组成两个Box，添加到Grid2D里
	for (int32 LaneIdx = 0; LaneIdx < ZoneStorage.Lanes.Num(); LaneIdx++)
	{
		FZoneLaneData& Lane = ZoneStorage.Lanes[LaneIdx];
		LinkGrid.Add(
			FLanePointID(LaneIdx, ELaneExtremity::Start),
			FBox::BuildAABB(
				ZoneStorage.LanePoints[Lane.PointsBegin], FVector::ZeroVector));
		LinkGrid.Add(
			FLanePointID(LaneIdx, ELaneExtremity::End),
			FBox::BuildAABB(
				ZoneStorage.LanePoints[Lane.PointsEnd - 1], FVector::ZeroVector));
	}
	// 遍历这个ZoneStorage上的所有车道
	TArray<FLanePointID> QueryResults;
	for (int32 LaneIndex = 0; LaneIndex < ZoneStorage.Lanes.Num(); LaneIndex++)
	{
		// 引用当前遍历到的车道
		FZoneLaneData& Lane = ZoneStorage.Lanes[LaneIndex];
		// 往车道里记录车道link开始的索引
		Lane.LinksBegin = ZoneStorage.LaneLinks.Num();
		// 这部分没想明白还
		// Add internal links
		int32 AdjacentLaneCount = 0;
		if (const int32* FirstLink = FirstLinkByLane.Find(LaneIndex))
		{
			for (int32 LinkIdx = *FirstLink; 
					LinkIdx < InternalLinks.Num(); LinkIdx++)
			{
				const FZoneShapeLaneInternalLink& Link = InternalLinks[LinkIdx];
				if (Link.LaneIndex != LaneIndex)
				{
					break;
				}
				ZoneStorage.LaneLinks.Add(Link.LinkData);
				if (Link.LinkData.Type == EZoneLaneLinkType::Adjacent)
				{
					AdjacentLaneCount++;
				}
			}
		}
		// 拷贝当前遍历到的车道
		// Add links to connected lanes
		const FZoneLaneData& SourceLane = ZoneStorage.Lanes[LaneIndex];
		// 当前遍历车道的起始位置
		const FVector& SourceStartPosition = 
			ZoneStorage.LanePoints[SourceLane.PointsBegin];
		// 当前遍历车道的终点位置
		const FVector& SourceEndPosition = 
			ZoneStorage.LanePoints[SourceLane.PointsEnd - 1];

		// 在起始位置创建盒子，查询和这个盒子相交的FLanePointID
		QueryResults.Reset();
		LinkGrid.Query(
			FBox::BuildAABB(
				SourceStartPosition, ConnectionToleranceExtent), QueryResults);
		for (FLanePointID LaneID : QueryResults)  
		{
			// 自己不用
			if (LaneID.Index == LaneIndex)
			{
				continue;
			}
			// 相交的除了自己，的另一个车道。也就是和遍历到的车道起始点链接的另一个车道
			const FZoneLaneData& DestLane = ZoneStorage.Lanes[LaneID.Index];
			const FVector& DestStartPosition = 
				ZoneStorage.LanePoints[DestLane.PointsBegin];
			const FVector& DestEndPosition = 
				ZoneStorage.LanePoints[DestLane.PointsEnd - 1];
			// 目标车道的tag符合条件
			if (SourceLane.Tags.ContainsAny(
				DestLane.Tags & BuildSettings.LaneConnectionMask))
			{
				// 如果当前车道和目标车道不是一个车道 && 目标车道是终点 && 当前车道起始点和目标车道终点距离满足阈值，则是入口车道，就是两个车道相连并同向
				if (SourceLane.ZoneIndex != DestLane.ZoneIndex
					&& LaneID.Extremity == ELaneExtremity::End
					&& FVector::DistSquared(SourceStartPosition, DestEndPosition) 
						< ConnectionToleranceSqr)
				{
					// Incoming lane
					FZoneLaneLinkData& Link = 
						ZoneStorage.LaneLinks.AddDefaulted_GetRef();
					Link.DestLaneIndex = LaneID.Index;
					Link.Type = EZoneLaneLinkType::Incoming;
					Link.SetFlags(EZoneLaneLinkFlags::None);
				}
				else if (
					SourceLane.ZoneIndex == DestLane.ZoneIndex 
					&& LaneID.Extremity == ELaneExtremity::Start
					&& FVector::DistSquared(SourceStartPosition, estStartPosition) 
						< ConnectionToleranceSqr)
				{
					// Splitting lane
					FZoneLaneLinkData& Link = ZoneStorage.LaneLinks.AddDefaulted_GetRef();
					Link.DestLaneIndex = LaneID.Index;
					Link.Type = EZoneLaneLinkType::Adjacent;
					Link.SetFlags(EZoneLaneLinkFlags::Splitting);
				}
			}
		}
	}
}
```