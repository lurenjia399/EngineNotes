# AddNavigationBoundsUpdateRequest
``` cpp
void UNavigationSystemV1::PerformNavigationBoundsUpdate(const TArray<FNavigationBoundsUpdateRequest>& UpdateRequests)
{
	// 1 会根据参数请求的类型，将请求中的NavBounds从RegisteredNavBounds成员变量中添加或者移除
	TArray<FBox> UpdatedAreas;
	for (const FNavigationBoundsUpdateRequest& Request : UpdateRequests)
	{
		switch (Request.UpdateRequest)
		{
		
		}
	}
	// 2 更新了NavBounds，也需要告诉ANavigationData，NavBounds改变了需要更新NavDataGenerator上的相关数据
	if (UpdatedAreas.Num())
	{
		for (ANavigationData* NavData : NavDataSet)
		{
			if (NavData)
			{
				NavData->OnNavigationBoundsChanged();	
			}
		}
	}
}
```

# GetNavigationBoundsForNavData

```cpp
int UNavigationSystemV1::GetNavigationBoundsForNavData(
	const ANavigationData& NavData, // 导航数据
	TArray<FBox>& OutBounds, // 输出符合条件的导航范围
	ULevel* InLevel) const // 只获取特定关卡的导航范围
{
	
}
```
