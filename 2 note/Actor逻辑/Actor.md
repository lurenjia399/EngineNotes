# 1 FActorSpawnParameters 创建参数
```cpp
/*
struct FActorSpawnParameters  
{  
	// actor的名称，就是那个....._01
    FName Name;  
	// actor的模板，如果设置了会使用Template上的属性，没设置就会取actor的cdo
    AActor* Template;  

    AActor* Owner;  

    APawn* Instigator;  
  
    class  ULevel* OverrideLevel;
    // 可以重写Actor的ParentComponent
		class   UChildActorComponent* OverrideParentComponent;
		// actor的所有权在服务器。此时正在同步创建客户端的，这时为true
		uint8   bRemoteOwned:1;
}*/

```

# 2 UWorld::SpawnActor
```cpp

AActor* UWorld::SpawnActor( 
	UClass* Class, 
	FTransform const* UserTransformPtr, 
	const FActorSpawnParameters& SpawnParameters )
{
/*
1 在创建的开始，如果是编辑器就检测CurrentLevel，如果不是编辑器就把PersistentLevel赋值给CurrentLevel。注意非编辑器下是没有CurrentLevel。
*/
#if WITH_EDITORONLY_DATA
	check( CurrentLevel ); 	
	check(GIsEditor || (CurrentLevel == PersistentLevel));
#else
	ULevel* CurrentLevel = PersistentLevel;
#endif

/*
2 一个Scoped结构体，用来检测生成Actor所需的时间的，通过spawnactortimer start这个命令行来记录,spawnactortimer end来关掉
*/
#if ENABLE_SPAWNACTORTIMER
	FScopedSpawnActorTimer SpawnTimer(
		Class->GetFName(),SpawnParameters.bDeferConstruction ? 
		ESpawnActorTimingType::SpawnActorDeferred : 
		ESpawnActorTimingType::SpawnActorNonDeferred);
#endif

/*
3 通过NewObject创建出了Actor
*/
	AActor* const Actor = NewObject<AActor>(
		LevelToSpawnIn, 
		Class, 
		NewActorName, 
		ActorFlags, 
		Template, 
		false/*bCopyTransientsFromClassDefaults*/,
		nullptr/*InInstanceGraph*/, 
		ExternalPackage);
	
/*
4 判断生成参数中是否有重写的OverrideParentComponent,如果有就重写Actor的ParentComponent
*/
	if (SpawnParameters.OverrideParentComponent)  
	{  
	    FActorParentComponentSetter::Set(
		    Actor, 
		    SpawnParameters.OverrideParentComponent);  
	}  
/*
5 这里会执行自定义的一个初始化方法
*/
	if (SpawnParameters.CustomPreSpawnInitalization)  
	{  
	    SpawnParameters.CustomPreSpawnInitalization(Actor);  
	}
/*
6 将创建出的Actor添加到所属的Level中
*/
	LevelToSpawnIn->TryAddActorToList( 
		Actor,
		/*bAddUnique*/false);
		
/*
7 赋值Actor的碰撞规则，创建Actor的时候是否创建在可以创建的地方
*/
	Actor->SpawnCollisionHandlingMethod = 
		CollisionHandlingMethod;
		
/*
8 广播 OnActorPreSpawnInitialization 代理
*/
	OnActorPreSpawnInitialization.Broadcast(Actor);
/*
9 执行PostSpawnInitialize方法[[#3 AActor PostSpawnInitialize]]
*/
	Actor->PostSpawnInitialize(
		UserTransform, 
		SpawnParameters.Owner, 
		SpawnParameters.Instigator, d
		SpawnParameters.IsRemoteOwned(), 
		SpawnParameters.bNoFail, 
		SpawnParameters.bDeferConstruction, 
		SpawnParameters.TransformScaleMethod);
		
	return Actor;
}

```
# 3 AActor::PostSpawnInitialize
```cpp
void AActor::PostSpawnInitialize(
	FTransform const& UserSpawnTransform, 
	AActor* InOwner, 
	APawn* InInstigator, 
	bool bRemoteOwned, 
	bool bNoFail, 
	bool bDeferConstruction, 
	ESpawnActorScaleMethod TransformScaleMethod)
{
/*
1 记录创建的时间
*/
	UWorld* const World = GetWorld();
	bool const bActorsInitialized = 
		World && World->AreActorsInitialized();
	CreationTime = (World ? World->GetTimeSeconds() : 0.f);

/*
2 交换Role和RemoteRole，一般是在客户端创建服务器同步下来的actor需要交换
*/
	ExchangeNetRoles(bRemoteOwned);
/*
3 设置生成参数上的各种属性
*/
	SetOwner(InOwner);
	SetInstigator(InInstigator);
/*
4 修改根组件，根组件得是c++创建出来的，然后设置根组件的位置
*/
	USceneComponent* const SceneRootComponent = 
		FixupNativeActorComponents(this);
	if (SceneRootComponent != nullptr)
	{
		SceneRootComponent->SetWorldTransform(
			FinalRootComponentTransform, 
			false, nullptr, 
			ETeleportType::ResetPhysics);
	}
/*
5 执行ActorComponent的OnComponentCreated方法,注意是执行C++构造函数中CreateDefaultSubobject创建出的Component的。
*/
	DispatchOnComponentsCreated(this);
/*
6 bHasDeferredComponentRegistration 设置这个标志位，这个标志位表明是我们创建的这个Actor他的UClass是蓝图类并且这个Actor没有创建任何的C++组件。
*/
	bHasDeferredComponentRegistration = 
		(SceneRootComponent == nullptr 
		&& Cast<UBlueprintGeneratedClass>(GetClass()) != nullptr);
/*
7 如果不需要延迟注册，就直接注册。
注册里面首先执行PreRegisterAllComponents方法，
然后是
*/
	if (!bHasDeferredComponentRegistration && GetWorld())
	{
		RegisterAllComponents();
	}
	
	// See if anything has deleted us
	if( !IsValidChecked(this) && !bNoFail )
	{
		return;
	}

	PostActorCreated();

	if (!bDeferConstruction)
	{
		FinishSpawning(UserSpawnTransform, true);
	}
	else if (SceneRootComponent != nullptr)
	{
	
		GSpawnActorDeferredTransformCache.Emplace(this, UserSpawnTransform);
	}
}
```

# 各种方法执行时机

1 PostLoad 
 1. 关卡加载时 - 当包含该 Actor 的关卡/地图被加载时
  2. 资产加载时 - 当 Actor 作为资产被 LoadObject 或类似函数加载时
  3. 编辑器中 - 打开包含该 Actor 的关卡时
