# 1 RegisterZoneShapeComponent
```c++
/*
void FZoneGraphBuilder::RegisterZoneShapeComponent(UZoneShapeComponent& ShapeComp)
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

void FZoneGraphBuilder::UnregisterZoneShapeComponent(UZoneShapeComponent& ShapeComp)
{
	if (int32* Index = ShapeComponentToIndex.Find(&ShapeComp))
	{
		check(ShapeComponents.IsValidIndex(*Index));
		FZoneGraphBuilderRegisteredComponent& Registered = ShapeComponents[*Index];
		// Remove from grid
		HashGrid.Remove(uint32(*Index), Registered.CellLoc);
		// Remove from index lookup
		ShapeComponentToIndex.Remove(Registered.Component);
		// Remove from list
		Registered.Component = nullptr;
		ShapeComponentsFreeList.Add(*Index);
	}

	bIsDirty = true;
#endif
}
*/
```
注册，每个UZoneShapeComponent的onRegister方法中就会执行