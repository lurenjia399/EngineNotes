# tick
```cpp
void UBehaviorTreeComponent::TickComponent(float DeltaTime, const ELevelTick TickType, FActorComponentTickFunction *ThisTickFunction)
{
	/*
	1 减少下一次Tick执行时间
	*/
	NextTickDeltaTime -= DeltaTime;
	if (NextTickDeltaTime > 0.0f)
	{
		AccumulatedTickDeltaTime += DeltaTime;
		ScheduleNextTick(NextTickDeltaTime);
		return;
	}
	/*
	1 累计tick的间隔，就是距离上次tick经过的时间
	2 每次执行完tick，就清零这个时间
	*/
	AccumulatedTickDeltaTime += DeltaTime;
	ON_SCOPE_EXIT
	{
		AccumulatedTickDeltaTime = 0.0f;
	};
	DeltaTime = AccumulatedTickDeltaTime;
	/*
	1 记录这次tick是否是第一次tick
	*/
	const bool bWasTickedOnce = bTickedOnce;
	bTickedOnce = true;
}
```