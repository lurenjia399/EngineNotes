1 SimpleVehicleControl
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
	// 7 如果ting
	if (bIsVehicleStoppingOverLaneExit)
	{
		const float MaxDistanceAlongLaneIfStopped = LaneLocationFragment.LaneLength - AgentRadiusFragment.Radius; 

		if (bIsOffLOD ||
			(bIsLowLOD && (LaneLocationFragment.DistanceAlongLane - MaxDistanceAlongLaneIfStopped <= 10.0f))) 
		{
			// (See all CROSSWALKOVERLAP.)
			LaneLocationFragment.DistanceAlongLane = MaxDistanceAlongLaneIfStopped - 1.0f/*cm*/;
		}
		else
		{
			// (See all CROSSWALKOVERLAP.)
			if (VehicleControlFragment.NextLane)
			{
				VehicleControlFragment.NextLane->bIsStoppedVehicleInPreviousLaneOverlappingThisLane = true;			
			}
			if (LaneLocationFragment.DistanceAlongLane >= LaneLocationFragment.LaneLength)
			{
				LaneLocationFragment.DistanceAlongLane = LaneLocationFragment.LaneLength;
			}
		}
	}
}
```