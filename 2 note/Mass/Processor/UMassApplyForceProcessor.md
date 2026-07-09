```cpp
FMassForceFragment& Force = ForceList[EntityIt];
FMassDesiredMovementFragment& DesiredMovement = MovementList[EntityIt];
// 通过力积分出DesiredVelocity
DesiredMovement.DesiredVelocity += Force.Value * DeltaTime;
// 清空力
Force.Value = FVector::ZeroVector;
```