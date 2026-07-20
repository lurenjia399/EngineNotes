# OnRegister
```cpp
void UAIPerceptionComponent::OnRegister()
{
	// 
	/*
	1 根据SensesConfig配置，向PerceptionSystem中注册Sense，会给每个不同的Sense分配一个SenseID，然后通过NewObject创建出Sense，添加到System中的Senses数组中
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

# OnUnregister
```cpp
void UAIPerceptionComponent::OnUnregister()
{
	CleanUp();
	Super::OnUnregister();
}

void UAIPerceptionComponent::CleanUp()
{
	if (bCleanedUp == false)
	{
		/*
		1 从PerceptionSystem中移除Listener，就是从ListenerContainer数组中移除
		2 获取PerceptionComp的Outer控制的Pawn，对Pawn执行UnregisterSource
		*/
		UAIPerceptionSystem* AIPerceptionSys = 
			UAIPerceptionSystem::GetCurrent(GetWorld());
		if (AIPerceptionSys != nullptr)
		{
			AIPerceptionSys->UnregisterListener(*this);
			AActor* MutableBodyActor = GetMutableBodyActor();
			if (MutableBodyActor)
			{
				AIPerceptionSys->UnregisterSource(*MutableBodyActor);
			}
		}
	}
}
```

# ProcessStimuli
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