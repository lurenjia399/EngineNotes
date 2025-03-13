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