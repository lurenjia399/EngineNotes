
# UpdateStreamingSources

```cpp
void UWorldPartitionSubsystem::UpdateStreamingSources()
{
	TArray<FWorldPartitionStreamingSource> ProviderStreamingSources;
	for (IWorldPartitionStreamingSourceProvider* StreamingSourceProvider : 
			GetStreamingSourceProviders())
	{
		if (bAllowPlayerControllerStreamingSources || 
			!Cast<APlayerController>(StreamingSourceProvider-
			>GetStreamingSourceOwner()))
		{
			ProviderStreamingSources.Reset();
			if (StreamingSourceProvider->GetStreamingSources(ProviderStreamingSources))
			{
				for (FWorldPartitionStreamingSource& ProviderStreamingSource : ProviderStreamingSources)
				{
					StreamingSources.Add(MoveTemp(ProviderStreamingSource));
				}
			}
		}
	}
}
```