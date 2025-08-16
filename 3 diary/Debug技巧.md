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