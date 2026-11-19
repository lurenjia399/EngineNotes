```cpp
class TGraphTask final 
: public TConcurrentLinearObject<TGraphTask<TTask>,FTaskGraphBlockAllocationTag>,
  public FBaseGraphTask
{
	// 外部创建Task的方法，第一个参数是先决Task,第二个参数没用
	static FConstructor CreateTask(
		const FGraphEventArray* Prerequisites = nullptr, 
		ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
	{
		return FConstructor(Prerequisites);
	}
	
	class FConstructor
	{
		// 构造Task并且尝试执行
		template<typename...T>
		FGraphEventRef ConstructAndDispatchWhenReady(T&&... Args)
		{
			FGraphEventRef Ref{ ConstructAndHoldImpl(Forward<T>(Args)...) };
			Ref->TryLaunch(sizeof(TGraphTask));
			return Ref;
		}
		
		// 构造Task但是不尝试执行
		template<typename...T>
		TGraphTask* ConstructAndHold(T&&... Args)
		{
			TGraphTask* Task = ConstructAndHoldImpl(Forward<T>(Args)...);
			return Task;
		}
	}
}
```
1 FGraphTask是继承Task的类
2 外部我们使用，就是创建GraphTask然后TryLunch尝试执行，在先决Task都执行结束后，会把Task传到FScheduler系统中，被放入到工作队列里面，等工作线程取任务执行。