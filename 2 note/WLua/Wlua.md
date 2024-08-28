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
	// 在lua中创建GameInstance的table，并在table里放一些h
	InitLua();
}
bool UAzureGame::Init(UAzureRuntimeGameInstance* pGameInst)
{
	wLua::LuaStatic::setLuaPath(LuaPath.c_str());
	// 创建\luaL_newstate。还有在lua中创建一些全局变量
	InitLuaModule();
}
```