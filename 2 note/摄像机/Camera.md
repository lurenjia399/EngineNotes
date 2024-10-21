ue中camera的创建是在APlayerController::PostInitializeComponents方法中
``` cpp
void APlayerController::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	if ( IsValid(this) && (GetNetMode() != NM_Client) )
	{
		// create a new player replication info
		// 这个是在uds上创建playerState，playerState是需要同步到客户端的
		InitPlayerState();
	}

	// 创建我们的PlayerCameraManager
	SpawnPlayerCameraManager();
	// ResetCameraMode
	ResetCameraMode(); 

	// 这边是在Client下创建一个HUD，好像不知道HUD是干嘛的
	if ( GetNetMode() == NM_Client )
	{
		SpawnDefaultHUD();
	}
	// 创建CheatManager，作弊Manager。干嘛的不清楚
	AddCheats();

	// 看注释说是，还不能用需要初始化？
	bPlayerIsWaiting = true;
	// 设置当前StateName
	StateName = NAME_Spectating; // Don't use ChangeState, because we want to defer spawning the SpectatorPawn until the Player is received
}
```
# 项目中
 在BP_AzureRuntimeGameMode 中设置 BP_RuntimePlayerController，然后在PlayerController中设置的AzureCameraManagerBP。并且用AzurePlayerCameraManager.lua重写。

## AzureCameraManager 创建

### 1 构造函数
```cpp
AAzureCameraManager::AAzureCameraManager(const FObjectInitializer& ObjectInitializer)  
    : Super(ObjectInitializer)  
{  
    // Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.  
    PrimaryActorTick.bCanEverTick = true;  
  
    // TickGroup改到TG_PostPhysic，避免GameThread取Curve值卡顿  
    PrimaryActorTick.TickGroup = ETickingGroup::TG_PostPhysics;  
    PrimaryActorTick.EndTickGroup = ETickingGroup::TG_PostPhysics;  
	// 创建了SubObject，都是不同模式的
    ViewModeComponentTPS = CreateDefaultSubobject<UAzurePlayerCameraViewModeComponentTPS>(TEXT("ViewModeComponentTPS"));  
    ViewModeComponentFPS = CreateDefaultSubobject<UAzurePlayerCameraViewModeComponentFPS>(TEXT("ViewModeComponentFPS"));  
    ViewModeComponentVehicle = CreateDefaultSubobject<UAzurePlayerCameraViewModeComponentVehicle>(TEXT("ViewModeComponentVehicle"));  
    ViewModeComponentFixed = CreateDefaultSubobject<UAzurePlayerCameraViewModeComponentFixed>(TEXT("ViewModeComponentFixed"));  
    ViewModeComponentFree = CreateDefaultSubobject<UAzurePlayerCameraViewModeComponentFree>(TEXT("ViewModeComponentFree"));  
    ViewModeComponentCustom = CreateDefaultSubobject<UAzurePlayerCameraViewModeComponentCustom>(TEXT("ViewModeComponentCustom"));  
    ViewModeComponentHome = CreateDefaultSubobject<UAzurePlayerCameraViewModeComponentHome>(TEXT("ViewModeComponentHome"));  
  
}
```
### 2 InitializeComponents
```cpp
void AAzureCameraManager::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	// 将创建的不同viewMode的SubObject都添加到ActorComponents里
	TArray<UActorComponent*> ActorComponents;
	GetComponents(UAzurePlayerCameraViewModeComponentBase::StaticClass(), ActorComponents);
	// 将不同的ViewMode添加到ViewModeMap，key是ViewMode的索引，value是ViewMode
	for (UActorComponent* ActorComponent : ActorComponents)
	{
		if (UAzurePlayerCameraViewModeComponentBase* ViewModeComponentBase = Cast<UAzurePlayerCameraViewModeComponentBase>(ActorComponent))
		{
			AddNewViewModeComponent(ViewModeComponentBase, ViewModeComponentBase->CameraViewModeBaseData.ViewModeIndex);
		}
	}
	// 在非uds情况，创建CameraAnimInst，动画蓝图
	bNeedCamAnimInst = !UKismetSystemLibrary::IsDedicatedServer(this);
	if (bNeedCamAnimInst)
	{
		InitCamAnimInst();
	}
}
```
### 3 lua侧的ReceiveBeginPlay
```lua
def.method().ReceiveBeginPlay = function(self)
    self.Super:ReceiveBeginPlay()
    -- 非服务器情况下走
    local bDedicatedServer = UEGlobal.KismetSystemLibrary.IsDedicatedServer(self)
    if not bDedicatedServer then
	    -- 初始化CameraMangaer中的self.CameraStateMaps。key是不同ViewMode的索引，value是不同的lua配置文件
        self:InitCameraManager()
		-- 设置标志位，完成beginplay的创建
        self.bFinishReceiveBeginPlay = true
		-- 广播CameraInitEvent事件
		AzureEventMgr.EventManager:raiseEvent(self, CameraEvents.CameraInitEvent())
    end
end
```

### 4 Tick
```cpp
void AAzureCameraManager::Tick(float DeltaSeconds)
{
	Super::Tick(DeltaSeconds);

	//先更新相机的自定义动画效果。
	// 这里边应该是涉及了放技能的时候摄像机的移动
	if (bNeedCamAnimInst && CamManTickAnim != 0)
	{
		CheckCamAnimInst();
		UpdateCamAnimInst(DeltaSeconds);
	}
	// 摄像机的特效
	UpdateCameraEffects(DeltaSeconds);
	// 这个我看就是正儿八经的处理摄像机参数
	bCachedCameraUpdateResult = BlueprintUpdateCameraCPP(DeltaSeconds);

	PostUpdateCameraCPP(DeltaSeconds);

	OnCameraUpdatedSet.Broadcast(this, DeltaSeconds);
}
```

BlueprintUpdateCameraCPP方法
```cpp
/*
*/
bool AAzureCameraManager::BlueprintUpdateCameraCPP(float DeltaTime)
{
	PlayerCameraManagerDeltaTime = DeltaTime;

	// 更新切换时间，ChangeViewModeTimeCur表示ViewModel已经切换了多少时间，ChangeViewModeTimeTotal表示ViewModel切换所需的时间
	if (ChangeViewModeTimeCur < ChangeViewModeTimeTotal)
	{
		ChangeViewModeTimeCur += DeltaTime;
	}
	if (ChangeViewModeTimeCur > ChangeViewModeTimeTotal)
	{
		ChangeViewModeTimeCur = ChangeViewModeTimeTotal;
		OnChangeViewModeEnd();
	}
	// ChangeViewModeValue 拿到当前ViewModel切换的比例，用于融合时候用
	float ChangeViewModeValue = CalculateChangeViewModeValue();

	//deprecated逻辑
	UWorld* CurWorld = GetWorld();
	//把上一帧的数据存起来
	LastFrameCacheCameraLocation = CameraLocation;
	LastFrameCacheCameraRotation = CameraRotation;
	LastFrameCacheCameraFOV = CameraFOV;

	//完成了初始化
	if (bFinishReceiveBeginPlay)
	{
		FVector CurViewModeLocation = FVector::ZeroVector;
		FRotator CurViewModeRotation = FRotator::ZeroRotator;
		float CurViewModeFOV = 90;

		FVector LastViewModeLocation = FVector::ZeroVector;
		FRotator LastViewModeRotation = FRotator::ZeroRotator;
		float LastViewModeFOV = 90;

		//计算当前ViewMode
		UAzurePlayerCameraViewModeComponentBase* CurViewModeComponent = GetCurViewModeReference();
		if (CurViewModeComponent)
		{
			CurViewModeComponent->TickFunc(DeltaTime);
			CurViewModeLocation = bCanChangeLocation ? CurViewModeComponent->GetCurCameraLocation() : LastFrameCacheCameraLocation;
			CurViewModeRotation = bCanChangeRotation ? CurViewModeComponent->GetCurCameraRotation() : LastFrameCacheCameraRotation;
			CurViewModeFOV = bCanChangeFOV ? CurViewModeComponent->GetCurCameraFOV() : LastFrameCacheCameraFOV;
		}

		UAzurePlayerCameraViewModeComponentBase* PrevViewModeComponent = nullptr;
		if (ChangeViewModeValue < 1) //小于1说明状态切换不完全，还需要计算上一帧的ViewMode
		{
			//计算 LastViewMode
			PrevViewModeComponent = GetViewModeReferenceByIndex(LastViewMode);
			if (PrevViewModeComponent)
			{
				PrevViewModeComponent->TickFunc(DeltaTime);
				LastViewModeLocation = bCanChangeLocation ? PrevViewModeComponent->GetCurCameraLocation() : LastFrameCacheCameraLocation;
				LastViewModeRotation = bCanChangeRotation ? PrevViewModeComponent->GetCurCameraRotation() : LastFrameCacheCameraRotation;
				LastViewModeFOV = bCanChangeFOV ? PrevViewModeComponent->GetCurCameraFOV() : LastFrameCacheCameraFOV;
			}
		}

		CameraLocation = UKismetMathLibrary::VLerp(LastViewModeLocation, CurViewModeLocation, ChangeViewModeValue);
		CameraRotation = LastViewModeRotation + ChangeViewModeValue * (CurViewModeRotation - LastViewModeRotation).GetNormalized();
		CameraFOV = UKismetMathLibrary::Lerp(LastViewModeFOV, CurViewModeFOV, ChangeViewModeValue);


		if (CurViewModeComponent && CurViewModeComponent->CameraViewModeBaseData.bCameraRotationSyncToController)
		{
			if (auto ch = GetCameraCharacter())
			{
				if (PrevViewModeComponent && PrevViewModeComponent->CameraViewModeBaseData.bCameraRotationSyncToController)
				{
					ch->SetControlRotation(CameraRotation);
				}
				else
				{
					ch->SetControlRotation(CurViewModeRotation);
				}
			}
		}

		//计算要切换的Viewmode
		if (bCanChangeViewMode)
		{
			//如果当前和要切换的不一致，则得把要切换的算起来
			if (CurViewMode != DesiredViewMode)
			{
				UAzurePlayerCameraViewModeComponentBase* DesiredViewModeComponent = GetDesiredViewModeReference();
				if (DesiredViewModeComponent)
				{
					if (CurViewMode != DesiredViewMode)
					{
						DesiredViewModeComponent->TickFunc(DeltaTime);
					}
				}
			}
		}
	}

	CachedCameraModeLocation = CameraLocation;
	CachedCameraModeRotation = CameraRotation;
	CachedCameraModeFOV = CameraFOV;


	return true;
}
```