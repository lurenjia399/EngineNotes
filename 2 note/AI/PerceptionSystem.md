# UAIPerceptionSystem::Tick

```cpp
void UAIPerceptionSystem::Tick(float DeltaSeconds)
{
	CurrentTime = World->GetTimeSeconds();
	// 有注册的感知原，就先处理
	/*
	1 有需要注册的感知源就先处理
	2 给
	*/
	if (SourcesToRegister.Num() > 0)
	{
		PerformSourceRegistration();
	}
	/*
	1 执行老化刺激的逻辑，每隔PerceptionAgingRate时间会计算一次，默认是0.3s
	2 计算误差值AgingDt，CurrentTime - NextStimuliAgingTick
	3 推进老化刺激的时间，时间为老化间隔PerceptionAgingRate + 误差值，添加误差值的原因是有可能因为掉帧导致帧长变长，所以推进老化时间也需要变长
	4 如果有过期的刺激，bSomeListenersNeedUpdateDueToStimuliAging标志位就为true
	*/
	bool bSomeListenersNeedUpdateDueToStimuliAging = false;
	if (NextStimuliAgingTick <= CurrentTime)
	{
		constexpr double Precision = 1./64.;
		const float AgingDt = FloatCastChecked<float>(
			CurrentTime - NextStimuliAgingTick, Precision);
		bSomeListenersNeedUpdateDueToStimuliAging = AgeStimuli(PerceptionAgingRate + AgingDt);
		NextStimuliAgingTick = CurrentTime + PerceptionAgingRate;
	}
	/*
	1 遍历所有Sense感知频道，Advance频道时间
	2 如果有到时间需要更新Sense了，bNeedsUpdate标志位为true
	3 如果bNeedsUpdate标志位为true，遍历所有的Listener，更新Listenr信息。执行Sense感知频道的tick方法
	*/
	bool bNeedsUpdate = false;
	for (UAISense* const SenseInstance : Senses)
	{
		bNeedsUpdate |= SenseInstance != nullptr && 
			SenseInstance->ProgressTime(DeltaSeconds);
	}
	if (bNeedsUpdate)
	{
		for (AIPerception::FListenerMap::TIterator 
			ListenerIt(ListenerContainer); ListenerIt; ++ListenerIt)
		{
			if (ListenerIt->Value.Listener.IsValid())
			{
				ListenerIt->Value.CacheLocation();
			}
			else
			{
				OnListenerRemoved(ListenerIt->Value);
				ListenerIt.RemoveCurrent();
			}
		}
		for (UAISense* const SenseInstance : Senses)
		{
			if (SenseInstance != nullptr)
			{
				SenseInstance->Tick();
			}
		}
		/*
		1 处理延迟刺激，并返回是否有可以触发的延迟刺激
		2 遍历DelayedStimuli数组，找到到达触发时间的刺激，把刺激添加到PerceptionComp中的StimuliToProcess数组中
		*/
		const bool bStimuliDelivered = DeliverDelayedStimuli(
			bNeedsUpdate ? RequiresSorting : NoNeedToSort);
		/*
		1 如果有Sense需要更新或者有延迟刺激需要触发或者有刺激老化需要更新
		2 遍历所有的Listener，找到Listener的PerceptionComp如果有需要处理的刺激就处理
		*/
		if (bNeedsUpdate || bStimuliDelivered || 
			bSomeListenersNeedUpdateDueToStimuliAging)
		{
			for (AIPerception::FListenerMap::TIterator
				 ListenerIt(ListenerContainer); ListenerIt; ++ListenerIt)
			{
				check(ListenerIt->Value.Listener.IsValid());

				if (ListenerIt->Value.HasAnyNewStimuli())
				{
					ListenerIt->Value.ProcessStimuli();
				}
			}
		}
	}
}
```

# UAIPerceptionComponent
## OnRegister
```cpp
void UAIPerceptionComponent::OnRegister()
{
	/*
	1 根据SensesConfig配置，向PerceptionSystem中注册Sense感知频道，会给每个不同的Sense分配一个SenseID，然后通过NewObject创建出Sense，添加到System中的Senses数组中
	2 会将SenseID按照位的偏移，添加到PerceptionFilter中记录在Int32类型的成员变量中。表示这个PerceptionComp会对哪些Sense做出反应，也就是监听了哪些Sense
	3 然后会像PerceptionSystem中添加ListenerContainer，把PerceptionComp作为Listener添加到ListenerContainer中
	*/
	UAIPerceptionSystem* AIPerceptionSys = 
		UAIPerceptionSystem::GetCurrent(GetWorld());
	if (AIPerceptionSys != nullptr)
	{
		PerceptionFilter.Clear();
		if (SensesConfig.Num() > 0)
		{
			for (auto SenseConfig : SensesConfig)
			{
				if (SenseConfig)
				{
					RegisterSenseConfig(*SenseConfig, *AIPerceptionSys);
				}
			}
			AIPerceptionSys->UpdateListener(*this);
		}
	}
}
```
## ProcessStimuli
```cpp
void UAIPerceptionComponent::ProcessStimuli()
{
	// 没有需要处理的刺激
	if(StimuliToProcess.Num() == 0)
	{
		return;
	}
	// 提前保存是否需要Boradcast
	const bool bBroadcastEveryTargetUpdate = OnTargetPerceptionUpdated.IsBound();
	const bool bBroadcastEveryTargetInfoUpdate = 
		OnTargetPerceptionInfoUpdated.IsBound();
	// 遍历所有的刺激
	for (FStimulusToProcess& SourcedStimulus : ProcessingStimuli)
	{
		const TObjectKey<AActor>& SourceKey = SourcedStimulus.Source;
		// 判断运行时感知数据是否存在，如果不存在就添加一个新的
		FActorPerceptionInfo* PerceptualInfo = PerceptualData.Find(SourceKey);
		AActor* SourceActor = nullptr;
		if (PerceptualInfo == nullptr)
		{
			if (SourcedStimulus.Stimulus.WasSuccessfullySensed() == false)
			{
				continue;
			}
			else
			{
				// 添加新的感知数据
				PerceptualInfo = &PerceptualData.Add(SourceKey, FActorPerceptionInfo(SourceActor));
				PerceptualInfo->DominantSense = DominantSenseID;
				// 标记是否是敌对的
				PerceptualInfo->bIsHostile = (FGenericTeamId::GetAttitude(GetOwner(), SourceActor) == ETeamAttitude::Hostile);
				// 标记是否是友好的
				PerceptualInfo->bIsFriendly = PerceptualInfo->bIsHostile ? false : (FGenericTeamId::GetAttitude(GetOwner(), SourceActor) == ETeamAttitude::Friendly);
			}
			// 把这次的刺激添加到LastSensedStimuli中
			FAIStimulus& StimulusStore = PerceptualInfo
				->LastSensedStimuli[SourcedStimulus.Stimulus.Type];
			if (SourcedStimulus.Stimulus.WasSuccessfullySensed())
			{
				bRequiresUpdate |= ConditionallyStoreSuccessfulStimulus(
					StimulusStore, SourcedStimulus.Stimulus);
			}
			// 如果需要更新，就广播改变
			if (bRequiresUpdate)
			{
				if (bBroadcastEveryTargetUpdate)
				{
					OnTargetPerceptionUpdated.Broadcast(
						SourceActor, StimulusStore);
				}
				if (bBroadcastEveryTargetInfoUpdate)
				{
					OnTargetPerceptionInfoUpdated.Broadcast(
						FActorPerceptionUpdateInfo(
							GetTypeHash(SourceKey), 
							PerceptualInfo->Target, 
							StimulusStore));
				}
			}
			// 处理需要忘记的Actor，之后不在监听刺激了
			for (AActor* ActorToForget : ActorsToForget)
			{
				ForgetActor(ActorToForget);
			}
			// 移除运行时感知数据
			for (const TObjectKey<AActor>& SourceKey : DataToRemove)
			{
				PerceptualData.Remove(SourceKey);
			}
		}
	}
}
```


