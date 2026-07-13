1 
2 InterpolatePositionAndOrientationAlongLane
```cpp
void InterpolatePositionAndOrientationAlongLane()
{
	// 1 初始化InOutLaneSegment，
	if (!IsValidLaneSegmentForDistanceAlongLane(InOutLaneSegment, 
		ZoneGraphStorage, LaneIndex, DistanceAlongLane))
	{
		InitLaneSegment(ZoneGraphStorage, LaneIndex, 
			DistanceAlongLane, InOutLaneSegment);
	}
}
```