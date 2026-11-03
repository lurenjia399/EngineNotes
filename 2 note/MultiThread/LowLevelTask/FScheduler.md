# FScheduler

```cpp
class FScheduler final : public FSchedulerTls
{
	void FScheduler::StartWorkers(...)
	{
		// 拿到ActiveWorkers原子变量，当前激活的工作线程有多少个
		uint32 OldActiveWorkers = ActiveWorkers.load(std::memory_order_relaxed);
		

	}
}
```