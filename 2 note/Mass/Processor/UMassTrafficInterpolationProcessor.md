1 
2 InterpolatePositionAndOrientationAlongLane
```cpp
void InterpolatePositionAndOrientationAlongLane()
{
	// 1 初始化InOutLaneSegment，表示当前车的位置在哪两个LanePoint之间
	if (!IsValidLaneSegmentForDistanceAlongLane(InOutLaneSegment, 
		ZoneGraphStorage, LaneIndex, DistanceAlongLane))
	{
		InitLaneSegment(ZoneGraphStorage, LaneIndex, 
			DistanceAlongLane, InOutLaneSegment);
	}
	// 2 计算当前车在片段中的插值系数
	const float Alpha = FMath::GetRangePct(
		InOutLaneSegment.StartProgression, 
		InOutLaneSegment.EndProgression, 
		DistanceAlongLane);
	// 3 根据类型，插值计算出当前车位置，
	switch (InterpolationMethod)
	{
		// 3.1 如果是线性的，位置就是根据插值系数直接插值位置，Forward就是方向
		case ETrafficVehicleMovementInterpolationMethod::Linear:
			InterpolatedLocation = FMath::Lerp(
				InOutLaneSegment.StartPoint, InOutLaneSegment.EndPoint, Alpha);
			InterpolatedForwardVector = InOutLaneSegment.EndPoint 
				- InOutLaneSegment.StartPoint;
			break;
		// 3.2 ru
		case ETrafficVehicleMovementInterpolationMethod::CubicBezier:

			InterpolatedLocation = UE::CubicBezier::Eval(InOutLaneSegment.StartPoint, InOutLaneSegment.StartControlPoint, InOutLaneSegment.EndControlPoint, InOutLaneSegment.EndPoint, Alpha);
			InterpolatedForwardVector = UE::CubicBezier::EvalDerivate(InOutLaneSegment.StartPoint, InOutLaneSegment.StartControlPoint, InOutLaneSegment.EndControlPoint, InOutLaneSegment.EndPoint, Alpha);
			
			break;
	}
}
```