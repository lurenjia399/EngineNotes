```cpp
class FTaskThreadBase : public FRunnable, FSingleThreadRunnable
{
	// 标记这个NamedThread是什么线程
	ENamedThreads::Type									ThreadId;
	/** TLS SLot that we store the FTaskThread* this pointer in. **/
	uint32												PerThreadIDTLSSlot;
	/** Used to signal stalling. Not safe for synchronization in most cases. **/
	FThreadSafeCounter									IsStalled;
	/** Array of tasks for this task thread. */
	TArray<FBaseGraphTask*> NewTasks;
	/** back pointer to the owning FWorkerThread **/
	FWorkerThread* OwnerWorker;
}

class FNamedTaskThread : public FTaskThreadBase
{
	FThreadTaskQueue Queues[ENamedThreads::NumQueues];
}
```