# Actor 或 Component的Tick
### Actor/Component Tick注册
流程图
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202403231611686.png)
1 从AActor::Beginplay开始注册的
``` cpp
void AActor::BeginPlay()  
{  
	//把没用的部分省略了
   RegisterAllActorTickFunctions(true, false); // Components are done below.  
   TInlineComponentArray<UActorComponent*> Components;  
	GetComponents(Components);  
  
	for (UActorComponent* Component : Components)  
	{  
	   // bHasBegunPlay will be true for the component if the component was renamed and moved to a new outer during initialization  
	   if (Component->IsRegistered() && !Component->HasBegunPlay())  
	   {      
		   Component->RegisterAllComponentTickFunctions(true);  
		   Component->BeginPlay();  
	   }   else  
	   {  
	      // When an Actor begins play we expect only the not bAutoRegister false components to not be registered  
	      //check(!Component->bAutoRegister);   }  
	}
}
```
2 通过RegisterAllActorTickFunctions方法注册Actor的TickFunction，两个参数一个是注册，一个是否注册component的TickFunction
```cpp
void AActor::RegisterAllActorTickFunctions(bool bRegister, bool bDoComponents)  
{  
   if(!IsTemplate())  
   {      // Prevent repeated redundant attempts  
      if (bTickFunctionsRegistered != bRegister)  
      {         
	     FActorThreadContext& ThreadContext = FActorThreadContext::Get();  
 
         RegisterActorTickFunctions(bRegister);// 这里实际调用的方法
         bTickFunctionsRegistered = bRegister;  

         ThreadContext.TestRegisterTickFunctions = nullptr;  
      }  
      if (bDoComponents)  
      {         
	      for (UActorComponent* Component : GetComponents())  
         {            
	         if (Component)  
            {               
		        Component->RegisterAllComponentTickFunctions(bRegister);  
            }        
          }      
       }
       // 省略了异步物理tick，不知道干啥的
    }
}
void AActor::RegisterActorTickFunctions(bool bRegister)  
{  
   if(bRegister)  
   {      
	   if(PrimaryActorTick.bCanEverTick)  
      {         
	     PrimaryActorTick.Target = this;//设置TickFunction的Target
         PrimaryActorTick.SetTickFunctionEnable(PrimaryActorTick.bStartWithTickEnabled                || PrimaryActorTick.IsTickFunctionEnabled());//设置状态
         PrimaryActorTick.RegisterTickFunction(GetLevel());//注册进去
      }   
    }   
    else  
	{  
      if(PrimaryActorTick.IsTickFunctionRegistered())  
      {         
	      PrimaryActorTick.UnRegisterTickFunction();         
      }  
    }  
   FActorThreadContext::Get().TestRegisterTickFunctions = this; // we will verify the super call chain is intact. Don't copy and paste this to another actor class!  
}
void FTickFunction::RegisterTickFunction(ULevel* Level)
{
	if (!IsTickFunctionRegistered())
	{
		// Only allow registration of tick if we are are allowed on dedicated server, or we are not a dedicated server
		const UWorld* World = Level ? Level->GetWorld() : nullptr;
		if(bAllowTickOnDedicatedServer || !(World && World->IsNetMode(NM_DedicatedServer)))
		{
			if (InternalData == nullptr)
			{
				InternalData.Reset(new FInternalData());
			}
			FTickTaskManager::Get().AddTickFunction(Level, this);//下文介绍下
			InternalData->bRegistered = true;
		}
	}
	else
	{
		check(FTickTaskManager::Get().HasTickFunction(Level, this));
	}
}
```
3 最终会走到RegisterActorTickFunctions这个方法种注册，也就是给PrimaryActorTick(它就是AActor里面的TickFunction)赋值。然后就是通过SetTickFunctionEnable设置TickFunctionEnable的状态为ETickState::Enabled，如果已经注册过就删掉再重新注册。最后就是RegisterTickFunction注册方法，通过FTickTaskManager的单例实际绑定Actor的TickFunction，注册完后设置标志位。
```cpp
// FTickTaskManager的方法
// ULevel* InLevel ->>>>>>>>> actor所处的Level
// FTickFunction* TickFunction ->>>>>>>> actor身上挂着的TickFunction
void AddTickFunction(ULevel* InLevel, FTickFunction* TickFunction)
	{
		check(TickFunction->TickGroup >= 0 && TickFunction->TickGroup < TG_NewlySpawned); // You may not schedule a tick in the newly spawned group...they can only end up there if they are spawned late in a frame.
		FTickTaskLevel* Level = TickTaskLevelForLevel(InLevel);
		Level->AddTickFunction(TickFunction);
		TickFunction->InternalData->TickTaskLevel = Level;//这边加个关联，取得时候好取吧
	}

// FTickTaskLevel的方法
void AddTickFunction(FTickFunction* TickFunction)
	{
		check(!HasTickFunction(TickFunction));
		if (TickFunction->TickState == FTickFunction::ETickState::Enabled)
		{
			// 保存到最主要的ticktasklevel里面的所有能够tick的TickFunctions数组
			AllEnabledTickFunctions.Add(TickFunction);// 把tickfunction保存到数组中
			if (bTickNewlySpawned)
			{
				// 这个看上去就是再tick的时候添加了tickFunction,就把它也添加到这里里面
				NewlySpawnedTickFunctions.Add(TickFunction);// 把tickfunction保存到数组中
			}
		}
		else
		{
			check(TickFunction->TickState == FTickFunction::ETickState::Disabled);
			// 这个当前就是不进行tick的TickFunctions数组
			AllDisabledTickFunctions.Add(TickFunction);// 把tickfunction保存到数组中
		}
	}
```
4 通过AddTickFunction方法将Actor的TickFunction绑定到Actor所处的Level的TickTaskLevel上面，绑定的具体操作就是把TickFunction保存到相应的数组中去。
5 注意Actor的Tick和Component的Tick没有顺序关系，他们都是通过FTickFunction::RegisterTickFunction这个方法来将TickFunction绑定到Level里面的。
6 还有个注册时机是再AActor::IncrementalRegisterComponents方法里面，说是跟切换场景相关，后续再看下
### Tick时机
```cpp
void AActor::InitializeDefaults()
{
	PrimaryActorTick.TickGroup = TG_PrePhysics; // 这里设置了TickGroup
	// Default to no tick function, but if we set 'never ticks' to false (so there is a tick function) it is enabled by default
	//这个为true，才会注册Tick，所以我们写的actor就可以设置这个来决定是否可以tick
	PrimaryActorTick.bCanEverTick = false; 
	PrimaryActorTick.bStartWithTickEnabled = true;//可以开始Tick的标志位
	PrimaryActorTick.SetTickFunctionEnable(false); // 设置TickFunction的状态
	bAsyncPhysicsTickEnabled = false;//什么同步物理tick的标志位
}

```
### Tick顺序和依赖关系
1 依赖关系的构建主要是通过TickFunction中的Prerequisites这个数组来决定。我们拿AController，UPawnMovementComponent和APawn之间的Tick顺序来进行展示。
```cpp


```


