```cpp
if(当前周期的剩余时间 > 0)
{
	if(当前交叉路口不会通行行人，车辆)
	{
		直接设置周期剩余时间为负的
	}
	else if(有红绿灯)
	{
		if(当前交叉路口的当前周期上的车道和人行道都没有车没有人
			&& 没有车在等待使用当前交叉路口
			&& 当前交叉路口没有放行的人行道
			&& 当前交叉路口剩余时间 > 路口准备关闭时间)
		{
			这个周期剩余时间 = 路口准备关闭时间 - 帧长
		}
	}
	// 更新当前周期的红绿灯,设置红绿灯的状态
	UpdateTrafficLightsForCurrentPeriod();
	// 减少当前周期剩余时间
	IntersectionFragment.PeriodTimeRemaining = 
		IntersectionFragment.PeriodTimeRemaining - CountDownSpeedSeconds;
}
if(0 < 当前周期剩余时间 <= 路口准备关闭时间)//黄灯
{
	// 设置当前路口车道为准备关闭，人行道不管，也就是黄灯
	IntersectionFragment.ApplyLanesActionToCurrentPeriod(
		EMassTrafficPeriodLanesAction::SoftPrepareToClose,
		EMassTrafficPeriodLanesAction::None,
		&MassCrowdSubsystem, false);
	// 更新当前红绿灯状态
	IntersectionFragment.UpdateTrafficLightsForCurrentPeriod();
}
if(当前周期剩余时间 <= 0 && 老的当前周期剩余时间 > 0)//红灯
{
	// 设置当前路口车道为关闭，人行道关闭
	IntersectionFragment.ApplyLanesActionToCurrentPeriod(
		EMassTrafficPeriodLanesAction::SoftClose, 
		EMassTrafficPeriodLanesAction::HardClose,
		&MassCrowdSubsystem, false);
	// 更新红绿灯状态
	IntersectionFragment.UpdateTrafficLightsForCurrentPeriod();
	IntersectionFragment.PedestrianLightsShowStop();
	// 路口关闭了，就遍历下一个路口
	continue;
}
if(当前周期剩余时间 <= 0 && 老的当前周期剩余时间 <= 0)//应该是绿灯
{
	if(HighLod && 路口车道有车辆要通过)
	{
		1 设置
		continue;
	}
	// 推进到下一周期
	IntersectionFragment.AdvancePeriod();
	// 设置路口车道，人行道的状态
	IntersectionFragment.ApplyLanesActionToCurrentPeriod(
		VehicleLanesAction, PedestrianLanesAction,
		&MassCrowdSubsystem, false);
	// 更新红绿灯
	IntersectionFragment.UpdateTrafficLightsForCurrentPeriod();
	// 更新当前周期剩余时间，在前边刚推进到下一周期了
	IntersectionFragment.AddTimeRemainingToCurrentPeriod();
}
```