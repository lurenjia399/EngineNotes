1 
2 FMassSteeringFragment 读写这个，主要就是计算SteerFragment中的DesiredVelocity，通过SteerDirection * MoveTargetFragment中的DesiredSpeed，其中浔的时停，就是把这个值变得很小
3 FMassForceFragment 读写这个，施加转向力，通过Force.Value = SteerK * (Steering.DesiredVelocity - DesiredMovement.DesiredVelocity)这个公式计算，用当前的期望速度和之前期望速度做差，通过期望做差算出转向力，不予实际速度挂钩，防止力 → 改变速度 → 影响下一帧的力计算 → 再次改变速度
4 如果MoveTarget是Move，

2 EMassMovementAction::Move，对Move类型的处理
```cpp
// entity展望未来距离，时间*MoveTarget中速度
const FVector::FReal LookAheadDistance = FMath::Max(
	1.0f, MovingSteeringParams.LookAheadTime * MoveTarget.DesiredSpeed.Get());
if (MoveTarget.GetCurrentAction() == EMassMovementAction::Move)
{
	/*
	1 计算出展望未来距离，如果接近stand目标的就乘上系数，越接近移动目标就展望越小的距离
	*/
	FVector::FReal ArrivalFade = 1.;
	if (MoveTarget.IntentAtGoal == EMassMovementAction::Stand)
	{
		ArrivalFade = FMath::Clamp(MoveTarget.DistanceToGoal / LookAheadDistance, 0., 1.);
	}
	const FVector::FReal SteeringPredictionDistance = LookAheadDistance * ArrivalFade;
	/*
	1 计算TargetSide朝左的向量，是MoveTarget.Forward叉乘Up向量，lane上的朝左向量
	2 计算Delta，是MoveTarget.Center就是lane上的位置，和实际entity的位置差值，是lane上位置指向entity实际位置的向量
	3 计算ForwardOffset，是entity位置在Forward方向上的投影
	4 计算SidewaysOffset，是entity位置在左向量方向上的投影
	5 计算SteerForward，是根据勾股定理，计算过entity位置与圆交点在Forward方向上的投影
	6 计算SteerTarget，计算转向的目标点，因为entity有Delta就是偏离了lane，需要在lane上找一个点让entity转向回来，回到lane上
	*/
	const FVector TargetSide = FVector::CrossProduct(MoveTarget.Forward, FVector::UpVector);
	const FVector Delta = CurrentLocation - MoveTarget.Center;
	const FVector::FReal ForwardOffset = FVector::DotProduct(MoveTarget.Forward, Delta);
	const FVector::FReal SidewaysOffset = FVector::DotProduct(TargetSide, Delta);
	const FVector::FReal SteerForward = FMath::Sqrt(FMath::Max(0., FMath::Square(SteeringPredictionDistance) - FMath::Square(SidewaysOffset)));
	FVector SteerTarget = MoveTarget.Center + MoveTarget.Forward * FMath::Clamp(ForwardOffset + SteerForward, 0., SteeringPredictionDistance);
	/*
	1 计算转向方向，就是entity当前位置指向转向目标点的向量
	2 计算转向速度，就是转向方向 * 转向速度标量
	3 计算出转向力，通过转向速度和当前移动速度的插值计算出
	*/
	FVector SteerDirection = SteerTarget - CurrentLocation;
	Steering.DesiredVelocity = SteerDirection * DesiredSpeed * Context.GetEntityManagerChecked().GetGlobalCustomTimeDilation();
	Force.Value = SteerK * (Steering.DesiredVelocity - DesiredMovement.DesiredVelocity);
}
```
