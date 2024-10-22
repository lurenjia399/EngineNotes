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

	//把上一帧的数据存起来
	LastFrameCacheCameraLocation = CameraLocation;
	LastFrameCacheCameraRotation = CameraRotation;
	LastFrameCacheCameraFOV = CameraFOV;

	// bFinishReceiveBeginPlay就是一个标志位，蓝图走到了BeginPlay就会赋值这个标志位
	if (bFinishReceiveBeginPlay)
	{
		// 提前声明的参数，分别是当前ViewMode的信息和上一帧ViewMode的信息
		FVector CurViewModeLocation = FVector::ZeroVector;
		FRotator CurViewModeRotation = FRotator::ZeroRotator;
		float CurViewModeFOV = 90;

		FVector LastViewModeLocation = FVector::ZeroVector;
		FRotator LastViewModeRotation = FRotator::ZeroRotator;
		float LastViewModeFOV = 90;

		// 计算当前ViewMode，通过对当前ViewMode的Tick，计算出摄像机的信息
		UAzurePlayerCameraViewModeComponentBase* CurViewModeComponent = GetCurViewModeReference();
		if (CurViewModeComponent)
		{
			CurViewModeComponent->TickFunc(DeltaTime);
			CurViewModeLocation = bCanChangeLocation ? CurViewModeComponent->GetCurCameraLocation() : LastFrameCacheCameraLocation;
			CurViewModeRotation = bCanChangeRotation ? CurViewModeComponent->GetCurCameraRotation() : LastFrameCacheCameraRotation;
			CurViewModeFOV = bCanChangeFOV ? CurViewModeComponent->GetCurCameraFOV() : LastFrameCacheCameraFOV;
		}
		// ChangeViewModeValue 是当前切换ViewMode的比例，小于1就说明没有完全切换成功
		UAzurePlayerCameraViewModeComponentBase* PrevViewModeComponent = nullptr;
		if (ChangeViewModeValue < 1)
		{
			// 计算切换前的ViewMode，通过对切换前ViewMode的Tick，计算出摄像机信息
			PrevViewModeComponent = GetViewModeReferenceByIndex(LastViewMode);
			if (PrevViewModeComponent)
			{
				PrevViewModeComponent->TickFunc(DeltaTime);
				LastViewModeLocation = bCanChangeLocation ? PrevViewModeComponent->GetCurCameraLocation() : LastFrameCacheCameraLocation;
				LastViewModeRotation = bCanChangeRotation ? PrevViewModeComponent->GetCurCameraRotation() : LastFrameCacheCameraRotation;
				LastViewModeFOV = bCanChangeFOV ? PrevViewModeComponent->GetCurCameraFOV() : LastFrameCacheCameraFOV;
			}
		}
		// 将摄像机信息，对切换前信息和当前信息进行插值
		CameraLocation = UKismetMathLibrary::VLerp(LastViewModeLocation, CurViewModeLocation, ChangeViewModeValue);
		CameraRotation = LastViewModeRotation + ChangeViewModeValue * (CurViewModeRotation - LastViewModeRotation).GetNormalized();
		CameraFOV = UKismetMathLibrary::Lerp(LastViewModeFOV, CurViewModeFOV, ChangeViewModeValue);
		// 是否使用Contrller的旋转，如果当前ViewMode使用切换前ViewMode不使用，则直接设置旋转为当前ViewMode的旋转。如果当前ViewMode使用切换前ViewMode也使用，则设置插值出来的旋转
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

		// 上边都是对当前ViewMode和切换前ViewMode处理。这里是执行ViewMode改变的具体部分。也就是某一帧直接切换ViewMode为新，下一帧及其以后帧开始由切换前ViewMode插值到新的ViewMode
		if (bCanChangeViewMode)
		{
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
	// 缓存下信息
	CachedCameraModeLocation = CameraLocation;
	CachedCameraModeRotation = CameraRotation;
	CachedCameraModeFOV = CameraFOV;
	return true;
}
```

UAzurePlayerCameraViewModeComponentBase::TickFunc
```cpp
void UAzurePlayerCameraViewModeComponentBase::TickFunc(float DeltaTime)
{
	// 摄像机是否有效，无效返回
	if (!PlayerCameraManagerIsValid())
	{
		return;
	}
	// 是否使用覆盖的信息
	if (bUseOverrideInTickFunc)
	{
		CameraViewModeBaseData.CurCameraLocation = CameraViewModeBaseData.OverrideLocation;
		CameraViewModeBaseData.CurCameraRotation = CameraViewModeBaseData.OverrideRotation;
		CameraViewModeBaseData.CurCameraFOV = CameraViewModeBaseData.OverrideFOV;
		return;
	}
	// 摄像机的Transform信息使用pawn身上的LookAtBone骨骼位置
	if (CameraViewModeBaseData.UpdateTargetLocationIndex == EUpdateTargetLocationIndex::UsePawnLookAtBone)
	{
		AActor* LookTarget = CameraViewModeBaseData.LookAtTarget.Get();
		AActor* CurTarget = LookTarget ? LookTarget : PlayerCameraManager->GetCurTargetActor();
		if (!CurTarget)
		{
			return;
		}
	}

	// 如果不是锁定切换State的时间，就计算下切换时间，和切换ViewMode的流程一样
	if (!CameraViewModeBaseData.bLockChangeStateTime)
	{
		if (CameraViewModeBaseData.ChangeStateTimeCur >= CameraViewModeBaseData.ChangeStateTimeTotal)
		{
			CameraViewModeBaseData.ChangeStateTimeCur = 0;
			CameraViewModeBaseData.ChangeStateTimeTotal = 0;
		}
		else
		{
			if (CameraViewModeBaseData.ChangeStateTimeCur < CameraViewModeBaseData.ChangeStateTimeTotal)
			{
				CameraViewModeBaseData.ChangeStateTimeCur += DeltaTime;
			}
			if (CameraViewModeBaseData.ChangeStateTimeCur > CameraViewModeBaseData.ChangeStateTimeTotal)
			{
				CameraViewModeBaseData.ChangeStateTimeCur = CameraViewModeBaseData.ChangeStateTimeTotal;
				if (PlayerCameraManagerIsValid())
				{
					PlayerCameraManager->BroadcastChangeStateEnd(CameraViewModeBaseData.ViewModeIndex);
				}
			}
		}
	}

	UpdateCameraLocationOffset(DeltaTime);
	UpdateCameraRotationOffset(DeltaTime);
	UpdateCameraFovOffset(DeltaTime);


	CalculateCurveValue();
	UpdateCameraRotation(DeltaTime);
	UpdateCameraOffset(DeltaTime);
	UpdateTargetLocation(DeltaTime);
	UpdatePivotOffset(DeltaTime);
	UpdateCameraLocation(DeltaTime);
	UpdateCameraFOV(DeltaTime);


	UpdateDof(GetRealOffsetForwardDist());

	if(!bSkipCheckCharacterLookAt)
	{
		CheckCharacterLookAt();
	}
}
```