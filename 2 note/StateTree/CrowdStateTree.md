# 
```cpp
enum class EStateTreeStateSelectionBehavior : uint8
{
	/** The State cannot be directly selected. */
	None,
	
	// 直接进入这个节点，不管有没有子节点
	TryEnterState UMETA(DisplayName = "Try Enter"),
	// 按找子节点的顺序挑选进入
	TrySelectChildrenInOrder UMETA(DisplayName = "Try Select Children In Order"),
	// 将子节点随机选一个
	TrySelectChildrenAtRandom UMETA(DisplayName = "Try Select Children At Random"),
	// 从子节点中选择Utility计算出最高的
	TrySelectChildrenWithHighestUtility UMETA(DisplayName = "Try Select Children With Highest Utility"),
	// 根据效用分数加权随机选择子状态
	TrySelectChildrenAtRandomWeightedByUtility UMETA(DisplayName = "Try Select Children At Random Weighted By Utility"),

	// 触发zhuang
	TryFollowTransitions UMETA(DisplayName = "Try Follow Transitions"),

	// Olds names that needs to be kept forever to ensure asset serialization to work correctly when UENUM() switched from serializing int to names.
	TrySelectChildrenAtUniformRandom UE_DEPRECATED(5.5, "Use TrySelectChildrenAtRandom Instead") = TrySelectChildrenAtRandom UMETA(Hidden),
	TrySelectChildrenBasedOnRelativeUtility UE_DEPRECATED(5.5, "Use TrySelectChildrenAtRandomWeightedByUtility Instead") = TrySelectChildrenAtRandomWeightedByUtility UMETA(Hidden)
};
```

```cpp
bool FStateTreeExecutionContext::SelectStateInternal(
	const FStateTreeExecutionFrame* CurrentParentFrame,
	FStateTreeExecutionFrame& CurrentFrame,
	const FStateTreeExecutionFrame* CurrentFrameInActiveFrames,
	TConstArrayView<FStateTreeStateHandle> PathToNextState,
	FStateSelectionResult& OutSelectionResult,
	const FStateTreeSharedEvent* TransitionEvent)
{
	
}
```
