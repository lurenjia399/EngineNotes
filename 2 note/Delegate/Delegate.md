测试用例：
```cpp
// .h文件
DECLARE_DELEGATE_OneParam(FTestActionDelegate, FVector);

class AWorldPartitionLearnCharacter : public ACharacter
{
	FTestActionDelegate TestActionDelegate;
}

// cpp文件
void AWorldPartitionLearnCharacter::BeginPlay()
{
	TestActionDelegate.BindLambda([]()->void{return;});
	TestActionDelegate.Execute(FVector());
}

```
# TDelegate
```cpp
DECLARE_DELEGATE_OneParam(FTestActionDelegate, FVector);
/*
	一步一步进行宏替换可得：
	#define DECLARE_DELEGATE_OneParam( DelegateName, Param1Type ) FUNC_DECLARE_DELEGATE( DelegateName, void, Param1Type )
	
	-> FUNC_DECLARE_DELEGATE(FTestActionDelegate, void, FVector)
	#define FUNC_DECLARE_DELEGATE( DelegateName, ReturnType, ... ) \
	typedef TDelegate<ReturnType(__VA_ARGS__)> DelegateName;
	
	-> typedef TDelegate<void(FVector)> FTestActionDelegate;
*/
//声明了一个 TDelegate<void(FVector)> 这个类型，名字是FTestActionDelegate
```
1 宏就是声明了一个TDelegate的模板类，下面我们看下这个模板类。
```cpp
/*
	空壳类，两个模板参数，一个是代理类型，一个是默认的FDefaultDelegateUserPolicy，这里采用了模板编程的Policy设计，后面有解释，
*/
template <typename DelegateSignature, typename UserPolicy = FDefaultDelegateUserPolicy>
class TDelegate
{
	static_assert(sizeof(UserPolicy) == 0, "Expected a function signature for the delegate template parameter");
};

/*
	UserPolicy是有默认值的，所以我们再用特化版本的时候，只要不修改，UserPolicy就是默认值，这也就是上文说的 DelegateType
*/
template <typename InRetValType, typename... ParamTypes, typename UserPolicy>
class TDelegate<InRetValType(ParamTypes...), UserPolicy> : public UserPolicy::FDelegateExtras
{
	// 这边就列举了成员变量，只有两个，一个是DelegateAllocator代表代理实例分配的内存位置，一个是DelegateSize代表代理实例的大小（这个大小是包括代理参数和代理函数的）
	FDelegateAllocatorType::ForElementType<FAlignedInlineDelegateType> DelegateAllocator;
	int32 DelegateSize = 0;
}
```
2 我们会使用TDelegate特化出的版本，TDelegate里面就有Bind方法和Excute方法。
```cpp
class TDelegate
{
	template<typename FunctorType, typename... VarTypes>
	inline void BindLambda(FunctorType&& InFunctor, VarTypes&&... Vars)
	{
		// 这边简略了，源码的意思就是通过CreateDelegateInstance创建出对应的实例
		Super::template CreateDelegateInstance<>(...);
	}
	// 这个创建实例的方法就是通过Allocate分配内存，用placement new的方式创建出TBaseFunctorDelegateInstance这个代理实例
	template<typename DelegateInstanceType, typename... DelegateInstanceParams>
	void CreateDelegateInstance(DelegateInstanceParams&&... Params)
	{
		FWriteAccessScope WriteScope = GetWriteAccessScope();

		IDelegateInstance* DelegateInstance = GetDelegateInstanceProtected();
		if (DelegateInstance)
		{
			DelegateInstance->~IDelegateInstance();
		}

		new(Allocate(sizeof(DelegateInstanceType))) DelegateInstanceType(Forward<DelegateInstanceParams>(Params)...);
	}
}
```
3 我们这个Bind的方法就是通过TDelegate来确定一个内存位置，在这个位置上创建出对应的代理实例，这个代理实例里保存了我们绑定的方法地址和参数。下面我们看下怎么Excute的。
```cpp
class TDelegate
{
	// 我们会根据TDelegate中的GetDelegateInstanceProtected方法来获取，其保存的代理实例，然后就通过代理实例里保存的func然后和Execute传进来的参数，执行就行了。
	FORCEINLINE RetValType Execute(ParamTypes... Params) const
	{
		const DelegateInstanceInterfaceType* LocalDelegateInstance = GetDelegateInstanceProtected();
		return LocalDelegateInstance->Execute(Forward<ParamTypes>(Params)...);
	}
	// 这个方法就是根据DelegateAllocator参数来获取内存中保存的代理实例
	FORCEINLINE IDelegateInstance* GetDelegateInstanceProtected()
	{
		return DelegateSize ? (IDelegateInstance*)DelegateAllocator.GetAllocation() : nullptr;
	}
}
```
# 问题
- Delegate流程是什么样的呢 ？
1 首先我们需要通过宏声明，然后创建出代理的成员变量，其实就是创建了一个TDelegate模板类，这个模板类是便特化出来的，然后有两个成员变量，一个代表我们在绑定时候的代理实例的内存位置，一个代表代理实例的大小。
2 然后我们通过Bind方法绑定函数， 就是通过TDelegate的分配方法，通过placement new的方式创建出代理实例，代理实例里有我们绑定的方法的地址和参数容器。
3 最后我们通过execute执行方法，就是通过TDelegate先获取我们的代理实例，然后通过代理实例中保存的方法和我们execute的参数来执行。
