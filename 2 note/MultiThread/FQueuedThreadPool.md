# FQueuedThreadPool
```cpp
class FQueuedThreadPool
{
	virtual bool Create(
		uint32 InNumQueuedThreads, 
		uint32 StackSize = (32 * 1024), 
		EThreadPriority ThreadPriority = TPri_Normal, 
		const TCHAR* Name = TEXT("UnknownThreadPool")) = 0;
		
	virtual void Destroy() = 0;
	
	virtual void AddQueuedWork( 
		IQueuedWork* InQueuedWork, 
		EQueuedWorkPriority InQueuedWorkPriority= EQueuedWorkPriority::Normal) = 0;

	virtual bool RetractQueuedWork(IQueuedWork* InQueuedWork) = 0;
}
```