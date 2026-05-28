
# UpdateStreamingSources

```cpp
void UWorldPartitionSubsystem::UpdateStreamingSources()
{
	// 清空StreamingSources数组
	StreamingSources.Reset();
	/*
	1 遍历StreamingSourceProviders，这个Provider是继承IWorldPartitionStreamingSourceProvider这个接口的，继承接口后才能作为Provider
	2 将Provider提供的StreamingSource添加到StreamingSources数组中
	*/
	TArray<FWorldPartitionStreamingSource> ProviderStreamingSources;
	for (IWorldPartitionStreamingSourceProvider* StreamingSourceProvider : 
			GetStreamingSourceProviders())
	{
		if (bAllowPlayerControllerStreamingSources || 
			!Cast<APlayerController>(StreamingSourceProvider
				->GetStreamingSourceOwner()))
		{
			ProviderStreamingSources.Reset();
			if (StreamingSourceProvider
				->GetStreamingSources(ProviderStreamingSources))
			{
				for (FWorldPartitionStreamingSource& ProviderStreamingSource : 
					ProviderStreamingSources)
				{
					StreamingSources.Add(MoveTemp(ProviderStreamingSource));
				}
			}
		}
	}
	
	// 重置StreamingSourcesVelocity
	for (auto& Pair : StreamingSourcesVelocity)
	{
		Pair.Value.Invalidate();
	}
	
	
}
```