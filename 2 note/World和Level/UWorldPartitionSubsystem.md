
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
	
	/*
	1 计算StreamingSource的平均速度，用位移/时间，并用上一个速度插值到当前速度
	*/
	StreamingSourcesHash = 0;
	const double CurrentTime = World->GetTimeSeconds();
	for (FWorldPartitionStreamingSource& StreamingSource : StreamingSources)
	{
		// Update streaming sources velocity
		if (!StreamingSource.Name.IsNone())
		{
			FStreamingSourceVelocity& SourceVelocity = StreamingSourcesVelocity.FindOrAdd(StreamingSource.Name, FStreamingSourceVelocity(StreamingSource.Name));
			StreamingSource.Velocity = SourceVelocity.GetAverageVelocity(StreamingSource.Location, CurrentTime);
		}

		// Update streaming source hash
		StreamingSource.UpdateHash();
		// Build hash for all streaming sources
		StreamingSourcesHash = HashCombine(StreamingSourcesHash, StreamingSource.GetHash());
	}
}
```

# UpdateServerClientsVisibleLevelNames
```cpp
void UWorldPartitionSubsystem::UpdateServerClientsVisibleLevelNames()
{
	UWorld* World = GetWorld();
	if (IsServer(World))
	{
		ServerClientsVisibleLevelNames.Reset();
		if (const UNetDriver* NetDriver = World->GetNetDriver())
		{
			for (UNetConnection* Connection : NetDriver->ClientConnections)
			{
				ServerClientsVisibleLevelNames.Add(Connection->GetClientWorldPackageName());
				ServerClientsVisibleLevelNames.Append(Connection->ClientVisibleLevelNames);
				ServerClientsVisibleLevelNames.Append(Connection->GetClientMakingVisibleLevelNames());
			}
		}
		FHashBuilder HashBuilder;
		TArray<FName> SortedServerClientsVisibleLevelNames = ServerClientsVisibleLevelNames.Array();
		SortedServerClientsVisibleLevelNames.Sort(FNameFastLess());
		HashBuilder << SortedServerClientsVisibleLevelNames;
		ServerClientsVisibleLevelsHash = HashBuilder.GetHash();
	}
}
```