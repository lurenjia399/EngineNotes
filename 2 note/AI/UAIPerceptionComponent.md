# OnRegister
```cpp
void UAIPerceptionComponent::OnRegister()
{
	// 根据SensesConfig配置，向PerceptionSystem中注册Sense，会给每个不同的Sense分配一个SenseID，然后通过NewObject创建出Sense
	UAIPerceptionSystem* AIPerceptionSys = 
		UAIPerceptionSystem::GetCurrent(GetWorld());
	if (AIPerceptionSys != nullptr)
	{
		PerceptionFilter.Clear();
		if (SensesConfig.Num() > 0)
		{
			for (auto SenseConfig : SensesConfig)
			{
				if (SenseConfig)
				{
					RegisterSenseConfig(*SenseConfig, *AIPerceptionSys);
				}
			}
			AIPerceptionSys->UpdateListener(*this);
		}
	}
}
```