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
}
```