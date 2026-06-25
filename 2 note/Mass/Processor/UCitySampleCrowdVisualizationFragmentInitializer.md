1 是一个UMassObserverProcessor，观察的是FCitySampleCrowdVisualizationFragment添加
2 在execute里面，获取到高模actor身上的CrowdCharacterData配置，通过随机取出配置中的头，身体等信息，组装成FStaticMeshInstanceVisualizationDesc
3 然后将FStaticMeshInstanceVisualizationDesc添加到RepresentationSubSystem中，并返回Handle保存到FMassRepresentationFragment这个里
4 RepresentationSubSystem会转发到UMassVisualizationComponent里添加，最终会将Handle添加到InstancedSMComponentsRequiringConstructing数组中

5 InstancedSMComponentsRequiringConstructing数组的处理，是在BeginVisualChanges方法中的，是在ProcessingPhase开始tick的时候执行
```cpp
void UMassRepresentationSubsystem::OnProcessingPhaseStarted(const float DeltaSeconds, const EMassProcessingPhase Phase) const
{
	check(VisualizationComponent);
	switch (Phase)
	{
		case EMassProcessingPhase::PrePhysics:
			VisualizationComponent->BeginVisualChanges();
			break;
		case EMassProcessingPhase::PostPhysics:/* Currently this is the end of phases signal */
			VisualizationComponent->EndVisualChanges();
			break;
		default:
			check(false); // Need to handle this case
			break;
	}
}
```
6 BeginVisualChanges这个方法，就会遍历Constructing数组，然后依次创建InstancedStaticMeshComp
7 在EndVisualChanges这个方法中