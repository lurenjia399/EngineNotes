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
	1 计算TargetSide朝左的向量，是Forward叉乘Up向量
	2 计算Delta，是MoveTarget中Center就是lane上的位置，和实际entity的位置差值
	*/
	const FVector TargetSide = FVector::CrossProduct(MoveTarget.Forward, FVector::UpVector);
	const FVector Delta = CurrentLocation - MoveTarget.Center;
}
```
