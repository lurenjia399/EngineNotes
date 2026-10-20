1 创建
```cpp
FRunnableThread* FRunnableThread::Create(
	class FRunnable* InRunnable, 
	const TCHAR* ThreadName,
	uint32 InStackSize,
	EThreadPriority InThreadPri, 
	uint64 InThreadAffinityMask,
	EThreadCreateFlags InCreateFlags)
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
	
}
```