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