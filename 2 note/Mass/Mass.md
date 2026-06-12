UE5的ECS：MASS框架(一)
https://zhuanlan.zhihu.com/p/441773595
mass框架
https://www.xianlongok.site/post/ea92a01c/#UE5-Mass-%E6%A1%86%E6%9E%B6%E4%BB%8B%E7%BB%8D

# 1 结构
## 1 FMassEntityHandle
Entity就相当于是 FMassEntityHandle，其中FMassEntityHandle的结构如下：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250716234618.png)
其中只有两个成员变量，显而易见的就是在一个大数组里存储的索引和一个唯一标识，和FWeakObjectPtr原理是一样的。那么这个大数组存储在哪呢？下面这张图就是创建 FMassEntityHandle 的方法。
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250716235838.png)
图中可以看到我们第一次走到这个Acquire方法会执行AddPage来添加页，也就是分配一大块内存（内存的大小 = sizeof(FEntityData) * CountPrePage(1 << 16)）,这个页就是一个EntityData数组，所有页中的FEntityData索引都是递增的，这个索引就是EntityHandle中的index。而我们的大数组呢，就是所有页的集合，页的申请内存是Acquire方法，自然的页的指针会保存到FConcurrentEntityStorage这个结构中，而FConcurrentEntityStorage这个结构是MassEntityManager里面的 EntityStorage 这个成员变量。==综上所述，大数组的指针保存在 MassEntityManager 的 EntityStorage 成员变量中，MassEntityHandle 中的index指向大数组中的EntityData。==

## 2 FMassEntityTemplate

FMassEntityTemplate的创建是由UMassEntityConfigAsset这种配置文件创建出来的，如：
```cpp
const FMassEntityTemplate& EntityTemplate = 
			MassEntityConfig->GetConfig()
				.GetOrCreateEntityTemplate(*WorldContextObject->GetWorld());
```
这种配置文件的整体结构如下：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250717234028.png)
GetOrCreateEntityTemplate这个方法我们来看下：
```cpp
const FMassEntityTemplate& FMassEntityConfig::GetOrCreateEntityTemplate(const UWorld& World) const
{
	// 0 创建出TemplateID，就是FGuid
	FMassEntityTemplateID TemplateID;
	if (const FMassEntityTemplate* ExistingTemplate = GetEntityTemplateInternal(World, TemplateID))
	{
		return *ExistingTemplate;
	}
	// 1 拿到EntityTemplateRegistry，一个注册器
	UMassSpawnerSubsystem* SpawnerSystem = UWorld::GetSubsystem<UMassSpawnerSubsystem>(&World);
	check(SpawnerSystem);
	FMassEntityTemplateRegistry& TemplateRegistry = SpawnerSystem->GetMutableTemplateRegistryInstance();

	//2 构建出BuildContext，帮助我们创建EntityTemplate的上下文
	FMassEntityTemplateData TemplateData;
	FMassEntityTemplateBuildContext BuildContext(TemplateData, TemplateID);
	//3 遍历配置文件中的Trait，包括自己的和自己Parent的，递归遍历
	TArray<UMassEntityTraitBase*> CombinedTraits;
	GetCombinedTraits(CombinedTraits);
	//4 通过BuildContext来Build我们的Trait,也就是执行Trait的BuildTemplate纯虚函数。
	BuildContext.BuildFromTraits(CombinedTraits, World);
	//5 设置EntityTemplate的名称为配置文件的名称
	BuildContext.SetTemplateName(GetNameSafe(ConfigOwner));
	//6 执行完2345后TemplateData中的数据就填充完毕了，最后一步是通过FMassEntityTemplate构造函数创建EntityTemplate并将其添加到注册器里
	return TemplateRegistry.FindOrAddTemplate(TemplateID, MoveTemp(TemplateData)).Get();
}
```
EntityTemplate怎么创建的呢？我们从FMassEntityTemplate类中能看到里面只有三个成员变量，其中TemplateData和TemplateID我们都已经创建出来了，只剩下Archetype了，而Archetype的创建是在构造函数中。那我们来看他的构造函数：
```cpp
FMassEntityTemplate::FMassEntityTemplate(const FMassEntityTemplateData& InData, FMassEntityManager& EntityManager, FMassEntityTemplateID InTemplateID)
	: TemplateData(InData)
	, TemplateID(InTemplateID)
{
	//1 改变下TemplateData这个里面的数据
	TemplateData.Sort();
	TemplateData.GetArchetypeCreationParams().DebugName = FName(GetTemplateName());
	//2 通过EntityManager的CreateArchetype方法创建Archetype
	const FMassArchetypeHandle ArchetypeHandle = EntityManager.CreateArchetype(GetCompositionDescriptor(), TemplateData.GetArchetypeCreationParams());
	//3 将Archetype赋值到EntityTemplate里面
	SetArchetype(ArchetypeHandle);
}

// Archetype的创建方法
FMassArchetypeHandle FMassEntityManager::CreateArchetype(
	const FMassArchetypeCompositionDescriptor& Composition, 
	const FMassArchetypeCreationParams& CreationParams)
{
		// 具体的创建只有这两行，一个是new一个是Initialize，这两部分就是初始化Archetype了，我们在介绍Archetype时候再看。其他我省略的内容都是填充缓存数组的，比如：FragmentHashToArchetypeMap（通过hash值索引到Archetype）,AllArchetypes（记录游戏中所有创建出的Archetype）,FragmentTypeToArchetypeMap（通过FragmentType索引到Archetype）
	FMassArchetypeData* NewArchetype = new FMassArchetypeData(CreationParams);
	NewArchetype->Initialize(*this, Composition, ArchetypeDataVersion);
}
```
## 3 Archetype
FMassEntityTemplate 是用来存放 Archetype 的，那什么是 Archetype 呢？类是对象的原型，UObject的原型是CDO，那么Entity的原型就是Archetype，可以这么理解吧。回过头来，现在我们看看Archetype的组成部分：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250718213137.png)
我们可以看到有很多的成员变量。我们来看看每个成员变量的含义：
```cpp
/*
	1 FMassArchetypeCompositionDescriptor CompositionDescriptor;//原型组成描述符，由五个Fragments, Tags, ChunkFragments, SharedFragments, ConstSharedFragments组成。在Trait创建的时候会生成相应的Fragments，并赋值给EntityTemplate的CompositionDescriptor, 然后在Archetype::Initialize方法中传递给Archetype的CompositionDescriptor成员变量。
	2 uint32 CreatedArchetypeDataVersion = 0;// 就是一个计数器，每次我们创建新的Archetype的时候，这个计数器+1，并且这个值一旦设置就不会被更改。
	3 TArray<FMassArchetypeFragmentConfig, TInlineAllocator<16>> FragmentConfigs;//这个看上去就是一个缓存用于加快索引。是一个数组，这个数组的key是Fragments的索引，value是Fragment的type和ArrayOffsetWithinChunk组成的FMassArchetypeFragmentConfig（ArrayOffsetWithinChunk指的是在Chunk中的偏移，Chunk是一块很大的内存，它里面保存了所有Entity的EntityHanle和Entity用到的所有的Fragment。Chunk的布局是所有的EntityHanle占一部分，所有相同的Fragment占一部分）。所以我们要想获取EntityA用到的FragmentB，就需要知道FragmentB在Chunk的偏移ArrayOffsetWithinChunk，有了这个偏移就能找到所有FragmentB,还需要知道EntityA在Chunk中是第几个，才能找到到EntityA用的FragmentB。
	4 TMap<const UScriptStruct*, int32> FragmentIndexMap;//这个也是加快索引的，和FragmentConfigs搭配使用，key和value是FragmentConfigs的key和value反过来。
	5 SIZE_T TotalBytesPerEntity = 0;//这个记录了EntityHandle的大小 + 所有不同Fragment的大小。字面意思上就是每个Entity所占的字节。
	6 const SIZE_T ChunkMemorySize = 0;//Chunk在内存中所占的大小。目前是代码写死的，是128 * 1024，也就是128KB。
	7 int32 NumEntitiesPerChunk;//字面意思就是Chunk能容纳多少个Entity。Chunk的结构是所有的EntityHangle放在一起，所有相同的Fragment放在一起。不同的Fragment之间对其大小可能不同，所以还需要字节对齐，就有可能出现占位的情况，但是呢占位的值不会大于对齐值（按四字节对齐，那占位的值不可能大于四字节）所以需要计算下所有不同Fragment的对齐值和，所以 
	NumEntitiesPerChunk = （ChunkMemorySize - 对齐值和）/ TotalBytesPerEntity
	注意：这是Chunk的计算方式，具体Chunk的内存结构是所有EnittyHangle存放一起，相同的Fragment存放在一起。
	8 TArray<FInstancedStruct> ChunkFragmentsTemplate;//这个就是ChunkFragments里的数据，按照Fragment从大到小排序后，赋值到ChunkFragmentsTemplate成员变量里。
	9 int32 EntityListOffsetWithinChunk;//这个属性在初始化的时候为0。正常来说会在Chunk的开头存放所有的EntityHanle，但是这个属性就是一个偏移，可以把所有的EntityHandle不放在开头放在这个偏移的后边。
//**************以上9个都在 FMassArchetypeData::Initialize 方法中赋值***********
	10 TArray<FMassArchetypeChunk> Chunks;//这个就是Archetype里的Chunk数组，会在生成Entity的时候创建一个Chunk。
	11 TMap<int32, int32> EntityMap;//缓存个索引，key是EntityHandle的index，value是Archetype中Entity的绝对索引。
//**************以上2个都在生成 Entity 的里面赋值***********
	12 uint32 EntityOrderVersion = 0;//当Archetype中的Entity数据顺序改变的时候会增加。
	13 UE::Mass::FArchetypeGroups Groups;//还不知道干嘛的，看上去是分组了？
*/

```

# 4 SpawnEntity
```cpp
TSharedPtr<FMassEntityManager::FEntityCreationContext> 
UMassSpawnerSubsystem::DoSpawning(
	const FMassEntityTemplate& EntityTemplate, 
	const int32 NumToSpawn, 
	FConstStructView SpawnData, 
	TSubclassOf<UMassProcessor> InitializerClass, 
	TArray<FMassEntityHandle>& OutEntities)
{
	// 向Archetype中的Chunk数组添加Entity数据
	// 1. Create required number of entities with EntityTemplate.Archetype
	TArray<FMassEntityHandle> SpawnedEntities;
	TSharedRef<FMassEntityManager::FEntityCreationContext> CreationContext
		= EntityManager->BatchCreateEntities(EntityTemplate.GetArchetype(), EntityTemplate.GetSharedFragmentValues(), NumToSpawn, SpawnedEntities);

	// 将Template中的InitialFragment复制到Archetype中
	// 2. Copy data from FMassEntityTemplate.Fragments.
	//		a. @todo, could be done as part of creation?
	TConstArrayView<FInstancedStruct> FragmentInstances = EntityTemplate.GetInitialFragmentValues();
	EntityManager->BatchSetEntityFragmentValues(CreationContext->GetEntityCollections(*EntityManager.Get()), FragmentInstances);
	
	// 执行初始化的Processor
	// 3. Run SpawnDataInitializer, if set. This is a special type of processor that operates on the entities to initialize them.
	// e.g., will run UInstancedActorsInitializerProcessor for Mass InstancedActors
	UMassProcessor* SpawnDataInitializer = SpawnData.IsValid() 
		? GetSpawnDataInitializer(InitializerClass) 
		: nullptr;

	if (SpawnDataInitializer)
	{
		FMassProcessingContext ProcessingContext(EntityManager, /*TimeDelta=*/0.0f);
		ProcessingContext.AuxData = SpawnData;
		UE::Mass::Executor::RunProcessorsView(MakeArrayView(&SpawnDataInitializer, 1), ProcessingContext, CreationContext->GetEntityCollections(*EntityManager.Get()));
	}

	OutEntities.Append(MoveTemp(SpawnedEntities));

	// 4. "OnEntitiesCreated" notifies will be sent out once the CreationContext gets destroyed (via its destructor).
	// The caller can postpone this moment keeping the returned CreationContext alive as long as needed.

	return CreationContext;
}
```

## 4 UMassEntitySettings
在引擎启动的时候就会执行 BuildProcessorListAndPhases 方法：
```cpp
void UMassEntitySettings::OnPostEngineInit()
{
	bEngineInitialized = true;
	BuildProcessorListAndPhases();
}
```
BuildProcessorListAndPhases 这个方法就是核心构建的方法：
```cpp
void UMassEntitySettings::BuildProcessorListAndPhases()
{
	if (bInitialized == true || bEngineInitialized == false)
	{
		return;
	}
	//1 构建ProcessorList，会赋值 ProcessorCDOs 这个成员变量，这个变量里就是所有的Processor。其中如果是 bAutoRegisterWithProcessingPhases 自动注册的Processor，那么也会添加到 ProcessingPhasesConfig 这个里面，这些都是配置文件能从项目配置中看到。
	BuildProcessorList();
	//2 编辑器模式下，会根据List创建出Processor对象
	BuildPhases();
	bInitialized = true;

	OnInitializedEvent.Broadcast();
}
```

## 5 UMassCompositeProcessor
## 6 FMassEntityQuery
这是个加速结构，我们通过不同的Fragment组成Chunk之后，我们需要操作
给这个Fragment赋值来实现具体的逻辑，这肯定不能遍历吧，所以就有了 FMassFragmentBitSet 这个结构来加速查询，我们从查询开始看：
```cpp
void FMassEntityQuery::ForEachEntityChunk(
			FMassExecutionContext& ExecutionContext, 
			const FMassExecuteFunction& ExecuteFunction)
{
	//1 
	FScopedEntityQueryContext ScopedQueryContext(
												*this, ExecutionContext);
	//
	
}
```

# 2 执行
## 1 MassProcessor
EProcessorExecutionFlags::Editor 这个枚举标志了此个Processor在哪个端执行的?

只要 Processor中的 bAutoRegisterWithProcessingPhases 这个成员变量设置成true（无论是在构造函数中设置还是.ini文件中设置），他就会在UMassEntitySettings 这个类的 OnPostEngineInit 方法中将此 Processor 注册到全局，也就是可以Tick了。
### 1.1 Processor的Execute怎么执行的
FMassExecutorDoneTask 这个是启动的task吧。
编辑器模式下，tick入口是 UMassEntityEditorSubsystem 中的 Tick方法：
```cpp
void UMassEntityEditorSubsystem::Tick(float DeltaTime)
{
	FGraphEventRef CompletionEvent;
	//1 for循环，遍历的是 EMassProcessingPhase,这就是个分组，我们的Processor可以在不同的Tick分组里执行。遍历的内容就是会每个分组都创建一个  FMassEditorPhaseTickTask，后边的依赖前边的，最后一个 FMassEditorPhaseTickTask赋值给 CompletionEvent。
	for (int PhaseIndex = 0; 
		PhaseIndex < (int)EMassProcessingPhase::MAX;
							++PhaseIndex)
	{
		const FGraphEventArray Prerequisites = { CompletionEvent };
		CompletionEvent = TGraphTask<UE::Mass::FMassEditorPhaseTickTask>::CreateTask(
			&Prerequisites).ConstructAndDispatchWhenReady(
PhaseManager, EMassProcessingPhase(PhaseIndex), DeltaTime);
	}
	//2 CompletionEvent 这个Task有效，我们就等待，一直等到Task执行完
	if (CompletionEvent.IsValid())
	{
		CompletionEvent->Wait();
	}
}
```
看了上边的部分，就是给 EMassProcessingPhase 枚举中的值都创建 FMassEditorPhaseTickTask 这个Task并Dispatch，并互相依赖，然后Wait一直等待所有的Task执行完。
所以呢咱们来看下每个Phase的 FMassEditorPhaseTickTask ：
```cpp
struct FMassEditorPhaseTickTask  
{
	// DoTask中就执行了PhaseManager::TriggerPhase方法
	void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
	{
		PhaseManager->TriggerPhase(Phase, DeltaTime, MyCompletionGraphEvent);
	}
}

const FGraphEventRef& 
	FMassProcessingPhaseManager::TriggerPhase(...)
{
	if (bIsAllowedToTick)
	{
		ProcessingPhases[(int)Phase].ExecuteTick(DeltaTime, LEVELTICK_All, CurrentThread, MyCompletionGraphEvent);
	}
	return MyCompletionGraphEvent;
}
```
看TriggerPhasse方法就能知道，FMassEditorPhaseTickTask 这个Task的目的就是让 ProcessingPhase 执行Tick方法，下面我们看下这个最关键的Tick:
```cpp
void FMassProcessingPhase::ExecuteTick(float DeltaTime, ELevelTick TickType, ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
{
	//1 PhaseStart
	PhaseManager->OnPhaseStart(*this);
	{
		OnPhaseStart.Broadcast(DeltaTime);
	}
	//2 执行部分 先创建出 FMassProcessingContext
	FMassProcessingContext Context(EntityManager, DeltaTime);
	if (bRunInParallelMode && PhaseManager->IsPaused() == false)
	{
		// 多线程部分，下面看
	}
	else
	{
		// 单线程部分，很简单就是直接直接我们写的Processor的tick了
		if (PhaseManager->IsPaused() == false)
		{
			UE::Mass::Executor::Run(*PhaseProcessor, Context);
		}
		// 3 OnPhaseEnd 这部分
		{
			OnPhaseEnd.Broadcast(DeltaTime);
		}
		PhaseManager->OnPhaseEnd(*this);
	}
}
```
 **这个FMassProcessingPhase::ExecuteTick方法的目的就是执行所有的Processor的tick。而这个ExecuteTick这个方法的调用在编辑器下是通过FMassEditorPhaseTickTask这个Task，非编辑器下是将FMassProcessingPhase（这个类继承自TickFunction）注册到EntityManager所在的World中(编辑器下EntityManager不属于World属于EditorSubSystem)，跟随World的tick执行。**
#### 1.1.1 OnPhaseStart 开始部分
这个开始部分的目的是将属于这个Phase的Processor拓扑排序，排除执行的先后，将排序结果赋值到UMassCompositeProcessor这个里面。会根据如下图：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250727195920.png)
会根据这个图的配置，构建图然后拓扑排序，其中Group可以配成A.B.C，这会被分成三个图的节点，A，A.B，A.B.C。
#### 1.1.2 执行部分
UE::Mass::Tweakables::bFullyParallel 这个变量控制执行时是单线程还是多线程：
```cpp
namespace UE::Mass::Tweakables
{
	bool bFullyParallel = MASS_DO_PARALLEL;
	bool bMakePrePhysicsTickFunctionHighPriority = true;

	// 这个是命令行吧
	FAutoConsoleVariableRef CVars[] = {
		{TEXT("mass.FullyParallel"), bFullyParallel, TEXT("Enables mass processing distribution to all available thread (via the task graph)")},
		{TEXT("mass.MakePrePhysicsTickFunctionHighPriority"), bMakePrePhysicsTickFunctionHighPriority, TEXT("Whether to make the PrePhysics tick function high priority - can minimise GameThread waits by starting parallel work as soon as possible")},
	};
}
```
1 单线程的情况下：
```cpp
UE::Mass::Executor::Run(*PhaseProcessor, Context);
// 就直接执行了这个方法，这个方法里面会遍历我们写的Processor，然后执行里面的Execute方法。
```
2 多线程的情况下：
```cpp
const FGraphEventRef PipelineCompletionEvent = UE::Mass::Executor::TriggerParallelTasks(
				*PhaseProcessor, // 此次Tick的UMassCompositeProcessor
				MoveTemp(Context),// 移动局部变量ProcessingContext
				[this, DeltaTime](){
					OnParallelExecutionDone(DeltaTime);
				} ,// Task执行完的回调
				CurrentThread);// 当前线程
```
首先就是调用了 TriggerParallelTasks 这个方法，这个方法里面就是执行了两个Task。这个方法最终要的可能就是第二个参数了，把局部变量移动到了方法中，目的就是在不拷贝的情况下延长局部变量的生命周期。下面我们 来看这个方法里的两个Task：
```cpp
FGraphEventRef TriggerParallelTasks(...)
{
	//1 在 ProcessingContext 局部变量的内存中new了一个ExecutionContext，并通过右移传出来，传出来这个等号会执行 FMassExecutionContext 的移动构造函数
	FMassExecutionContext ExecutionContext = MoveTemp(ProcessingContext).GetExecutionContext();
//2 第一个Task，内部会创建 FMassProcessorTask 这个Task,这个Task就会执行我们自己定义 UMassProcessor 的 Execute 方法。并且我们把ExecutionContext 的引用传入到Execute中，在Execute中可能通过PushBuffer将命令添加到了ExecutionContext的CommandBuffer中。这个是在任意的工作线程中执行的。
	FGraphEventRef CompletionEvent;
	{
		CompletionEvent = Processor.DispatchProcessorTasks(
		ProcessingContext.GetEntityManager(), 
		ExecutionContext, 
		{});
	}
//3 第二个Task，在第一个Task执行完之后，ExecutionContext里面可能有CommandBuffer了，这里用右移延长局部变量的生命周期。然后呢这个Task里就是执行CommandBuffer里各种命令。这个是在主线程执行的，因为需要执行CommandBuffer里的命令。
	if (CompletionEvent.IsValid())
	{
		const FGraphEventArray Prerequisites = 
		{ CompletionEvent };
		CompletionEvent=TGraphTask<FMassExecutorDoneTask>::
					CreateTask(&Prerequisites)
						.ConstructAndDispatchWhenReady(
							MoveTemp(ExecutionContext), //右移
							OnDoneNotification, //回调函数
							Processor.GetName(), //
							CurrentThread);
	}
	return CompletionEvent;
}
```

多线程的情况下需要注意的就是：
==1 创建了ProcessingContext局部变量，并把局部变量右移到创建Task的方法中（这里不用拷贝的原因就是数据大不想拷贝，不用引用的原因是在Task还未知执行的时候引用对象就会销毁，所以用右移将局部变量中数据移到别的方法中延长生命周期）。==
==2 在 TriggerParallelTasks 方法里面，在局部变量的内存中用placement new的方式创建了ExecutionContext并右移出来（只是借用局部变量ProcessingContext的内存创建ExecutionContext并移动出来，局部变量ProcessingContext里面的数据什么都没变）。==
==3 在创建两个Task的方法里面，传给第一个Task的 ExecutionContext 是引用，这是因为这个Task之后的第二个Task还需要用 ExecutionContext ，所以这里不能是移动，移动的话 ExecutionContext 这个里的数据就没了，第二个Task还怎么用呢？所以第二个Task传的移动。==
3 DispatchProcessorTasks 是个虚函数
```cpp
FGraphEventRef UMassCompositeProcessor::DispatchProcessorTasks(
	const TSharedPtr<FMassEntityManager>& EntityManager, 
	FMassExecutionContext& ExecutionContext, 
	const FGraphEventArray& InPrerequisites)
{
	FGraphEventArray Events;
	Events.AddDefaulted(FlatProcessingGraph.Num());
	
	FGraphEventArray Prerequisites;
	TArray<FGraphEventArray> AdditionalEvents;
	
	for (int32 NodeIndex = 0; NodeIndex < FlatProcessingGraph.Num(); ++NodeIndex)
	{
		FDependencyNode& ProcessingNode = FlatProcessingGraph[NodeIndex];

		if (ensureMsgf(ProcessingNode.Processor, TEXT("")))
		{
			// 依赖数组，
			Prerequisites.Reset(ProcessingNode.Dependencies.Num());
			for (const int32 DependencyIndex : ProcessingNode.Dependencies)
			{
				Prerequisites.Add(Events[DependencyIndex]);
			}
			// this means there are some inactive processors so we need to consider additional dependencies
			if (AdditionalEvents.Num())
			{
				for (const int32 DependencyIndex : ProcessingNode.Dependencies)
				{
					Prerequisites.Append(AdditionalEvents[DependencyIndex]);
				}
			}

			if (ProcessingNode.Processor->IsActive())
			{
#if HOTTA_ENGINE_MODIFY // add by wujingjing
				if (FixedTimeTime > 0.0f)
				{
					if (ProcessingNode.Processor->ShouldTickEveryFrame())
					{
						Events[NodeIndex] = ProcessingNode.Processor->DispatchProcessorTasks(EntityManager, ExecutionContext, Prerequisites);
					}
					else
					{
						if (bShouldFixedTick)
						{
							Events[NodeIndex] = ProcessingNode.Processor->DispatchProcessorTasks(EntityManager, LocalExecutionContext, Prerequisites);
						}
						else
						{
							// 创建占位任务以维持依赖链。
							FGraphEventRef SkippedEvent = FFunctionGraphTask::CreateAndDispatchWhenReady([](){}, TStatId(), &Prerequisites, ENamedThreads::AnyHiPriThreadHiPriTask);
							Events[NodeIndex] = SkippedEvent;
						}
					}
				}
				else
				{
					Events[NodeIndex] = ProcessingNode.Processor->DispatchProcessorTasks(EntityManager, ExecutionContext, Prerequisites);
				}
#else
				Events[NodeIndex] = ProcessingNode.Processor->DispatchProcessorTasks(EntityManager, ExecutionContext, Prerequisites);
#endif
			}
			else
			{
				if (AdditionalEvents.Num() == 0)
				{
					// lazy initialization
					AdditionalEvents.AddDefaulted(FlatProcessingGraph.Num());
				}
				// if the processor is not going to run at all we store its Prerequisites so that
				// processors waiting for this given processor to finish will keep their place
				// in the overall processing graph
				// NOTE: this is safer than just ignoring the dependencies since even though this
				// processor is not running, the subsequent processors might unknowingly rely on
				// implicit dependencies that the current processor was ensuring. 
				AdditionalEvents[NodeIndex].Append(MoveTemp(Prerequisites));
			}
		}
	}
}
```

#### 1.1.3 OnPhaseEnd 结束部分
```cpp
void FMassProcessingPhaseManager::OnPhaseEnd(FMassProcessingPhase& Phase)
{
	CurrentPhase = EMassProcessingPhase::MAX;
	if (GetEntityManagerRef().Defer().HasPendingCommands())
	{
		GetEntityManagerRef().FlushCommands();
	}
}
```
这个结束部分就比较简单了，就是执行CommandBuffer里面的命令。在单线程情况下会执行 FlushCommands ，在多线程情况下 FMassExecutorDoneTask 这个Task里面会执行一次 FlushCommands，不管Task是否执行完CommandBuffer里的命令，都会在OnPhaseEnd里在执行一次。

问题：
1 
```cpp
/*
1 TArray<const UScriptStruct*, TInlineAllocator<16>> SortedFragmentList;  
CompositionDescriptor.Fragments.ExportTypes(SortedFragmentList);
这个Fragments里面会有重复的么？也就是SortedFragmentList这个数组里会有重复值么？
目前来看应该是可能有重复的，没找到去重的逻辑，而且TArray的add也支持添加重复元素。
*/
```
2 processor并行的实现


### 1.2 RunProcessorsView

```cpp
void RunProcessorsView(
	TArrayView<UMassProcessor* const> Processors, 
	FProcessingContext& ProcessingContext, 
	TConstArrayView<FMassArchetypeEntityCollection> EntityCollections)
{
	// 从处理上下文中获取ExecutionContext上下文
	FMassExecutionContext& ExecutionContext = 
		ProcessingContext.GetExecutionContext();
	// 从ProcessingContext上下文获取EntityManager
	FMassEntityManager& EntityManager = *ProcessingContext.GetEntityManager();
	// 一个Scope，增加计数表示正在处理的Process增加一个
	FMassEntityManager::FScopedProcessing ProcessingScope = 
		EntityManager.NewProcessingScope();

	if (EntityCollections.Num() == 0)
	{
		ExecuteProcessors(EntityManager, Processors, ExecutionContext);
	}
	else
	{
		for (const FMassArchetypeEntityCollection& Collection : EntityCollections)
		{
			ExecutionContext.SetEntityCollection(Collection);
			ExecuteProcessors(EntityManager, Processors, ExecutionContext);
			ExecutionContext.ClearEntityCollection();
		}
	}
}
```

# 3 Processor

## 1 UMassTrafficUpdateVelocityProcessor
```cpp
UMassTrafficUpdateVelocityProcessor::UMassTrafficUpdateVelocityProcessor()
	: EntityQuery_Conditional(*this)
{
	// 自动注册tick
	bAutoRegisterWithProcessingPhases = true;
	ExecutionOrder.ExecuteInGroup = 
		UE::MassTraffic::ProcessorGroupNames::VehicleBehavior;
	ExecutionOrder.ExecuteAfter.Add(
		UE::MassTraffic::ProcessorGroupNames::FrameStart);
	ExecutionOrder.ExecuteAfter.Add(
		UE::MassTraffic::ProcessorGroupNames::PreVehicleBehavior);
	ExecutionOrder.ExecuteAfter.Add(
		UE::MassTraffic::ProcessorGroupNames::VehicleSimulationLOD);
	ExecutionOrder.ExecuteAfter.Add(
		UMassTrafficInterpolationProcessor::StaticClass()->GetFName());
}
```
# 基本概念
FMassEntityTemplateBuildContext
```cpp
1 数组TraitsData，数组中的元素是FTraitData（保留TraitObject的指针，添加的MassFragment，添加的MassTag）。在创建EntityTemplate的时候，通过RequireFragment，RequireTag方法向FTraitData中的添加，通过AddFragment方法向TemplateData中添加。
2 FMassEntityTemplateData& TemplateData，在Context创建时传进来的，具体的Template的数据
{
	1 FMassArchetypeCompositionDescriptor Composition;原型的组成描述结构，包含组成远程的Fragment,Tag,ChunkFragments,SharedFragments,ConstSharedFragments。每种Fragment都有对应的BitSet结构加速添加查询。就是每种不同的Fragment都在bitArray中占一位，通过与或非来添加查询。
	2 FMassArchetypeSharedFragmentValues SharedFragmentValues;
	3 TArray<FInstancedStruct> InitialFragmentValues;
	4 TArray<FObjectFragmentInitializerFunction> ObjectInitializers;缓存的TFunction,类似于构造函数，在创建出Template的时候立刻执行的回调
	5 FMassArchetypeCreationParams CreationParams;创建原型的一些参数
	6 FString TemplateName;
}
```
FMassEntityTemplate：
```cpp
1 FMassEntityTemplateData TemplateData; 在创建Template的时候创建的局部变量，通过move一直移动到Template中
2 FMassArchetypeHandle Archetype;  记录原型指针的成员变量
3 FMassEntityTemplateID TemplateID; 在创建Template的时候生成的唯一ID
// 流程：
1 由资产UMassEntityConfigAsset调用GetOrCreateEntityTemplate这个方法创建出来的，创建出来后保存到MassSpawnerSubsystem的TemplateRegistry结构中
2 在创建方法中，会首先创建一个MassEntityTemplateBuildContext，下面所有的步骤都会将Context传进去
3 接下来会拿到配置的所有Traits，这些Trait都是InstanceObject的形式。所有的Trait都执行BuildTemplate方法
```
FArchetype：
```cpp
1 chunk的组成是，多个EntityHandle挨着存放 + 每个EntityHandle的FragmentA挨着存放 + 每个EntityHandle的FragmentB挨着存放
// 流程：
1 EnittyTemplate里会存储Archetypehandle，所以原型也是在Template的创建流程里创建的，是通过FMassEntityManager::CreateArchetype这个方法来创建的
```

