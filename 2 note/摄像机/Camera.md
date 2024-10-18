ue中camera的创建是在APlayerController::PostInitializeComponents方法中
``` cpp
void APlayerController::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	if ( IsValid(this) && (GetNetMode() != NM_Client) )
	{
		// create a new player replication info
		// 这个是在uds上创建playerState，playerState是需要同步到客户端的
		InitPlayerState();
	}

	// 创建我们的PlayerCameraManager
	SpawnPlayerCameraManager();
	// ResetCameraMode
	ResetCameraMode(); 

	// 这边是在Client下创建一个HUD，好像不知道HUD是干嘛的
	if ( GetNetMode() == NM_Client )
	{
		SpawnDefaultHUD();
	}
	// 创建CheatManager，作弊Manager。干嘛的不清楚
	AddCheats();

	// 看注释说是，还不能用需要初始化？
	bPlayerIsWaiting = true;
	// 设置当前StateName
	StateName = NAME_Spectating; // Don't use ChangeState, because we want to defer spawning the SpectatorPawn until the Player is received
}
```
d