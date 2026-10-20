1 创建
```cpp
FRunnableThread* FRunnableThread::Create(
	class FRunnable* InRunnable, // 线程执行任务的实体，具体干活的类，必须传进来
	const TCHAR* ThreadName,// 线程的名称，方便调试和性能分析时识别线程
	uint32 InStackSize,// 线程的栈空间大小，单位为字节
	EThreadPriority InThreadPri, // 线程调度优先级
	uint64 InThreadAffinityMask,//CPU 亲和性掩码，指定线程允许在哪些cpu核上运行
	EThreadCreateFlags InCreateFlags)// 
{
	// 静态方法，返回当前这个平台是否支持多线程
	bool bCreateRealThread = FPlatformProcess::SupportsMultithreading();

	FRunnableThread* NewThread = nullptr;

	// 如果支持多线程，就new一个对应平台的FRunnableThread，参考FRunnableThreadWin
	if (bCreateRealThread)
	{
		check(InRunnable);
		NewThread = FPlatformProcess::CreateRunnableThread();
	}
	// 如果不支持多线程，new一个假的用来替代真正的FRunnableThread
	else if (InRunnable->GetSingleThreadInterface())
	{
		NewThread = new FFakeThread();
	}

	// 执行初始化的逻辑，不同的FRunnableThread会重写初始化的逻辑
	if (NewThread)
	{
		SetupCreatedThread(NewThread, InRunnable, ThreadName, InStackSize, InThreadPri, InThreadAffinityMask, InCreateFlags);
	}

	return NewThread;
}
```
系统线程的基类，通过Create静态方法创建出FFakeThread并返回，参数中InRunnable承载了具体线程执行的任务。不同的平台通过继承这个FRunnableThread才实现不同的系统线程创建。下面看下FRunnableThreadWin这个实现：
```cpp
class FRunnableThreadWin  : public FRunnableThread
{
	// 重写了这个方法，用来通过调用Windows的系统接口创建出Thread，并设置一些线程的属性
	virtual bool CreateInternal(
		FRunnable* InRunnable, 
		const TCHAR* InThreadName,  
	    uint32 InStackSize = 0,  
	    EThreadPriority InThreadPri = TPri_Normal, 
	    uint64 InThreadAffinityMask = 0,  
	    EThreadCreateFlags InCreateFlags = EThreadCreateFlags::None) override;
	// 
	uint32 Run();
}
```