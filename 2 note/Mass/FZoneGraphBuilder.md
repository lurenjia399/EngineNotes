# 1 RegisterZoneShapeComponent
```c++
/*
void FZoneGraphBuilder::RegisterZoneShapeComponent(UZoneShapeComponent& ShapeComp)
{
	// TODO: Move the builder into editor. This call is guarded editor only at call site already.
#if WITH_EDITOR
	// TODO: we could potentially separate out automatic building logic, and use object iterator to get all relevant data instead.
	// Add to list
	int32 Index = ShapeComponentsFreeList.IsEmpty() ? ShapeComponents.AddDefaulted() : ShapeComponentsFreeList.Pop();
	FZoneGraphBuilderRegisteredComponent& Registered = ShapeComponents[Index];
	Registered.Component = &ShapeComp;
	// Add to grid
	const FBoxSphereBounds Bounds = ShapeComp.CalcBounds(ShapeComp.GetComponentTransform());
	Registered.CellLoc = HashGrid.Add(uint32(Index), Bounds.GetBox());
	// Add to index lookup
	ShapeComponentToIndex.Add(&ShapeComp, Index);
	// Compute shape hash, used to detect if the shape has changed.
	Registered.ShapeHash = ShapeComp.GetShapeHash();

	bIsDirty = true;
#endif
}
*/
```
注册，每个UZoneShapeComponent的onRegister方法中就会执行