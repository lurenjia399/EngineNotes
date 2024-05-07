# 1 World的创建流程
1 我们从windows的启动开始看
```cpp
// LaunchWindows.cpp文件中的main函数
int32 WINAPI WinMain(_In_ HINSTANCE hInInstance, _In_opt_ HINSTANCE hPrevInstance, _In_ char* pCmdLine, _In_ int32 nCmdShow)
{
	int32 Result = LaunchWindowsStartup(hInInstance, hPrevInstance, pCmdLine, nCmdShow, nullptr);
	LaunchWindowsShutdown();
	return Result;
}
LAUNCH_API int32 LaunchWindowsStartup( HINSTANCE hInInstance, HINSTANCE hPrevInstance, char*, int32 nCmdShow, const TCHAR* CmdLine )
{
	TRACE_BOOKMARK(TEXT("WinMain.Enter"));

	// Setup common Windows settings
	SetupWindowsEnvironment();

	int32 ErrorLevel			= 0;
	hInstance				= hInInstance;

	if (!CmdLine)
	{
		CmdLine = ::GetCommandLineW();
		if ( ProcessCommandLine() )
		{
			CmdLine = *GSavedCommandLine;
		}
	}

	// If we're running in unattended mode, make sure we never display error dialogs if we crash.
	if ( FParse::Param( CmdLine, TEXT("unattended") ) )
	{
		SetErrorMode(SEM_FAILCRITICALERRORS | SEM_NOGPFAULTERRORBOX | SEM_NOOPENFILEERRORBOX);
	}

#if !(UE_BUILD_SHIPPING && WITH_EDITOR)
	// Named mutex we use to figure out whether we are the first instance of the game running. This is needed to e.g.
	// make sure there is no contention when trying to save the shader cache.
	GIsFirstInstance = MakeNamedMutex( CmdLine );

	if ( FParse::Param( CmdLine,TEXT("crashreports") ) )
	{
		GAlwaysReportCrash = true;
	}
#endif

	bool bNoExceptionHandler = FParse::Param(CmdLine,TEXT("noexceptionhandler"));
	(void)bNoExceptionHandler;
	// Using the -noinnerexception parameter will disable the exception handler within native C++, which is call from managed code,
	// which is called from this function.
	// The default case is to have three wrapped exception handlers 
	// Native: WinMain() -> Native: GuardedMainWrapper().
	// The inner exception handler in GuardedMainWrapper() catches crashes/asserts in native C++ code and is the only way to get the
	// correct callstack when running a 64-bit executable. However, XAudio2 sometimes (?) don't like this and it may result in no sound.
#ifdef _WIN64
	if ( FParse::Param(CmdLine,TEXT("noinnerexception")) || FApp::IsBenchmarking() || bNoExceptionHandler)
	{
		GEnableInnerException = false;
	}
#endif	

	// When we're running embedded, assume that the outer application is going to be handling crash reporting
#if UE_BUILD_DEBUG
	if (GUELibraryOverrideSettings.bIsEmbedded || !GAlwaysReportCrash)
#else
	if (GUELibraryOverrideSettings.bIsEmbedded || bNoExceptionHandler || (FPlatformMisc::IsDebuggerPresent() && !GAlwaysReportCrash))
#endif
	{
		// Don't use exception handling when a debugger is attached to exactly trap the crash. This does NOT check
		// whether we are the first instance or not!
		ErrorLevel = GuardedMain( CmdLine );
	}
	else
	{
		// Use structured exception handling to trap any crashes, walk the the stack and display a crash dialog box.
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__try
#endif
 		{
			GIsGuarded = 1;
			// Run the guarded code.
			ErrorLevel = GuardedMainWrapper( CmdLine );
			GIsGuarded = 0;
		}
#if !PLATFORM_SEH_EXCEPTIONS_DISABLED
		__except( GEnableInnerException ? EXCEPTION_EXECUTE_HANDLER : ReportCrash( GetExceptionInformation( ) ) )
		{
#if !(UE_BUILD_SHIPPING && WITH_EDITOR)
			// Release the mutex in the error case to ensure subsequent runs don't find it.
			ReleaseNamedMutex();
#endif
			// Crashed.
			ErrorLevel = 1;
			if(GError)
			{
				GError->HandleError();
			}
			LaunchStaticShutdownAfterError();
			FPlatformMallocCrash::Get().PrintPoolsUsage();
			FPlatformMisc::RequestExit( true );
		}
#endif
	}

	TRACE_BOOKMARK(TEXT("WinMain.Exit"));

	return ErrorLevel;
}

LAUNCH_API void LaunchWindowsShutdown()
{
	// Final shut down.
	FEngineLoop::AppExit();

#if !(UE_BUILD_SHIPPING && WITH_EDITOR)
	// Release the named mutex again now that we are done.
	ReleaseNamedMutex();
#endif

	// pause if we should
	if (GShouldPauseBeforeExit)
	{
		Sleep(INFINITE);
	}
}
```