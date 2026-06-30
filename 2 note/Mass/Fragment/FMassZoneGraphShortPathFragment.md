1 
2 
```cpp
bool FMassZoneGraphShortPathFragment::RequestPath(
	const FMassZoneGraphCachedLaneFragment& CachedLane, //CacheLane
	const FZoneGraphShortPathRequest& Request, // 生成ShortPath的请求
	const float InCurrentDistanceAlongLane, // 当前已经沿着lane走了多少距离
	const float AgentRadius)//entity的半径
{
	// 根据当前lane起始距离，找到cachelane中存储点的位置和切线，思路就是首先找到位置在哪两个点之间，然后计算出插值比例T，然后插值计算
	FVector StartLanePosition;
	FVector StartLaneTangent;
	CachedLane.GetPointAndTangentAtDistance(
		CurrentDistanceAlongLane, StartLanePosition, StartLaneTangent);
	// 做出lane上起始位置到请求起始位置的向量，将其投影到车道的左向量上，判断投影长度是否在lane的范围里面，如果不在范围里，bStartOffLane这个就是true
	const FVector StartDelta = StartPosition - StartLanePosition;
	const FVector StartLeftDir = FVector::CrossProduct(
		StartLaneTangent, FVector::UpVector);
	float StartLaneOffset = FloatCastChecked<float>
		(FVector::DotProduct(StartLeftDir, StartDelta), 
			UE::LWC::DefaultFloatPrecision);
	float StartLaneForwardOffset = FloatCastChecked<float>
		(FVector::DotProduct(StartLaneTangent, StartDelta) * TangentSign, 
			UE::LWC::DefaultFloatPrecision);
	const bool bStartOffLane = 
		StartLaneForwardOffset < -OffLaneCapSlop
		|| StartLaneOffset < -(DeflatedLaneRight + OffLaneEdgeSlop)
		|| StartLaneOffset > (DeflatedLaneLeft + OffLaneEdgeSlop);
	// 如果请求点在lane外部，因为不往回倒退，所以如果请求点在lane开始点的前边，这是需要将开始点往前偏移一段投影距离
	if (bStartOffLane)
	{
		const float StartForwardOffset = FMath::Clamp(
				Request.AnticipationDistance.Get() + StartLaneForwardOffset, 
				0.0f, Request.AnticipationDistance.Get());
		StartDistanceAlongPath += StartForwardOffset * TangentSign;
	}
	// 如果请求点在lane外部，需要往Points点集里添加一个点
	if (bStartOffLane)
	{
		FMassZoneGraphPathPoint& StartPoint = Points[NumPoints++];
		StartPoint.DistanceAlongLane = FMassInt16Real10(CurrentDistanceAlongLane);
		StartPoint.Position = Request.StartPosition;
		StartPoint.Tangent = FMassSnorm8Vector2D(StartLaneTangent * TangentSign);
		StartPoint.bOffLane = true;
		StartPoint.bIsLaneExtrema = false;
		CachedLane.GetPointAndTangentAtDistance(
			StartDistanceAlongPath, StartLanePosition, StartLaneTangent);
		const FVector LeftDir = FVector::CrossProduct(
			StartLaneTangent, FVector::UpVector);
		StartPosition = StartLanePosition + LeftDir * StartLaneOffset;
		const FVector DirToClampedPoint = StartPosition - StartPoint.Position;
		StartPoint.Tangent = MassSnorm8Vector2D(DirToClampedPoint.GetSafeNormal());
	}
	// 如果请求点不在lane外部，也需要点集里添加一个点
	if (!bStartOffLane || bHasMidPath)
	{
		FMassZoneGraphPathPoint& Point = Points[NumPoints++];
		Point.DistanceAlongLane = FMassInt16Real10(StartDistanceAlongPath);
		Point.Position = StartPosition;
		Point.Tangent = FMassSnorm8Vector2D(StartLaneTangent * TangentSign);
		Point.bOffLane = false;
		Point.bIsLaneExtrema = false;
	}
}
```