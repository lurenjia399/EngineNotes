在GameInstance里

```cpp

void UAzureGameInstance::Init()
{
	// 这个Init方法里会创建我们的_AzureWLuaImpl，_AzureWLuaImpl是继承LuaImpl，LuaImpl是继承Lua的，在Lua::Init方法里面会执行OnPreInit和OnPostInit方法。在AzureWLuaImpl::OnPreInit这个方法里面就会有每个类的Register。在
	if (!m_AzureGame->Init(this))
		return;
	// 在lua中创建GameInstance的table，并在table里放一些函数
	InitLua();
}
bool UAzureGame::Init(UAzureRuntimeGameInstance* pGameInst)
{
	wLua::LuaStatic::setLuaPath(LuaPath.c_str());
	// 创建\luaL_newstate。还有在lua中创建一些全局变量
	InitLuaModule();
}
```