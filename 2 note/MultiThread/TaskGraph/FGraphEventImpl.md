```cpp
// 静态方法创建出GraphEvent
FGraphEventRef FBaseGraphTask::CreateGraphEvent()
{
	FGraphEventImpl* GraphEvent = new FGraphEventImpl;
	return FGraphEventRef{ GraphEvent, false };
}
```

```cpp
class FGraphEventImpl : public FBaseGraphTask
{
public:
	FGraphEventImpl()
		: FBaseGraphTask(nullptr)
	{
		Init(TEXT("GraphEvent"), UE::Tasks::ETaskPriority::Normal, UE::Tasks::EExtendedTaskPriority::TaskEvent);
	}

	static void* operator new(size_t Size);
	static void operator delete(void* Ptr);

private:
	virtual void ExecuteTask() override final
	{
		// 空实现，不能有东西
		checkNoEntry();
	}
};
```

1 这个就是在TaskBase里面说到的TaskEvent，是一种空的Task，没有任何的实现，只用来占位。
2 Task之间的依赖关系是在执行前建立好的，但是在执行中创建了新的Task，就可能用到。比如
A->B，A执行完会执行B，在A执行的过程中创建了A2，想让A2执行完，B才能执行。这时可以一开始建立A->TaskEvent->B这样的依赖关系，然后让TaskEvent依赖A2，这样就能保证A和A2执行完才会执行B了。