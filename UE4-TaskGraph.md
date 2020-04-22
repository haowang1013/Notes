# Init
```
int32 FEngineLoop::PreInitPreStartupScreen(const TCHAR* CmdLine)
{
	...
	FTaskGraphInterface::Startup(FPlatformMisc::NumberOfCores());
	FTaskGraphInterface::Get().AttachToThread(ENamedThreads::GameThread);
}
```

# Shutdown
```
void FEngineLoop::Exit()
{
	...
	FTaskGraphInterface::Shutdown();
}
```

# FGraphEvent

FGraphEvent contains a list of tasks that rely on the completion of this event.

FGraphEvent can also depend on other FGraphEvent until it fires - FGraphEvent::DontCompleteUntil

When the task completes, DispatchSubsequents is called (either by user code or TG):

- If the current event has "EventsToWaitFor", a special new "gather" task is created to wait for those events

- Otherwise all the tasks in the "SubsequentList" is popped and ConditionalQueueTask is called upon

  ConditionalQueueTask decrements FBaseGraphTask::NumberOfPrerequistitesOutstanding and queues the task if it reaches 0

# Task Dependencies

When creating new task, the user can pass in a "FGraphEventArray" which contains the event from all the existing tasks that the new task must depend on

Example:
```
// Task 0 and Task 1 have no dependency
auto Task0 = FFunctionGraphTask::CreateAndDispatchWhenReady([]()
{
	UE_LOG(LogTemp, Log, TEXT("start task 0..."));
	FPlatformProcess::Sleep(1);
	UE_LOG(LogTemp, Log, TEXT("finish task 0"));
}, {}, nullptr);

auto Task1 = FFunctionGraphTask::CreateAndDispatchWhenReady([]() 
{
	UE_LOG(LogTemp, Log, TEXT("start task 1..."));
	FPlatformProcess::Sleep(3);
	UE_LOG(LogTemp, Log, TEXT("finish task 1"));
}, {}, nullptr);

// Task 2 depends on Task 0 and Task 1
FGraphEventArray Task2Dependencies;
Task2Dependencies.Add(Task0);
Task2Dependencies.Add(Task1);
auto Task2 = FFunctionGraphTask::CreateAndDispatchWhenReady([]()
{
	UE_LOG(LogTemp, Log, TEXT("start task 2..."));
	FPlatformProcess::Sleep(2);
	UE_LOG(LogTemp, Log, TEXT("finish task 2"));
}, {}, &Task2Dependencies);

// Task 3 only depends on Task 0
auto Task3 = FFunctionGraphTask::CreateAndDispatchWhenReady([]()
{
	UE_LOG(LogTemp, Log, TEXT("start task 3..."));
	FPlatformProcess::Sleep(1);
	UE_LOG(LogTemp, Log, TEXT("finish task 3"));
}, {}, { Task0 });
```

Note that if a dependended task is already finished when the depending task is created, it will not be added as a prerequisite

See TGraphTask::SetupPrereqs and TGraphTask::PrerequisitesComplete

# Task Creation

Some examples:

```
// Lambda function task
auto Task = FFunctionGraphTask::CreateAndDispatchWhenReady([]()
{
	// Do stuff
}, {}, nullptr);
```
