在GameInstance里

```cpp

void UAzureGameInstance::Init()
{
	
	if (!m_AzureGame->Init(this))
		return;
	// 在lua中创建GameInstance的table，并在table里放一些函数
	InitLua();
}
bool UAzureGame::Init(UAzureRuntimeGameInstance* pGameInst)
{
	// 创建luaL_newstate。这个方法里会创建我们的_AzureWLuaImpl，然后调用Init方法。_AzureWLuaImpl是继承LuaImpl，LuaImpl是继承Lua的，在Lua::Init方法里面会执行OnPreInit和OnPostInit方法。在AzureWLuaImpl::OnPreInit这个方法里面就会执行每个类的Register。在_AzureWLuaImpl::OnPostInit这个方法里面会执行每个类的SetMtLink方法。
	InitLuaModule();
}
```