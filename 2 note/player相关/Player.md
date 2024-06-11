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
		// 创建出localPlayer
		if(ViewportClient->SetupInitialLocalPlayer(Error) == NULL)
		{
			UE_LOG(LogEngine, Fatal,TEXT("%s"),*Error);
		}
	}
	bIsInitialized = true;
}
```
字面意思不是很复杂，在这个方法里面，首先需要创建出设置好的GameInstance进而创建一系列的WorldContext，也就是说GameInstance是客户端服务器都存在的。然后如果是客户端的话，就创建出ViewportClient，进而创建出LocalPlayer。
## 2 SetupInitialLocalPlayer
```cpp
ULocalPlayer* UGameViewportClient::SetupInitialLocalPlayer(FString& OutError)
{
	// 拿到WorldContext里面存储的GameInstance，也就是上面创建出的那个
	UGameInstance * ViewportGameInstance = GEngine->GetWorldContextFromGameViewportChecked(this).OwningGameInstance;
	// Create the initial player - this is necessary or we can't render anything in-game.
	// 创建LocalPlayer
	return ViewportGameInstance->CreateInitialPlayer(OutError);
}
```