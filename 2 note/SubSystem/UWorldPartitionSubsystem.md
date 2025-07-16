https://zhuanlan.zhihu.com/p/158717151 基本概念
我们wp的SubSystem来举例。

# WorldPartitionSubsystem的Initialize

```cpp
class ENGINE_API UWorld final : public UObject, public FNetworkNotify
{
	FObjectSubsystemCollection<UWorldSubsystem> SubsystemCollection;
}
```
1 在world中会有成员变量SubsystemCollection，我们自己创建的WorldPartitionSubsystem在创建之后就会添加到这个Collection里。
```cpp
void UWorld::InitWorld(...)
 ->void UWorld::InitializeSubsystems()
  ->void FSubsystemCollectionBase::Initialize(UObject* NewOuter)//参数就是world
   ->USubsystem* FSubsystemCollectionBase::AddAndInitializeSubsystem(...)//将我们UWorldPartitionSubsystem添加到world的SubsystemCollection中了
```
2 添加的流程就像这样，走到FSubsystemCollectionBase::Initialize方法中，然后Add到Collection里。
```cpp
void FSubsystemCollectionBase::Initialize(UObject* NewOuter)
{
	// SubSystem的生命周期跟随Outer，而这里的Outer就是World
	Outer = NewOuter;
	// BaseType是我们SubSystem的类型，也就是UWorldSubsystem
	// SubsystemMap就是Collection中的存储结构，存放我们的UWorldSubsystem
	if (ensure(BaseType) && ensureMsgf(SubsystemMap.Num() == 0)
	{
		TArray<UClass*> SubsystemClasses;
		// 直接递归遍历找BaseType的子类
		GetDerivedClasses(BaseType, SubsystemClasses, true);
		// 把找到SubSystemClass都创建成Object，然后都Add进来
		for (UClass* SubsystemClass : SubsystemClasses)
		{
			AddAndInitializeSubsystem(SubsystemClass);
		}
	}
}
USubsystem* FSubsystemCollectionBase::AddAndInitializeSubsystem(UClass* SubsystemClass)
{
	if (!SubsystemMap.Contains(SubsystemClass))
	{
		const USubsystem* CDO = SubsystemClass->GetDefaultObject<USubsystem>();
		if (CDO->ShouldCreateSubsystem(Outer))
		{
			USubsystem* Subsystem = NewObject<USubsystem>(Outer, SubsystemClass);
			SubsystemMap.Add(SubsystemClass,Subsystem);
			Subsystem->InternalOwningSubsystem = this;
			Subsystem->Initialize(*this);
		}
	}
}
```
3 主要通过寻找继承自UWorldSubsystem的子类，然后根据子类通过newObject创建出SubSystem对象，并添加到SubsystemMap数组中。最后还会调用Subsystem的Initialize方法。
# WorldPartitionSubsystem的Deinitialize
```cpp
void UWorld::CleanupWorldInternal(...)
{
	// 在清掉world里的时候，就会执行Collection的Deinitialize
	if (bCleanupResources)
	{
		SubsystemCollection.Deinitialize();
	}
}
```
1 在清掉world里的时候，就会执行Collection的Deinitialize
```cpp
void FSubsystemCollectionBase::Deinitialize()
{
	// 遍历我们的map,对每个元素都执行Deinitialize
	for (auto Iter = SubsystemMap.CreateIterator(); Iter; ++Iter)
	{
		UClass* KeyClass = Iter.Key();
		USubsystem* Subsystem = Iter.Value();
		if (Subsystem != nullptr && Subsystem->GetClass() == KeyClass)
		{
			Subsystem->Deinitialize();
			Subsystem->InternalOwningSubsystem = nullptr;
		}
	}
	SubsystemMap.Empty();
	Outer = nullptr;
}
```
1 遍历我们的SubsystemMap,对每个元素都执行Deinitialize