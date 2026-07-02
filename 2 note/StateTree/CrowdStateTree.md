# 
```cpp
enum class EStateTreeStateSelectionBehavior : uint8
{
	/** The State cannot be directly selected. */
	None,
	
	// 直接进入这个节点，不管有没有子节点
	/** When state is considered for selection, it is selected even if it has child states. */
	TryEnterState UMETA(DisplayName = "Try Enter"),
	// 按找子节点的顺序挑选进入
	/** When state is considered for selection, try to select the first child state (in order they appear in the child list). If no child states are present, behaves like SelectState. */
	TrySelectChildrenInOrder UMETA(DisplayName = "Try Select Children In Order"),
	// 将子节点打散，挑第一个进入
	/** When state is considered for selection, shuffle the order of child states and try to select the first one. If no child states are present, behaves like SelectState. */
	TrySelectChildrenAtRandom UMETA(DisplayName = "Try Select Children At Random"),
	
	/** When state is considered for selection, try to select the child state with highest utility score. If there is a tie, it will try to select in order. */
	TrySelectChildrenWithHighestUtility UMETA(DisplayName = "Try Select Children With Highest Utility"),

	/** When state is considered for selection, randomly pick one of its child states. The probability of selecting each child state is its normalized utility score */
	TrySelectChildrenAtRandomWeightedByUtility UMETA(DisplayName = "Try Select Children At Random Weighted By Utility"),

	/** When state is considered for selection, try to trigger the transitions instead. */
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
