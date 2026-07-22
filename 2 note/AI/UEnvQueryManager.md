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
	1 CurrentTest < 0 表示没有执行过Test，需要先执行Generate
	2 如果Generator没有正在运行，先清空数据之后运行
	3 
	*/
	if (CurrentTest < 0)
	{
		if (!OptionItem.Generator->IsCurrentlyRunningAsync())
		{
			RawData.Reset();
			Items.Reset();
			ItemType = OptionItem.ItemType;
			bPassOnSingleResult = false;
			ValueSize = (ItemType->GetDefaultObject<UEnvQueryItemType>())->GetValueSize();
		}
		if (bRunGenerator)
		{
			FScopeCycleCounterUObject GeneratorScope(OptionItem.Generator);
			OptionItem.Generator->GenerateItems(*this);
		}
		bIsCurrentlyRunningAsync |= OptionItem.Generator->IsCurrentlyRunningAsync();
		bStepDone = !bIsCurrentlyRunningAsync;

		if (bStepDone)
		{
			FinalizeGeneration();
		}
	}
}
```