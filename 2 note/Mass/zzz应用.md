# 1 Flee
1 触发

```cpp
1 通过UZoneGraphDisturbanceAnnotationBPLibrary::TriggerDanger这个往ZoneGraphAnnotationSubSystem的Event数组中添加一个。
2 ZoneGraphAnnotationSubSystem会把AnnotatonComp记录到数组中，在subsystem的tick中会遍历AnnotationComp数组，每个Comp都执行HandleEvents，和TickAnnotation。
3 在HandleEvents方法中，会将Danger记录到Comp数组中，在TickAnnotation中会处理这个Danger。
4 在TickAnnotation中会修改ZoneGraphAnnotationSubSystem中的AnnotationTagContainer成员变量。
5 结束，通过TriggerDanger触发的最终目的就是修改AnnotationTagContainer这个变量。
6 UZoneGraphAnnotationComponent这个Comp是挂在AHTMassCrowdSpawner这个上的。
```

2 修改FMassZoneGraphAnnotationFragment
```cpp
1 UMassZoneGraphAnnotationTagsInitializer继承了UMassObserverProcessor，来观察FMassZoneGraphAnnotationFragment
2 如果entity添加了这个Fragment，首先会找FMassZoneGraphLaneLocationFragment这个，来确定entity在哪条Zone上，然后就会从ZoneGraphAnnotationSubSystem的AnnotationTagContainer变量中获取所在Lane的Tag，设置到Fragment中
```

3 stateTree更新状态
```cpp
1 通过FHTMassZoneGraphAnnotationEvaluator这个，来获取entity身上的来观察FMassZoneGraphAnnotationFragment中的Tag，然后传递到statetree中
2 stateTree会根据这个Evaluator的Tag来进入Flee状态。
3 执行FHTMassZoneGraphFindEscapeDanger这个Task，这个计算出EscapeTargetLocation
4 FHTMassZoneGraphFindEscapeDanger
4 然后执行FMassZoneGraphPathFollowTask这个Task，这个Task会设置MoveTarget,ShortPath,CacheLane这三个
```

# 2 受击反应

1 触发

```cpp
1 AHTCrowdCharacterActorBase::TakeDamage 和 OnCapsuleBeginOverlap 这个方法中，会将受击信息存储到UHTMassComponentHitSubsystem中的Map里面，key是entity，value是受击信息，然后会发送SignalEntity
2 然后播放受击montage，通过已经加载动画层，播放不同动画层上的受击montage
```

2 UMassStateTreeProcessor
```cpp
1 UMassStateTreeProcessor继承UMassSignalProcessorBase，在收到Signal回调后会执行SingnalEntities方法
2 回调方法里会创建ExecuteContext，来执行StateTree
```

3 stateTree更新状态
```cpp
1 FHTMassComponentHitEvaluator 这个从UHTMassComponentHitSubsystem中根据entityhandle来获取HitResult击中信息，并根据信息来设置标志位
2 stateTree根据不同的标志位来进入到不同的受击状态（MoveHit还是TakeDamage）
3 StateTree内部不同状态会执行不同的Task
```

1 MoveHit（在移动中收到攻击）
会执行两个StateTask，FHTCrowdCharacterMassStandTask
```cpp
1 通过这个Task，创建一个新的MoveTarget
2 在SteerToMoveTargetProcessor中处理MoveTarget，计算出SteerVelocity
3 在Applymovement里根据Velocity计算位移

```
FMassLookAtTask，修改的FMassLookAtFragment这个片段，
```cpp
1 这个Task里主要设置FMassLookAtFragment这个Fragment
2 玩家撞击到行人，就会通过玩家身上的massagentComponent的获取entity，然后设置这个Fragment，让其朝向玩家。玩家身上的massagentComp是配置的BP_MassExperience中在controller上创建出来的
3 核心处理是通过UMassLookAtProcessor这个来做的，更新LookAt和LookTrajectory
4 然后通过UMassProcessor_Animation来将LookAt和LookTrajectory记录到AnimInstance里
5 AnimInstance更新数据，在通过瞄准偏移实现
```

# 3 Wander

### 1
```cpp
1 Crowd中的EntityConfig会配置UMassZoneGraphNavigationTrait这个，这个Trait会添加FMassZoneGraphLaneLocationFragment
2 FMassZoneGraphLaneLocationFragment这个被添加到Entity后，UMassZoneGraphLocationInitializer这个ObserverProcrssor就会被触发，执行execute方法，方法中的内容是以entity的transform为中心，配置为半径，按照ZoneGraphSubsystem.FindNearestLane方法查询离中心最近的lane，将结果记录到FMassZoneGraphLaneLocationFragment里
3 在UMassStateTreeActivationProcessor执行execute的时候，会首先创建statetree，然后执行StateTreeExecutionContext.Start方法
4 在start方法中，会执行一遍statetree，如果走到了Wander节点中
5 首先执行FMassZoneGraphFindWanderTarget这个Task，简单说就是通过entity位置设置WanderTargetLocation供给给其他Task使用
6 然后执行FHTMassZoneGraphPathFollowTask这个Task，这个Task会设置MoveTarget,ShortPath,CacheLane这三个
```
### 2 FMassZoneGraphFindWanderTarget
```cpp
EStateTreeRunStatus FMassZoneGraphFindWanderTarget::EnterState(
	FStateTreeExecutionContext& Context, 
	const FStateTreeTransitionResult& Transition) const
{
	// 从配置中获取这次移动的距离
	const float MoveDistance = GetDefault<UMassCrowdSettings>()->GetMoveDistance();
	
	// 记录漫游的lane
	InstanceData.WanderTargetLocation.LaneHandle = LaneLocation.LaneHandle;
	// 记录漫游的距离
	InstanceData.WanderTargetLocation.TargetDistance = 
		LaneLocation.DistanceAlongLane + MoveDistance;
	// 记录链接lane的类型
	InstanceData.WanderTargetLocation.NextExitLinkType = EZoneLaneLinkType::None;
	// 重置nextLane
	InstanceData.WanderTargetLocation.NextLaneHandle.Reset();
	// 重置是否反向移动
	InstanceData.WanderTargetLocation.bMoveReverse = false;
	// 在lane的终点是否还要移动
	InstanceData.WanderTargetLocation.EndOfPathIntent = EMassMovementAction::Move;
	// 如果当前想要移动的距离 大于 当前所在lane的长度
	if (InstanceData.WanderTargetLocation.TargetDistance > LaneLocation.LaneLength)
	{
		
		auto FindCandidates = [this, 
			&ZoneGraphAnnotationSubsystem, 
			&MassCrowdSubsystem, 
			ZoneGraphStorage, 
			LaneLocation, 
			&Candidates, 
			&CombinedWeight](const EZoneLaneLinkType Type)-> bool
			{
				// 找LinkLane
				TArray<FZoneGraphLinkedLane> LinkedLanes;
					UE::ZoneGraph::Query::GetLinkedLanes(
					*ZoneGraphStorage, 
					LaneLocation.LaneHandle, 
					Type, 
					EZoneLaneLinkFlags::All, 
					EZoneLaneLinkFlags::None, 
					LinkedLanes);
			}
			return !Candidates.IsEmpty();
		};
		// 首先找Outgoing的linklane,找不到就找Adjacent的
		if (FindCandidates(EZoneLaneLinkType::Outgoing))
		{
			InstanceData.WanderTargetLocation.NextExitLinkType = EZoneLaneLinkType::Outgoing;
		}
		else
		{
			InstanceData.WanderTargetLocation.TargetDistance = LaneLocation.DistanceAlongLane;
			if (FindCandidates(EZoneLaneLinkType::Adjacent))
			{
				InstanceData.WanderTargetLocation.NextExitLinkType = 
					EZoneLaneLinkType::Adjacent;
			}
		}
		// 最终设置NextLane为找到的linklane
		InstanceData.WanderTargetLocation.NextLaneHandle = LinkedLane.DestLane;
	}
}
```
### 3 FHTMassZoneGraphPathFollowTask
```cpp
EStateTreeRunStatus FHTMassZoneGraphPathFollowTask::EnterState(
	FStateTreeExecutionContext& Context, 
	const FStateTreeTransitionResult& Transition) const
{
	if (!RequestPath(MassContext, *TargetLocation))
	{
		return EStateTreeRunStatus::Failed;
	}

	return EStateTreeRunStatus::Running;
}

bool FHTMassZoneGraphPathFollowTask::RequestPath(
	FMassStateTreeExecutionContext& Context, 
	const FMassZoneGraphTargetLocation& RequestedTargetLocation) const
{
	FZoneGraphShortPathRequest& PathRequest = RequestFragment.PathRequest;
	// 开始点是MoveTarget.Center
	PathRequest.StartPosition = MoveTarget.Center;
	// 是否是反向移动
	PathRequest.bMoveReverse = RequestedTargetLocation.bMoveReverse;
	// 沿着lane需要移动的距离
	PathRequest.TargetDistance = RequestedTargetLocation.TargetDistance;
	// Nextlane
	PathRequest.NextLaneHandle = RequestedTargetLocation.NextLaneHandle;
	// CurLane与NextLane的链接类型
	PathRequest.NextExitLinkType = RequestedTargetLocation.NextExitLinkType;
	// 移动到终点的移动行为
	PathRequest.EndOfPathIntent = RequestedTargetLocation.EndOfPathIntent;
	// 是否设置了移动终点
	PathRequest.bIsEndOfPathPositionSet = 
		RequestedTargetLocation.EndOfPathPosition.IsSet();
	// 移动终点位置
	PathRequest.EndOfPathPosition = 
		RequestedTargetLocation.EndOfPathPosition.Get(FVector::ZeroVector);
	// 是否设置移动终点方向
	PathRequest.bIsEndOfPathDirectionSet = 
		RequestedTargetLocation.EndOfPathDirection.IsSet();
	// 移动终点方向
	PathRequest.EndOfPathDirection.Set(
		RequestedTargetLocation.EndOfPathDirection.Get(FVector::ForwardVector));
	// 如果起点或者终点在lane外，需要平滑移动的距离
	PathRequest.AnticipationDistance = 
		RequestedTargetLocation.AnticipationDistance;
	// 终点便宜
	PathRequest.EndOfPathOffset.Set(
		FMath::RandRange(-AgentRadius.Radius, AgentRadius.Radius));
	
	// 
	MoveTarget.CreateNewAction(EMassMovementAction::Move, *World);
	// 
	return UE::MassNavigation::ActivateActionMove(
		*World, 
		Context.GetOwner(), 
		Context.GetEntity(), 
		ZoneGraphSubsystem, 
		LaneLocation, 
		PathRequest, 
		AgentRadius.Radius, DesiredSpeed, MoveTarget, ShortPath, CachedLane);

}

bool ActivateActionMove(const UWorld& World,
			const UObject* Requester,
			const FMassEntityHandle Entity,
			const UZoneGraphSubsystem& ZoneGraphSubsystem,
			const FMassZoneGraphLaneLocationFragment& LaneLocation,
			const FZoneGraphShortPathRequest& PathRequest,
			const float AgentRadius,
			const float DesiredSpeed,
			FMassMoveTargetFragment& InOutMoveTarget,
			FMassZoneGraphShortPathFragment& OutShortPath,
			FMassZoneGraphCachedLaneFragment& OutCachedLane)
{
	// 设置参数的速度
	InOutMoveTarget.DesiredSpeed.Set(DesiredSpeed);
	// 填充FMassZoneGraphCachedLaneFragment
	OutCachedLane.CacheLaneData(*ZoneGraphStorage, LaneLocation.LaneHandle, 
		LaneLocation.DistanceAlongLane, PathRequest.TargetDistance, 
		InflateDistance);
	// 填充FMassZoneGraphShortPathFragment
	if (OutShortPath.RequestPath(OutCachedLane, PathRequest, 
		LaneLocation.DistanceAlongLane, AgentRadius))
	{
		// 设置移动到目标后的移动状态
		InOutMoveTarget.IntentAtGoal = OutShortPath.EndOfPathIntent;
		// 距离目标的距离
		InOutMoveTarget.DistanceToGoal = (OutShortPath.NumPoints > 0) ?
			OutShortPath.Points[OutShortPath.NumPoints - 1].
				DistanceAlongLane.Get() 
			: 0.0f;
	}
}
```

# 4 红绿灯

##  UMassTrafficIntersectionSpawnDataGenerator
```cpp
// 每个交叉区域都会创建出一个entity
void UMassTrafficIntersectionSpawnDataGenerator::Generate() const
{
	FMassEntitySpawnDataGeneratorResult Result;
	Result.SpawnData.InitializeAs<FMassTrafficIntersectionsSpawnData>();
	FMassTrafficIntersectionsSpawnData& IntersectionsSpawnData = Result.SpawnData.GetMutable<FMassTrafficIntersectionsSpawnData>();
	
	// 实际创建SpawnData方法
	Generate(QueryOwner, EntityTypes, Count, IntersectionsSpawnData);

	// 创建entity
	Result.NumEntities = IntersectionsSpawnData.IntersectionFragments.Num();
	Result.EntityConfigIndex = IntersectionEntityConfigIndex;
	Result.SpawnDataProcessor = 
		UMassTrafficInitIntersectionsProcessor::StaticClass();
	
	FinishedGeneratingSpawnPointsDelegate.Execute(MakeArrayView(&Result, 1));
}
```

```cpp
void UMassTrafficIntersectionSpawnDataGenerator::Generate(
	UObject& QueryOwner, 
	TConstArrayView<FMassSpawnedEntityType> EntityTypes, 
	int32 Count, 
	FMassTrafficIntersectionsSpawnData& OutIntersectionsSpawnData) const
{
	// 遍历TrafficLane
	for (const FZoneGraphTrafficLaneData& TrafficLaneData : 
		TrafficZoneGraphData.TrafficLaneDataArray)
	{
		// 一个zone对应一个IntersectionFragment也对应一个IntersectionDesc。如果车道是交叉车道，直接创建Intersection。
		if (TrafficLaneData.ConstData.bIsIntersectionLane 
			&& !MassTrafficSettings->CloseTrafficLaneFilter.Pass(LaneData.Tags))
		{
			const int32 IntersectionZoneIndex = LaneData.ZoneIndex;
			FindOrAddIntersection();
		}
		// 如果当前车道不是交叉车道，但是是最右侧车道。
		else if (TrafficLaneData.bIsRightMostLane)
		{
			if(车道的OutgoingLane是交叉车道)
			{
				// 为交叉车道的IntersectionDesc添加一条Side。
				
				if(车道的相邻车道有Split类型的)
				{
					//1 把当前车道的所有下一条车道添加到side中
					//2 把当前车道链接的交叉车道的下一条道添加到side中
				}
				else
				{
					//1 把当前车道的所有下一条车道添加到side中
					//2 继续遍历当前车道的左车道，把左车道下一条道添加到side中
				}
			}
		}
		
		UE::MassTraffic::FMassTrafficBasicHGrid IntersectionSideHGrid;
		{
			// 构建红绿灯的cell，key是红绿灯数组索引
		}
		UE::MassTraffic::FMassTrafficBasicHGrid 
			CrosswalkLaneMidpoint_HGrid(100.0f);
		{
			// 构建人行道的cell，key是CrosswalkLane的索引
		}
		// 构建交叉路口
		for (FMassTrafficIntersectionFragment& IntersectionFragment : 
			OutIntersectionsSpawnData.IntersectionFragments)
		{
			// 1 确定交叉路口中点，用来找人行道
			// 2 重新排序交叉路口的side，沿顺时针顺序
			// 3 遍历交叉路口，找到隐藏出口
			// 4 找到路口side的人行道，人心道的incoming为CrosswalkWaitingLane
			// 5 遍历side，找到红绿灯记录到side中
			IntersectionDetail->Build();
		}
		// 遍历IntersectionFragments
		for (FMassTrafficIntersectionFragment& IntersectionFragment : 
			OutIntersectionsSpawnData.IntersectionFragments)
		{
			// 如果只有这个交叉路口只有两个边，说明是直行的有两个红绿灯
			if (IntersectionDetail->Sides.Num() == 2 )
			{
				// 添加一个周期，让两边的车道通过。就是把边上记录的车道添加到周期中
				// 在添加一个周期，让两边的人行道通过。就是把人行道添加到周期中
			}
			// 不止两条边，并且有红绿灯
			else if (IntersectionDetail->bHasTrafficLights)
			{
				// 遍历每天边，为每条边都添加一个车辆通过的周期
				// 在添加一个周期，让每条边上的人行道通过
			}
			// 不止两条边，没有红绿灯
			else if (!IntersectionDetail->bHasTrafficLights)
			{
				// 遍历每天边，为每条边都添加一个车辆通过的周期
				// 在添加一个周期，让每条边上的人行道通过
			}
		}
	}
}
```
## UMassTrafficInitIntersectionsProcessor

```CPP
1 通过FMemory::Memswap方法将FMassTrafficIntersectionsSpawnData中的数据都交换到FMassTrafficIntersectionFragment中
2 RestartIntersection 重新开始交叉路口，初始化状态
3 RegisteredTrafficIntersections 注册这个map，key是ZoneIndex,value是Intersection这个entity
```

## UMassTrafficUpdateIntersectionsProcessor
```cpp
if(当前周期的剩余时间 > 0)
{
	if(当前交叉路口不会通行行人，车辆)
	{
		直接设置周期剩余时间为负的
	}
	else if(有红绿灯)
	{
		if(当前交叉路口的当前周期上的车道和人行道都没有车没有人
			&& 没有车在等待使用当前交叉路口
			&& 当前交叉路口没有放行的人行道
			&& 当前交叉路口剩余时间 > 路口准备关闭时间)
		{
			这个周期剩余时间 = 路口准备关闭时间 - 帧长
		}
	}
	// 更新当前周期的红绿灯,设置红绿灯的状态
	UpdateTrafficLightsForCurrentPeriod();
	// 减少当前周期剩余时间
	IntersectionFragment.PeriodTimeRemaining = 
		IntersectionFragment.PeriodTimeRemaining - CountDownSpeedSeconds;
}
if(0 < 当前周期剩余时间 <= 路口准备关闭时间)//黄灯
{
	// 设置当前路口车道为准备关闭，人行道不管，也就是黄灯
	IntersectionFragment.ApplyLanesActionToCurrentPeriod(
		EMassTrafficPeriodLanesAction::SoftPrepareToClose,
		EMassTrafficPeriodLanesAction::None,
		&MassCrowdSubsystem, false);
	// 更新当前红绿灯状态
	IntersectionFragment.UpdateTrafficLightsForCurrentPeriod();
}
if(当前周期剩余时间 <= 0 && 老的当前周期剩余时间 > 0)//红灯
{
	// 设置当前路口车道为关闭，人行道关闭
	IntersectionFragment.ApplyLanesActionToCurrentPeriod(
		EMassTrafficPeriodLanesAction::SoftClose, 
		EMassTrafficPeriodLanesAction::HardClose,
		&MassCrowdSubsystem, false);
	// 更新红绿灯状态
	IntersectionFragment.UpdateTrafficLightsForCurrentPeriod();
	IntersectionFragment.PedestrianLightsShowStop();
	// 路口关闭了，就遍历下一个路口
	continue;
}
if(当前周期剩余时间 <= 0 && 老的当前周期剩余时间 <= 0)//应该是绿灯
{
	if(HighLod && 路口车道有车辆要通过)
	{
		1 设置
		continue;
	}
	// 推进到下一周期
	IntersectionFragment.AdvancePeriod();
	// 设置路口车道，人行道的状态
	IntersectionFragment.ApplyLanesActionToCurrentPeriod(
		VehicleLanesAction, PedestrianLanesAction,
		&MassCrowdSubsystem, false);
	// 更新红绿灯
	IntersectionFragment.UpdateTrafficLightsForCurrentPeriod();
	// 更新当前周期剩余时间，在前边刚推进到下一周期了
	IntersectionFragment.AddTimeRemainingToCurrentPeriod();
}
```

## MassTrafficLightVisualizationProcessor.cpp
```cpp
1 UMassTrafficIntersectionLODCollectorProcessor 用父类LodCollector
2 UMassTrafficIntersectionVisualizationLODProcessor 用父类计算LOD
3 UMassTrafficLightVisualizationProcessor 用父类创建红绿灯Actor
4 UMassTrafficLightUpdateCustomVisualizationProcessor 根据actor还是ISM改变外观
```
## ApplyLanesActionToCurrentPeriod
```cpp
void FMassTrafficIntersectionFragment::ApplyLanesActionToCurrentPeriod(
	const EMassTrafficPeriodLanesAction VehicleLanesAction,
	const EMassTrafficPeriodLanesAction PedestrianLanesAction,
	UMassCrowdSubsystem* MassCrowdSubsystem,
	const bool bForce) 
{
	// 在工作线程执行，会根据参数，直接设置LaneState
	if (PedestrianLanesAction == EMassTrafficPeriodLanesAction::Open)
	{
		MassCrowdSubsystem->SetLaneState(
			LaneHandle, ECrowdLaneState::Opened);			
	}
	else if (PedestrianLanesAction == EMassTrafficPeriodLanesAction::HardClose 
		|| PedestrianLanesAction == EMassTrafficPeriodLanesAction::SoftClose)
	{
		MassCrowdSubsystem->SetLaneState(
			LaneHandle, ECrowdLaneState::Closed);			
	}
}

bool UMassCrowdSubsystem::SetLaneState(const FZoneGraphLaneHandle LaneHandle, ECrowdLaneState NewState)
{
	if (!LaneHandle.IsValid())
	{
		return false;
	}
	
	FZoneGraphCrowdLaneData* CrowdLaneData = GetMutableCrowdLaneData(LaneHandle);
	const bool bSuccess = CrowdLaneData != nullptr;
	if (bSuccess)
	{
		// 直接设置Annotation
		CrowdLaneData->SetState(NewState);
		ZoneGraphAnnotationSubsystem
			->SendEvent(FZoneGraphCrowdLaneStateChangeEvent(LaneHandle, NewState));
	}
	return bSuccess;
}
```

## statetree状态更新
```cpp
1 在Idling中直接检测AnnotationTags，如果包含WaitingLane，不包括ClosedLane，就进入WaitatIntersection节点，就是等待交叉路口节点
2 进入之后，会执行FMassCrowdClaimWaitSlotTask，这里面会更新WaitArea的数量，如果等待的人太过了满了，就会改变LaneTag为Closedlane，退出WaitatIntersection，重新Wander，也就是重新找漫游的位置
3 会在evaluator中判断IsIntersectionNeedQuickPass，就是在红灯的时候还在人行道上，会通过这个方法改变SpeedScale，随后在Wander的PathFollowTask中会根据速度跑过去
```
# 5 SmartObject
```cpp
1 在StateTree的FHTMassFindSmartObjectTask::Tick方法中，会通过FindCandidatesAsync方法创建一个entity，通过pushCommond来添加FMassSmartObjectLaneLocationRequestFragment和FMassSmartObjectRequestResultFragment
2 UMassSmartObjectCandidatesFinderProcessor 这个根据Request，在场景中查找SO，找到了就记录到FMassSmartObjectRequestResultFragment中
3 如果在Processor填充了ResultFragment，下次在FHTMassFindSmartObjectTask::Tick方法里就会使用ResultFragment里的Candidate，作为statetree中的OUT变量供给其他Task
4 OUT有值后，就会进入SmartObjects节点，在这个节点里会通过FHTMassClaimSmartObjectTask这个来输出So的Slot
5 有了SO的Slot后，会进入GetSmartObjectTarget节点，节点里通过FHTMassFindSmartObjectNavTargetTask来找到Slot的位置输出
6 有了Slot位置后，进入FindSmartObjectPath节点，节点里通过FHTMassNavMeshCalculatePathTask来计算出Path，在异步计算Path的时候，会先进入WaitForPathReady的叶子节点
7 有了Path后，进入ReachSmartObjectTarget节点，节点里通过FHTMassFollowPathTask移动
8 移动完成后，进入UseSmartObject，节点里通过FMassUseSmartObjectTask来StartUsingSmartObject
9 使用SO结束后，进入GetBackToRoadTarget节点，节点里通过FHTMassGetNearestLaneBackTargetTask来执行FindNearestLane方法找到距离Slot位置最近的lane上的最近点
10 找到lane上最近点后，进入FindBackToRoadPath节点，通过FHTMassNavMeshCalculatePathTask计算回到lane上最近点的Path
11 找到回到lane的Path后，进入ReachBackToRoadTarget节点，通过FHTMassFollowPathTask移动
12 移动完成后回到Idling节点，重新SelectState
```

```cpp
Root
	->ldling
		->SmartObjects
			->GetSmartObjectTarget
				->FindSmartObjectPath
					->ReachSmartObjectTarget
					->WaitForPathReady
			->UseSmartObject
			->GetBackToRoadTarget
				->FindBackToRoadPath
					->ReachBackToRoadTarget
					->WaitForPathReady
```

# 6 打伞
```cpp 
1 在行人entity中添加WeatherFragment，UHTCrowdWeatherProcessor在execute中获取当前天气，如果雨天雪天通过commond的方式让entity播放montage，并给entity身上添加WeatherTag
2 ISM的打伞表现是通过UHTCrowdAdditionalVisualizationProcessor改变的，这个Processor在WeatherTag添加后执行
```
1 UHTCrowdAdditionalVisualizationTrait
```cpp
1 通过UMassRepresentationSubsystem来创建出ISMComp，并返回Handle
2 把FHTCrowdAdditionalParametersSharedFragment通过AddConstSharedFragment添加到ConstShared中
```
2 UHTCrowdAdditionalInitializer
```cpp
1 是一个ObserveProcessor，观察的FHTCrowdAdditionalVisualizationFragment的添加
2 初始化FHTCrowdAdditionalVisualizationFragment，根据概率选择伞的模型，是否带伞
```
3 UHTCrowdAdditionalVisualizationProcessor
```cpp
1 这个里面通过给ISM数组的AddBatchedTransform方法，来在更新伞ISM的位置，并更新伞的ISM中的CustomData来更新动画数据，通过读取AnimToTexture得来的
```
#  火车
```cpp
1 构建火车路线，以火车站Actor为基准构建，通过FindLaneOverlaps方法找到车站Actor覆盖的Lanes，通过FindNearestLane找到路线的起始点。沿着起始点开始遍历Linklanes，直到遍历完或者回到起点。顺便将火车路线，火车交叉路口等信息记录下来，保存到SaveActor身上（就是一个摆到大世界的Actor）。
2 创建火车的Spawner，配置火车的entityConfig
3 创建火车的Generator，在火车的路线上选择生成点，spawnentity
```
# 变道
```cpp
1 UMassTrafficLaneChangingProcessor 来开启变道流程，选择目标车道，将车直接Teleport到目标车道，计算出变道开始位置，变道结束位置。
2 UMassTrafficVehicleControlProcessor SimpleVehicle计算目标速度，PIDVehicle计算油门刹车转向值，SimpleVehicle就是哪些ISM车，PIDVehicle就是Actor车其中LowActor是mass模拟物理，HightActor是走VehicleMovementComp的真实物理
3 UMassTrafficChooseNextLaneProcessor 选择下一条车道，就是当前车道走完了需要进入的车道，变道过程中没用
4 UMassTrafficInterpolationProcessor SimpleVehicle的变道插值过程，改变车的Transform，如果在开始变道后，车上的数据就已经在目标车道了，这里需要将Transform根据变道进度插值，模拟车变道的移动
5 UMassTrafficVehiclePhysicsProcessor包含PIDVehicle的变道插值过程，先计算出变道的位置，然后
```