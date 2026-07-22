# StartTree

```cpp
void UBehaviorTreeComponent::StartTree(UBehaviorTree& Asset, EBTExecutionMode::Type ExecuteMode)
{
	/*
	1 StopTree
	*/
	StopTree(EBTStopMode::Safe);

	TreeStartInfo.Asset = &Asset;
	TreeStartInfo.ExecuteMode = ExecuteMode;
	TreeStartInfo.bPendingInitialize = true;
	/*
	1 核心初始化的操作
	*/
	ProcessPendingInitialize();
}
```

```cpp
bool UBehaviorTreeComponent::PushInstance(UBehaviorTree& TreeAsset)
{
	/*
	1 向 Manager 请求加载这棵树，拿到树的根节点 RootNode 和这棵树需要的总内存大小 InstanceMemorySize。
	2 每次Load相同的tree，会从缓存中找信息，当某个 AI 第一次跑某棵行为树，LoadTree 会把树"编译"一次（StaticDuplicateObject 复制出一套节点对象、算好每个节点的内存偏移），存进 LoadedTemplates。之后所有跑同一棵树的 AI，LoadTree 直接返回缓存里那同一个 Root 节点指针——不再复制。
	*/
	const bool bLoaded = BTManager->LoadTree(TreeAsset, RootNode, InstanceMemorySize);
	
	if (bLoaded)
	{
		/*
		1 添加一个BehaviorTreeInstance，填入InstanceStack数组中，表示一个行为树实例
		2 添加一个BehaviorTreeInstanceId，填入KnownInstances数组中，表示一个行为树子树实例。UpdateInstanceId 用"从根到当前节点的路径"生成一个唯一标识，用来复用同一子树在同一位置的历史内存。
		3 这个实例是在BehaviorTreeComp上的，每个AI一个
		*/
		FBehaviorTreeInstance& NewInstance = InstanceStack.AddDefaulted_GetRef();
		NewInstance.InstanceIdIndex = UpdateInstanceId(&TreeAsset, ActiveNode, InstanceStack.Num() - 1);
		NewInstance.RootNode = RootNode;
		NewInstance.ActiveNode = NULL;
		NewInstance.ActiveNodeType = EBTActiveNode::Composite;
		/*
		1 每个AI单独存储的行为树实例，赋值实例独有的内存空间InstanceMemory
		*/
		FBehaviorTreeInstanceId& InstanceInfo = 
			KnownInstances[NewInstance.InstanceIdIndex];
		int32 NodeInstanceIndex = InstanceInfo.FirstNodeInstance;
		const bool bFirstTime = (InstanceInfo.InstanceMemory.Num() != InstanceMemorySize);
		if (bFirstTime)
		{
			InstanceInfo.InstanceMemory.AddZeroed(InstanceMemorySize);
			InstanceInfo.RootNode = RootNode;
		}
		/*
		1 如果Node设置了bCreateNodeInstance，就需要创建NodeInstance，填入NodeInstances数组中
		*/
		NewInstance.SetInstanceMemory(InstanceInfo.InstanceMemory);
		NewInstance.Initialize(*this, *RootNode, NodeInstanceIndex, bFirstTime ? EBTMemoryInit::Initialize : EBTMemoryInit::RestoreSubtree);
	}
}
```
