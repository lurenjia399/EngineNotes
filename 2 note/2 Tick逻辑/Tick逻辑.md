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
 
         RegisterActorTickFunctions(bRegister);  
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
	      PrimaryActorTick.Target = this;  
         PrimaryActorTick.SetTickFunctionEnable(PrimaryActorTick.bStartWithTickEnabled || PrimaryActorTick.IsTickFunctionEnabled());  
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