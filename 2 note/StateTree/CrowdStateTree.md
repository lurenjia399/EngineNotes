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
3 HitReceived也被UMassStateTreeProcessor这个订阅，SignalEntities这个方法就是订阅回调，执行statetree的状态更新
4 statetree中有FHTMassComponentHitEvaluator，在Evaluator的tick会通过UHTMassComponentHitSubsystem获取当前Entity是否被击中，如果击中了就进入Hit的状态
```