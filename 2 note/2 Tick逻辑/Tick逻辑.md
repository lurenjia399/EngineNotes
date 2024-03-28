### 1 Actor/Component Tick注册
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
5 注意Actor的Tick和Component的Tick没有顺序关系，他们都是通过FTickFunction::RegisterTickFunction这个方法来将TickFunction绑定到Level里面的，Actor和身上挂的Component之间的Tick也没有关系，要是有关系MovementComponent也不会在注册的时候绑定和Owner之间的依赖了。
6 还有个注册时机是再AActor::IncrementalRegisterComponents方法里面，说是跟切换场景相关，后续再看下
### 2 TickFunction初始化设置
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
### 3 TickFunction顺序和依赖关系

1 依赖关系的构建主要是通过TickFunction中的Prerequisites这个数组来决定。我们拿AController，UPawnMovementComponent和APawn之间的Tick顺序来进行展示。
```cpp
void AController::SetPawn(APawn* InPawn)
{
	RemovePawnTickDependency(Pawn);

	Pawn = InPawn;
	Character = (Pawn ? Cast<ACharacter>(Pawn) : NULL);

	AttachToPawn(Pawn);

	AddPawnTickDependency(Pawn); // 这里设置Tick依赖
}

```
2 我们在controller中的SetPawn方法中设置的Tick依赖，什么时候执行的SetPawn就后面在整理他的逻辑了。
```cpp
void AController::AddPawnTickDependency(APawn* NewPawn)
{
	if (NewPawn != NULL)
	{
		bool bNeedsPawnPrereq = true;
		UPawnMovementComponent* PawnMovement = NewPawn->GetMovementComponent();
		if (PawnMovement && PawnMovement->PrimaryComponentTick.bCanEverTick)
		{
			PawnMovement->PrimaryComponentTick.AddPrerequisite(this, this->PrimaryActorTick);//

			// Don't need a prereq on the pawn if the movement component already sets up a prereq.
			// bTickBeforeOwner 这个标志位是Movementcomponent特有的，因为MovementComponent
			// 的tick可以设置为Owner之前
			if (PawnMovement->bTickBeforeOwner || NewPawn->PrimaryActorTick.GetPrerequisites().Contains(FTickPrerequisite(PawnMovement, PawnMovement->PrimaryComponentTick)))
			{
				bNeedsPawnPrereq = false;
			}
		}
		
		if (bNeedsPawnPrereq)
		{
			NewPawn->PrimaryActorTick.AddPrerequisite(this, this->PrimaryActorTick);
		}
	}
}
```
3 先设置UPawnMovementComponent的Tick依赖AController的Tick，如果APawn的Tick不依赖UPawnMovementComponent的Tick，那么就在设置APawn的Tick依赖AController的Tick。
```cpp
// 这里就是UMovementComponent注册TickFunction的地方
void UMovementComponent::RegisterComponentTickFunctions(bool bRegister)
{
	Super::RegisterComponentTickFunctions(bRegister);

	// Super may start up the tick function when we don't want to.
	UpdateTickRegistration();

	// If the owner ticks, make sure we tick first
	AActor* Owner = GetOwner();
	// 在构造函数中bTickBeforeOwner这个是true，不过可以在蓝图中设成false
	if (bTickBeforeOwner && bRegister && PrimaryComponentTick.bCanEverTick && Owner && Owner->CanEverTick())
	{
		Owner->PrimaryActorTick.AddPrerequisite(this, PrimaryComponentTick);
	}
}

```

4 在UMovementComponent构造函数中bTickBeforeOwner这个是true，不过可以在蓝图中设成false，它的作用就是绑定Owner和UMovementComponent之间的依赖关系，Owner的Tick在UMovementComponent的Tick之后。

### 4 TickFunction执行
#### StartFrame
流程图
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202403241521828.png)
1 执行到World::Tick之前的流程：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202403241524671.png)

```cpp
// LaunchWindows.cpp，windows平台的main函数
int32 WINAPI WinMain(...)
->LaunchWindowsStartup(...)
->int32 GuardedMain(...)
	->EngineTick(void)//Launch.cpp文件中
		->FEngineLoop::Tick()
			->UGameEngine::Tick(...)
				->UWorld::Tick(...)
```
2 UWorld::Tick在执行前会先收集需要Tick的TickFunction，也就是StartFrame方法
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202403241539734.png)

```cpp
// FTickTaskManager中的方法
virtual void StartFrame(UWorld* InWorld, float InDeltaSeconds, ELevelTick InTickType, const TArray<ULevel*>& LevelsToTick) override
	{
		//省略了无关的
		
		// 这些就是设置Tick上下文的一些参数
		Context.TickGroup = ETickingGroup(0); // reset this to the start tick group
		Context.DeltaSeconds = InDeltaSeconds;
		Context.TickType = InTickType;
		Context.Thread = ENamedThreads::GameThread;
		Context.World = InWorld;

		bTickNewlySpawned = true;
		TickTaskSequencer.StartFrame();//初始化TickTaskSequencer的一些参数
		FillLevelList(LevelsToTick);//这个方法是将Level中的TickTaskLevel保存起来

		int32 NumWorkerThread = 0;
		bool bConcurrentQueue = false;
		// 这里省略了对bConcurrentQueue的赋值，在windows就是false
		if (!bConcurrentQueue)
		{
			int32 TotalTickFunctions = 0;
			for( int32 LevelIndex = 0; LevelIndex < LevelList.Num(); LevelIndex++ )
			{
				// 这里是执行保存的TickTaskLevel中的StartFrame，也是执行写初始化吧
				TotalTickFunctions += LevelList[LevelIndex]->StartFrame(Context);
			}
			// 省略了两个有关统计的宏
			for( int32 LevelIndex = 0; LevelIndex < LevelList.Num(); LevelIndex++ )
			{
				// 这里是执行保存的TickTaskLevel中的QueueAllTicks，下面介绍下
				LevelList[LevelIndex]->QueueAllTicks();
			}
		}
		else
		{
			// 这个else就不看了，反正windows平台肯定会走上边
		}
	}
```
3 首先会对Context进行个赋值，初始化TickTaskSequencer这个变量，然后将参数中的Level中TickTaskLevel保存到LevelList变量中，然后对其执行StartFrame和QueueAllTicks方法。
```cpp
// FTickTaskLevel中的方法
void QueueAllTicks()
	{
		FTickTaskSequencer& TTS = FTickTaskSequencer::Get();
		// 遍历AllEnabledTickFunctions数组，这个数组是在注册的是否赋值的
		// [[Tick逻辑#1 Actor/Component Tick注册]]
		for (TSet<FTickFunction*>::TIterator It(AllEnabledTickFunctions); It; ++It)
		{
			FTickFunction* TickFunction = *It;
			TickFunction->QueueTickFunction(TTS, Context); // 关键方法
			//当前这个Tickfunction还不能Tick，Tick时间还没到，放到TickFunctionsToReschedule数组里面，在World::tick的最后一行会执行EndFrame方法会对TickFunctionsToReschedule数组进行处理，通过TickFunctions中的InternalData这个结构的next指针串成链表的形式，链表的头指针是AllCoolingDownTickFunctions.Head是头指针
			if (TickFunction->TickInterval > 0.f)
			{
				It.RemoveCurrent();
				RescheduleForInterval(TickFunction, TickFunction->TickInterval);
			}
		}
		// 处理一遍处在cd的TickFunction
		int32 EnabledCooldownTicks = 0;
		float CumulativeCooldown = 0.f;
		while (FTickFunction* TickFunction = AllCoolingDownTickFunctions.Head)
		{
			if (TickFunction->TickState == FTickFunction::ETickState::Enabled)
			{
				CumulativeCooldown += TickFunction->InternalData->RelativeTickCooldown;
				TickFunction->QueueTickFunction(TTS, Context);// 关键方法
				RescheduleForInterval(TickFunction, TickFunction->TickInterval - (Context.DeltaSeconds - CumulativeCooldown)); // Give credit for any overrun
				AllCoolingDownTickFunctions.Head = TickFunction->InternalData->Next;
			}
			else
			{
				break;
			}
		}
	}

// 设置标志位，缓存到数组中
void RescheduleForInterval(FTickFunction* TickFunction, float InInterval)
	{
		TickFunction->InternalData->bWasInterval = true;
		TickFunctionsToReschedule.Add(FTickScheduleDetails(TickFunction, InInterval));
	}
```
4 首先会遍历所有可Tick的TickFunctions，执行QueueTickFunction方法，然后将为到时间的TickFunction放到TickFunctionsToReschedule数组中（在World::tick的最后一行会执行EndFrame方法会对TickFunctionsToReschedule数组进行处理，通过TickFunctions中的InternalData这个结构的next指针串成链表的形式，链表的头指针是AllCoolingDownTickFunctions.Head是头指针），然后遍历AllCoolingDownTickFunctions这个链表，都执行一遍QueueTickFunction方法。
```cpp
void FTickFunction::QueueTickFunction(FTickTaskSequencer& TTS, const struct FTickContext& TickContext)
{
	checkSlow(TickContext.Thread == ENamedThreads::GameThread); // we assume same thread here
	check(IsTickFunctionRegistered());

	if (InternalData->TickVisitedGFrameCounter != GFrameCounter)
	{
		InternalData->TickVisitedGFrameCounter = GFrameCounter;
		if (TickState != FTickFunction::ETickState::Disabled)
		{
			ETickingGroup MaxPrerequisiteTickGroup =  ETickingGroup(0);

			FGraphEventArray TaskPrerequisites;
			// 这部分代码很多省略掉
			// 这部分执行的代码很多，总的来说就是一个后序的递归遍历，目的是得到两个变量MaxPrerequisiteTickGroup和TaskPrerequisites，这两个变量是Prerequisites中最大的TickGroup和（里面存的是TickFunction的Task指针）

			// 这部分代码很多省略掉
			// 下面这一个大部分就是计算ActualStartTickGroup和ActualEndTickGroup变量，这两个变量需要考虑Prerequisites中的值，所以有可能和开始设置时不符

			// 这边时最终的核心方法，后面介绍下
			if (TickState == FTickFunction::ETickState::Enabled)
			{
				TTS.QueueTickTask(&TaskPrerequisites, this, TickContext);
			}
		}
		InternalData->TickQueuedGFrameCounter = GFrameCounter;
	}
}
```
5 通过递归的方式遍历TickFunction中的Prerequisites数组，找到其最大的TickGroup并复制给ActualStartTickGroup和ActualEndTickGroup，其中还有写细微的调整。
```cpp
	// FTickTaskSequencer中的方法
	FORCEINLINE void QueueTickTask(const FGraphEventArray* Prerequisites, FTickFunction* TickFunction, const FTickContext& TickContext)
	{
		StartTickTask(Prerequisites, TickFunction, TickContext);
		TGraphTask<FTickFunctionTask>* Task = (TGraphTask<FTickFunctionTask>*)TickFunction->InternalData->TaskPointer;
		AddTickTaskCompletion(TickFunction->InternalData->ActualStartTickGroup, TickFunction->InternalData->ActualEndTickGroup, Task, TickFunction->bHighPriority);
	}

```
6 执行了StartTickTask和AddTickTaskCompletion方法，前者创建了Task后者将Task存储到TickTasks或者是HiPriTickTasks数组中。

```cpp
// FTickTaskSequencer中的方法
	FORCEINLINE void StartTickTask(const FGraphEventArray* Prerequisites, FTickFunction* TickFunction, const FTickContext& TickContext)
	{
		checkSlow(TickFunction->InternalData);
		checkSlow(TickFunction->InternalData->ActualStartTickGroup >=0 && TickFunction->InternalData->ActualStartTickGroup < TG_MAX);

		FTickContext UseContext = TickContext;

		bool bIsOriginalTickGroup = (TickFunction->InternalData->ActualStartTickGroup == TickFunction->TickGroup);

		// 确定使用的线程
		if (TickFunction->bRunOnAnyThread && bAllowConcurrentTicks && bIsOriginalTickGroup)
		{
			if (TickFunction->bHighPriority)
			{
				UseContext.Thread = CPrio_HiPriAsyncTickTaskPriority.Get();
			}
			else
			{
				UseContext.Thread = CPrio_NormalAsyncTickTaskPriority.Get();
			}
		}
		else
		{
			UseContext.Thread = ENamedThreads::SetTaskPriority(ENamedThreads::GameThread, TickFunction->bHighPriority ? ENamedThreads::HighTaskPriority : ENamedThreads::NormalTaskPriority);
		}
		// 创建新的Task并将指针赋值给TaskPointer
		TickFunction->InternalData->TaskPointer = TGraphTask<FTickFunctionTask>::CreateTask(Prerequisites, TickContext.Thread).ConstructAndHold(TickFunction, &UseContext, bLogTicks, bLogTicksShowPrerequistes);
	}
```
7 这段有点复杂看不太懂，多线程的智识，知道是设置线程，创建Task。
```cpp
// FTickTaskSequencer中的方法
FORCEINLINE void AddTickTaskCompletion(ETickingGroup StartTickGroup, ETickingGroup EndTickGroup, TGraphTask<FTickFunctionTask>* Task, bool bHiPri)
	{
		checkSlow(StartTickGroup >=0 && StartTickGroup < TG_MAX && EndTickGroup >=0 && EndTickGroup < TG_MAX && StartTickGroup <= EndTickGroup);
		if (bHiPri)
		{
			HiPriTickTasks[StartTickGroup][EndTickGroup].Add(Task);
		}
		else
		{
			TickTasks[StartTickGroup][EndTickGroup].Add(Task);
		}
		new (TickCompletionEvents[EndTickGroup]) FGraphEventRef(Task->GetCompletionEvent());
	}

```
8 将7中创建的Task保存到HiPriTickTasks或者是TickTasks数组中（这两个二维数组的索引是StartTickGroup和EndTickGroup），还用了placement的方式在TickCompletionEvents数组（这个一维数组的索引是EndTickGroup）的地方new了FGraphEventRef。

![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202403241755877.png)

9 这个是 https://zhuanlan.zhihu.com/p/412418542 这个里面的图，也画出了我描述的StartFrame的过程借用一下

10 还有点小问题，EndTickGroup有什么用？可能和具体task什么时候执行有关?后面看多线程的时候在总结下。

#### RunTickGroup

在StartFrame之后就会进行每个Group的Tick。
```cpp
virtual void RunTickGroup(ETickingGroup Group, bool bBlockTillComplete ) override
	{
		// 这个方法就是执行当前这个TickGroup
		TickTaskSequencer.ReleaseTickGroup(Group, bBlockTillComplete);
		
		Context.TickGroup = ETickingGroup(Context.TickGroup + 1); // new actors go into the next tick group because this one is already gone
		if (bBlockTillComplete) // we don't deal with newly spawned ticks within the async tick group, they wait until after the async stuff
		{
			QUICK_SCOPE_CYCLE_COUNTER(STAT_TickTask_RunTickGroup_BlockTillComplete);

			bool bFinished = false;
			for (int32 Iterations = 0;Iterations < 101; Iterations++)
			{
				int32 Num = 0;
				for( int32 LevelIndex = 0; LevelIndex < LevelList.Num(); LevelIndex++ )
				{
					Num += LevelList[LevelIndex]->QueueNewlySpawned(Context.TickGroup);
				}
				if (Num && Context.TickGroup == TG_NewlySpawned)
				{
					SCOPE_CYCLE_COUNTER(STAT_TG_NewlySpawned);
					TickTaskSequencer.ReleaseTickGroup(TG_NewlySpawned, true);
				}
				else
				{
					bFinished = true;
					break;
				}
			}
			if (!bFinished)
			{
				// this is runaway recursive spawning.
				for( int32 LevelIndex = 0; LevelIndex < LevelList.Num(); LevelIndex++ )
				{
					LevelList[LevelIndex]->LogAndDiscardRunawayNewlySpawned(Context.TickGroup);
				}
			}
		}
	}
```
我们首先看下这个ReleaseTickGroup方法，具体如何执行TickGroup的。

```cpp
void ReleaseTickGroup(ETickingGroup WorldTickGroup, bool bBlockTillComplete)
	{
		{
			// 判断条件是否是单线程的
			if (SingleThreadedMode()|| CVarAllowAsyncTickDispatch.GetValueOnGameThread() == 0)
			{
				DispatchTickGroup(ENamedThreads::GameThread, WorldTickGroup);
			}
			else
			{
				FTaskGraphInterface::Get().WaitUntilTaskCompletes(
					TGraphTask<FDipatchTickGroupTask>::CreateTask(nullptr, ENamedThreads::GameThread).ConstructAndDispatchWhenReady(*this, WorldTickGroup));
			}
		}

		if (bBlockTillComplete || SingleThreadedMode())
		{
			for (ETickingGroup Block = WaitForTickGroup; Block <= WorldTickGroup; Block = ETickingGroup(Block + 1))
			{
				CA_SUPPRESS(6385);
				if (TickCompletionEvents[Block].Num())
				{
					TRACE_CPUPROFILER_EVENT_SCOPE(TickCompletionEvents);
					FTaskGraphInterface::Get().WaitUntilTasksComplete(TickCompletionEvents[Block], ENamedThreads::GameThread);
					if (SingleThreadedMode() || Block == TG_NewlySpawned || CVarAllowAsyncTickCleanup.GetValueOnGameThread() == 0 || TickCompletionEvents[Block].Num() < 50)
					{
						ResetTickGroup(Block);
					}
					else
					{
CleanupTasks.Add(TGraphTask<FResetTickGroupTask>::CreateTask(nullptr, ENamedThreads::GameThread).ConstructAndDispatchWhenReady(*this, Block));
					}
				}
			}
			WaitForTickGroup = ETickingGroup(WorldTickGroup + (WorldTickGroup == TG_NewlySpawned ? 0 : 1)); // don't advance for newly spawned
		}
		else
		{
FTaskGraphInterface::Get().ProcessThreadUntilIdle(ENamedThreads::GameThread);
			check(WorldTickGroup + 1 < TG_MAX && WorldTickGroup != TG_NewlySpawned);
		}
	}
```
1 ReleaseTickGroup这个方法分为两部分，第一部分是DispatchTickGroup，如果是单线程的话就直接调用DispatchTickGroup这个方法，如果是多线程就创建Task，现在还不太了解这个。第二部分是判断是否需要等待前面的TickGroup执行完（也就是bBlockTillComplete这个标志位），如果需要等待，就调用WaitUntilTasksComplete方法，等待执行完。
##### DispatchTickGroup
这个是将我们的Task压入到对应线程的无所优先级队列里
```cpp
void DispatchTickGroup(ENamedThreads::Type CurrentThread, ETickingGroup WorldTickGroup)
	{
		// 处理高优先级TickTask
		for (int32 IndexInner = 0; IndexInner < TG_MAX; IndexInner++)
		{
			TArray<TGraphTask<FTickFunctionTask>*>& TickArray = HiPriTickTasks[WorldTickGroup][IndexInner]; //-V781
			if (IndexInner < WorldTickGroup)
			{
				check(TickArray.Num() == 0); // makes no sense to have and end TG before the start TG
			}
			else
			{
				for (int32 Index = 0; Index < TickArray.Num(); Index++)
				{
					TickArray[Index]->Unlock(CurrentThread);
				}
			}
			TickArray.Reset();
		}
		// 处理普通优先级TickTask
		for (int32 IndexInner = 0; IndexInner < TG_MAX; IndexInner++)
		{
			TArray<TGraphTask<FTickFunctionTask>*>& TickArray = TickTasks[WorldTickGroup][IndexInner]; //-V781
			if (IndexInner < WorldTickGroup)
			{
				check(TickArray.Num() == 0); // makes no sense to have and end TG before the start TG
			}
			else
			{
				for (int32 Index = 0; Index < TickArray.Num(); Index++)
				{
					TickArray[Index]->Unlock(CurrentThread);
				}
			}
			TickArray.Reset();
		}
	}
```
2 整体分成了两部分，一部分处理高优先级TickTask，一部分出普通优先级TickTask，处理方式是一样的，遍历所有的TickGroup，找到当前TickGroup的Task然后调用UnLock方法执行。
```cpp
// TGraphTask类的
void Unlock(ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread)
	{
		// 这个感觉和Debug相关？不知道干嘛的
		TaskTrace::Launched(GetTraceId(), nullptr, Subsequents.IsValid(), ((TTask*)&TaskStorage)->GetDesiredThread(), sizeof(*this));

		bool bWakeUpWorker = true;
		ConditionalQueueTask(CurrentThreadIfKnown, bWakeUpWorker);
	}
// FBaseGraphTask类的
void ConditionalQueueTask(ENamedThreads::Type CurrentThread, bool& bWakeUpWorker)
	{
		if (NumberOfPrerequistitesOutstanding.Decrement()==0)
		{
			QueueTask(CurrentThread, bWakeUpWorker);
			bWakeUpWorker = true;
		}
	}
// FBaseGraphTask类的
void QueueTask(ENamedThreads::Type CurrentThreadIfKnown, bool bWakeUpWorker)
	{
		TaskTrace::Scheduled(GetTraceId());
		// 最终会调用到这个QueueTask方法里面
		FTaskGraphInterface::Get().QueueTask(this, bWakeUpWorker, ThreadToExecuteOn, CurrentThreadIfKnown);
	}
```
3 经过一系列的过渡调用，最终会调用到FTaskGraphImplementation这个类的这个QueueTask这个方法里面。
```cpp
void QueueTask(class FBaseGraphTask* Task, bool bWakeUpWorker, ENamedThreads::Type InThreadToExecuteOn, ENamedThreads::Type InCurrentThreadIfKnown) override
	{
		// 前面的部分都省略掉了，最关键的就是这一部分
		{
			int32 QueueToExecuteOn = ENamedThreads::GetQueueIndex(InThreadToExecuteOn);
			InThreadToExecuteOn = ENamedThreads::GetThreadIndex(InThreadToExecuteOn);
			FTaskThreadBase* Target = &Thread(InThreadToExecuteOn);
			if (InThreadToExecuteOn == ENamedThreads::GetThreadIndex(CurrentThreadIfKnown))
			{
				Target->EnqueueFromThisThread(QueueToExecuteOn, Task);
			}
			else
			{
				Target->EnqueueFromOtherThread(QueueToExecuteOn, Task);
			}
		}
	}
```
4 看上去就是拿到我们要的线程，然后就执行EnqueueFromThisThread这个方法，参数QueueToExecuteOn应该是个偏移啥的把，Task这个就是我们的TickTask吧。
```cpp
virtual void EnqueueFromThisThread(int32 QueueIndex, FBaseGraphTask* Task) override
	{
		uint32 PriIndex = ENamedThreads::GetTaskPriority(Task->GetThreadToExecuteOn()) ? 0 : 1;
		int32 ThreadToStart = Queue(QueueIndex).StallQueue.Push(Task, PriIndex);
		check(ThreadToStart < 0); // if I am stalled, then how can I be queueing a task?
	}

```
5 最后就是把我们的Task压入到Queue里面。这个队列是什么无锁优先级队列

##### WaitUntilTasksComplete

```cpp
//FTaskGraphImplementation中的方法
virtual void WaitUntilTasksComplete(const FGraphEventArray& Tasks, ENamedThreads::Type CurrentThreadIfKnown = ENamedThreads::AnyThread) final override
	{
		ENamedThreads::Type CurrentThread = CurrentThreadIfKnown;
		if (ENamedThreads::GetThreadIndex(CurrentThreadIfKnown) == ENamedThreads::AnyThread)
		{
		//这部分全省略了，tick这部分使用GameThread来执行的，不属于AnyThred
		}
		else
		{
			// 赋值下线程的类型
			CurrentThreadIfKnown = ENamedThreads::GetThreadIndex(CurrentThreadIfKnown);
		}

		if (CurrentThreadIfKnown != ENamedThreads::AnyThread && CurrentThreadIfKnown < NumNamedThreads && !IsThreadProcessingTasks(CurrentThread))
		{
			if (Tasks.Num() < 8)
			{
				bool bAnyPending = false;
				for (int32 Index = 0; Index < Tasks.Num(); Index++)
				{
					FGraphEvent* Task = Tasks[Index].GetReference();
					if (Task && !Task->IsComplete())
					{
						bAnyPending = true;
						break;
					}
				}
				if (!bAnyPending)
				{
					return;
				}
			}

			// named thread process tasks while we wait
			TGraphTask<FReturnGraphTask>::CreateTask(&Tasks, CurrentThread).ConstructAndDispatchWhenReady(CurrentThread);
			ProcessThreadUntilRequestReturn(CurrentThread);
		}
		else
		{
			// 这部分在tick不关键，省略了
		}
	}
```
