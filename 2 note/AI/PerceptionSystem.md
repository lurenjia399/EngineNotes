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
		// 有sense需要更新或者有可以触发的延迟刺激或者
		if (bNeedsUpdate || bStimuliDelivered || 
			bSomeListenersNeedUpdateDueToStimuliAging)
		{
			for (AIPerception::FListenerMap::TIterator ListenerIt(ListenerContainer); ListenerIt; ++ListenerIt)
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


