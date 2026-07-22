# Tick
```cpp
void UEnvQueryManager::Tick(float DeltaTime)
{
	/*
	
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