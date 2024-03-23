# Actor 或 Component的Tick
### 注册
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
         PrimaryActorTick.SetTickFunctionEnable(PrimaryActorTick.bStartWithTickEnabled                || PrimaryActorTick.IsTickFunctionEnabled());  
         PrimaryActorTick.RegisterTickFunction(GetLevel());  
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
```
3 最终会走到RegisterActorTickFunctions这个方法种注册，也就是给PrimaryActorTick(它就是AActor里面的TickFunction)赋值。然后就是通过SetTickFunctionEnable设置TickFunctionEnable的状态为ETickState::Enabled，如果已经注册过就删掉再重新注册。最后就是RegisterTickFunction注册方法。
```cpp
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
		  FTickTaskManager::Get().AddTickFunction(Level, this);  
	      InternalData->bRegistered = true;  
	   }   
    }   
	else  
	{  
	    check(FTickTaskManager::Get().HasTickFunction(Level, this));  
	}
}

```