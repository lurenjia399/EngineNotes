# 1 World的创建流程
1 我们从windows的启动开始看
```cpp
// LaunchWindows.cpp文件中的main函数
int32 WINAPI WinMain(_In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char* pCmdLine, _In_ int32 nCmdShow)
{
	int32 Result = LaunchWindowsStartup(hInInstance, hPrevInstance, pCmdLine, nCmdShow, nullptr);
	LaunchWindowsShutdown();
	return Result;
}
LAUNCH_API int32 LaunchWindowsStartup( HINSTANCE hInInstance, HINSTANCE hPrevInstance, char*, int32 nCmdShow, const TCHAR* CmdLine )
{
	// 这里面很多方法，但是往后走的就是这个方法
	ErrorLevel = GuardedMain( CmdLine );
}

```
2 不同平台都有GuardedMain这个方法，下面看看这个windows的这个
```cpp
int32 GuardedMain( const TCHAR* CmdLine )
{
#if !(UE_BUILD_SHIPPING)
	if (FParse::Param(CmdLine, TEXT("waitforattach")))
	{
		while (!FPlatformMisc::IsDebuggerPresent());
		UE_DEBUG_BREAK();
	}
#endif

	BootTimingPoint("DefaultMain");

	// Super early init code. DO NOT MOVE THIS ANYWHERE ELSE!
	FCoreDelegates::GetPreMainInitDelegate().Broadcast();

	// make sure GEngineLoop::Exit() is always called.
	// 这个是方法中创建了个CleanupGuard对象，在这个方法结束的时候会调到析构函数
	struct EngineLoopCleanupGuard 
	{ 
		~EngineLoopCleanupGuard()
		{
			// Don't shut down the engine on scope exit when we are running embedded
			// because the outer application will take care of that.
			if (!GUELibraryOverrideSettings.bIsEmbedded)
			{
				EngineExit();
			}
		}
	} CleanupGuard;
	
	// 删掉这个宏，不知道干啥的

	// 引擎初始化前的初始化
	int32 ErrorLevel = EnginePreInit( CmdLine );

	// exit if PreInit failed.
	if ( ErrorLevel != 0 || IsEngineExitRequested() )
	{
		return ErrorLevel;
	}

	{// 还记得这个写法么，{}括号括起来，然后对同步量上锁，在括号结束的时候解锁
		
		FScopedSlowTask SlowTask(100, NSLOCTEXT("EngineInit", "EngineInit_Loading", "Loading..."));

		SlowTask.EnterProgressFrame(80);

		SlowTask.EnterProgressFrame(20);
	// WITH_EDITOR 这个宏的意思在编译的时候，我们是否用editer编译
#if WITH_EDITOR
		// 这个GIsEditor变量是说我们是否在编辑器模式下启动游戏
		if (GIsEditor)
		{
			ErrorLevel = EditorInit(GEngineLoop);
		}
		else
#endif
		{
			ErrorLevel = EngineInit();
		}
	}

	// 时间的代码，不清楚干嘛的，删掉了

	// Don't tick if we're running an embedded engine - we rely on the outer
	// application ticking us instead.
	// 这部分就是EngineTick了
	if (!GUELibraryOverrideSettings.bIsEmbedded)
	{
		while( !IsEngineExitRequested() )
		{
			EngineTick();
		}
	}

	TRACE_BOOKMARK(TEXT("Tick loop end"));

	// 如果编译了并且是Editor启动，就退出Editor
#if WITH_EDITOR
	if( GIsEditor )
	{
		EditorExit();
	}
#endif
	return ErrorLevel;
}

```
3 下面我们看EngineInit方法，就不看Editor部分了，看Game部分
```cpp
int32 EngineInit()
{
	int32 ErrorLevel = GEngineLoop.Init();

	return( ErrorLevel );
}

int32 FEngineLoop::Init()
{
	// 省略调一些不关心的方法

	// Figure out which UEngine variant to use.
	// 这部分就是创建Enigne，有UGameEngine和UUnrealEdEngine的区别，
	// 可以在DefaultEngine.ini文件中配置
	UClass* EngineClass = nullptr;
	if( !GIsEditor )
	{
		SCOPED_BOOT_TIMING("Create GEngine");
		// We're the game.
		FString GameEngineClassName;
		GConfig->GetString(TEXT("/Script/Engine.Engine"), TEXT("GameEngine"), GameEngineClassName, GEngineIni);
		EngineClass = StaticLoadClass( UGameEngine::StaticClass(), nullptr, *GameEngineClassName);
		if (EngineClass == nullptr)
		{
			UE_LOG(LogInit, Fatal, TEXT("Failed to load UnrealEd Engine class '%s'."), *GameEngineClassName);
		}
		GEngine = NewObject<UEngine>(GetTransientPackage(), EngineClass);
	}
	else
	{
#if WITH_EDITOR
		// We're UnrealEd.
		FString UnrealEdEngineClassName;
		GConfig->GetString(TEXT("/Script/Engine.Engine"), TEXT("UnrealEdEngine"), UnrealEdEngineClassName, GEngineIni);
		EngineClass = StaticLoadClass(UUnrealEdEngine::StaticClass(), nullptr, *UnrealEdEngineClassName);
		if (EngineClass == nullptr)
		{
			UE_LOG(LogInit, Fatal, TEXT("Failed to load UnrealEd Engine class '%s'."), *UnrealEdEngineClassName);
		}
		GEngine = GEditor = GUnrealEd = NewObject<UUnrealEdEngine>(GetTransientPackage(), EngineClass);
#else
		check(0);
#endif
	}

	FEmbeddedCommunication::ForceTick(11);
	check( GEngine );
	// 这个可能是loading的设计？后面看下这个movieplayer
	GetMoviePlayer()->PassLoadingScreenWindowBackToGame();
    if (FPreLoadScreenManager::Get())
    {
        FPreLoadScreenManager::Get()->PassPreLoadScreenWindowBackToGame();
    }

	{// 上锁？
		SCOPED_BOOT_TIMING("GEngine->ParseCommandline()");
		GEngine->ParseCommandline();
	}

	FEmbeddedCommunication::ForceTick(12);

	{// 初始化引擎的时间
		SCOPED_BOOT_TIMING("InitTime");
		InitTime();
	}

	SlowTask.EnterProgressFrame(60);

	{// 引擎的init，后面展开说下
		SCOPED_BOOT_TIMING("GEngine->Init");
		GEngine->Init(this);
	}

	// Call init callbacks
	FCoreDelegates::OnPostEngineInit.Broadcast();

	SlowTask.EnterProgressFrame(30);

	// initialize engine instance discovery
	// 不知道这部分干啥的
	if (FPlatformProcess::SupportsMultithreading())
	{
		SCOPED_BOOT_TIMING("SessionService etc");
		if (!IsRunningCommandlet())
		{
			SessionService = FModuleManager::LoadModuleChecked<ISessionServicesModule>("SessionServices").GetSessionService();
			
			if (SessionService.IsValid())
			{
			SessionService->Start();
		}
		}

		EngineService = new FEngineService();
	}
	// 不知道这部分干啥的
	{
		SCOPED_BOOT_TIMING("IProjectManager::Get().LoadModulesForProject(ELoadingPhase::PostEngineInit)");
		// Load all the post-engine init modules
		if (!IProjectManager::Get().LoadModulesForProject(ELoadingPhase::PostEngineInit) || !IPluginManager::Get().LoadModulesForEnabledPlugins(ELoadingPhase::PostEngineInit))
		{
			RequestEngineExit(TEXT("One or more modules failed PostEngineInit"));
			return 1;
		}
	}
	// Engine的start方法
	// GameEngine的里面走的是StartGameInstance
	{
		SCOPED_BOOT_TIMING("GEngine->Start()");
		GEngine->Start();
	}

	FEmbeddedCommunication::ForceTick(13);

	// 下面这部分还是loading相关的，后面再说
    if (FPreLoadScreenManager::Get() && FPreLoadScreenManager::Get()->HasActivePreLoadScreenType(EPreLoadScreenTypes::EngineLoadingScreen))
    {
		SCOPED_BOOT_TIMING("WaitForEngineLoadingScreenToFinish");
		FPreLoadScreenManager::Get()->SetEngineLoadingComplete(true);
        FPreLoadScreenManager::Get()->WaitForEngineLoadingScreenToFinish();
    }
    else
    {
		SCOPED_BOOT_TIMING("WaitForMovieToFinish");
		GetMoviePlayer()->WaitForMovieToFinish();
    }
	// 不了解这个方法
	FTraceAuxiliary::EnableChannels();

	// 不知道这部分干啥的,去掉把

// 在编辑器模式下，加在几个模块
#if WITH_EDITOR
	if (GIsEditor)
	{
		FModuleManager::Get().LoadModule(TEXT("ProfilerClient"));
	}

	FModuleManager::Get().LoadModule(TEXT("SequenceRecorder"));
	FModuleManager::Get().LoadModule(TEXT("SequenceRecorderSections"));
#endif
	// 设置标志位
	GIsRunning = true;
	// 不是编辑器，开始viewport渲染?
	if (!GIsEditor)
	{
		// hide a couple frames worth of rendering
		FViewport::SetGameRenderingEnabled(true, 3);
	}

	FEmbeddedCommunication::ForceTick(15);
	// 后面就全是些云里雾里的操作了，有机会看吧
	return 0;
}
```
4 这个FEngineLoop::Init方法主要的作用就是，1 创建出UGameEngine或者UUnrealEdEngine。2然后做一些初始化以及Engine::init方法。3 还有一部分是movieplayer的部分，这部分好像是loading相关，后面看，下面看下UGameEngine::init方法。
```cpp
void UGameEngine::Init(IEngineLoop* InEngineLoop)
{
	DECLARE_SCOPE_CYCLE_COUNTER(TEXT("UGameEngine Init"), STAT_GameEngineStartup, STATGROUP_LoadTime);
	

	// Call base.
	// 调用基类，里面也一堆东西，后面再看吧
	UEngine::Init(InEngineLoop);
// 不知道这个干嘛的
#if USE_NETWORK_PROFILER
	FString NetworkProfilerTag;
	if( FParse::Value(FCommandLine::Get(), TEXT("NETWORKPROFILER="), NetworkProfilerTag ) )
	{
		GNetworkProfiler.EnableTracking(true);
	}
#endif

	// Load and apply user game settings
	GetGameUserSettings()->LoadSettings();
	GetGameUserSettings()->ApplyNonResolutionSettings();

	// Create game instance.  For GameEngine, this should be the only GameInstance that ever gets created.
	// 根据GameMapsSettings配置生成对应的gameInstance，也是在DefaultEngine.ini里面配的
	{
		FSoftClassPath GameInstanceClassName = GetDefault<UGameMapsSettings>()->GameInstanceClass;
		UClass* GameInstanceClass = (GameInstanceClassName.IsValid() ? LoadObject<UClass>(NULL, *GameInstanceClassName.ToString()) : UGameInstance::StaticClass());
		
		if (GameInstanceClass == nullptr)
		{
			UE_LOG(LogEngine, Error, TEXT("Unable to load GameInstance Class '%s'. Falling back to generic UGameInstance."), *GameInstanceClassName.ToString());
			GameInstanceClass = UGameInstance::StaticClass();
		}
		// 初始化GameInstance，后面看下这部分
		GameInstance = NewObject<UGameInstance>(this, GameInstanceClass);
		GameInstance->InitializeStandalone();
	}
 
	// 这部分不太了解，去掉了

	// Initialize the viewport client.
	// 初始化viewport
	UGameViewportClient* ViewportClient = NULL;
	// 不确定这个标志位
	if(GIsClient)
	{
		ViewportClient = NewObject<UGameViewportClient>(this, GameViewportClientClass);
		ViewportClient->Init(*GameInstance->GetWorldContext(), GameInstance);
		GameViewport = ViewportClient;
		GameInstance->GetWorldContext()->GameViewport = ViewportClient;
	}

	LastTimeLogsFlushed = FPlatformTime::Seconds();

	// Attach the viewport client to a new viewport.
	// 不确定这个是否会走
	if(ViewportClient)
	{
		// This must be created before any gameplay code adds widgets
		bool bWindowAlreadyExists = GameViewportWindow.IsValid();
		if (!bWindowAlreadyExists)
		{
			UE_LOG(LogEngine, Log, TEXT("GameWindow did not exist.  Was created"));
			GameViewportWindow = CreateGameWindow();
		}

		CreateGameViewport( ViewportClient );

		if( !bWindowAlreadyExists )
		{
			SwitchGameWindowToUseGameViewport();
		}

		FString Error;
		if(ViewportClient->SetupInitialLocalPlayer(Error) == NULL)
		{
			UE_LOG(LogEngine, Fatal,TEXT("%s"),*Error);
		}

		UGameViewportClient::OnViewportCreated().Broadcast();
	}

	UE_LOG(LogInit, Display, TEXT("Game Engine Initialized.") );

	// for IsInitialized()
	bIsInitialized = true;
}
```
5 UGameEngine::init方法里面不是特别的复杂，主要就是找到需要创建的GameInstance，然后创建出来进而初始化，后面就是创建ViewportClient这个东西了。下面看下GameInstance创建部分。
```cpp
void UGameInstance::InitializeStandalone(const FName InPackageName, UPackage* InWorldPackage)
{
	// Creates the world context. This should be the only WorldContext that ever gets created for this GameInstance.
	// 第一步，创建WorldContext
	WorldContext = &GetEngine()->CreateNewWorldContext(EWorldType::Game);
	WorldContext->OwningGameInstance = this;

	// In standalone create a dummy world from the beginning to avoid issues of not having a world until LoadMap gets us our real world
	// 第二步，创建DummyWorld，虚假的World，知道LoadMap加在真正的world
	UWorld* DummyWorld = UWorld::CreateWorld(EWorldType::Game, false, InPackageName, InWorldPackage);
	DummyWorld->SetGameInstance(this);
	WorldContext->SetCurrentWorld(DummyWorld);
	
	// 第三步，GameInstance初始化操作
	Init();
}

```
6 主要就是分成了三部分。
6.1 创建WorldContext
```cpp
// 第一步
FWorldContext& UEngine::CreateNewWorldContext(EWorldType::Type WorldType)
{
	// 直接new的
	FWorldContext* NewWorldContext = new FWorldContext;
	// 将WorldContext添加到了WorldList里面
	WorldList.Add(NewWorldContext);
	// 设置参数
	NewWorldContext->WorldType = WorldType;
	NewWorldContext->ContextHandle = FName(*FString::Printf(TEXT("Context_%d"), NextWorldContextHandle++));

	return *NewWorldContext;
}
```
6.2 UWorld::CreateWorld方法创建DummyWorld，虚假的World，直到LoadMap加在真正的world
```cpp
UWorld* UWorld::CreateWorld(const EWorldType::Type InWorldType, bool bInformEngineOfWorld, FName WorldName, UPackage* InWorldPackage, bool bAddToRoot, ERHIFeatureLevel::Type InFeatureLevel)
{
	// 设置了一些状态
	if (InFeatureLevel >= ERHIFeatureLevel::Num)
	{
		InFeatureLevel = GMaxRHIFeatureLevel;
	}
	UPackage* WorldPackage = InWorldPackage;
	if ( !WorldPackage )
	{
		WorldPackage = CreatePackage(nullptr);
	}
	if (InWorldType == EWorldType::PIE)
	{
		WorldPackage->SetPackageFlags(PKG_PlayInEditor);
	}
	// Mark the package as containing a world.  This has to happen here rather than at serialization time,
	// so that e.g. the referenced assets browser will work correctly.
	if ( WorldPackage != GetTransientPackage() )
	{
		WorldPackage->ThisContainsMap();
	}

	// Create new UWorld, ULevel and UModel.
	const FString WorldNameString = (WorldName != NAME_None) ? WorldName.ToString() : TEXT("Untitled");
	UWorld* NewWorld = NewObject<UWorld>(WorldPackage, *WorldNameString);
	NewWorld->SetFlags(RF_Transactional);
	NewWorld->WorldType = InWorldType;
	NewWorld->FeatureLevel = InFeatureLevel;
	// 这个方法里会创建PersistentLevel
	NewWorld->InitializeNewWorld(UWorld::InitializationValues().ShouldSimulatePhysics(false).EnableTraceCollision(true).CreateNavigation(InWorldType == EWorldType::Editor).CreateAISystem(InWorldType == EWorldType::Editor));

	// Clear the dirty flag set during SpawnActor and UpdateLevelComponents
	WorldPackage->SetDirtyFlag(false);

	if ( bAddToRoot )
	{
		// Add to root set so it doesn't get garbage collected.
		NewWorld->AddToRoot();
	}

	// Tell the engine we are adding a world (unless we are asked not to)
	if( ( GEngine ) && ( bInformEngineOfWorld == true ) )
	{
		// 广播事件
		GEngine->WorldAdded( NewWorld );
	}

	return NewWorld;
}
```
6.3 GameInstance初始化操作
```cpp
void UGameInstance::Init()
{
	// 蓝图实现这个方法，蓝图调用
	ReceiveInit();

	if (!IsRunningCommandlet())
	{
		// 创建UOnlineSession
		UClass* SpawnClass = GetOnlineSessionClass();
		OnlineSession = NewObject<UOnlineSession>(this, SpawnClass);
		if (OnlineSession)
		{
			OnlineSession->RegisterOnlineDelegates();
		}

		if (!IsDedicatedServerInstance())
		{
			TSharedPtr<GenericApplication> App = FSlateApplication::Get().GetPlatformApplication();
			if (App.IsValid())
			{
				App->RegisterConsoleCommandListener(GenericApplication::FOnConsoleCommandListener::CreateUObject(this, &ThisClass::OnConsoleInput));
			}
		}

		FNetDelegates::OnReceivedNetworkEncryptionToken.BindUObject(this, &ThisClass::ReceivedNetworkEncryptionToken);
		FNetDelegates::OnReceivedNetworkEncryptionAck.BindUObject(this, &ThisClass::ReceivedNetworkEncryptionAck);
	}

	SubsystemCollection.Initialize(this);
}
```
7 接下来就是start的部分，也是从FEngineLoop::Init方法中调用。就是创建真正的默认world
```cpp
{
	SCOPED_BOOT_TIMING("GEngine->Start()");
	// 看上去只有GameEngine有这个start，EditorEngine没有
	GEngine->Start();
}
void UGameEngine::Start()
{
	UE_LOG(LogInit, Display, TEXT("Starting Game."));

	GameInstance->StartGameInstance();
}
```
8 接下来看下GameInstance中的StartGameInstance这个方法
```cpp
void UGameInstance::StartGameInstance()
{
	UEngine* const Engine = GetEngine();

	// 前边全省略了，不关心
	// 这个就是读UGameMapsSetting这个配置了，也在DefaultEngine.ini里有
	const UGameMapsSettings* GameMapsSettings = GetDefault<UGameMapsSettings>();
	// 拿到GameDefaultMap
	const FString& DefaultMap = GameMapsSettings->GetGameDefaultMap();

	FString PackageName;
	// 这个判断，是否通过命令行来指定map，
	//路径/ALSVServer-Win64-DebugGame.exe Game/ThirdPersonCPP/Maps/ThirdPersonExampleMap -log
	// 如果没有命令行，就从DefaultEngine.ini里取
	if (!FParse::Token(Tmp, PackageName, 0) || **PackageName == '-')
	{
		PackageName = DefaultMap + GameMapsSettings->LocalMapOptions;
	}
	// 将定好的map填入到url中
	FURL URL(&DefaultURL, *PackageName, TRAVEL_Partial);
	if (URL.Valid)
	{
		// 这个方法就是加载我们的map
		BrowseRet = Engine->Browse(*WorldContext, URL, Error);
	}
	// If waiting for a network connection, go into the starting level.
	if (BrowseRet == EBrowseReturnVal::Failure)
	{
		// 这边没啥东西，就是加在地图失败的处理
	}
	// Handle failure.
	if (BrowseRet == EBrowseReturnVal::Failure)
	{
		// 这边也是，加载地图失败了弹窗
		FMessageDialog::Open(EAppMsgType::Ok, Message);
		FPlatformMisc::RequestExit(false);
		return;
	}
	// 广播OnStartGameInstance事件
	BroadcastOnStart();
}

```
9 下面我们看看默认的World具体是怎么创建出来的，也就是通过UEngine::Browse这个方法会调到UEngine::LoadMap
```cpp
bool UEngine::LoadMap( FWorldContext& WorldContext, FURL URL, class UPendingNetGame* Pending, FString& Error )
{
	// 代码很长，这里简化下
	// 1 一堆宏，不看了
	// 2 做一些前置初始化，赋值bMapNeedLoad
	bool bMapNeedLoad = !URL.HasOption(TEXT("KeepWorld"));
	WorldContext.NewLoadMap = bMapNeedLoad;
	if (WorldContext.World())
	{
		WorldContext.World()->bIsLevelStreamingFrozen = false;
	}
	{
		FCoreUObjectDelegates::PreLoadMap.Broadcast(URL.Map);
	}
	// 3 创建了一个结构体的局部变量，作用是make sure there is a matching PostLoadMap() no matter how we exit
	// 4 判断if (bMapNeedLoad)，应该就是处理当前world中数据
	// 5 记录StartTime
	double  StartTime = FPlatformTime::Seconds();
	// 6 我们自己加了这个
	FWorldTraverseUtils WorldTraverseUtils(bMapNeedLoad);
	// 7 卸载当前的world
	{
		// 7.1 Clean up networking
		// 7.2 调用FlushLevelStreaming方法
		// 7.3 我们自己加了这个
		WorldTraverseUtils.OnPreChangeWorld(WorldContext.World(), nullptr);
		// 7.4 Disassociate the players from their PlayerControllers in this world.
		// 就是通过GameInstance找到playercontroler和pawn，都DestroyActor销毁掉
		// 7.5 销毁掉world里的所有Actor，有些例外
		// 7.6 Destroy all player states  in this world.
		// 就是WorldContext.World()->DestroyActor(PlayerState, true);这样删掉
		// 7.7 RouteEndPlay方法，还不知道干嘛的
		// 7.8 // Do this after destroying pawns/playercontrollers, in case that spawns new things (e.g. dropped weapons)
			WorldContext.World()->CleanupWorld();
		// 7.9 删掉当前world
		GEngine->WorldDestroyed(WorldContext.World());
		WorldContext.World()->RemoveFromRoot();
		// 7.9.1 标记world中的所有level和streaminglevel，后续让gc把它们干掉
		// 7.9.2 设置当前世界为nullptr
		WorldContext.SetCurrentWorld(nullptr);
	}
	// 8 PIE的操作，
	// If this world is a PIE instance, we need to check if we are traveling to another PIE instance's world.
	// If we are, we need to set the PIERemapPrefix so that we load a copy of that world, instead of loading the
	// PIE world directly.
	// 9 加载新的world
	{
		// 9,1 LoadPackage加载
		// 9.2 拿到newworld,是个空的？
		NewWorld = UWorld::FindWorldInPackage(WorldPackage);
		// 9.3 判断newworld的WorldType，如果是game,pie就不看了
		NewWorld = CreatePIEWorldByLoadingFromPackage(WorldContext, URL.Map, WorldPackage);
		// 9.4 字面意思，很重要的
		if (bMapNeedLoad)
		{
			NewWorld->SetGameInstance(WorldContext.OwningGameInstance);
			GWorld = NewWorld;
			WorldContext.SetCurrentWorld(NewWorld);
			WorldContext.World()->WorldType = WorldContext.WorldType;
			if (!WorldContext.World()->bIsWorldInitialized)
			{
				// InitWorld
				WorldContext.World()->InitWorld();
			}
		}
		WorldContext.World()->SetGameMode(URL);
		WorldContext.World()->CreateAISystem();
		WorldContext.World()->InitializeActorsForPlay(URL, true, &Context);
		FNavigationSystem::AddNavigationSystemToWorld(*WorldContext.World(), FNavigationSystemRunMode::GameMode);
		// 9.5 我们自己加了这个
		WorldTraverseUtils.OnPostChangeWorld(WorldContext.World());
		// 9.6 beginplay?
		WorldContext.World()->BeginPlay();
	}
}
```
10 至此，GameEngine的world创建流程是，我们的GameInstance经过InitializeStandalone方法创建出DummyWorld空的world，然后通过StartGameInstance方法调用到loadmap创建出默认的World。
11 还有个问题，他为什么要先创建个空的，然后再加载默认的地图呢？