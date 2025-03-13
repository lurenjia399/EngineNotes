深入理解UObject的构造流程
https://blog.csdn.net/hacning/article/details/133614424
UObject解析
https://zhuanlan.zhihu.com/p/140339299
基础概念
https://zhuanlan.zhihu.com/p/529711999
UE4资源加载
https://blog.csdn.net/qq_43034470/article/details/120888095
UE4的资源管理
https://zhuanlan.zhihu.com/p/357904199


查看蓝图资源格式：
资产既然可以被序列化成文件保存在硬盘，那它是以怎样的格式存储对象呢？打开 Edit Preferences , General->Experimental->Core->Text Asset Format Support (勾选)
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
	// 在GetPackageLinker方法中，会通过NewObject创建了一个空的UPackage，还会创建LinkerLoad，通过LinkerLoad加载资源中的ImportMap依赖的Object
	Linker = GetPackageLinker(InOuter, PackagePath, LoadFlags, nullptr, InReaderOverride, &InOutLoadContext, ImportLinker, InstancingContext);
	// 加载资源中的ExportMap
	Linker->LoadAllObjects(GEventDrivenLoaderEnabled);
	// 标记package的flag为加载完成
	if (!bFullyLoadSkipped)
	{
		Result->SetFlags(RF_WasLoaded);
	}
	return Result;
}
```
3 在LoadPackageInternal方法中主要就是两个部分，第一部分是创建LinkerLoad序列化蓝图资源并加载依赖，然后是加载创建ExportObject
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
		// 最后将找到的Object赋值
		if (FindObject != nullptr)
		{
			Import.XObject = FindObject;
		}
	}
}
```
4.1 先看第一部分，序列化蓝图资源是在CreateLoader方法中，加载依赖的Object则是在ProcessPackageSummary方法中。
有这么几种情况：import依赖的Object没有outer，那么依赖的肯定是个package，直接加载package即可，加载出来的package就是依赖的Object。如果import依赖的Object有outer，就先认证（Verify）outer，因为outer的最上层肯定是package，就从package中找到依赖的Object即可。总的来说就是找到依赖的Object，如果依赖的package就直接加载，如果不是就从package中找到依赖的Object。
```cpp
void FLinkerLoad::LoadAllObjects(bool bForcePreload)
{
	// 加载创建ExportObject
	for(int32 ExportIndex = 0; ExportIndex < ExportMap.Num(); ++ExportIndex)
	{
		UObject* LoadedObject = CreateExportAndPreload(ExportIndex, bForcePreload);
	}
	// 设置一个标志位，标记package里面的ExportObject都加载完了
	if( LinkerRoot )
	{
		LinkerRoot->MarkAsFullyLoaded();
	}
}
UObject* FLinkerLoad::CreateExportAndPreload(int32 ExportIndex, bool bForcePreload)
{
	// 创建或找到了ExportObject
	UObject* Object = CreateExport(ExportIndex);
	// 如果ExportObject是UClass类型就从蓝图资源中找数据序列化到UClass里,看上去是这样？
	if (Object && (bForcePreload || dynamic_cast<UClass*>(Object) || Object->IsTemplate() || dynamic_cast<UObjectRedirector*>(Object)))
	{
		Preload(Object);
	}

	return Object;
}
UObject* FLinkerLoad::CreateExport( int32 Index )
{
	// 获取ExportObject的UClass。这个Class肯定是在importMap依赖里的，就从对应的package中找出来就行。
	UClass* LoadClass = GetExportLoadClass(Index);
	// 获取ExportObject的UClass的CDO
	UObject* Template = UObject::GetArchetypeFromRequiredInfo(LoadClass, ThisParent, Export.ObjectName, Export.ObjectFlags);
	// 从ThisParent，也就是从this->LinkerRoot(package)中找下ExportObject
	UObject* ActualObjectWithTheName = StaticFindObjectFastInternal(nullptr, ThisParent, Export.ObjectName, true);
	// 如果找见了ExportObject，就返回
	if (ActualObjectWithTheName && (ActualObjectWithTheName->GetClass() == LoadClass))
	{
		Export.Object = ActualObjectWithTheName;
		Export.Object->SetLinker(this, Index);
		return Export.Object;
	}
	// 创建出ExportObject的UClass的CDO
	LoadClass->GetDefaultObject();
	// 根据outer,uclass,name来创建ExportObject
	Export.Object = StaticConstructObject_Internal(Params);
	// 设置linker
	Export.Object->SetLinker( this, Index );
	// 添加ExportObject
	CurrentLoadContext->AddLoadedObject(Export.Object);
}
```
4.2 第二部分，加载ExportMap中的Object，根据outer，ucalss, name来创建出来。
# 2 FindObject，FindObjectFast，FindObjectChecked，FindObjectSafe

``` cpp
UObject* StaticFindObjectFastInternalThreadSafe(...)
{
	// 一般都是由packet的，如果调用时没提供package，就会自己找package
	if (ObjectPackage != nullptr)
	{
		// 通过package获取hash
		int32 Hash = GetObjectOuterHash(ObjectName, (PTRINT)ObjectPackage);
		// 这里给HashTable上锁了，所以是线程安全的
		FHashTableLock HashLock(ThreadHash);
		// 遍历
		for (TMultiMap<int32, uint32>::TConstKeyIterator HashIt(ThreadHash.HashOuter, Hash); HashIt; ++HashIt)
		{
			// 拿到正在遍历的Object
			uint32 InternalIndex = HashIt.Value();
			UObject* Object = static_cast<UObject*>(GUObjectArray.IndexToObject(InternalIndex)->Object);
			if
				// 遍历Object名称得和ObjectName一致
				((Object->GetFName() == ObjectName)
				// 遍历Object不能带有ExcludeFlags这个表示的标志位
				&& !Object->HasAnyFlags(ExcludeFlags)
				// 遍历Object的outer得是我们的Package
				&& Object->GetOuter() == ObjectPackage
				// 遍历Object的UClass得和所需的一致
				&& (ObjectClass == nullptr || (bExactClass ? Object->GetClass() == ObjectClass : Object->IsA(ObjectClass)))
				// 遍历Object不能带有某个标志位
				&& !Object->HasAnyInternalFlags(ExclusiveInternalFlags))
			{
				if (Result)
				{
				}
				else
				{
					// if条件都满足，就算是找到了
					Result = Object;
				}
			}
		}
	}
	// 没有package的情况下
	else
	{
	}
}
```
这个几个接口最终执行都是这个StaticFindObjectFastInternalThreadSafe方法，只是有些条件判断吧，分成了这些接口
# 3 TStrongObjectPtr，TWeakObjectPtr，TSoftClassPtr


# 问题

- staticloadobject流程？
内部执行loadpackage方法，首先会创建一个空的upackage，然后创建linkerload来加载蓝图资源，通过对蓝图资源中的各种段进行解析，比如importmap来加载依赖（一般都是写uclass），exportmap来加载可以被其他资源依赖的object。通过staticloadobjet来加载蓝图，会返回蓝图类也就是blueprintgeneratedclass，所在的package名称是蓝图路径。

- UPackage是什么？
UPackage是硬盘上的蓝图资源在内存上的对应。举个例子，我们手上有一个名称为TestA的空蓝图继承自Actor。我们通过StaticLoadObject方法来加载TestA蓝图，首先就会创建一个空的Package，然后为这个package创建LinkerLoad，然后通过LinkerLoad来序列化蓝图资源，进而加载importMap和exportMap。除此之外我们通过SpawnActor来创建TestA的对象，对象的outer是level，level的outer是world，world的outer是UPackage，这也说明了每个UObject的最上层都是一个UPackage，决定了UObject会序列化到内存的哪个地方。
- UArchetype到底是啥？





