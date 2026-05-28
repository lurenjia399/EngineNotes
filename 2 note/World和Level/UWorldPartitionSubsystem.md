
# UpdateStreamingSources

```cpp
void UWorldPartitionSubsystem::UpdateStreamingSources()
{
	/*
	1 遍历StreamingSourceProviders，这个Provider是某一个
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
}
```