# 初始化InitAnim

```cpp
void USkeletalMeshComponent::InitAnim(bool bForceReinit)
{
	if ( GetSkeletalMeshAsset() != nullptr && IsRegistered() )
	{
		const bool bInitializedAnimInstance = InitializeAnimScriptInstance(bForceReinit, !bTickAnimationNow);
		UpdateComponentToWorld();
	}
}
```

```cpp
bool USkeletalMeshComponent::InitializeAnimScriptInstance(bool bForceReinit, bool bInDeferRootNodeInitialization)
{
	if (IsRegistered())
	{
		USkeletalMesh* SkelMesh = GetSkeletalMeshAsset();
		if (NeedToSpawnAnimScriptInstance())
		{
			// 创建出我们的动画蓝图
			AnimScriptInstance = NewObject<UAnimInstance>(this, AnimClass);
			if (AnimScriptInstance)
			{
				ResetLinkedAnimInstances();
				// 初始化动画蓝图
				AnimScriptInstance->InitializeAnimation(bInDeferRootNodeInitialization);
				// 如果SkeletalMeshComponent已经BegunPlay了，这里也开始下动画蓝图
				if (HasBegunPlay())
				{
					// c++执行动画蓝图BeginPlay节点
					AnimScriptInstance->NativeBeginPlay();
					// 蓝图执行动画蓝图BeginPlay节点
					AnimScriptInstance->BlueprintBeginPlay();
				}
				bInitializedMainInstance = true;
			}
		}
	}
}
```

```cpp
void UAnimInstance::InitializeAnimation(bool bInDeferRootNodeInitialization)
{
	// 初始化FAnimInstanceProxy这个
	GetProxyOnGameThread<FAnimInstanceProxy>().Initialize(this);
	// C++执行节点
	NativeInitializeAnimation();
	// 蓝图执行节点
	BlueprintInitializeAnimation();
	// 初始化FAnimInstanceProxy这个
	GetProxyOnGameThread<FAnimInstanceProxy>().InitializeRootNode(bInDeferRootNodeInitialization);
	// 初始化FAnimInstanceProxy这个
	GetProxyOnGameThread<FAnimInstanceProxy>().BindNativeDelegates();
}
```
# TickComponent

``` cpp
void USkeletalMeshComponent::TickComponent(...)
{
	// 看上去啥也没有，执行父类的了
	Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
}
void USkinnedMeshComponent::TickComponent(...)
{
	if (ShouldTickPose())
	{
		TickPose(DeltaTime, false);
	}
	if( ShouldUpdateTransform(bLODHasChanged) )
	{
		// LeaderPoseComponent一般在换装系统中用，不同的SkelComp有相同的谷歌位置。
		if( LeaderPoseComponent.IsValid() )
		{
			UpdateFollowerComponent();
		}
		else 
		{
			// 主要是这里刷新Bone的Transform
			RefreshBoneTransforms(ThisTickFunction);
		}
	}
}
```
1 USkinnedMeshComponent的RefreshBoneTransforms方法会进行一次转发，会执行RefreshBoneTransforms_WithTeleport方法里面
```cpp
void USkeletalMeshComponent::RefreshBoneTransforms_WithTeleport(FActorComponentTickFunction* TickFunction, ETeleportType Teleport)
{
	DispatchParallelEvaluationTasks(TickFunction);
}
void USkeletalMeshComponent::DispatchParallelEvaluationTasks(FActorComponentTickFunction* TickFunction)
{
	
}
```