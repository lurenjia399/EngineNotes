https://zhuanlan.zhihu.com/p/675932469
先给个文章，后面看这部分，先记录下

# 1 FDelayAction的添加
我们先从简单的开始，拿蓝图中Delay接口举例。
```cpp
void UKismetSystemLibrary::Delay(const UObject* WorldContextObject, float Duration, FLatentActionInfo LatentInfo )
{
	if (UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull))
	{
		FLatentActionManager& LatentActionManager = World->GetLatentActionManager();
		if (LatentActionManager.FindExistingAction<FDelayAction>(LatentInfo.CallbackTarget, LatentInfo.UUID) == NULL)
		{
			LatentActionManager.AddNewAction(LatentInfo.CallbackTarget, LatentInfo.UUID, new FDelayAction(Duration, LatentInfo));
		}
	}
}
```
1 这个静态方法就是蓝图中的Delay节点执行的部分，字面意思就是获取World成功，然后通过World获取到LatentActionManager（这个东西是存储在GameInstance里面的，构造函数中创建），然后创建一个FDelayAction，添加到LatenActionManager里面。
```cpp
class FDelayAction : public FPendingLatentAction
{
public:
	float TimeRemaining;
	FName ExecutionFunction;
	int32 OutputLink;
	FWeakObjectPtr CallbackTarget;

	FDelayAction(float Duration, const FLatentActionInfo& LatentInfo)
		: TimeRemaining(Duration)
		, ExecutionFunction(LatentInfo.ExecutionFunction)
		, OutputLink(LatentInfo.Linkage)
		, CallbackTarget(LatentInfo.CallbackTarget)
	{
	}

	virtual void UpdateOperation(FLatentResponse& Response) override
	{
		TimeRemaining -= Response.ElapsedTime();
		Response.FinishAndTriggerIf(TimeRemaining <= 0.0f, ExecutionFunction, OutputLink, CallbackTarget);
	}
};
```
2 这个FDelayAction非常的简单，即使进行一些赋值，然后重写了UpdateOperation方法，这个方法应该是Tick中会执行到。
```cpp
void FLatentActionManager::AddNewAction(UObject* InActionObject, int32 UUID, FPendingLatentAction* NewAction)
{
	TSharedPtr<FObjectActions>& ObjectActions = ObjectToActionListMap.FindOrAdd(InActionObject);
	if (!ObjectActions.Get())
	{
		ObjectActions = MakeShareable(new FObjectActions());
	}
	ObjectActions->ActionList.Add(UUID, NewAction);

	LatentActionsChangedDelegate.Broadcast(InActionObject, ELatentActionChangeType::ActionsAdded);
}
```
3 这个方法就是添加新的Action，很简单，就是添加到ObjectToActionListMap这个map数组里面，然后还会发个广播。
