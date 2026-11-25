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
		// 构造Task并且尝试执行。返回值FGraphEventRef就是增加了GraphTask的引用计数
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
2 外部我们使用，就是创建GraphTask然后TryLunch尝试执行，在先决Task都执行结束后，才会执行我们创建的GraphTask的TryLunch方法。
3 在任何线程都能创建GraphTask。
4 GraphTask在哪个线程执行，取决于GetDesiredThread方法的返回值
5 NamedTask只能在NamedThread上执行，不会被其他工作线程偷出来执行
6 

# ConstructAndHoldImpl
```cpp
template<typename...T>
TGraphTask* ConstructAndHoldImpl(T&&... Args)
{
	// 根据先决Task，new出GraphTask
	TGraphTask* Task = new TGraphTask(Prerequisites);
	// new出Task执行对象，就是我们自己定义的任务对象
	TTask* TaskObject = new(&Task->TaskStorage) TTask(Forward<T>(Args)...);

	// 根据自己定义的GetDesiredThread，转换出实际的TaskPriority和ExtendedTaskPriority
	UE::Tasks::ETaskPriority Pri;
	UE::Tasks::EExtendedTaskPriority ExtPri;
	UE::Tasks::Private::TranslatePriority(TaskObject->GetDesiredThread(), 
		Pri, ExtPri);
	
	// 初始化GraphTask
	Task->Init(Pri, ExtPri);

	return Task;
}
```
1 创建出GraphTask，GraphTask是继承FBaseGraphTask，FBaseGraphTask是继承UE::Tasks::Private::FTaskBase的。

# DontCompleteUntil
```cpp
void DontCompleteUntil(FGraphEventRef NestedTask)
{
	if (!NestedTask)
	{
		return;
	}

	// 如果是TaskEvent，说明是个空实现，占位用的，这里直接给空Task添加先决Task
	if (IsTaskEvent())
	{
		AddPrerequisites(*NestedTask);
	}
	// 如果不是TaskEvent，说明是普通Task，不能直接添加先决Task，因为执行的时候不能建立依赖。这里直接添加嵌套Task。
	else
	{
		// 只能给当前正在执行的Task添加嵌套任务
		checkSlow(UE::Tasks::Private::GetCurrentTask() == this);
		AddNested(*NestedTask);
	}
}
```