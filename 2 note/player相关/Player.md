https://www.cnblogs.com/u-n-owen/p/16443936.html

# 1 UGameEngine::Init
## 1 开始
在GameEngine::init方法中会创建localPlayer
```cpp
void UGameEngine::Init(IEngineLoop* InEngineLoop)
{
	// Call base.
	UEngine::Init(InEngineLoop);

	// Create game instance.  For GameEngine, this should be the only GameInstance that ever gets created.
	// 创建出所需的GameInstance
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(InitGameInstance);
		FSoftClassPath GameInstanceClassName = GetDefault<UGameMapsSettings>()->GameInstanceClass;
		UClass* GameInstanceClass = (GameInstanceClassName.IsValid() ? LoadObject<UClass>(NULL, *GameInstanceClassName.ToString()) : UGameInstance::StaticClass());
		
		if (GameInstanceClass == nullptr)
		{
			UE_LOG(LogEngine, Error, TEXT("Unable to load GameInstance Class '%s'. Falling back to generic UGameInstance."), *GameInstanceClassName.ToString());
			GameInstanceClass = UGameInstance::StaticClass();
		}

		GameInstance = NewObject<UGameInstance>(this, GameInstanceClass);

		GameInstance->InitializeStandalone();
	}
 
//  	// Creates the initial world context. For GameEngine, this should be the only WorldContext that ever gets created.
//  	FWorldContext& InitialWorldContext = CreateNewWorldContext(EWorldType::Game);

	IMovieSceneCaptureInterface* MovieSceneCaptureImpl = nullptr;
#if WITH_EDITOR
	if (!IsRunningDedicatedServer() && !IsRunningCommandlet())
	{
		MovieSceneCaptureImpl = IMovieSceneCaptureModule::Get().InitializeFromCommandLine();
		if (MovieSceneCaptureImpl)
		{
			StartupMovieCaptureHandle = MovieSceneCaptureImpl->GetHandle();
		}
	}
#endif

	// Initialize the viewport client.
	UGameViewportClient* ViewportClient = NULL;
	if(GIsClient)
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(InitGameViewPortClient);
		ViewportClient = NewObject<UGameViewportClient>(this, GameViewportClientClass);
		ViewportClient->Init(*GameInstance->GetWorldContext(), GameInstance);
		GameViewport = ViewportClient;
		GameInstance->GetWorldContext()->GameViewport = ViewportClient;
	}

	LastTimeLogsFlushed = FPlatformTime::Seconds();

	// Attach the viewport client to a new viewport.
	if(ViewportClient)
	{
		TRACE_CPUPROFILER_EVENT_SCOPE(AttachGameViewport);
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

		TRACE_CPUPROFILER_EVENT_SCOPE(BroadcastOnViewPortCreated);
		UGameViewportClient::OnViewportCreated().Broadcast();
	}

	UE_LOG(LogInit, Display, TEXT("Game Engine Initialized.") );

	// for IsInitialized()
	bIsInitialized = true;
}
```
