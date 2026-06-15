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
2 在subsystem的tick中，会遍历HitResults，通过Si
```