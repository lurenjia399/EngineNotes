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