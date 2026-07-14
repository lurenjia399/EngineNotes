1 
2 FMassTrafficObstacleAvoidanceFragment
```cpp
struct MASSTRAFFIC_API FMassTrafficObstacleAvoidanceFragment : public FMassFragment
{
	// 到跟车的最短距离，就是当前车的车头到前一辆车的屁股，还有多少距离就要跟前车撞上了
	float DistanceToNext = TNumericLimits<float>::Max();
	
	float TimeToCollidingObstacle = TNumericLimits<float>::Max();
	float DistanceToCollidingObstacle = TNumericLimits<float>::Max();
	FMassEntityHandle Obstacle;
};
```