基本知识 https://www.cnblogs.com/u-n-owen/p/16443936.html
我们首先看下什么时候会创建LocalPlayer
# 1 LocalPlayer
## 1 UGameEngine::Init
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

ULocalPlayer* UGameInstance::CreateInitialPlayer(FString& OutError)
{
	return CreateLocalPlayer(IPlatformInputDeviceMapper::Get().GetPrimaryPlatformUser(), OutError, false);
}

ULocalPlayer* UGameInstance::CreateLocalPlayer(FPlatformUserId UserId, FString& OutError, bool bSpawnPlayerController)
{
	check(GetEngine()->LocalPlayerClass != NULL);

	ULocalPlayer* NewPlayer = NULL;
	int32 InsertIndex = INDEX_NONE;
	UGameViewportClient* GameViewport = GetGameViewportClient();
	if (LocalPlayers.Num() < MaxSplitscreenPlayers)
	{
		// 创建出我们的LocalPlayer
		NewPlayer = NewObject<ULocalPlayer>(GetEngine(), GetEngine()->LocalPlayerClass);
		// 将创建出的LocalPlayer添加到GameInstance中的LocalPlayers数组中
		InsertIndex = AddLocalPlayer(NewPlayer, UserId);
		UWorld* CurrentWorld = GetWorld();
		// 下面是创建对应的PlayerController，如果参数是需要创建的话
		// 我们走的这里是不用创建的，但也还是看下吧
		if (bSpawnPlayerController && InsertIndex != INDEX_NONE && CurrentWorld != nullptr)
		{
			// 如果不是客户端，就创建PlayerController
			if (CurrentWorld->GetNetMode() != NM_Client)
			{
				// server; spawn a new PlayerController immediately
				if (!NewPlayer->SpawnPlayActor("", OutError, CurrentWorld))
				{
					RemoveLocalPlayer(NewPlayer);
					NewPlayer = nullptr;
				}
			}
			// 这个分支不是很了解
			else if (CurrentWorld->IsPlayingReplay())
			{
			}
			else
			{
				// client; ask the server to let the new player join
				// 是给服务器发消息，但不知道怎么客户端收到回复消息
				TArray<FString> Options;
				NewPlayer->SendSplitJoin(Options);
			}
		}
	}
	return NewPlayer;
}
```
通过两个函数的转发创建出了LocalPlayer，并通过GameInstance添加到了数组中，然后会根据参数来判断是否要在服务器创建出对应的PlayerController。
所以看上去Init这部分就是，在客户端创建出了GameInstance，然后创建出LocalPlayer了，服务器只创建出了GameInstance。
# 2 UEngine::LoadMap
```cpp
bool UEngine::LoadMap( FWorldContext& WorldContext, FURL URL, class UPendingNetGame* Pending, FString& Error )
{

	// 这边首先是会通过GameInstance来清理当前World中Loacal相关的东西，比如PlayerController,Pawn
	// Disassociate the players from their PlayerControllers in this world.
	if (WorldContext.OwningGameInstance != nullptr)
	{
		for(auto It = WorldContext.OwningGameInstance->GetLocalPlayerIterator(); It; ++It)
		{
			ULocalPlayer *Player = *It;
			if(Player->PlayerController && Player->PlayerController->GetWorld() == WorldContext.World())
			{
				if(Player->PlayerController->GetPawn())
				{
					WorldContext.World()->DestroyActor(Player->PlayerController->GetPawn(), true);
				}
				WorldContext.World()->DestroyActor(Player->PlayerController, true);
				Player->PlayerController = nullptr;
			}
			// reset split join info so we'll send one after loading the new map if necessary
			Player->bSentSplitJoin = false;
			// When loading maps, clear out mids that are referenced as they may prevent the world from shutting down cleanly and the local player will not be cleaned up until later
			Player->CleanupViewState(); 
		}
	}
	// 清理完当前World中的各种数据，然后就开始转让GameInstance，WorldContext所有权
	NewWorld->SetGameInstance(WorldContext.OwningGameInstance);
	GWorld = NewWorld;
	WorldContext.SetCurrentWorld(NewWorld);
	WorldContext.World()->WorldType = WorldContext.WorldType;
	
	// 在偏后的位置有这么一段代码
	// 根据GameInstance中的LoacalPlayer，在新World中创建出PlayerController
	// Spawn play actors for all active local players
	if (WorldContext.OwningGameInstance != NULL)
	{
		for(auto It = WorldContext.OwningGameInstance->GetLocalPlayerIterator(); It; ++It)
		{
			FString Error2;
			if(!(*It)->SpawnPlayActor(URL.ToString(1),Error2,WorldContext.World()))
			{
				UE_LOG(LogEngine, Fatal, TEXT("Couldn't spawn player: %s"), *Error2);
			}
		}
	}
}
```
在LoadMap中会首先把当前World中的LocalPlayer相关的都删掉，然后在根据GameInstance在新World中创建新的。