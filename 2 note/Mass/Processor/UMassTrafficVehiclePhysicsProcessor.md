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
	// 穿透修正，简单来说就是遍历轮子，找到轮子陷在地下的距离（理想地面位置 - 轮子触碰地面位置）在Up向量上的投影就是陷在地下距离，再把这个距离加到车的实际位置上
	if (SolvePositionCorrect(VehicleWorldTransform, 
		RawLaneLocationTransform, SimplePhysicsVehicleFragment))
	{
		TransformFragment.SetTransform(VehicleWorldTransform);
	}
	// 悬挂射线检测，类似射线检测，从
	PerformSuspensionTraces(
		SimplePhysicsVehicleFragment,
		VehicleWorldTransform,
		RawLaneLocationTransform,
		SuspensionTraceHitResults,
		SuspensionTargets,
		bVisLog,
		UE::MassTraffic::EntityToColor(QueryContext.GetEntity(Index)));
}
```