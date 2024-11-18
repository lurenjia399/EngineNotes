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

	// 这个是设置ResetLocationOffset的时间，也是一样通过timer记录经过的时间，然后经过时间/总时间得到比值，然后将LocationOffset按照比值插值到0。
	UpdateCameraLocationOffset(DeltaTime);
	// 和上面插值一样，插值ResetRotationOffset，都是从Offset到0的插值
	UpdateCameraRotationOffset(DeltaTime);
	// 和上面一样，插值FovOffset
	UpdateCameraFovOffset(DeltaTime);

	// 设置了CurStageTimePercent和CurCurveValue，CurStageTimePercent代表Stage切换进度由ChangeStateTimeCur / ChangeStateTimeTotal得出，CurCurveValue代表曲线值，横坐标是切换进度，纵坐标是曲线值
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

##### 4.1 UpdateCameraRotation_Implementation
```cpp
UAzurePlayerCameraViewModeComponentBase::UpdateCameraRotation_Implementation(float DeltaTime)
{
	/*
	// 首先是根据下面这个枚举来计算DesiredRotation和DesiredSpeed
	enum class EUpdateRotationIndex : uint8
	{
		UsePawnRotation = 0 UMETA(DisplayName = "使用Pawn的Rotation"),
		UseControllerRotation = 1 UMETA(DisplayName = "使用Controller的Rotation"),
		UseConfigRotation = 2 UMETA(DisplayName = "使用config配置Rotation"),
		UseLastFrameRotation = 3 UMETA(DisplayName = "使用上一帧的旋转"),
		UseFreeControl = 4 UMETA(DisplayName = "自由控制Rotation"),
	};
	*/
	switch (CameraViewModeBaseData.UpdateCameraRotationIndex)
	{
		// 就拿这一个举例了，就是对当前Rotation和DesiredRotation进行插值，插值的比例是Stage的切换进度
		case EUpdateRotationIndex::UseFreeControl:
		{
			CameraViewModeBaseData.DesiredCameraRotation = UKismetMathLibrary::RLerp(CameraViewModeBaseData.Data_BeforeChangeState.DesiredCameraRotation, CameraViewModeBaseData.Data_DesiredChangeState.DesiredCameraRotation, CameraViewModeBaseData.CurCurveValue, true);
			CameraViewModeBaseData.CameraRotationSpeed = UKismetMathLibrary::Lerp(CameraViewModeBaseData.Data_BeforeChangeState.CameraRotationSpeed, CameraViewModeBaseData.Data_DesiredChangeState.CameraRotationSpeed, CameraViewModeBaseData.CurCurveValue);
			break;
		}
	}
	/*
	// 接下来是根据offset枚举来计算offset值
	enum class EUpdateRotationOffsetIndex : uint8
	{
		UseNoneRotationOffset = 0 UMETA(DisplayName = "不使用增加的CameraOffset"),
		UseCamAnimRotationOffset = 1 UMETA(DisplayName = "使用CamAnim里增加的CameraOffset"),
		UseBreathRotationOffset = 2 UMETA(DisplayName = "使用呼吸的CameraOffset"),
		UseCamAnimAndBreathRotationOffset = 3 UMETA(DisplayName = "使用CamAnim里增加的CameraOffset再叠加呼吸的CameraOffset"),
	};
	*/
	switch (CameraViewModeBaseData.UpdateCameraRotationOffsetIndex)
	{
		// 这个是说用动画蓝图里的偏移，调用接口获取
		case EUpdateRotationOffsetIndex::UseCamAnimRotationOffset:
		{
			CameraViewModeBaseData.CameraRotationOffset = PlayerCameraManager->GetAnimRotationOffset();
			break;
		}
		// 这个是添加角色的呼吸，读取了配置的Offset，这里我们没用注掉了
		case EUpdateRotationOffsetIndex::UseBreathRotationOffset:
		{
	// 		auto Char = Cast<AAzureCharacterBase>(PlayerCameraManager->GetControlledPawn());
	// 		if (IsValid(Char))
	// 		{
	// 			CameraViewModeBaseData.CameraRotationOffset.Pitch = Char->BreathRotation.Pitch;
	// 			CameraViewModeBaseData.CameraRotationOffset.Yaw = Char->BreathRotation.Yaw;
	// 		}
			break;
		}
		// 默认是没有Offset的
		default:
		{
			CameraViewModeBaseData.CameraRotationOffset = FRotator::ZeroRotator;
			break;
		}
	}
	// 第三步就是计算CurCameraRotation，也是通过插值计算,目标值是DesiredCameraRotationWithCache是通过DesiredRotation和Offset等一系列相加的
	CameraViewModeBaseData.CurCameraRotation = UKismetMathLibrary::RInterpTo(CameraViewModeBaseData.CurCameraRotation, DesiredCameraRotationWithCache, DeltaTime, CameraViewModeBaseData.CameraRotationSpeed);
	// 第四步就是刷新锁定的YawPitch
	RefreshCoLookYawPitch(DeltaTime);
}

```

#### 4.2 锁定具体实现RefreshCoLookYawPitch

```cpp
bool UAzurePlayerCameraViewModeComponentBase::RefreshCoLookYawPitch(float dt, int* piRet)
{
	// 锁定目标根节点位置
	FVector vCoLookAtRootPos = pCoLookAtActor->GetActorLocation();
	// 具体锁定目标位置，可以配置调整
	FVector vCoLookPos = vCoLookAtRootPos + pCoLookAtActor->GetActorUpVector() * (m_pCoLookAtInfo->fHei * m_pCoLookAtInfo->fHeiRatio);
	// 玩家根节点位置
	FVector CamLookAt_Host_Pos = CurTarget->GetActorLocation();
	// 摄像机指向锁定目标向量
	FVector vCamToCoLookRoot = vCoLookAtRootPos - CameraViewModeBaseData.CurCameraLocation;
	// 玩家根节点指向锁定目标向量
	FVector vHostToCoLookRoot = vCoLookAtRootPos - CamLookAt_Host_Pos;
	// 摄像机指向玩家向量
	FVector vLookDir_CamToLookDest = CameraViewModeBaseData.CurTargetLocation - CameraViewModeBaseData.CurCameraLocation;
	// 这个方法是根据配置，调整pitch轴
	Inner_CoLookAt_AdjustPitch(fDist_CamToCoLookRoot, vCamToCoLookRoot, &pParamsCommon);
	
	//	非AlwaysLock，且非ChangingYaw：在锁定目标pCoLookAtActor的Bounds出屏幕范围时才ChangeYaw
	if (Mode != CAM_COLOOK_MODE::AlwaysLock && !bCoLookChangingYaw)
	{
		// 判断锁定目标的Comp是否渲染了
		bool bLastRendered = false;
		static const float LastRenderedToleranceTime = 0.1f;
		auto MeshComp = UAzureGameplayLibrary::GetMeshComponentInActor(pCoLookAtActor);
		if (MeshComp)
			bLastRendered = MeshComp->WasRecentlyRendered(LastRenderedToleranceTime);
		else
			bLastRendered = pCoLookAtActor->WasRecentlyRendered(LastRenderedToleranceTime);
		// 如果锁定目标未渲染，直接认定需要改变Yaw
		if (!bLastRendered)
		{
			bYawNeedChange = true;
		}
		// 如果锁定目标已经渲染了，就需要判断其Bounds是否在屏幕里了
		else
		{
			/*
			判断是否在屏幕里的具体步骤:
			1 获取屏幕显示范围，通过APlayerController::GetViewportSize方法
			2 根据Socket构建AABB包围盒，没有Socket就通过Comp->Bounds属性获取FBox
			3 调整屏幕判定矩形，屏幕显示范围 * 配置（0.9）
			4 将Box的8个顶点依次投影到屏幕上，判断是否在判定矩形中，如果在矩形中就直接返回在屏幕里。投影用APlayerController::ProjectWorldLocationToScreenWithDistance方法，判断点是否在矩形中就判断x,y坐标是否在矩形范围里就行
			5 如果没有点在矩形中，就需要判断FBox的12条边是否在矩形中。线段是否在矩形就判断是否与矩形的对角线相交就行，两条线段相交就判断线段端点是否在另一条的一侧，点在线段一侧用叉积就行
			6 如果没有线段在矩形中，就需要改变摄像机的Yaw。根据配置计算出摄像机到LookTarget向量的Yaw
			*/
			bool bInView = UAzureGameplayLibrary::IsActorModelOrCapsuleInViewport(pCoLookAtActor,
				m_pCoLookAtInfo->SkelSocketName,
				pParamsCommon->JudgeInView_UseSkelSocketBoundExt,
				pParamsCommon->JudgeInView_UseCapsuleBoundsFirst,
				pParamsCommon->JudgeInView_ModelBoundsScale,
				pParamsCommon->JudgeInView_ScreenWidthRatio,
				pParamsCommon->JudgeInView_ScreenHeightRatio);
			if (!bInView)
				bYawNeedChange = true;
		}
		if (!bYawNeedChange)
		{
			Rot_CamToLookDest = FRotator(m_fPitchDegDest, m_fYawDegDest, CameraViewModeBaseData.CurCameraRotation.Roll);
			fOrgCamToLookDestYaw = Rot_CamToLookDest.Yaw;
		}
	}
	// bYawNeedChange 如果LockTarget不在屏幕中就为true，意为需要改变Yaw
	if (bYawNeedChange)
	{
		// 求得玩家到LockTarget 和 摄像机到LookTarget 夹角实际值并限制到[-180， 180]
		fYawDiff = FMath::UnwindDegrees(Rot_HostToLookDest.Yaw - Rot_CamToLookDest.Yaw);
		if (Mode == CAM_COLOOK_MODE::HalfLock || Mode == CAM_COLOOK_MODE::AlwaysLock)
		{
			// 根据摄像机偏移，插值求出 玩家到LockTarget 和 摄像机到LookTarget 夹角理论值
			float fMinDist = GetMinOffsetDist();
			float fMaxDist = GetMaxOffsetDist();
			float fDistAlpha = (CurDistOffset - fMinDist) / (fMaxDist - fMinDist);
			fDistAlpha = FMath::Clamp(fDistAlpha, 0.f, 1.f);
			float DesireYaw = FMath::Lerp(pParamsCommon->DesireYaw_MinDist, pParamsCommon->DesireYaw_MaxDist, fDistAlpha);
			float _CoLookAt_fDiffYawRange = m_pCoLookAtInfo->m_bChangeRotateStart ? DesireYaw : pParamsCommon->DiffYawRange;
			
			// 根据夹角实际值和夹角理论值，计算出摄像机到LookTarget的Yaw
			if (fYawDiff > _CoLookAt_fDiffYawRange + COLOOK_YAW_ERROR)
			{
				Rot_CamToLookDest.Yaw += FMath::Max(fYawDiff - _CoLookAt_fDiffYawRange, _CoLookAt_fDiffYawRange * 0.1f);
			}
			else if (fYawDiff < -_CoLookAt_fDiffYawRange - COLOOK_YAW_ERROR)
			{
				Rot_CamToLookDest.Yaw -= FMath::Max(-(fYawDiff + _CoLookAt_fDiffYawRange), _CoLookAt_fDiffYawRange * 0.1f);
			}
			else
			{
				bool b = true;
			}
		}
	}
	if (bYawNeedChange)
	{
		// 拿到LookTarget指向摄像机向量
		vLookDir_CamToLookDest = Rot_CamToLookDest.Vector();
		FVector vLookDir_LookDestToCam = -vLookDir_CamToLookDest;
		// 用LookTarget指向摄像机向量 和 以玩家根节点为中心，摄像机位置偏移为半径的球求交，得到Desired的摄像机位置
		float fHitLenInner = -1.f, fHitLenOuter = -1.f;
		if (UAzureMathLibrary::LineSphereIntersectionWithHitOut(fHitLenInner, fHitLenOuter,
			vCoLookAtDest, vLookDir_LookDestToCam, fLookDist_CamToLookDest, CamLookAt_Host_Pos, CurDistOffset)
			&& fHitLenOuter > 0 //	用远距离的
			)
		{
			// Desired摄像机位置
			FVector vIntersect = vCoLookAtDest + vLookDir_LookDestToCam * fHitLenOuter;
			// Desired摄像机位置看向玩家根节点
			FVector vDesireCamDir = CamLookAt_Host_Pos - vIntersect;
			float fLookDist_DesireCamDir;
			vDesireCamDir.ToDirectionAndLength(vDesireCamDir, fLookDist_DesireCamDir);
			FRotator vDesireCamDir_Rot = vDesireCamDir.ToOrientationRotator();
			// 记录Yaw
			m_fYawDegDest = vDesireCamDir_Rot.Yaw;
		}
	}
	bool bReach = false;
	if ( Mode == CAM_COLOOK_MODE::AlwaysLock || (m_pCoLookAtInfo && !m_pCoLookAtInfo->m_bChangeRotateEnd))
	{
		// 上一帧摄像机旋转
		FRotator fPreRotation = CameraViewModeBaseData.CurCameraRotation;
		// 计算出的Desired摄像机旋转，总共摄像机需要旋转到DesiredRotation
		FRotator DesireRotaion = FRotator(m_fPitchDegDest, m_fYawDegDest, CameraViewModeBaseData.CurCameraRotation.Roll);
		// 根据帧长和速度计算这一帧能够旋转多少
		FRotator InterpRotation = UKismetMathLibrary::RInterpTo(fPreRotation, DesireRotaion, dt, GetCamChangeSpeed());
		if (m_pCoLookAtInfo->m_CoLookType == CAM_COLOOK_TYPE::LockTarget || m_pCoLookAtInfo->Mode == CAM_COLOOK_MODE::AlwaysLock)
			bReach = false;
		else
			bReach = FVector::DotProduct(InterpRotation.Vector(), DesireRotaion.Vector()) > COLOOK_ANGLE_THRESH_Cos;

		CameraViewModeBaseData.RotationOffset_Cache += InterpRotation - fPreRotation;

#if COLOOK_DEBUG
		UE_LOG(LogTemp, Log, TEXT("bCoLookChangingYaw: Cur[%f -> %f], Dest[%f], fYawSpeed[%f], dt[%f], YawMaxSpeed[%f],fYawDegDiff[%f],bReach[%d],CoLookType[%d]"),
			fPreRotation.Yaw, CurYaw, m_fYawDegDest, GetCamChangeSpeed(), dt, pParamsCommon->MaxRotateSpeed, fYawDegDiff, bReach?1:0,m_pCoLookAtInfo->m_CoLookType);
#endif
		m_pCoLookAtInfo->m_bChangeRotateStart = true;
		if (Rot_CamToLookDest.Yaw != fOrgCamToLookDestYaw)
			m_pCoLookAtInfo->m_bChangingYaw = true;
		if (FMath::Abs(CameraViewModeBaseData.CurCameraRotation.Pitch - m_fPitchDegDest) > COLOOK_THRESH)
			m_pCoLookAtInfo->m_bChangingPitch = true;

		if (bReach)
		{
			if (m_pCoLookAtInfo)
			{
				m_pCoLookAtInfo->m_bChangingYaw = false;
				m_pCoLookAtInfo->m_bChangingPitch = false;
				m_pCoLookAtInfo->m_bChangeRotateStart = false;
				m_pCoLookAtInfo->m_bChangeRotateEnd = true;
			}
		}

#if COLOOK_DEBUG
		//	Cur Dir
		color = FColor::Green;
		FVector rotCur = CameraViewModeBaseData.CurCameraRotation.Vector();
		DrawDebugDirectionalArrow(pWorld, CameraViewModeBaseData.CurTargetLocation, CameraViewModeBaseData.CurTargetLocation + rotCur * fLen, arrowSize, color, false, fLifeTime);
#endif
	}
}
```