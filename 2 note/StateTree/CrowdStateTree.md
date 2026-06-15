# 1 Flee
1 条件设置
```cpp
bool FZoneGraphTagMaskCondition::TestCondition(
	FStateTreeExecutionContext& Context) const  
{  
    const FInstanceDataType& InstanceData = 
	    Context.GetInstanceData(*this);  
    return InstanceData.Left.
	    CompareMasks(InstanceData.Right, Operator) ^ bInvert;  
}
```

UMassZoneGraphAnnotationTrait


UMassStateTreeSchema

# 受击反应
```cpp
1 ACitySampleCrowdCharacter::TakeDamage 在character中收到伤害，会把自己添加到UHTMassComponentHitSubsystem这个subsystem的HitResults中
2 在subsystem的tick中，会遍历HitResults，通过SignalEntity发送HitReceived的信号
3 HitReceived也被UMassStateTreeProcessor这个订阅，SignalEntities这个方法就是订阅回调，执行statetree
4 statetree的执行会处理Evaluator的tick，在tick里会设置不同的标志位，在statetree中根据不同的标志位来进入到不同的受击状态（MoveHit还是TakeDamage）
```
1 MoveHit（在移动中收到攻击）
会执行两个StateTask，FHTCrowdCharacterMassStandTask
```cpp
{
	FMassMoveTargetFragment& MoveTarget = 
		Context.GetExternalData(MoveTargetHandle);
	MoveTarget.CreateNewAction(EMassMovementAction::Animate, *World);  
	UE::MassNavigation::ActivateActionAnimate(
		*World, Context.GetOwner(), 
		MassContext.GetEntity(), MoveTarget);
}
1 修改MassMoveTargetFragment这个，在UMassProcessor_Animation这个里面读取MassMoveTargetFragment进行处理
2 MassMoveTargetFragment这个Fragment

```
FMassLookAtTask，修改的FMassLookAtFragment这个片段，
