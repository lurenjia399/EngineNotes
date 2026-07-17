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
	// 求解车的线速度，角速度等各种信息
	SimulateDriveForces(
		DeltaTime,
		GravityZ,
		PIDVehicleControlFragment,
		SimplePhysicsVehicleFragment,
		VelocityFragment,
		AngularVelocityFragment,
		TransformFragment,
		VehicleWorldTransform,
		RawLaneLocationTransform,
		SuspensionTraceHitResults,
		bVisLog);
	// 再次穿透修正
	SolvePositionCorrect(TransformFragment.GetMutableTransform(),
		RawLaneLocationTransform, SimplePhysicsVehicleFragment);
	// 悬挂约束，这是一次 PBD悬挂约束迭代——对每个触地轮,用弹簧-阻尼公式算出把挂点拉向贴地目标所需的位置/旋转修正,直接微调车身的 Transform,多次迭代后收敛出真实的车身高度与俯仰/侧倾姿态。
	SolveSuspensionConstraintsIteration(DeltaTime, SimplePhysicsVehicleFragment, VelocityFragment, AngularVelocityFragment, TransformFragment, VehicleWorldTransform, RawLaneLocationTransform, VehicleVelocity, SuspensionTargets, bVisLog);
	// 把车的位置转到车道本地坐标系,对横向 Y 和垂直 Z做软阈值渐进钳制(近处放任、远处强拉、绝不越界),既保留自然晃动又保证车不脱离车道;
	ClampLateralDeviation(TransformFragment, VehicleWorldTransform, RawLaneLocationTransform);
	// 最后更新车的速度
	UpdateCoMVelocity(DeltaTime, SimplePhysicsVehicleFragment, TransformFragment, VelocityFragment, AngularVelocityFragment, VehicleWorldTransform);

}
```