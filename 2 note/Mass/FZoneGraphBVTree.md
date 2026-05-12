# 1 Build
```cpp
void FZoneGraphBVTree::Build(TStridedView<const FBox> Boxes)
{
	// 初始化当前状态
	// Reset current state
	Nodes.Reset();
	Origin = FVector::ZeroVector;
	QuantizationScale = 0.0f;

	if (Boxes.Num() == 0)
	{
		return;
	}
	
	// 找到所有盒子的最小值和最大值，也就是能容纳所有盒子的盒子
	// Calculate quantization values from the bounds containing all the boxes.
	FBox TotalBounds(ForceInit);
	for (const FBox& Box : Boxes)
	{
		TotalBounds += Box;
	}
	// 容纳盒子的尺寸，最大坐标 减 最小坐标
	const FVector BoxSize = TotalBounds.GetSize();
	// 获取最大尺寸的某一个维度的最大
	const float MaxDimension = FMath::Max(1.0f, BoxSize.GetMax());
	// 量化的每个计量单位的大小
	QuantizationScale = MaxQuantizedCoord / MaxDimension;
	// 容纳盒子的最小点，认为是起始点
	Origin = TotalBounds.Min;

	// Quantize boxes
	TArray<FZoneGraphBVNode> Items;
	int32 Index = 0;
	for (const FBox& Box : Boxes)
	{
		FZoneGraphBVNode& Item = Items.AddDefaulted_GetRef();
		Item = CalcNodeBounds(Box);
		Item.Index = Index++;
	}

	// Build tree
	Nodes.Reserve(Items.Num() * 2 - 1);
	UE::ZoneGraph::BVTree::Subdivide(Items, 0, Items.Num(), Nodes);
}
```
