# 3 particle System 粒子特效系统

## 1 编辑器界面
![[image-20240304172808296.png]]

我们就按照这个界面讲述，首先看下界面的布局：
![[image-20240304173036847.png]]

其中

```c++
/*  
{  
    CascadeConfiguration.h  
    看上去就是把发射器模块缓存起来，干什么用的  
    Cascade.cpp 和 .h  
    就是用来描述整个特效的总类  
    CascadeParticleSystemComponent.h  
    这个就是窗口里播放的那个特效吧  
}  
​  
{  
    CascadeEmitterCanvasClient.cpp 和 .h  
    是用来描述，绘制Emitters这个界面里的Emitter的，也可以响应点击事件啥的  
    CascadeEmitterCanvas.cpp 和 .h  
    是用来描述，绘制Emitters这个外观界面的  
}  
{  
    CascadeActions.cpp 和 .h  
    应该是用来注册编辑器上的ui的，  
    CascadeEmitterHitProxies.cpp 和 .h  
    用来创建声明类的，点击到的东西具体是个什么类型  
}  
​  
{  
    ScascadePreviewViewport.cpp 和 .h  
    用来创建Viewport的  
    CascadeEdPreviewViewportClient.cpp 和 .h  
    创建Viewport里面的具体的东西  
    CascadePreviewViewportToolBar.cpp 和 .h  
    这个是Viewport窗口里面那个两个Toolbar  
}  
​  
{  
    CascadeModule.cpp 和 .h  
    这个就是看上去就是一个模块的主类，里面能创建Cascade这个界面  
}  
*/
```

### 1 创建新的发射器 OnNewEmitter

```c++
void FCascade::OnNewEmitter()  
{  
    FText Transaction = NSLOCTEXT("UnrealEd", "NewEmitter", "Create New Emitter");  
      
    // https://zhuanlan.zhihu.com/p/606722605  
    // 这个的作用是 Undo / Redo  
    BeginTransaction( Transaction );  
    ParticleSystem->PreEditChange(NULL);  
    ParticleSystemComponent->PreEditChange(NULL);  
​  
    // 这部分就是构建新的ParticleSpriteEmitter  
    UClass* NewEmitClass = UParticleSpriteEmitter::StaticClass();  
    UParticleEmitter* NewEmitter = NewObject<UParticleEmitter>(ParticleSystem, NewEmitClass, NAME_None, RF_Transactional);  
    UParticleLODLevel* LODLevel = NewEmitter->GetLODLevel(0);  
    if (LODLevel == NULL)  
    {  
        /*  
        在构建新的emitter的时候，会走进来。  
        创建一个ParticleLODLevel，也就是界面上那个发射器，创建的时候会根据参数值，  
            来将这个新的ParticleLODLevel放入到LODLevels数组的位置中，这个数组  
            key就是LOD的值，value就是对应的ParticleLODLevel。  
        在创建ParticleLODLevel的时候也会创建两个ParticleModule，也就是Required和Spawn。  
        */  
        int32 Index = NewEmitter->CreateLODLevel(0);  
        LODLevel = NewEmitter->GetLODLevel(0);  
    }  
    // 这两句就是在界面上设置Emitter左边条的颜色  
    NewEmitter->EmitterEditorColor = FColor::MakeRandomColor();  
    NewEmitter->EmitterEditorColor.A = 255;  
​  
    // Set to sensible default values  
    // 就是设置一些默认值  
    /*  
    创建了四个ParticleModule，Lifetime,InitialSize,InitialVelocity,ColoeOverLife  
    */  
    NewEmitter->SetToSensibleDefaults();  
    // 更新Emitter里的每一个ParticleLODLevel中的每一个ParticleModule  
    // 注意这个里面会执行到这个CacheEmitterModuleInfo方法，他会将ParticleModule的一些信息缓存到数组中  
    NewEmitter->UpdateModuleLists();  
    // 广播个数据改变  
    NewEmitter->PostEditChange();  
    // https://zhuanlan.zhihu.com/p/606722605  
    // 这个的作用是 Undo / Redo  
    NewEmitter->SetFlags(RF_Transactional);  
    for (int32 LODIndex = 0; LODIndex < NewEmitter->LODLevels.Num(); LODIndex++)  
    {  
        UParticleLODLevel* NewEmitterLODLevel = NewEmitter->GetLODLevel(LODIndex);  
        if (NewEmitterLODLevel)  
        {  
            NewEmitterLODLevel->SetFlags(RF_Transactional);  
            check(NewEmitterLODLevel->RequiredModule);  
            NewEmitterLODLevel->RequiredModule->SetTransactionFlag();  
            check(NewEmitterLODLevel->SpawnModule);  
            NewEmitterLODLevel->SpawnModule->SetTransactionFlag();  
            for (int32 jj = 0; jj < NewEmitterLODLevel->Modules.Num(); jj++)  
            {  
                UParticleModule* pkModule = NewEmitterLODLevel->Modules[jj];  
                pkModule->SetTransactionFlag();  
            }  
        }  
    }  
​  
    // Add to selected emitter  
    // 把新创建的Emitter 添加到数组里  
    ParticleSystem->Emitters.Add(NewEmitter);  
​  
    // Setup the LOD distances  
    // 设置LOD距离，每一层LODLevel 对应距离为2500  
    if (ParticleSystem->LODDistances.Num() == 0)  
    {  
        UParticleEmitter* Emitter = ParticleSystem->Emitters[0];  
        if (Emitter)  
        {  
            ParticleSystem->LODDistances.AddUninitialized(Emitter->LODLevels.Num());  
            for (int32 LODIndex = 0; LODIndex < ParticleSystem->LODDistances.Num(); LODIndex++)  
            {  
                ParticleSystem->LODDistances[LODIndex] = LODIndex * 2500.0f;  
            }  
        }  
    }  
    if (ParticleSystem->LODSettings.Num() == 0)  
    {  
        UParticleEmitter* Emitter = ParticleSystem->Emitters[0];  
        if (Emitter)  
        {  
            ParticleSystem->LODSettings.AddUninitialized(Emitter->LODLevels.Num());  
            for (int32 LODIndex = 0; LODIndex < ParticleSystem->LODSettings.Num(); LODIndex++)  
            {  
                ParticleSystem->LODSettings[LODIndex] = FParticleSystemLOD::CreateParticleSystemLOD();  
            }  
        }  
    }  
​  
    // 结束改变  
    ParticleSystemComponent->PostEditChange();  
    ParticleSystem->PostEditChange();  
​  
    ParticleSystem->SetupSoloing();  
    // https://zhuanlan.zhihu.com/p/606722605  
    // 这个的作用是 Undo / Redo  
    EndTransaction( Transaction );  
​  
    if (FEngineAnalytics::IsAvailable())  
    {  
        FEngineAnalytics::GetProvider().RecordEvent(TEXT("Editor.Usage.Cascade.NewEmitter"));  
    }  
​  
    // Refresh viewport  
    if (EmitterCanvas.IsValid())  
    {  
        EmitterCanvas->RefreshViewport();  
    }  
}
```

这个方法执行完之后就会创建出一个默认的Emitters，如图
![[image-20240305171609637.png]]
其中Required和Spawn都是在UParticleEmitter::CreateLODLevel()方法里创建Emitter必须要有的

剩下的四个模块都是在ParticleComponents.cpp的UParticleSpriteEmitter::SetToSensibleDefaults()方法里创建的。

### 2 创建新的模块 OnNewModule

```c++
// 传进来的参数是，我们想添加的module，在缓存数组中的索引是多少  
void FCascade::OnNewModule(int32 Idx)  
{  
    if (SelectedEmitter == NULL)  
    {  
        return;  
    }  
​  
    // 必须在LOD == 0 的时候创建Module，也就是最低等级的LOD才可以创建添加Module  
    int32 CurrLODLevel = GetCurrentlySelectedLODLevelIndex();  
    if (CurrLODLevel != 0)  
    {  
        // Don't allow creating modules if not at highest LOD  
        FMessageDialog::Open(EAppMsgType::Ok, NSLOCTEXT("Cascade", "CascadeLODAddError", "Must be on lowest LOD (0) to create modules"));  
        return;  
    }  
​  
    // 根据索引和缓存数组取出了需要创建module的class  
    UClass* NewModClass = ParticleModuleClasses[Idx];  
    check(NewModClass->IsChildOf(UParticleModule::StaticClass()));  
​  
    bool bIsEventGenerator = false;  
​  
    // 如果在最低等级LOD的ParticleLODLevel里面存在了TypeData Module，就不能在添加这个Module了  
    if (NewModClass->IsChildOf(UParticleModuleTypeDataBase::StaticClass()))  
    {  
        // Make sure there isn't already a TypeData module applied!  
        UParticleLODLevel* LODLevel = SelectedEmitter->GetLODLevel(0);  
        if (LODLevel->TypeDataModule != 0)  
        {  
            FMessageDialog::Open(EAppMsgType::Ok, NSLOCTEXT("UnrealEd", "Error_TypeDataModuleAlreadyPresent", "A TypeData module is already present.\nPlease remove it first."));  
            return;  
        }  
    }  
    else if (NewModClass == UParticleModuleEventGenerator::StaticClass())  
    {  
        // 这个EventGenerator不能在GPU粒子类型的里面添加  
        // 如果在最低等级LOD的ParticleLODLevel里面存在了EventGenerator Module，就不能在添加这个Module了  
        bIsEventGenerator = true;  
        // Make sure there isn't already an EventGenerator module applied!  
        UParticleLODLevel* LODLevel = SelectedEmitter->GetLODLevel(0);  
        if (LODLevel->EventGenerator != NULL)  
        {  
            FMessageDialog::Open(EAppMsgType::Ok, NSLOCTEXT("UnrealEd", "Error_EventGeneratorModuleAlreadyPresent", "An EventGenerator module is already present.\nPlease remove it first."));  
            return;  
        }  
    }  
    else if (NewModClass == UParticleModuleParameterDynamic::StaticClass())  
    {  
        // 如果在最低等级LOD的已经使用了ParameterDynamic，就不能在添加这个Module了  
        // Make sure there isn't already an DynamicParameter module applied!  
        UParticleLODLevel* LODLevel = SelectedEmitter->GetLODLevel(0);  
        for (int32 CheckMod = 0; CheckMod < LODLevel->Modules.Num(); CheckMod++)  
        {  
            UParticleModuleParameterDynamic* DynamicParamMod = Cast<UParticleModuleParameterDynamic>(LODLevel-  >Modules[CheckMod]);  
            if (DynamicParamMod)  
            {  
                FMessageDialog::Open(EAppMsgType::Ok, NSLOCTEXT("UnrealEd", "Error_DynamicParameterModuleAlreadyPresent", "A DynamicParameter module is already present.\nPlease remove it first."));  
                return;  
            }  
        }  
    }  
​  
    // https://zhuanlan.zhihu.com/p/606722605  
    // 这个的作用是 Undo / Redo  
    FText Transaction = NSLOCTEXT("UnrealEd", "CreateNewModule", "Create New Module");  
    BeginTransaction( Transaction );  
    ModifyParticleSystem();  
    ModifySelectedObjects();  
    if( NewModClass == UParticleModuleTypeDataMesh::StaticClass() )  
    {  
        // TypeDataMeshes update the LOD levels RequiredModule material so mark it for transaction  
        UParticleLODLevel* LODLevel = SelectedEmitter->GetLODLevel( 0 );  
        if( LODLevel->RequiredModule )  
        {  
            LODLevel->RequiredModule->Modify();  
        }  
    }  
    ParticleSystem->PreEditChange(NULL);  
    ParticleSystemComponent->PreEditChange(NULL);  
​  
    // Construct it and add to selected emitter.  
    // 创建新的Module，设置颜色，初始值  
    UParticleModule* NewModule = NewObject<UParticleModule>(ParticleSystem, NewModClass, NAME_None, RF_Transactional);  
    NewModule->ModuleEditorColor = FColor::MakeRandomColor();  
    NewModule->SetToSensibleDefaults(SelectedEmitter);  
    NewModule->LODValidity = 1;  
    NewModule->SetTransactionFlag();  
​  
    // 将新的Module添加到Modules数组里面  
    UParticleLODLevel* LODLevel = SelectedEmitter->GetLODLevel(0);  
    if (bIsEventGenerator == true)  
    {  
        LODLevel->Modules.Insert(NewModule, 0);  
        LODLevel->EventGenerator = Cast<UParticleModuleEventGenerator>(NewModule);  
    }  
    else  
    {  
        LODLevel->Modules.Add(NewModule);  
    }  
​  
    // 将模块也添加到所有LODLvel的Modules数组里面  
    for (int32 LODIndex = 1; LODIndex < SelectedEmitter->LODLevels.Num(); LODIndex++)  
    {  
        LODLevel = SelectedEmitter->GetLODLevel(LODIndex);  
        // 这个数据是为了标志NewModule这个模块，对哪些LOD有效，这边用了位与的方式存储  
        NewModule->LODValidity |= (1 << LODIndex);  
        if (bIsEventGenerator == true)  
        {  
            LODLevel->Modules.Insert(NewModule, 0);  
            LODLevel->EventGenerator = Cast<UParticleModuleEventGenerator>(NewModule);  
        }  
        else  
        {  
            LODLevel->Modules.Add(NewModule);  
        }  
    }  
​  
    // 更新模块列表  
    SelectedEmitter->UpdateModuleLists();  
​  
    // 各种结束操作  
    ParticleSystemComponent->PostEditChange();  
    ParticleSystem->PostEditChange();  
    EndTransaction( Transaction );  
    if (FEngineAnalytics::IsAvailable())  
    {  
        FEngineAnalytics::GetProvider().RecordEvent(TEXT("Editor.Usage.Cascade.NewModule"), TEXT("Class"), NewModClass->GetName());  
    }  
    ParticleSystem->MarkPackageDirty();  
    // Refresh viewport  
    if (EmitterCanvas.IsValid())  
    {  
        EmitterCanvas->RefreshViewport();  
    }  
}
```

### 3 初始化Cascade

```c++
// 传进来的参数  
void FCascade::InitCascade(const EToolkitMode::Type Mode, const TSharedPtr< class IToolkitHost >& InitToolkitHost, UObject* ObjectToEdit)  
{  
    // 拿到这个Cascade的数据资源UParticleSystem  
    ParticleSystem = CastChecked<UParticleSystem>(ObjectToEdit);  
​  
    // 设置LOD部分，遍历Emitter中的LODLevel中的Modules，设置每一个Module的LODValidity  
    ParticleSystem->EditorLODSetting = 0;  
    ParticleSystem->SetupLODValidity();  
​  
    // Support undo/redo  
    ParticleSystem->SetFlags(RF_Transactional);  
​  
    // 当前在Cascade界面选择的LODIdx  
    CurrentLODIdx = 0;  
​  
    // 创建一些配置信息  
    EditorOptions = NewObject<UCascadeOptions>();  
    check(EditorOptions);  
    EditorConfig = NewObject<UCascadeConfiguration>();  
    check(EditorConfig);  
​  
    // 下面这段乱七八糟的，看上去也是支持 undo/redo的操作吧，然后记录打印?  
    FString Description;  
    for (int32 EmitterIdx = 0; EmitterIdx < ParticleSystem->Emitters.Num(); EmitterIdx++)  
    {  
        UParticleEmitter* Emitter = ParticleSystem->Emitters[EmitterIdx];  
        if (Emitter)  
        {  
            Description += FString::Printf(TEXT("Emitter%d["), EmitterIdx);  
            Emitter->SetFlags(RF_Transactional);  
            for (int32 LODIndex = 0; LODIndex < Emitter->LODLevels.Num(); LODIndex++)  
            {  
                UParticleLODLevel* LODLevel = Emitter->GetLODLevel(LODIndex);  
                if (LODLevel)  
                {  
                    Description += FString::Printf(TEXT("LOD%d("), LODIndex);  
                    LODLevel->SetFlags(RF_Transactional);  
                    check(LODLevel->RequiredModule);  
                    LODLevel->RequiredModule->SetTransactionFlag();  
                    check(LODLevel->SpawnModule);  
                    LODLevel->SpawnModule->SetTransactionFlag();  
                    if (LODLevel->Modules.Num() > 0)  
                    {  
                        Description += FString::Printf(TEXT("Modules%d"), LODLevel->Modules.Num());  
                        for (int32 ModuleIdx = 0; ModuleIdx < LODLevel->Modules.Num(); ModuleIdx++)  
                        {  
                            UParticleModule* pkModule = LODLevel->Modules[ModuleIdx];  
                            pkModule->SetTransactionFlag();  
                        }  
                    }  
                    Description += TEXT(")");  
                    if (Emitter->LODLevels.Num() > LODIndex + 1)  
                    {  
                        Description += TEXT(",");  
                    }  
                }  
            }  
            Description += TEXT("]");  
            if (ParticleSystem->Emitters.Num() > EmitterIdx + 1)  
            {  
                Description += TEXT(",");  
            }  
        }  
    }  
    if (Description.Len() > 0 && FEngineAnalytics::IsAvailable())  
    {  
        FEngineAnalytics::GetProvider().RecordEvent(TEXT("Editor.Usage.Cascade.Init"), TEXT("Overview"), Description);  
    }  
​  
    // 创建我们的CascadeParticleSystemComponent，用于播放特效  
    ParticleSystemComponent = NewObject<UCascadeParticleSystemComponent>();  
​  
    // 初始化数据  
    LocalVectorFieldPreviewComponent = NewObject<UVectorFieldComponent>();  
    bIsSoloing = false;  
    bTransactionInProgress = false;  
    SetSelectedModule(NULL, NULL);  
    CopyModule = NULL;  
    CopyEmitter = NULL;  
    CurveToReplace = NULL;  
    DetailMode = GlobalDetailMode = GetCachedScalabilityCVars().DetailMode;  
    RequiredSignificance = EParticleSignificanceLevel::Low;  
    bIsToggleMotion = false;  
    MotionModeRadius = EditorOptions->MotionModeRadius;  
    AccumulatedMotionTime = 0.0f;  
    TimeScale = CachedTimeScale = 1.0f;  
    bIsToggleLoopSystem = true;  
    bIsPendingReset = false;  
    ResetTime = BIG_NUMBER;  
    TotalTime = 0.0;  
    ParticleMemoryUpdateTime = 5.0f;  
    bParticleModuleClassesInitialized = false;  
​  
    // 把我们所有的Particle相关的UClass记录到两个数组里面ParticleModuleClasses,ParticleModuleBaseClasses  
    InitParticleModuleClasses();  
​  
    // Create a new emitter if the particle system is empty...  
    // 如果数据里面没有Emitter，就创建一个，这个上面说过了  
    if (ParticleSystem->Emitters.Num() == 0)  
    {  
        OnNewEmitter();  
    }  
​  
    // 下面就都是编辑器和UI的操作了， 全都省略了  
    // FAssetEditorToolkit::InitAssetEditor 通过这个来初始化界面的布局，Viewport， Emitters，Details，CurveEditor这四个  
    GEditor->RegisterForUndo(this);  
      
}
```

### 4 打开Cascade界面流程

```c++
void FAssetTypeActions_ParticleSystem::OpenAssetEditor(...)  
 -> TSharedRef<ICascade> CreateCascade(...) // 这个方法是在CascadeModule.cpp里的FCascadeModule类中方法，会把创建出的Cascade存到  CascadeToolkits数组里，就像是打开了很多的Cascade界面，都由CascadeModule统一管理  
  -> void FCascade::InitCascade(...)// 这个里面创建UCascadeParticleSystemComponent，初始化数据  
   -> void OnNewEmitter() // 创建一个Emitter  
   -> // 一些编辑器更新的方法
```

### 5 Viewport窗口中的粒子生成

这个部分我们先看编辑器中Viewport这个窗口里的如何生成的

```c++
// 位置是在SpawnTab这个地方，我们只看Viewport这个tab，其余的都省略了  
TSharedRef<SDockTab> FCascade::SpawnTab(const FSpawnTabArgs& SpawnTabArgs, FName TabIdentifier)  
{  
    if (TabIdentifier == FName(TEXT("Cascade_PreviewViewport")))  
    {  
        TSharedRef<SDockTab> SpawnedTab = SNew(SDockTab)  
            .Label( NSLOCTEXT("Cascade", "CascadeViewportTitle", "Viewport"))  
            [  
                PreviewViewport.ToSharedRef()  
            ];  
​  
        PreviewViewport->ParentTab = SpawnedTab;  
​  
        // Set emitter to use the particle system we are editing.  
        ParticleSystemComponent->SetTemplate(ParticleSystem);  
​  
        // 这个就是初始化做一些初始化的工作,后面详细看  
        ParticleSystemComponent->InitializeSystem();  
        // 这个激活  
        //   
        ParticleSystemComponent->ActivateSystem();  
​  
        // Set camera position/orientation  
        PreviewViewport->GetViewportClient()->SetPreviewCamera(ParticleSystem->ThumbnailAngle, ParticleSystem-  >ThumbnailDistance);  
        PreviewViewport->GetViewportClient()->SetViewLocationForOrbiting(FVector::ZeroVector);  
​  
        return SpawnedTab;  
    }  
}

void UParticleSystemComponent::InitializeSystem()  
{  
    // 一些多线程的设置吧,和一些一些提前检查的判断  
    SCOPE_CYCLE_COUNTER(STAT_ParticleInitializeTime);  
    ForceAsyncWorkCompletion(STALL);  
​  
    // System settings may have been lowered. Support late deactivation.  
    const bool bDetailModeAllowsRendering = DetailMode <= GetCurrentDetailMode();  
​  
    // 是否允许创建粒子，判定  
    if( GIsAllowingParticles && bDetailModeAllowsRendering )  
    {  
        if (IsTemplate() == true)  
        {  
            return;  
        }  
​  
        // 都是一些设置啥的吧，看不懂具体干什么的   
        if( Template != NULL )  
        {  
            EmitterDelay = Template->Delay;  
​  
            if( Template->bUseDelayRange )  
            {  
                const float Rand = RandomStream.FRand();  
                EmitterDelay     = Template->DelayLow + ((Template->Delay - Template->DelayLow) * Rand);  
            }  
        }  
​  
        // Allocate the emitter instances and particle data  
        // 最终要的就是这个InitParticles方法  
        InitParticles();  
        if (IsRegistered())  
        {  
            AccumTickTime = 0.0;  
            if ((IsActive() == false) && (bAutoActivate == true) && (bWasDeactivated == false))  
            {  
                SetActive(true);  
            }  
        }  
    }  
}  
// 这个里面乱七八糟一堆，主要就是创建EmitterInstance，然后设置一些参数  
void UParticleSystemComponent::InitParticles()  
{  
    // 一些提前判断，多线程啥的吧  
    LLM_SCOPE(ELLMTag::Particles);  
    SCOPE_CYCLE_COUNTER(STAT_ParticleSystemComponent_InitParticles);  
    if (IsTemplate() == true)  
    {  
        return;  
    }  
    ForceAsyncWorkCompletion(ENSURE_AND_STALL);  
    check(GetWorld());  
    UE_LOG(LogParticles,Verbose,  
        TEXT("InitParticles @ %fs %s"), GetWorld()->TimeSeconds,  
        Template != NULL ? *Template->GetName() : TEXT("NULL"));  
​  
    if (Template != NULL)  
    {  
        // 设置一些状态  
        WarmupTime = Template->WarmupTime;  
        WarmupTickRate = Template->WarmupTickRate;  
        bIsViewRelevanceDirty = true;  
        const int32 GlobalDetailMode = GetCurrentDetailMode();  
        const bool bCanEverRender = CanEverRender();  
​  
        //simplified version.  
        int32 NumInstances = EmitterInstances.Num();  
        int32 NumEmitters = Template->Emitters.Num();  
        const bool bIsFirstCreate = NumInstances == 0;  
        EmitterInstances.SetNumZeroed(NumEmitters);  
​  
        bWasCompleted = bIsFirstCreate ? false : bWasCompleted;  
​  
        bool bClearDynamicData = false;  
        int32 PreferredLODLevel = LODLevel;  
        bool bSetLodLevels = LODLevel > 0;  
​  
        // 遍历所有的Emitter，看是否创建EmitterInstance  
        for (int32 Idx = 0; Idx < NumEmitters; Idx++)  
        {  
            UParticleEmitter* Emitter = Template->Emitters[Idx];  
            if (Emitter)  
            {  
                FParticleEmitterInstance* Instance = NumInstances == 0 ? NULL : EmitterInstances[Idx];  
                check(GlobalDetailMode < EParticleDetailMode::PDM_MAX);  
                const bool bDetailModeAllowsRendering = DetailMode <= GlobalDetailMode && (Emitter->DetailModeBitmask & (1 << GlobalDetailMode));  
                const bool bShouldCreateAndOrInit = bDetailModeAllowsRendering && Emitter->HasAnyEnabledLODs() && bCanEverRender;  
​  
                if (bShouldCreateAndOrInit)  
                {  
                    if (Instance)  
                    {  
                        Instance->SetHaltSpawning(false);  
                        Instance->SetHaltSpawningExternal(false);  
                    }  
                    else  
                    {  
                        // 这个就是创建EmitterInstance，如果没有实例，并且还需要创建  
                        Instance = Emitter->CreateInstance(this);  
                        // 天骄到实例数组里面  
                        EmitterInstances[Idx] = Instance;  
                    }  
​  
                    if (Instance)  
                    {  
                        Instance->bEnabled = true;  
                        // 设置一些参数, SpriteTemplate, Component 
                        // 1 设置了SpriteTemplate
                        // 2 设置了Component
                        // 3 设置Duration, delay的值
                        Instance->InitParameters(Emitter, this);  
                        // 初始化数据啥的吧，里面有个InstanceData数组，里面存储了EmitterInstance所需的所有ParticleModuleInstance的数据  
                        // 通过偏移拿到具体ParticleModuleInstance的数据  
                        Instance->Init();  
​  
                        PreferredLODLevel = FMath::Min(PreferredLODLevel, Emitter->LODLevels.Num());  
                        bSetLodLevels |= !bIsFirstCreate;  
                    }  
                }  
                else  
                {  
                    // 不需要创建粒子，粒子还存在，就销毁掉  
                    if (Instance)  
                    {  
#if STATS  
                        Instance->PreDestructorCall();  
#endif  
​  
                        // AZURE, fix bug will cause crash!!!  
                        // clean up other instances that may point to this one  
                        for (int32 InnerIndex = 0; InnerIndex < EmitterInstances.Num(); InnerIndex++)  
                        {  
                            if (InnerIndex != Idx && EmitterInstances[InnerIndex] != NULL)  
                            {  
                                EmitterInstances[InnerIndex]->OnEmitterInstanceKilled(Instance);  
                            }  
                        }  
​  
                        delete Instance;  
                        EmitterInstances[Idx] = NULL;  
                        bClearDynamicData = true;  
                    }  
                }  
            }  
        }  
​  
        if (bClearDynamicData)  
        {  
            ClearDynamicData();  
        }  
​  
        // 这部分就是初始化的时候设置LOD了，在创建的时候LOD不为0，就给EmitterInstance设置上  
        if (bSetLodLevels)  
        {  
            if (PreferredLODLevel != LODLevel)  
            {  
                // This should never be higher...  
                check(PreferredLODLevel < LODLevel);  
                LODLevel = PreferredLODLevel;  
            }  
​  
            for (int32 Idx = 0; Idx < EmitterInstances.Num(); Idx++)  
            {  
                FParticleEmitterInstance* Instance = EmitterInstances[Idx];  
                // set the LOD levels here  
                if (Instance)  
                {  
                    Instance->CurrentLODLevelIndex  = LODLevel;  
​  
                    if (Instance->CurrentLODLevelIndex >= Instance->SpriteTemplate->LODLevels.Num())  
                    {  
                        Instance->CurrentLODLevelIndex = Instance->SpriteTemplate->LODLevels.Num()-1;  
                    }  
                    Instance->CurrentLODLevel = Instance->SpriteTemplate->LODLevels[Instance->CurrentLODLevelIndex];  
                }  
            }  
        }  
    }  
}  
void UParticleSystemComponent::ActivateSystem(...)  
{  
    // 激活的代码很多，写一部分吧  
      
    // 这个就是从未激活到激活的时候走进去，  
    // 设置Component的重要性等级，只有Emitter的SignificanceLevel >= Component的重要性等级才会生成粒子  
    // 广播了一个事件，最终会在UParticleSystemComponent的SetManagingSignificance方法响应，设置开启重要性等级的检测  
    if (!IsActive())  
    {  
        LastSignificantTime = World->GetTimeSeconds();  
        RequiredSignificance = EParticleSignificanceLevel::Low;  
​  
        //Call this now after any attachment has happened.  
        OnSystemPreActivationChange.Broadcast(this, true);  
    }  
}  
​
```
### 6 Viewport窗口中的粒子Tick

```c++
void UParticleSystemComponent::TickComponent(...)  
    ->void ComputeTickComponent_Concurrent()  
     ->void FParticleEmitterInstance::Tick(float DeltaTime, bool bSuppressSpawning)  
      ->float Tick_SpawnParticles(...)  
       ->float Spawn(float DeltaTime)  
        ->void SpawnParticles(...)  
         ->void PreSpawn(...)  
         ->void UParticleModule::Spawn(..)//这是个虚函数  
         ->void PostSpawn(...)
```

tick的开始就是在TickComponent之中，里面乱七八糟执行一堆东西，看不太懂，总之会执行到ComputeTickComponent_Concurrent方法来对整个粒子系统进行tick

```c++
void UParticleSystemComponent::ComputeTickComponent_Concurrent()  
{  
    // 开始这段可能就是一些性能指标之类的标志位，不关键  
    FInGameScopedCycleCounter InGameCycleCounter(GetWorld(), EInGamePerfTrackers::VFXSignificance, IsInGameThread() ? EInGamePerfTrackerThreads::GameThread : EInGamePerfTrackerThreads::OtherThread, bIsManagingSignificance);  
    SCOPE_CYCLE_COUNTER(STAT_ParticleComputeTickTime);  
    FScopeCycleCounterUObject AdditionalScope(AdditionalStatObject(), GET_STATID(STAT_ParticleComputeTickTime));  
    SCOPE_CYCLE_COUNTER(STAT_ParticlesOverview_GT_CNC);  
    PARTICLE_PERF_STAT_CYCLES_GT(FParticlePerfStatsContext(GetWorld(), Template, this), TickConcurrent);  
​  
    // Tick Subemitters.  
    // 下面这个就是遍历所有的EmitterInstances  
    int32 EmitterIndex;  
    NumSignificantEmitters = 0;  
    for (EmitterIndex = 0; EmitterIndex < EmitterInstances.Num(); EmitterIndex++)  
    {  
        FParticleEmitterInstance* Instance = EmitterInstances[EmitterIndex];  
        FScopeCycleCounterEmitter AdditionalScopeInner(Instance);  
#if WITH_EDITOR  
        uint32 StartTime = FPlatformTime::Cycles();  
#endif  
        // 应该是对下一个EmitterInstance进行预加载之类的操作  
        if (EmitterIndex + 1 < EmitterInstances.Num())  
        {  
            FParticleEmitterInstance* NextInstance = EmitterInstances[EmitterIndex+1];  
            FPlatformMisc::Prefetch(NextInstance);  
        }  
​  
        // 判断EmitterInstance中的UParticleEmitter是否存在，这个UParticleEmitter就是创建实例时候赋值的  
        if (Instance && Instance->SpriteTemplate)  
        {  
            // 检测这个UParticleEmitter里面真的有LODLevel，也就是真正的Emitter数据  
            check(Instance->SpriteTemplate->LODLevels.Num() > 0);  
​  
            UParticleLODLevel* SpriteLODLevel = Instance->SpriteTemplate->GetCurrentLODLevel(Instance);  
            if (SpriteLODLevel && SpriteLODLevel->bEnabled)  
            {  
                // 是否开启了重要性管理，这个标志位是通过SetManagingSignificance()方法设置为true的  
                // 在ParticleSystemComponent->ActivateSystem()激活中通过广播事件  
                // 然后在通过FCascade::OnComponentActivationChange()方法调用进来的  
                if (bIsManagingSignificance)  
                {  
                    // 这个就是判断粒子的重要性等级是否>=Component的SignificantLevel，如果满足才会生成粒子  
                    bool bEmitterIsSignificant = Instance->SpriteTemplate->IsSignificant(RequiredSignificance);  
                    if (bEmitterIsSignificant)  
                    {  
                        ++NumSignificantEmitters;  
                        Instance->SetHaltSpawning(false);  
                        Instance->SetFakeBurstWhenSpawningSupressed(false);  
                        Instance->bEnabled = true;  
                    }  
                    else  
                    {  
                        Instance->SetHaltSpawning(true);  
                        Instance->SetFakeBurstWhenSpawningSupressed(true);  
                        if (Instance->SpriteTemplate->bDisableWhenInsignficant)  
                        {  
                            Instance->bEnabled = false;  
                        }  
                    }  
                }  
                else  
                {  
                    ++NumSignificantEmitters;  
                }  
​  
                // 开始每个EmitterInstance的Tick  
                Instance->Tick(DeltaTimeTick, bSuppressSpawning);  
                // 这个方法是OverrideMatrial  
                // 我们可以在ParticleSystem里面设置Matrials slot数组，是一个键值对,key是name,value是Matrial  
                // 如果我们在每个Emitter中的Required模块上配置了NamedMaterialOverrrides，并且和上面的键值对的key对应  
                // 则会重写CurrMaterial，为Slot中配置的值，  
                // 但如果Component中的EmitterMaterials这个配置的话，就采用Component的  
                Instance->Tick_MaterialOverrides(EmitterIndex);  
                // 记录一下所有激活的粒子数量  
                TotalActiveParticles += Instance->ActiveParticles;  
            }  
​  
#if WITH_EDITOR  
            uint32 EndTime = FPlatformTime::Cycles();  
            Instance->LastTickDurationMs += FPlatformTime::ToMilliseconds(EndTime - StartTime);  
#endif  
        }  
    }  
    if (bAsyncWorkOutstanding)  
    {  
        FPlatformMisc::MemoryBarrier();  
        bAsyncWorkOutstanding = false;  
    }  
}
```

```c++
// 下面这个就是Tick实例,UE的注释很清晰了  
void FParticleEmitterInstance::Tick(float DeltaTime, bool bSuppressSpawning)  
{  
    SCOPE_CYCLE_COUNTER(STAT_SpriteTickTime);  
​  
    check(SpriteTemplate);  
    check(SpriteTemplate->LODLevels.Num() > 0);  
​  
    // If this the FirstTime we are being ticked?  
    bool bFirstTime = (SecondsSinceCreation > 0.0f) ? false : true;  
​  
    // Grab the current LOD level  
    UParticleLODLevel* LODLevel = GetCurrentLODLevelChecked();  
​  
    // Handle EmitterTime setup, looping, etc.  
    // 这个是个算各种时间的方法  
    float EmitterDelay = Tick_EmitterTimeSetup(DeltaTime, LODLevel);  
​  
    if (bEnabled)  
    {  
        // Kill off any dead particles  
        // 判断粒子是否激活，然后销毁掉  
        // 毁掉的方式看上去就是在ParticleIndices数组中删掉对应索引  
        KillParticles();  
​  
        // Reset particle parameters.  
        // 重新设置了轨道的值，看上去就是初始出现的位置速度啥的，以及更新粒子的生存时间  
        ResetParticleParameters(DeltaTime);  
​  
        // Update the particles  
        CurrentMaterial = LODLevel->RequiredModule->Material;  
        Tick_ModuleUpdate(DeltaTime, LODLevel);  
​  
        // Spawn new particles.  
        // 生成这边就是先判断需要生成的数量，然后开始生成  
        // 有preSpawn Spawn postSpawn方法  
        // preSpawn就是先生成一基础数据，位置，速度啥的  
        // Spawn就是遍历所有的module，调用具体的module里面的Spawn方法，更新数据  
        //  
        SpawnFraction = Tick_SpawnParticles(DeltaTime, LODLevel, bSuppressSpawning, bFirstTime);  
​  
        // PostUpdate (beams only)  
        Tick_ModulePostUpdate(DeltaTime, LODLevel);  
​  
        if (ActiveParticles > 0)  
        {  
            // Update the orbit data...  
            UpdateOrbitData(DeltaTime);  
            // Calculate bounding box and simulate velocity.  
            UpdateBoundingBox(DeltaTime);  
        }  
​  
        Tick_ModuleFinalUpdate(DeltaTime, LODLevel);  
​  
        CheckEmitterFinished();  
​  
        // Invalidate the contents of the vertex/index buffer.  
        IsRenderDataDirty = 1;  
    }  
    else  
    {  
        FakeBursts();  
    }  
​  
    // 'Reset' the emitter time so that the delay functions correctly  
    EmitterTime += EmitterDelay;  
​  
    // Store the last delta time.  
    LastDeltaTime = DeltaTime;  
​  
    // Reset particles position offset  
    PositionOffsetThisTick = FVector::ZeroVector;  
​  
    INC_DWORD_STAT_BY(STAT_SpriteParticles, ActiveParticles);  
}  
```
