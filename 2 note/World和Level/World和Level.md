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

	{
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
