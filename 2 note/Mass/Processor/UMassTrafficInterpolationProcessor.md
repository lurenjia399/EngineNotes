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
		// 3.2 如果是3次贝塞尔曲线，位置和方向都由控制点插值过来
		case ETrafficVehicleMovementInterpolationMethod::CubicBezier:
			InterpolatedLocation = UE::CubicBezier::Eval(
				InOutLaneSegment.StartPoint, InOutLaneSegment.StartControlPoint,
				InOutLaneSegment.EndControlPoint, 
				InOutLaneSegment.EndPoint, Alpha);
			InterpolatedForwardVector = UE::CubicBezier::EvalDerivate(
				InOutLaneSegment.StartPoint, InOutLaneSegment.StartControlPoint,
				InOutLaneSegment.EndControlPoint, 
				InOutLaneSegment.EndPoint, Alpha);
			break;
	}
	// 4 插值处Up向量，根据插值出的Forward方向和Up向量合成矩阵，得出矩阵的四元数
	const FVector InterpolatedUpVector = FMath::Lerp(
		InOutLaneSegment.LaneSegmentStartUp, 
		InOutLaneSegment.LaneSegmentEndUp, 
		Alpha); 
	const FQuat InterpolatedOrientation = FRotationMatrix::MakeFromXZ(
		InterpolatedForwardVector, 
		InterpolatedUpVector).ToQuat();
	// 传出去
	OutPosition = InterpolatedLocation;
	OutOrientation = InterpolatedOrientation;
}
```
3 void AdjustVehicleTransformDuringLaneChange
```cpp
void AdjustVehicleTransformDuringLaneChange()
{
	// 如果不在变道中，不需要执行
	if (!LaneChangeFragment.IsLaneChangeInProgress())
	{
		return;
	}
	// 计算出剩余变道比例LaneChangeProgressionScale，也就是变道进度[-1,1]，向右变是负数左变是正数
	const float LaneChangeProgressionScale = LaneChangeFragment.
		GetLaneChangeProgressionScale(DistanceAlongLane);
	// 取变变道进度绝对值[0,1]
	const float Alpha_Linear = FMath::Abs(LaneChangeProgressionScale);
	// 计算出向左还是向右变道，向左是1，向右是-1
	const float Sign = (LaneChangeProgressionScale >= 0.0f ? 1.0f : -1.0f);
	// 将比例带入 3x^2 - 2x^3 函数，计算出变道进度
	const float Alpha_Cubic = SimpleNormalizedCubicSpline(Alpha_Linear);
	// 将比例带入 6x - 6x^2 函数，上边的导数，计算出变道进度
	const float Alpha_CubicDerivative = SimpleNormalizedCubicSplineDerivative(
		Alpha_Linear);
	// 根据变道进度计算沿y轴偏移
	const FVector OffsetVector = 
		(Sign * LaneChangeFragment.DistanceBetweenLanes_Begin * Alpha_Cubic) 
		* Transform.GetUnitAxis(EAxis::Y);
	
	FQuat LocalRotationToApply;
	{
		const float InitialYaw = LaneChangeFragment.Yaw_Initial;
		float FinalYaw = Transform.GetRotation().Euler().Z;
		if (InitialYaw - FinalYaw < -180.0f)
		{
			FinalYaw -= 360.0f;
		}
		if (InitialYaw - FinalYaw > 180.0f)
		{
			FinalYaw += 360.0f;
		}
		const float DeltaLaneChangeDistance = LaneChangeFragment.DistanceAlongLane_Final_End - LaneChangeFragment.DistanceAlongLane_Final_Begin;
		const float MaxYawDelta = FMath::RadiansToDegrees(FMath::Atan2(LaneChangeFragment.DistanceBetweenLanes_Begin, DeltaLaneChangeDistance));

		const float Yaw = FMath::Lerp(0.0f, InitialYaw - FinalYaw, Alpha_Cubic)  +  (-Sign * Alpha_CubicDerivative * MaxYawDelta);

		LocalRotationToApply = FQuat::MakeFromEuler(FVector(0.0f, 0.0f, Yaw));
	}
}
```