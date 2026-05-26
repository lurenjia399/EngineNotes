# 1 预测
## PredictionKey
```cpp
// 客户端生成PredictionKey方法，服务器不生成
void FPredictionKey::GenerateDependentPredictionKey()
{
	if (bIsServerInitiated)
	{
		// Can't have dependent keys on server keys, use same key
		return;
	}

	KeyType Previous = Current;
	if (Base == 0)
	{
		Base = Current;
	}

	// 下面介绍了，增加Current计数
	GenerateNewPredictionKey();

	if (Previous > 0)
	{
		FPredictionKeyDelegates::AddDependency(Current, Previous);
	}
}

void FPredictionKey::GenerateNewPredictionKey()
{
	// 静态局部变量，只初始化一次，只能在这个方法里使用，生命周期是整个进程
	static KeyType GKey = 1;
	// 增加Current计数
	Current = GKey++;
	// 应该是防止增加计数的时候，计数不变，保证计数正常增加
	if (GKey <= 0)
	{
		GKey = 1;
	}
}
```

```cpp
bool FPredictionKey::NetSerialize(FArchive& Ar, class UPackageMap* Map, bool& bOutSuccess)
{
	// 设置网络版本
	Ar.UsingCustomVersion(FEngineNetworkCustomVersion::Guid);
	// 是不支持同步PredictionKey的版本
	const bool bReplicateDeprecatedBaseForDemoPurposes = (Ar.EngineNetVer() < FEngineNetworkCustomVersion::PredictionKeyBaseNotReplicated);

	// 同步数据的第一位表示Connection是否有效
	uint8 ValidKeyForConnection = 0;
	if (Ar.IsSaving())
	{	
		ValidKeyForConnection = (Current > 0) 
			&& (bIsServerInitiated || 
				(PredictiveConnectionObjectKey == FObjectKey()) 
				|| (PredictiveConnectionObjectKey == FObjectKey(Map)));
	}
	Ar.SerializeBits(&ValidKeyForConnection, 1);

	// 同步数据的第二位表示是否有Base，（在Connection有效的情况下）
	uint8 HasBaseKey = 0;
	if (bReplicateDeprecatedBaseForDemoPurposes && ValidKeyForConnection)
	{
		if (Ar.IsSaving())
		{
			HasBaseKey = Base > 0;
		}
		Ar.SerializeBits(&HasBaseKey, 1);
	}

	// 同步数据的第三位是bIsServerInitiated
	uint8 ServerInitiatedByte = bIsServerInitiated;
	Ar.SerializeBits(&ServerInitiatedByte, 1);
	bIsServerInitiated = ServerInitiatedByte & 1;

	// 如果是有效Connection，再把Currnt和Base序列化进去
	if (ValidKeyForConnection)
	{
		Ar << Current;
		if (HasBaseKey)
		{
			Ar << Base;
		}
	}
	// 如果是从Ar里读
	if (Ar.IsLoading())
	{
		// 如果服务器还没初始化过这个PredictionKey，这里就标记下
		if (!bIsServerInitiated)
		{
			// 记录这个Map
			PredictiveConnectionObjectKey = FObjectKey(Map);
		}
	}

	bOutSuccess = true;
	return true;
}
```
## 1 FScopedPredictionWindow
```cpp
FScopedPredictionWindow::FScopedPredictionWindow(
	UAbilitySystemComponent* InAbilitySystemComponent, 
	bool bCanGenerateNewKey)
{

	ClearScopedPredictionKey = false;
	// 是在Scoped析构的时候，是否需要同步
	SetReplicatedPredictionKey = false;

	// 设置Owner是AbilityComp
	Owner = InAbilitySystemComponent;
	// 如果不是可同步的 || 是权威的
	if (!ensure(Owner.IsValid()) 
		|| InAbilitySystemComponent->IsNetSimulating() == false)
	{
		return;
	}

	// Because of the check above, this is expected to only run on the client.
	// 如果要生成新的Key
	if (bCanGenerateNewKey)
	{
		
		ClearScopedPredictionKey = true;
		// 把当前AbilityComp中的PredictionKey缓存到Window中
		RestoreKey = InAbilitySystemComponent->ScopedPredictionKey;
		// 通过AbilityComp生成新的PredictionKey
		InAbilitySystemComponent-
		>	ScopedPredictionKey.GenerateDependentPredictionKey();
	}

#if !UE_BUILD_SHIPPING
	// Add some debugging functionality if we're the first scoped prediction window (will become a BaseKey value)
	if (bCanGenerateNewKey && !RestoreKey.IsValidKey())
	{
		// Intercept any RPC's of FPredictionKeys so that we can mark them as sent to the server.
		// For RPC's not marked as sent-to-server, we can log them and track them down with breakpoints.
		const UWorld* NetWorldPtr = Owner->GetWorld();
		UNetDriver* NetDriver = NetWorldPtr ? NetWorldPtr->GetNetDriver() : nullptr;
		if (NetDriver && NetDriver->GetNetMode() == NM_Client)
		{
			DebugSavedNetDriver = NetDriver;
			DebugSavedOnSendRPC = NetDriver->SendRPCDel;
			DebugBaseKeyOfChain = InAbilitySystemComponent->ScopedPredictionKey.Current;

			NetDriver->SendRPCDel.BindWeakLambda(InAbilitySystemComponent,
				[this, SendRPCDel = DebugSavedOnSendRPC](AActor* Actor, UFunction* Function, void* Parameters, FOutParmRec* OutParms, FFrame* Stack, UObject* SubObject, bool& bBlockSendRPC)
				{
					// Chain the previous RPC callback (e.g. packet loss simulation)
					SendRPCDel.ExecuteIfBound(Actor, Function, Parameters, OutParms, Stack, SubObject, bBlockSendRPC);

					// If we've already reset this, it's an indication this PredictionKey Chain has already been sent.
					if (!DebugBaseKeyOfChain.IsSet())
					{
						return;
					}

					const FPredictionKey::KeyType BaseKey = DebugBaseKeyOfChain.GetValue();

					// See if we are communicating a key based on this scope, if so, reset our debug value so as not to warn that it was never sent on Scope destruct.
					TArray<FPredictionKey> PassedInKeys = UE::AbilitySystem::Private::FindPredictionKeysInPropertyLink(Function->PropertyLink, static_cast<uint8_t*>(Parameters));
					for (const FPredictionKey& PassedInKey : PassedInKeys)
					{
						if (PassedInKey.Current == BaseKey || PassedInKey.Base == BaseKey)
						{
							UE_LOG(LogPredictionKey, Verbose, TEXT("Sent key %s with %s to confirm BaseKey %d"), *PassedInKey.ToString(), *Function->GetName(), BaseKey);
							DebugBaseKeyOfChain.Reset();
							break;
						}
					}
				});
		}
	}
#endif
}
```


## 2 GA预测流程

客户端激活GA，并通知服务器校验
```cpp
// 客户端来TryActivateAbility
UAbilitySystemComponent::InternalTryActivateAbility(...)
{
	// 其余不关心的全省略掉，只关心预测相关的
	
	// 创建GA的激活信息
	ActivationInfo = FGameplayAbilityActivationInfo(ActorInfo->OwnerActor.Get());
	
	// GA是预测执行的
	if (Ability->GetNetExecutionPolicy() == 
		EGameplayAbilityNetExecutionPolicy::LocalPredicted)
	{
		
		// 客户端生成新的PredictionKey，并创建PredictionWindow
		FScopedPredictionWindow ScopedPredictionWindow(this, true);
		// 在激活信息中设置PredictionKey
		ActivationInfo.SetPredicting(ScopedPredictionKey);
		
		// This must be called immediately after GeneratePredictionKey to prevent problems with recursively activating abilities
		// 发送RPC，RPC带上PredictionKey，让服务器验证激活GA
		if (TriggerEventData)
		{
			ServerTryActivateAbilityWithEventData(Handle, Spec->InputPressed, ScopedPredictionKey, *TriggerEventData);
		}
		else
		{
			CallServerTryActivateAbility(Handle, Spec->InputPressed, ScopedPredictionKey);
		}

		// 给这个PredictionKey生成回调，在服务器回应客户端会执行
		// When this prediction key is caught up, we better know if the ability was confirmed or rejected
		ScopedPredictionKey.NewCaughtUpDelegate()
			.BindUObject(this, 
				&UAbilitySystemComponent::OnClientActivateAbilityCaughtUp, 
					Handle, ScopedPredictionKey.Current);
		// 激活GA
		AbilitySource->CallActivateAbility(
			Handle, ActorInfo, ActivationInfo, 
			OnGameplayAbilityEndedDelegate, TriggerEventData);
	}
}
```
服务器校验
```cpp
// 收到客户端发送的RPC后，服务器也TryActivateAbility
void UAbilitySystemComponent::InternalServerTryActivateAbility(
	FGameplayAbilitySpecHandle Handle, 
	bool InputPressed, 
	const FPredictionKey& PredictionKey, 
	const FGameplayEventData* TriggerEventData)
{
#if WITH_SERVER_CODE

	// 如果是一些异常情况，就直接通知客户端激活失败
	FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(Handle);
	if (!Spec)
	{
		ClientActivateAbilityFailed(Handle, PredictionKey.Current);
		return;
	}
	const UGameplayAbility* AbilityToActivate = Spec->Ability;
	if (!ensure(AbilityToActivate))
	{
		ClientActivateAbilityFailed(Handle, PredictionKey.Current);
		return;
	}
	if (AbilityToActivate->GetNetSecurityPolicy() == 
		EGameplayAbilityNetSecurityPolicy::ServerOnlyExecution ||
			AbilityToActivate->GetNetSecurityPolicy() == 
			EGameplayAbilityNetSecurityPolicy::ServerOnly)
	{
		ClientActivateAbilityFailed(Handle, PredictionKey.Current);
		return;
	}

	//
	// Consume any pending target info, to clear out cancels from old executions
	ConsumeAllReplicatedData(Handle, PredictionKey);
	// 根据客户端传上来的PredictionKey创建一个PredictionWindow
	FScopedPredictionWindow ScopedPredictionWindow(this, PredictionKey);

	UGameplayAbility* InstancedAbility = nullptr;
	Spec->InputPressed = true;

	// Attempt to activate the ability (server side) and tell the client if it succeeded or failed.
	// 服务器上尝试激活Ability
	if (InternalTryActivateAbility(
		Handle, PredictionKey, &InstancedAbility, nullptr, TriggerEventData))
	{
		// TryActivateAbility handles notifying the client of success, but let's still log it
	}
	// 服务器上尝试激活Ability失败了，就通知客户端校验失败
	else
	{
		ClientActivateAbilityFailed(Handle, PredictionKey.Current);
		Spec->InputPressed = false;
		MarkAbilitySpecDirty(*Spec);
	}
#endif
}

// 服务器由上边调用，校验能否激活Ability
UAbilitySystemComponent::InternalTryActivateAbility(...)
{
	if (Ability->GetNetExecutionPolicy() == 
			EGameplayAbilityNetExecutionPolicy::LocalOnly 
			|| (NetMode == ROLE_Authority))
	{
		// if we're the server and don't have a valid key or this ability should be started on the server create a new activation key
		// 判断服务器上是否需要创建PredictionKey
		bool bCreateNewServerKey = NetMode == ROLE_Authority &&
			(!InPredictionKey.IsValidKey() ||
			(Ability->GetNetExecutionPolicy() == 
				EGameplayAbilityNetExecutionPolicy::ServerInitiated ||
			  Ability->GetNetExecutionPolicy() == 
			  EGameplayAbilityNetExecutionPolicy::ServerOnly));
		// 如果需要创建新的PredictionKey，说明不是校验
		if (bCreateNewServerKey)
		{
			ActivationInfo.ServerSetActivationPredictionKey(
				FPredictionKey::CreateNewServerInitiatedKey(this));
		}
		// PredictionKey是有效的，说明需要校验
		else if (InPredictionKey.IsValidKey())
		{
			// 把PredictionKey设置到GASpec里
			// Otherwise if available, set the prediction key to what was passed up
			ActivationInfo.ServerSetActivationPredictionKey(InPredictionKey);
		}

		// 根据PredictionKey来创建PredictionWindow
		// we may have changed the prediction key so we need to update the scoped key to match
		FScopedPredictionWindow ScopedPredictionWindow(this, ActivationInfo.GetActivationPredictionKey());

		// ----------------------------------------------
		// Tell the client that you activated it (if we're not local and not server only)
		// 通知客户端，服务器校验通过
		// ----------------------------------------------
		if (!bIsLocal && Ability->GetNetExecutionPolicy() != EGameplayAbilityNetExecutionPolicy::ServerOnly)
		{
			ClientActivateAbilitySucceed(
					Handle, ActivationInfo.GetActivationPredictionKey());
		}

		// ----------------------------------------------
		//	Call ActivateAbility (note this could end the ability too!)
		// ----------------------------------------------
		// 服务器执行GAInstance的ActivateAbility方法
		// Create instance of this ability if necessary
		if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerExecution)
		{
			InstancedAbility = CreateNewInstanceOfAbility(*Spec, Ability);
			InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
		else
		{
			AbilitySource->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
	}
}
```

校验失败
```cpp
void UAbilitySystemComponent::ClientActivateAbilityFailed_Implementation(
	FGameplayAbilitySpecHandle Handle, 
	int16 PredictionKey)
{
	// Tell anything else listening that this was rejected
	// 根据PredictionKey上绑定的执行失败广播
	if (PredictionKey > 0)
	{
		FPredictionKeyDelegates::BroadcastRejectedDelegate(PredictionKey);
	}

	// Find the actual UGameplayAbility	
	// 客户端找不到被拒绝的GA	
	FGameplayAbilitySpec* Spec = FindAbilitySpecFromHandle(Handle);
	if (Spec == nullptr)
	{
		return;
	}
	// 设置这个GASpec为被拒绝的
	// The ability should be either confirmed or rejected by the time we get here
	if (Spec->ActivationInfo.GetActivationPredictionKey().Current == PredictionKey)
	{
		Spec->ActivationInfo.SetActivationRejected();
	}
	// 设置GASpec里的GAInstance为被拒绝的，并EndAbility
	TArray<UGameplayAbility*> Instances = Spec->GetAbilityInstances();
	for (UGameplayAbility* Ability : Instances)
	{
		if (Ability->CurrentActivationInfo.GetActivationPredictionKey().Current == PredictionKey)
		{
			Ability->CurrentActivationInfo.SetActivationRejected();
			Ability->K2_EndAbility();
		}
	}
}
```

校验成功
```cpp
void UAbilitySystemComponent::ClientActivateAbilitySucceedWithEventData_Implementation(
	FGameplayAbilitySpecHandle Handle, 
	FPredictionKey PredictionKey, 
	FGameplayEventData TriggerEventData)
{
	// 客户端上的GASpec标记成确认
	// Confirm and allow the remote ending of the ability
	ActivationInfo.SetActivationConfirmed();
	// 是预测类型的GA
	const bool bLocallyPredicted = AbilityToActivate->NetExecutionPolicy == EGameplayAbilityNetExecutionPolicy::LocalPredicted;
	if (bLocallyPredicted)
	{
		if (bNonInstanced)
		{
			// AbilityToActivate->ConfirmActivateSucceed(); // This doesn't do anything for non instanced
		}
		else
		{
			// 通知GASpec里面的GAInstance也经过服务器校验了,并广播
			// Find the one we predictively spawned, tell them we are confirmed
			bool found = false;
			TArray<UGameplayAbility*> Instances = Spec->GetAbilityInstances();
			for (UGameplayAbility* LocalAbility : Instances)
			{
				if (LocalAbility != nullptr && LocalAbility->GetCurrentActivationInfo().GetActivationPredictionKey() == PredictionKey)
				{
					LocalAbility->ConfirmActivateSucceed();
					found = true;
					break;
				}
			}

			if (!found)
			{
				ABILITY_LOG(Verbose, TEXT("Ability %s was confirmed by server but no longer exists on client (replication key: %s)"), *AbilityToActivate->GetName(), *PredictionKey.ToString());
			}
		}
	}
}
```

GA激活可能会施加CD，减能量的buff，这些buff的预测回滚怎么实现的？

# GE预测（只有Instance可以预测）

非权威端执行
```cpp
FActiveGameplayEffectHandle UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(
	const FGameplayEffectSpec &Spec, // GE描述符
	FPredictionKey PredictionKey// 带有这个PredictionKey
	)
{
	/*
	1 客户端不允许预测持续型的GE，需要在权威端执行持续型GE
	2 权威端如果有PredictionKey，就清掉,此次执行添加GE的操作就没有PredictionKey
	*/
	// Don't allow prediction of periodic effects
	if (PredictionKey.IsValidKey() && Spec.GetPeriod() > 0.f)
	{
		if (IsOwnerActorAuthoritative())
		{
			// Server continue with invalid prediction key
			PredictionKey = FPredictionKey();
		}
		else
		{
			// Client just return now
			return FActiveGameplayEffectHandle();
		}
	}
	
	// 不是权威端，PredictionKey是客户端Key，瞬时GE，意思就是可以预测的GE
	bool bTreatAsInfiniteDuration = GetOwnerRole() != ROLE_Authority 
		&& PredictionKey.IsLocalClientKey() 
		&& Spec.Def->DurationPolicy == EGameplayEffectDurationType::Instant;
	// 如果是可以预测的，就激活GE，也就是创建ActiveGameplayEffect，回滚的核心操作在这方法里绑定的
	if (Spec.Def->DurationPolicy != EGameplayEffectDurationType::Instant 
		|| bTreatAsInfiniteDuration)
	{
		AppliedEffect = ActiveGameplayEffects.ApplyGameplayEffectSpec(
			Spec, PredictionKey, bFoundExistingStackableGE);
	}
}

FActiveGameplayEffect* FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec(
	const FGameplayEffectSpec& Spec, 
	FPredictionKey& InPredictionKey, 
	bool& bFoundExistingStackableGE
	)
{
	// 捕获需要的属性，记录在CapturedRelevantAttributes里
	AppliedEffectSpec.CaptureAttributeDataFromTarget(Owner);
	// 计算需要修改属性的修改值，记录在Modifiers里
	AppliedEffectSpec.CalculateModifierMagnitudes();
	// 在PredictionKey上绑定服务器校验结果的回调
	InPredictionKey.NewRejectOrCaughtUpDelegate(
		FPredictionKeyEvent::CreateUObject(Owner, 
		&UAbilitySystemComponent::RemoveActiveGameplayEffect_AllowClientRemoval,
		AppliedActiveGE->Handle, -1));
	// 应用GE效果，走GE里Components那些
	const bool bInvokeGameplayCueEvents = 
		(Spec.Def->DurationPolicy != EGameplayEffectDurationType::Instant);
		InternalOnActiveGameplayEffectAdded(
			*AppliedActiveGE, bInvokeGameplayCueEvents);
}
```

权威端校验
```cpp
FActiveGameplayEffectHandle UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(
	const FGameplayEffectSpec &Spec, 
	FPredictionKey PredictionKey)
{
	else if (Spec.Def->DurationPolicy == EGameplayEffectDurationType::Instant)
	{
		// 如果是InstanceGE，直接执行GE效果
		// This is a non-predicted instant effect (it never gets added to ActiveGameplayEffects)
		ExecuteGameplayEffect(*OurCopyOfSpec, PredictionKey);
	}
}
```

1 InstanceGE可以通过非权威端预测，在非权威端会通过ApplyGameplayEffectSpec方法将预测ActiveGE添加到ActiveGEContainer里（这里有个池化技术），放到Container里就相当于将InstanceGE变成了持续型GE。然后也会计算InstanceGE的属性变化量。
2 InstanceGE在权威端是不执行ApplyGameplayEffectSpec方法的，直接执行ExecuteGameplayEffect方法来应用效果。
3 权威端在FScopedPredictionWindow这个scope析构的时候会往客户端同步这个PredictionKey，告诉客户端追上了CatchUpTo。
4 注意：通过ApplyGameplayEffectSpec方法可以将ActiveGE添加到GameplayEffects_Internal 这个数组中，这个数组也是增量同步的数组。如果是InstanceGE，非权威端是预测添加到数组中的，权威端是直接执行GE效果的，权威端通过PredictionKey的增量同步来移除非权威端数组中的值。

# 属性预测


非权威端执行(InstanceGE)
```cpp
// 非权威端InstanceGE，或者非InstanceGE会执行
FActiveGameplayEffect* FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec(
	const FGameplayEffectSpec& Spec, 
	FPredictionKey& InPredictionKey, 
	bool& bFoundExistingStackableGE)
{
	// 快照属性
	AppliedEffectSpec.CaptureAttributeDataFromTarget(Owner);
	// 计算属性的Modifier，先算出属性的改变量
	AppliedEffectSpec.CalculateModifierMagnitudes();
	
	// 不是InstanceGE可以执行GC，这个里面根据GESpec计算出属性Modifier，然后
	const bool bInvokeGameplayCueEvents = 
		(Spec.Def->DurationPolicy != EGameplayEffectDurationType::Instant);
	InternalOnActiveGameplayEffectAdded(
		*AppliedActiveGE, bInvokeGameplayCueEvents);
}

void FActiveGameplayEffectsContainer::InternalOnActiveGameplayEffectAdded(
	FActiveGameplayEffect& Effect, 
	const bool bInvokeGameplayCueEvents)
{
	// 具体激活GE方法 SetActiveGameplayEffectInhibit，里面会把属性Mod添加到聚合器里
	FActiveGameplayEffectHandle EffectHandle = Effect.Handle;
	Owner->SetActiveGameplayEffectInhibit(MoveTemp(EffectHandle), !bActive, bInvokeGameplayCueEvents);
}


FActiveGameplayEffectHandle UAbilitySystemComponent::SetActiveGameplayEffectInhibit(FActiveGameplayEffectHandle&& ActiveGEHandle, bool bInhibit, bool bInvokeGameplayCueEvents)
{
	if (ActiveGE->bIsInhibited != bInhibit)
	{
		ActiveGE->bIsInhibited = bInhibit;
		
		/*
		1 创建新的Scope，在这个析构的时候会 BroadcastOnDirty 广播脏属性
		2 在 FindOrCreateAttributeAggregator 方法中会监听脏属性回调
		3 脏属性回调中会执行属性计算，本地根据新的属性mod和BaseValue计算出CurrentValue
		*/
		FScopedActiveGameplayEffectLock ScopeLockActiveGameplayEffects(
			ActiveGameplayEffects);

		FScopedAggregatorOnDirtyBatch	AggregatorOnDirtyBatcher;
		if (bInhibit)
		{
			ActiveGameplayEffects.
				RemoveActiveGameplayEffectGrantedTagsAndModifiers(
				*ActiveGE, bInvokeGameplayCueEvents);
		}
		else
		{
			ActiveGameplayEffects.AddActiveGameplayEffectGrantedTagsAndModifiers(
				*ActiveGE, bInvokeGameplayCueEvents);
		}
}


void FActiveGameplayEffectsContainer::AddActiveGameplayEffectGrantedTagsAndModifiers(
	FActiveGameplayEffect& Effect, 
	bool bInvokeGameplayCueEvents)
{
	// 如果是 InstanceGE 。
	if (Effect.Spec.GetPeriod() <= UGameplayEffect::NO_PERIOD)
	{
		for (int32 ModIdx = 0; ModIdx < Effect.Spec.Modifiers.Num(); ++ModIdx)
		{
			// 客户端根据属性创建新的聚合器，监听OnDirty回调，回调里执行属性改变
			FAggregator* Aggregator = 
				FindOrCreateAttributeAggregator(
				Effect.Spec.Def->Modifiers[ModIdx].Attribute).Get();
			// 应用这个聚合器
			if (ensure(Aggregator))
			{
				Aggregator->AddAggregatorMod(
					EvaluatedMagnitude, 
					ModInfo.ModifierOp, 
					ModInfo.EvaluationChannelSettings.GetEvaluationChannel(), 
					&ModInfo.SourceTags, 
					&ModInfo.TargetTags, 
					Effect.PredictionKey.WasLocallyGenerated(), 
					Effect.Handle);
			}
		}
	}
}

```

权威端校验(InstanceGE)
```cpp
FActiveGameplayEffectHandle UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(
	const FGameplayEffectSpec &Spec, 
	FPredictionKey PredictionKey)
{
	else if (Spec.Def->DurationPolicy == EGameplayEffectDurationType::Instant)
	{
		// 如果是InstanceGE，直接执行GE效果
		ExecuteGameplayEffect(*OurCopyOfSpec, PredictionKey);
	}
}
```

同步属性改变
```cpp
void UAttributeSet::PreNetReceive()
{
	FScopedAggregatorOnDirtyBatch::BeginNetReceiveLock();
}
	
void UAttributeSet::PostNetReceive()
{
	FScopedAggregatorOnDirtyBatch::EndNetReceiveLock();
}

/*
1 这个FScopedAggregatorOnDirtyBatch含义一样，就是广播脏属性事件
2 客户端的脏属性回调是在AddActiveGameplayEffectGrantedTagsAndModifiers这个方法里绑定的
3 脏属性回调里依然是执行一次属性计算
*/

```

1 非权威端在 ApplyGameplayEffectSpec 方法中执行，根据Modifier配置直接生成新的聚合器，然后根据聚合器计算出改变值，就是通过BaseValue + 改变值 = CurrentValue，非权威端通过Mod计算出CurrentValue。
2 InstanceGE的权威端直接调用ExecuteGameplayEffect这个方法，执行一遍属性改变的流程，直接改变属性的BaseValue。
3 等到属性同步下来后，会执行FScopedAggregatorOnDirtyBatch这个结构体的析构，然后就会执行OnAttributeAggregatorDirty这个方法，客户端会重新根据新的BaseValue在计算一遍CurrentValue。
4 在CatchUpTo追上客户端后，客户端会执行RemoveActiveGameplayEffect方法，把数组中的预测ActiveGE移除掉，顺便移除预测聚合器Mod。
注意 4和3都是属性同步，4是ASC中的ReplicatedPredictionKeyMap，3是UA



# EGameplayAbilityInstancingPolicy

```cpp
/**
	 *	How the ability is instanced when executed. This limits what an ability can do in its implementation. For example, a NonInstanced
	 *	Ability cannot have state. It is probably unsafe for an InstancedPerActor ability to have latent actions, etc.
	 */
	enum Type : int
	{
		// 这个GA不会创建实例，在GASpec中只会有CDO
		// This ability is never instanced. Anything that executes the ability is operating on the CDO.
		NonInstanced UE_DEPRECATED_FORGAME(5.5, "Use InstancedPerActor as the default to avoid confusing corner cases"),
		// 这个GA会创建实例，在GiveAbility的时候就会创建了
		// Each actor gets their own instance of this ability. State can be saved, replication is possible.
		InstancedPerActor,
		// 在需要用到实例的时候才会创建
		// We instance this ability each time it is executed. Replication currently unsupported.
		InstancedPerExecution,
	};
```

# GE激活的内存池实现

```cpp
FActiveGameplayEffectsContainer::~FActiveGameplayEffectsContainer()
{
	using namespace UE::GameplayEffect;

	FActiveGameplayEffect* PendingGameplayEffect = PendingGameplayEffectHead;

	if (HasActiveGameplayEffectFix(EActiveGameplayEffectFix::CleanupAllPendingActiveGEs))
	{
		while (PendingGameplayEffectHead)
		{
			FActiveGameplayEffect* Next = PendingGameplayEffectHead->PendingNext;
			delete PendingGameplayEffectHead;
			PendingGameplayEffectHead = Next;
		}
	}
	else
	{
		if (PendingGameplayEffectHead)
		{
			FActiveGameplayEffect* Next = PendingGameplayEffectHead->PendingNext;
			delete PendingGameplayEffectHead;
			PendingGameplayEffectHead = Next;
		}
	}
}
```

```cpp
FActiveGameplayEffectHandle UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf(
	const FGameplayEffectSpec &Spec, 
	FPredictionKey PredictionKey)
{
	// 只显示相关方法，其余的省略掉
	
	// 这个Scope在创建时会执行IncrementLock，在析构时会执行DecrementLock
	FScopedActiveGameplayEffectLock ScopeLock(ActiveGameplayEffects);
	
	AppliedEffect = ActiveGameplayEffects.ApplyGameplayEffectSpec(
		Spec, PredictionKey, bFoundExistingStackableGE);
}

void FActiveGameplayEffectsContainer::IncrementLock()
{
	ScopedLockCount++;
}

void FActiveGameplayEffectsContainer::DecrementLock()
{
	if (--ScopedLockCount == 0)
	{
		// ------------------------------------------
		// Move any pending effects onto the real list
		// ------------------------------------------
		FActiveGameplayEffect* PendingGameplayEffect = PendingGameplayEffectHead;
		FActiveGameplayEffect* Stop = *PendingGameplayEffectNext;
		bool ModifiedArray = false;

		while (PendingGameplayEffect != Stop)
		{
			if (!PendingGameplayEffect->IsPendingRemove)
			{
				GameplayEffects_Internal.Add(MoveTemp(*PendingGameplayEffect));
				ModifiedArray = true;
			}
			else
			{
				PendingRemoves--;
			}
			PendingGameplayEffect = PendingGameplayEffect->PendingNext;
		}

		// Reset our pending GameplayEffect linked list
		PendingGameplayEffectNext = &PendingGameplayEffectHead;

		// -----------------------------------------
		// Delete any pending remove effects
		// -----------------------------------------
		for (int32 idx=GameplayEffects_Internal.Num()-1; idx >= 0 && PendingRemoves > 0; --idx)
		{
			FActiveGameplayEffect& Effect = GameplayEffects_Internal[idx];

			if (Effect.IsPendingRemove)
			{
				UE_LOG(LogGameplayEffects, Verbose, TEXT("%s: Finish PendingRemove: %s. Auth: %d"), *GetNameSafe(Owner->GetOwnerActor()), *Effect.GetDebugString(), IsNetAuthority());
				
				// Remove this handle from the global map
				Effect.Handle.RemoveFromGlobalMap();

				GameplayEffects_Internal.RemoveAtSwap(idx, EAllowShrinking::No);
				ModifiedArray = true;
				PendingRemoves--;
			}
		}

		if (!ensure(PendingRemoves == 0))
		{
			UE_LOG(LogGameplayEffects, Error, TEXT("~FScopedActiveGameplayEffectLock has %d pending removes after a scope lock removal"), PendingRemoves);
			PendingRemoves = 0;
		}

		if (ModifiedArray)
		{
			MarkArrayDirty();
		}
	}
}
```


```cpp
FActiveGameplayEffect* FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec(
	const FGameplayEffectSpec& Spec, 
	FPredictionKey& InPredictionKey, 
	bool& bFoundExistingStackableGE)
{
	/*
	1 GameplayEffects_Internal 这个数组就是实际存储ActiveGE的数组，这个判断的含义是数组是否还有空间容纳新的数据，<=0就是不能容纳了，需要扩容了。如果需要扩容了就先用Pending队列来存储ActiveGE 
	*/
	if (GameplayEffects_Internal.GetSlack() <= 0)
	{
		const FActiveGameplayEffect* PreviousPendingNext =
			(*PendingGameplayEffectNext) ? 
				(*PendingGameplayEffectNext)->PendingNext : nullptr;
		/*
		2 如果*PendingGameplayEffectNext的内容是空的，说明Pending队列里还没东西，就需要New一个新的ActiveGE放到队列中
		*/
		if (*PendingGameplayEffectNext == nullptr)
		{
			AppliedActiveGE = new FActiveGameplayEffect(NewHandle, Spec, GetWorldTime(), GetServerWorldTime(), InPredictionKey);
			*PendingGameplayEffectNext = AppliedActiveGE;
		}
		/*
		3 如果*PendingGameplayEffectNext的内容不是空的，说明已经通过MoveTemp操作把Pending队列里的数据右移到 GameplayEffects_Internal 这个实际数组中了，这里就复用Pending队列，所以就直接在PendingNext指向的位置创建ActiveGE
		*/
		else
		{
			**PendingGameplayEffectNext = FActiveGameplayEffect(NewHandle, Spec, GetWorldTime(), GetServerWorldTime(), InPredictionKey);
			AppliedActiveGE = *PendingGameplayEffectNext;
		}
		/*
		4 重新赋值PendingNext指针，让他指向新创建的ActiveGE的Next
		*/
		PendingGameplayEffectNext = &AppliedActiveGE->PendingNext;
	}
	else
	{

		/*
		5 FActiveGameplayEffect 这个结构体重写了operate new方法，所以这里执行的placement new 实际上等同于 GameplayEffects_Internal.add（新的ActiveGE）
		*/
		AppliedActiveGE = new(GameplayEffects_Internal) FActiveGameplayEffect(NewHandle, Spec, GetWorldTime(), GetServerWorldTime(), InPredictionKey);
	}
}
```
1 ApplyGameplayEffectSpec 这个方法就是实际激活GE的方法，里面会创建ActiveGE，会把ActiveGE放到GameplayEffects_Internal这个数组里面。
2 如果GameplayEffects_Internal这个数组已经满了，就会使用Pending队列的方式，在堆上创建ActiveGE。
3 在实际激活GE前，会创建 FScopedActiveGameplayEffectLock 这个Scope，这个ActiveGEScope的作用就是在析构的时候，把Pending队列上的ActiveGE右移到GameplayEffects_Internal这个数组里，右移完后Pending队列上只存在空壳，实际数据就在GameplayEffects_Internal数组中了。
4 FActiveGameplayEffectsContainer 在这个的析构函数，才会遍历Pending队列，一次执行delete，所以也不会内存泄漏。

# 关键
1 PredictionKey设计方式
2 Pending队列的池化设计方式
3 FastArraySerializer的使用