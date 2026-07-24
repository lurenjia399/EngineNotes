# 0 
```cpp
0 在UE::Mass::ProcessorGroupNames::Behavior组，会更新statetree，在statetree中会创建新的Movetarget
1 在UE::Mass::ProcessorGroupNames::Tasks组，会通过UMassNavMeshPathFollowProcessor这个来更新MoveTarget中的数据，是根据CacheLane，Shortpath等信息。还会有UMassSteerToMoveTargetProcessor这个来通过转向速度和当前速度计算出力，这个是每一帧开始的初始力用来驱动entity向
2 紧接着在UE::Mass::ProcessorGroupNames::Avoidance组，会通过MovingAvoidance，StandAvoidance等避障Processor，来给FMassForceFragment中添加分离力
3 然后在UE::Mass::ProcessorGroupNames::ApplyForces组，通过UMassApplyForceProcessor来将这一帧产生的MassForceFragment力，积分到FMassDesiredMovementFragment速度中，就是力乘时间
4 然后在UE::Mass::ProcessorGroupNames::Movement组，通过UMassApplyMovementProcessor来积分速度，得到位移设置到FTransformFragment中
5 然后在UE::Mass::ProcessorGroupNames::UpdateWorldFromMass组，通过UHTMassTransformToActorTranslator来根据entity的transform来更新Actor的信息
```
# 1 UMassApplyForceProcessor
```cpp
//在初始化、卡顿、断点后恢复等场景,单帧 DeltaTime 可能突然很大;v += F*dt 里 dt 一大,速度就会瞬间被算出一个爆炸值,agent 直接"弹飞"。上限 0.1s(相当于 10fps)是个安全阀。
const float DeltaTime = FMath::Min(0.1f, Context.GetDeltaTimeSeconds());

FMassForceFragment& Force = ForceList[EntityIt];
FMassDesiredMovementFragment& DesiredMovement = MovementList[EntityIt];
// 通过力积分出DesiredVelocity
DesiredMovement.DesiredVelocity += Force.Value * DeltaTime;
// 清空力
Force.Value = FVector::ZeroVector;
```

# 2 UMassApplyMovementProcessor
```cpp
// 获取entity的Desired速度
Velocity.Value = MovementList[EntityIt].DesiredVelocity;
// 积分entity当前位置
CurrentLocation += Velocity.Value * DeltaTime;
CurrentTransform.SetTranslation(CurrentLocation);
```

# 3 UMassNavigationSmoothHeightProcessor
```cpp
// 单独处理entity的高度平滑，负责 entity 高度(Z轴)的平滑跟随:因为 Mass 的避障/运动是纯 2D(XY)的,Z被单独用一个指数平滑滤波器逐帧逼近移动目标的高度,从而让 agent在坡道、台阶、地形高度变化处平顺地上下起伏,而不是高度瞬间跳变。
if (MoveTarget.GetCurrentAction() == EMassMovementAction::Move 
	|| MoveTarget.GetCurrentAction() == EMassMovementAction::Stand)
{
	FVector CurrentLocation = CurrentTransform.GetLocation();
	FMath::ExponentialSmoothingApprox(CurrentLocation.Z, MoveTarget.Center.Z, DeltaTime, MovementParams.HeightSmoothingTime);
	CurrentTransform.SetLocation(CurrentLocation);
}
```
