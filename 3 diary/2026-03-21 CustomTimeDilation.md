1 停止WP的流送
```cpp
/*
static IConsoleVariable* CVar = IConsoleManager::Get()
	.FindConsoleVariable(TEXT("wp.Runtime.UpdateStreamingSources"));  
if (CVar)  
{  
    CVar->Set(ForceCloseUpdateStreamingSources ==0, ECVF_SetByCode);  
}

*/
```
2 剩下就是通过位操作，记录暂停类型，然后广播出去
3 不同的actor监听，然后修改自己的CustomTimeDilation属性，把tick间隔变得很小