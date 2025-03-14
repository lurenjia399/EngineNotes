# WLua
LUA_REGISTRYINDEX注册表解释，c代码可以访问，lua无法访问
https://www.cnblogs.com/zsb517/p/6418929.html

## 初始化

![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202503141701774.png)

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
	// 创建luaL_newstate。这个方法里会创建我们的_AzureWLuaImpl(_AzureWLuaImpl是继承LuaImpl，LuaImpl是继承Lua的)，然后调用Lua::Init方法，Lua::Init方法里面会创建FLuaObjectReferencer这个单例类。在Lua::Init方法里面会执行OnPreInit和OnPostInit方法。在_AzureWLuaImpl::OnPreInit这个方法里面就会执行每个类的Register。在_AzureWLuaImpl::OnPostInit这个方法里面会执行每个类的SetMtLink方法。
	// 每个类的Register方法，就是命名为类名称的元表，在元表里添加导出的静态函数，然后将元表注册到全局表LUA_REGISTRYINDEX中
	// 每个类的SetMtLink方法，就是通过类的UClass找到父类，然后通过类的元表__index和__newIndex来决定父类，建立继承关系
	InitLuaModule();
}

```

```cpp
bool Lua::Init(UGameInstance* InGI, bool bPrintDisplay /* = false */)
{
	// 创建lua_state
	lua_State * tempL = luaL_newstate();
	mainL = tempL;

	//
	for (int i = 1; i <= RegistryIndex::REGINDEX_MAX; ++i)
	{
		// 通过switch创建相应的弱表。
		// 比如REGINDEX_UOBJECT_TO_USERDATA索引就代表一个key和value都是弱引用的表。
		// REGINDEX_WEAKUOBJECT_TO_USERDATA索引就代表一个key是弱引用的表。
		switch (i)
		{
			case REGINDEX_UOBJECT_TO_USERDATA:
				lua_newtable(mainL);//ref
				lua_newtable(mainL); //ref,mt
				lua_pushstring(mainL, "kv"); //ref,mt,"v"
				lua_setfield(mainL, -2, "__mode"); //ref,mt
				lua_setmetatable(mainL, -2); //ref
				break;
			case REGINDEX_WEAKUOBJECT_TO_USERDATA:
			case REGINDEX_WEAKUOBJECT_TO_USERDATA_FOR_WIDGET:
				lua_newtable(mainL);//ref
				lua_newtable(mainL); //ref,mt
				lua_pushstring(mainL, "k"); //ref,mt,"v"
				lua_setfield(mainL, -2, "__mode"); //ref,mt
				lua_setmetatable(mainL, -2); //ref
				break;
			default:
				lua_newtable(mainL);
		}
		// 第一个参数是luaState的指针，第二个参数是LUA_REGISTRYINDEX。这个函数的作用是弹出栈顶的值，并且用一个新分配的整数key把这个值注册到注册表里，然后返回这个整数key。
		// 简单理解就是将我们在switch分支中创建的弱表添加到了注册表中。
		int ref = luaL_ref(mainL, LUA_REGISTRYINDEX);
	}
	// 接入gc系统，创建出uobjectReferencer，uobjectReferencer是继承自FGCObject的
	uobjectReferencer = new FLuaObjectReferencer();
	uobjectReferencer->Init(mainL);

	// 在_AzureWLuaImpl::OnPreInit这个方法里面就会执行每个类的Register。每个类的Register方法，创建一张元表，这张元表的内容是类导出的各个方法。然后将表放入到全局注册表中，key是类名。
	OnPreInit(L);
	// _AzureWLuaImpl::OnPostInit这个方法里面会执行每个类的SetMtLink方法。每个类的SetMtLink方法，就是通过类的UClass找到父类，然后通过类的元表__index和__newIndex来决定父类，建立继承关系
	OnPostInit(L);
}
```
## 蓝图中绑定lua文件
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
## 通过UObject返回userdata
FLuaUtils::ReturnUObject
```cpp
LuaUObjectUserData* FLuaUtils::ReturnUObject(lua_State* L, UObject* Obj)
{
	return ReturnUObjectPrivate(L, Obj, STRONGWEAK_AUTO);
}

LuaUObjectUserData* ReturnUObjectPrivate(lua_State * L, UObject * Obj, LuaRefType luaRefType)
{
	// 首先会尝试从Lua的注册表中的全局弱表（g_udref）中获取到UObject对应的LightUserData对象
	AzureHelpFuncs::tryGetUserdataFromWeakTable(L, Obj, regIndex);
	//如果lightuserdata对象为空，则调用lua_newuserdata新生成一个LuaUObjectUserData对象（标记为Born），并加入到FLuaObjectReferencer中，也就是纳入gc系统
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
// 从注册表的regIndex + 1这个索引下获取table，然后获取这个table的lightuserdata这个索引下的userdata
void tryGetUserdataFromWeakTable(lua_State * L, void * Obj, uint8 regIndex)
{
	lua_rawgeti(L, LUA_REGISTRYINDEX, regIndex + 1);
	lua_pushlightuserdata(L, Obj); //ref,obj
	lua_gettable(L, -2); //ref,ud
}
```

1 首先会尝试从Lua注册表中的全局弱表（g_udref）中获取到UObject对应的LightUserData对象, 把这个lightuserdata尝试转换为LuaUObjectUserData对象。如果找到了就直接返回了。
3 如果lightuserdata对象为空，则调用lua_newuserdata新生成一个LuaUObjectUserData对象（标记为Born），然后赋值对象中的stamp，uobj以及flag，并加入到FLuaObjectReferencer的ScriptCreatedObjects映射表中,也就是纳入到gc系统中。
4 通过UObject的UClass获取到ClassName，然后再注册表中找ClassName对应的元表。如果找不到就创建一个。
5 如果找到了，就将其设置为lightuserdata的元表。然后在将lightuserdata添加到注册表中的全局弱表里，key为UObject*。
## 通过userdata返回UObject
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
1 首先我们手中有userdata，就把userdata转换成LuaUObjectUserData，然后返回其中的uobj成员变量（是一个指针指向了UObject）就行。
## 关键点

- lua的注册表里都有什么？
> 1 我们自己定义的全局弱表是在注册表中，key是REGINDEX_UOBJECT_TO_USERDATA 这些索引，value是表
> 2 类的元表也都在注册表中，key是类名，value是元表，元表中是类导出的各种函数以及变量
- lua是怎么拿到UObject对象的？
> 主要就是通过FLuaUtils::ReturnUObject这个接口将表示UObject的userdata放到栈中，将这个userdata返回到lua侧。这个方法首先就是从全局表中获取UObject对应的lightuserdata，如果获取不到就创建一个新的lightuserdata，然后就是设置lightuserdata的元表（元表的表名是GetClass的名称），如果元表不存在就创建一个并注册到全局表中。
- lua是怎么模拟UObject对象的继承关系的？
> 在GameInstance::Init的时候会先进行导出方法和类元表的注册，并将元表注册到lua全局表LUA_REGISTRYINDEX中，然后就会执行SetMtLink方法，通过UClass获取父类，然后通过元表中的__index和__newindex来完成继承关系。
- lua是怎么调用到UObject的方法的呢？
> c侧会创建luaObject的userdata，和userdata的元表，然后lua侧会保存这个userdata，lua侧调用方法就是会调用到userdata中的元表里的方法，元表里的方法是在每个类的Register方法里静态注册的。

> 第一部分是初始化的过程，是在GameInstance::Init方法里面，执行wlua类的注册，将静态导出的方法全写到元表里，并将元表保存到_G全局表和register表里，key都是类名称。然后就是往这个元表里填充信息，会将一个闭包给赋值__index字段，闭包的内容就是优先从类中找反射出来的属性或者方法找不到就是父类中找。
> 第二部分就是lua通过userdata来表示UObject对象，c++创建userdata然后会通过UObject的名称从registry表里找对应元表，找到后就设置userdata的元表。并将userdata保存到registry表中，key为UObject对应的lightuserdata。并将UObject和userdata映射保存到管理类中，管理类重写了GC方法，在c++GC的时候找到UObject对应的lightuserdata，进而在registry表中清掉userdata。
> 第三部分就是UObject和lua文件相关联使lua能够重写蓝图方法，需要在UObject蓝图中填上lua文件路径。然后会在SpawnActor的流程里面调用关联的方法InitLuaActor，在方法中首先通过require加载lua文件并将返回值保存到registry中，然后是遍历UClass中的UFunction，在lua文件中找同名的函数，清空原来UFunction上的字节码，然后替换成lua函数的。
# lua中按步骤执行

## 1 Coroutine

```lua
local function main()
    local index = 0
    while true do
        index = index + 1
        if index == 1 then
            print("打开冰箱门")
        elseif index == 2 then
            print("把大象放进去")
        elseif index == 3 then
            print("关上冰箱门")
        end
        coroutine.yield()
    end
end

local co = coroutine.create(main)
coroutine.resume(co)
coroutine.resume(co)
coroutine.resume(co)

```
## 2 Closure

```lua
local function closureType(func)  
    local index = 0  
    local function closureType_inner()  
        index = index + 1  
        func(index)  
    end  
    return closureType_inner  
end

local task = closureType(function(index)
    if index == 1 then
        print("打开冰箱门")
    elseif index == 2 then
        print("把大象放进去")
    elseif index == 3 then
        print("关上冰箱门")
    end
end)

task()
task()
task()
```

# 肉鸽
## 弱引用的使用
https://www.cnblogs.com/sifenkesi/p/3850760.html
注意：只有拥有显示构造的对象类型会被自动从weak表中移除，值类型boolean、number是不会自动从weak中移除的。而string类型虽然也由gc来负责清理，但是string没有显示的构造过程，因此也不会自动从weak表中移除，对于string的内存管理有单独的策略。

```lua
-- 创建一表，默认的表是强引用的，这个表里有3个属性
strongTable = {}
strongTable[1] = function() print("强引用表1") end
strongTable[2] = function() print("强引用表2") end
strongTable[3] = {1,2,3}

do
	print(#strongTable) -- 3，表有3个数据
	collectgarbage()
	print(#strongTable) -- 3，gc后表依然还有3个数据
end
setmetatable(strongTable, {__mode = "v"}) --设置表为弱引用表，value是弱引用
do 
	--print(#strongTable) -- 3，表有3个数据
	--collectgarbage()
	--print(#strongTable) -- 0，弱表情况下，gc后表中就没数据了
end
do
	globalref = strongTable[1] -- 引用一个弱表中的数据
	print(#strongTable) -- 3，表有3个数据
	collectgarbage()
	print(#strongTable) -- 1，弱表情况下，gc后表中还有一个被引用的数据
end

```
1 应用场景，怪物上有个属性需要从0慢慢增长到100。服务器是每1s同步一下属性值，我这就需要通过速度和时间来模拟这个属性，就需要额外的变量，然后我创建了一个弱表，key是怪物，value就是额外的这些变量。
## 关卡加载

```cpp
// 游戏中进入副本会通过StreamLevel的方法是加载，
```

![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202501241038250.png)
1 肉鸽第一次进入的主关卡包含了三个AlwaysLoad的子关卡（光照地图，navmesh地图，主房间也就是中间房间）和两个手动加载的子关卡（开始房间，结束房间，开始房间和结束房间应该完全一致）。
2 我们在进入副本的时候会受到self_enter_scene消息，然后会LoadStreamingLevel来加载主地图，主地图加载完成后，通过Package来找到AlwaysLoad子关卡StreamingLevel，然后再通过子关卡的package来找到路径，进而再加载。
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202501241051721.png)
3 下面就会加载rogue配置中的关卡，开始房间和结束房间等，这边开始房间和结束房间在第二步已经加载了，这里就会设置bVisible标志位让他显示出来。
4 收到NT_ROGUE_INIT，进入肉鸽的消息后，我们就会判断上述的这些场景是否加载完，加载完了，就会设置门的状态（服务器设置的场景参数），从场景中找门就是遍历场景actor，看哪个实现了接口。
5 切换下一关是无缝切换，首先会收到NT_READY_NEXT_NODE这个消息，首先将玩家的移动tick关掉然后teleport到开始房间（每一关的开始房间都是一样的），然后卸载旧结束房间->加载新的结束房间->加载新的主房间->卸载旧的主房间，然后才会收到SelfEnterScene正常进入场景的消息。