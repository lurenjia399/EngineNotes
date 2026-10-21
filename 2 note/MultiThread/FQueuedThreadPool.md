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
		
	// 线程池销毁线程的方法，不同的线程池重写
	virtual void Destroy() = 0;
	
	// 向线程池添加任务，如果有空闲的线程就执行任务，如果没有就放到优先级队列里
	virtual void AddQueuedWork( 
		IQueuedWork* InQueuedWork, 
		EQueuedWorkPriority InQueuedWorkPriority= EQueuedWorkPriority::Normal) = 0;
	
	// 线程池尝试回收任务
	virtual bool RetractQueuedWork(IQueuedWork* InQueuedWork) = 0;
	
	// 基类的静态分配方法，默认new一个普通的线程池
	static CORE_API FQueuedThreadPool* Allocate()  
	{  
	    return new FQueuedThreadPoolBase;  
	}
}
```
基类的实现，不同的线程池需要继承然后重写方法。
# FQueuedThreadPoolBase

```cpp
class FQueuedThreadPoolBase : public FQueuedThreadPool
{
	bool CreateInternal(
		bool bForkable, // 创建子进程，子进程继承父进程当时的内存状态。ueyou'te'shu
		uint32 InNumQueuedThreads, // 线程池创建的线程数量
		uint32 StackSize, // 每个线程栈的大小
		EThreadPriority ThreadPriority, // 线程的优先级
		const TCHAR* Name)// 线程池的名称
	{
		
	}
}
```