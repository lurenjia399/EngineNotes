# 1 World的创建流程

![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202405132122388.png)

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
11 还有个问题，他为什么要先创建个空的（过渡地图），然后再加载默认的地图呢？
至于为什么需要设置过渡地图？试想如果没有过渡地图，那么在当前地图中加载（异步加载的，后面会详细介绍）的新地图之后，此时World同时存在新旧两个地图，如果都比较大的话，内存告急！
# 2 流式关卡切换
一种将大世界地图分成小level的方式，然后通过关卡之间的加载卸载来模拟大世界的逻辑
## 2.1 添加子关卡的两种方式
### 1 手动添加
1 
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202405132128759.png)
再Levels编辑器中执行AddExisting节点就行，是在persistent level中添加子关卡。
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202405132132966.png)
添加完后就是这样
2 代码部分:
#### AddExistingLevel_Executed
点击了AddExisting就会执行AddExistingLevel_Executed方法
```cpp
ActionList.MapAction( Commands.World_AddExistingLevel,
		FExecuteAction::CreateSP( this, &FStreamingLevelCollectionModel::AddExistingLevel_Executed ) );
		
void FStreamingLevelCollectionModel::AddExistingLevel_Executed()
{
	AddExistingLevel();
}

void FStreamingLevelCollectionModel::AddExistingLevel(bool bRemoveInvalidSelectedLevelsAfter)
{
	if (!bAssetDialogOpen)
	{
		bAssetDialogOpen = true;
		// 选择SubLevel的代理
		FEditorFileUtils::FOnLevelsChosen LevelsChosenDelegate = FEditorFileUtils::FOnLevelsChosen::CreateSP(this, &FStreamingLevelCollectionModel::HandleAddExistingLevelSelected, bRemoveInvalidSelectedLevelsAfter);
		// 取消选择SubLevel的代理
		FEditorFileUtils::FOnLevelPickingCancelled LevelPickingCancelledDelegate = FEditorFileUtils::FOnLevelPickingCancelled::CreateSP(this, &FStreamingLevelCollectionModel::HandleAddExistingLevelCancelled);
		const bool bAllowMultipleSelection = true;
		// 弹出可选择SubLevel的界面，把两个代理传进去
		FEditorFileUtils::OpenLevelPickingDialog(LevelsChosenDelegate, LevelPickingCancelledDelegate, bAllowMultipleSelection);
	}
}
```
#### HandleAddExistingLevelSelected
下面我们看下HandleAddExistingLevelSelected这个方法，再选择了SubLevel之后会走到
```cpp
void FStreamingLevelCollectionModel::HandleAddExistingLevelSelected(const TArray<FAssetData>& SelectedAssets, bool bRemoveInvalidSelectedLevelsAfter)
{
	bAssetDialogOpen = false;

	// 遍历挑选的SubLevels，将每个PackageName存到数组中
	TArray<FString> PackageNames;
	for (const FAssetData& AssetData : SelectedAssets)
	{
		PackageNames.Add(AssetData.PackageName.ToString());
	}

	// Save or selected list, adding a new level will clean it up
	// 当前选择的SubLevels是否有效
	FLevelModelList SavedInvalidSelectedLevels = InvalidSelectedLevels;

	// 添加SubLevels的方法，注意AddedLevelStreamingClass这个uclass
	EditorLevelUtils::AddLevelsToWorld(CurrentWorld.Get(), MoveTemp(PackageNames), AddedLevelStreamingClass);

	// Force a cached level list rebuild
	// 初始化各种信息
	PopulateLevelsList();

	// 移除无效的SubLevel,没啥用感觉
	if (bRemoveInvalidSelectedLevelsAfter)
	{
		InvalidSelectedLevels = SavedInvalidSelectedLevels;
		RemoveInvalidSelectedLevels_Executed();
	}
}
```
代码不长，逻辑也不复杂，注意AddedLevelStreamingClass这个uclass，这个是我们可以选择的SubLevel的加载方式，如果是SetAddStreamingMethod_AlwaysLoaded这种情况就会创建出ULevelStreamingAlwaysLoaded这个类，如果是SetAddStreamingMethod_Blueprint这种情况就会创建出ULevelStreamingDynamic这个类。
#### AddLevelsToWorld
下面看下AddLevelsToWorld实现添加SubLevel的方法
```cpp
ULevel* UEditorLevelUtils::AddLevelsToWorld(UWorld* InWorld, TArray<FString> PackageNames, TSubclassOf<ULevelStreaming> LevelStreamingClass)
{
	// World没有，返回nullptr
	if (!ensure(InWorld))
	{
		return nullptr;
	}

	// 编辑器里面那个加载东西的进度条
	FScopedSlowTask SlowTask(PackageNames.Num(), LOCTEXT("AddLevelsToWorldTask", "Adding Levels to World"));
	SlowTask.MakeDialog();

	// Sort the level packages alphabetically by name.
	// 按照PackageName的字母顺序对SubLevels进行排序
	PackageNames.Sort();

	// Fire ULevel::LevelDirtiedEvent when falling out of scope.
	FScopedLevelDirtied LevelDirtyCallback;

	// Try to add the levels that were specified in the dialog.
	ULevel* NewLevel = nullptr;
	for (const FString& PackageName : PackageNames)
	{
		SlowTask.EnterProgressFrame();

		// [[World和Level#ULevelStreaming 创建]]
		if (ULevelStreaming* NewStreamingLevel = AddLevelToWorld_Internal(InWorld, *PackageName, LevelStreamingClass))
		{
			NewLevel = NewStreamingLevel->GetLoadedLevel();
			if (NewLevel)
			{
				LevelDirtyCallback.Request();
			}
		}
	} // for each file

	  // Set the last loaded level to be the current level
	  // 设置当前Level
	if (NewLevel)
	{
		if (InWorld->SetCurrentLevel(NewLevel))
		{
			FEditorDelegates::NewCurrentLevel.Broadcast();
		}
	}

	// For safety
	if (GLevelEditorModeTools().IsModeActive(FBuiltinEditorModes::EM_Landscape))
	{
		GLevelEditorModeTools().ActivateDefaultMode();
	}

	// Broadcast the levels have changed (new style)
	// 广播事件
	InWorld->BroadcastLevelsChanged();
	FEditorDelegates::RefreshLevelBrowser.Broadcast();

	// Update volume actor visibility for each viewport since we loaded a level which could potentially contain volumes
	// 看英文注释
	if (GUnrealEd)
	{
		GUnrealEd->UpdateVolumeActorVisibility(nullptr);
	}

	return NewLevel;
}
```
就是一个对加载出的ULevelStreaming进行初始化啥的，里面包含些广播事件，初始化啥的。
#### ULevelStreaming 创建
下面看下ULevelStreaming的创建流程
```cpp
ULevelStreaming* UEditorLevelUtils::AddLevelToWorld_Internal(UWorld* InWorld, const TCHAR* LevelPackageName, TSubclassOf<ULevelStreaming> LevelStreamingClass, const FTransform& LevelTransform)
{
	ULevel* NewLevel = nullptr;
	ULevelStreaming* StreamingLevel = nullptr;
	// 是否是主关卡？
	bool bIsPersistentLevel = (InWorld->PersistentLevel->GetOutermost()->GetName() == FString(LevelPackageName));

	// 如果是主关卡 或者 已经加载过了，就弹出提示，不看了这部分
	if (bIsPersistentLevel || FLevelUtils::FindStreamingLevel(InWorld, LevelPackageName))
	{
		// 不看了省略
	}
	else
	{
		// If the selected class is still NULL or the selected class is abstract, abort the operation.
		if (LevelStreamingClass == nullptr || LevelStreamingClass->HasAnyClassFlags(CLASS_Abstract))
		{
			return nullptr;
		}

		const FScopedBusyCursor BusyCursor;

		// 根据LevelStreamingClass创建出我们的ULevelStreaming
		StreamingLevel = NewObject<ULevelStreaming>(InWorld, LevelStreamingClass, NAME_None, RF_NoFlags, NULL);

		// Associate a package name.
		// 将创建出的StreamingLevel和World关联起来,参数是选择的SubLevel资源名称
		// 将创建出的StreamingLevel放入到选择的SubLevel资源中的World里的StreamingLevelsToConsider这个变量里
		StreamingLevel->SetWorldAssetByPackageName(LevelPackageName);
		// SteamingLevel的初始化
		StreamingLevel->LevelTransform = LevelTransform;
		StreamingLevel->LevelColor = FLinearColor::MakeRandomColor();

		// Add the new level to world.
		// 向当前World中添加StreamingLevel，就是给StreamingLevels赋值
		// 在编辑器下，TargetState是ETargetState::LoadedNotVisible，并给StreamingLevelsToConsider赋值
		// 在exe下，TargetState是ETargetState::Unloaded，没有给StreamingLevelsToConsider赋值
		InWorld->AddStreamingLevel(StreamingLevel);

		// Refresh just the newly created level.
		// 刷新下新创建出的streamingLevel？编辑器会走
		TArray<ULevelStreaming*> LevelsForRefresh;
		LevelsForRefresh.Add(StreamingLevel);
		InWorld->RefreshStreamingLevels(LevelsForRefresh);
		InWorld->MarkPackageDirty();

		// 获取SteamingLevel中包的ULevel
		NewLevel = StreamingLevel->GetLoadedLevel();
		if (NewLevel != nullptr)
		{
			EditorLevelUtils::SetLevelVisibility(NewLevel, true, true);

			// Levels migrated from other projects may fail to load their world settings
			// If so we create a new AWorldSettings actor here.
			// 为新关卡设置WorldSettings
			if (NewLevel->GetWorldSettings(false) == nullptr)
			{
				UWorld* SubLevelWorld = CastChecked<UWorld>(NewLevel->GetOuter());

				FActorSpawnParameters SpawnInfo;
				SpawnInfo.SpawnCollisionHandlingOverride = ESpawnActorCollisionHandlingMethod::AlwaysSpawn;
				SpawnInfo.Name = GEngine->WorldSettingsClass->GetFName();
				AWorldSettings* NewWorldSettings = SubLevelWorld->SpawnActor<AWorldSettings>(GEngine->WorldSettingsClass, SpawnInfo);

				NewLevel->SetWorldSettings(NewWorldSettings);
			}
		}
	}

	// 广播事件
	if (NewLevel) // if the level was successfully added
	{
		FEditorDelegates::OnAddLevelToWorld.Broadcast(NewLevel);
	}

	return StreamingLevel;
}
```
看上就是通过NewObject创建出ULevelStreaming这个对象，然后通过UWorld::AddStreamingLevel这个方法将其赋值给World的StreamingLevels和StreamingLevelsToConsider数组。再添加完成后StreamingLevel的CurrentState是UnLoaded，如果是编辑器TargetState是LoadedNotVisibe，如果是exe的话TargetState是Unloaded。
### 2 WorldComposition方式
首先需要在WorldSettings中开启：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202405222014460.png)
在开启之后引擎会遍历PersistentLevel所在文件夹下所有的关卡，生成WorldCompositionTile。
#### 1 FUnrealEdMisc::EnableWorldComposition、
```cpp
bool FUnrealEdMisc::EnableWorldComposition(UWorld* InWorld, bool bEnable)
{
	if (InWorld->WorldComposition == nullptr)
	{
		FString RootPackageName = InWorld->GetOutermost()->GetName();
		// Map should be saved to disk
		if (!FPackageName::DoesPackageExist(RootPackageName))
		{
			return false;
		}
		// All existing sub-levels on this map should be removed
		// 移除所有PersistentLevel的所有子关卡
		int32 NumExistingSublevels = InWorld->GetStreamingLevels().Num();
		if (NumExistingSublevels > 0)
		{
			return false;
		}
		AWorldSettings* WorldSettings = InWorld->GetWorldSettings();
		UClass* WorldCompositionClass = UWorldComposition::StaticClass();
		if (WorldSettings && WorldSettings->WorldCompositionClass.Get())
		{
			WorldCompositionClass = WorldSettings->WorldCompositionClass.Get();
		}
		// 创建出WorldComposition对象
		auto WorldCompostion = NewObject<UWorldComposition>(InWorld, WorldCompositionClass);
		// All map files found in the same and folder and all sub-folders will be added ass sub-levels to this map
		// Make sure user understands this
		// WorldComposition会自动扫描文件夹下的子关卡，然后创建对应的Tile
		int32 NumFoundSublevels = WorldCompostion->GetTilesList().Num();
		if (NumFoundSublevels)
		{
		}
		InWorld->WorldComposition = WorldCompostion;
		UWorldComposition::WorldCompositionChangedEvent.Broadcast(InWorld);
	}
	return true;
}
```
这一部分没啥东西，就是创建开启WorldComposition，判断persistentLevel是否有子关卡，弹出一些提示框之类的
#### 2 UWorldComposition::PostInitProperties
```cpp
void UWorldComposition::PostInitProperties()
{
	Super::PostInitProperties();

	if (!IsTemplate() && !GetOutermost()->HasAnyPackageFlags(PKG_PlayInEditor))
	{
		Rescan();
	}
}

void UWorldComposition::Rescan()
{
	// Save tiles state, so we can restore it for dirty tiles after rescan is done
	FTilesList SavedTileList = Tiles;
		
	Reset();	
	// Add found tiles to a world composition, except persistent level
	for (const FString& TilePackageName : Gatherer.TilesCollection)
	{
		// 这里边一系列的操作就是创建Tile，也就是FWorldCompositionTile对象
		// 然后添加到Tiles列表里面
		Tiles.Add(Tile);
	}
	
	// Create streaming levels for each Tile
	// 根据Tiles列表创建出ULevelStreaming，然后存到TilesStreaming数组里面
	PopulateStreamingLevels();

	// Calculate absolute positions since they are not serialized to disk
	// 计算每个Tile的Position,通过遍历Tile然后将期父亲的Position加上的方式确定
	CaclulateTilesAbsolutePositions();
}
```
扫描PersistentLevel文件夹下的所有关卡，填充TilesStreaming数组，确定每个Tile的AbsolutePosition。
#### 3 UWorldComposition::PostLoad
```cpp
void UWorldComposition::PostLoad()
{
	Super::PostLoad();

	UWorld* World = GetWorld();

	if (World->IsGameWorld())
	{
		// Remove streaming levels created by World Browser, to avoid duplication with streaming levels from world composition
		World->SetStreamingLevels(TilesStreaming);
	}
}
```
这个方法就是添加到StreamingLevelsToConsider这个数组里面
## 2.2 流式关卡加载流程
### 1 又是在Engine::Tick方法里面，我们就看GameEngine吧
```cpp
if( GIsServer == true )
{
	SCOPE_CYCLE_COUNTER(STAT_UpdateLevelStreaming);
	// 这个关键UpdateLevelStreaming方法
	Context.World()->UpdateLevelStreaming();
}
```
### 2 UWorld::UpdateLevelStreaming
```cpp
void UWorld::UpdateLevelStreaming()
{
	// 增加计数器，表示开始处理StreamingLevelsToConsider这个数组？
	StreamingLevelsToConsider.BeginConsideration();

	// 遍历StreamingLevelsToConsider数组，对其中每个元素进行UpdateStreamingState
	for (int32 Index = StreamingLevelsToConsider.GetStreamingLevels().Num() - 1; Index >= 0; --Index)
	{
		if (ULevelStreaming* StreamingLevel = StreamingLevelsToConsider.GetStreamingLevels()[Index])
		{
			bool bUpdateAgain = true;
			bool bShouldContinueToConsider = true;
			while (bUpdateAgain && bShouldContinueToConsider)
			{
				bool bRedetermineTarget = false;
				FStreamingLevelPrivateAccessor::UpdateStreamingState(StreamingLevel, bUpdateAgain, bRedetermineTarget);

				// 注意这个地方，每次UpdateStreamingState之后，会根据判断执行一次DetermineTargetState
				if (bRedetermineTarget)
				{
					bShouldContinueToConsider = FStreamingLevelPrivateAccessor::DetermineTargetState(StreamingLevel);
				}
			}

			if (!bShouldContinueToConsider)
			{
				StreamingLevelsToConsider.RemoveAt(Index);
			}
		}
		else
		{
			StreamingLevelsToConsider.RemoveAt(Index);
		}
	}

	// 减少计数器，表示结束处理StreamingLevelsToConsider这个数组？
	StreamingLevelsToConsider.EndConsideration();


	// In case more levels has been requested to unload, force GC on next tick 
	if (GLevelStreamingForceGCAfterLevelStreamedOut != 0)
	{
		if (NumLevelsPendingPurge < FLevelStreamingGCHelper::GetNumLevelsPendingPurge())
		{
			GEngine->ForceGarbageCollection(true); 
		}
	}
}
```
主要就是对World中StreamingLevelsToConsider这个数组每个元素的处理。所以我们想要加载关卡首先就需要存到StreamingLevelsToConsider这个数组里面，然后world在tick的时候会进行加载。
### 3 ULevelStreaming::UpdateStreamingState
这个方法分成了两部分，一部分是匿名方法异步加载关卡，一部分是对StreamingLevel的state进行处理。这里我们就看下第一部分
```cpp
void ULevelStreaming::UpdateStreamingState(bool& bOutUpdateAgain, bool& bOutRedetermineTarget)
{
	auto UpdateStreamingState_RequestLevel = [&]()
	{
		// 很长的方法，主要就这个，其他的都先省略掉
		RequestLevel(World, bAllowLevelLoadRequests, (bBlockOnLoad ? ULevelStreaming::AlwaysBlock : ULevelStreaming::BlockAlwaysLoadedLevelsOnly));
		
	};
}

bool ULevelStreaming::RequestLevel(UWorld* PersistentWorld, bool bAllowLevelLoadRequests, EReqLevelBlock BlockPolicy)
{
	// Quit early in case load request already issued
	// 当前关卡正在加载，直接返回
	if (CurrentState == ECurrentState::Loading)
	{
		return true;
	}

	// Previous attempts have failed, no reason to try again
	// 当前关卡加载失败了，不在进行重新加载
	if (CurrentState == ECurrentState::FailedToLoad)
	{
		return false;
	}

	// Package name we want to load
	const bool bIsGameWorld = PersistentWorld->IsGameWorld();
	const FName DesiredPackageName = bIsGameWorld ? GetLODPackageName() : GetWorldAssetPackageFName();
	const FName LoadedLevelPackageName = GetLoadedLevelPackageName();

	// Check if currently loaded level is what we want right now
	// 当前关卡已经加载了，并且DesiredLevel也是当前关卡，直接返回
	if (LoadedLevel && LoadedLevelPackageName == DesiredPackageName)
	{
		return true;
	}

	// Can not load new level now, there is still level pending unload
	// 已经标记卸载当前关卡了，直接返回
	if (PendingUnloadLevel)
	{
		return false;
	}

	// Can not load new level now either, we're still processing visibility for this one
	// 现在正在处理当前关卡的可见性，也就是再可见和不可见的状态切换中，直接返回
	ULevel* PendingLevelVisOrInvis = (PersistentWorld->GetCurrentLevelPendingVisibility() ? PersistentWorld->GetCurrentLevelPendingVisibility() : PersistentWorld->GetCurrentLevelPendingInvisibility());
    if (PendingLevelVisOrInvis && PendingLevelVisOrInvis == LoadedLevel)
    {
		
		return false;
	}

	// Validate that our new streaming level is unique, check for clash with currently loaded streaming levels
	// 这个循环是检测要创建的当前关卡是不是独一无二的，检测关卡中的World是不是被其他卸载的关卡引用
	// 如果是的话，就直接返回
	for (ULevelStreaming* OtherLevel : PersistentWorld->GetStreamingLevels())
	{
		if (OtherLevel == nullptr || OtherLevel == this)
		{
			continue;
		}

		const ECurrentState OtherState = OtherLevel->GetCurrentState();
		if (OtherState == ECurrentState::FailedToLoad || OtherState == ECurrentState::Removed || (OtherState == ECurrentState::Unloaded && (OtherLevel->TargetState == ETargetState::Unloaded || OtherLevel->TargetState == ETargetState::UnloadedAndRemoved)))
		{
			// If the other level isn't loaded or in the process of being loaded we don't need to consider it
			continue;
		}

		if (OtherLevel->WorldAsset == WorldAsset)
		{ 
			if (OtherLevel->GetIsRequestingUnloadAndRemoval())
			{
				return false; // Cannot load new level now, retry until the OtherLevel is done unloading
			}
			else
			{
				
				CurrentState = ECurrentState::FailedToLoad;
				return false;
			}
		}
	}
	
	EPackageFlags PackageFlags = PKG_ContainsMap;
	int32 PIEInstanceID = INDEX_NONE;

	// Try to find the [to be] loaded package.
	// 找一下当前关卡的UPackage
	UPackage* LevelPackage = (UPackage*)StaticFindObjectFast(UPackage::StaticClass(), nullptr, DesiredPackageName, 0, 0, RF_NoFlags, EInternalObjectFlags::PendingKill);

	// copy streaming level on demand if we are in PIE
	// (the world is already loaded for the editor, just find it and copy it)
	if ( LevelPackage == nullptr && PersistentWorld->IsPlayInEditor() )
	{
		// 当前关卡的UPackage没加载，并且是编辑器模式下有段逻辑
		// 不看了，直接省略掉
	}

	// Package is already or still loaded.
	if (LevelPackage)
	{
		// 这边是当前关卡的UPackage已经加载好了，不需要再启动线程去加载了，直接走加载完成后的逻辑
		// 代码省略掉
		return true;
	}

	// Async load package if world object couldn't be found and we are allowed to request a load.
	if (bAllowLevelLoadRequests)
	{
		const FName DesiredPackageNameToLoad = bIsGameWorld ? GetLODPackageNameToLoad() : PackageNameToLoad;
		const FString PackageNameToLoadFrom = DesiredPackageNameToLoad != NAME_None ? DesiredPackageNameToLoad.ToString() : DesiredPackageName.ToString();

		// 当前关卡UPackage存在
		if (FPackageName::DoesPackageExist(PackageNameToLoadFrom))
		{
			// 切换当前关卡的State为Loading
			CurrentState = ECurrentState::Loading;
			// 增加处在Loading中的关卡计数器
			FWorldNotifyStreamingLevelLoading::Started(PersistentWorld);
			// 一个静态数组，保存下当前关卡UPackage和World
			ULevel::StreamedLevelsOwningWorld.Add(DesiredPackageName, PersistentWorld);
			// 保存数组
			UWorld::WorldTypePreLoadMap.FindOrAdd(DesiredPackageName) = PersistentWorld->WorldType;

			// 开启线程，加载当前关卡资源
			// 加载完成后调用ULevelStreaming::AsyncLevelLoadComplete方法
			LoadPackageAsync(DesiredPackageName.ToString(), nullptr, *PackageNameToLoadFrom, FLoadPackageAsyncDelegate::CreateUObject(this, &ULevelStreaming::AsyncLevelLoadComplete), PackageFlags, PIEInstanceID, GetPriority());

			// streamingServer: server loads everything?
			// Editor immediately blocks on load and we also block if background level streaming is disabled.
			if (BlockPolicy == AlwaysBlock || (ShouldBeAlwaysLoaded() && BlockPolicy != NeverBlock))
			{
				if (IsAsyncLoading())
				{
				}

				// Finish all async loading.
				FlushAsyncLoading();
			}
		}
		else
		{
			CurrentState = ECurrentState::FailedToLoad;
			return false;
		}
	}

	return true;
}
```
异步加载关卡之前的一些条件处理，判断是否能够加载当前关卡。在LoadPackageAsync这个方法之前会设置关卡的CurrentState为Loading状态，此时当前关卡的CurrenState是Loading，TargetState是LoadedNotVisibe。
### 4 ULevelStreaming::AsyncLevelLoadComplete

```cpp
void ULevelStreaming::AsyncLevelLoadComplete(const FName& InPackageName, UPackage* InLoadedPackage, EAsyncLoadingResult::Type Result)
{
	// 更新新关卡的CurrentState为LoadedNotVisible
	CurrentState = ECurrentState::LoadedNotVisible;
	if (UWorld* World = GetWorld())
	{
		if (World->GetStreamingLevels().Contains(this))
		{
			// 减少处在Loading中的关卡计数器
			FWorldNotifyStreamingLevelLoading::Finished(World);
		}
	}

	if (InLoadedPackage)
	{
		UPackage* LevelPackage = InLoadedPackage;
		
		// Try to find a UWorld object in the level package.
		// 从Package里面找到world
		UWorld* World = UWorld::FindWorldInPackage(LevelPackage);

		if (World)
		{
			// 拿到world中的主Level
			ULevel* Level = World->PersistentLevel;
			if (Level)
			{
				UWorld* LevelOwningWorld = Level->OwningWorld;
				if (LevelOwningWorld)
				{
					ULevel* PendingLevelVisOrInvis = (LevelOwningWorld->GetCurrentLevelPendingVisibility() ? LevelOwningWorld->GetCurrentLevelPendingVisibility() : LevelOwningWorld->GetCurrentLevelPendingInvisibility());
					if (PendingLevelVisOrInvis && PendingLevelVisOrInvis == LoadedLevel)
					{
					}
					else
					{
						// 设置新关卡，把旧关卡卸载掉
						SetLoadedLevel(Level);
						// Broadcast level loaded event to blueprints
						// 广播事件
						OnLevelLoaded.Broadcast();
						// AZURE
						if (OnAzureLevelLoaded.IsBound())
							OnAzureLevelLoaded.Execute();
					}
				}
				// 处理关卡需要的初始化数据
				Level->HandleLegacyMapBuildData();
				IStreamingManager::Get().AddLevel(Level);
				if (LODPackageNames.Num() > 0)
				{
					Level->bRequireFullVisibilityToRender = true;
					// LOD levels should not be visible on server
					Level->bClientOnlyVisible = LODPackageNames.Contains(InLoadedPackage->GetFName());
				}
			
				if (ensure(LevelOwningWorld) && LevelOwningWorld->WorldType == EWorldType::Editor)
				{
					LevelOwningWorld->AddLevel(Level);
				}
			}
			else
			{
			}
		}
		else
		{
			// 这便是package里面没有world的情况，看上去就是资源有问题，只看正常情况，省略
		}
	}
	else if (Result == EAsyncLoadingResult::Canceled)
	{
		// Cancel level streaming
		CurrentState = ECurrentState::Unloaded;
		SetShouldBeLoaded(false);
	}
	else
	{
		CurrentState = ECurrentState::FailedToLoad;
 		SetShouldBeLoaded(false);
	}

	// Clean up the world type list and owning world list now that PostLoad has occurred
	UWorld::WorldTypePreLoadMap.Remove(InPackageName);
	ULevel::StreamedLevelsOwningWorld.Remove(InPackageName);
}
```
用新的线程加载完关卡后的回调函数，保存各种数据吧。这个回调函数结束后CurrentState是LoadedNotVIsible，TargetState是LoadedNotVisibe。
## 2.3 流式关卡卸载流程
### 1 基本流程
如果玩家不在volume中或者是某些情况，首先会UpdateStreamingState改变CurrentState。
```cpp
// 第一次tick走到UpdateStreamingState
// CurrentState::MakingInvisible，ETargetState::LoadedNotVisible
case ECurrentState::LoadedVisible:
	switch (TargetState)
	{
	case ETargetState::LoadedNotVisible:
		CurrentState = ECurrentState::MakingInvisible;
		bOutUpdateAgain = true;
		break;
	break;

// 第二次tick走到UpdateStreamingState
// CurrentState::LoadedNotVisible，ETargetState::Unloaded
case ECurrentState::MakingInvisible:
	if (ensure(LoadedLevel))
	{
		// Hide loaded level, incrementally if necessary
		World->RemoveFromWorld(LoadedLevel, !bShouldBlockOnUnload && World->IsGameWorld());

		// Inform the scene once we have finished making the level invisible
		if (!LoadedLevel->bIsVisible)
		{
			if (World->Scene)
			{
				World->Scene->OnLevelRemovedFromWorld(World, LoadedLevel->bIsLightingScenario);
			}

			CurrentState = ECurrentState::LoadedNotVisible;
			bOutUpdateAgain = true;
			bOutRedetermineTarget = true;
		}
	}
	break;

// 第三次tick走到UpdateStreamingState
case ECurrentState::LoadedNotVisible:
	switch (TargetState)
	{
	case ETargetState::Unloaded:
		DiscardPendingUnloadLevel(World);
		ClearLoadedLevel();
		DiscardPendingUnloadLevel(World);

		bOutUpdateAgain = true;
		bOutRedetermineTarget = true;
		break;

	break;
```
### 2 RemoveFromWorld
```cpp
// 1 停掉Level里Actor
for (int32 ActorIdx = 0; ActorIdx < Level->Actors.Num(); ActorIdx++)
{
	if (AActor* Actor = Level->Actors[ActorIdx])
	{
		Actor->RouteEndPlay(EEndPlayReason::RemovedFromWorld);
	}
}
// 2 移除Level里的Pawn
// Remove any pawns from the pawn list that are about to be streamed out
for (APawn* Pawn : TActorRange<APawn>(this))
{
	if (Pawn->IsInLevel(Level))
	{
		AController* Controller = Pawn->GetController();
		// This should have happened as part of the RouteEndPlay above, but ensuring to validate this assumption and maintain behavior
		// with RemovePawn having been deprecated
		if (!ensure(Controller == nullptr || (Controller->GetPawn() == Pawn)))
		{
			Controller->UnPossess();
		}
	}
	else if (UCharacterMovementComponent* CharacterMovement = Cast<UCharacterMovementComponent>(Pawn->GetMovementComponent()))
	{
		// otherwise force floor check in case the floor was streamed out from under it
		CharacterMovement->bForceNextFloorCheck = true;
	}
}
// 3 移除渲染资源？
Level->ReleaseRenderingResources();
// 4 StreamingManager里面移除Level，这个StreamingManager是干嘛的呢？
// Remove from the world's level array and destroy actor components.
IStreamingManager::Get().RemoveLevel( Level );
// 5 Unregister Level中的Component,也就是Level自带的ModelComponent和其中Actor的Component
Level->ClearLevelComponents();
// 6 通知服务器改变Level状态，通过PlayerController
// 7 设置bIsVisible为false
Level->bIsVisible = false;
```
从world中移除掉LoadedLevel中的东西，改变LoadedLevel的bIsVisible状态
### 3 ClearLoadedLevel
```cpp
void ClearLoadedLevel() { SetLoadedLevel(nullptr); }
void ULevelStreaming::SetLoadedLevel(ULevel* Level)
{ 
	// 设置要UnloadLevel和LoadedLevel
	PendingUnloadLevel = LoadedLevel;
	LoadedLevel = Level;
	bHasCachedLoadedLevelPackageName = false;

	// Cancel unloading for this level, in case it was queued for it
	FLevelStreamingGCHelper::CancelUnloadRequest(LoadedLevel);

	// Add this level to the correct collection
	const ELevelCollectionType CollectionType =	bIsStatic ? ELevelCollectionType::StaticLevels : ELevelCollectionType::DynamicSourceLevels;

	UWorld* World = GetWorld();

	FLevelCollection& LC = World->FindOrAddCollectionByType(CollectionType);
	LC.RemoveLevel(PendingUnloadLevel);

	if (LoadedLevel)
	{
		// 这里不走这个，省略得了
	}
	else
	{
		CurrentState = ECurrentState::Unloaded;
	}

	World->UpdateStreamingLevelShouldBeConsidered(this);
}
```
最主要的就是设置LoadedLevel为nullptr，剩下的也就是字面意思了不复杂。
### 4 DiscardPendingUnloadLevel
```cpp
void ULevelStreaming::DiscardPendingUnloadLevel(UWorld* PersistentWorld)
{
	if (PendingUnloadLevel)
	{
		// 如果可以移除Level但是可见的，说明还没从world中移除，手动移除下
		if (PendingUnloadLevel->bIsVisible)
		{
			PersistentWorld->RemoveFromWorld(PendingUnloadLevel);
		}
		// 正常流程，可以移除Level是不可见的
		if (!PendingUnloadLevel->bIsVisible)
		{
			// 调用这个Helper把Level存到LevelsPendingUnload这个数组里
			FLevelStreamingGCHelper::RequestUnload(PendingUnloadLevel);
			PendingUnloadLevel = nullptr;
		}
	}
}
void FLevelStreamingGCHelper::RequestUnload( ULevel* InLevel )
{
	if (!IsRunningCommandlet())
	{
		check( InLevel );
		check( InLevel->bIsVisible == false );
		LevelsPendingUnload.AddUnique( InLevel );
	}
}
```
主要目的就是将可移除Level添加到LevelsPendingUnload这个数组里面，然后再PrepareStreamedOutLevelsForGC这个方法中将关卡卸载。
### 2 FLevelStreamingGCHelper::PrepareStreamedOutLevelsForGC
在这个方法里添加监听代理
```cpp
void FLevelStreamingGCHelper::AddGarbageCollectorCallback()
{
	// Only register for garbage collection once
	static bool GarbageCollectAdded = false;
	if ( GarbageCollectAdded == false )
	{
		FCoreUObjectDelegates::GetPreGarbageCollectDelegate().AddStatic( FLevelStreamingGCHelper::PrepareStreamedOutLevelsForGC );
		FCoreUObjectDelegates::GetPostGarbageCollect().AddStatic( FLevelStreamingGCHelper::VerifyLevelsGotRemovedByGC );
		GarbageCollectAdded = true;
	}
}
```
在这个方法里广播的代理
```cpp
void CollectGarbageInternal(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	FCoreUObjectDelegates::GetPreGarbageCollectDelegate().Broadcast();
}
```
在这个方法里卸载的Level
```cpp
void FLevelStreamingGCHelper::PrepareStreamedOutLevelsForGC()
{
	Level->CleanupLevel();
}
```

# 3 流式关卡触发流程
## 1 LevelStreamingVolume
这个就是在PersistentLevel里面放置一个ALevelStreamingVolume这个Actor，类似于一个包围盒的东西，判断玩家是否在这个Volume里面，进而触发加载StreamingLevel。
```cpp
void UWorld::Tick( ELevelTick TickType, float DeltaSeconds )
{
	ProcessLevelStreamingVolumes();
}
```
看样子也是在TIck里触发的
```cpp
void UWorld::ProcessLevelStreamingVolumes(FVector* OverrideViewLocation)
{
	// 在此方法里赋值，会保存由Volume触发的StreamingLevel
	TArray<ULevelStreaming*> LevelStreamingObjectsWithVolumes;
	// Volume的类型不是SVB_BlockingOnLoad，就会存储。
	// 是LevelStreamingObjectsWithVolumes子集
	TSet<ULevelStreaming*> LevelStreamingObjectsWithVolumesOtherThanBlockingLoad;
	// 记录需要看到的StreamingLevel
	TMap<ULevelStreaming*,FVisibleLevelStreamingSettings> VisibleLevelStreamingObjects;
	// key是world中所有的AVolume,value是标志位，标识viewpoint是否在volume中
	TMap<AVolume*,bool> VolumeMap;
	// 遍历world中所有的playerController
	{
		// 记录viewpoint
		FVector ViewLocation(0,0,0);
		// 遍历LevelStreamingObjectsWithVolumes数组
		{
			// 遍历StreamingLevel中的所有Volumes
			{
				// 计算viewpoint是否在Volume中，并填充VolumeMap
				bViewpointInVolume=
						StreamingVolume->EncompassesPoint( ViewLocation );		
				VolumeMap.Add( StreamingVolume, bViewpointInVolume );
				// 如果viewpoint在volume中
				if ( bViewpointInVolume )
				{
					// setting
					StreamingSettings |= 
							FVisibleLevelStreamingSettings( (EStreamingVolumeUsage) StreamingVolume->StreamingUsage );
					// 填充VisibleLevelStreamingObjects
					VisibleLevelStreamingObjects.
							Add( LevelStreamingObject, StreamingSettings );
					// 如果条件满足，退出所有volumes的遍历
					if ( StreamingSettings.AllSettingsEnabled() )
					{
						break;
					}
				}
			}
		}
	}
	// 后面继续遍历LevelStreamingObjectsWithVolumes数组
	// 如果是应该加载，通过
	if( bShouldBeLoaded )
	{
		/// 设置bShouldBeLoaded变量，应该加载StreamingLevel了
	LevelStreamingObject->SetShouldBeLoaded(true);
	// SetShouldBeVisible(true)，
	// 这个方法里也会走DetermineTargetState方法
	// 将TargetState设置为ETargetState::LoadedNotVisible,然后添加进StreamingLevelsToConsider数组里
	LevelStreamingObject->SetShouldBeVisible(NewStreamingSettings->ShouldBeVisible(bOriginalShouldBeVisible));
	// 设置状态
	LevelStreamingObject->bShouldBlockOnLoad	= NewStreamingSettings->ShouldBlockOnLoad();
	}
	// 这个是不应该加载，但已经加载过了，也就是需要卸载
	else if (LevelStreamingObject->ShouldBeLoaded())
	{
		if (TimeSeconds < LevelStreamingObject->LastVolumeUnloadRequestTime ||
			TimeSeconds - LevelStreamingObject->LastVolumeUnloadRequestTime > LevelStreamingObject->MinTimeBetweenVolumeUnloadRequests ||  LevelStreamingObject->LastVolumeUnloadRequestTime < 0.1f)
		{
			if (GetPlayerControllerIterator())
			{
				LevelStreamingObject->LastVolumeUnloadRequestTime = TimeSeconds;
				LevelStreamingObject->SetShouldBeLoaded(false);
				LevelStreamingObject->SetShouldBeVisible(false);
			}
		}
	}
	
}
```
1 遍历world中所有的StreamingLevels，来填充LevelStreamingObjectsWithVolumes和LevelStreamingObjectsWithVolumesOtherThanBlockingLoad两个数组。
2 遍历world中所有的playerController，算下玩家位置或者pov位置，判断位置在哪个StreamingLevel中的Volume中，填充VisibleLevelStreamingObjects数组。
3 遍历带有Volumes的StreamingLevels，如果在Volume里面就改变StreamingLevel的Target状态为LoadedNotVisible，也给之后UpdateStreamingState这个方法真正加载Level提供条件
## 2 WorldComposition
```cpp
// 在UWorld::Tick方法里面走
if( !bIsPaused )
{
	// Issues level streaming load/unload requests based on local players being inside/outside level streaming volumes.
	if (IsGameWorld())
	{
		// 这个上面看了，是Volume的推流机制
		ProcessLevelStreamingVolumes();
		// 这个就是需要看的WorldComposition推流机制
		if (WorldComposition)
		{
			WorldComposition->UpdateStreamingState();
		}
	}
}
```
也是在world的Tick里面，首先会检测Volume推流，然后就会检测WorldComposition推流
### 1 UWorldComposition::UpdateStreamingState
```cpp
void UWorldComposition::UpdateStreamingState(const FVector* InLocations, int32 Num)
{
	UWorld* OwningWorld = GetWorld();

	// Get the list of visible and hidden levels from current view point
	TArray<FDistanceVisibleLevel> DistanceVisibleLevels;
	TArray<FDistanceVisibleLevel> DistanceHiddenLevels;
	// 判断玩家位置和Level的距离
	GetDistanceVisibleLevels(InLocations, Num, DistanceVisibleLevels, DistanceHiddenLevels);
	
	// Dedicated server always blocks on load
	bool bShouldBlock = (OwningWorld->GetNetMode() == NM_DedicatedServer);
	
	// Set distance hidden levels to unload
	for (const FDistanceVisibleLevel& Level : DistanceHiddenLevels)
	{
		// 距离过远隐藏关卡
		CommitTileStreamingState(OwningWorld, Level.TileIdx, false, false, bShouldBlock, Level.LODIndex);
	}

	// Set distance visible levels to load
	for (const FDistanceVisibleLevel& Level : DistanceVisibleLevels)
	{
		// 距离合适显示关卡
		CommitTileStreamingState(OwningWorld, Level.TileIdx, true, true, bShouldBlock, Level.LODIndex);
	}
}
```
首先判断玩家和Level之间的距离赋值DistanceHiddenLevels和DistanceVisibleLevels数组，然后根据数组用CommitTileStreamingState来隐藏显示关卡。
### 2 UWorldComposition::GetDistanceVisibleLevels
```cpp
void UWorldComposition::GetDistanceVisibleLevels(
	const FVector* InLocations,
	int32 NumLocations,
	TArray<FDistanceVisibleLevel>& OutVisibleLevels,
	TArray<FDistanceVisibleLevel>& OutHiddenLevels) const
{
	const UWorld* OwningWorld = GetWorld();

	FIntPoint WorldOriginLocationXY = FIntPoint(OwningWorld->OriginLocation.X, OwningWorld->OriginLocation.Y);
			
	for (int32 TileIdx = 0; TileIdx < Tiles.Num(); TileIdx++)
	{
		if (OwningWorld->IsNetMode(NM_DedicatedServer))
		{
		}
		else if (IsRunningCommandlet())
		{
		}
		else
		{
			for (int32 LODIdx = INDEX_NONE; LODIdx < NumAvailableLOD; ++LODIdx)
			{
				for (int32 LocationIdx = 0; LocationIdx < NumLocations; ++LocationIdx)
				{
					FSphere QuerySphere(InLocations[LocationIdx], TileStreamingDistance);
					if (FMath::SphereAABBIntersection(QuerySphere, LevelBounds))
					{
						VisibleLevel.LODIndex = LODIdx;
						bIsVisible = true;
						break;
					}
				}
			}
		}

		if (bIsVisible)
		{
			OutVisibleLevels.Add(VisibleLevel);
		}
		else
		{
			OutHiddenLevels.Add(VisibleLevel);
		}
	}
}
```
大部分代码都省略掉了，主要就是遍历Tile，然后以玩家位置为中心Tile的可见距离为半径构建圆，将圆和Tile中关卡的AABB包围盒判断相交，如果相交放入VisibleLevel不相交放入HiddenLevel。
### 3 UWorldComposition::CommitTileStreamingState
```cpp
bool UWorldComposition::CommitTileStreamingState(UWorld* PersistentWorld, int32 TileIdx, bool bShouldBeLoaded, bool bShouldBeVisible, bool bShouldBlock, int32 LODIdx)
{
	// 一些条件判断，提前返回的，就省略掉了
	// Quit early in case we have cooldown on streaming state changes
	// 当前在切换阈值中，不改变Level状态
	const bool bUseStreamingStateCooldown = (PersistentWorld->IsGameWorld() && PersistentWorld->FlushLevelStreamingType == EFlushLevelStreamingType::None);
	if (bUseStreamingStateCooldown && TilesStreamingTimeThreshold > 0.0)
	{
		const double CurrentTime = FPlatformTime::Seconds();
		const double TimePassed = CurrentTime - Tile.StreamingLevelStateChangeTime;
		if (TimePassed < TilesStreamingTimeThreshold)
		{
			return false;
		}

		// Save current time as state change time for this tile
		Tile.StreamingLevelStateChangeTime = CurrentTime;
	}

	// Commit new state
	// 改变关卡状态,等待下一帧的StreamingLevel状态机切换
	StreamingLevel->bShouldBlockOnLoad	= bShouldBlock;
	StreamingLevel->SetShouldBeLoaded(bShouldBeLoaded);
	StreamingLevel->SetShouldBeVisible(bShouldBeVisible);
	StreamingLevel->SetLevelLODIndex(LODIdx);
	return true;
}
```
不复杂，字面意思，先有一些条件判断，然后设置StreamingLevel的状态，等待下一帧状态机的切换进而触发可见和不可见的过程。
## 3 手动触发
### 1 蓝图中Load Stream Level
```cpp
void UGameplayStatics::LoadStreamLevel(const UObject* WorldContextObject, FName LevelName, bool bMakeVisibleAfterLoad, bool bShouldBlockOnLoad, FLatentActionInfo LatentInfo)
{
	if (UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull))
	{
		FLatentActionManager& LatentManager = World->GetLatentActionManager();
		if (LatentManager.FindExistingAction<FStreamLevelAction>(LatentInfo.CallbackTarget, LatentInfo.UUID) == nullptr)
		{
			FStreamLevelAction* NewAction = new FStreamLevelAction(true, LevelName, bMakeVisibleAfterLoad, bShouldBlockOnLoad, LatentInfo, World);
			LatentManager.AddNewAction(LatentInfo.CallbackTarget, LatentInfo.UUID, NewAction);
		}
	}
}
```

