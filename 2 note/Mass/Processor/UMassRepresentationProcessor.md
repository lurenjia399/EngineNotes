
UMassLODDistanceCollectorProcessor
UMassCrowdVisualizationLODProcessor ：public UMassVisualizationLODProcessor
UMassRepresentationProcessor

# UMassLODDistanceCollectorProcessor
1 在游戏线程执行，因为需要读取UMassLODSubsystem
2 UMassLODSubsystem的作用就是管理Viewer数组，通过一系列的添加方法填充Viewer数组
```cpp
1 void UMassLODSubsystem::AddPlayerViewer(APlayerController& PlayerController)
2 void UMassLODSubsystem::AddStreamingSourceViewer(const FName StreamingSourceName)
3 void UMassLODSubsystem::AddActorViewer(AActor& ActorViewer)
4 void UMassLODSubsystem::AddEditorViewer(const int32 HashValue, const int32 ClientIndex)
```
3 在Processor执行的时候分为两步，第一步会根据Viewer的位置构建视锥体，第二步就是遍历查询到的Entity
4 在第二步中会修改FMassViewerInfoFragment这个，会将entity距离Viewer最近的距离设置进去。
5 这也是这个Proce
