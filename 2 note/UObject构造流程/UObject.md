深入理解UObject的构造流程
https://blog.csdn.net/hacning/article/details/133614424
UObject解析
https://zhuanlan.zhihu.com/p/140339299
基础概念
https://zhuanlan.zhihu.com/p/529711999



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
```
3 在LoadPackageInternal方法中主要就是两个部分，第一部分是创建LinkerLoad序列化蓝图资源并加载以来，然后是
```cpp
FLinkerLoad* FLinkerLoad::CreateLinker(...)
{
	// 创建LinkerLoad
	FLinkerLoad* Linker = CreateLinkerAsync(LoadContext, Parent, PackagePath, LoadFlags, InstancingContext,TFunction<void()>([](){}));
	// 具体执行，里边就两个部分，一个是将蓝图序列化到内存中，一个是加载外部依赖
	Linker->Tick(0.f, false, false, nullptr);
}
FLinkerLoad::ELinkerStatus FLinkerLoad::Tick(...)
{
	// 1 创建序列化工具类，用于将硬盘中蓝图资源序列化到内存当中
	CreateLoader(TFunction<void()>([]() {}));
	// 2 具体的序列化内容，ImportMap，ExpoertMap等等，这里也只会将依赖的那些信息加载到内存中，但是实际的object还没加载，会在序列化的最一步FinalizeCreation中将依赖的object加载到内存
	ProcessPackageSummary(ObjectNameWithOuterToExportMap);
}

// 下面就开始加载依赖的Object了
FLinkerLoad::ELinkerStatus FLinkerLoad::FinalizeCreation(...)
{
	Verify();
}
void FLinkerLoad::Verify()
{
	for (int32 ImportIndex = 0; ImportIndex < Summary.ImportCount; ImportIndex++)
	{
		FObjectImport& Import = ImportMap[ImportIndex];
		VerifyImport( ImportIndex );
	}
}
FLinkerLoad::EVerifyResult FLinkerLoad::VerifyImport(int32 ImportIndex)
{
	FObjectImport& Import = ImportMap[ImportIndex];
	// 没有outer的情况下，className也不是package。有问题应该舍弃掉
	if (Import.OuterIndex.IsNull() && Import.ClassName!=NAME_Package )
	{
		return false;
	}
	// 没有outer,className是package。就说明import了一个package，直接加载package就行
	else if (Import.OuterIndex.IsNull())
	{
		TmpPkg = LoadImportPackage(Import, SlowTask);
	}
	// 有outer的情况下
	else
	{
		// 如果outer也在importmap里，那就递归Verify。
		if (Import.OuterIndex.IsImport())
		{
			VerifyImport(Import.OuterIndex.ToImport());
			FObjectImport& OuterImport = Imp(Import.OuterIndex);
			// !OuterImport.SourceLinker 说明OuterImport是一个脚本package。XObject存在说明，OuterImport对应的package已经在内存中了
			if (!OuterImport.SourceLinker && OuterImport.XObject)
			{
				FObjectImport* Top;
				for (Top = &OuterImport; Top->OuterIndex.IsImport(); Top = &Imp(Top->OuterIndex))
				{
				}
				// 把最上层的package赋值给TmpPkg
				UPackage* Package = Cast<UPackage>(Top->XObject);
				TmpPkg = Package;
			}
			// 如果没有SourceLinker，就用outer的SourceLinker，outer如果是脚本package，也是没有SourceLinker的
			if (!Import.SourceLinker)
			{
				Import.SourceLinker = OuterImport.SourceLinker;
			}
		}
	}
	bool bCameFromMemoryOnlyPackage = false;
	// Pkg表示要想获取import的Object，从哪个package里找。TmpPkg表示了最上层的package。
	if (!Pkg && TmpPkg &&
		(TmpPkg->HasAnyPackageFlags(PKG_InMemoryOnly) || IsContextInstanced()))
	{
		Pkg = TmpPkg;
		bCameFromMemoryOnlyPackage = true;
	}
	// 如果有Pkg，就能在Pkg里按照名字找到Object
	const bool bFindObjectByName = (Pkg == nullptr) && ((LoadFlags & LOAD_FindIfFail) != 0);
	// Import.SourceIndex表示Object的ExportMap索引
	if( Import.SourceIndex==INDEX_NONE && (Pkg!=nullptr || bFindObjectByName))
	{
		TStringBuilder<256> ImportClassTemp;
		Import.ClassPackage.ToString(ImportClassTemp);
		// 从全局中找到ClassPackage代表的package
		UObject* ClassPackage = FindObject<UPackage>( nullptr, *ImportClassTemp );
		// 从Package中找到ClassName代表的UClass
		Import.ClassName.ToString(ImportClassTemp);
		UClass* FindClass = FindObject<UClass>(ClassPackage, *ImportClassTemp);
		// 从Package中找到名称为ObjectName这个的UObject
		UObject* FindObject = FindImportFast(FindClass, bFindObjectByName ? nullptr : FindOuter, Import.ObjectName, bFindObjectByName);
		if (FindObject != nullptr)
		{
			Import.XObject = FindObject;
		}
	}
}
```
4 先看第一部分，序列化蓝图资源是在CreateLoader方法中，加载依赖则是在ProcessPackageSummary方法中。
5 

NewObject

