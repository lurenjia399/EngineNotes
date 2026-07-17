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
	// 计算每个轮子的悬挂约束，类似射线检测。根据车身位姿和悬挂配置,算出该轮子悬挂射线的世界空间  Start(悬挂顶端)和 End(悬挂完全伸展、轮子最低点)，用线段 vs 平面求交(纯数学,不碰物理)。如果这段悬挂射线穿过了车道平面,ImpactPoint 就是交点——即"轮子踩到地面的点",bBlockingHit = true。
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