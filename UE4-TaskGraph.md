# FTaskGraphImplementation

- FTaskGraphInterface is implemented by FTaskGraphImplementation
```
class FTaskGraphImplementation : public FTaskGraphInterface
{
	// per-thread data for all the threads, including the named and unnamed threads
	FWorkerThread		WorkerThreads[MAX_THREADS];

	// per-priority task queue for the unnamed threads (MAX_THREAD_PRIORITIES is 3)
	FStallingTaskQueue<FBaseGraphTask, PLATFORM_CACHE_LINE_SIZE, 2>	IncomingAnyThreadTasks[MAX_THREAD_PRIORITIES];
}
```

- FWorkerThread
```
struct FWorkerThread
{
	/** The actual FTaskThread that manager this task **/
	FTaskThreadBase*	TaskGraphWorker;
	
	/** For internal threads, the is non-NULL and holds the information about the runable thread that was created. **/
	FRunnableThread*	RunnableThread;
}
```

- FNamedTaskThread
```
class FNamedTaskThread : public FTaskThreadBase
{
	/** Grouping of the data for an individual queue. **/
	struct FThreadTaskQueue
	{
		FStallingTaskQueue<FBaseGraphTask, PLATFORM_CACHE_LINE_SIZE, 2> StallQueue;

		/** We need to disallow reentry of the processing loop **/
		uint32 RecursionGuard;

		/** Indicates we executed a return task, so break out of the processing loop. **/
		bool QuitForReturn;

		/** Indicates we executed a return task, so break out of the processing loop. **/
		bool QuitForShutdown;

		/** Event that this thread blocks on when it runs out of work. **/
		FEvent*	StallRestartEvent;
	};

	// NumQueues is 2
	FThreadTaskQueue Queues[ENamedThreads::NumQueues];
}
```

- FTaskThreadAnyThread
```
class FTaskThreadAnyThread : public FTaskThreadBase
{
	/** Grouping of the data for an individual queue. **/
	struct FThreadTaskQueue
	{
		/** Event that this thread blocks on when it runs out of work. **/
		FEvent* StallRestartEvent;
		/** We need to disallow reentry of the processing loop **/
		uint32 RecursionGuard;
		/** Indicates we executed a return task, so break out of the processing loop. **/
		bool QuitForShutdown;
		/** Should we stall for tuning? **/
		bool bStallForTuning;
		FCriticalSection StallForTuning;
	};

	FThreadTaskQueue Queue;
}
```

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

Lambda function task:

```
auto Task = FFunctionGraphTask::CreateAndDispatchWhenReady([]()
{
	// Do stuff
}, {}, nullptr);
```

Custom task class with sub tasks:
```
class FMyTaskGraphTask
{
public:
	FMyTaskGraphTask(int32 InLeve, FString InMessage) : Level(InLeve), Message(InMessage)
	{}

	FORCEINLINE TStatId GetStatId() const
	{
		RETURN_QUICK_DECLARE_CYCLE_STAT(FMyTaskGraphTask, STATGROUP_TaskGraphTasks);
	}

	ENamedThreads::Type GetDesiredThread() { return ENamedThreads::AnyThread; }

	static ESubsequentsMode::Type GetSubsequentsMode() { return ESubsequentsMode::TrackSubsequents; }

	void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
	{
		UE_LOG(LogTemp, Log, TEXT("%s"), *Message);
		if (Level < 10)
		{
			FString NewMessage = Message + TEXT(" Child");
			// Add a sub task which must complete until the current task is considered completed
			MyCompletionGraphEvent->DontCompleteUntil(TGraphTask<FMyTaskGraphTask>::CreateTask(NULL, CurrentThread).ConstructAndDispatchWhenReady(Level + 1, NewMessage));
		}		
	}

private:
	int32 Level;
	FString Message;
};

// Create the main task and wait for its completion
auto Task = TGraphTask<FMyTaskGraphTask>::CreateTask(nullptr).ConstructAndDispatchWhenReady(0, TEXT("Main Task"));
FTaskGraphInterface::Get().WaitUntilTasksComplete({ Task });
UE_LOG(LogTemp, Log, TEXT("All Finished!"));
```

# Task Dispatching

- ConstructAndDispatchWhenReady is called the construct the actual task object and try to dispatch it

- It calls TGraphTask::Setup, which calls SetupPrereqs

- SetupPrereqs goes through all the Prerequisites that the task depends on, try to add the task itself as a subsequent by calling "AddSubsequent"

- If AddSubsequent fails, it means the particular prerequisite is already finished, and "AlreadyCompletedPrerequisites" increments

- Then "PrerequisitesComplete" is called, which checks if all the prerequisites are completed, in which case "QueueTask" is called

```
virtual void QueueTask(FBaseGraphTask* Task, ENamedThreads::Type ThreadToExecuteOn, ENamedThreads::Type InCurrentThreadIfKnown = ENamedThreads::AnyThread) final override
{
	if (ThreadToExecuteOn is AnyThread)
	{
		if (multi-threading is supported)
		{
			Figure out task the thread priority
			Push the task to a per-thread-priority entry in "IncomingAnyThreadTasks"
			Call "StartTaskThread" to wake up the thread if possible
		}
		else
		{
			set ThreadToExecuteOn to GameThread
		}
	}

	Figure out the current thread

	Get the FTaskThreadBase instance based on ThreadToExecuteOn, call EnqueueFromThisThread or EnqueueFromOtherThread
}
```
