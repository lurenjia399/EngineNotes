# Tick
```cpp
void UEnvQueryManager::Tick(float DeltaTime)
{
	/*
	1 QueryInstance 就是一个QueryTempate创建的实例，
	*/
	TSharedPtr<FEnvQueryInstance> QueryInstance = RunningQueries[Index];
	FEnvQueryInstance* QueryInstancePtr = QueryInstance.Get();
	if (QueryInstancePtr == nullptr || QueryInstancePtr->IsFinished())
	{
		++Index;
	}
	else
	{
		QueryInstancePtr->ExecuteOneStep(TimeLeft);
	}
}
```

#
```cpp
void FEnvQueryInstance::ExecuteOneStep(double TimeLimit)
{
	/*
	1 
	*/
	if (CurrentTest < 0)
	{
		if (!OptionItem.Generator->IsCurrentlyRunningAsync())
		{
			SCOPE_CYCLE_COUNTER(STAT_AI_EQS_GeneratorTime);
			DEC_DWORD_STAT_BY(STAT_AI_EQS_NumItems, Items.Num());

			RawData.Reset();
			Items.Reset();
			ItemType = OptionItem.ItemType;
			bPassOnSingleResult = false;
			ValueSize = (ItemType->GetDefaultObject<UEnvQueryItemType>())->GetValueSize();
		}
	}
}
```