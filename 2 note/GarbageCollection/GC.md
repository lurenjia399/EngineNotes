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

	// 三个可达标志位
	ReachabilityFlag0 = 1 << 0,//可达的Object
	ReachabilityFlag1 = 1 << 1,//不可达的Object
	ReachabilityFlag2 = 1 << 2,//可能不可达的Object
	
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
// 这块代码删减了挺多的还
bool TryCollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	UE::GC::PreCollectGarbageImpl<true>(ObjectKeepFlags);
	
	CollectGarbageImpl<true>(KeepFlags);

	// 增加可达性分析次数，就像一个计数器
	FinishIteration();
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
	// 开始可达性分析，标记可能不可达标记
	StartReachabilityAnalysis(KeepFlags, Options);
	do
	{
		PerformReachabilityAnalysisPass(Options);
	} while ((VerseGCActive() || !Private::GReachableObjects.IsEmpty() || !Private::GReachableClusters.IsEmpty()) && !GReachabilityState.IsSuspended());
}
```
3 分成了三部分，第一部分StartReachabilityAnalysis
# 3 标记初始化 StartReachabilityAnalysis
```cpp
void StartReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
{
	// 用另一个线程将继承自GCObject的对象放到InitialReferences数组中
	BeginInitialReferenceCollection(Options);
	// 标记Object为不可达
	MarkObjectsAsUnreachable(KeepFlags);
}
```
1 StartReachabilityAnalysis 又分为了两部分，第一部分通过工作线程将我们GCObject放到InitialReferences数组中。第二部分将簇中的Object都标记为可达（可达和可能不可达交换了,所以代表不可达），簇中若含有垃圾则解散簇并将簇中Object添加到InitialObjects数组中，然后还将根Object添加到InitialObjects数组中。我们主要看下第二部分。
```cpp

FORCENOINLINE void MarkObjectsAsUnreachable(const EObjectFlags KeepFlags)
{
	// 如果没有引用垃圾
	if (!Stats.bFoundGarbageRef)
	{
		// 交换可达标记和可能不可达标记，这边直接交换的是标志位，也就是变量名是GReachableObjectFlag可达的，但是代表的数据是不可达的
		Swap(GReachableObjectFlag, GMaybeUnreachableObjectFlag);
	}
	// 处理簇，先将簇中根Object和簇中Object标记为可达（可达和可能不可达交换了,所以代表不可达），如果簇中有垃圾，我们就把簇中Object都添加到InitialObjects数组中并解散簇。
	MarkClusteredObjectsAsReachable(GatherOptions, InitialObjects);
	// 方法和处理簇的方法一样，将根Object标记为可达（可达和可能不可达交换了,所以代表不可达），然后将根Object添加到InitialObjects数组中。
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
			// 处理簇的根Object。如果簇根是根或者跳过gc则标记为可达。这里虽然是可达标记但实际是可能不可达，因为前边交换过可达和可能不可达
		   bool bKeepCluster=RootItem->HasAnyFlags(EInternalObjectFlags_RootFlags);
			if (bKeepCluster)
			{
				RootItem->FastMarkAsReachableInterlocked_ForGC();
				ThreadState.Payload.KeepClusters.Add(RootItem);
			}
			// 处理簇中的Object。和处理簇的根Object同理
			for (int32 ObjectIndex : Cluster.Objects)
			{
				FUObjectItem* ClusteredItem = &GUObjectArray.GetObjectItemArrayUnsafe()[ObjectIndex];
				ClusteredItem->FastMarkAsReachableAndClearReachaleInClusterInterlocked_ForGC();
				if (!bKeepCluster && ClusteredItem->HasAnyFlags(EInternalObjectFlags_RootFlags))
				{
					ThreadState.Payload.KeepClusters.Add(RootItem);
					bKeepCluster = true;
				}
			}
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
	MarkReferencedClustersAsReachable：把簇中引用的object都标记为可达。第一步处理簇引用的其他簇根，如果其他簇根不是垃圾，就标记为可达（交换后的代表可能不可达），如果是垃圾就清掉引用。第二步处理簇中MutableObjects（不在簇中但依然被簇引用。看上去是object属于多个簇，但只在一个簇中这种情况。），将其中Object标记为可达（交换后的代表可能不可达），将从可达变为可能不可达的Object添加到ObjectsToSerialize数组中。第三步，如果簇中有垃圾（无论是引用的簇根还是mutableObjects中有垃圾），我们就把簇中Object都添加到InitialObjects数组中，并解散簇。
	*/
	for (FUObjectItem* ObjectItem : MarkClustersResults.KeepClusters)
	{
		MarkReferencedClustersAsReachable<EGCOptions::None>(ObjectItem->GetClusterIndex(), OutRootObjects);
	}
}
```
2 MarkObjectsAsUnreachable这个方法是标记所有的Object为可能不可达，里面首先交换可达和可能不可达标记，然后就调用标记可达的相关方法来实现可能不可达标记。

# 4 通过引用关系修改标记 PerformReachabilityAnalysisPass

```cpp
void PerformReachabilityAnalysisPass(const EGCOptions Options)
{
	if (!Private::GReachableObjects.IsEmpty())
	{
		Private::GReachableObjects.PopAllAndEmpty(InitialObjects);
		ConditionallyAddBarrierReferencesToHistory(*Context);
	}
	// 第一次进行可达性分析 || (没有发现引用垃圾 && 没有暂停可达性分析)
	else if (GReachabilityState.GetNumIterations() == 0 || (Stats.bFoundGarbageRef && !GReachabilityState.IsSuspended()))
	{
		// 目的是将GGCObjectReferencer数组中的值添加到InitialReferences数组中,然后返回
		Context->InitialNativeReferences = GetInitialReferences(Options);
	}
	// 把InitialObjects数组里的值塞到Context里
	Context->SetInitialObjectsUnpadded(InitialObjects);
	if (!Private::GReachableClusters.IsEmpty())
	{
		TArray<FUObjectItem*> KeepClusterRefs;
		Private::GReachableClusters.PopAllAndEmpty(KeepClusterRefs);
		for (FUObjectItem* ObjectItem : KeepClusterRefs)
		{
			MarkReferencedClustersAsReachable<EGCOptions::None>(ObjectItem->GetClusterIndex(), InitialObjects);
		}
	}
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
		// TReachabilityProcessor<Options> Processor;
		// CollectReferencesForGC<TReachabilityCollector<Options>>(Processor, Context);
		// 这里是获取Dispatcher，GetDispatcher的定义有两个地方，一个是默认的，一个是TReachabilityCollector类里的，后者是全特化版本。实际执行会根据Collector类型的不同执行不同的版本，TReachabilityCollector类型执行后者，TDefaultCollector类型执行前者。这里Collector的类型是TReachabilityCollector，所以Dispatcher是TBatchDispatcher类型。
		CollectorType Collector(Processor, Context);
		decltype(GetDispatcher(Collector, Processor, Context)) Dispatcher = GetDispatcher(Collector, Processor, Context);
		// 
		for (UObject** InitialReference : Context.InitialNativeReferences)
		{
			Dispatcher.HandleKillableReference(*InitialReference, EMemberlessId::InitialReference, EOrigin::Other);
		}
		
		
		// 这个Dispatcher的类型是TDirectDispatcher<TReachabilityProcessor<Parallel>>
		Dispatcher.HandleKillableReference(*InitialReference, EMemberlessId::InitialReference, EOrigin::Other);
	}
}

```
1 判断Object之间的引用关系，通过HandleKillableReference方法
```cpp
// TBatchDispatcher 类中的方法
void HandleKillableReference(UObject*& Object, FMemberId MemberId, EOrigin Origin)
{
	QueueReference(Context.GetReferencingObject(), Object, MemberId, ProcessorType::MayKill(Origin, true));
}
void QueueReference(const UObject* ReferencingObject,  UObject*& Object, FMemberId MemberId, EKillable Killable)
{
	// 蓝图类型为true，其他类型为false
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
	// 会往UnvalidatedReferences数组里添加，直到添加满了就执行DrainUnvalidatedFull方法，这个是核心方法，会遍历查看是否有引用
	UnvalidatedReferences.Push(UnvalidatedReference);
	if (UnvalidatedReferences.IsFull())
	{
		DrainUnvalidatedFull();
	}
}
```
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
```
# 问题

- GReachabilityState.GetNumIterations()这是什么含义？
可达性分析的次数，看上去就是一个计数器，在每次可达性分析（FReachabilityAnalysisState::PerformReachabilityAnalysis）之后会++。
- 


