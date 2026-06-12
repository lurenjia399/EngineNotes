# 1 TickFunction注册
贴个文章，写的很多很详细 https://www.cnblogs.com/kekec/p/14781454.html
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
# 2 TickFunction初始化设置
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
# 3 TickFunction顺序和依赖关系

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

# 4 TickFunction执行
## StartFrame
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
			->UGameEngine::Tick(...)//这里边就是启动了Tick，具体是哪个Engine是分开的
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
7 知道是设置线程，创建Task。
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
8 将7中创建的Task保存到HiPriTickTasks或者是TickTasks数组中（这两个二维数组的索引是StartTickGroup和EndTickGroup），还用了placement的方式在TickCompletionEvents数组（这个一维数组的索引是EndTickGroup）的地方new了FGraphEventRef，作用感觉上是保存了Task是否完成的事件？。

![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202403241755877.png)

9 这个是 https://zhuanlan.zhihu.com/p/412418542 这个里面的图，也画出了我描述的StartFrame的过程借用一下

10 还有点小问题，EndTickGroup有什么用？可能和具体task什么时候执行有关?后面看多线程的时候在总结下。

## RunTickGroup

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
### DispatchTickGroup
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
2 整体分成了两部分，一部分处理高优先级TickTask，一部分处理普通优先级TickTask，处理方式是一样的，遍历所有的TickGroup，找到当前TickGroup的Task然后调用UnLock方法执行。
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
5 最后就是把我们的Task压入到Queue里面。这个队列是什么无锁优先级队列，每个线程都有两个Queue，一个优先级高一个低，里面真正存储数据的队列是这个StallQueue。

### WaitUntilTasksComplete
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
			// 具体执行线程中队列的方法
			ProcessThreadUntilRequestReturn(CurrentThread);
		}
		else
		{
			// 这部分在tick不关键，省略了
		}
	}
// 创建，赋值了TaskStorage
FGraphEventRef ConstructAndDispatchWhenReady(T&&... Args)
	{
		new ((void *)&Owner->TaskStorage) TTask(Forward<T>(Args)...);
		return Owner->Setup(Prerequisites, CurrentThreadIfKnown);
	}
```
1 执行ProcessThreadUntilRequestReturn方法
```cpp
//FTaskGraphImplementation的方法
virtual void ProcessThreadUntilRequestReturn(ENamedThreads::Type CurrentThread) final override
	{
		int32 QueueIndex = ENamedThreads::GetQueueIndex(CurrentThread);
		CurrentThread = ENamedThreads::GetThreadIndex(CurrentThread);
		Thread(CurrentThread).ProcessTasksUntilQuit(QueueIndex);
	}
// FNamedTaskThread的方法
virtual void ProcessTasksUntilQuit(int32 QueueIndex) override
	{
		check(Queue(QueueIndex).StallRestartEvent); // make sure we are started up

		Queue(QueueIndex).QuitForReturn = false;
		verify(++Queue(QueueIndex).RecursionGuard == 1);
		const bool bIsMultiThread = FTaskGraphInterface::IsMultithread();
		do
		{
			const bool bAllowStall = bIsMultiThread;
			ProcessTasksNamedThread(QueueIndex, bAllowStall);
		} while (!Queue(QueueIndex).QuitForReturn && !Queue(QueueIndex).QuitForShutdown && bIsMultiThread); // @Hack - quit now when running with only one thread.
		verify(!--Queue(QueueIndex).RecursionGuard);
	}

uint64 ProcessTasksNamedThread(int32 QueueIndex, bool bAllowStall)
	{
		// 把前边这些宏全都省略了
		while (!Queue(QueueIndex).QuitForReturn)
		{
			const bool bIsRenderThreadAndPolling = bIsRenderThreadMainQueue && (GRenderThreadPollPeriodMs >= 0);
			const bool bStallQueueAllowStall = bAllowStall && !bIsRenderThreadAndPolling;
			// 从我们线程中的无锁队列中弹出Task
			FBaseGraphTask* Task = Queue(QueueIndex).StallQueue.Pop(0, bStallQueueAllowStall);
			TestRandomizedThreads();
			if (!Task)
			{
				// 这边Task不存在走，不看了
			}
			else
			{
				// 关键的执行方法
				Task->Execute(NewTasks, ENamedThreads::Type(ThreadId | (QueueIndex << ENamedThreads::QueueIndexShift)), true);
				ProcessedTasks++;
				TestRandomizedThreads();
			}
		}
		// 省略掉性能统计的
		return ProcessedTasks;
	}
```
2 执行FNamedTaskThread的ProcessTasksUntilQuit这个方法，然后执行到Task的Execute方法。
```cpp
// FBaseGraphTask中的方法
FORCEINLINE void Execute(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThread, bool bDeleteOnCompletion)
	{
		checkThreadGraph(LifeStage.Increment() == int32(LS_Executing));

		UE::FInheritedContextScope InheritedContextScope = RestoreInheritedContext();
		// 执行Task
		ExecuteTask(NewTasks, CurrentThread, bDeleteOnCompletion);
	}
```
3 就是一个中转把，会执行到ExecuteTask这个虚方法。
```cpp
// TGraphTask 中的方法
void ExecuteTask(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThread, bool bDeleteOnCompletion) override
	{
		// 一些检查的东西，省略掉
		
		TTask& Task = *(TTask*)&TaskStorage;
		{
			TaskTrace::FTaskTimingEventScope TaskEventScope(GetTraceId());
			FScopeCycleCounter Scope(Task.GetStatId(), true);
			// 最关键的DoTask方法
			Task.DoTask(CurrentThread, Subsequents);
			Task.~TTask();
		}
		
		TaskConstructed = false;

		// 一些设置吧,不看了

		// Task执行完成后，删除
		if (bDeleteOnCompletion)
		{
			DeleteTask();
		}
	}
```
4 会执行Task.DoTask这个方法，这个Task是FTickFunctionTask这个类型的，是在StartFrame中创建然后存储到数组中的Task，
```cpp
// FTickFunctionTask中的方法
void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
	{
		if (bLogTick)
		{
			if (bLogTicksShowPrerequistes)
			{
				Target->ShowPrerequistes();
			}
		}
		if (Target->IsTickFunctionEnabled())
		{
			// 关键的执行Tick方法
			Target->ExecuteTick(Target->CalculateDeltaTime(Context), Context.TickType, CurrentThread, MyCompletionGraphEvent);
		}
		Target->InternalData->TaskPointer = nullptr;  // This is stale and a good time to clear it for safety
	}
```
5 然后就会执行到Target->ExecuteTick方法，也就是执行到Tickfunction的ExecuteTick的方法。下面拿Actor举例。
```cpp
void FActorTickFunction::ExecuteTick(float DeltaTime, enum ELevelTick TickType, ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
{
	if (Target && IsValidChecked(Target) && !Target->IsUnreachable())
	{
		if (TickType != LEVELTICK_ViewportsOnly || Target->ShouldTickIfViewportsOnly())
		{
			Target->TickActor(DeltaTime*Target->CustomTimeDilation, TickType, *this);
		}
	}
}
void AActor::TickActor( float DeltaSeconds, ELevelTick TickType, FActorTickFunction& ThisTickFunction )
{
	//root of tick hierarchy

	// Non-player update.
	// If an Actor has been Destroyed or its level has been unloaded don't execute any queued ticks
	if (IsValidChecked(this) && GetWorld())
	{
		Tick(DeltaSeconds);	// perform any tick functions unique to an actor subclass
	}
}
```
6 最终通过Actor身上的PrimaryActorTick这个成员变量，执行到FActorTickFunction::ExecuteTick这个方法，进而执行到AActor::TickActor这个方法，也就是Actor的Tick。

# 5 TickFunction总结
1 我们Actor里会有一个FActorTickFunction类型的成员变量PrimaryActorTick，他会在Actor::Begin的时候把自己注册到Level里面（具体是在Level里的TicktaskLevel里的AllEnabledTickFunction里）
2 当我们World进行Tick的时候，会首先执行StartFrame方法，这个就是从1中保存的数组中拿到TickFunction，然后创建出对应的TickFunctionTask，保存到Task数组中。
3 然后执行RunTickGroup方法，根据不同的TickGroup来执行，首先将2中保存的Task压入到对应线程的无锁优先级队列中。最终执行就是从队列中pop出来TickFunctionTask，然后执行这个TickFunctionTask::DoTask方法，最终执行到TickFunction的ExecuteTick方法。

# 6 TimerManager添加

```cpp
UGameInstance::UGameInstance(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
	, TimerManager(new FTimerManager(this))
	, LatentActionManager(new FLatentActionManager())
{
}
```
1 我们会在GameInstance初始化的时候创建一个TimerManager类
```cpp
// 函数定义
FORCEINLINE void SetTimer(FTimerHandle& InOutHandle, FTimerDelegate const& InDelegate, float InRate, bool InbLoop, float InFirstDelay = -1.f)
	{
		InternalSetTimer(InOutHandle, FTimerUnifiedDelegate(InDelegate), InRate, InbLoop, InFirstDelay);
	}
// 这样调用
GetWorld()->GetTimerManager().SetTimer(newTimer, FTimerDelegate::CreateUObject(this, &UUserWidget::AnimationTimerFinished, InAnimation), TotalTime, false);
```
2 通过SetTimer来进行调用，第一个参数是FTimerHandle用来标识这个Timer的，第二个参数是Delegate用来timer结束回调的，第三个参数Timer的间隔，第四个参数是否循环，第五个参数是首次启动延迟多长时间。
```cpp
void FTimerManager::InternalSetTimer(FTimerHandle& InOutHandle, FTimerUnifiedDelegate&& InDelegate, float InRate, bool InbLoop, float InFirstDelay)
{
	SCOPE_CYCLE_COUNTER(STAT_SetTimer);

	// not currently threadsafe
	check(IsInGameThread());

	if (FindTimer(InOutHandle))
	{
		// if the timer is already set, just clear it and we'll re-add it, since 
		// there's no data to maintain.
		InternalClearTimer(InOutHandle);
	}

	if (InRate > 0.f)
	{
		// set up the new timer
		// 创建个FTimerData并组装
		FTimerData NewTimerData;
		NewTimerData.TimerDelegate = MoveTemp(InDelegate);

		NewTimerData.Rate = InRate;
		NewTimerData.bLoop = InbLoop;
		NewTimerData.bRequiresDelegate = NewTimerData.TimerDelegate.IsBound();

		// Set level collection
		const UWorld* const OwningWorld = OwningGameInstance ? OwningGameInstance->GetWorld() : nullptr;
		if (OwningWorld && OwningWorld->GetActiveLevelCollection())
		{
			NewTimerData.LevelCollection = OwningWorld->GetActiveLevelCollection()->GetType();
		}

		const float FirstDelay = (InFirstDelay >= 0.f) ? InFirstDelay : InRate;

		// 创建FTimerHandle并组装
		FTimerHandle NewTimerHandle;
		if (HasBeenTickedThisFrame())
		{
			NewTimerData.ExpireTime = InternalTime + FirstDelay;
			NewTimerData.Status = ETimerStatus::Active;
			NewTimerHandle = AddTimer(MoveTemp(NewTimerData));
			ActiveTimerHeap.HeapPush(NewTimerHandle, FTimerHeapOrder(Timers));
		}
		else
		{
			// Store time remaining in ExpireTime while pending
			NewTimerData.ExpireTime = FirstDelay;
			NewTimerData.Status = ETimerStatus::Pending;
			NewTimerHandle = AddTimer(MoveTemp(NewTimerData));
			PendingTimerSet.Add(NewTimerHandle);
		}
		// 返回Handle
		InOutHandle = NewTimerHandle;
	}
	else
	{
		InOutHandle.Invalidate();
	}
}

// 判断当前帧是否执行过Tick了
bool FORCEINLINE HasBeenTickedThisFrame() const
{
	return (LastTickedFrame == GFrameCounter);
}
```
3 主要目的是通过AddTimer创建FTimerHandle，并为其组装参数。需要注意的可能就是如果在当前帧已经执行过TimerManager的Tick了，就添加到ActiveTimerHeap这个数组里（可能是个堆结构）让其下一帧执行，如果当前帧还没执行过TimerManager，就添加到PendingTimerSet这个数组里，等到当前帧执行的时候在添加到ActiveTimerHeap这个数组里让其下一帧执行。

```cpp
FTimerHandle FTimerManager::AddTimer(FTimerData&& TimerData)
{
	// 绑定Timer代理的那个UObject
	const void* TimerIndicesByObjectKey = TimerData.TimerDelegate.GetBoundObject();
	TimerData.TimerIndicesByObjectKey = TimerIndicesByObjectKey;

	// 将timerData添加到timers数组里
	int32 NewIndex = Timers.Add(MoveTemp(TimerData));

	FTimerHandle Result = GenerateHandle(NewIndex);
	Timers[NewIndex].Handle = Result;

	if (TimerIndicesByObjectKey)
	{
		// 添加一个map，key是UObject，value是TimerHandle
		TSet<FTimerHandle>& HandleSet = ObjectToTimers.FindOrAdd(TimerIndicesByObjectKey);

		bool bAlreadyExists = false;
		HandleSet.Add(Result, &bAlreadyExists);
	}

	return Result;
}
```
4 字面意思，不复杂，就是添加进缓存，取得时候方便。
# 7 TimerManager执行Tick

1 这种TimerTick的方式是在RunTickGroup之后执行，也是在World::Tick的方法里面，但是Level的CollectionType得是ELevelCollectionType::DynamicSourceLevels这种类型，具体什么样的Level符合要求后面再看。
```cpp
/** Indicates the type of a level collection, used in FLevelCollection. */
enum class ELevelCollectionType : uint8
{
	/**
	 * The dynamic levels that are used for normal gameplay and the source for any duplicated collections.
	 * Will contain a world's persistent level and any streaming levels that contain dynamic or replicated gameplay actors.
	 */
	DynamicSourceLevels,

	/** Gameplay relevant levels that have been duplicated from DynamicSourceLevels if requested by the game. */
	DynamicDuplicatedLevels,

	/**
	 * These levels are shared between the source levels and the duplicated levels, and should contain
	 * only static geometry and other visuals that are not replicated or affected by gameplay.
	 * These will not be duplicated in order to save memory.
	 */
	StaticLevels,

	MAX
};
```
2 这是这个ELevelCollectionType枚举。

```cpp
if (TickType != LEVELTICK_TimeOnly && !bIsPaused)
{
	SCOPE_TIME_GUARD_MS(TEXT("UWorld::Tick - TimerManager"), 5);
	STAT(FScopeCycleCounter Context(GetTimerManager().GetStatId());)
	// 具体执行Tick的起始
	GetTimerManager().Tick(DeltaSeconds);
}
```
3 具体执行Tick的方法，在RunTickGroup之后进行。

```cpp
void FTimerManager::Tick(float DeltaTime)
{
	// 去掉了宏
	
	// 这一帧已经处理了，就提前返回
	if (HasBeenTickedThisFrame())
	{
		return;
	}
	// 开始Tick的事件
	const double StartTime = FPlatformTime::Seconds();
	bool bDumpTimerLogsThresholdExceeded = false;
	int32 NbExpiredTimers = 0;
	// 每次Tick间隔的累计
	InternalTime += DeltaTime;

	UWorld* const OwningWorld = OwningGameInstance ? OwningGameInstance->GetWorld() : nullptr;
	UWorld* const LevelCollectionWorld = OwningWorld;

	// 去掉了宏

	while (ActiveTimerHeap.Num() > 0)
	{
		// 这部分都是处理ActiveTimerHeap这个数组的，后面介绍，这里省略掉
	}
	
	// Timer has been ticked.
	// 记录当前帧已经Tick过了
	LastTickedFrame = GFrameCounter;

	// If we have any Pending Timers, add them to the Active Queue.
	// 把PendingTimerSet数组里的数据添加到ActiveTimerHeap数组中
	if( PendingTimerSet.Num() > 0 )
	{
		for (FTimerHandle Handle : PendingTimerSet)
		{
			FTimerData& TimerToActivate = GetTimer(Handle);

			// Convert from time remaining back to a valid ExpireTime
			TimerToActivate.ExpireTime += InternalTime;
			TimerToActivate.Status = ETimerStatus::Active;
			ActiveTimerHeap.HeapPush( Handle, FTimerHeapOrder(Timers) );
		}
		PendingTimerSet.Reset();
	}
}
```
4 执行Tick的逻辑，先是处理激活的Timer（ActiveTimerHeap），然后记录当前帧号，处理当前帧没来得及处理的Timer（PendingTimerSet这个数组里的），将其放入ActiveTimerHeap数组中，等待下一帧执行。

```cpp
while (ActiveTimerHeap.Num() > 0)
{
	// 拿到堆顶的Handle
	FTimerHandle TopHandle = ActiveTimerHeap.HeapTop();

	// Test for expired timers
	// 拿到堆顶的TimerData
	int32 TopIndex = TopHandle.GetIndex();
	FTimerData* Top = &Timers[TopIndex];
	// 如果是ActivePendingRemoval这种状态就移除timer
	if (Top->Status == ETimerStatus::ActivePendingRemoval)
	{
		ActiveTimerHeap.HeapPop(TopHandle, FTimerHeapOrder(Timers), /*bAllowShrinking=*/ false);
		RemoveTimer(TopHandle);
		continue;
	}
	// 如果当前累计的事件 > Timer的过期事件，也就是到时间了可以执行Timer了
	if (InternalTime > Top->ExpireTime)
	{
		// 这几行不关键也不知道干嘛的

		// Set the relevant level context for this timer
		const int32 LevelCollectionIndex = OwningWorld ? OwningWorld->FindCollectionIndexByType(Top->LevelCollection) : INDEX_NONE;
		
		FScopedLevelCollectionContextSwitch LevelContext(LevelCollectionIndex, LevelCollectionWorld);

		// Remove it from the heap and store it while we're executing
		// 从堆中pop出我们timer
		ActiveTimerHeap.HeapPop(CurrentlyExecutingTimer, FTimerHeapOrder(Timers), /*bAllowShrinking=*/ false);
		// 改变我们Timer的状态
		Top->Status = ETimerStatus::Executing;

		// Determine how many times the timer may have elapsed (e.g. for large DeltaTime on a short looping timer)
		// 计算Timer执行的次数，如果Timer间隔短帧间隔长，CallCount次数会变多？
		// 如果是循环执行，那肯定到时间就执行，所以可能一帧执行多次
		int32 const CallCount = Top->bLoop ? 
			FMath::TruncToInt( (InternalTime - Top->ExpireTime) / Top->Rate ) + 1
			: 1;

		// 删掉宏，不知道干嘛的

		// Now call the function
		for (int32 CallIdx=0; CallIdx<CallCount; ++CallIdx)
		{ 
			// 广播我们的那个TimerDelegate，在创建时候绑定的
			Top->TimerDelegate.Execute();

			// Update Top pointer, in case it has been invalidated by the Execute call
			// 看英文注释意思是，更新当前的TimerData指针，防止Delegate里面把Top弄没
			Top = FindTimer(CurrentlyExecutingTimer);
			
			// 如果Timer没有了或者是状态不对就结束执行
			if (!Top || Top->Status != ETimerStatus::Executing)
			{
				break;
			}
		}
		// test to ensure it didn't get cleared during execution
		if (Top)
		{
			// if timer requires a delegate, make sure it's still validly bound (i.e. the delegate's object didn't get deleted or something)
			// 如果循环Timer，就更改Timer的过期时间，在添加回ActiveTimerHeap数组中
			if (Top->bLoop && (!Top->bRequiresDelegate || Top->TimerDelegate.IsBound()))
			{
				// Put this timer back on the heap
				Top->ExpireTime += CallCount * Top->Rate;
				Top->Status = ETimerStatus::Active;
				ActiveTimerHeap.HeapPush(CurrentlyExecutingTimer, FTimerHeapOrder(Timers));
			}
			else
			{
				// 不是循环timer，就直接移除掉了
				RemoveTimer(CurrentlyExecutingTimer);
			}

			CurrentlyExecutingTimer.Invalidate();
		}
	}
	else
	{
		// Top的timer的时间都没到，后面的timer肯定也到不了，就结束ActiveTimerHeap遍历
		// no need to go further down the heap, we can be finished
		break;
	}
}
```
5 这个方法就是执行Timer的部分，总的来说就是获取堆顶的Timer，然后判断时间是否到期然后执行对应的Delegate，还需要判断Timer是否循环，如果循环就更新ExpireTime时间。
# 8 TickableGameObject添加
1 这个添加部分就先就是继承FTickableGameObject这个类，然后会在构造函数时添加
```cpp
struct FTickableStatics
{
	// TickableObjects互斥量
	FCriticalSection TickableObjectsCritical;
	// 保存TickableObject的数组，每个元素时简化版的FTickableGameObject
	TArray<FTickableObjectBase::FTickableObjectEntry> TickableObjects;
	// NewTickableObjects的互斥量
	FCriticalSection NewTickableObjectsCritical;
	// 保存NewTickableObject的数组，每个元素是FTickableGameObject指针
	TSet<FTickableGameObject*> NewTickableObjects;
	// 标志位，标记这个TickGameObject是否正在Tick
	bool bIsTickingObjects = false;
	// 向NewTickableObjects数组中添加数据
	void QueueTickableObjectForAdd(FTickableGameObject* InTickable)
	{
		// 给互斥量上锁，感觉上类似
		// std::lock_guard<std::mutex> lk(NewTickableObjectsCritical)
		FScopeLock NewTickableObjectsLock(&NewTickableObjectsCritical);
		NewTickableObjects.Add(InTickable);
	}
	// 向NewTickableObjects数组中移除
	void RemoveTickableObjectFromNewObjectsQueue(FTickableGameObject* InTickable)
	{
		FScopeLock NewTickableObjectsLock(&NewTickableObjectsCritical);
		NewTickableObjects.Remove(InTickable);
	}
	// 获取单例
	static FTickableStatics& Get()
	{
		static FTickableStatics Singleton;
		return Singleton;
	}
};
FTickableGameObject::FTickableGameObject()
{
	FTickableStatics& Statics = FTickableStatics::Get();

	if (UObjectInitialized())
	{
		Statics.QueueTickableObjectForAdd(this);
	}
	else
	{
		AddTickableObject(Statics.TickableObjects, this);
	}
}
```
2 首先获取个静态变量，是个静态单例，充当个容器的作用，保存FTickableGameObject。然后通过这个UObject是否初始化来添加到不同的数组里面。

```cpp
FTickableGameObject::~FTickableGameObject()
{
	FTickableStatics& Statics = FTickableStatics::Get();	
	Statics.RemoveTickableObjectFromNewObjectsQueue(this);	
	FScopeLock LockTickableObjects(&Statics.TickableObjectsCritical);
	RemoveTickableObject(Statics.TickableObjects, this, Statics.bIsTickingObjects);
}
```
3 他还有个虚构函数，就是简单的从static数组中移除先前添加进去的GameObject。
# 9 TickableGameObject执行
```cpp
// EditotEngine中执行
if (bAWorldTicked)
{
	FTickableGameObject::TickObjects(nullptr, TickType, false, DeltaSeconds);
}
// GameEngine中执行
{  
   SCOPE_TIME_GUARD(TEXT("UGameEngine::Tick - TickObjects"));  
   FTickableGameObject::TickObjects(nullptr, LEVELTICK_All, false, DeltaSeconds);  
}
// World::Tick中RunTickGroup之后执行
{
	SCOPE_TIME_GUARD_MS(TEXT("UWorld::Tick - TickObjects"), 5);
	FTickableGameObject::TickObjects(this, TickType, bIsPaused, DeltaSeconds);
}
```
1 有三处执行的地方，都一一列举出来了，需要注意的可能就是wold为nullptr的情况，分成了Editor和Game两个执行的地方。
```cpp
{
	SCOPE_TIME_GUARD_MS(TEXT("UWorld::Tick - TickObjects"), 5);
	FTickableGameObject::TickObjects(this, TickType, bIsPaused, DeltaSeconds);
}
```
2 具体执行tick是在RunTickGroup和TimerManager的Tick之后执行。
```cpp
void FTickableGameObject::TickObjects(UWorld* World, const int32 InTickType, const bool bIsPaused, const float DeltaSeconds)
{
	SCOPE_CYCLE_COUNTER(STAT_TickableGameObjectsTime);
	CSV_SCOPED_TIMING_STAT_EXCLUSIVE(Tickables);

	FTickableStatics& Statics = FTickableStatics::Get();

	check(IsInGameThread());

	// It's a long lock but it's ok, the only thing we can block here is the GC worker thread that destroys UObjects
	FScopeLock LockTickableObjects(&Statics.TickableObjectsCritical);

	{
		// 首先对NewTickableObjectsCritical上锁，防止被修改
		FScopeLock NewTickableObjectsLock(&Statics.NewTickableObjectsCritical);
		// 遍历NewTickableObjects数组，将里面的全都放到TickableObjects这个数组里面
		for (FTickableGameObject* NewTickableObject : Statics.NewTickableObjects)
		{
			AddTickableObject(Statics.TickableObjects, NewTickableObject);
		}
		// 清空NewTickableObjects数组
		Statics.NewTickableObjects.Empty();
	}// 这行执行完后，解锁NewTickableObjectsCritical

	// 判断Tick数据里是否为空
	if (Statics.TickableObjects.Num() > 0)
	{
		check(!Statics.bIsTickingObjects);
		// 修改标志位
		Statics.bIsTickingObjects = true;

		bool bNeedsCleanup = false;
		const ELevelTick TickType = (ELevelTick)InTickType;
		// 开始遍历数组
		for (const FTickableObjectEntry& TickableEntry : Statics.TickableObjects)
		{
			if (FTickableGameObject* TickableObject = static_cast<FTickableGameObject*>(TickableEntry.TickableObject))
			{
				// If it is tickable and in this world
				// 条件判断
				// Tick类型是总是Tick or IsTickable
				// TickableGameObjectWorld的world是当前world
				// TickableObject允许Tick
				if (((TickableEntry.TickType == ETickableTickType::Always) || TickableObject->IsTickable()) 
					&& (TickableObject->GetTickableGameObjectWorld() == World)
					&& TickableObject->IsAllowedToTick())
				{
					const bool bIsGameWorld = InTickType == LEVELTICK_All || (World && World->IsGameWorld());
					// 一堆条件判断
					if ((GIsEditor && TickableObject->IsTickableInEditor()) ||
						(bIsGameWorld && ((!bIsPaused && TickType != LEVELTICK_TimeOnly) || (bIsPaused && TickableObject->IsTickableWhenPaused()))))
					{
						// 性能统计的？不知道干啥的
						FScopeCycleCounter Context(TickableObject->GetStatId());
						// 执行TickableObject的Tick方法
						TickableObject->Tick(DeltaSeconds);

						// In case it was removed during tick
						// 如果在Tick过程中，被删掉了
						if (TickableEntry.TickableObject == nullptr)
						{
							bNeedsCleanup = true;
						}
					}
				}
			}
			else
			{
				bNeedsCleanup = true;
			}
		}
		// 移除所有的TickableObject
		if (bNeedsCleanup)
		{
			Statics.TickableObjects.RemoveAll([](const FTickableObjectEntry& Entry) { return Entry.TickableObject == nullptr; });
		}
		// 改变正在Tick的标志位
		Statics.bIsTickingObjects = false;
	}
}
```
3 逻辑很清晰简单，总的来说就是遍历TickableObjects这个数组，做一些能否Tick的条件判断，如果能Tick就执行TickableObject的Tick方法了。
# 10 TickableGameObject使用
1 游戏中技能cd就是通过TickableGameObject这个实现的
2 我们创建自己的类FLuaCoolDownUserObject 继承 FTickableGameObject，然后重写Tick方法
3 FLuaCoolDownUserObject还可以继承FUserObject，FUserObject这个是UImage中的内部类
4 在FTickableGameObject的Tick中通过id获取CoolDownItem这个类，然后通过CoolDownItem里面的数据更新UImage
5 CoolDownItem这个里面的数据是通过监听OnWorldTickStart这个代理来更新的，也就是World开始Tick的时候就会更新cd的时间数据，然后再通过TickableGameObject的Tick来更新cd的图标
6 总的来说就是，我们会把技能cd数据和技能cd图标分开表示，数据的更新再Worl::Tick的开始的时候，图标的更新再数据更新之后的FTickableGameObject::Tick中。

# 问题
- tick依赖用多了会怎么样？会出现循环依赖的问题么？
	a.循环依赖的这样，a依赖b，b又依赖a
	b.性能问题，本身tick没有时序，可以多线程加速并行，但是依赖的多了就变成同步的了

![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250505221418.png)
1 应该是不会循环依赖的，我们是在QueueTickFunction方法中递归的执行Prerequisites依赖数组的，这里每个tickFunction会记录执行的帧号，如果帧号不同才会递归执行，所以应该是不会循环依赖的。但如果依赖的话就会导致一方的tick逻辑不会执行。
2 依赖用多了就会有性能问题，本身tick是通过多线程这边创建task执行的，但是依赖多了很自然就变成同步的了，性能肯定不好吧。


# 额外
```cpp
// 还有这种添加tick的方式
TickHandle = FTSTicker::GetCoreTicker().AddTicker(FTickerDelegate::CreateUObject(this, &UHTCityEventSubsystem::Tick), 0.0f);

FTSTicker::GetCoreTicker().RemoveTicker(TickHandle);
```