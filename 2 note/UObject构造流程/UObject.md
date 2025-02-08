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
	// InOuter由ResoveName创建出
	if (InOuter)  
	{
		// 从package中找Object
		Result = StaticFindObjectFast(ObjectClass, InOuter, *StrName);
	}
	if (Result && UE::GC::GIsIncrementalReachabilityPending)  
	{  
	    UE::GC::MarkAsReachable(Result);  
	}
	return Result;
}
```
1 调用StaticLoadObjectInternal方法，这个方法主要就是两部分，一部分加载package，一部分通过package找到Object。
2 ResolveName方法就是加载package。如果不是内置package，就通过loadPackage方法加载。
```cpp
UPackage* LoadPackage(...)
{
	return LoadPackageInternal(InOuter, PackagePath, LoadFlags, nullptr, InReaderOverride, InstancingContext, DiffPackagePath);
}

UPackage* LoadPackageInternal(...)
{
	FLinkerLoad* Linker = nullptr;
	// 在GetPackageLinker方法中，会通过NewObject创建了一个空的UPackage，还会创建LinkerLoad
	Linker = GetPackageLinker(InOuter, PackagePath, LoadFlags, nullptr, InReaderOverride, &InOutLoadContext, ImportLinker, InstancingContext);

	Linker->LoadAllObjects(GEventDrivenLoaderEnabled);
	
	if (!bFullyLoadSkipped)
	{
		// Mark package as loaded.
		Result->SetFlags(RF_WasLoaded);
	}
	return Result;
}

FLinkerLoad* FLinkerLoad::CreateLinker(...)
{
	// 创建LinkerLoad
	FLinkerLoad* Linker = CreateLinkerAsync(LoadContext, Parent, PackagePath, LoadFlags, InstancingContext,TFunction<void()>([](){}));
	// 具体执行
	Linker->Tick(0.f, false, false, nullptr)
}
```
3 
NewObject

