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