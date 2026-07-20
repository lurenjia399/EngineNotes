# Tick

```cpp
void UAIPerceptionSystem::Tick(float DeltaSeconds)
{
	CurrentTime = World->GetTimeSeconds();
	// 有注册的感知原，就先处理
	if (SourcesToRegister.Num() > 0)
	{
		PerformSourceRegistration();
	}
	/*
	1 执行老化刺激的逻辑，每隔PerceptionAgingRate时间会计算一次，默认是0.3s
	2 计算误差值AgingDt，CurrentTime - NextStimuliAgingTick
	3 推进老化刺激的时间，时间为老化间隔PerceptionAgingRate + 误差值，添加误差值的原因是有可能因为掉帧导致帧长变长，
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
	// 遍历所有Sense，Advance时间
	bool bNeedsUpdate = false;
	for (UAISense* const SenseInstance : Senses)
	{
		bNeedsUpdate |= SenseInstance != nullptr && 
			SenseInstance->ProgressTime(DeltaSeconds);
	}
	// 如果有需要更新的Sense
	if (bNeedsUpdate)
	{
		// 遍历所有的Listener，就是PerceptionComp缓存位置信息，如果没用了就移除掉
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
		// 遍历所有的Sense，执行Sense的Tick
		for (UAISense* const SenseInstance : Senses)
		{
			if (SenseInstance != nullptr)
			{
				SenseInstance->Tick();
			}
		}
		// 处理延迟刺激，并返回是否有可以触发的延迟刺激
		const bool bStimuliDelivered = DeliverDelayedStimuli(
			bNeedsUpdate ? RequiresSorting : NoNeedToSort);
		// 有sense需要更新或者有可以触发的延迟刺激或者有Listnner刺激时间到了，就遍历所有的Listener来执行刺激
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

# RegisterSource
```cpp
void UAIPerceptionSystem::RegisterSource(AActor& SourceActor)
{
	
}
```
# 2 


