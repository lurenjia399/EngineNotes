```cpp
class FNamedTaskThread : public FTaskThreadBase
{
	// 工作队列，有两个MainQueue和LocalQueue
	FThreadTaskQueue Queues[ENamedThreads::NumQueues];
}
```

# 