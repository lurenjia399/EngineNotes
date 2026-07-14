1 
2 FMassTrafficObstacleAvoidanceFragment
```cpp
struct MASSTRAFFIC_API FMassTrafficObstacleAvoidanceFragment : public FMassFragment
{
	// 到跟车的最短距离
	float DistanceToNext = TNumericLimits<float>::Max();
	float TimeToCollidingObstacle = TNumericLimits<float>::Max();
	float DistanceToCollidingObstacle = TNumericLimits<float>::Max();
	FMassEntityHandle Obstacle;
};
```