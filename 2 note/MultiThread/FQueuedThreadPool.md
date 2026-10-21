# FQueuedThreadPool
```cpp
class FQueuedThreadPool
{
	// 线程池创建线程的方法，不同的线程池重写
	virtual bool Create(
		uint32 InNumQueuedThreads, 
		uint32 StackSize = (32 * 1024), 
		EThreadPriority ThreadPriority = TPri_Normal, 
		const TCHAR* Name = TEXT("UnknownThreadPool")) = 0;
	// 线程池销毁线程d
	virtual void Destroy() = 0;
	
	virtual void AddQueuedWork( 
		IQueuedWork* InQueuedWork, 
		EQueuedWorkPriority InQueuedWorkPriority= EQueuedWorkPriority::Normal) = 0;

	virtual bool RetractQueuedWork(IQueuedWork* InQueuedWork) = 0;
	
	// 基类的静态分配方法，默认new一个普通的线程池
	static CORE_API FQueuedThreadPool* Allocate()  
	{  
	    return new FQueuedThreadPoolBase;  
	}
}
```