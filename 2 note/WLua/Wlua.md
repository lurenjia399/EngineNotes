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
	// 创建luaL_newstate。这个方法里会创建我们的_AzureWLuaImpl，然后调用Lua::Init方法。_AzureWLuaImpl是继承LuaImpl，LuaImpl是继承Lua的，在Lua::Init方法里面会执行OnPreInit和OnPostInit方法。在AzureWLuaImpl::OnPreInit这个方法里面就会执行每个类的Register。在_AzureWLuaImpl::OnPostInit这个方法里面会执行每个类的SetMtLink方法。
	InitLuaModule();
}
```

这个图显示的是UObject的初始化，在InitLuaActor方法中的第二个参数就是我们在蓝图里写的lua文件路径
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202408291046828.png)

```cpp
bool LuaActorBase::InitLuaActor(UObject * This, const FString& inLuaModule,lua_State * inL)
{
	nativeClass = (!This->GetClass()->HasAnyClassFlags(CLASS_CompiledFromBlueprint) && This->GetClass()->HasAnyClassFlags(CLASS_Native));

	// 用lua中的require函数来加载，返回值就是require函数返回值在栈里的序号。require(*inLuaModule)
	int nret = require_caller.CallWithMultiRet("require", TCHAR_TO_UTF8(*inLuaModule));
	
}
```
