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
    // 可以重写Actor的ParentCo
		class   UChildActorComponent* OverrideParentComponent;
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
3 通过NewObject创建了
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
}

```