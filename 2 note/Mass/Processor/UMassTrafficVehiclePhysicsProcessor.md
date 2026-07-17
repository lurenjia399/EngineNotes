1 
2 
```cpp
void UMassTrafficVehiclePhysicsProcessor::Execute(
	FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	// 根据插值计算出当前车所在的位置
	FTransform RawLaneLocationTransform;
	UE::MassTraffic::InterpolatePositionAndOrientationAlongLane(
		*ZoneGraphStorage, LaneLocationFragment.LaneHandle.Index, 
		LaneLocationFragment.DistanceAlongLane, 
		ETrafficVehicleMovementInterpolationMethod::CubicBezier, 
		InterpolationFragment.LaneLocationLaneSegment, RawLaneLocationTransform);
	// 在位置上添加横向偏移
	RawLaneLocationTransform.AddToTranslation(
		RawLaneLocationTransform.GetRotation().GetRightVector() * 
		LaneOffsetFragment.LateralOffset);
	// 穿透修正，简单来说就是遍历轮子，找到轮子陷在地下的距离（理想地面位置 - 轮子触碰地面wei'zh）
	if (SolvePositionCorrect(VehicleWorldTransform, 
		RawLaneLocationTransform, SimplePhysicsVehicleFragment))
	{
		TransformFragment.SetTransform(VehicleWorldTransform);
	}
}
```