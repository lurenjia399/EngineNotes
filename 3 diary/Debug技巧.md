1
```cpp
/*
if (!ensureMsgf(IsInGameThread(), TEXT("Calling %hs is supported only on the Game Tread"), __FUNCTION__))
{
	return;
}
这样直接弹
*/
```
2 
```cpp
/*
{,,UE4Editor-Engine.dll}::GPlayInEditorContextString {,,UnrealEditor-Engine.dll}::GPlayInEditorContextString

把 {,,UnrealEditor-Engine.dll}::GPlayInEditorContextString 添加到rider watch即可,以后每次能用。
UE4用这个 {,,UE4Editor-Engine.dll}::GPlayInEditorContextString
*/
```
3 
```cpp
/*
调试Dump
https://www.jetbrains.com/help/clion/core-dump-debug.html#process

1 Select Run | Debug Core Dump from the main menu
2 选择Dump文件
3 选择项目生成的exe文件
*/
```
4 蓝图的csc面板显示不了。点Window选loadLayout，选ue4经典布局，就能显示出来了

5 C++打印堆栈
```cpp
/*
//Debug Log输出：  
FString StackInfo = FString::Printf(TEXT("Stack：[%u]"),GFrameNumber);  
TArray<FProgramCounterSymbolInfo> stacks = FPlatformStackWalk::GetStack(1,10);  
for (int i = 0;i < stacks.Num();++i)  
{  
    StackInfo += FString("\r\n\t")+FString::Printf(TEXT("%s:%d"),ANSI_TO_TCHAR(stacks[i].FunctionName), stacks[i].LineNumber);  
}  
if (DrivableVehicle)  
{  
    UE_LOG(LogHTVehicle, Warning, TEXT("Authority Start Get Off Vehicle! Player:[%s],VehicleID:[%s],\r\n\t %s"), *GetName(), *(DrivableVehicle->VehicleID.ToString()),*StackInfo);  
}  
else  
{  
    UE_LOG(LogHTVehicle, Warning, TEXT("Authority Start Get Off Vehicle! Player:[%s],VehicleID:NULL,\r\n\t %s"), *GetName(),*StackInfo);  
}
*/
// 这个也可以
FDebug::DumpStackTraceToLog
```

6
```cpp
/*
static FAutoConsoleCommandWithOutputDevice GDumpCachedObjectsCmd(
	TEXT("HTObjectPool.DumpCachedObjects"),
	TEXT("Dumps object pool info to the log"),
		FConsoleCommandWithOutputDeviceDelegate::CreateStatic([](FOutputDevice& OutputDevice)
		{
			for (const FWorldContext& Context : GEngine->GetWorldContexts())
			{
				UWorld* World = Context.World();
				if (World && World->IsGameWorld())
				{
					if (const UHTObjectPoolSubsystem* ObjectPoolSubsystem = World->GetSubsystem<UHTObjectPoolSubsystem>())
					{				
						ObjectPoolSubsystem->DumpCachedObjects(OutputDevice);
					}
				}
			}
		})
	);
*/
```
7 直接打印玩家位置
```cpp
// cheat PrintObjectProperty Player.RootComponent.RelativeRotation
// PrintObjectProperty Player.RootComponent.RelativeRotation
```
8 改变log级别
```cpp
// cheat Log LogHTAI Log
```
9 定义log
```cpp
// DEFINE_LOG_CATEGORY_STATIC(LogVehicleAutopilot, Log, All)
```