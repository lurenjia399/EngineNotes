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
1 修改MassMoveTargetFragment这个，在UMassProcessor_Animation这个里面读取MassMoveTargetFragment进行处理把Fragment数据设置到动画蓝图里,UMassSteerToMoveTargetProcessor这个也会读取处理来改变entity运动状态
2 MassMoveTargetFragment这个Fragment是由UMassSteeringTrait这个Trait添加的
3 UMassProcessor_Animation 这个里会将MoveTargetFragment里的数据赋值给UMassCrowdAnimInstance的MassMovementInfo变量，
4 在 UMassProcessor_Animation 之后 才会执行UMassSteerToMoveTargetProcessor

```
FMassLookAtTask，修改的FMassLookAtFragment这个片段，
```cpp
1 这个Task里主要设置FMassLookAtFragment这个Fragment
2 玩家撞击到行人，就会通过玩家身上的massagentComponent的获取entity，然后设置这个Fragment，让其朝向玩家。玩家身上的massagentComp是配置的BP_MassExperience中在controller上创建出来的
3 核心处理是通过UMassLookAtProcessor这个来做的
```
