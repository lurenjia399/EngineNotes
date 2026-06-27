
  1. Behavior/AI 系统
     └─> 计算导航路径、避障
         └─> 更新
  FMassDesiredMovementFragment（期望速度和方向）

  2. UMassApplyForceProcessor（ApplyForces 组）
     └─> 应用外部力（推挤、碰撞等）
         └─> 更新
  FMassDesiredMovementFragment.DesiredVelocity

  3. UMassApplyMovementProcessor（Movement 组）⭐⭐⭐
     └─> 读取 FMassDesiredMovementFragment
         └─> 更新 FMassVelocityFragment
             └─> 更新 FTransformFragment.Transform.Location
                 公式：位置 += 速度 × DeltaTime

  4. MassProcessor_Animation（PostPhysics 组）
     └─> 根据速度更新动画状态
         └─> 更新朝向（Transform.Rotation）

  5. UMassProcessor_CrowdVisualizationCustomData（CrowdVisu
  alization 组）
     └─> 读取 FTransformFragment
         └─> 调用 UpdateISMTransform
             └─> 将 Transform 传递给渲染系统
                 └─> 屏幕上角色移动！
1 