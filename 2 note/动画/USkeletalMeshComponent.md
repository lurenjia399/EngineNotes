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
					AnimScriptInstance->NativeBeginPlay();
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
		if( LeaderPoseComponent.IsValid() )
		{
			UpdateFollowerComponent();
		}
		else 
		{
			RefreshBoneTransforms(ThisTickFunction);
		}
	}
}
```