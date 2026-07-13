1 
2 InterpolatePositionAndOrientationAlongLane
```cpp
void InterpolatePositionAndOrientationAlongLane()
{
	// 1 初始化InOutLaneSegment，表示当前车的位置在哪两个LanePoint之间
	if (!IsValidLaneSegmentForDistanceAlongLane(InOutLaneSegment, 
		ZoneGraphStorage, LaneIndex, DistanceAlongLane))
	{
		InitLaneSegment(ZoneGraphStorage, LaneIndex, 
			DistanceAlongLane, InOutLaneSegment);
	}
	// 2 
}
```