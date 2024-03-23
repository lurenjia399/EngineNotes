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
      {         FActorThreadContext& ThreadContext = FActorThreadContext::Get();  
         check(ThreadContext.TestRegisterTickFunctions == nullptr);  
         RegisterActorTickFunctions(bRegister);  
         bTickFunctionsRegistered = bRegister;  
         checkf(ThreadContext.TestRegisterTickFunctions == this, TEXT("Failed to route Actor RegisterTickFunctions (%s)"), *GetFullName());  
         ThreadContext.TestRegisterTickFunctions = nullptr;  
      }  
      if (bDoComponents)  
      {         for (UActorComponent* Component : GetComponents())  
         {            if (Component)  
            {               Component->RegisterAllComponentTickFunctions(bRegister);  
            }         }      }  
      if (bAsyncPhysicsTickEnabled)  
      {         if (FPhysScene_Chaos* Scene = static_cast<FPhysScene_Chaos*>(GetWorld()->GetPhysicsScene()))  
         {            if (bRegister)  
            {               Scene->RegisterAsyncPhysicsTickActor(this);  
            }            else  
            {  
               Scene->UnregisterAsyncPhysicsTickActor(this);  
            }         }      }   }}
```