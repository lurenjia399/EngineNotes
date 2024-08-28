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
bool UAzureGame::Init(UAzureRuntimeGameInstance* pGameInst)
{
	wLua::LuaStatic::setLuaPath(LuaPath.c_str());
	// 创建
	InitLuaModule();
}
```