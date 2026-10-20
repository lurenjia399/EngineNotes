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
系统线程的基类，通过Create静态方法创建出FFakeThread并返回，参数中InRunnable承载了具体线程执行的任务。不同的平台通过继承这个FRunnableThread才实现不同的系统线程创建。
下面看下FRunnableThreadWin这个实现：
```cpp
class FRunnableThreadWin  : public FRunnableThread
{
	// 重写了这个方法，用来通过调用Windows的系统接口创建出Thread，并设置一些线程的属性，
	virtual bool CreateInternal(...) override;
	{
		Runnable = InRunnable;
		ThreadAffinityMask = InThreadAffinityMask;
		// 创建一个不同线程间需要手动触发的同步Event
		ThreadInitSyncEvent = FPlatformProcess::GetSynchEventFromPool(true);
		ThreadName = InThreadName ? InThreadName : TEXT("Unnamed UE");
		ThreadPriority = InThreadPri;
		// 调用系统接口，创建出Thread
		Thread = CreateThread(NULL, InStackSize, _ThreadProc, this, STACK_SIZE_PARAM_IS_A_RESERVATION | CREATE_SUSPENDED, (::DWORD*)&ThreadID);
		if (Thread == NULL)
		{...}
		else
		{
			// 恢复创建出的线程，就是让新线程执行，但是新线程有可能无法获取cpu
			ResumeThread(Thread);
			// 设置同步Event，让当前线程永久等待。这个同步Event会在新线程的Run方法中触发
			ThreadInitSyncEvent->Wait(INFINITE);
			
			// 在同步Event之后说明新线程已经执行了，这里重置新线程的优先级
			SetThreadPriority(InThreadPri);
		}
		// 回收同步Event，并返回创建出的新线程
		FPlatformProcess::ReturnSynchEventToPool(ThreadInitSyncEvent);
		ThreadInitSyncEvent = nullptr;
		return Thread != NULL;
	}
	
	// 唤醒新线程后，新线程会执行此方法。将线程的ID和FRunnableThreadWin执行绑定起来，记录到FThreadManager这个单例里面，然后执行GuardedRun方法
	static ::DWORD STDCALL _ThreadProc(LPVOID pThis)
	{
		check(pThis);
		auto* ThisThread = (FRunnableThreadWin*)pThis;
		FThreadManager::Get().AddThread(ThisThread->GetThreadID(), ThisThread);
		return ThisThread->GuardedRun();
	}
	
	// 新线程开始执行的方法，执行Run方法
	uint32 GuardedRun();
	
	// 新线程的Run方法
	uint32 FRunnableThreadWin::Run()
	{
		uint32 ExitCode = 1;
		// 必须保证Runnable干活的类存在
		check(Runnable);
		// 如果Runnable已经初始化了
		if (Runnable->Init() == true)
		{
			// 触发同步Event，告诉那边不用等待了
			ThreadInitSyncEvent->Trigger();
			SetTls();
			// 执行Runnable干活类的Run方法
			ExitCode = Runnable->Run();
			Runnable->Exit();
			FreeTls();
		}
		// 如果Runnable
		else
		{
			ThreadInitSyncEvent->Trigger();
		}
		return ExitCode;
	}
}
```