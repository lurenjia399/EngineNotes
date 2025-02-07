深入理解UObject的构造流程
https://blog.csdn.net/hacning/article/details/133614424
UObject解析
https://zhuanlan.zhihu.com/p/140339299



# 1 StaticLoadObject

```cpp
UObject* StaticLoadObject(...)
{
	// 核心方法 StaticLoadObjectInternal
	UObject* Result = StaticLoadObjectInternal(ObjectClass, InOuter, InName, Filename, LoadFlags, Sandbox, bAllowObjectReconciliation, InstancingContext);
	
	if (!Result)
	{
		// 有问题，没等加载成功Object
	}
}
```

```cpp
UObject* StaticLoadObjectInternal(UObject*& InPackage, FString& InOutName, bool Create, bool Throw, uint32 LoadFlags /*= LOAD_None*/, const FLinkerInstancingContext* InstancingContext)
{
	// ResolveName方法
	ResolveName(InOuter, StrName, true, true, LoadFlags & (LOAD_EditorOnly | LOAD_NoVerify | LOAD_Quiet | LOAD_NoWarn | LOAD_DeferDependencyLoads), InstancingContext);
	// InOuter由ResoveName创建出，蓝图package的话是空的
	if (InOuter)  
	{
		// 从package中找Object，因为是空的找不见
		Result = StaticFindObjectFast(ObjectClass, InOuter, *StrName);
		if (!Result)
		{
			if (!InOuter->GetOutermost()->HasAnyPackageFlags(PKG_CompiledIn))
			{
				LoadPackage(NULL, *InOuter->GetOutermost()->GetName(), LoadFlags & ~LOAD_Verify, nullptr, InstancingContext);
			}
			Result = StaticFindObjectFast(ObjectClass, InOuter, *StrName);
		}
	}
	if (Result && UE::GC::GIsIncrementalReachabilityPending)  
	{  
	    UE::GC::MarkAsReachable(Result);  
	}
	return Result;
}
```
1 调用StaticLoadObjectInternal方法，在这个方法里
2 ResolveName方法，拿到所需资源的package。如果不是蓝图package，就通过loadPackage方法加载，如果是蓝图package，就创建新的package并设置flags为PKG_CompiledIn。

NewObject

