0
```cpp
1 计算AnimStateID
```

1 MassActor通过这个来更新AnimInstanceData数据，然后动画蓝图读取更新
2 FCrowdAnimationFragment 
```cpp
struct CITYSAMPLEMASSCROWD_API FCrowdAnimationFragment : public FMassFragment
{
	GENERATED_BODY()

	UPROPERTY()
	TObjectPtr<UAnimToTextureDataAsset> AnimToTextureData = nullptr;// 在UCitySampleCrowdVisualizationFragmentInitializer这个ObserverProcessor中赋值的，直接取MassCharacter中配置的头资源

	float GlobalStartTime = 0.0f;//动画开始播放的时间，（全局累计时间 - 动画开始播放的时间）* 动画播放速率 = 动画播放的位置。
	float PlayRate = 1.0f;//动画播放速率
	int32 AnimationStateIndex = 0;//动画状态索引，ECrowdAnimationType这个枚举的索引
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
3 填充FMassCrowdAnimInstanceData信息
```cpp
USTRUCT(BlueprintType)
struct FMassCrowdAnimInstanceData
{
	GENERATED_BODY()

	UPROPERTY(transient, EditAnywhere, BlueprintReadOnly, Category = MassCrowd)
	UAnimSequence* FarLODAnimSequence = nullptr;//取到FCrowdAnimationFragment中表示的动画序列
	UPROPERTY(transient, EditAnywhere, BlueprintReadOnly, Category = MassCrowd)
	FTransform MassEntityTransform;//MassEntity的位置
	UPROPERTY(transient, EditAnywhere, BlueprintReadOnly, Category = MassCrowd)
	FVector LookAtDirection = FVector::ForwardVector;// 读取Entity的FMassLookAtFragment中记录的LookAt方向，局部空间下
	UPROPERTY(transient, EditAnywhere, BlueprintReadOnly, Category = MassCrowd)
	float FarLODPlaybackStartTime = 0.0f;//FarLODAnimSequence动画序列，从哪个时间点开始播放
	UPROPERTY(transient, EditAnywhere, BlueprintReadOnly, Category = MassCrowd)
	float AnimSequenceLength = 0.0f;//FarLODAnimSequence动画序列的长度，就是播放多长时间就播放完了
	UPROPERTY(transient, EditAnywhere, BlueprintReadOnly, Category = MassCrowd)
	float Significance = 0.0f;//角色计算出的Lod等级
	UPROPERTY(transient, EditAnywhere, BlueprintReadOnly, Category = MassCrowd)
	bool bSwappedThisFrame = true;//这一帧是否要ISM和Actor之间切换
};
```
4 将FMassCrowdAnimInstanceData设置到UMassCrowdAnimInstance中，让动画蓝图读取

# UMassProcessor_CrowdVisualizationCustomData
```cpp
1 执行UMassUpdateISMProcessor::UpdateISMTransform这个方法，会将当前entityhandle，transform添加到SharedData数组中。一个ISMComp会有一个SharedData，SharedData里会记录这个ISMComp中不同Instance上的数据。
2 执行UMassCrowdUpdateISMVertexAnimationProcessor::UpdateISMVertexAnimation方法，将entity的动画数据（5个float类型的）添加到SharedData中的CustomData里。
3 执行WriteCustomDataFloatsAtStartIndex方法，指定将颜色数据添加到entity身上的头所在的SharedData中的CustomData里。
```