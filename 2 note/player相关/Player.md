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
		FSoftClassPath GameInstanceClassName = GetDefault<UGameMapsSettings>()->GameInstanceClass;
		UClass* GameInstanceClass = (GameInstanceClassName.IsValid() ? LoadObject<UClass>(NULL, *GameInstanceClassName.ToString()) : UGameInstance::StaticClass());
		if (GameInstanceClass == nullptr)
		{
			GameInstanceClass = UGameInstance::StaticClass();
		}
		GameInstance = NewObject<UGameInstance>(this, GameInstanceClass);
		GameInstance->InitializeStandalone();
	}

	// Initialize the viewport client.
	// 核心，如果在客户端模式下，就创建出ViewportClient
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
	// 如果ViewportClient创建出来了，自然也就是在客户端情况下
	if(ViewportClient)
	{
		// This must be created before any gameplay code adds widgets
		bool bWindowAlreadyExists = GameViewportWindow.IsValid();
		CreateGameViewport( ViewportClient );
		FString Error;
		// chuang'jian
		if(ViewportClient->SetupInitialLocalPlayer(Error) == NULL)
		{
			UE_LOG(LogEngine, Fatal,TEXT("%s"),*Error);
		}
	}

	UE_LOG(LogInit, Display, TEXT("Game Engine Initialized.") );

	// for IsInitialized()
	bIsInitialized = true;
}
```
