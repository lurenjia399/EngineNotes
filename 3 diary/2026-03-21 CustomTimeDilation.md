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