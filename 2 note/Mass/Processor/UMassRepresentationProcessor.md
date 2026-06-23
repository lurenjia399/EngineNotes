
UMassLODDistanceCollectorProcessor
UMassCrowdVisualizationLODProcessor ：public UMassVisualizationLODProcessor
UMassRepresentationProcessor

# UMassLODDistanceCollectorProcessor
1 在游戏线程执行，因为需要读取UMassLODSubsystem
2 UMassLODSubsystem的作用就是管理Viewer，通过一系列的添加方法
```cpp
1 void UMassLODSubsystem::AddPlayerViewer(APlayerController& PlayerController)
2 void UMassLODSubsystem::AddStreamingSourceViewer(const FName StreamingSourceName)
3 void UMassLODSubsystem::AddActorViewer(AActor& ActorViewer)
4 void UMassLODSubsystem::AddEditorViewer(const int32 HashValue, const int32 ClientIndex)

```
