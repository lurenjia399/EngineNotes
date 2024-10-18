ue中camera的创建是在APlayerController::PostInitializeComponents方法中
``` cpp
void APlayerController::PostInitializeComponents()
{
	Super::PostInitializeComponents();

	if ( IsValid(this) && (GetNetMode() != NM_Client) )
	{
		// create a new player replication info
		InitPlayerState();
	}

	SpawnPlayerCameraManager();
	ResetCameraMode(); 

	if ( GetNetMode() == NM_Client )
	{
		SpawnDefaultHUD();
	}

	AddCheats();

	bPlayerIsWaiting = true;
	StateName = NAME_Spectating; // Don't use ChangeState, because we want to defer spawning the SpectatorPawn until the Player is received
}
```