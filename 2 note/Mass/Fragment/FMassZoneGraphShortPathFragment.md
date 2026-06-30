1 
2 
```cpp
bool FMassZoneGraphShortPathFragment::RequestPath(
	const FMassZoneGraphCachedLaneFragment& CachedLane, //CacheLane
	const FZoneGraphShortPathRequest& Request, // 生成ShortPath的请求
	const float InCurrentDistanceAlongLane, // 当前已经沿着lane走了多少距离
	const float AgentRadius)//entity的半径
{
	// 根据当前lane起始位置，找到cachelane中存储点的位置和切线，思路就是首先找到位置在哪两个点之间，然后计算出插值比例T，然后插值计算
	FVector StartLanePosition;
	FVector StartLaneTangent;
	CachedLane.GetPointAndTangentAtDistance(
		CurrentDistanceAlongLane, StartLanePosition, StartLaneTangent);
	// 
	const FVector StartDelta = StartPosition - StartLanePosition;
	const FVector StartLeftDir = FVector::CrossProduct(
		StartLaneTangent, FVector::UpVector);
	float StartLaneOffset = FloatCastChecked<float>
		(FVector::DotProduct(StartLeftDir, StartDelta), 
			UE::LWC::DefaultFloatPrecision);
	float StartLaneForwardOffset = FloatCastChecked<float>
		(FVector::DotProduct(StartLaneTangent, StartDelta) * TangentSign, 
			UE::LWC::DefaultFloatPrecision);
	// The point is off-lane if behind the start, or beyond the boundary.
	const bool bStartOffLane = 
		StartLaneForwardOffset < -OffLaneCapSlop
		|| StartLaneOffset < -(DeflatedLaneRight + OffLaneEdgeSlop)
		|| StartLaneOffset > (DeflatedLaneLeft + OffLaneEdgeSlop);
}
```