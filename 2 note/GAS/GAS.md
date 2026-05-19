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
```cpp
UAbilitySystemComponent::InternalTryActivateAbility(...)
{
	// 其余不关心的全省略掉，只关心预测相关的
	
	// 创建GA的激活信息
	ActivationInfo = FGameplayAbilityActivationInfo(ActorInfo->OwnerActor.Get());
	
	// GA是预测执行的
	if (Ability->GetNetExecutionPolicy() == 
		EGameplayAbilityNetExecutionPolicy::LocalPredicted)
	{
		
		// 客户端生成新的PredictionKey
		FScopedPredictionWindow ScopedPredictionWindow(this, true);
		// 在激活信息中设置PredictionKey
		ActivationInfo.SetPredicting(ScopedPredictionKey);
		
		// This must be called immediately after GeneratePredictionKey to prevent problems with recursively activating abilities
		if (TriggerEventData)
		{
			ServerTryActivateAbilityWithEventData(Handle, Spec->InputPressed, ScopedPredictionKey, *TriggerEventData);
		}
		else
		{
			CallServerTryActivateAbility(Handle, Spec->InputPressed, ScopedPredictionKey);
		}

		// When this prediction key is caught up, we better know if the ability was confirmed or rejected
		ScopedPredictionKey.NewCaughtUpDelegate().BindUObject(this, &UAbilitySystemComponent::OnClientActivateAbilityCaughtUp, Handle, ScopedPredictionKey.Current);

		if (Ability->GetInstancingPolicy() == EGameplayAbilityInstancingPolicy::InstancedPerExecution)
		{
			// For now, only NonReplicated + InstancedPerExecution abilities can be Predictive.
			// We lack the code to predict spawning an instance of the execution and then merge/combine
			// with the server spawned version when it arrives.

			if (Ability->GetReplicationPolicy() == EGameplayAbilityReplicationPolicy::ReplicateNo)
			{
				InstancedAbility = CreateNewInstanceOfAbility(*Spec, Ability);
				InstancedAbility->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
			}
			else
			{
				ABILITY_LOG(Error, TEXT("InternalTryActivateAbility called on ability %s that is InstancedPerExecution and Replicated. This is an invalid configuration."), *Ability->GetName() );
			}
		}
		else
		{
			AbilitySource->CallActivateAbility(Handle, ActorInfo, ActivationInfo, OnGameplayAbilityEndedDelegate, TriggerEventData);
		}
	}
}
```