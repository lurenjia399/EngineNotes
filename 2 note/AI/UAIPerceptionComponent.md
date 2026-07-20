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
				SourceActor = CastChecked<AActor>(SourceKey.ResolveObjectPtr(), ECastCheckedType::NullAllowed);
				if (SourceActor == nullptr)
				{
					continue;
				}
				PerceptualInfo = &PerceptualData.Add(SourceKey, FActorPerceptionInfo(SourceActor));
				PerceptualInfo->DominantSense = DominantSenseID;
				PerceptualInfo->bIsHostile = (FGenericTeamId::GetAttitude(GetOwner(), SourceActor) == ETeamAttitude::Hostile);
				PerceptualInfo->bIsFriendly = PerceptualInfo->bIsHostile ? false : (FGenericTeamId::GetAttitude(GetOwner(), SourceActor) == ETeamAttitude::Friendly);
			}
		}
	}
}
```