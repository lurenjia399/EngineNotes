https://zhuanlan.zhihu.com/p/392864278

gc基本概念
https://zhuanlan.zhihu.com/p/401956734
基本概念
https://mytechplayer.com/archives/ue5%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86%E5%8E%9F%E7%90%86
ue5垃圾回收
https://zhuanlan.zhihu.com/p/17382215157

ue这边使用的算法是标记清除的算法，也就是两个阶段，先标记后清除。要想标记就需要先获得所有的Object对象，然后再用某种清楚规则来清除。

# 1 获取所有对象
```cpp
UObjectBase::UObjectBase(...)
{
	AddObject(InName, InInternalFlags, InInternalIndex, InSerialNumber);
}
void UObjectBase::AddObject(...)
{
	// 将Object放到了GUObject数组里面
	GUObjectArray.AllocateUObjectIndex(this, InternalFlagsToSet, InInternalIndex, InSerialNumber);
}
```
最终会将我们的Object组装成FUObjectItem结构，放入到GUObjectArray数组中。
```cpp
struct FUObjectItem
{
	class UObjectBase* Object;//对应的就是我们Object
	int32 Flags;//EInternalObjectFlags 的标志位
	int32 ClusterRootIndex;// 当前所属簇的索引
	int32 SerialNumber;//Object的序列码
}
enum class EInternalObjectFlags : int32
{
	None = 0,

	// 三个可达标志位，可达，可能不可达，不可达。含义会改变所以这里命令是这样。
	ReachabilityFlag0 = 1 << 0,
	ReachabilityFlag1 = 1 << 1,
	ReachabilityFlag2 = 1 << 2,
	
	LoaderImport = 1 << 20,//对象由其他package依赖，导致需要加载
	Garbage = 1 << 21,// 标记对象为垃圾
	ReachableInCluster = 1 << 23,// object是一个簇中的Object，同时也被其他簇引用
	ClusterRoot = 1 << 24,// 簇根节点，不会被gc回收
	Native = 1 << 25,//C++类对象
	Async = 1 << 26,//异步对象，不存在于游戏线程，存在于其他工作线程
	AsyncLoading = 1 << 27,// 对象正在异步加载
	Unreachable = 1 << 28,// 对象不可达，会被gc回收
	RootSet = 1 << 30, // 根节点，不会被gc回收，即使没有被引用
	PendingConstruction = 1 << 31,// 对象正在执行构造函数，UObjectBase的构造函数

	// 拥有这个标记的，在gc中会被跳过
	GarbageCollectionKeepFlags = Native | Async | AsyncLoading | LoaderImport,
};
```
# 2 触发GC
```cpp
// 手动通过这个方法来gc，设置bFullPurgeTriggered标志位，在下一帧调用gc
void UEngine::ForceGarbageCollection(bool bForcePurge)
{
	TimeSinceLastPendingKillPurge = 1.0f + GetTimeBetweenGarbageCollectionPasses();
	bFullPurgeTriggered = bFullPurgeTriggered || bForcePurge;
}
void UWorld::Tick(...)
{
	// 在World的Tick里会触发gc
	GEngine->ConditionalCollectGarbage();
}
void UEngine::ConditionalCollectGarbage()
{
	// TryCollectGarbage 调用gc
	if (bFullPurgeTriggered)
	{
		if (TryCollectGarbage(GARBAGE_COLLECTION_KEEPFLAGS, true))
		{
			ForEachObjectOfClass(UWorld::StaticClass(),[](UObject* World)
			{
				CastChecked<UWorld>(World)->CleanupActors();
			});
			bFullPurgeTriggered = false;
			bShouldDelayGarbageCollect = false;
			TimeSinceLastPendingKillPurge = 0.0f;
		}
	}
}
```
1 可以通过ForceGarbageCollection手动触发gc，在下一帧就会调用TryCollectGarbage方法来走gc流程。
```cpp
// KeepFlags = GIsEditor ? RF_Standalone : RF_NoFlags
// 这个KeepFlags只有再编辑器下才会有值，其余的全为RF_NoFlags
// 这个bPerformFullPurge为true，可能是代表全量gc？
bool TryCollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	UE::GC::CollectGarbageInternal(KeepFlags, bPerformFullPurge);
}

void CollectGarbageInternal(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	GReachabilityState.CollectGarbage(KeepFlags, bPerformFullPurge);
}
void FReachabilityAnalysisState::CollectGarbage(EObjectFlags KeepFlags, bool bFullPurge)
{
	ObjectKeepFlags = KeepFlags;
	bPerformFullPurge = bFullPurge;
	PerformReachabilityAnalysisAndConditionallyPurgeGarbage(
	bReachabilityUsingTimeLimit);
}
void FReachabilityAnalysisState::PerformReachabilityAnalysisAndConditionallyPurgeGarbage(bool bReachabilityUsingTimeLimit)
{
	UE::GC::PreCollectGarbageImpl<true>(ObjectKeepFlags);
	// 这个方法会转发到CollectGarbageImpl这个方法里
	PerformReachabilityAnalysis();
	UE::GC::PostCollectGarbageImpl<true>(ObjectKeepFlags);
}
void FReachabilityAnalysisState::PerformReachabilityAnalysis()
{
	// 如果没有暂停，就把Iteration计数归0
	if (!bIsSuspended)
	{
		NumIterations = 0;
	}
	if (bPerformFullPurge)
	{
		UE::GC::CollectGarbageFull(ObjectKeepFlags);
	}
	// 执行完可达性分析后就增加计数
	NumIterations++;
}

void CollectGarbageImpl(EObjectFlags KeepFlags)
{
	// Options里应该就包括一个多线程gc的标志位EGCOptions::Parallel
	const EGCOptions Options = GetReferenceCollectorOptions(bPerformFullPurge);
	FRealtimeGC GC;
	// 可达性分析
	GC.PerformReachabilityAnalysis(KeepFlags, Options);
}

```
2 最终走到GarbageCollection.cpp的PerformReachabilityAnalysis可达性分析方法里面。
```cpp
void PerformReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
{
	// 开始可达性分析，将所有Object标记为可能不可达并将根Object标记为可达。
	StartReachabilityAnalysis(KeepFlags, Options);
	do
	{
		// 可达性分析的pass
		PerformReachabilityAnalysisPass(Options);
	} while ((VerseGCActive() || !Private::GReachableObjects.IsEmpty() || !Private::GReachableClusters.IsEmpty()) && !GReachabilityState.IsSuspended());
}
```
# 3 标记初始化 StartReachabilityAnalysis
```cpp
void StartReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
{
	// 用另一个线程将继承自GCObject的对象放到InitialReferences数组中
	BeginInitialReferenceCollection(Options);
	InitialObjects.Reset();
	// 如果需要cook也就是不是编辑器 && FGCObject::GGCObjectReferencer这个对象不需要考虑gc，也都放到InitialObjects数组中。含义应该是FGCObject::GGCObjectReferencer这个对象即使不会被gc，但也需要检测其引用的其他对象是否需要gc。
	if (FPlatformProperties::RequiresCookedData() && GUObjectArray.IsDisregardForGC(FGCObject::GGCObjectReferencer))
	{
		InitialObjects.Add(FGCObject::GGCObjectReferencer);
	}
	// 初始化标记，将所有Object标记为可能不可达，然后将根Object标记为可达。
	MarkObjectsAsUnreachable(KeepFlags);
}
```
1 StartReachabilityAnalysis 又分为了两部分，第一部分通过工作线程将我们GCObject放到InitialReferences数组中。第二部分将所有Object都标记为可能不可达，然后将根Object标记为可达。我们主要看下第二部分。
```cpp

FORCENOINLINE void MarkObjectsAsUnreachable(const EObjectFlags KeepFlags)
{
	// 如果没有引用垃圾
	if (!Stats.bFoundGarbageRef)
	{
		// 交换可达和可能不可达的含义，将所有Object都标记为了可能不可达。
		Swap(GReachableObjectFlag, GMaybeUnreachableObjectFlag);
	}
	MarkClusteredObjectsAsReachable(GatherOptions, InitialObjects);
	// 将根Object去掉可能不可达标记，并添加可达标记。
	// 编辑器模式下，还把所有Object中带有RF_Standalone标志位的Object也去掉可能不可达标记并添加可达标记了
	MarkRootObjectsAsReachable(GatherOptions, KeepFlags, InitialObjects);
}

FORCENOINLINE void MarkClusteredObjectsAsReachable(const EGatherOptions Options, TArray<UObject*>& OutRootObjects)
{
	FMarkClustersState GatherClustersState;
	// 将整个簇数组划分，每个工作线程负责几个簇，记录在FMarkClustersState结构中
	GatherClustersState.Start(Options, ClusterArray.Num(), 0, NumThreads);
	// 通过工作线程来遍历簇数组，有几个工作线程遍历几次，每个工作线程只处理自己负责的簇
	ParallelFor(...)
	{// 工作线程处理主体
		// 从全局数组中拿到簇的根ObjectItem
		FUObjectItem* RootItem = &GUObjectArray.GetObjectItemArrayUnsafe()[Cluster.RootIndex];
		// 簇中根Object还不是垃圾，就把簇中其余Object都标记为可达
		if (!RootItem->IsGarbage())
		{
			// 代码就省略了，处理簇中Object，如果Object带有跳过gc标志位，把簇根和簇中Object都标记为可达
		}
		else
		{
			ThreadState.Payload.ClustersToDissolve.Add(RootItem);
		}
	}
	// 将所有工作线程处理的结果整合到MarkClustersResults中
	FMarkClustersArrays MarkClustersResults;
	GatherClustersState.Finish(MarkClustersResults);
	/* 如果簇根是垃圾，直接解散簇
	DissolveClusterAndMarkObjectsAsUnreachable，解散簇的方法。第一步设置簇中object的所属簇索引为0，并将簇中object标记为可能不可达（这里虽然标记为可能不可达，但是之前有交换所以这里实际标记成可达了，为啥呢？）。第二步解散掉簇引用的其他簇，递归调用。
	*/
	for (FUObjectItem* ObjectItem : MarkClustersResults.ClustersToDissolve)
	{
		if (ObjectItem->HasAnyFlags(EInternalObjectFlags::ClusterRoot))
		{
		GUObjectClusters.DissolveClusterAndMarkObjectsAsUnreachable(ObjectItem);
		GUObjectClusters.SetClustersNeedDissolving();
		}
	}
	/* 如果簇根是根Object或者是跳过gc，簇中其余Object是根或者跳过gc，则就需要保留簇，这个方法中首先将簇根和簇中Object都标记为可能不可达。
	MarkReferencedClustersAsReachable：把簇中引用的object都标记为可达。第一步处理簇引用的其他簇根，如果其他簇根不是垃圾，就标记为可达，如果是垃圾就清掉引用。第二步处理簇中MutableObjects（不在簇中但依然被簇引用。看上去是object属于多个簇，但只在一个簇中这种情况。），将其中Object标记为可达，将从可达变为可能不可达的Object添加到ObjectsToSerialize数组中。第三步，如果簇中有垃圾（无论是引用的簇根还是mutableObjects中有垃圾），我们就把簇中Object都添加到InitialObjects数组中，并解散簇。
	*/
	for (FUObjectItem* ObjectItem : MarkClustersResults.KeepClusters)
	{
		MarkReferencedClustersAsReachable<EGCOptions::None>(ObjectItem->GetClusterIndex(), OutRootObjects);
	}
}
```
2 MarkObjectsAsUnreachable这个方法主要是标记根Object为可达的，并将根Object添加到InitialObjects数组中。MarkObjectsAsUnreachable在这个方法的开头就交换了可达和可能不可达标记，很巧妙的交换了可达和可能不可达。举个例子就是：A代表可达，B代表可能不可达，交换后A代表可能不可达，B代表可达。并没有根据含义去改变值，而是直接改变了值得含义，很巧妙。
此时InitialObjects数组里就只有根Object吧，并且标记都为可达。
# 4 通过引用关系修改标记 PerformReachabilityAnalysisPass

```cpp
void PerformReachabilityAnalysisPass(const EGCOptions Options)
{
	if (!Private::GReachableObjects.IsEmpty())
	{
		Private::GReachableObjects.PopAllAndEmpty(InitialObjects);
		ConditionallyAddBarrierReferencesToHistory(*Context);
	}
	// 第一次进行可达性分析 || (引用垃圾 && 没有暂停可达性分析)
	else if (GReachabilityState.GetNumIterations() == 0 || (Stats.bFoundGarbageRef && !GReachabilityState.IsSuspended()))
	{
		//目的是将GGCObjectReferencer数组中的值添加到InitialReferences数组中,然后返回
		Context->InitialNativeReferences = GetInitialReferences(Options);
	}
	if (!Private::GReachableClusters.IsEmpty())
	{
		TArray<FUObjectItem*> KeepClusterRefs;
		Private::GReachableClusters.PopAllAndEmpty(KeepClusterRefs);
		for (FUObjectItem* ObjectItem : KeepClusterRefs)
		{
			MarkReferencedClustersAsReachable<EGCOptions::None>(ObjectItem->GetClusterIndex(), InitialObjects);
		}
	}
	// 把InitialObjects数组里的值塞到Context里
	Context->SetInitialObjectsUnpadded(InitialObjects);
	// 处理Object引用关系，最终会执行PerformReachabilityAnalysisOnObjectsInternal方法
	PerformReachabilityAnalysisOnObjects(Context, Options);
}
```

```cpp
void PerformReachabilityAnalysisOnObjectsInternal(FWorkerContext& Context)
{
	TReachabilityProcessor<Options> Processor;
	CollectReferencesForGC<TReachabilityCollector<Options>>(Processor, Context);
}

FORCEINLINE void CollectReferencesForGC(ProcessorType& Processor, UE::GC::FWorkerContext& Context)
{
	// 多线程版本执行的
	if constexpr (IsParallel(ProcessorType::Options))
	{
		ProcessAsync([](void* P, FWorkerContext& C) { FastReferenceCollector(*reinterpret_cast<ProcessorType*>(P)).ProcessObjectArray(C); }, &Processor, Context);
	}
	// 没有多线程执行的，执行的逻辑是一样的
	else
	{
		FastReferenceCollector(Processor).ProcessObjectArray(Context);
	}
}
```

```cpp
void ProcessObjectArray(FWorkerContext& Context)
{
	// 遍历InitialNativeReferences数组
	for (UObject** InitialReference : Context.InitialNativeReferences)
	{
		// 这里是获取Dispatcher，GetDispatcher的定义有两个地方，一个是默认的，一个是TReachabilityCollector类里的，后者是全特化版本。实际执行会根据Collector类型的不同执行不同的版本，TReachabilityCollector类型执行后者，TDefaultCollector类型执行前者。这里Collector的类型是TReachabilityCollector，所以Dispatcher是TBatchDispatcher类型。
		CollectorType Collector(Processor, Context);
		decltype(GetDispatcher(Collector, Processor, Context)) Dispatcher = GetDispatcher(Collector, Processor, Context);
		// 
		for (UObject** InitialReference : Context.InitialNativeReferences)
		{
			Dispatcher.HandleKillableReference(*InitialReference, EMemberlessId::InitialReference, EOrigin::Other);
		}
		// CurrentObjects是while循环中需要处理的Object数组
		TConstArrayView<UObject*> CurrentObjects = Context.InitialObjects;
		while (true)
		{
			// 处理Object，标记Object和其相关的Object
			ProcessObjects(Dispatcher, CurrentObjects);
			// 每次循环的时候，如果不是第一次循环，就需要将ObjectsToSerialize中的block内存释放掉
			if (CurrentObjects.GetData() != Context.InitialObjects.GetData())
			{
			Context.ObjectsToSerialize.FreeOwningBlock(CurrentObjects.GetData());
			}
			// 一次循环处理完就超时了，那就把积压的Object处理掉，直接返回不继续处理了
			if (Processor.IsTimeLimitExceeded())
			{
				FlushWork(Dispatcher);
				Dispatcher.Suspend();
				SuspendWork(Context);
				return;
			}
			// 每个Block的大小
			int32 BlockSize = FWorkBlock::ObjectCapacity;
			// ProcessObjects的过程中，从可能不可达到可达新增的Object
			FWorkBlockifier& RemainingObjects = Context.ObjectsToSerialize;
			// 弹出Block
			FWorkBlock* Block = RemainingObjects.PopFullBlock<Options>();
			if(!Block)
			{
				// 把积压在UnvalidatedReferences或者是在ValidatedReferences中的Object都强行处理掉
				FlushWork(Dispatcher);
				// 处理完积压的后在弹出Block
				Block = RemainingObjects.PopFullBlock<Options>()
				// 如果Block仍然为空，就结束死循环
				if (!Block)
				{
					break;
				}
			}
			// while循环结束后，更新下一次循环需要处理的Object数组
			CurrentObjects = MakeArrayView(Block->Objects, BlockSize);
	}
}

```
1 会开启一个死循环，每次循环都会通过ProcessObjects方法，处理CurrentObjects数组中的Object对象，标记每个对象的引用Object为可达，将新增的可达对象添加到ObjectsToSerialize中。ProcessObjects方法结束，就会更新CurrentObjects数组，将新增的可达对象放到CurrentObjects数组里，供下次循环使用。直到ObjectsToSerialize中不存在任何Object或者时间到了，退出死循环。
2 ObjectsToSerialize这个是FWorkBlockifier结构，看上去是一个链表，链表的每个节点都是一个block，每个block就相当于一个数组，数组中存放的是Object。在往ObjectsToSerialize添加Object的时候会先添加到block中，如果block满了就会在创建一个block，把他们连起来。所以在每次循环处理结束后，会弹出block。
## 1 标记Object方法，ProcessObjects
```cpp
FORCEINLINE_DEBUGGABLE void ProcessObjects(DispatcherType& Dispatcher, TConstArrayView<UObject*> CurrentObjects)
{
	// 遍历Object数组，对每个Object进行处理
	for (FPrefetchingObjectIterator It(CurrentObjects); It.HasMore(); It.Advance())
	{
		// Object对象
		UObject* CurrentObject = It.GetCurrentObject();
		// Object的UClass
		UClass* Class = CurrentObject->GetClass();
		// Object的Outer
		UObject* Outer = CurrentObject->GetOuter();
		// 如果UClass没创建Schema，就再这里创建下
		if (!!(Options & EGCOptions::AutogenerateSchemas) && !Class->HasAnyClassFlags(CLASS_TokenStreamAssembled))
		{			
			Class->AssembleReferenceTokenStream();
		}
		// 获取UClass的Schema
		FSchemaView Schema = Class->ReferenceSchema.Get();
		// 标记UClass
		Dispatcher.HandleImmutableReference(Class, EMemberlessId::Class, EOrigin::Other);
		// 标记Outer
		Dispatcher.HandleImmutableReference(Outer, EMemberlessId::Outer, EOrigin::Other);
		// 如果Schema存在，就通过Schema标记Object引用的对象。Schema中记录了Object引用对象的偏移，我们通过Object的位置加上偏移，就能得到引用对象了。
		if (!Schema.IsEmpty())
		{
			Private::VisitMembers(Dispatcher, Schema, CurrentObject);
		}
	}
}
```
标记对象的流程走的TBatchDispatcher的HandleKillableReference方法，下面详细看下。
## 2 标记Object，HandleKillableReference
```cpp
// TBatchDispatcher 类中的方法
void HandleKillableReference(UObject*& Object, FMemberId MemberId, EOrigin Origin)
{
	QueueReference(Context.GetReferencingObject(), Object, MemberId, ProcessorType::MayKill(Origin, true));
}
void QueueReference(const UObject* ReferencingObject,  UObject*& Object, FMemberId MemberId, EKillable Killable)
{
	// 蓝图类型为EKillable::Yes，其他类型为EKillable::No
	if (Killable == EKillable::Yes)
	{
		FPlatformMisc::Prefetch(&Object);
		KillableBatcher.PushReference(FMutableReference{&Object});
	}
	else
	{
		ImmutableBatcher.PushReference(FImmutableReference{Object});
	}
}
void PushReference(UnvalidatedReferenceType UnvalidatedReference)
{
	// 会往UnvalidatedReferences数组里添加，直到添加满了就执行DrainUnvalidatedFull方法
	UnvalidatedReferences.Push(UnvalidatedReference);
	if (UnvalidatedReferences.IsFull())
	{
		DrainUnvalidatedFull();
	}
}
```
1 会将每个Object添加到UnvalidatedReferences无效引用数组里，如果无效引用数组满了，就会执行DrainUnvalidated方法。下面详细看下。
```cpp
void DrainUnvalidated(const uint32 Num)
{
	// 记录GUObjectAllocator中的永久对象池
	FPermanentObjectPoolExtents Permanent(PermanentPool);
	FValidatedBitmask ValidsA, ValidsB;
	// ValidsA里每一位代表Object是否在永久对象池里，0代表在，1代表不在。所以表示不在永久对象池里的Object才会被gc处理。（现在游戏里的UClass都是再永久对象池的吧）
	for (uint32 Idx = 0; Idx < Num; ++Idx)
	{
		UObject* Object = GetObject(UnvalidatedReferences[Idx]);
		uint64 ObjectAddress = reinterpret_cast<uint64>(Object);
		ValidsA.Set(Idx, !Permanent.Contains(Object));
	}
	// ValidsB里每一位代表Object是否在内存中存在，0代表不存在，1代表存在。
	for (uint32 Idx = 0; Idx < Num; ++Idx)
	{
		UObject* Object = GetObject(UnvalidatedReferences[Idx]);
		ValidsB.Set(Idx, (!!Object) & IsObjectHandleResolved(reinterpret_cast<FObjectHandle&>(Object)));
	}
	// ValidsA和ValidsB进行与的操作，最后不在永久对象池并且在内存中的Object会被标记为1
	FValidatedBitmask Validations = FValidatedBitmask::And(ValidsA, ValidsB);
	// 数Validations中1的数量，也就是Object的数量
	uint32 NumValid = Validations.CountBits();
	// 会将Validations中记录的Object添加到ValidatedReferences数组中，如果ValidatedReferences满了就执行DrainValidatedFull方法
	uint32 UnvalidatedIdx = 0;
	for (uint32 Slack = ValidatedReferences.Slack(); NumValid >= Slack; Slack = ValidatedBatchSize)
	{
		QueueValidReferences(Slack, Validations,  UnvalidatedIdx);
		DrainValidatedFull();
		NumValid -= Slack;
	}
	QueueValidReferences(NumValid, Validations, UnvalidatedIdx);
	// 清掉UnvalidatedReferences数组的内容
	UnvalidatedReferences.Num = 0;
}
```
2 DrainUnvalidated方法主要就是遍历UnvalidatedReferences数组里Object，将满足条件的Object添加到ValidatedReferences这个数组里。条件是Object不存在永久对象池中并且Object依然还在内存中。在ValidatedReferences数组满了的时候，会调用DrainValidated方法。
```cpp
void DrainValidated(const uint32 Num)
{
	// 遍历ValidatedReferences数组，对里边每个Object执行HandleBatchedReference方法
	for (uint32 Idx = 0; Idx < Num; ++Idx)
	{
		ProcessorType::HandleBatchedReference(Context, ValidatedReferences[Idx], Metadatas[Idx]);
	}
	ValidatedReferences.Num = 0;
}

static void HandleBatchedReference(FWorkerContext& Context, FImmutableReference Reference, FReferenceMetadata Metadata)
{
	// 记录是否引用垃圾了，如果不是清除垃圾阶段 && Object是垃圾 就说明引用垃圾了
	DetectGarbageReference(Context, Metadata);
	HandleValidReference(Context, Reference, Metadata);
}

FORCEINLINE static bool HandleValidReference(FWorkerContext& Context, FImmutableReference Reference, FReferenceMetadata Metadata)
{
	// 清掉可能不可达标记添加可达标记。如果成功清掉可能不可达标记返回值为true，返回值得含义应该就是防止多次添加到ObjectsToSerialize数组中。
	if (Metadata.ObjectItem->MarkAsReachableInterlocked_ForGC())
	{
		// 如果Object不是簇根，就添加到ObjectsToSerialize数组里，后边继续处理
		if (!Metadata.Has(EInternalObjectFlags::ClusterRoot))
		{
			Context.ObjectsToSerialize.Add<Options>(Reference.Object);
		}
		else
		{
			// 这个是对簇的处理
			MarkReferencedClustersAsReachableThunk<Options>(Metadata.ObjectItem->GetClusterIndex(), Context.ObjectsToSerialize);
		}
		return true;
	}
	// 如果Object没有理论可能不可达标记，else是对簇的处理
	else  
	{
		
	}
	
	return false;
}
```
3 最终标记流程结束，从根Object开始，获取根Object得UClass，Outer，引用，将其添加到UnvalidatedReferences数组中，如果UnvalidatedReferences数组满了，会将数组中满足条件（不在永久对象池和在内存中）的Object添加到ValidatedReferences数组中，ValidatedReferences数组满了就遍历数组内容，将其可能不可达标记去掉，并添加可达标记，并添加到ObjectsToSerialize数组中。
最终标记流程结束，遍历到得Object会被标记为可达的，没遍历到的Object是可能不可达的。
# 5 引用关系的信息收集
```cpp
// 这个方法是在UClass创建的过程中执行的
void ProcessNewlyLoadedUObjects(FName Package, bool bCanProcessNewlyLoadedObjects)
{
	// 其余部分全都省略了，这里只关注gc的部分
	if (bNewUObjects && !GIsInitialLoad)
	{
		for (UClass* Class : AllNewClasses)
		{
			if (!Class->HasAnyFlags(RF_ClassDefaultObject) && !Class->HasAnyClassFlags(CLASS_TokenStreamAssembled))
			{
				Class->AssembleReferenceTokenStream();
			}
		}
	}
}
// cpp的UClass不用上锁，其余的需要上锁，防止多线程数据竞争
void UClass::AssembleReferenceTokenStream(bool bForce)
{
	if (ClassFlags & CLASS_Native)
	{
		AssembleReferenceTokenStreamInternal(bForce);
	}
	else
	{
		FScopeLock NonNativeLock(&GAssembleSchemaLock);
		AssembleReferenceTokenStreamInternal(bForce);
	}
}
```

```cpp
void UClass::AssembleReferenceTokenStreamInternal(bool bForce)
{
	// 没有创建过Token，也就是没有CLASS_TokenStreamAssembled标志位，才会走进去
	if (!HasAnyClassFlags(CLASS_TokenStreamAssembled) || bForce)
	{
		FSchemaBuilder Schema(0);
		FSchemaView SuperSchema;
		// 首先将父类的SuperSchema组装到Schema到开头
		if (UClass* SuperClass = GetSuperClass())
		{
			SuperClass->AssembleReferenceTokenStreamInternal();	
			SuperSchema = SuperClass->ReferenceSchema.Get();
			Schema.Append(SuperSchema);
		}
		// 遍历UClass中的Property，不遍历父类中的Property，每个Property都执行EmitReferenceInfo方法，EmitReferenceInfo这个方法每种Property都是自己实现的。
		for (TFieldIterator<FProperty> It(this, EFieldIteratorFlags::ExcludeSuper); It; ++It)
		{
			FProperty* Property = *It;
			FPropertyStackScope PropertyScope(DebugPath, Property);
			Property->EmitReferenceInfo(Schema, 0, EncounteredStructProps, DebugPath);
		}
		// 这边还处理了没有经过UHT的c++类，就是那种裸的C++类
		if (ClassFlags & CLASS_Intrinsic)
		{
			Schema.Append(UE::GC::GetIntrinsicSchema(this));
		}
		// 通过将Schema构建成FSchemaView，最后将其保存到UClass的ReferenceSchema结构中
		FSchemaView View(bReuseSuper ? SuperSchema : Schema.Build(GetARO(this)), Origin);
		// 最后将首地址存到UClass中
		ReferenceSchema.Set(View);
		// 记录已经处理过的标志位
		ClassFlags |= CLASS_TokenStreamAssembled;
	}
}
```
1 UClass中每个不同的FProperty都有不同的EmitReferenceInfo方法。我们拿UObject的举例
```cpp
void FObjectProperty::EmitReferenceInfo(...)
{
	// ArrayDim看上去是原生数组的形式，可能是 UObject* Test[4]，这样的属性ArrayDim就为4？可能是这样吧
	for (int32 Idx = 0, Num = ArrayDim; Idx < Num; ++Idx)
	{
		Schema.Add(UE::GC::DeclareMember(DebugPath, BaseOffset + GetOffset_ForGC() + Idx * sizeof(FObjectPtr), UE::GC::EMemberType::Reference));
	}
}
```
2 向Schema中添加DeclareMember结构，这个结构记录了DebugPath，当前属性在类中偏移量，属性标志位。这个偏移量的计算用到了GetOffset_ForGC方法，这个方法返回Offset_Internal成员变量，Offset_Internal的赋值是在创建属性FProperty的时候赋值的，也就是在UClass创建的时候。
```cpp
// 构建FSchemaView
FSchemaView FSchemaBuilder::Build(ObjectAROFn ARO)
{
	// 首先将Schema中的数据根据偏移的大小排序，小的在前，大的在后
	Algo::SortBy(Members, &FMemberDeclaration::Offset);
	// 然后还把ARO(AddReferenceObject函数指针)指向的添加进来
	Members.Add(GenerateTerminator(ARO));
	// 分配内存，存放Members中的数据
	FMemberWord* FirstWord = (FMemberWord*)(Header + 1);
	FMemory::Memcpy(FirstWord, Packed.Words.GetData(), sizeof(FMemberWord) * Packed.Words.Num());
	FMemory::Memcpy(FirstWord + Packed.Words.Num(), Packed.DebugNames.GetData(), sizeof(FName) * Packed.DebugNames.Num());
	// 返回首地址handler组成的FSchemaView
	return BuiltSchema.Emplace(FSchemaView(FirstWord)).Get();
}
```
3 通过Build方法，首先将Schema中的数据（UObject中存在的UProperty修饰的属性）根据偏移大小排序，然后分配一块内存来存放这些数据，然后再将这块内存的首地址返回，最后将首地址存到UClass中。
# 6 清除阶段

```cpp
void PostCollectGarbageImpl(EObjectFlags KeepFlags)
{
	// 交换不可达和可能不可达含义，将所有标记为可能不可达的Object标记成不可达。
	Swap(GUnreachableObjectFlag, GMaybeUnreachableObjectFlag);
	// 解散掉不可达的簇，参数是否是多线程
	const EGatherOptions GatherOptions = GetObjectGatherOptions();
	DissolveUnreachableClusters(GatherOptions);
	// 清掉弱引用，但是什么情况下会有弱引用呢？
	ClearWeakReferences<true>(AllContexts);
	// 收集不可达Object
	GGatherUnreachableObjectsState.Init();
	if (bPerformFullPurge || !GAllowIncrementalGather)
	{
		GatherUnreachableObjects(GatherOptions, 0.0);
	}
	// 解锁Object的HashTable
	UnlockUObjectHashTables();
	// 垃圾收集是否正在进行的标志位
	GIsGarbageCollecting = false;
	// 在全局hash表中移除不可达Object的信息
	UnhashUnreachableObjects(false);
	IncrementalPurgeGarbage(false);
}
```
## 1 收集不可达Object
```cpp
bool GatherUnreachableObjects(UE::GC::EGatherOptions Options, double TimeLimit)
{
	// 和之前遍历根Object一样，都是用多线程优化遍历数组
	if (!GGatherUnreachableObjectsState.IsSuspended())
	{
		const int32 FirstIndex = GExitPurge ? 0 : GUObjectArray.GetFirstGCIndex();
		GGatherUnreachableObjectsState.Start(Options, GUObjectArray.GetObjectArrayNum(), FirstIndex);
	}
	// 下面是遍历主体，伪代码
	{
		FUObjectItem* ObjectItem = &GUObjectArray.GetObjectItemArrayUnsafe()[Iterator.Index++];
		// 如果遍历到的Object不可达，就取出来
		if (ObjectItem->IsUnreachable())
		{
			Iterator.Payload.Add(ObjectItem);
		}
		// 如果遍历超时了，就退出遍历
		if (Timer.IsTimeLimitExceeded())
		{
			return;
		}
	}
	// 遍历结束后，如果没有超时，就将遍历到的不可达Object都放入GUnreachableObjects数组中
	const bool bCompleted = !TimeLimitExceededFlag.load();
	if (bCompleted)
	{
		GGatherUnreachableObjectsState.Finish(GUnreachableObjects);
	}
}
```
1 遍历GUObjectArray数组，将其中不可达Object全部收集到GUnreachableObjects数组中。这里我们的TimeLimit时间限制是0，也就是说肯定会将不可达Object全部收集成功。但如果有时间限制，就有可能时间到了但还每收集完，如果是这样GUnreachableObjects数组还依然是空的，不会往里添加东西。
## 1 在hash表中移除信息
```cpp
bool UnhashUnreachableObjects(bool bUseTimeLimit, double TimeLimit)
{
	// 如果不可达Object还未收集完
	if (GGatherUnreachableObjectsState.IsPending())
	{
		const EGatherOptions GatherOptions = GetObjectGatherOptions();
		const double GatherTimeLimit = GIncrementalGatherTimeLimit > 0.0f ? GIncrementalGatherTimeLimit : TimeLimit;
		bTimeLimitReached = GatherUnreachableObjects(GatherOptions, bUseTimeLimit ? GatherTimeLimit : 0.0);
		if (!bTimeLimitReached)
		{
			if (bUseTimeLimit)
			{
				TimeLimit -= FMath::Min(TimeLimit, FPlatformTime::Seconds() - GCStartTime);
			}
		}
		else
		{
			return bTimeLimitReached;
		}
	}
	
	// GUnrechableObjectIndex这个变量是在收集不可达Object方法中初始化为0的
	const bool bFirstIteration = (GUnrechableObjectIndex == 0);
	// 不可达数组中的索引
	while (GUnrechableObjectIndex < GUnreachableObjects.Num())
	{
		// 处理不可达数组中的Object，调用其ConditionalBeginDestroy
		FUObjectItem* ObjectItem = GUnreachableObjects[GUnrechableObjectIndex++];
		{
			UObject* Object = static_cast<UObject*>(ObjectItem->Object);
			Object->ConditionalBeginDestroy();
		}
	}
}
bool UObject::ConditionalBeginDestroy()
{
	if( !HasAnyFlags(RF_BeginDestroyed) )
	{
		SetFlags(RF_BeginDestroyed);
		BeginDestroy();
	}
}

void UObject::BeginDestroy()
{
	// 设置自己的LinkerLoad为空
	SetLinker( NULL, INDEX_NONE );
	// 在hash表中移除自己及和自己相关的uclass，outer
	LowLevelRename(NAME_None);
	// 从hash表中移除package和自己的联系
	SetExternalPackage(nullptr);
}
```
1 最终调用了BeginDestroy的方法，目的就是再全局hash表中移除自己的信息。如果参数bUseTimeLimit是false，代表了没有时间限制，也就是肯定能将GUnreachableObjects数组中的不可达Object全部处理完，此时不可达索引GUnrechableObjectIndex就等于GUnreachableObjects数组的长度。但如果有了时间限制，就有可能时间到了导致没处理完，此时不可达索引GUnrechableObjectIndex就会小于GUnreachableObjects数组的长度，并且索引还标识了处理到哪个Object了。
## 2 增量清除垃圾
```cpp
void IncrementalPurgeGarbage(bool bUseTimeLimit, double TimeLimit)
{
	// 如果当前还在移除hash引用
	if (IsIncrementalUnhashPending())
	{
		bTimeLimitReached = UnhashUnreachableObjects(bUseTimeLimit, TimeLimit);
	}
	if (!bTimeLimitReached)
	{
		bCompleted = IncrementalDestroyGarbage(bUseTimeLimit, TimeLimit);
	}
}
```

# 问题

- FProperty类的ArrayDim属性是什么含义？
用于表示该属性是一个 **静态数组（Static Array）** 的大小，也就是用UPROPERTY修饰的数组。类似于int Test[10]。
- GReachabilityState.GetNumIterations()这是什么含义？
可达性分析的次数，看上去就是一个计数器，在每次可达性分析（FReachabilityAnalysisState::PerformReachabilityAnalysis）之后会++。
- gc的标记流程是什么样的呢？
- 如何记录UObject之间的引用关系呢？
- gc是如何清除的？
- UE5的增量标记是怎么实现的，如何追踪指针引用关系改变的
- gc多线程是怎么使用的
- FUObjectHashTables这个hash桶是干嘛的
- LinkerLoad是每个UObject都有啊


