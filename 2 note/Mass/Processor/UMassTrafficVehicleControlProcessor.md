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
	1 如果没有NextLane，或者me
	*/
	const bool bMustStopAtLaneExit = UE::MassTraffic::ShouldStopAtLaneExit();

}
```