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
	3 通过GenerateItems真正生成候选点
	4 如果生成候选点不是异步的过程，就直接调用FinalizeGeneration完成生成，生成完了做收尾(比如去重、建索引)
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
	/*
	1 已经执行过Generate了，需要执行test
	2 如果这是最后一个 Test、查询模式是
  SingleResult(只要一个最优点)、且这个 Test 能当"最终条件",那么就不用给所有点算完再选——边测边短路。而且在开始前,如果前面有打分性质的Test,会先 SortScores() 排序,让高分点先被测,进一步加速短路。- Done 判断:满足"不在异步 + (所有 Item 测完 / 找到单结果 / 没有新进展)"才算这个 Test 完成,然后FinalizeTest()(应用过滤、归一化分数等)
	*/
	else if (OptionItem.Tests.IsValidIndex(CurrentTest))
	{
		{
			FScopeCycleCounterUObject TestScope(TestObject);
			TestObject->RunTest(*this);
		}
	}
	/*
	1 如果这一步都执行完了，没有任何异步的过程，就推进到下一个Test
	*/
	if (bStepDone)
	{
		CurrentTest++;
		CurrentTestStartingItem = 0;
	}
	/*
	1 
	*/
	if (!bIsCurrentlyRunningAsync && IsFinished() == false && (OptionItem.Tests.Num() == CurrentTest || NumValidItems <= 0))
	{
		if (NumValidItems > 0)
		{
			// found items, sort and finish
			FinalizeQuery();
		}
		else
		{
			// no items here, go to next option or finish			
			if (OptionIndex + 1 >= Options.Num())
			{
				// out of options, finish processing without errors
				FinalizeQuery();
			}
			else
			{
				OptionIndex++;
				CurrentTest = -1;
			}
		}
	}
}
```