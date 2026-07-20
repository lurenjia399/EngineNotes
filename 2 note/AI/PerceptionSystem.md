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
	1 遍历所有Sense，Advance时间，这个时间是最大值FLT_MAX
	2 如果有到时间需要更新Sense了，bNeedsUpdate标志位为true
	3 如果bNeedsUpdate标志位为true，遍历所有的Listener，更新Listenr信息。执行Sense的tick方法
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
		// 有sense需要更新或者有可以触发的延迟刺激或者有Listnner刺激时间到了，就遍历所有的Listener来执行刺激
		/*
		1 如果有Sense需要更新或者有延迟刺激需要触发或者有刺激老化需要更新
		2 
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

# RegisterSource
```cpp
void UAIPerceptionSystem::RegisterSource(AActor& SourceActor)
{
	
}
```
# 2 


