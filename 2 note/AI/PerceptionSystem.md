# UAIPerceptionSystem
## Tick
```cpp
void UAIPerceptionSystem::Tick(float DeltaSeconds)
{
	/*
	1 有需要注册的感知源就先处理
	2 给相应的感知频道注册感知源
	3 注册相应的刺激来源
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
## OnNewPawn
```cpp
/*
1 在pawn创建的时候会遍历感知频道，如果有符合要求的就会向感知频道中添加感知源
2 UAISense_Sight 视觉是感知所有pawn的，一般bAutoRegisterAllPawnsAsSources都是true，所以会把pawn注册到视觉感知频道中
*/
void UAIPerceptionSystem::OnNewPawn(APawn& Pawn)
{
	if (bHandlePawnNotification == false)
	{
		return;
	}

	for (UAISense* Sense : Senses)
	{
		if (Sense == nullptr)
		{
			continue;
		}

		if (Sense->WantsNewPawnNotification())
		{
			Sense->OnNewPawn(Pawn);
		}

		if (Sense->ShouldAutoRegisterAllPawnsAsSources())
		{
			FAISenseID SenseID = Sense->GetSenseID();
			RegisterSource(SenseID, Pawn);
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

# UAISense_Sight

## RegisterTarget
```cpp
bool UAISense_Sight::RegisterTarget(
	AActor& TargetActor, 
	const TFunction<void(FAISightQuery&)>& OnAddedFunc)
{
	/*
	1 向视觉频道中GetOrAdd，获取SightTarget视觉目标，补充视觉目标的信息
	*/
	FAISightTarget* SightTarget = ObservedTargets.Find(TargetActor.GetUniqueID());
	if (SightTarget == nullptr || SightTarget->GetTargetActor() != &TargetActor)
	{
		FAISightTarget NewSightTarget(&TargetActor);
		SightTarget = &(ObservedTargets.
			Add(NewSightTarget.TargetId, NewSightTarget));
		if (IAISightTargetInterface* InterfaceComponent = TargetActor.
			FindComponentByInterface<IAISightTargetInterface>())
		{
			SightTarget->WeakSightTargetInterface = InterfaceComponent;
		}
		else 
		{
			SightTarget->WeakSightTargetInterface = 
				Cast<IAISightTargetInterface>(&TargetActor);
		}
	}
	SightTarget->TeamId = FGenericTeamId::GetTeamIdentifier(&TargetActor);
	/*
	1 遍历所有的Listener，找到具有听觉的Listener
	2 给Listener注册一个SightQuery，这个视觉查询的OserverId是listener，targetId是感知源Actor,Importance就是[10,60]，沿着listener距离target线性增加，越靠近listener越大
	3 如果新添加了视觉查询，就立即清空下次更新时间，下一次tick执行Update方法
	4 每一个感知源对所有Listener都会创建一个Query
	*/
	bool bNewQueriesAdded = false;
	AIPerception::FListenerMap& ListenersMap = *GetListeners();
	const FVector TargetLocation = TargetActor.GetActorLocation();
	for (AIPerception::FListenerMap::TConstIterator ItListener(ListenersMap); ItListener; ++ItListener)
	{
		const FPerceptionListener& Listener = ItListener->Value;
		if (!Listener.HasSense(GetSenseID()) || Listener.GetBodyActor() == &TargetActor)
		{
			continue;
		}
		const FDigestedSightProperties& PropDigest = DigestedProperties[Listener.GetListenerID()];
		const IGenericTeamAgentInterface* ListenersTeamAgent = Listener.GetTeamAgent();
		if (RegisterNewQuery(Listener, ListenersTeamAgent, TargetActor, SightTarget->TargetId, TargetLocation, PropDigest, OnAddedFunc))
		{
			bNewQueriesAdded = true;
		}
	}
}
```
## Update
```cpp
float UAISense_Sight::Update()
{
	/*
	1 视野内的Query排序，根据Importance和上次处理时间计算得分，越大越先处理
	*/
	ForEach(SightQueriesInRange, RecalcScore);
	SightQueriesInRange.Sort(FAISightQuery::FSortPredicate());
	/*
	1 遍历在视野内的Query和视野外的Query
	*/
	for (int32 QueryIndex = 0; QueryIndex < 
		SightQueriesInRange.Num() + SightQueriesOutOfRange.Num(); ++QueryIndex)
	{
		/*
		1 如果处理40个Query超过0.05s就退出遍历。如果Trace次数超过6次就退出遍历。如果异步trace超过10次就退出遍历。
		*/
		NumQueriesProcessed++;
		if ((NumQueriesProcessed % MinQueriesPerTimeSliceCheck) == 0 
			&& FPlatformTime::Seconds() > TimeSliceEnd)
		{
			bHitTimeSliceLimit = true;
		}
		if (bHitTimeSliceLimit || TracesCount >= MaxTracesPerTick 
			|| AsyncTracesCount >= MaxAsyncTracesPerTick)
		{
			break;
		}
		/*
		1 计算Visibility
		*/
		if (TargetActor && ListenerPtr)
		{
			const EVisibilityResult VisibilityResult = 
				ComputeVisibility(
					World, //当前World
					*SightQuery, // Query
					Listener, // PerceptionComp所代表的Listener
					ListenerBodyActor, // PerCeptionComp的Outer上PC控制的Pawn
					Target, //SightTarget,感知源构成的目标带有阵营等其他信息
					TargetActor, // 感知源，其他Pawn
					PropDigest, // 配置的视觉参数
					StimulusStrength, // 传出来的数据，刺激强度
					SeenLocation, 
					NumberOfLoSChecksPerformed, 
					NumberOfAsyncLosCheckRequested);
		}
	}
}
```
## ComputeVisibility
```cpp
UAISense_Sight::EVisibilityResult UAISense_Sight::ComputeVisibility(...) const
{
	/*
	1 判断是否需要直接算作看到目标，如果上次看到位置满足配置的距离判断，就返回Visible
	*/
	if (ShouldAutomaticallySeeTarget(PropDigest, &SightQuery, Listener, TargetActor, OutStimulusStrength))
	{
		OutSeenLocation = FAISystem::InvalidLocation;
		return EVisibilityResult::Visible;
	}
	/*
	1 
	*/
	const FVector TargetLocation = TargetActor->GetActorLocation();
	const float SightRadiusSq = SightQuery.GetLastResult() ? PropDigest.LoseSightRadiusSq : PropDigest.SightRadiusSq;
	if (!FAISystem::CheckIsTargetInSightCone(Listener.CachedLocation, Listener.CachedDirection, PropDigest.PeripheralVisionAngleCos, PropDigest.PointOfViewBackwardOffset, PropDigest.NearClippingRadiusSq, SightRadiusSq, TargetLocation))
	{
		return EVisibilityResult::NotVisible;
	}
}
```