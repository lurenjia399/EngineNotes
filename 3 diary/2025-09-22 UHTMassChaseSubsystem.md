# 1 AHTMonsterCharacter

# UHTMassChaseSubsystem 追击system

```cpp
/*
void UHTMassChaseSubsystem::Tick(float DeltaTime)
{
	// 更新玩家罪恶值相关的，还有生成各种警察的操作
	UpdatePlayerCrime(DeltaTime);
	// 更新路网上的警察行为
	UpdateCrowdChase(DeltaTime);
}
*/
```

# UpdatePlayerCrime
更新玩家罪恶值方法，罪恶值满足条件开始创建警察
```cpp
void UHTMassChaseSubsystem::UpdatePlayerCrime(float DeltaTime)  
{
	// 1 满足某种条件，创建路网警察。就是通过FMassEntityQuery来查询玩家周围是否有警察，然后把警察添加到ChaseCrowdMap缓存中。
	DoPoliceCrowdChase();
	// 2 根据鼠鼠创建的数量，将需要补充的鼠鼠，添加到PoliceShuShuManagerCache缓存中
	DoPoliceShuShuChase(HTPlayerState->GetSinLevel());
	// 3 警员的数量，添加到PoliceCharacterManagerCache缓存中
	DoPoliceCharacterChase(HTPlayerState->GetSinLevel());
	// 4 警车的数量，添加到PoliceCharacterManagerCache缓存中
	DoPoliceCarChase(HTPlayerState->GetSinLevel());
	// 5 警员障碍物，添加到PoliceObstacleManagerCache缓存中
	DoPoliceObstacle(HTPlayerState->GetSinLevel());
	
}
```
# UpdateCrowdChase
```cpp
/*
// 遍历缓存的路网警察，是有一个枚举标记，每做完一步操作就会进入下一个枚举
void UHTMassChaseSubsystem::UpdateCrowdChase(float DeltaTime)  
{
		for (TMap<FMassEntityHandle, FHTMassCrowdChaseCache>::TIterator It(ChaseCrowdMap); It; ++It)  
	{
		
	}
}

*/
```