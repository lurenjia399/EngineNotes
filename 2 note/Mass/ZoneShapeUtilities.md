# 1 TessellateSplineShape
```cpp

void TessellateSplineShape(
	TConstArrayView<FZoneShapePoint> Points, //需要调整的所有点
	const FZoneLaneProfile& LaneProfile, //车道
	const FZoneGraphTagMask ZoneTags, 
	const FMatrix& LocalToWorld,
	FZoneGraphStorage& OutZoneStorage, 
	TArray<FZoneShapeLaneInternalLink>& OutInternalLinks)
{
	const UZoneGraphSettings* ZoneGraphSettings = GetDefault<UZoneGraphSettings>();
	const FZoneGraphBuildSettings& BuildSettings = 
		ZoneGraphSettings->GetBuildSettings();

	const bool bClosedShape = false;

	// 添加一个新的Zone
	const int32 ZoneIndex = OutZoneStorage.Zones.Num();
	FZoneData& Zone = OutZoneStorage.Zones.AddDefaulted_GetRef();
	Zone.Tags = ZoneTags;

	// 将spline离散成点的误差，是直接读的ZoneGraphSetting里面的配置
	const float TessTolerance = GetMinLaneProfileTessellationTolerance(
		LaneProfile, ZoneTags, BuildSettings);

	// Flatten spline segments to points.
	// 离散成点
	TArray<FShapePoint> CurvePoints;
	FlattenSplineSegments(Points, bClosedShape, LocalToWorld, TessTolerance, CurvePoints);

	// Remove points which are too close to each other.
	// 移除靠的特别近的点
	RemoveDegenerateSegments(CurvePoints, bClosedShape);

	// Calculate edge normals.
	// 计算点的edge normal，也就是UP和Forword叉乘的结果
	TArray<FVector> EdgeNormals;
	CalculateEdgeNormals(CurvePoints, EdgeNormals);

	// 重新计算赋值开始点和最终点的edge Normal
	CalculateStartAndEndNormals(Points, LocalToWorld, EdgeNormals[0], EdgeNormals[EdgeNormals.Num() - 1]);

	// 计算点的Right向量
	// Calculate miter extrusion at vertices
	CalculateMiters(CurvePoints, EdgeNormals);

#if HOTTA_ENGINE_MODIFY
	//���ŷ��߷���ƽ��OffsetAlongNormal
	for (int32 i = 0; i < CurvePoints.Num(); i++)
	{
		FShapePoint& Point = CurvePoints[i];
		Point.Position = Point.Position + Point.Right * OffsetAlongNormal;
	}
#endif

	// 计算道路的边缘点
	Zone.BoundaryPointsBegin = OutZoneStorage.BoundaryPoints.Num();
	const float TotalWidth = LaneProfile.GetLanesTotalWidth();
	const float HalfWidth = TotalWidth * 0.5f;
	for (int32 i = 0; i < CurvePoints.Num(); i++)
	{
		const FShapePoint& Point = CurvePoints[i];
		OutZoneStorage.BoundaryPoints.Add(Point.Position - Point.Right * HalfWidth);
	}
	for (int32 i = CurvePoints.Num() - 1; i >= 0; i--)
	{
		const FShapePoint& Point = CurvePoints[i];
		OutZoneStorage.BoundaryPoints.Add(Point.Position + Point.Right * HalfWidth);
	}
	Zone.BoundaryPointsEnd = OutZoneStorage.BoundaryPoints.Num();

	// Build lanes
	// 拿到ShapeComp第一个点的Forward向量，转换到世界坐标，这个就是起始点车的行驶方向
	const FVector StartForward = 
		LocalToWorld.TransformVector(
			Points[0].Rotation.RotateVector(
				FVector::ForwardVector).GetSafeNormal());
	// 拿到ShapeComp最后一个点的Forward向量，转换到世界坐标，这个就是终止点车的行驶方向
	const FVector EndForward = 
		LocalToWorld.TransformVector(
			Points.Last().Rotation.RotateVector(
				FVector::ForwardVector).GetSafeNormal());

	TArray<FShapePoint> LanePoints;
	Zone.LanesBegin = OutZoneStorage.Lanes.Num();
	const int32 NumLanes = LaneProfile.Lanes.Num();

	const uint16 FirstPointID = 0;
	const uint16 LastPointID = uint16(Points.Num() - 1);
	
	float CurWidth = 0.0f;
	// 遍历ShapeComp的车道配置上的所有车道
	for (int32 i = 0; i < NumLanes; i++)
	{
		const FZoneLaneDesc& LaneDesc = LaneProfile.Lanes[i];

		// Skip spacer lanes, but apply their width.
		if (LaneDesc.Direction == EZoneLaneDirection::None)
		{
			CurWidth += LaneDesc.Width;
			continue;
		}
		// 在ZoneStorage的Lanes上添加新的，并赋值数据
		FZoneLaneData& Lane = OutZoneStorage.Lanes.AddDefaulted_GetRef();
		Lane.ZoneIndex = ZoneIndex;
		Lane.Width = LaneDesc.Width;
		Lane.Tags = LaneDesc.Tags | ZoneTags;
		// Store which inputs points corresponds to the lane start/end point.
		Lane.StartEntryId = FirstPointID;
		Lane.EndEntryId = LastPointID;
		const int32 CurrentLaneIndex = OutZoneStorage.Lanes.Num() - 1;

		// 添加车道内部的链接，用来查看是否可变道，逆行
		/*
		1 每一条车道都会检测相邻的车道，看相邻的车道是否反向或者同向，并且记录相邻的车道是坐车道还是右车道
		*/
		// Add internal adjacent links.
		AddAdjacentLaneLinks(CurrentLaneIndex, i, LaneProfile.Lanes, OutInternalLinks);
		// 车道的一半
		const float LanePos = HalfWidth - (CurWidth + LaneDesc.Width * 0.5f);

		// 将point沿着right向量平移到车道箭头上，然后把结果记录到LanPoints上
		// Create lane points.
		LanePoints.Reset();
		if (LaneDesc.Direction == EZoneLaneDirection::Forward)
		{
			for (int32 j = 0; j < CurvePoints.Num(); j++)
			{
				const FShapePoint& Point = CurvePoints[j];
				FShapePoint& NewPoint = LanePoints.Add_GetRef(Point);
				NewPoint.Position += Point.Right * LanePos;
			}
		}
		else if (LaneDesc.Direction == EZoneLaneDirection::Backward)
		{
			Swap(Lane.StartEntryId, Lane.EndEntryId);

			for (int32 j = CurvePoints.Num() - 1; j >= 0; j--)
			{
				const FShapePoint& Point = CurvePoints[j];
				FShapePoint& NewPoint = LanePoints.Add_GetRef(Point);
				NewPoint.Position += Point.Right * LanePos;
			}
		}
		else
		{
			ensure(false);
		}
 
		/*
			这里用于简化形状：
		  - 检查中间点 Mid 是否足够接近由 Start 到 End 构成的线段
		  - 如果距离平方小于容差平方（ToleranceSqr），说明中间点是冗余的
		  - 移除中间点以简化曲线，减少点的数量
		
		  计算原理:
		  1. 将点投影到线段所在的直线上
		  2. 如果投影点在线段内，返回点到投影点的距离平方
		  3. 如果投影点在线段外，返回点到最近端点的距离平方
		*/
		const float LaneTessTolerance = BuildSettings.GetLaneTessellationTolerance(Lane.Tags);
		SimplifyShape(LanePoints, LaneTessTolerance);

		// 记录车道点的位置和up向量
		Lane.PointsBegin = OutZoneStorage.LanePoints.Num();
		for (const FShapePoint& Point : LanePoints)
		{
			OutZoneStorage.LanePoints.Add(Point.Position);
			OutZoneStorage.LaneUpVectors.Add(Point.Up);
		}
		Lane.PointsEnd = OutZoneStorage.LanePoints.Num();
		// 计算车道上点Forward向量，把车的行驶方向都存到LaneTangentVectors里，存的是世界空间下的行驶方向
		// Calculate per point forward.
		if (LaneDesc.Direction == EZoneLaneDirection::Forward)
		{
			OutZoneStorage.LaneTangentVectors.Add(StartForward);
		}
		else
		{
			OutZoneStorage.LaneTangentVectors.Add(-EndForward);
		}
		
		for (int32 PointIndex = Lane.PointsBegin + 1; PointIndex < Lane.PointsEnd - 1; PointIndex++)
		{
			const FVector WorldTangent = (OutZoneStorage.LanePoints[PointIndex + 1] - OutZoneStorage.LanePoints[PointIndex - 1]).GetSafeNormal();
			OutZoneStorage.LaneTangentVectors.Add(WorldTangent);
		}

		if (LaneDesc.Direction == EZoneLaneDirection::Forward)
		{
			OutZoneStorage.LaneTangentVectors.Add(EndForward);
		}
		else
		{
			OutZoneStorage.LaneTangentVectors.Add(-StartForward);
		}

		// 最后累加车道宽度
		CurWidth += LaneDesc.Width;
	}
	// 记录车道结束索引，车道只包含车能行驶的，不包括障碍物
	Zone.LanesEnd = OutZoneStorage.Lanes.Num();

	// 记录车道上点到车道开始点的距离
	// Calculate progression distance along lanes.
	OutZoneStorage.LanePointProgressions.AddZeroed(
		OutZoneStorage.LanePoints.Num() 
			- OutZoneStorage.LanePointProgressions.Num());
	for (int32 i = Zone.LanesBegin; i < Zone.LanesEnd; i++)
	{
		const FZoneLaneData& Lane = OutZoneStorage.Lanes[i];
		CalculateLaneProgression(OutZoneStorage.LanePoints, Lane.PointsBegin, Lane.PointsEnd, OutZoneStorage.LanePointProgressions);
	}

	// shapeComp构成的车道描述的boundingbox，就是边缘点的和
	// Calculate zone bounding box, all lanes are assumed to be inside the boundary
	Zone.Bounds.Init();
	for (int32 i = Zone.BoundaryPointsBegin; i < Zone.BoundaryPointsEnd; i++)
	{
		Zone.Bounds += OutZoneStorage.BoundaryPoints[i];
	}
}

```

# 2 FlattenSplineSegments
```cpp
static void FlattenSplineSegments(TConstArrayView<FZoneShapePoint> Points, bool bClosed, const FMatrix& LocalToWorld, const float Tolerance, TArray<FShapePoint>& OutPoints)
{
	// Tessellate points.
	const int32 NumPoints = Points.Num();
	int StartIdx = bClosed ? (NumPoints - 1) : 0;
	int Idx = bClosed ? 0 : 1;

	
	/*
		为什么只对开放曲线添加起点？
	  - 闭合曲线会在循环中处理所有点
	  - 开放曲线需要显式添加第一个点
	*/
	if (!bClosed)
	{
		const FZoneShapePoint& StartShapePoint = Points[StartIdx];
		if (StartShapePoint.Type != FZoneShapePointType::LaneProfile)
		{
			const FVector WorldPosition = LocalToWorld.TransformPosition(StartShapePoint.Position);
			const FVector WorldUp = LocalToWorld.TransformVector(StartShapePoint.Rotation.RotateVector(FVector::UpVector)).GetSafeNormal();
			OutPoints.Add(FShapePoint(WorldPosition, WorldUp));
		}
	}

	TArray<FVector> TempPoints;
	TArray<float> TempProgression;

	while (Idx < NumPoints)
	{
		// 定义局部变量
		FVector StartPosition(ForceInitToZero), 
			StartControlPoint(ForceInitToZero), 
			EndControlPoint(ForceInitToZero), 
			EndPosition(ForceInitToZero);
		// 生成白塞尔曲线的起始点，终止点和起始控制点，终止控制点
		GetCubicBezierPointsFromShapeSegment(Points[StartIdx], Points[Idx], LocalToWorld, StartPosition, StartControlPoint, EndControlPoint, EndPosition);

		// TODO: The Bezier tessellation does not take into account the roll when calculating tolerance.
		// Maybe we should have a templated version which would do the up axis interpolation too.

		TempPoints.Reset();
		if (Points[StartIdx].Type == FZoneShapePointType::LaneProfile)
		{
			TempPoints.Add(StartPosition);
		}
		UE::CubicBezier::Tessellate(TempPoints, StartPosition, StartControlPoint, EndControlPoint, EndPosition, Tolerance);

		TempProgression.SetNum(TempPoints.Num());

		// Interpolate up vector for points
		float TotalDist = FVector::Dist(StartPosition, TempPoints[0]);
		for (int32 i = 0; i < TempPoints.Num() - 1; i++)
		{
			TempProgression[i] = TotalDist;
			TotalDist += FVector::Dist(TempPoints[i], TempPoints[i + 1]);
		}
		TempProgression[TempProgression.Num() - 1] = TotalDist;

		// Add points and interpolate up axis
		const FQuat StartRotation = Points[StartIdx].Rotation.Quaternion();
		const FQuat EndRotation = Points[Idx].Rotation.Quaternion();
		for (int32 i = 0; i < TempPoints.Num(); i++)
		{
			const float Alpha = TempProgression[i] / TotalDist;
			FQuat Rotation = FMath::Lerp(StartRotation, EndRotation, Alpha);
			const FVector WorldUp = LocalToWorld.TransformVector(Rotation.RotateVector(FVector::UpVector)).GetSafeNormal();
			OutPoints.Add(FShapePoint(TempPoints[i], WorldUp));
		}

		StartPosition = EndPosition;
		StartIdx = Idx;
		Idx++;
	}
}
```