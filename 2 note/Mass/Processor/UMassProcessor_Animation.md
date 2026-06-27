
1 FCrowdAnimationFragment 
```cpp
struct CITYSAMPLEMASSCROWD_API FCrowdAnimationFragment : public FMassFragment
{
	GENERATED_BODY()

	UPROPERTY()
	TObjectPtr<UAnimToTextureDataAsset> AnimToTextureData = nullptr;

	float GlobalStartTime = 0.0f;//动画开始播放的时间，（全局累计时间 - 动画开始播放的时间）* 动画播放速率 = 动画播放的位置。为了动画播放速率
	float PlayRate = 1.0f;//动画播放速率
	int32 AnimationStateIndex = 0;
	bool bSwappedThisFrame = false;//在MassProcessor_Animation中计算，如果PrevRepresentation和CurrentRepresentation不同，有一个是Actor，有一个不是就设置为true，表示这一帧要切换ISM和Actor

	// 新增字段
	float OriginalPlayRate = 1.0f;    // 保存原始播放速率
	float PauseStartTime = 0.0f;      // 记录暂停开始时间
	bool bWasPaused = false;
	
	void SetAnimToTextureData(UAnimToTextureDataAsset* IntAnimToTextureData)
	{
		AnimToTextureData = IntAnimToTextureData;
		AnimToTextureDataGuard = MakeShared<FGCObjectScopeGuard>(AnimToTextureData);

	}
private:
	TSharedPtr<FGCObjectScopeGuard> AnimToTextureDataGuard;
};
```