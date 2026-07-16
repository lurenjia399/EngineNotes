1 
```cpp
1 计算当前车道的速度限制
2 计算在当前车道Exit是否需要停下来。（如果没有NextLane了需要停下来。如果NextLane是交叉路口，交叉路口的下一条车道已经容纳不了当前Entity车了需要停下来。如果下一条车道要因为周期改变关闭需要停下来。）
3 重新计算停不下来状态。（如果当前车已经超过车道的Exit了，就停不下来了。如果Lod变成Off了，就去掉停不下来状态。如果速度本身就很小了，去掉停不下来状态）
4 计算目标速度。（如果当前跟车距离小于理想跟车距离，就减少速度。如果有障碍物，还得在调整速度。如果要在车道Exit停下来，还需要更新速度）
5 MoveVehicleToNextLane，如果车已经进入NextLane了
	1 移除当前车道的信息（占用空间和车道上车的数量）
	2 添加跟车信息，NextVehicleFragment中记录的
	3 添加目标车道的信息（占用空间和车道上车的数量）
	4 目标车道有GhostVehicle，将当前车添加到Ghost的数组中
```
2 SimpleVehicleControl
```cpp
void UMassTrafficVehicleControlProcessor::SimpleVehicleControl
(
	// 把参数全省略了
) const
{
	// 1 NoiseInput 是基于行驶距离的噪声输入(注意后面噪声随车走而推进,不是随时间)。这样同一辆车在同一段路上的噪声是稳定可复现的,不会抖。- 输出 LateralOffset:车相对车道中心线的横向微偏移,让车流看起来不是一条直线上的珠子。
	const float NoiseValue = UE::MassTraffic::CalculateNoiseValue(
		VehicleControlFragment.NoiseInput, MassTrafficSettings->NoisePeriod);
	LaneOffsetFragment.LateralOffset = GetLateralOffset(NoiseValue);
	// 2 计算最小的SpeedLimit，
	const float SpeedLimit = UE::MassTraffic::GetSpeedLimitAlongLane(
			LaneLocationFragment.LaneLength,//车所在车道的长度
			VehicleControlFragment.CurrentLaneConstData.SpeedLimit,// 当前车道的限速
			VehicleControlFragment.CurrentLaneConstData.AverageNextLanesSpeedLimit,
			LaneLocationFragment.DistanceAlongLane, //当前车道已经走了的距离
			VehicleControlFragment.Speed,//当前车的速度
			MassTrafficSettings->SpeedLimitBlendTime
	);
	const float VariedSpeedLimit =  FMath::Min(
		SpeedLimitOverride, 
		GetVariedSpeedLimit(SpeedLimit, 
			RandomFractionFragment, BusInfoFragment, 
			VehicleControlFragment, AppendixFragment, NoiseValue));
			
	// 3 计算车应不应该停在道路末端
	/*
	1 如果没有NextLane，或者没有Nextlane的Nextlane就应该停止
	2 如果NextLane是交叉路口，已经在交叉路口的车辆都能通过交叉路口，但自己在交叉路口的NextLane已经容纳不下了，就需要停止，如果CurrLane的剩余长度能容纳3倍半径，就可以变道
	3 如果下一条车道关闭了，但是Entity已经超过CurrLane的Exit了，就不能被停下来
	*/
	const bool bMustStopAtLaneExit = UE::MassTraffic::ShouldStopAtLaneExit();
	// 3.1 如果是刹不住车了，就不刹了
	if (!bIsOffLOD && bVehicleCantStopAtLaneExit) 
	{
		SetVehicleCantStopAtLaneExit(VehicleControlFragment, 
			LaneLocationFragment, NextVehicleFragment, EntityManager);
	}
	// 3.2 如果必须刹车，撤销刹不住
	if (bMustStopAtLaneExit && VehicleControlFragment.bCantStopAtLaneExit) 
	{
		UnsetVehicleCantStopAtLaneExit(VehicleControlFragment);
		bVehicleCantStopAtLaneExit = false;
	}
	// 3.3 如果进入了OffLOD的区域，撤销刹不住
	if (bIsOffLOD && VehicleControlFragment.bCantStopAtLaneExit) 
	{
		UnsetVehicleCantStopAtLaneExit(VehicleControlFragment);
		bVehicleCantStopAtLaneExit = false;
	}
	// 3.4 如果车速本身就快停了，撤销刹不住
	if (VehicleControlFragment.Speed < 0.1f 
		&& !bIsFrontOfVehicleBeyondEndOfLane 
		&& VehicleControlFragment.bCantStopAtLaneExit)
	{
		UnsetVehicleCantStopAtLaneExit(VehicleControlFragment);
		bVehicleCantStopAtLaneExit = false;		
	}
	// 4 计算车的TargetSpeed
	/*
	1 如果实际的跟车距离小于理想的跟车距离，距离越近速度应该越小
	2 如果距离障碍物碰撞时间小于理想碰撞时间，距离越近速度应该越小
	3 如果必须在LaneExit停下，如果剩余的长度小于理想的停止位置，差距越大与应该减速
	4 最后限制不允许速度为负值
	5 通过加速度逼近TargetSpeed，不直接设置防止突变
	*/
	const float TargetSpeed = UE::MassTraffic::CalculateTargetSpeed();
	// 5 设置刹车灯，如果刹车灯持续时间>0就亮灯
	if (VehicleControlFragment.BrakeLightHysteresis > SMALL_NUMBER)
	{
		VehicleLightsFragment.bBrakeLights = true;
	}
	else
	{
		VehicleLightsFragment.bBrakeLights = false;
		VehicleControlFragment.BrakeLightHysteresis = 0.0f;
	}
	// 6 把这一帧移动的距离积起来
	const float MaxDistanceDelta = FMath::Max(
		AvoidanceFragment.DistanceToNext - 
		MassTrafficSettings->MinimumDistanceToObstacleRange.X, 0.0f); 
	const float DistanceDelta = FMath::Min(
		VariableTickFragment.DeltaTime * 
		VehicleControlFragment.Speed, MaxDistanceDelta);
	LaneLocationFragment.DistanceAlongLane += DistanceDelta;
	VehicleControlFragment.NoiseInput += DistanceDelta;
	// 7 如果停的超过了LaneExit，需要重新设置DistanceAlongLane
	if (bIsVehicleStoppingOverLaneExit)
	{
		//就是和人行道相交了，需要设置
		//	bIsStoppedVehicleInPreviousLaneOverlappingThisLane
	}
	// 8 如果移动超过了CurrLane，如果有下一条车道就变道，如果没有就停下来等
	else if (LaneLocationFragment.DistanceAlongLane 
		>= LaneLocationFragment.LaneLength)
	{
		
		if (VehicleControlFragment.NextLane)
		{
			const FMassEntityHandle VehicleEntity = Context.GetEntity(EntityIndex); 
			bool bIsVehicleStuck = false;
			UE::MassTraffic::MoveVehicleToNextLane(
				EntityManager,
				MassTrafficSubsystem,
				VehicleEntity,
				AgentRadiusFragment,
				RandomFractionFragment,
				VehicleControlFragment,
				VehicleLightsFragment,
				LaneLocationFragment,
				Context.GetMutableFragmentView<
					FMassTrafficNextVehicleFragment>()[EntityIndex],
				LaneChangeFragment,
				bIsVehicleStuck);
		}
		else
		{
			LaneLocationFragment.DistanceAlongLane = LaneLocationFragment.LaneLength;
		}
	}
}
```
2 MoveVehicleToNextLane
```cpp
void MoveVehicleToNextLane(/*省略了参数*/)
{
	// 1 如果是当前车道的末尾车，就移除末尾车记录并清空车道数据（车道上的车辆，车道上空余长度），如果不是末尾车，根据车的长度更新车道数据
	if (CurrentLane.TailVehicle == VehicleEntity)
	{
		CurrentLane.TailVehicle.Reset();
		CurrentLane.ClearVehicleOccupancy();
	}
	else
	{
		CurrentLane.RemoveVehicleOccupancy(SpaceTakenByVehicleOnLane);
	}
	// 2 更细车辆的位置，减去车道长度
	LaneLocationFragment.DistanceAlongLane -= LaneLocationFragment.LaneLength;
	// 3 改变新车道的信息，重置准备使用标志位，如果是停不住车就改变计数，记录前一条车道信息，更新ZoneGraphLocationFragment
	FZoneGraphTrafficLaneData& NewCurrentLane = *VehicleControlFragment.NextLane;
	NewCurrentLane.bIsVehicleReadyToUseLane = false;
	if (VehicleControlFragment.bCantStopAtLaneExit)
	{
		--NewCurrentLane.NumReservedVehiclesOnLane;
		VehicleControlFragment.bCantStopAtLaneExit = false;
	}
	VehicleControlFragment.PreviousLaneIndex = 
		LaneLocationFragment.LaneHandle.Index;
	VehicleControlFragment.PreviousLaneLength = LaneLocationFragment.LaneLength;
	LaneLocationFragment.LaneHandle = NewCurrentLane.LaneHandle;
	LaneLocationFragment.LaneLength = NewCurrentLane.Length;
	VehicleControlFragment.CurrentLaneConstData = NewCurrentLane.ConstData;
	--NewCurrentLane.NumVehiclesApproachingLane;
	// 4 重置NextLane，如果只有一条就直接设，如果超过一条了就等着ChooseNextLaneProcessor设置
	if (NewCurrentLane.NextLanes.Num() == 1)
	{
		VehicleControlFragment.NextLane = NewCurrentLane.NextLanes[0];
		++VehicleControlFragment.NextLane->NumVehiclesApproachingLane;
	}
	else
	{
		VehicleControlFragment.NextLane = nullptr;
	}
	// 5 更新转向灯
	VehicleLightsFragment.bLeftTurnSignalLights = NewCurrentLane.bTurnsLeft;
	VehicleLightsFragment.bRightTurnSignalLights = NewCurrentLane.bTurnsRight;
	// 6 设置NextVehicle,就是当前车的前一辆车
	if (NewCurrentLane.TailVehicle.IsSet())
	{
		NextVehicleFragment.SetNextVehicle(
			VehicleEntity, NewCurrentLane.TailVehicle);
	}
	// 7 向新车道记录车的信息，增加车的数量，减少空余长度
	NewCurrentLane.AddVehicleOccupancy(SpaceTakenByVehicleOnLane);
	// 8 在新车道了，重置停不下来的标志位
	VehicleControlFragment.bCantStopAtLaneExit = false;
	// 9 记录当前是新车道最后一辆车
	NewCurrentLane.TailVehicle = VehicleEntity;
	// 10 结束变道
	if (LaneChangeFragment)
	{
		LaneChangeFragment->EndLaneChangeProgression(VehicleLightsFragment, NextVehicleFragment, EntityManager);
		LaneChangeFragment->bBlockAllLaneChangesUntilNextLane = false;
	}
	// 11 检查目标车道是否有幽灵尾车：1 有一辆车正在变道并入这条车道，且它抢先占了"队尾"的逻辑位置。 2. 建立避碰关系：既然当前车辆（VehicleEntity）马上要成为这条车道的真正尾车（排在最后面），那它前面实际上还挤着这辆正在变道的"幽灵车"。为了不让当前车辆因为看不到这辆变道车而加速追尾，需要把这辆变道车设为当前车辆的"前车"（Next）。3. 但不是直接设置，而是委托：因为变道过程本身会动态更新它自己的"跟随者列表"（可能中途取消变道、完成变道等），所以不能直接写死VehicleEntity 的 NextVehicleFragment，而是调用变道车自己的LaneChangeFragment->AddOtherLaneChangeNextVehicle_ForVehicleBehind把 VehicleEntity登记为"变道车后面新出现的跟随者"。这样变道车的 LaneChangeFragment 就接管了这个关系的生命周期——等它变道完成或取消时，会负责去清理/更新 VehicleEntity 的 NextVehicleFragment。
	if (NewCurrentLane.GhostTailVehicle_FromLaneChangingVehicle.IsSet())
	{
		FMassTrafficVehicleLaneChangeFragment* LaneChangeFragment_GhostTailEntity =
			EntityManager.GetFragmentDataPtr<FMassTrafficVehicleLaneChangeFragment>
			(NewCurrentLane.GhostTailVehicle_FromLaneChangingVehicle);
		if (LaneChangeFragment_GhostTailEntity && LaneChangeFragment_GhostTailEntity->IsLaneChangeInProgress())
		{
			LaneChangeFragment_GhostTailEntity->AddOtherLaneChangeNextVehicle_ForVehicleBehind(VehicleEntity, EntityManager);
		}
		NewCurrentLane.GhostTailVehicle_FromLaneChangingVehicle = FMassEntityHandle();
	}
	// 11 处理由Split和Merge的Ghost车
	{
		1 如果新车道上有GhostVehicle_FromSplit，就把GhostVehicle转移到NextVehicleFragment中，并清掉新车道上的GhostVehicle。MergeVehicle同理
		2 将自己作为Ghost登记到新车道的Splitlane上
		3 如果旧车道上还记录着自己，就移除掉
	}
}
```
3 PIDVehicleControl
```cpp
void UMassTrafficVehicleControlProcessor::PIDVehicleControl()
{
	// 速度控制前瞻的最小距离(厘米)。即使车速为0,也至少往前看这么远。 
	float SpeedControlMinLookAheadDistance = 
		MassTrafficSettings->SpeedControlMinLookAheadDistance;
	// 速度控制前瞻的时间(秒)。乘以当前车速得到"按时间算的前瞻距离"。
	float SpeedControlLaneLookAheadTime = 
		MassTrafficSettings->SpeedControlLaneLookAheadTime;
	// 转向前瞻的最小距离(厘米）。
	float SteeringControlMinLookAheadDistance = 
		MassTrafficSettings->SteeringControlMinLookAheadDistance;
	// 转向前瞻的时间(秒),同样乘以车速。
	float SteeringControlLaneLookAheadTime = 
		MassTrafficSettings->SteeringControlLaneLookAheadTime;
	// 转弯时的速度缩放系数。弯道曲率越大,用这个  系数把目标速度按比例压低,让车过弯时减速,避免高速过弯"飘出去"。
	float TurnSpeedScale = MassTrafficSettings->TurnSpeedScale;
	// 计算速度控制的LookAhead距离，速度 * LookAheadTime
	const float SpeedControlLookAheadDistance = 
		FMath::Max(SpeedControlMinLookAheadDistance, 
		SpeedControlLaneLookAheadTime * VehicleControlFragment.Speed);
	// 计算速度控制的前瞻点，就是DistanceAlongLane + LookAheadDistance这个值，只不过通过3次贝塞尔曲线插值出来一个结果。
	UE::MassTraffic::InterpolatePositionAndOrientationAlongContinuousLanes(...);
	// 相同的方式，计算转向控制的前瞻点
	UE::MassTraffic::InterpolatePositionAndOrientationAlongContinuousLanes(...);
	// 如果当前在变道，就需要根据前瞻点来计算出现在应该设置的位置
	if (LaneChangeFragment && LaneChangeFragment->IsLaneChangeInProgress())
	{
		FTransform SteeringControlChaseTargetTransform
			(SteeringControlChaseTargetOrientation, 
			SteeringControlChaseTargetLocation);
		UE::MassTraffic::AdjustVehicleTransformDuringLaneChange(
			*LaneChangeFragment, LaneLocationFragment.DistanceAlongLane 
			+ SteeringControlLookAheadDistance, 
			  SteeringControlChaseTargetTransform, 
			  EntityManager.GetWorld());
		SteeringControlChaseTargetLocation = 
			SteeringControlChaseTargetTransform.GetLocation();
		SteeringControlChaseTargetOrientation = 
			SteeringControlChaseTargetTransform.GetRotation();
	}
}
```