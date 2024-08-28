在GameInstance里

```cpp
if (!m_AzureGame->Init(this))
		return;


	//	Init lua(cppInterface)
	InitLua();
void UAzureGameInstance::Init()
{
	if (!m_AzureGame->Init(this))
		return;
	
	InitLua();
}
```