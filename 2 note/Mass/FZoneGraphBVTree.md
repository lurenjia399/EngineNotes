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

```cpp 
// ai解释的这个方法
void FZoneGraphBVTree::Build(TStridedView<const FBox> Boxes)

  作用：将一组 3D 包围盒构建成高效的空间查询树。

  ---
  详细步骤解析

  第1步：重置状态（第100-102行）

  Nodes.Reset();
  Origin = FVector::ZeroVector;
  QuantizationScale = 0.0f;

  清空之前的树结构，准备重新构建。

  ---
  第2步：计算量化参数（第109-120行）

  // 计算包含所有盒子的总包围盒
  FBox TotalBounds(ForceInit);
  for (const FBox& Box : Boxes)
  {
      TotalBounds += Box;  // 扩展总边界
  }

  const FVector BoxSize = TotalBounds.GetSize();
  const float MaxDimension = FMath::Max(1.0f, BoxSize.GetMax());
  QuantizationScale = MaxQuantizedCoord / MaxDimension;
  Origin = TotalBounds.Min;

  什么是量化？

  量化是将浮点坐标压缩为整数的过程，节省内存并提高缓存效率。

  具体例子

  // 假设输入的区域包围盒（世界坐标，单位：米）
  Box1: Min(0, 0, 0),     Max(100, 50, 10)
  Box2: Min(200, 100, 5), Max(300, 200, 15)
  Box3: Min(400, 150, 0), Max(500, 250, 20)

  // 步骤1：计算总包围盒
  TotalBounds: Min(0, 0, 0), Max(500, 250, 20)

  // 步骤2：计算尺寸
  BoxSize = (500, 250, 20)
  MaxDimension = 500  // 最大维度

  // 步骤3：计算量化比例
  MaxQuantizedCoord = 65534  // uint16 最大值 - 1
  QuantizationScale = 65534 / 500 = 131.068

  // 步骤4：设置原点
  Origin = (0, 0, 0)

  量化转换示例

  // 世界坐标 → 量化坐标
  世界坐标 (100, 50, 10):
    量化X = (100 - 0) × 131.068 = 13106
    量化Y = (50 - 0) × 131.068 = 6553
    量化Z = (10 - 0) × 131.068 = 1310

  结果: uint16(13106, 6553, 1310)

  // 内存节省：
  浮点: 3 × 4字节 = 12字节
  整数: 3 × 2字节 = 6字节
  节省: 50%

  ---
  第3步：量化所有盒子（第122-129行）

  TArray<FZoneGraphBVNode> Items;
  int32 Index = 0;
  for (const FBox& Box : Boxes)
  {
      FZoneGraphBVNode& Item = Items.AddDefaulted_GetRef();
      Item = CalcNodeBounds(Box);  // 转换为量化坐标
      Item.Index = Index++;        // 记录原始索引
  }

  CalcNodeBounds 内部逻辑（第106-113行）

  FZoneGraphBVNode CalcNodeBounds(const FBox& Box) const
  {
      const FVector LocalMin = (Box.Min - Origin) *
  QuantizationScale;
      const FVector LocalMax = (Box.Max - Origin) *
  QuantizationScale;

      return FZoneGraphBVNode(
          uint16(LocalMin.X), uint16(LocalMin.Y),
  uint16(LocalMin.Z),
          uint16(LocalMax.X + 1), uint16(LocalMax.Y + 1),
  uint16(LocalMax.Z + 1)
      );
  }

  注意：+1 是为了避免舍入误差导致的边界遗漏。

  转换示例

  // 输入：3个区域
  Boxes[0]: Min(0, 0, 0),     Max(100, 50, 10)
  Boxes[1]: Min(200, 100, 5), Max(300, 200, 15)
  Boxes[2]: Min(400, 150, 0), Max(500, 250, 20)

  // 输出：量化节点数组
  Items[0]: MinXYZ(0, 0, 0),         MaxXYZ(13107, 6554, 1311),
  Index=0
  Items[1]: MinXYZ(26214, 13107, 655), MaxXYZ(39321, 26214, 1966),
  Index=1
  Items[2]: MinXYZ(52428, 19660, 0),   MaxXYZ(65535, 32767, 2622),
  Index=2

  ---
  第4步：构建树结构（第131-133行）

  Nodes.Reserve(Items.Num() * 2 - 1);  // 预分配内存
  UE::ZoneGraph::BVTree::Subdivide(Items, 0, Items.Num(), Nodes);

  为什么是 2n - 1 个节点？

  BVH 是完全二叉树：
  - 叶子节点数 = n（输入的盒子数）
  - 内部节点数 = n - 1
  - 总节点数 = n + (n - 1) = 2n - 1

  Subdivide 算法（递归分割）

  // 伪代码
  Subdivide(Items, BeginIndex, EndIndex, OutNodes):
      1. 计算当前范围的总包围盒
      2. 找到最长的轴（X/Y/Z）
      3. 沿该轴排序
      4. 从中间分割成两半
      5. 递归处理左右子树

  构建过程示例

  输入：3个区域 [A, B, C]

  步骤1：计算总包围盒
      [ABC]
      最长轴：X轴

  步骤2：沿X轴排序
      A(X=0-100), B(X=200-300), C(X=400-500)

  步骤3：分割（中间位置）
           [ABC]
          /     \
        [A]    [BC]
                /  \
              [B]  [C]

  最终树结构（线性存储）：
  Nodes[0]: [ABC] 包围盒, Index=-2 (跳到兄弟节点)
  Nodes[1]: [A]   包围盒, Index=0  (叶子，原始索引0)
  Nodes[2]: [BC]  包围盒, Index=-2
  Nodes[3]: [B]   包围盒, Index=1  (叶子，原始索引1)
  Nodes[4]: [C]   包围盒, Index=2  (叶子，原始索引2)

  ---
  完整示例：从输入到输出

  输入

  TArray<FBox> Boxes = {
      FBox(FVector(0,0,0),     FVector(100,50,10)),   // Zone A
      FBox(FVector(200,100,5), FVector(300,200,15)),  // Zone B
      FBox(FVector(400,150,0), FVector(500,250,20))   // Zone C
  };

  BVTree.Build(MakeStridedView(Boxes));

  输出（内存中的树）

  // 量化参数
  Origin = (0, 0, 0)
  QuantizationScale = 131.068

  // 节点数组（深度优先遍历顺序）
  Nodes[0]: MinXYZ(0,0,0) MaxXYZ(65535,32767,2622) Index=-2  //
  根节点
  Nodes[1]: MinXYZ(0,0,0) MaxXYZ(13107,6554,1311)  Index=0   // Zone
   A
  Nodes[2]: MinXYZ(26214,13107,0) MaxXYZ(65535,32767,2622) Index=-2
  Nodes[3]: MinXYZ(26214,13107,655) MaxXYZ(39321,26214,1966) Index=1
    // Zone B
  Nodes[4]: MinXYZ(52428,19660,0) MaxXYZ(65535,32767,2622) Index=2
    // Zone C

  查询示例

  // 查询：点 (250, 150, 10) 在哪些区域？
  FBox QueryBox(FVector(250,150,10), FVector(250,150,10));

  BVTree.Query(QueryBox, [](const FZoneGraphBVNode& Node) {
      UE_LOG(LogTemp, Log, TEXT("Found Zone: %d"), Node.Index);
  });

  // 遍历过程：
  // 1. 检查 Nodes[0] (根) → 重叠 ✓ → 进入子节点
  // 2. 检查 Nodes[1] (A) → 不重叠 ✗ → 跳过
  // 3. 检查 Nodes[2] (BC) → 重叠 ✓ → 进入子节点
  // 4. 检查 Nodes[3] (B) → 重叠 ✓ → 输出 Index=1
  // 5. 检查 Nodes[4] (C) → 不重叠 ✗ → 结束
  查询示例

  // 查询：点 (250, 150, 10) 在哪些区域？
  FBox QueryBox(FVector(250,150,10), FVector(250,150,10));

  BVTree.Query(QueryBox, [](const FZoneGraphBVNode& Node) {
      UE_LOG(LogTemp, Log, TEXT("Found Zone: %d"), Node.Index);
  });

  // 遍历过程：
  // 1. 检查 Nodes[0] (根) → 重叠 ✓ → 进入子节点
  // 2. 检查 Nodes[1] (A) → 不重叠 ✗ → 跳过
  // 3. 检查 Nodes[2] (BC) → 重叠 ✓ → 进入子节点
  // 4. 检查 Nodes[3] (B) → 重叠 ✓ → 输出 Index=1
  // 5. 检查 Nodes[4] (C) → 不重叠 ✗ → 结束

  // 输出：Found Zone: 1 (Zone B)

  ---
  性能优势

  ┌────────────┬────────────┬────────────┐
  │    操作    │  线性搜索  │   BVH 树   │
  ├────────────┼────────────┼────────────┤
  │ 内存       │ 12字节/盒  │ 6字节/节点 │
  ├────────────┼────────────┼────────────┤
  │ 查询       │ O(n)       │ O(log n)   │
  ├────────────┼────────────┼────────────┤
  │ 1000个区域 │ 1000次检查 │ ~10次检查  │
  └────────────┴────────────┴────────────┘

  ---
  总结

  这个 Build 方法做了四件事：

  1. 量化 - 将浮点坐标压缩为 uint16，节省 50% 内存
  2. 归一化 - 计算统一的坐标系（Origin + Scale）
  3. 转换 - 将所有输入盒子转换为量化节点
  4. 分割 - 递归构建二叉树，优化空间查询

  最终得到一个紧凑、高效、缓存友好的空间索引结构。
```
