# 初始化

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
	// 创建luaL_newstate。这个方法里会创建我们的_AzureWLuaImpl，然后调用Lua::Init方法。Lua::Init方法里面会创建FLuaObjectReferencer这个单例类。_AzureWLuaImpl是继承LuaImpl，LuaImpl是继承Lua的，在Lua::Init方法里面会执行OnPreInit和OnPostInit方法。在_AzureWLuaImpl::OnPreInit这个方法里面就会执行每个类的Register。在_AzureWLuaImpl::OnPostInit这个方法里面会执行每个类的SetMtLink方法。
	// 每个类的Register方法，就是命名为类名称的元表，在元表里添加导出的静态函数，然后将元表注册到全局表LUA_REGISTRYINDEX中
	// 每个类的SetMtLink方法，就是通过类的UClass找到父类，然后通过类的元表__index和__newIndex来决定父类，建立继承关系
	InitLuaModule();
}
```
# 蓝图中绑定lua文件
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
	// 这个是把require(*inLuaModule)这个返回值，放到了栈里
	lua_rawgeti(L, LUA_REGISTRYINDEX, inst);//uobj,obj
	userdata->SetUserDataValue(L,-2);//uobj
	// 后面还有一系列的方法，看上去是将UFunction宏标记的函数导出到lua的表中？
}
```
# 通过UObject返回LuaObject
FLuaUtils::ReturnUObject
```cpp
LuaUObjectUserData* FLuaUtils::ReturnUObject(lua_State* L, UObject* Obj)
{
	return ReturnUObjectPrivate(L, Obj, STRONGWEAK_AUTO);
}
/*
1.)首先会尝试从Lua的全局弱表（g_udref）中获取到UObject对应的LightUserData对象, 把这个lightuserdata尝试转换为LuaUObjectUserData对象，如果这个对象的flag是GC状态，那么就调用接口RemoveObjectReference把它从ScriptCreatedObjects中和Lua全局弱表（g_udref）中移出并置空该对象。如果lightuserdata对象为空，则调用lua_newuserdata新生成一个LuaUObjectUserData对象（标记为Born），并加入到FLuaObjectReferencer的ScriptCreatedObjects映射表中。

2.)接着通过UE4的反射机制查找这个UObject本身的元表是否设置成功，如果没有的话重新设置元表（其中包括字段i-d，__gc, name),并注册到全局表_G中,然后设置该userdata的元表为以该UObject类名称（GetClass）为表名，并再Lua全局弱表中g_udref写入weakT[lud]= userdata(UELuaObject)
*/
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
		// 将userdata加入到FLuaObjectReferencer的ScriptCreatedObjects映射表中
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
			// 如果名称为clsname原表不存在赋值exist为false,如果原表存在break
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
# 通过LuaObject返回UObject
FLuaUtils::GetUObject
```cpp
/*
GetUObject函数用来从Lua对象返回对应的UObject对象。首先获取Lua对象对应的FLuaObjectReferencer对象 但是如果满足以下条件就需要从ScriptCreatedObjects表以及Lua弱表g_udref里移除对应的映射关系，并返回nullptr

1.)LuaUObjectUserData的flag是DESTORY或者GC
2.)FLuaObjectReferencer对象里的UObject和ScriptCreatedObjects表里的UObject不是同一个对象
3.)FLuaObjectReferencer对象里的UObject处于被销毁状态（IsPendingKill为true）
*/
UObject * FLuaUtils::GetUObject(lua_State * L, int ParamIndex, const char * dummy,wLua::LuaUObjectUserData** ppUserData)
{
	return GetUObject(L, ParamIndex, ppUserData);
}

UObject * FLuaUtils::GetUObject(lua_State * L, int ParamIndex,wLua::LuaUObjectUserData** ppUserData,int * pStatus)
{
	// 拿到栈中ParamIndex索引处的userdata
	wLua::LuaUObjectUserData * userdata = wLua::FLuaUtils::IsUObjectUserDataType(L, ParamIndex);
	// 将userdata赋值到外部
	if (ppUserData)
		*ppUserData = userdata;
	// 如果Object不存在了，就返回nullptr
	UObject* Obj = userdata->uobj;
	if (!Obj)
	{
		if (pStatus)
			*pStatus = NullUObject;
		return nullptr;
	}
	// 如果这些条件判断通过了，就清空userdata
	if ((userdata->flag & wLua::LuaObjectFlag::DESTROYED)
		|| (userdata->flag & wLua::LuaObjectFlag::GC)
		|| (userdata->IsWeakPtr() && !userdata->IsValidUObject())
		|| ((userdata->flag & wLua::LuaObjectFlag::UNBIND) && !userdata->IsWeakPtr() && !wLua::FLuaObjectReferencer::Get(L).HasUObject(Obj, userdata->stamp))
		|| ((userdata->flag & wLua::LuaObjectFlag::UNBIND) && userdata->IsWeakPtr() &&  !wLua::FLuaObjectReferencer::Get(L).HasWeakUObject(Obj, userdata->stamp))
		|| !IsValid(Obj)
		)
	{
		if (pStatus)
			*pStatus = PendingKillUObject;
		FLuaUtils::RemoveObjectReference(Obj,userdata->stamp ,L);
		userdata->uobj = nullptr;
		return nullptr;
	}
	return Obj;
}

```

# 关键点

- lua是怎么拿到UObject对象的？
> 主要就是通过FLuaUtils::ReturnUObject这个接口将表示UObject的userdata放到栈中，将这个userdata返回到lua侧。这个方法首先就是从全局表中获取UObject对应的lightuserdata，如果获取不到就创建一个新的lightuserdata，然后就是设置lightuserdata的元表（元表的表名是GetClass的名称），如果元表不存在就创建一个并注册到全局表中。
- lua是怎么模拟UObject对象的继承关系的？
> 在GameInstance::Init的时候会先进行导出方法和类元表的注册，并将元表注册到lua全局表LUA_REGISTRYINDEX中，然后就会执行SetMtLink方法，通过UClass获取父类，然后通过元表中的__index和__newindex来完成继承关系。
- lua是怎么调用到UObject的方法的呢？
> c侧会创建luaObject的userdata，和userdata的元表，然后lua侧会保存这个userdata，lua侧调用方法就是会调用到userdata中的元表里的方法，元表里的方法是在每个类的Register方法里静态注册的。

- lua是怎么和c++交互的呢？
> 总共分成了几部分，1 lua如何表示UObject对象。2 lua如何模拟UObject对象的继承关系。3 lua如何调用到UObject的方法。4 c++如何调用lua的方法。