https://zhuanlan.zhihu.com/p/710512404 wp创建流程
https://zhuanlan.zhihu.com/p/708136867 生成sell流程
https://zhuanlan.zhihu.com/p/687020988
https://zhuanlan.zhihu.com/p/502053365
https://hebohang.github.io/p/worldpartition%E8%A7%A3%E6%9E%90/
# 1 WorldPasrtition初始化
```cpp
// 在WorldSetting中wp被标记成了Instanced，看上去就是在蓝图实例化WorldSetting的时候就会实例化WorldPartition。
/*
	比较好玩的事情，如果把VisibleAnywhere和 Instanced 一起使用的话，他不会显示成树状结构，而是直接影藏掉property本身，把他的子树的properties合并到外面来。意思就是WorldPasrtition中被标记visible的属性可以直接在蓝图中看到
*/
class AWorldSettings : public AInfo, public IInterface_AssetUserData
{
	UPROPERTY(VisibleAnywhere, Category=WorldPartitionSetup, Instanced)
	TObjectPtr<UWorldPartition> WorldPartition;
}

```
1 打开编辑器下的wp是在序列化worldsetting的时候序列化进来的。运行编辑器下的wp也是在序列化worldsetting的时候序列化进来的。
1 在编辑器模式下，我们在蓝图实例化worldsetting的时候也会实例化wp，然后把wp保存到WorldSetting中，而WorldSetting会作为level里actors的第一个元素序列化到map文件中，所以在exe模式下序列化map文件后，也就会自然的创建出wp了。
# 2 Actor的替身，WorldPartitionActorDesc的创建

```cpp
void UWorldPartition::Initialize()
->UActorDescContainerInstance* RegisterActorDescContainerInstance(...)
 ->void UActorDescContainerInstance::Initialize(...)
  ->void RegisterContainer(...)
  ->FWorldPartitionActorDescInstance* AddActorDescInstance(...)
```
1 在wp初始化的时候，我们会通过RegisterActorDescContainerInstance方法来创建注册出ActorDescContainerInstance这个对象。
2 在ActorDescContainerInstance这个对象初始化的时候会首先通过RegisterContainer方法来创建出ActorDescContainer，而ActorDescContainer这个的初始化会遍历当前地图中的actor，对其生成WorldPartitionActorDesc。
3 在RegisterContainer方法之后，就已经创建出ActorDescContainer并且里面的值都是WorldPartitionActorDesc。然后就通过AddActorDescInstance方法遍历ActorDescContainer对里面的WorldPartitionActorDesc都生成对应的ActorDescInstance并添加到ActorDescContainerInstance中。
4 WorldPartitionActorDescInstance和WorldPartitionActorDesc的区别看上去就是，WorldPartitionActorDesc表示actor的静态数据，名称路径packagename啥的，而WorldPartitionActorDescInstance是包含一些动态数据，actor的指针引用啥的。
```cpp
class FWorldPartitionActorDescInstance
{
	TObjectPtr<UActorDescContainerInstance>		ContainerInstance;

	mutable uint32								SoftRefCount;
	mutable uint32								HardRefCount;
	TOptional<FDataLayerInstanceNames>			ResolvedDataLayerInstanceNames;
	bool										bIsForcedNonSpatiallyLoaded;
	bool										bIsRegisteringOrUnregistering;
	mutable FText*								UnloadedReason;
	mutable int32								AsyncLoadID;

	// Instancing Path if set
	TOptional<FSoftObjectPath>					ActorPath;

	mutable TWeakObjectPtr<AActor>				ActorPtr;
	FWorldPartitionActorDesc*					ActorDesc;

	TObjectPtr<UActorDescContainerInstance>		ChildContainerInstance;
}

class FWorldPartitionActorDesc
{
	FGuid							Guid;
	FTopLevelAssetPath				BaseClass;
	// Not serialized, comes from initialization data
	FTopLevelAssetPath				NativeClass;	
	// Not serialized, comes from initialization data
	FName							ActorPackage;
	// Not serialized,comes from initialization data
	FSoftObjectPath					ActorPath;
	FName							ActorLabel;
	FTransform						ActorTransform;
	FVector							BoundsLocation;
	FVector							BoundsExtent;
	// 还有很多
}
```
# 3 生成StreamingCell
```cpp
void UWorldPartition::OnBeginPlay()
{
	// 生成参数，定义一个结构体
	FGenerateStreamingParams Params = FGenerateStreamingParams()
	// 生成上下文，定义结构体
	TArray<FString> OutGeneratedLevelStreamingPackageNames;
	FGenerateStreamingContext Context = FGenerateStreamingContext()
		.SetLevelPackagesToGenerate(&OutGeneratedLevelStreamingPackageNames}
	// 生成方法
	GenerateStreaming(Params, Context);
	// 将levelpackage添加到GeneratedLevelStreamingPackageNames成员变量中
	for (const FString& PackageName : OutGeneratedLevelStreamingPackageNames)
	{
		FString Package = PackageName;
		GeneratedLevelStreamingPackageNames.Add(Package);
	}
	
	RuntimeHash->OnBeginPlay();
	ExternalDataLayerManager->OnBeginPlay();
}
```
1 在编辑器下运行游戏，会执行OnBeginPlay方法，这里面是开始执行生成cell的开始
```cpp
bool UWorldPartition::GenerateContainerStreaming(const FGenerateStreamingParams& InParams, FGenerateStreamingContext& InContext)
{
	StreamingGenerator.PreparationPhase(InParams.ContainerInstanceCollection);
}
```
2 会执行PreparationPhase方法
2.1 这个里面首先通过CreateActorDescViewMap方法，创建了FStreamingGenerationActorDescView，这个里面的值就是FWorldPartitionActorDescInstance，是代表运行时的ActorDesc数据。
2.2 然后执行ResolveContainerDescriptor方法，遍历ActorDescViewMap中所有的ActorDescView，对其执行以下操作：
- 如果wp没有开启stream，也需要设置world中ActorDescView不能stream。
- 如果wp没有开启stream，也需要设置world中ActorDescView的bIsForcedNoRuntimeGrid为true。
- 设置world中ActorDescView的ResolvedDataLayerInstanceNames和RuntimeDataLayerInstanceNames，也就是actor所在的datalayer
- 设置world中ActorDescView的RuntimedHLODLayer，取得时worldsetting蓝图中设置的
- 设置world中ActorDescView的parentView，就是actor的attach到的那个actor，作为parentview
2.3 然后执行ValidateContainerDescriptor方法，拿到map中所有静态actor的guid。执行大量验证，检查所有Actor的设置与WorldPartition的设置有没有冲突，这里可能会强行改变一些ActorView的设置。这里还会处理Actor的引用，处于一套引用链下的Actor都应该在同一DataLayers、GridName下。还会区分时编辑器下的引用还是runtime时的引用
2.4 然后执行UpdateContainerDescriptor方法，通过GenerateObjectsClusters来生成Cluster，因为Actor之间有强引用或者父子关系，则必须出现在同一个Cell，不然这个引用关系就会被打破。所以这里将有引用关系的Actor组合在一起，视为一个划分Cell的基本单位 —— Cluster。如果一个Actor没有引用任何别的Actor，也没被别的Actor引用，那这一个Actor就会单独处于一个Cluster中。
2.5 然后计算bounds，将所有的ActorDesc的Bound都相加得出。

```cpp
bool UWorldPartition::GenerateContainerStreaming(const FGenerateStreamingParams& InParams, FGenerateStreamingContext& InContext)
{
	StreamingGenerator.PreparationPhase(InParams.ContainerInstanceCollection);

	auto GenerateRuntimeHash = [this](...)
	{
		// 组装UWorldPartitionStreamingPolicy，也就是生成ActorSetContainerInstance和ActorSetInstance，ActorSet是代表一簇相关的actor，而ActorSetContainerInstance是ActorSetInstance容器
		StreamingPolicy = NewObject<UWorldPartitionStreamingPolicy>();
		FStreamingGenerationContextProxy GenerationContextProxy(StreamingGenerator.GetStreamingGenerationContext(InParams.ContainerInstanceCollection));
		GenerationContextProxy.SetActorSetInstanceFilter([ExternalDataLayerAsset = InActorDescContainerInstance->GetExternalDataLayerAsset()](const IStreamingGenerationContext::FActorSetInstance& InActorSetInstance) { return (InActorSetInstance.GetExternalDataLayerAsset() == ExternalDataLayerAsset); });

		// 生成cell的具体流程，其中RuntimeHash也会随着wp的序列化而序列化，如果在worldsetting面板中改变RuntimeHashclass，就会改变RuntimeHash，具体逻辑在FWorldPartitionDetails::HandleWorldPartitionRuntimeHashClassChanged方法中。
		if (RuntimeHash->GenerateStreaming(StreamingPolicy, &GenerationContextProxy, InContext.PackagesToGenerate))
		{
			StreamingPolicy->SetContainerResolver(StreamingGenerator.GetContainerResolver());
			StreamingPolicy->PrepareActorToCellRemapping();
			StreamingPolicy->SetShouldMergeStreamingSourceInfo(RuntimeHash->GetShouldMergeStreamingSourceInfo());
			return true;
		}
		return false;
	};

	const UActorDescContainerInstance* BaseContainerInstance = InParams.ContainerInstanceCollection.GetBaseContainerInstance();
	const bool bBaseContainerGenerationSuccess = GenerateRuntimeHash(BaseContainerInstance);
}

```
3.1 在开始生成Cell前，为了将生成过程与ActorDesc的具体组织方式解耦，这里需要用前面生成的ActorDescView、Cluster信息，构建一个IStreamingGenerationContext。它将Cluster构建成一个个FActorSetInstance表示一组互相有引用，RuntimeGrid、DataLayer一致的Actor，且必须处于同一个Cell。具体构建过程在FWorldPartitionStreamingGenerator::GetStreamingGenerationContext()构造函数中。
## 3.2 UWorldPartitionRuntimeHashSet
具体生辰cell流程，也就是GenerateStreaming方法。首先就是生成PersistentPartitionDesc
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250504181724.png)
3.3 下面就是将所有FActorSetInstance分配到不同的URuntimePartition。
- bIsSpatiallyLoaded=false的ActorSetInstance会被分配到PersistentPartition。
- RuntimeGrid=None，分配到第一个我们配置的RuntimePartitions。
- RuntimeGrid还可以是GridName：HLODLayerObjectName这种形式。则这个Actor会被分配到`GridName`下的HLODSteps配置`HLODLayerObjectName`对应的`PartitionLayer`。
	- 生成的HLODActor会自动把GridName配置成这种形式。
	- 如果Grid没有这个HLOD配置，这个ActorSet会被直接忽略！！！
	- 如果HLODActor上配置的HLODLayer在对应的WorldPartition上不存在，则在前面的验证中就会报错。
### 3.21 URuntimePartitionLHGrid
3.4 经过3.3得到了一组`[URuntimePartition <-> TArray<FActorSetInstance>]`。对所有Pair调用`URuntimePartition::GenerateStreaming`，生成`URuntimePartition::FCellDesc`，每个CellDesc都包含一组`FActorSetInstance`。注意这里根据配置，有不同的`URuntimePartition`子类，实现了不同的Cell划分方式，其中最常用的是URuntimePartitionLHGrid，它的划分方式最复杂，也最常用。
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250504192533.png)
`URuntimePartitionLHGrid`本质上是把整个World划分成具有MipLevel等级的3D格子。
计算某个`ActorSet`属于哪个`Cell`时，根据这组`ActorSet`的Bound计算它应该所处的Cell。输出为`FCellCoord`:
```cpp
struct FCellCoord
{
	int64 X;
	int64 Y;
	int64 Z;
	int32 Level;
}
```
这里的Level类似于MipLevel，以ActorBound的最大长度的Extent计算Level:
```cpp
static inline int32 GetLevelForBox(const FBox& InBox, int32 InCellSize)
{
	const FVector Extent = InBox.GetExtent();
	const FVector::FReal MaxLength = Extent.GetMax() * 2.0;
	return FMath::CeilToInt32(FMath::Max<FVector::FReal>(FMath::Log2(MaxLength / InCellSize), 0));
}
```
通过对数来划分等级：
- 如果ActorSet的Bounds的最大边长 < CellSize，level就是0。
- CellSize < 如果ActorSet的Bounds的最大边长 < CellSize * 2，level就是1。
- CellSize * 2 < 如果ActorSet的Bounds的最大边长 < CellSize * 4，level就是2。
- 以此类推。
然后以ActorBound的中心点坐标计算所处的Cell坐标，这里没有别的处理，就是把整个3D World划均匀分成CellSizeForLevel大小的三维格子，Bound中心点的坐标除以CellSizeForLevel向下取整，得到Bound中心点所在的格子，作为这个ActorSet最终划分到的格子：
```cpp
static inline FCellCoord GetCellCoords(const FVector& InPos, int32 InCellSize, int32 InLevel)
{
	check(InLevel >= 0);
	const int64 CellSizeForLevel = (int64)InCellSize * (1LL << InLevel);
	return FCellCoord(
		FMath::FloorToInt(InPos.X / CellSizeForLevel),
		FMath::FloorToInt(InPos.Y / CellSizeForLevel),
		FMath::FloorToInt(InPos.Z / CellSizeForLevel),
		InLevel
	);
}
```
3.5 经过3.4划分好Cell之后，我们得到一组`[URuntimePartition <-> FCellDesc]`，其中FCellDesc包含了一组ActorSet，FCellDesc没有考虑处于不同DataLayer的ActorSet的情况，所以再将FCellDesc重新划分成FCellDescInstance。
> 如果一个`FCellDesc`的ActorSet处于不同DataLayer，其它参数保持一致的情况下分裂成两个`FCellDescInstance`，仅DataLayer不同。

4 经过3.5我们得到了一组`[RuntimePartition <-> CellDescInstances]`，然后遍历对每一个FCellDescInstance创建UWorldPartitionRuntimeCell，这里我们创建具体的是UWorldPartitionRuntimeLevelStreamingCell。并对每一个处于这个Cell中的FActorInstance调用UWorldPartitionRuntimeCell::AddActorToCell，构建对应的TArray<`FWorldPartitionRuntimeCellObjectMapping`> Packages;每个Package都是一个Actor的路径引用，用于最终生成UWorldPartitionLevelStreamingDynamic，实现通过流关卡加载卸载Cell。然后通过将UWorldPartitionRuntimeLevelStreamingCell保存到UWorldPartitionRuntimeHashSet::RuntimeStreamingData数据中
## 3.5 UWorldPartitionRuntimeSpatialHash
另一种划分场景的方式是通过UWorldPartitionRuntimeSpatialHash来划分。
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250505160607.png)
1 用能够容纳下所有actor的尺寸来除以配置CellSize来计算出需要分多少个格子，这里乘2是因为二维的有x轴上的格子和y轴上的格子，赋值给GdridSize。然后取对数+1表示能够划分成多少个等级。
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250505160827.png)
2 接下来就是场景的划分并记录，记录每个格子的划分原点，格子的尺寸，格子的数量，格子所处的等级。level越高，格子数量呈2倍减少，格子尺寸成2倍增加。在这种情况每上一个等级格子的边界和下一级的边界都是一样的，那么处于边界上的actorset就不知道怎么划分了，所以有了bUseAlignedGridLevels标志位，在false的情况下，每一级的格子数量会加1，也就是变相加了偏移。
3 接下来划分有效的场景格子，也就是那些放ActorSet的格子都是哪几个。
- 如果`UWorldPartitionRuntimeSpatialHash`的设置上勾选了`bPlaceSmallActorsUsingLocation`，就用的是ActorSet的Bound中心位置(不考虑Z坐标)，计算它所处的Cell坐标，也就是位于哪个场景格子中。
- 如果上面条件不满足，就会遍历所有level的场景，看当前actorSet的Bound和当前level场景中有几个格子相交，如果是一个格子相交，那么就说明当前actorSet要放到这个格子里。相交的计算就是通过actorSet的Bound最小值和最大值是否能放在同一个格子中判断的，具体在GetNumIntersectingCells方法中。

# 运行时StreamingCell切换
## 1 初始化

![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250430151636.png)
2 Initialize方法，会从world的初始化流程中调用过来。
```cpp
void UWorldPartition::Initialize(UWorld* InWorld, const FTransform& InTransform)
{
	// 赋值world
	World = InWorld;
	// 状态枚举，正在初始化
	InitState = EWorldPartitionInitState::Initializing;
	if (bIsEditor || bIsGame || bIsPIEWorldTravel || bIsDedicatedServer)
	{
		FName ContainerPackageName = UActorDescContainerInstance::GetContainerPackageNameFromWorld(OuterWorld);
		ActorDescContainerInstance = RegisterActorDescContainerInstance(UActorDescContainerInstance::FInitializeParams(ContainerPackageName));
		CreateAndInitializeDataLayerManager();
		InitializeActorDescContainerEditorStreaming(ActorDescContainerInstance);
	}
	// 初始化datalayermanager
	DataLayerManager = NewObject<UDataLayerManager>(this, TEXT("DataLayerManager"), RF_Transient);
	DataLayerManager->Initialize();
	// 状态枚举，初始化完成
	InitState = EWorldPartitionInitState::Initialized;
}
```
## Streaming流程
```cpp
void UWorld::InternalUpdateStreamingState()
{
	ProcessLevelStreamingVolumes();

	// Update WorldComposition required streaming levels
	if (WorldComposition)
	{
		WorldComposition->UpdateStreamingState();
	}

	// Update World Subsystems required streaming levels
	const TArray<UWorldSubsystem*>& WorldSubsystems = SubsystemCollection.GetSubsystemArray<UWorldSubsystem>(UWorldSubsystem::StaticClass());
	for (UWorldSubsystem* WorldSubsystem : WorldSubsystems)
	{
	    WorldSubsystem->UpdateStreamingState();
	}
}
```
在UWorld::Tick中，会调用UWorld::InternalUpdateStreamingState()，其中对所有WorldSubsystem调用其UpdateStreamingState，这里自然会调到UWorldPartitionSubsystem中。
```cpp
void UWorldPartitionSubsystem::UpdateStreamingState()
{
	UWorldPartitionSubsystem::UpdateStreamingStateInternal(GetWorld());
}
```
1 在UpdateStreamingStateInternal方法中首先是更新StreamingSource，通过UWorldPartitionSubsystem::UpdateStreamingSources方法：这里主要逻辑就是从每个`StreamingSourceProviders`中获取`FWorldPartitionStreamingSource`。`StreamingSourcesVelocity`保存了每个Source的历史速度信息，这里随后会更新`FWorldPartitionStreamingSource`的速度。
只要实现了`IWorldPartitionStreamingSourceProvider`，并主动把自己注册到`UWorldPartitionSubsystem`，就会被考虑进来。`UWorldPartitionStreamingSourceComponent`实现了这个接口，可以直接使用。`APlayerController`也实现了这个，其中有大量关于流送的设置，有选项可以开关，也判断了只有LocalController才会生成Source。如果是Server，则所有PlayerController都会生成`StreamingSource`:
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250505181552.png)
然后在playercontroller上也可以设置streamSource的各种状态，比如形状，TargetState等
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250505181804.png)
默认情况下，StreamingSource代表的加载范围是一个球形，以自己的位置为原点，Partition的LoadingRange为半径。判断与Cell中content的Bounds（不是cell的bounds是里面actor组成的bounds）是否相交用的是FMath::SphereAABBIntersection()，其实现的逻辑是只要计算球心不在立方体中，球心距立方体边界平方和的大小和半径平方和大小就行。
2 接下来是UpdateStreamingState：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250505183402.png)
这里处理了Server和Clinet的不同，还有是否开启了ServerStreaming。如果是Server，默认没有开启流送，一开始就认为所有Cell都需要考虑流送。如果是Client则会对每一个`StreamingSrouce`都判断与他相交的Cell，只有这些相交的Cell需要考虑流送。
3 UWorldPartitionStreamingPolicy::UpdateStreamingPerformance这个方法是在UpdateStreamingState这个方法里执行的，主要目的就是性能判断，性能的预计算是在UpdateStreamingState方法中，计算出streamingsource距离cell的距离，以及距离和streamingsource半径的比例。然后在UpdateStreamingPerformance方法中判断，
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250506175724.png)
`GBlockOnSlowStreamingRatio`默认是0.25，`GBlockOnSlowStreamingWarningFactor`是2。如果Source的加载半径是256，则`StreamingSource.Center`离Cell 8m 时如果还没加载好，就会被判定为性能是`Critical`。11.3m 时还没加载好就会被判断为Slow。
在`UWorldPartitionStreamingPolicy::UpdateStreamingState()`的最后，针对当前帧判定需要激活的Cell，调用`UpdateStreamingPerformance(FrameActivateCells)`更新性能。只要有一个Cell是`Critical`，就阻塞加载，通过将World的`World->bRequestedBlockOnAsyncLoading`属性设为true实现。
最终在`UWorld::BlockTillLevelStreamingCompleted()`中处理。里面会再执行一遍`UWorldPartitionSubsystem::UpdateStreamingStateInternal`，由于之前把性能状态设为了`Critical`，这一次会跳过所有不能阻塞加载的Cell(通常配置在对应的PartitionGrid上)，以最大程度上减少这次阻塞加载需要的时间。
4 UWorldPartitionStreamingPolicy::UpdateStreamingState()处理完后，ToActivateCells中是当前状态下所有应该激活的Cell，ToLoadCells是所有应该是Load状态的Cell。所有不在Load或Activate的状态的Cell都应该卸载，通过对比它们与ActivatedCells、LoadedCells，ToXXX中没有的Cell就该被卸载掉，直接调用UWorldPartitionRuntimeLevelStreamingCell::Unload()。对于Server，如果没有开启流送，则直接认为所有Cell都是应该加载的，但还是会处理DataLayer的可见性。默认情况下，在Server上，所有PlayerController Source都会被注册进来。此外，对于ClientVisible的Cell，Server有特殊处理，开启流送的情况下如果Server考虑卸载一个Cell，会尊重Client的状态，如果Client的这个Cell还是可见的，则先不卸载它。
5 处理activate的cell：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250505191053.png)
这个激活的过程就是根据cell创建UWorldPartitionLevelStreamingDynamic，然后设置bShouldBeLoaded，bShouldBeVisible相关标志位实现。
4 处理load的cell：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250505191650.png)
逻辑部分就是和activate一样了。


```cpp
void UWorldPartitionSubsystem::UpdateStreamingStateInternal(const UWorld* InWorld, UWorldPartition* InWorldPartition)
{
	ON_SCOPE_EXIT
	{
		WorldPartitionSubsystem->OnStreamingStateUpdated().Broadcast();
	};
	
	if (WorldPartitionSubsystem)
	{
		// Update streaming sources，更新世界的StreamingSource数组
		WorldPartitionSubsystem->UpdateStreamingSources();
		// Update server's clients visible levels，在服务器上更新，客户端可见的Level
		WorldPartitionSubsystem->UpdateServerClientsVisibleLevelNames();
	}
}
```


# Hlod创建流程
如需生成HLOD代理网格体，需将Actor添加至HLOD层，然后将其 **移动性（Mobility）** 设置为 **静态（Static）**，并告知Actor生成HLOD。方法是使用 **构建（Build） > 构建HLOD（Build HLODs）** 或者使用 **WorldPartitionHLODsBuilder** commandlet。
## 1 build

```cpp
// 起始位置
void FLevelEditorActionCallbacks::BuildHLODs_Execute()
{
	// Build HLOD
	FEditorBuildUtils::EditorBuild(GetWorld(), FBuildOptions::BuildHierarchicalLOD);
}

// 中间的流程省略了，这里就展示下build过程
{
	bDoBuild = GEditor->WarnAboutHiddenLevels( InWorld, false );
	if ( bDoBuild )
	{
		GEditor->ResetTransaction( NSLOCTEXT("UnrealEd", "BuildHLODMeshes", "Building Hierarchical LOD Meshes") );

		// We can't set the busy cursor for all windows, because lighting
		// needs a cursor for the lighting options dialog.
		const FScopedBusyCursor BusyCursor;

		if (InWorld->IsPartitionedWorld())
		{
			bShouldMapCheck = false;
			bDirtyPersistentLevel = false;
		}

		TriggerHierarchicalLODBuilder(InWorld);
	}
}
// 最终走到了FWorldPartitionEditorModule::Build方法里面
```
## 2 WorldPartitionHLODsBuilder commandlet

```cpp
bool UWorldPartitionHLODsBuilder::RunInternal(UWorld* InWorld, const FCellInfo& InCellInfo, FPackageSourceControlHelper& PackageHelper)
{
	if (bRet && ShouldRunStep(EHLODBuildStep::HLOD_Setup))
	{
		bRet = SetupHLODActors();
	}
	if (bRet && ShouldRunStep(EHLODBuildStep::HLOD_Build))
	{
		bRet = BuildHLODActors();
	}
}
```
### HLOD_Setup
1 根据命令行的参数来执行不同的操作，下面看下SetupHLODActors方法
```cpp
bool UWorldPartitionHLODsBuilder::SetupHLODActors()
{
	WorldPartition->SetupHLODActors(SetupHLODActorsParams);
}
void UWorldPartition::SetupHLODActors(const FSetupHLODActorsParams& Params)
{
	RuntimeHash->SetupHLODActors(StreamingGenerator.GetStreamingGenerationContext(InContainerInstanceCollection), Params);
}
```
2 对于 UWorldPartitionRuntimeHashSet::SetupHLODActors
```cpp
bool UWorldPartitionRuntimeHashSet::SetupHLODActors(...) const
{
	// 1 首先用当前map里的ActorSet来生成streamingCell，用的是MainGrid的配置
	TMap<URuntimePartition*, TArray<URuntimePartition::FCellDescInstance>> RuntimePartitionsStreamingDescs;
	GenerateRuntimePartitionsStreamingDescs(CurrentHLODStreamingGenerationContext
		.Get(), RuntimePartitionsStreamingDescs);
	// 2 拿到其中一个StreamingCel，为每个cell里的Actorset所配置的hlodlayer生成对应的hlodActor
	TArray<AWorldPartitionHLOD*> CellHLODActors = WPHLODUtilities->CreateHLODActors(HLODCreationContext, HLODCreationParams, ActorInstances);
	// 3 在第2步生成了level0的hlodActor，如果这些hlodActor配置了自己的hlodlayer，那么就会在第3步继续生成对应的level1的hlodActor，这样的递归循环操作
	TArray<AWorldPartitionHLOD*> CellHLODActors = WPHLODUtilities->CreateHLODActors(HLODCreationContext, HLODCreationParams, ActorInstances);
}
```
3 对于`UWorldPartitionRuntimeSpatialHash::SetupHLODActors()`:
```cpp
bool UWorldPartitionRuntimeSpatialHash::SetupHLODActors(...) const
{
	// 1 获取map中所有actordesc上配置得HLODLayer
	TMap<UHLODLayer*, int32> HLODLayersLevels = GatherHLODLayers(StreamingGenerationContext, WorldPartition);
	TArray<UHLODLayer*> HLODLayers;
	HLODLayersLevels.GetKeys(HLODLayers);
	// 2 如果我们的HLODLayer勾上了IsSpatiallyLoaded，就会给对应的hlodlayer创建出对应的FSpatialHashRuntimeGrid
	TMap<FName, FSpatialHashRuntimeGrid> HLODGrids = CreateHLODGrids(HLODLayersLevels);
	// 3 将wp中配置的所有grid都赋值到局部变量GridsMapping中
	TMap<FName, int32> GridsMapping;
	GridsMapping.Add(NAME_None, 0);
	for (int32 i = 0; i < Grids.Num(); i++)
	{
		const FSpatialHashRuntimeGrid& Grid = Grids[i];
		check(!GridsMapping.Contains(Grid.GridName));
		GridsMapping.Add(Grid.GridName, i);
	}
	// 4 将map中AWorldPartitionHLOD这个类型的actor添加到局部变量Context中
	FHLODCreationContext Context;
	// 5 构建GridActorSetInstances数组。数组索引是map中每个ActorSet所在的grid在Grds中的索引，数组的值是ActorSet数组
	TArray<TArray<const IStreamingGenerationContext::FActorSetInstance*>> GridActorSetInstances;
	// 6 生成level0的hlodlayerActor。遍历Grids数组了，为每个grid都走GenerateHLODActors方法，每个grid里会生成很多的cell，每个不同的cell都会生成对应的AWorldPartitionHLod。
	GenerateHLODActors(Grids[GridIndex], 0, GridActorSetInstances[GridIndex]);
	// 7 然后对新的AWorldPartitionHLOD创建下一级AWorldPartitionHLOD，只会对勾上IsSpatiallyLoaded这个的hlodlayer创建下一级
	GenerateHLODActors(HLODGrids[HLODGridName], HLODLayersLevels[HLODGrids[HLODGridName].HLODLayer] + 1, HLODActorSetInstancePtrs);
}
```
总结下就是，如果我们的Actor上的hlodlayer没有勾选空间加载，那么只会在Grid上的cell的里面的actorSet生成对应的AWorldPartitionHLod，是level0级的。反之，还会在HLODGrid上的cell的里面的HLODActor生成下一级的AWorldPartitionHLod。
### HLOD_Build
1 根据命令行的参数来执行不同的操作，下面看下BuildHLODActors方法
```cpp
bool UWorldPartitionHLODsBuilder::BuildHLODActors()
{
	if (WorldPartition)
	{
		// 从WorldPartition里遍历到HLODActor，如果不存在就返回false
		TArray<FGuid> HLODActorsToBuild;
		if (!GetHLODActorsToBuild(HLODActorsToBuild))
		{
			return false;
		}
		// 遍历这些收集到的HLODActor，然后执行BuildHLOD方法
		for (int32 CurrentActor = ResumeBuildIndex; CurrentActor < HLODActorsToBuild.Num(); ++CurrentActor)
		{
			FWorldPartitionReference ActorRef(WorldPartition, HLODActorGuid);
			AWorldPartitionHLOD* HLODActor = CastChecked<AWorldPartitionHLOD>(ActorRef.GetActor());
			HLODActor->BuildHLOD(bForceBuild);
		}
	}
}
```
2 首先会收集WorldPatition里面所有的HLODActor，然后遍历这个HLODActor数组，对其中每个元素都执行BuildHLOD方法。
```cpp
uint32 FWorldPartitionHLODUtilities::BuildHLOD(AWorldPartitionHLOD* InHLODActor)
{
	// 1 首先加载hlodactor中的sourceActors，加载成streaminglevel的形式，也就是会把SourceActors这些Actor都放到streaminglevel中。这里走的方法是UWorldPartitionHLODSourceActorsFromCell::LoadSourceActors这个方法
	ULevelStreaming* LevelStreaming = nullptr;
	{
		FAutoScopedDurationTimer LoadTimeScope;
		LevelStreaming = LoadSourceActors(InHLODActor, bIsDirty);
		LoadTimeMS = FMath::RoundToInt(LoadTimeScope.GetTime() * 1000);
	}
	// 2 遍历level里面的actor，将actor身上挂着的hlodRelevant的comp都收集到这个HLODRelevantComponents里面
	if (LevelStreaming->GetLoadedLevel())
	{
		HLODRelevantComponents = GatherHLODRelevantComponents(LevelStreaming->GetLoadedLevel()->Actors);
	}
	// 3 将HLODActor(可以复用以前的)的HLODHash，与HLODActor的信息和HLODRelevantComponents计算的Hash比较，不同则说明数据发生变化，需要重新构建。否则跳过构建
	uint32 OldHLODHash = bIsDirty ? 0 : InHLODActor->GetHLODHash();
	uint32 NewHLODHash = ComputeHLODHash(InHLODActor, HLODRelevantComponents);
	if (OldHLODHash == NewHLODHash)
	{
		return OldHLODHash;
	}
	// 4 获取hlodlayer配置的构建方式，根据不同的buildClass来执行不同的Build方法，返回构建出的Comp
	const UHLODLayer* HLODLayer = InHLODActor->GetSourceActors()->GetHLODLayer();
	TSubclassOf<UHLODBuilder> HLODBuilderClass = GetHLODBuilderClass(HLODLayer);
	HLODComponents = HLODBuilder->Build(HLODBuildContext);
	// 5 将构建出的Comp都Attach到HLODActor身上
	InHLODActor->SetHLODComponents(HLODComponents);
}
```
3 对每个HlodActor都执行以下操作，首先获取其SourceActor，然后通过SourceActor配置的HlodLayer里面配置的build方式创建出对应的Comp，然后将其Attach到HlodActor上。下面看下各种不同的build方式。
#### 1 Instancing
```cpp
TArray<UActorComponent*> UHLODBuilderInstancing::Build(const FHLODBuildContext& InHLODBuildContext, const TArray<UActorComponent*>& InSourceComponents) const
{
	// UHLODBuilderInstancing将相同的Mesh合并成InstancedStaticMesh。默认使用的是`UHLODInstancedStaticMeshComponent
	TArray<UActorComponent*> HLODComponents = UHLODBuilder::BatchInstances(InSourceComponents);
}
```

#### 2 MeshMerge
```cpp
TArray<UActorComponent*> UHLODBuilderMeshMerge::Build(const FHLODBuildContext& InHLODBuildContext, const TArray<UActorComponent*>& InSourceComponents) const
{
	// 把所有StaticMeshComponent合并成一个StaticMesh。
	MeshMergeUtilities.MergeComponentsToStaticMesh(SourcePrimitiveComponents, InHLODBuildContext.World, UseSettings, HLODMaterial, InHLODBuildContext.AssetsOuter->GetPackage(), InHLODBuildContext.AssetsBaseName, Assets, MergedActorLocation, 0.25f, false);

}
```
#### 3 MeshSimplify
```cpp
TArray<UActorComponent*> UHLODBuilderMeshSimplify::Build(const FHLODBuildContext& InHLODBuildContext, const TArray<UActorComponent*>& InSourceComponents) const
{
	// 目的也是把所有StaticMeshComponent合并成一个StaticMesh，但是合并的应该是一个简化的mesh这是和MeshMerge的区别所在
}
```
#### 4 MeshApproximate
```cpp
TArray<UActorComponent*> UHLODBuilderMeshApproximate::Build(const FHLODBuildContext& InHLODBuildContext, const TArray<UActorComponent*>& InSourceComponents) const
{
	//它与`SimplifiedMesh`相比，这里会剔除掉看不见的Mesh，这可以减少许多不必要的三角面。内部的Mesh就完全不存在，所以这种合并的结果通常比`SimplifiedMesh`的结果还要简化。这对于大规模的室内场景的HLOD是巨大的提升
}
```
# 问题
1 BundleGuid是干嘛用的
2 actor的runtimegrid不同，但他们在一个actorset里，grid会改么，改成什么呢？
3 为什么grid要是一个数组呢？为啥需要多个呢？
可以设置场景中不同actorset的streaming范围
4 actorset表示一组相关的actor，这里是怎么判断相关的呢？
在WorldPartitionActorDesc创建的时候，就会创建actor的引用关系，通过序列化actor的元数据来获取的。
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250506173435.png)
5 
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250506152223.png)
原因是物体E在划分格子的边界上，这时就会导致level等级的增加。ue提供了以下方式解决：
- 物体Bounds小于格子长度的，直接用物体的location来进行划分
- 边界上的Actor在向上查找完全包含自己的Cell时过度向上提升的原因是，Level每高一级，GridSize就减小一倍,但是这样会导致下一个Level的Cell的边与上一个是完美重合的。所以我们就在每一级的GridSize都增加1，让每一级的格子不完全对其。（GridSize是指这一级会被分成多少个格子）
6 我们在场景中有一个地板和三个立方体，我们在streaming加载的时候会出现立方体先加载出来，导致立方体掉落，这个该怎么解决？
- 我们在划分的时候可能会划分成四个cell，这四个cell的加载顺序无法保证就可能出现上述问题。
- 我们可以设置不同的Grid，也就是将地板和立方体应用不同的加载策略，地板设置成100米就加载，立方体设置成50米加载。
- 在不同的Grid上也可以设置优先级，如果加载的范围都是100米，那么就可以设置优先级，让立方体先加载。
7 在世界分区中进行人物传送（传送）时，由于流送需要一定时间，如果将人物直接转移到目标位置，很可能因为周边物体还没有加载完成，角色因为重力直接掉到地底
- 关闭角色Tick，将角色传送到目标点，以角色自身作为流送源触发周边区域的流送， **轮询（FTSTicker）** 到目标区域加载完成时开启Tick
- 在目标位置Spawn一个带流送源组件的Actor，开启流送，轮询加载是否完成，完成后再将角色传送过去
8 场景划分是怎么划分的呢？
ue5提供了两种划分的模式：UWorldPartitionRuntimeHashSet和UWorldPartitionRuntimeSpatialHash
- UWorldPartitionRuntimeHashSet这种划分的方式是3d的，就是将场景划分成具有level的三维格子，然后求出ActorSet的Bounds的中心点在哪个格子里，生成对应的cell。
- UWorldPartitionRuntimeSpatialHash这种划分方式是2d的，先将场景划分成棋盘格，然后根据ActorSet确定放在哪个格子中，生成对应的Cell。
9 再有了cell之后是怎么streaming的呢？
- 在每帧的tick里面，会进行streaming的状态改变，首先会更新StremingSource，更新状态（在范围里的cell是load还是activate），更新速度（）。然后就是通过Source的范围和cell的Bounds进行距离判断，判断这一次哪些cell需要load哪些cell需要unload。然后根据cell创建出denamicStreaminglevel，设置其标志位bshouldvisible，bshouldloaded等，具体逻辑是在UWorldPartitionRuntimeLevelStreamingCell中。
- 设置标志位bshouldvisible，bshouldloaded之后，就和streaminglevel的加载一样了，通过tick中的状态切换从unload到loadedvisible会经过的状态是UnLoad->LoadedNotVisible->MakingVisible->LoadedVisible，期间主要是RequestLevel方法来loaded，和AddToWorld方法来初始化Actor。
- RequestLevel是被UWorldPartitionLevelStreamingDynamic方法重写了，里面的具体逻辑是首先创建一个空的level，然后异步加载Cell中记录的Actor的package，加载actor完成后添加到level中。等到AddToWorld的时候在初始化，注册，beginplay这个level中的Actor。
10 hlodactor的level有什么用？ 
- 我们会先将场景划分cell，然后遍历cell里的所有actorset，对每一个actorset生成对应的hlodActor这些都是level0的，如果生成的hlodactor他还有hloadlayer的话（设置hlodlayer的parentlayer），就生成下一级的HlodActor。
- 在UWorldPartitionRuntimeSpatialHash划分条件下，不是空间加载的hlodlayer就不会有等级划分都是0级，如果是空间加载的hlodlayer，就会设置parentlayer为level1级的。
11 hlodactor中的sourceActor什么时候赋值的？
- 是在生成hlodactor的时候赋值的，也就是划分完cell之后，如果cell里面的actor配置的是同一个hlodlayer那么就会生成一个hlodactor，然后设置sourceActor为原始actor。如果不同cell里的actor使用的相同的hlodlayer，是会生成不同的hlodactor的，他们的sourceActor都是各自的原始actor。
12 HLOD是怎么运作的？
- 构建部分：首先是setup的过程，先将场景划分成cell，遍历cell里的Actor，如果配置了HLodLayer就会创建出对应的WorldPartitionHLodActor。同一个Cell里相同的HLodLayer只会有一个HlodActor，不同的Cell里相同的HLodLayer会有不同的HlodActor。接下来就是Build部分，拿到WorldPartition里所有的HlodActor，然后遍历，根据不同HloadLayer配置，来执行不同的build方法创建出Comp，然后将Comp，Attach到HlodActor身上。
- 运行切换部分：我们在运行是会有一个persistentGrid来划分不会空间加载的actor（包括普通actor和HlodActor），然后还会有个MainGrid来划分空间加载的普通Actor，而那些HLodActor会通过HlodGrid来划分，我们在RunitmePartitions里配置的HlodLayer都会有对应的HlodGrid，通过HlodGrid划分HlodActor，生成对应的Cell，然后通过StreamingSource的距离加载出对应的Cell。
13 命令行
- wp.Runtime.ToggleDrawRuntimeHash2D 开关世界分区运行时哈希的2D调试显示。
- wp.Runtime.ToggleDrawRuntimeHash3D 开关世界分区运行时哈希的3D调试显示。
- wp.Runtime.ShowRuntimeSpatialHashGridLevel 选择在显示世界分区运行时哈希时显示的网格级别。
- wp.Runtime.ShowRuntimeSpatialHashGridLevelCount 选择在显示世界分区运行时哈希时要显示多少个网格级别。
- wp.Runtime.ShowRuntimeSpatialHashGridIndex 显示世界分区运行时哈希时，显示指定的网格。 无效的索引将导致显示所有网格。
- wp.Runtime.RuntimeSpatialHashCellToSourceAngleContributionToCellImportance 取0到1之间的值，用于调节"流送源-单元网格"向量和"流送源-单元网格"向量之间的角度对单元网格重要性的贡献。 该值越接近于0，角度对单元重要性的贡献就越小。
- wp.Editor.DumpStreamingGenerationLog 查看wp中所有actor的信息
14 DataLayerType中Editor和RuntimeTime有什么区别？
- Editor是编辑器下使用的，用作分类管理，打包后是不生效的，也就是打包后这个数据层就相当于没有。
- Runtime是运行时使用的，就是可以动态控制的，我们通过加载数据层在控制actor的显示与否。具体打包后生效是在 FDataLayerUtils::ResolveRuntimeDataLayerInstanceNames 方法中吧。
15 同一个actor配多个数据层，他是or还是and的关系呢？
- 由DataLayersLogicOperator WorldPartition中的这个属性，或者是config文件中的NewMapsDataLayersLogicOperator这个属性决定，yh这边用的是or。