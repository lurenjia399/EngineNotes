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
bool TryCollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	UE::GC::PreCollectGarbageImpl<true>(ObjectKeepFlags);
	
	CollectGarbageImpl<true>(KeepFlags);
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
	// 开始可达性分析，就好像一个初始化的过程
	StartReachabilityAnalysis(KeepFlags, Options);
	do
	{
		PerformReachabilityAnalysisPass(Options);
	} while ((VerseGCActive() || !Private::GReachableObjects.IsEmpty() || !Private::GReachableClusters.IsEmpty()) && !GReachabilityState.IsSuspended());
}
```
3 分成了三部分，第一部分StartReachabilityAnalysis
## StartReachabilityAnalysis
```cpp
void StartReachabilityAnalysis(EObjectFlags KeepFlags, const EGCOptions Options)
{
	// 将继承自GCObject的对象放到InitialReferences数组中
	BeginInitialReferenceCollection(Options);
	// 将簇中的Object都标记为可达，簇中若含有垃圾则解散簇并将簇中Object添加到InitialObjects数组中。将根Object也都标记为可达，并将根Object添加到InitialObjects数组中
	MarkObjectsAsUnreachable(KeepFlags);
}
```
1 StartReachabilityAnalysis 又分为了两部分，第一部分通过工作线程将我们GCObject放到InitialReferences数组中。第二部分将簇中的Object都标记为可达，簇中若含有垃圾则解散簇并将簇中Object添加到InitialObjects数组中。将根Object也都标记为可达。我们主要看下第二部分。
```cpp

FORCENOINLINE void MarkObjectsAsUnreachable(const EObjectFlags KeepFlags)
{
	// 如果没有引用垃圾
	if (!Stats.bFoundGarbageRef)
	{
		// 交换可达标记和可能不可达标记，这边直接交换的是标志位，也就是变量名是GReachableObjectFlag可达的，但是代表的数据是不可达的
		Swap(GReachableObjectFlag, GMaybeUnreachableObjectFlag);
	}
	// 处理簇，先将簇中根Object和簇中Object标记为可达。
	MarkClusteredObjectsAsReachable(GatherOptions, InitialObjects);
	// 方法和处理簇的方法一样，将根Object标记为可达。
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
	DissolveClusterAndMarkObjectsAsUnreachable，解散簇的方法。第一步设置簇中object的所属簇索引为0，并将object标记为可能不可达（这里虽然标记为可能不可达，但是之前有交换所以这里实际标记成可达了，为啥呢？）。第二步解散掉簇引用的其他簇，递归调用。
	*/
	for (FUObjectItem* ObjectItem : MarkClustersResults.ClustersToDissolve)
	{
		if (ObjectItem->HasAnyFlags(EInternalObjectFlags::ClusterRoot))
		{
		GUObjectClusters.DissolveClusterAndMarkObjectsAsUnreachable(ObjectItem);
		GUObjectClusters.SetClustersNeedDissolving();
		}
	}
	/* 如果簇根是根Object或者是跳过gc，簇中其余Object是根或者跳过gc，则就需要保留簇。
	MarkReferencedClustersAsReachable：把簇中引用的object都标记为可达。第一步处理簇引用的其他簇根，如果其他簇根不是垃圾，就标记为可达，如果是垃圾就清掉引用。第二步处理簇中MutableObjects（不在簇中但依然被簇引用。看上去是object属于多个簇，但只在一个簇中这种情况。），如果Object不是垃圾并且不是可达的就标记为可达，并将其添加到InitialObjects数组中，如果是垃圾就清掉引用。第三步，如果簇中有垃圾（无论是引用的簇根还是mutableObjects中有垃圾），我们就把簇中Object都添加到InitialObjects数组中，并解散簇。
	*/
	for (FUObjectItem* ObjectItem : MarkClustersResults.KeepClusters)
	{
		MarkReferencedClustersAsReachable<EGCOptions::None>(ObjectItem->GetClusterIndex(), OutRootObjects);
	}
}
```
2 MarkObjectsAsUnreachable这个方法是标记所有的Object为可能不可达，里面首先交换可达和可能不可达标记，然后就调用标记可达的相关方法来实现可能不可达标记。

## PerformReachabilityAnalysisPass

```cpp
void PerformReachabilityAnalysisPass(const EGCOptions Options)
{
	if (!Private::GReachableObjects.IsEmpty())
	{
		Private::GReachableObjects.PopAllAndEmpty(InitialObjects);
		ConditionallyAddBarrierReferencesToHistory(*Context);
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


