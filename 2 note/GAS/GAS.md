# 1 预测
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
		// 把当前AbilityComp中的Pre
		RestoreKey = InAbilitySystemComponent->ScopedPredictionKey;
		InAbilitySystemComponent->ScopedPredictionKey.GenerateDependentPredictionKey();
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
	
	// GA是预测执行的
	if (Ability->GetNetExecutionPolicy() == 
		EGameplayAbilityNetExecutionPolicy::LocalPredicted)
	{
		

		// 客户端生成新的PredictionKey
		FScopedPredictionWindow ScopedPredictionWindow(this, true);

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