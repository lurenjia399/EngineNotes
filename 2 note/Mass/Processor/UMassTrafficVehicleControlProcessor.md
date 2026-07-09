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
	// 4 如果是刹不住车了，就不刹了
	if (!bIsOffLOD && bVehicleCantStopAtLaneExit) 
	{
		SetVehicleCantStopAtLaneExit(VehicleControlFragment, 
			LaneLocationFragment, NextVehicleFragment, EntityManager);
	}
	// 5 如果必须刹车，撤销刹不住
	if (bMustStopAtLaneExit && VehicleControlFragment.bCantStopAtLaneExit) 
	{
		UnsetVehicleCantStopAtLaneExit(VehicleControlFragment);
		bVehicleCantStopAtLaneExit = false;
	}
}
```