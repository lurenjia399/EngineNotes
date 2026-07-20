# OnRegister
```cpp
void UAIPerceptionComponent::OnRegister()
{
	// 
	/*
	1 根据SensesConfig配置，向PerceptionSystem中注册Sense，会给每个不同的Sense分配一个SenseID，然后通过NewObject创建出Sense，添加到System中的Senses数组中
	2 会将SenseID按照位的偏移，添加到PerceptionFilter中记录在Int32类型的成员变量中。表示这个PerceptionComp会对哪些Sense做出反应，也就是监听了哪些Sense
	3 然后会像PerceptionSystem中添加ListenerContainer，把PerceptionComp作为Listener添加到ListenerContainer中
	*/
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

# OnUnregister
```cpp
void UAIPerceptionComponent::OnUnregister()
{
	CleanUp();
	Super::OnUnregister();
}

void UAIPerceptionComponent::CleanUp()
{
	if (bCleanedUp == false)
	{
		ForgetAll();

		UAIPerceptionSystem* AIPerceptionSys = 
			UAIPerceptionSystem::GetCurrent(GetWorld());
		if (AIPerceptionSys != nullptr)
		{
			AIPerceptionSys->UnregisterListener(*this);
			AActor* MutableBodyActor = GetMutableBodyActor();
			if (MutableBodyActor)
			{
				AIPerceptionSys->UnregisterSource(*MutableBodyActor);
			}
		}

		if (HasAnyFlags(RF_BeginDestroyed) == false)
		{
			AActor* Owner = GetOwner();
			if (Owner != nullptr)
			{
				Owner->OnEndPlay.RemoveDynamic(this, &UAIPerceptionComponent::OnOwnerEndPlay);
			}
		}

		bCleanedUp = true;
	}
}
```