# 1 测试用例

MyTest.h 文件
```cpp

#pragma once

#include "CoreMinimal.h"
#include "UObject/Object.h"
#include "MyTest.generated.h"

/**
 * 
 */
UCLASS()
class AZURE_API UMyTest : public UObject
{
	GENERATED_BODY()
	public:
	UMyTest();

	UFUNCTION()
	int AddFunc(int a, int b);
};

```

MyTest.cpp文件
```cpp
#include "MyTest.h"

UMyTest::UMyTest()
{
}

int UMyTest::AddFunc(int a, int b)
{
	return a + b;
}
```
# 2 展开宏

## MyTest.h
```cpp
class AZURE_API UMyTest : public UObject
{
public:
	static void execAddFunc( UObject* Context, FFrame& Stack, RESULT_DECL ); // 我们添加的方法
private: 
	static void StaticRegisterNativesUMyTest(); 		// 注册我们的UMyTest类
	friend struct Z_Construct_UClass_UMyTest_Statics; 	// 将这个声明为我们UMyTest类的友元类
public: 
private:
	UMyTest& operator=(UMyTest&&);   					// 禁用赋值构造函数，移动版本
	UMyTest& operator=(const UMyTest&);   				// 禁用赋值构造函数，普通版本
	NO_API static UClass* GetPrivateStaticClass(); 		// 真正内部生成的UClass函数
public: 
	/** Bitwise union of #EClassFlags pertaining to this class.*/ 
	enum {StaticClassFlags=COMPILED_IN_FLAGS(0)}; 
	/** Typedef for the base class ({{ typedef-type }}) */ 
	typedef UObject Super;
	/** Typedef for {{ typedef-type }}. */ 
	typedef UMyTest ThisClass;
	/** Returns a UClass object representing this class at runtime */ 
	inline static UClass* StaticClass() 				// 获取这个类对应的UClass
	{ 
		return GetPrivateStaticClass(); 
	} 
	/** Returns the package this class belongs in */ 
	inline static const TCHAR* StaticPackage() 
	{ 
		return TEXT("/Script/Azure"); 
	} 
	/** Returns the static cast flags for this class */ 
	inline static EClassCastFlags StaticClassCastFlags() 
	{ 
		return CASTCLASS_None; 
	} 
	/** For internal use only; use StaticConstructObject() to create new objects. */ 
	inline void* operator new(const size_t InSize, EInternal InInternalOnly, UObject* InOuter = (UObject*)GetTransientPackage(), FName InName = NAME_None, EObjectFlags InSetFlags = RF_NoFlags) 
	{ // 重载普通的new，然后进行内存分配
		return StaticAllocateObject(StaticClass(), InOuter, InName, InSetFlags); 
	} 
	/** For internal use only; use StaticConstructObject() to create new objects. */ 
	inline void* operator new( const size_t InSize, EInternal* InMem ) 
	{ // 重载placement new，不进行内存分配
		return (void*)InMem; 
	}
	friend FArchive &operator<<( FArchive& Ar, UMyTest*& Res ) 
	{ // 序列化相关，参数不同
		return Ar << (UObject*&)Res; 
	} 
	friend void operator<<(FStructuredArchive::FSlot InSlot, UMyTest*& Res) 
	{ // 序列化相关，参数不同
		InSlot << (UObject*&)Res; 
	}
private: 
	NO_API UMyTest(UMyTest&&); 							// 禁用构造函数，移动版本
	NO_API UMyTest(const UMyTest&); 					// 禁用构造函数，普通版本
public:
	/** DO NOT USE. This constructor is for internal usage only for hot-reload purposes. */
	NO_API UMyTest(FVTableHelper& Helper); 				// 构造函数
	static UObject* __VTableCtorCaller(FVTableHelper& Helper) // 这部分热更，先忽略掉
	{ 
		return new (EC_InternalUseOnlyConstructor, (UObject*)GetTransientPackage(), NAME_None, RF_NeedLoad | 					             		RF_ClassDefaultObject | RF_TagGarbageTemp) UMyTest(Helper); 
	}
	static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())UMyTest; }// 对构造函数和new重载的调用
	
public:
	UMyTest(); //我自己写的构造函数

	UFUNCTION()
	int AddFunc(int a, int b); //添加的方法
};
/*
	这部分是咱们的展开过程:
	
 	1 展开 GENERATED_BODY()
 		{
			#define BODY_MACRO_COMBINE_INNER(A,B,C,D) A##B##C##D
			#define BODY_MACRO_COMBINE(A,B,C,D) BODY_MACRO_COMBINE_INNER(A,B,C,D)
			#define GENERATED_BODY(...) BODY_MACRO_COMBINE(CURRENT_FILE_ID,_,__LINE__,_GENERATED_BODY);
 		}
 		CURRENT_FILE_ID -> Azure_Source_Azure_MyTest_h
 		{
 			是在MyTest.generated.h中定义
 			#undef CURRENT_FILE_ID
 			#define CURRENT_FILE_ID Azure_Source_Azure_MyTest_h
 			表示的也是MyTest.h这个文件的路径
 		}
		__LINE__ -> 15
		{
			是GENERATED_BODY()这个宏所定义的行数，这里是15行
			目的是一个.h文件可以定义多个UClass类
		}
	  展开的结果 -> Azure_Source_Azure_MyTest_h_15_GENERATED_BODY
	2 继续展开上面的结果(Azure_Source_Azure_MyTest_h_15_GENERATED_BODY)
		在MyTest.generated.h中找到对应宏的定义:
		{
			#define Azure_Source_Azure_MyTest_h_15_GENERATED_BODY \
			public: \
				Azure_Source_Azure_MyTest_h_15_PRIVATE_PROPERTY_OFFSET \
				Azure_Source_Azure_MyTest_h_15_SPARSE_DATA \
				Azure_Source_Azure_MyTest_h_15_RPC_WRAPPERS_NO_PURE_DECLS \
				Azure_Source_Azure_MyTest_h_15_INCLASS_NO_PURE_DECLS \
				Azure_Source_Azure_MyTest_h_15_ENHANCED_CONSTRUCTORS \
			忽略掉了其中的 PRAGMA_ENABLE_DEPRECATION_WARNINGS 这个
			依次来看其中的宏：
			Azure_Source_Azure_MyTest_h_15_PRIVATE_PROPERTY_OFFSET -> 空值，因为没有添加一个private的成员变量
			Azure_Source_Azure_MyTest_h_15_SPARSE_DATA -> 空值
			Azure_Source_Azure_MyTest_h_15_RPC_WRAPPERS_NO_PURE_DECLS -> DECLARE_FUNCTION(execAddFunc) 声明了FUNCTION
			{
				#define Azure_Source_Azure_MyTest_h_15_RPC_WRAPPERS_NO_PURE_DECLS \
				 \
					DECLARE_FUNCTION(execAddFunc);
			}
			Azure_Source_Azure_MyTest_h_15_INCLASS_NO_PURE_DECLS -> 定义了这么些东西
			{
				#define Azure_Source_Azure_MyTest_h_15_INCLASS_NO_PURE_DECLS \
				private: \
					static void StaticRegisterNativesUMyTest(); \
					friend struct Z_Construct_UClass_UMyTest_Statics; \
				public: \
					DECLARE_CLASS(UMyTest, UObject, COMPILED_IN_FLAGS(0), CASTCLASS_None, TEXT("/Script/Azure"), NO_API) \
					DECLARE_SERIALIZER(UMyTest)
			}
			Azure_Source_Azure_MyTest_h_15_ENHANCED_CONSTRUCTORS -> 定义了这么些东西
			{
				#define Azure_Source_Azure_MyTest_h_15_ENHANCED_CONSTRUCTORS \
				private: \
					NO_API UMyTest(UMyTest&&); \
					NO_API UMyTest(const UMyTest&); \
				public: \
					DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyTest); \
					DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyTest); \
					DEFINE_DEFAULT_CONSTRUCTOR_CALL(UMyTest)
			}
		}
	  展开的结果 ->
	  {
		public:
			DECLARE_FUNCTION(execAddFunc);
		private: 
			static void StaticRegisterNativesUMyTest(); 
			friend struct Z_Construct_UClass_UMyTest_Statics; 
		public: 
			DECLARE_CLASS(UMyTest, UObject, COMPILED_IN_FLAGS(0), CASTCLASS_None, TEXT("/Script/Azure"), NO_API) 
			DECLARE_SERIALIZER(UMyTest)
		private: 
			NO_API UMyTest(UMyTest&&); 
			NO_API UMyTest(const UMyTest&); 
		public: 
			DECLARE_VTABLE_PTR_HELPER_CTOR(NO_API, UMyTest); 
			DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER(UMyTest); 
			DEFINE_DEFAULT_CONSTRUCTOR_CALL(UMyTest)
	  }
	3 下面展(DECLARE_FUNCTION) -> static void execAddFunc( UObject* Context, FFrame& Stack, RESULT_DECL )
	{
		#define DECLARE_FUNCTION(func) static void func( UObject* Context, FFrame& Stack, RESULT_DECL )
	}
	3 下面展开(DECLARE_CLASS)
	{
		内容很多不详细写了，就写参数替换
		TClass -> UMyTest
		TSuperClass -> UObject
		TStaticFlags -> COMPILED_IN_FLAGS(0)
		TStaticCastFlags -> CASTCLASS_None
		TPackage -> TEXT("/Script/Azure")
		TRequiredAPI -> NO_API
	}
	4 下面展开(DECLARE_SERIALIZER)
	{
		序列化部分
		参数是 TClass -> UMyTest
		#define DECLARE_SERIALIZER( TClass ) \
		friend FArchive &operator<<( FArchive& Ar, TClass*& Res ) \
		{ \
			return Ar << (UObject*&)Res; \
		} \
		friend void operator<<(FStructuredArchive::FSlot InSlot, TClass*& Res) \
		{ \
			InSlot << (UObject*&)Res; \
		}
	}
	5 下面展开(DECLARE_VTABLE_PTR_HELPER_CTOR) -> NO_API UMyTest(FVTableHelper& Helper);
	{
		#define DECLARE_VTABLE_PTR_HELPER_CTOR(API, TClass) \
		API TClass(FVTableHelper& Helper);
	}
	6 下面展开(DEFINE_VTABLE_PTR_HELPER_CTOR_CALLER)
	{
		static UObject* __VTableCtorCaller(FVTableHelper& Helper) // 这部分热更，先忽略掉，就不替换了
        { 
            return new (EC_InternalUseOnlyConstructor, (UObject*)GetTransientPackage(), NAME_None, RF_NeedLoad | 	 	                 				RF_ClassDefaultObject | RF_TagGarbageTemp) UMyTest(Helper); 
        }
	}
	7 下面展开(DEFINE_DEFAULT_CONSTRUCTOR_CALL)
	{
		#define DEFINE_DEFAULT_CONSTRUCTOR_CALL(TClass) \
		static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass; }
	}
 */
```
## MyTest.cpp
```cpp
#include "MyTest.h"
// ----------------------这里是我添加的展开 MyTest.gen.cpp 开始


void EmptyLinkFunctionForGeneratedCodeMyTest() {}
// Cross Module References
AZURE_API UClass* Z_Construct_UClass_UMyTest_NoRegister();
AZURE_API UClass* Z_Construct_UClass_UMyTest();
COREUOBJECT_API UClass* Z_Construct_UClass_UObject();
UPackage* Z_Construct_UPackage__Script_Azure();
// End Cross Module References
void UMyTest::execAddFunc( UObject* Context, FFrame& Stack, RESULT_DECL )
{//这个应该就是我们写的那个方法，.h文件中的实现
	P_GET_PROPERTY(FIntProperty,Z_Param_a);
	P_GET_PROPERTY(FIntProperty,Z_Param_b);
	P_FINISH;
	P_NATIVE_BEGIN;
	*(int32*)Z_Param__Result=P_THIS->AddFunc(Z_Param_a,Z_Param_b);
	P_NATIVE_END;
}
void UMyTest::StaticRegisterNativesUMyTest()
{//注册我们的UMyTest类，.h文件中的实现
	UClass* Class = UMyTest::StaticClass();
	static const FNameNativePtrPair Funcs[] = {
		{ "AddFunc", &UMyTest::execAddFunc },
	};
	FNativeFunctionRegistrar::RegisterFunctions(Class, Funcs, UE_ARRAY_COUNT(Funcs));
}
// 这里应该也是我们的那个function，导致添加的类-----------------开始
// 这个是Z_Construct_UFunction_UMyTest_AddFunc_Statics结构体的初始化，里面有静态成员变量，所以也需要初始化，这里是开始
struct Z_Construct_UFunction_UMyTest_AddFunc_Statics
{
	struct MyTest_eventAddFunc_Parms
	{
		int32 a;
		int32 b;
		int32 ReturnValue;
	};
	static const UE4CodeGen_Private::FUnsizedIntPropertyParams NewProp_a;
	static const UE4CodeGen_Private::FUnsizedIntPropertyParams NewProp_b;
	static const UE4CodeGen_Private::FUnsizedIntPropertyParams NewProp_ReturnValue;
	static const UE4CodeGen_Private::FPropertyParamsBase* const PropPointers[];
#if WITH_METADATA
	static const UE4CodeGen_Private::FMetaDataPairParam Function_MetaDataParams[];
#endif
	static const UE4CodeGen_Private::FFunctionParams FuncParams;
};
	const UE4CodeGen_Private::FUnsizedIntPropertyParams Z_Construct_UFunction_UMyTest_AddFunc_Statics::NewProp_a = { "a", 				nullptr, 			(EPropertyFlags)0x0010000000000080, UE4CodeGen_Private::EPropertyGenFlags::Int, 							RF_Public|RF_Transient|RF_MarkAsNative, 		1, 	STRUCT_OFFSET(MyTest_eventAddFunc_Parms, a), 								METADATA_PARAMS(nullptr, 0) };
	const UE4CodeGen_Private::FUnsizedIntPropertyParams Z_Construct_UFunction_UMyTest_AddFunc_Statics::NewProp_b = { "b", 				nullptr, 			(EPropertyFlags)0x0010000000000080, UE4CodeGen_Private::EPropertyGenFlags::Int, 							RF_Public|RF_Transient|RF_MarkAsNative, 		1, STRUCT_OFFSET(MyTest_eventAddFunc_Parms, b), METADATA_PARAMS(nullptr, 		0) };
	const UE4CodeGen_Private::FUnsizedIntPropertyParams Z_Construct_UFunction_UMyTest_AddFunc_Statics::NewProp_ReturnValue = { 				"ReturnValue", nullptr, (EPropertyFlags)0x0010000000000580, UE4CodeGen_Private::EPropertyGenFlags::Int, 						RF_Public|RF_Transient|RF_MarkAsNative, 1, STRUCT_OFFSET(MyTest_eventAddFunc_Parms, ReturnValue), 								METADATA_PARAMS(nullptr, 0) };
	const UE4CodeGen_Private::FPropertyParamsBase* const Z_Construct_UFunction_UMyTest_AddFunc_Statics::PropPointers[] = {
		(const UE4CodeGen_Private::FPropertyParamsBase*)&Z_Construct_UFunction_UMyTest_AddFunc_Statics::NewProp_a,
		(const UE4CodeGen_Private::FPropertyParamsBase*)&Z_Construct_UFunction_UMyTest_AddFunc_Statics::NewProp_b,
		(const UE4CodeGen_Private::FPropertyParamsBase*)&Z_Construct_UFunction_UMyTest_AddFunc_Statics::NewProp_ReturnValue,
		};
#if WITH_METADATA
	const UE4CodeGen_Private::FMetaDataPairParam Z_Construct_UFunction_UMyTest_AddFunc_Statics::Function_MetaDataParams[] = {
		{ "ModuleRelativePath", "MyTest.h" },
	};
#endif
	const UE4CodeGen_Private::FFunctionParams Z_Construct_UFunction_UMyTest_AddFunc_Statics::FuncParams = { 
        (UObject*(*)())Z_Construct_UClass_UMyTest, nullptr, "AddFunc", nullptr, nullptr, sizeof(MyTest_eventAddFunc_Parms), 			Z_Construct_UFunction_UMyTest_AddFunc_Statics::PropPointers, 										 	  						UE_ARRAY_COUNT(Z_Construct_UFunction_UMyTest_AddFunc_Statics::PropPointers), 
        RF_Public|RF_Transient|RF_MarkAsNative, (EFunctionFlags)0x00020401, 0, 0, 											 			METADATA_PARAMS(Z_Construct_UFunction_UMyTest_AddFunc_Statics::Function_MetaDataParams,  										UE_ARRAY_COUNT(Z_Construct_UFunction_UMyTest_AddFunc_Statics::Function_MetaDataParams)) };
// 这个是Z_Construct_UFunction_UMyTest_AddFunc_Statics结构体的初始化，这里是而结束

UFunction* Z_Construct_UFunction_UMyTest_AddFunc()
{
	static UFunction* ReturnFunction = nullptr;
	if (!ReturnFunction)
	{
		UE4CodeGen_Private::ConstructUFunction(ReturnFunction, Z_Construct_UFunction_UMyTest_AddFunc_Statics::FuncParams);
	}
	return ReturnFunction;
}
// 这里应该也是我们的那个function，导致添加的类-----------------结束

UClass* Z_Construct_UClass_UMyTest_NoRegister()
{
	return UMyTest::StaticClass();
}
	
// 这个是Z_Construct_UClass_UMyTest_Statics结构体的初始化，这里是开始
struct Z_Construct_UClass_UMyTest_Statics
{
	static UObject* (*const DependentSingletons[])();
	static const FClassFunctionLinkInfo FuncInfo[];
#if WITH_METADATA
	static const UE4CodeGen_Private::FMetaDataPairParam Class_MetaDataParams[];
#endif
	static const FCppClassTypeInfoStatic StaticCppClassTypeInfo;
	static const UE4CodeGen_Private::FClassParams ClassParams;
};
UObject* (*const Z_Construct_UClass_UMyTest_Statics::DependentSingletons[])() = {
	(UObject* (*)())Z_Construct_UClass_UObject,
	(UObject* (*)())Z_Construct_UPackage__Script_Azure,
};
const FClassFunctionLinkInfo Z_Construct_UClass_UMyTest_Statics::FuncInfo[] = {
	{ &Z_Construct_UFunction_UMyTest_AddFunc, "AddFunc" }, // 1255610792
};
#if WITH_METADATA
const UE4CodeGen_Private::FMetaDataPairParam Z_Construct_UClass_UMyTest_Statics::Class_MetaDataParams[] = {
	{ "Comment", "/**\n * \n */" },
	{ "IncludePath", "MyTest.h" },
	{ "ModuleRelativePath", "MyTest.h" },
};
#endif
const FCppClassTypeInfoStatic Z_Construct_UClass_UMyTest_Statics::StaticCppClassTypeInfo = {
	TCppClassTypeTraits<UMyTest>::IsAbstract,
};
const UE4CodeGen_Private::FClassParams Z_Construct_UClass_UMyTest_Statics::ClassParams = {
	&UMyTest::StaticClass,//这个值后面会讲到
	nullptr,
	&StaticCppClassTypeInfo,
	DependentSingletons,
	FuncInfo,
	nullptr,
	nullptr,
	UE_ARRAY_COUNT(DependentSingletons),
	UE_ARRAY_COUNT(FuncInfo),
	0,
	0,
	0x001000A0u,
	METADATA_PARAMS(Z_Construct_UClass_UMyTest_Statics::Class_MetaDataParams, UE_ARRAY_COUNT(Z_Construct_UClass_UMyTest_Statics::Class_MetaDataParams))
};
// 这个是Z_Construct_UClass_UMyTest_Statics结构体的初始化，这里是结束

//
UClass* Z_Construct_UClass_UMyTest()
{
	static UClass* OuterClass = nullptr;
	if (!OuterClass)
	{
		UE4CodeGen_Private::ConstructUClass(OuterClass, Z_Construct_UClass_UMyTest_Statics::ClassParams);
	}
	return OuterClass;
}

// StaticClass方法就会调用GetPrivateStaticClass这个方法，返回Mytest的UClass
UClass* UMyTest::GetPrivateStaticClass() 
{ 
	static UClass* PrivateStaticClass = NULL; 
	if (!PrivateStaticClass) 
	{ 
		/* this could be handled with templates, but we want it external to avoid code bloat */ 
		GetPrivateStaticClassBody( 
			StaticPackage(), 
			(TCHAR*)TEXT("UMyTest") + 1 + ((StaticClassFlags & CLASS_Deprecated) ? 11 : 0), 
			PrivateStaticClass, 
			StaticRegisterNativesUMyTest, 
			sizeof(UMyTest), 
			alignof(UMyTest), 
			(EClassFlags)UMyTest::StaticClassFlags, 
			UMyTest::StaticClassCastFlags(), 
			UMyTest::StaticConfigName(), 
			(UClass::ClassConstructorType)InternalConstructor<UMyTest>, // 构造函数，在cdo创建过程中调用，这个值后面会讲到
			(UClass::ClassVTableHelperCtorCallerType)InternalVTableHelperCtorCaller<UMyTest>, 
			&UMyTest::AddReferencedObjects, 
			&UMyTest::Super::StaticClass, 
			&UMyTest::WithinClass::StaticClass 
		); 
	} 
	return PrivateStaticClass; 
}
// 模板UMyTest，就是调用下UMyTest::StaticClass
template<> AZURE_API UClass* StaticClass<UMyTest>()
{
	return UMyTest::StaticClass();
}
// 与.h文件中一一对应
UMyTest::UMyTest(FVTableHelper& Helper) : Super(Helper) {};

// 这个静态变量用于收集UClass信息，后面详细看下
static TClassCompiledInDefer<UMyTest> AutoInitializeUMyTest(TEXT("UMyTest"), sizeof(UMyTest), 2889205096);
// 这个静态变量也用于收集UClass信息，后面详细看下
static FCompiledInDefer Z_CompiledInDefer_UClass_UMyTest(Z_Construct_UClass_UMyTest, &UMyTest::StaticClass, TEXT("/Script/Azure"), TEXT("UMyTest"), false, nullptr, nullptr, nullptr);

// -----------------------这里是我添加的展开 MyTest.gen.cpp 结束
UMyTest::UMyTest()
{
}

int UMyTest::AddFunc(int a, int b)
{
	return a + b;
}
```
### 收集UClass信息
```cpp
static TClassCompiledInDefer<UMyTest> AutoInitializeUMyTest(TEXT("UMyTest"), sizeof(UMyTest), 2889205096); 

static FCompiledInDefer Z_CompiledInDefer_UClass_UMyTest(Z_Construct_UClass_UMyTest, &UMyTest::StaticClass, TEXT("/Script/Azure"), TEXT("UMyTest"), false, nullptr, nullptr, nullptr);

/*
这有两个静态的成员变量，看下他们的初始化过程
*/
```
#### AutoInitializeUMyTest
```cpp
static TClassCompiledInDefer<UMyTest> AutoInitializeUMyTest(TEXT("UMyTest"), sizeof(UMyTest), 2889205096); 

template <typename TClass>
struct TClassCompiledInDefer : public FFieldCompiledInInfo
{
	TClassCompiledInDefer(const TCHAR* InName, SIZE_T InClassSize, uint32 InCrc)
	: FFieldCompiledInInfo(InClassSize, InCrc)
	{
		UClassCompiledInDefer(this, InName, InClassSize, InCrc); // 收集信息
	}
	virtual UClass* Register() const override
	{
        LLM_SCOPE(ELLMTag::UObject);
		return TClass::StaticClass();
	}
	virtual const TCHAR* ClassPackage() const override
	{
		return TClass::StaticPackage();
	}
};

void UClassCompiledInDefer(FFieldCompiledInInfo* ClassInfo, const TCHAR* Name, SIZE_T ClassSize, uint32 Crc)
{
	const FName CPPClassName = Name;
	// 最终添加到了DeferredClassRegistration里面
	GetDeferredClassRegistration().Add(ClassInfo); 
}

static TArray<FFieldCompiledInInfo*>& GetDeferredClassRegistration()
{
	static TArray<FFieldCompiledInInfo*> DeferredClassRegistration;
	return DeferredClassRegistration;
}
```
#### Z_CompiledInDefer_UClass_UMyTest
```cpp
static FCompiledInDefer Z_CompiledInDefer_UClass_UMyTest(Z_Construct_UClass_UMyTest, &UMyTest::StaticClass, TEXT("/Script/Azure"), TEXT("UMyTest"), false, nullptr, nullptr, nullptr);

struct FCompiledInDefer
{
	FCompiledInDefer(class UClass *(*InRegister)(), class UClass *(*InStaticClass)(), const TCHAR* PackageName, const TCHAR* Name, bool bDynamic, const TCHAR* DynamicPackageName = nullptr, const TCHAR* DynamicPathName = nullptr, void 					(*InInitSearchableValues)(TMap<FName, FName>&) = nullptr)
	{
		if (bDynamic)
		{
		GetConvertedDynamicPackageNameToTypeName().Add(FName(DynamicPackageName), FName(Name));
		}
        // 收集信息
		UObjectCompiledInDefer(InRegister, InStaticClass, Name, PackageName, bDynamic, DynamicPathName, InInitSearchableValues);
	}
};

void UObjectCompiledInDefer(UClass *(*InRegister)(), UClass *(*InStaticClass)(), const TCHAR* Name, const TCHAR* PackageName, bool bDynamic, const TCHAR* DynamicPathName, void (*InInitSearchableValues)(TMap<FName, FName>&))
{
	if (!bDynamic)
	{
		{
			FString NoPrefix(UObjectBase::RemoveClassPrefix(Name));
			NotifyRegistrationEvent(PackageName, *NoPrefix, ENotifyRegistrationType::NRT_Class, ENotifyRegistrationPhase::NRP_Added, (UObject *(*)())(InRegister), false);
			NotifyRegistrationEvent(PackageName, *(FString(DEFAULT_OBJECT_PREFIX) + NoPrefix), ENotifyRegistrationType::NRT_ClassCDO, ENotifyRegistrationPhase::NRP_Added, (UObject *(*)())(InRegister), false);

			TArray<UClass *(*)()>& DeferredCompiledInRegistration = GetDeferredCompiledInRegistration();
			checkSlow(!DeferredCompiledInRegistration.Contains(InRegister));
			DeferredCompiledInRegistration.Add(InRegister); // 最终添加到了DeferredCompiledInRegistration数组里面
		}
	}
}

static TArray<class UClass *(*)()>& GetDeferredCompiledInRegistration()
{
	static TArray<class UClass *(*)()> DeferredCompiledInRegistration;
	return DeferredCompiledInRegistration;
}
```
再进一步跟一下就会发现，上述两个static初始化的过程有些不同，

- AutoInitializeUMyTest初始化，最终添加到DeferredClassRegistration这个数组里面的是FFieldCompiledInInfo这个结构。这个FFieldCompiledInInfo结构主要是里面有个 Register方法，通过 Register方法能够调用GetPrivateStaticClass方法，进而创建出此类的UClass（这个过程后面还会讲一下）。
- Z_CompiledInDefer_UClass_UMyTest初始化，最终添加到DeferredCompiledInRegistration这个数组里面的Z_Construct_UClass_UMyTest这个方法（返回的也是上面创建出的UClass）。这里面的操作就是用创建出的UClass对象，在这个对象基础上在调用一些构造函数来初始化属性和函数等一些信息。

这两个初始化注册的操作都是在执行main函数之前就完成的，因为他们都是静态的对象，所以在main函数执行之前，对应的两个数组中就已经有了相应的信息，只等着运行其中相应的函数了。

## UClass注册，生成和CDO创建
### UObject的UClass创建

### UClass注册
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/202408261718309.png)
根据关卡中的部分截图可知，引擎启动之后会走到FEngineLoop::PreInit方法中，在这个方法里面会执行PreInitPreStartupScreen方法，而这个PreInitPreStartupScreen方法中
```cpp
// 这个方法里面执行的东西很多，就列举这两个吧，关于模块加载的方法
int32 FEngineLoop::PreInitPreStartupScreen(const TCHAR* CmdLine)
{
	{
		if (!LoadCoreModules())
		{
			UE_LOG(LogInit, Error, TEXT("Failed to load Core modules."));
			return 1;
		}
	}
	{
		LoadPreInitModules();
	}
}

bool FEngineLoop::LoadCoreModules()
{
	// 这个LoadModule方法最终会走到具体模块的StartupModule方法中
	// IMPLEMENT_MODULE( FCoreUObjectModule, CoreUObject );这个宏定义了CoreUObject模块，也就是说会走到FCoreUObjectModule类的StartupModule方法中
	return FModuleManager::Get().LoadModule(TEXT("CoreUObject")) != nullptr;
}
void FEngineLoop::LoadPreInitModules()
{
	// 这个方法看上去也全是加载模块
	FModuleManager::Get().LoadModule(TEXT("Engine"));
	FModuleManager::Get().LoadModule(TEXT("Renderer"));
	FModuleManager::Get().LoadModule(TEXT("AnimGraphRuntime"));
	FModuleManager::Get().LoadModule(TEXT("Landscape"));
	FModuleManager::Get().LoadModule(TEXT("RenderCore"));
}

```
#### 1 FCoreUObjectModule::StartupModule
```cpp
// 这个就是FCoreUObjectModule类的StartupModule方法
virtual void StartupModule() override
{	
	// 字面意思就是把CompiledInClasses这个里面的所有东西全部Register掉，后面还会详细看下
	UClassRegisterAllCompiledInClasses();
	void InitUObject();
	FCoreDelegates::OnInit.AddStatic(InitUObject);
}

/*
	字面意思就是把CompiledInClasses这个里面的所有东西全部Register掉
*/
void UClassRegisterAllCompiledInClasses()
{
	// 把所有热更的东西全都删掉了
	SCOPED_BOOT_TIMING("UClassRegisterAllCompiledInClasses");

    //获取DeferredClassRegistration数组，然后全部调用Register方法，上面讲过这个数组还记得么？
	TArray<FFieldCompiledInInfo*>& DeferredClassRegistration = GetDeferredClassRegistration();
	for (const FFieldCompiledInInfo* Class : DeferredClassRegistration)
	{
		UClass* RegisteredClass = Class->Register();
	}
	DeferredClassRegistration.Empty();
}

/*
UClass* RegisteredClass = Class->Register();这个方法最终会调用到
GetPrivateStaticClass这个里面，也就是创建出UClass了，这个咱们在创建UObject的UClass讲过了

总结下：
1 跟据DeferredClassRegistration这个数组里收集的元数据，依次通过placement new的方式创建出了UClass
2 初始化了新创建的这个UClass，设置了SuperStruct（记录对象父类的UClass，IsChildOf这里面使用的），ClassWithin(当前类可以属于哪个UClass）
3 初始注册步骤，没有实际注册，只是将信息添加到了PendingRegistrants数组中和FPendingRegistrant链表上
*/

```




### MyTest的UClass创建流程
ue会根据module来编译生成对应的dll，在引擎初始化的时候会加载这些dll，然后每次加载都会调用下面这个切入函数来完成UClass的注册。

切入函数（ProcessNewlyLoadedUObjects），这个是加载咱们的那个dll之后会调用到这里：
![image-20230601114508215.png](https://gitee.com/lurenjia399/image/raw/master/image/202408261639367.png)

![image-20230531110405446.png](https://gitee.com/lurenjia399/image/raw/master/image/202408261639624.png)

```cpp
/*
这个方法是通过回调调用进来的
FModuleManager::Get().OnProcessLoadedObjectsCallback().AddStatic(ProcessNewlyLoadedUObjects);
*/
void ProcessNewlyLoadedUObjects(FName Package, bool bCanProcessNewlyLoadedObjects)
{
    //就留了这部分，其他全省略了
	UClassRegisterAllCompiledInClasses();//使用上面讲的DeferredClassRegistration这个数组，创建出UClass

	const TArray<UClass* (*)()>& DeferredCompiledInRegistration = GetDeferredCompiledInRegistration();
	const TArray<FPendingStructRegistrant>& DeferredCompiledInStructRegistration = GetDeferredCompiledInStructRegistration();
	const TArray<FPendingEnumRegistrant>& DeferredCompiledInEnumRegistration = GetDeferredCompiledInEnumRegistration();

	bool bNewUObjects = false;
	while (GFirstPendingRegistrant || DeferredCompiledInRegistration.Num() || DeferredCompiledInStructRegistration.Num() || 				DeferredCompiledInEnumRegistration.Num())
	{
		bNewUObjects = true;
		UObjectProcessRegistrants();//完成实际注册
		UObjectLoadAllCompiledInStructs();

		FCoreUObjectDelegates::CompiledInUObjectsRegisteredDelegate.Broadcast(Package);

		UObjectLoadAllCompiledInDefaultProperties();//使用上面讲的DeferredCompiledInRegistration这个数组，初始化属性，函数，CDO
	}
}
```
#### 1 UClassRegisterAllCompiledInClasses
```cpp
/*
	字面意思就是把CompiledInClasses这个里面的所有东西全部Register掉
*/
void UClassRegisterAllCompiledInClasses()
{
	// 把所有热更的东西全都删掉了
	SCOPED_BOOT_TIMING("UClassRegisterAllCompiledInClasses");

    //获取DeferredClassRegistration数组，然后全部调用Register方法，上面讲过这个数组还记得么？
	TArray<FFieldCompiledInInfo*>& DeferredClassRegistration = GetDeferredClassRegistration();
	for (const FFieldCompiledInInfo* Class : DeferredClassRegistration)
	{
		UClass* RegisteredClass = Class->Register();
	}
	DeferredClassRegistration.Empty();
}

/*
UClass* RegisteredClass = Class->Register();这个方法最终会调用到
GetPrivateStaticClass这个里面，也就是创建出UClass了，这个咱们在创建UObject的UClass讲过了
*/

/*
总结下这个方法：
1 跟据DeferredClassRegistration这个数组里收集的元数据，依次通过placement new的方式创建出了UClass。
2 初始化了新创建的这个UClass，设置了SuperStruct（记录对象父类的UClass，IsChildOf这里面使用的），ClassWithin(当前类可以属于哪个UClass）。
3 初始注册步骤，没有实际注册，只是将信息添加到了PendingRegistrants数组中和FPendingRegistrant链表上。
*/
```
#### 2 UObjectProcessRegistrants
```cpp
static void UObjectProcessRegistrants()
{
    // 把一些宏，注释，检查啥的乱七八糟的都去掉了
    
	TArray<FPendingRegistrant> PendingRegistrants;
    // 这个方法就是通过上面那个全局链表，依次把链表上的东西都放到PendingRegistrants数组中，按顺序，下面看下
	DequeuePendingAutoRegistrants(PendingRegistrants);

	for(int32 RegistrantIndex = 0;RegistrantIndex < PendingRegistrants.Num();++RegistrantIndex)
	{
		const FPendingRegistrant& PendingRegistrant = PendingRegistrants[RegistrantIndex];
		// 这个方法就是通过上面那个全局map，依次找到，然后调用DeferredRegister方法
		UObjectForceRegistration(PendingRegistrant.Object, false);
        // Register may have resulted in new pending registrants being enqueued, so dequeue those.
        // 注册可能导致新的待定注册者被排队，所以重排
        // 这是因为在注册一个UObject的时候，有可能会触发另一个module的加载，从而有新的注册项进来
		DequeuePendingAutoRegistrants(PendingRegistrants);
	}
}

// 就是个链表元素添加到数组中的操作
static void DequeuePendingAutoRegistrants(TArray<FPendingRegistrant>& OutPendingRegistrants)
{
	FPendingRegistrant* NextPendingRegistrant = GFirstPendingRegistrant;
	GFirstPendingRegistrant = NULL;
	GLastPendingRegistrant = NULL;
	while(NextPendingRegistrant)
	{
		FPendingRegistrant* PendingRegistrant = NextPendingRegistrant;
		OutPendingRegistrants.Add(*PendingRegistrant);
		NextPendingRegistrant = PendingRegistrant->NextAutoRegister;
		delete PendingRegistrant;
	};
}

void UObjectForceRegistration(UObjectBase* Object, bool bCheckForModuleRelease)
{
	TMap<UObjectBase*, FPendingRegistrantInfo>& PendingRegistrants = FPendingRegistrantInfo::GetMap();

	FPendingRegistrantInfo* Info = PendingRegistrants.Find(Object);
	if (Info)
	{
		const TCHAR* PackageName = Info->PackageName;
		const TCHAR* Name = Info->Name;
		PendingRegistrants.Remove(Object);  // delete this first so that it doesn't try to do it twice
        // 这个才是真正的注册操作
		Object->DeferredRegister(UClass::StaticClass(),PackageName,Name);
	}
}

// 真正的注册操作
void UObjectBase::DeferredRegister(UClass *UClassStaticClass,const TCHAR* PackageName,const TCHAR* InName)
{
	UPackage* Package = CreatePackage(PackageName);
	Package->SetPackageFlags(PKG_CompiledIn);
	OuterPrivate = Package; //设置了这个Outer

	ClassPrivate = UClassStaticClass;//设置了UClass*的类型

	// Add to the global object table.
	AddObject(FName(InName), EInternalObjectFlags::None);

	GUObjectArray.IndexToObject(InternalIndex)->ClearFlags(EInternalObjectFlags::PendingConstruction);
}
```




