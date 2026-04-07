# 1 RegisterZoneShapeComponent
```c++
/*
void FZoneGraphBuilder::RegisterZoneShapeComponent(
	UZoneShapeComponent& ShapeComp)
{
	// 道理一样，获取数组中空闲的索引
	int32 Index = 
		ShapeComponentsFreeList.IsEmpty() ? 
		ShapeComponents.AddDefaulted() : ShapeComponentsFreeList.Pop();
	// 拿到索引中的BuilderComp对象
	FZoneGraphBuilderRegisteredComponent& Registered = ShapeComponents[Index];
	// 组装BuilderComp对象，缓存Comp指针
	Registered.Component = &ShapeComp;
	// 组装BuilderComp对象，计算Comp的Bounds
	const FBoxSphereBounds Bounds = 
		ShapeComp.CalcBounds(ShapeComp.GetComponentTransform());
	// 将Bounds放到2D的Grid平面上，计算出位于哪个格子中，将结果记录到BuilderComp对象里
	Registered.CellLoc = HashGrid.Add(uint32(Index), Bounds.GetBox());
	// 缓存Comp到数组索引的map，加速查询
	ShapeComponentToIndex.Add(&ShapeComp, Index);
	// 创建一个唯一的hash值，也能通过这个值来检测到BuilderComp对象是否改变
	Registered.ShapeHash = ShapeComp.GetShapeHash();
	// 标记为脏的
	bIsDirty = true;
}

void FZoneGraphBuilder::UnregisterZoneShapeComponent(
	UZoneShapeComponent& ShapeComp)
{
	// 从数组中找Comp
	if (int32* Index = ShapeComponentToIndex.Find(&ShapeComp))
	{
		// 拿到BuilderComp对象
		FZoneGraphBuilderRegisteredComponent& Registered = ShapeComponents[*Index];
		// 从HashGrid中移除掉
		HashGrid.Remove(uint32(*Index), Registered.CellLoc);
		// 从加速结构中移除掉
		ShapeComponentToIndex.Remove(Registered.Component);
		// 置空
		Registered.Component = nullptr;
		ShapeComponentsFreeList.Add(*Index);
	}

	bIsDirty = true;
}
*/
```
注册，每个UZoneShapeComponent的onRegister方法或者是onUnRegister就会执行
# 2 Build
```cpp
/*
void FZoneGraphBuilder::Build(AZoneGraphData& ZoneGraphData)
{
	
	ULevel* CurrentLevel = ZoneGraphData.GetLevel();
	ZoneGraphData.Modify();
	
	// 重置Storage中的所有数据
	FZoneGraphStorage& ZoneStorage = ZoneGraphData.GetStorageMutable();
	ZoneStorage.Reset();
	//
	for (FZoneGraphBuilderRegisteredComponent& Registered : ShapeComponents)
	{
		if (Registered.Component && Registered.Component->GetComponentLevel() == CurrentLevel)
		{
			
		}
}
*/
```