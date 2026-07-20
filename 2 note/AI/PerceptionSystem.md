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
	// 遍历所有Sense，
	bool bNeedsUpdate = false;
	for (UAISense* const SenseInstance : Senses)
	{
		bNeedsUpdate |= SenseInstance != nullptr && 
			SenseInstance->ProgressTime(DeltaSeconds);
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


