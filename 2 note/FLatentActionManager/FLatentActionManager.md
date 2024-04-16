https://zhuanlan.zhihu.com/p/675932469
先给个文章，后面看这部分，先记录下

# 1 FLatentActionManager的添加
我们先从简单的开始，拿蓝图中Delay接口举例，就是将FDelayAction添加到FLatentActionManager里面。
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
# 2 FLatentActionManager的Tick

1 FLatentActionManager的Tick也是在World::Tick的里面，是在RunTickGroup之后，在TimerManager和TickableGameObject之前。
```cpp
if( !bIsPaused )
{
	CSV_SCOPED_TIMING_STAT_EXCLUSIVE(FlushLatentActions);
	// This will process any latent actions that have not been processed already
	CurrentLatentActionManager.ProcessLatentActions(nullptr, DeltaSeconds);
}
```
2 就是拿到当前的latentActionManager然后调用ProcessLatentActions方法。
```cpp
void FLatentActionManager::ProcessLatentActions(UObject* InObject, float DeltaTime)
{
	if (InObject && !InObject->GetClass()->HasAnyClassFlags(CLASS_CompiledFromBlueprint))
	{
		return;
	}

#if LATENT_ACTION_PROFILING_ENABLED
	GLatentActionStats.Reset();
	const double StartTime = FPlatformTime::Seconds();
#endif // LATENT_ACTION_PROFILING_ENABLED

	if (!ActionsToRemoveMap.IsEmpty())
	{
		// 这个部分就是看是否可以出的Action还有
	}

	if (InObject)
	{
		// 这部分就是看是否传进来了Action，world::tick里面是没有的
	}
	else if (!ObjectToActionListMap.IsEmpty())
	{
		SCOPE_CYCLE_COUNTER(STAT_TickLatentActions);
		// 遍历这个ObjectToActionListMap数组，这个数组就是添加进去的Action
		for (FObjectToActionListMap::TIterator ObjIt(ObjectToActionListMap); ObjIt; ++ObjIt)
		{	
			// 拥有这个Action的UObject的Weak_ptr
			TWeakObjectPtr<UObject> WeakPtr = ObjIt.Key();
			// 通过weak_ptr获取Uobject,这个就是需要调用callback的UObject
			UObject* Object = WeakPtr.Get();
			// 拿到FObjectActions,里面是ActionList和标志位bProcessedThisFrame
			FObjectActions* ObjectActions = ObjIt.Value().Get();
			check(ObjectActions);
			// 拿到ActionList,也就是map，key是唯一id，value是具体的LatentAction
			FActionList& ObjectActionList = ObjectActions->ActionList;
			if (Object)
			{
				// Tick all outstanding actions for this object
				// 判断标志位 && ActionList里面有具体的LatentAction
				if (!ObjectActions->bProcessedThisFrame && ObjectActionList.Num() > 0)
				{
#if LATENT_ACTION_PROFILING_ENABLED
					FScopedLatentActionTimer Timer(Object);
#endif // LATENT_ACTION_PROFILING_ENABLED
					// 执行具体的Tick方法
					TickLatentActionForObject(DeltaTime, ObjectActionList, Object);
					ensure(ObjectActions == ObjIt.Value().Get());
					// Tick完了之后改变标志位
					ObjectActions->bProcessedThisFrame = true;
				}
			}
			else
			{
				// Terminate all outstanding actions for this object, which has been GCed
				// 看上去应该是拥有Action的Object销毁了，但Action还没销毁，就发个销毁通知
				for (TMultiMap<int32, FPendingLatentAction*>::TConstIterator It(ObjectActionList); It; ++It)
				{
					if (FPendingLatentAction* Action = It.Value())
					{
						Action->NotifyObjectDestroyed();
						delete Action;
					}
				}
				ObjectActionList.Reset();
			}

			// Remove the entry if there are no pending actions remaining for this object (or if the object was NULLed and cleaned up)
			// 如果当前没有ActionList，就移除
			if (ObjectActionList.Num() == 0)
			{
				ObjIt.RemoveCurrent();
			}
		}
	}
}
```
3 Tick的流程，遍历ObjectToActionListMap数组，找到具体的ActionList（也就是个map，key是LatentAction的唯一id，value是LatentAction）和Object（调用回调的UObject）。然后调用TickLatentActionForObject方法。
```cpp
void FLatentActionManager::TickLatentActionForObject(float DeltaTime, FActionList& ObjectActionList, UObject* InObject)
{
	typedef TPair<int32, FPendingLatentAction*> FActionListPair;
	TArray<FActionListPair, TInlineAllocator<4>> ItemsToRemove;
	// 创建这个Response
	FLatentResponse Response(DeltaTime);
	// 遍历我们的ActionList，拿到具体了LatentAction，执行其UpdateOperation方法
	for (TMultiMap<int32, FPendingLatentAction*>::TConstIterator It(ObjectActionList); It; ++It)
	{
		FPendingLatentAction* Action = It.Value();

		Response.bRemoveAction = false;

		Action->UpdateOperation(Response);

		if (Response.bRemoveAction)
		{
			ItemsToRemove.Emplace(It.Key(), Action);
		}
	}

	// Remove any items that were deleted
	// 遍历移除数组，将里面的东西从ActionList中移除
	for (const FActionListPair& ItemPair : ItemsToRemove)
	{
		const int32 ItemIndex = ItemPair.Key;
		FPendingLatentAction* DyingAction = ItemPair.Value;
		ObjectActionList.Remove(ItemIndex, DyingAction);
		delete DyingAction;
	}

	// 如果移除数组中有东西的话，就向回调UObject广播个Delegate
	if (ItemsToRemove.Num() > 0)
	{
		LatentActionsChangedDelegate.Broadcast(InObject, ELatentActionChangeType::ActionsRemoved);
	}

	// Trigger any pending execution links
	// 下边这个是触发执行回调的逻辑
	for (FLatentResponse::FExecutionInfo& LinkInfo : Response.LinksToExecute)
	{
		if (LinkInfo.LinkID != INDEX_NONE)
		{
			if (UObject* CallbackTarget = LinkInfo.CallbackTarget.Get())
			{
				check(CallbackTarget == InObject);

				if (UFunction* ExecutionFunction = CallbackTarget->FindFunction(LinkInfo.ExecutionFunction))
				{
					CallbackTarget->ProcessEvent(ExecutionFunction, &(LinkInfo.LinkID));
				}
				else
				{
					UE_LOG(LogScript, Warning, TEXT("FLatentActionManager::ProcessLatentActions: Could not find latent action resume point named '%s' on '%s' called by '%s'"),
						*LinkInfo.ExecutionFunction.ToString(), *(CallbackTarget->GetPathName()), *(InObject->GetPathName()));
				}
			}
			else
			{
				UE_LOG(LogScript, Warning, TEXT("FLatentActionManager::ProcessLatentActions: CallbackTarget is None."));
			}
		}
	}
}
```
4 TickLatentActionForObject这个方法总共分成了四部分，第一部分是对ObjectActionList的遍历，找到具体的LatentAction，然后执行其UpdateOperation方法。第二部分是对ItemsToRemove数组的遍历，将需要移除的Acton从ObjectActionList中移除掉。第三部分是ItemsToRemove数组里有东西的话，就向Object发送个Delegate。第四部分就是通过反射调用回调方法了。