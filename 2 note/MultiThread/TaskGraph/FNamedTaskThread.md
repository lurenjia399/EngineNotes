```cpp
class FTaskThreadBase : public FRunnable, FSingleThreadRunnable
{
	// 标记这个NamedThread是什么线程
	ENamedThreads::Type									ThreadId;
	// TLS中的槽位，每个线程的工作队列啥的都会放在这个槽位上
	uint32												PerThreadIDTLSSlot;
	/** Used to signal stalling. Not safe for synchronization in most cases. **/
	FThreadSafeCounter									IsStalled;
	// 当前线程上的任务数组
	TArray<FBaseGraphTask*> NewTasks;
	// 指向FNamedTaskThread
	FWorkerThread* OwnerWorker;
}

class FNamedTaskThread : public FTaskThreadBase
{
	// 工作队列，有两个MainQueue和LocalQueue
	FThreadTaskQueue Queues[ENamedThreads::NumQueues];
}
```