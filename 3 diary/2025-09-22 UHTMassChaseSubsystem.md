# 1 AHTMonsterCharacter

# UHTMassChaseSubsystem 追击system

```cpp
/*
void UHTMassChaseSubsystem::Tick(float DeltaTime)
{
	UpdatePlayerCrime(DeltaTime);
}
*/
```

# UpdatePlayerCrime
更新玩家罪恶值方法，罪恶值满足条件开始创建警察
```cpp
void UHTMassChaseSubsystem::UpdatePlayerCrime(float DeltaTime)  
{
	// 1 满足某种条件，创建路网警察。就是通过FMassEntityQuery来cha'x
	DoPoliceCrowdChase();
}
```