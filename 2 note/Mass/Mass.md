UE5的ECS：MASS框架(一)
https://zhuanlan.zhihu.com/p/441773595

# 1 FMassEntityHandle
Entity就相当于是 FMassEntityHandle，其中FMassEntityHandle的结构如下：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250716234618.png)
其中只有两个成员变量，显而易见的就是在一个大数组里存储的索引和一个唯一标识，和FWeakObjectPtr原理是一样的。那么这个大数组存储在哪呢？下面这张图就是创建 FMassEntityHandle 的方法。
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250716235838.png)
图中可以看到大数组索引就是 EntityIndex。我们第一次走到这个Acquire方法会执行AddPage来添加页，也就是分配一大块内存（内存的大小 = sizeof(FEntityData) * countprepage）,这个页中会有count数量的FEntityData，所有页中的FEntityData索引都是递增的。==所以呢 EntityIndex 代表的就是页中FEntityData的索引。而我们的大数组呢，就是所有页的集合。==

# 2 FMassEntityTemplate

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
# Archetype
FMassEntityTemplate 是用来存放 Archetype 的，那什么是 Archetype 呢？类是对象的原型，UObject的原型是CDO，那么Entity的原型就是Archetype，可以这么理解吧。回过头来，现在我们看看Archetype的组成部分：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250718213137.png)
我们可以看到有很多的成员变量。我们来看看每个成员变量的含义：
```cpp
/*
	1 FMassArchetypeCompositionDescriptor CompositionDescriptor;//原型组成描述符，由五个Fragments, Tags, ChunkFragments, SharedFragments, ConstSharedFragments组成。在Trait创建的时候会生成相应的Fragments，并赋值给EntityTemplate的CompositionDescriptor, 然后在Archetype::Initialize方法中传递给Archetype的CompositionDescriptor成员变量。
	2 uint32 CreatedArchetypeDataVersion = 0;// 就是一个计数器，每次我们创建新的Archetype的时候，这个计数器+1，并且这个值一旦设置就不会被更改。
	3 TArray<FMassArchetypeFragmentConfig, TInlineAllocator<16>> FragmentConfigs;//这个看上去就是一个缓存用于加快索引。是一个数组，这个数组的key是Fragments的索引，value是Fragment的type和ArrayOffsetWithinChunk组成的FMassArchetypeFragmentConfig（ArrayOffsetWithinChunk指的是在Chunk中的偏移，Chunk是一块很大的内存，它里面保存了所有en't）。
	4 TMap<const UScriptStruct*, int32> FragmentIndexMap;//这个也是加快索引的，和FragmentConfigs搭配使用，key和value是FragmentConfigs的key和value反过来。
	5 SIZE_T TotalBytesPerEntity = 0;//这个记录了MassEntityHandle的大小 + 所有Fragment的大小。字面意思上就是每个Entity所占的字节。
	6 const SIZE_T ChunkMemorySize = 0;//Chunk在内存中所占的大小。目前是代码写死的，是128 * 1024，也就是128KB。
	7 int32 NumEntitiesPerChunk;//Chunk由两部分组成，一部分是固定大小用于保存所有Fragment的对齐和（Fragment这个结构体所需内存对齐，按多少内存对齐，这个多少的和），还有一部分就是存放所有Fragment的地方，所以 
	NumEntitiesPerChunk = （ChunkMemorySize - 固定和）/ TotalBytesPerEntity
	8 
*/

	

```

问题：
```cpp
/*
1 TArray<const UScriptStruct*, TInlineAllocator<16>> SortedFragmentList;  
CompositionDescriptor.Fragments.ExportTypes(SortedFragmentList);
这个Fragments里面会有重复的么？也就是SortedFragmentList这个数组里会有重复值么？
目前来看应该是可能有重复的，没找到去重的逻辑，而且TArray的add也支持添加重复元素。
*/
```
# 1 MassSample学习
## 1 # CrowdGym 场景


