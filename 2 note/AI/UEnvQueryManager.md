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
		
	}
}
```