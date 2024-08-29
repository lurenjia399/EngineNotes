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
// 这个方法总的来说就是在lua那边创建userdata，来和This一一对应，并设置一些元方法
bool LuaActorBase::InitLuaActor(UObject * This, const FString& inLuaModule,lua_State * inL)
{
	nativeClass = (!This->GetClass()->HasAnyClassFlags(CLASS_CompiledFromBlueprint) && This->GetClass()->HasAnyClassFlags(CLASS_Native));

	// 用lua中的require函数来加载，返回值就是require函数返回值在栈里的序号。require(*inLuaModule)
	int nret = require_caller.CallWithMultiRet("require", TCHAR_TO_UTF8(*inLuaModule));
	// 通过UObject对象获取lua对象。
	userdata = wLua::FLuaUtils::ReturnUObject(L, This); //uobj
}
```

FLuaUtils::ReturnUObject
```cpp
LuaUObjectUserData* FLuaUtils::ReturnUObject(lua_State* L, UObject* Obj)
{
	return ReturnUObjectPrivate(L, Obj, STRONGWEAK_AUTO);
}

LuaUObjectUserData* ReturnUObjectPrivate(lua_State * L, UObject * Obj, LuaRefType luaRefType)
{
	// 首先会尝试从Lua的全局弱表（g_udref）中获取到UObject对应的LightUserData对象
	AzureHelpFuncs::tryGetUserdataFromWeakTable(L, Obj, regIndex);
	//如果lightuserdata对象为空，则调用lua_newuserdata新生成一个LuaUObjectUserData对象（标记为Born），并加入到FLuaObjectReferencer的ScriptCreatedObjects映射表中
	if (lua_isnil(L, -1) || !valid)
	{
		lua_pop(L, 1); //ref
		int32 AlignedSize = isWeakPtr ? sizeof(LuaUObjectUserDataWeakPtr) : sizeof(LuaUObjectUserData);
		// 创建Userdata，也就是生成LuaUObjectUserData对象
		void * data = lua_newuserdata(L, AlignedSize); //ref,ud
		if (isWeakPtr)
			userdata = new(data) LuaUObjectUserDataWeakPtr;
		else
			userdata = new(data) LuaUObjectUserData;
		userdata->stamp = userdata;
		userdata->uobj = Obj;
		userdata->flag = wLua::LuaObjectFlag::BORN;
		// 加入到FLuaObjectReferencer的ScriptCreatedObjects映射表中
		wLua::FLuaObjectReferencer::Get(L).AddObjectReference(Obj,userdata-
		ANSICHAR clsname[NAME_SIZE];
		bool exist = true;
		UClass * cls = Obj->GetClass();
		
		UClass* firstNativeClass = nullptr;
		while (cls)
		{
			cls->GetFName().GetPlainANSIString(clsname);
			lua_pushstring(L, clsname);//ref,ud,key
			// 从全局弱表中招键为clsname的原表，然后将原表放到栈顶
			lua_rawget(L, LUA_REGISTRYINDEX); //ref,ud,mt
			// 如果原表不存在赋值exist为false,如果原表存在break
			if (lua_isnil(L, -1))
				exist = false;
			else
				break;
			// 弹出元表
			lua_pop(L, 1); //ref,ud
			// 让cls指向下一个
			cls = cls->GetSuperClass();
		}
		// 这部分是元表不存在，就创建一个名称为clsname的元表
		if(!exist)
		{
			FName fname = firstNativeClass->GetFName();
			const FNameEntry* entry = fname.GetDisplayNameEntry();
			ANSICHAR ansiName[NAME_SIZE];
			entry->GetAnsiName(ansiName);
			lua_pushstring(L, ansiName);//ref,ud,mt,mtname
			lua_getglobal(L, "_G");//ref,ud,mt,mtname,global
			lua_pushvalue(L, -2);//ref,ud,mt,mtname,global,mtname
			lua_pushvalue(L, -4);//ref,ud,mt,mtname,global,mtname,mt
			lua_settable(L, -3); //ref,ud,mt,mtname,global
			lua_pop(L, 1);//ref,ud,mt,mtname
			lua_pushvalue(L, -2);//ref,ud,mt,mtname,mt
			lua_rawset(L, LUA_REGISTRYINDEX);//ref,ud,mt	 //for luaL_checkudata
		}
		// 将元表和ref绑定上吧
		ua_setmetatable(L, -2); //ref,ud
		lua_pushlightuserdata(L, Obj); //ref,ud,obj
		lua_pushvalue(L, -2); //ref,ud,obj,ud
		lua_settable(L, -4); //ref,ud
		
		lua_remove(L, -2); //ud
		return userdata
}
```