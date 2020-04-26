![TG Diagram](/UML/TaskGraph.png)

# Task Graph Interface

- The global TG (task graph) system is exposed by a **FTaskGraphInterface** singleton, which can be accessed via FTaskGraphInterface::Get()

- TG initialization
```
int32 FEngineLoop::PreInitPreStartupScreen(const TCHAR* CmdLine)
{
	...
	FTaskGraphInterface::Startup(FPlatformMisc::NumberOfCores());
	FTaskGraphInterface::Get().AttachToThread(ENamedThreads::GameThread);
}
```

# Task Creation

The most generic way to implement a task is to define a custom task class with several required functions:
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

There're also other helper construct to make simpler task creation easier, such as a lambda version:
```
auto Task = FFunctionGraphTask::CreateAndDispatchWhenReady([]()
{
	// Do stuff
}, {}, nullptr);
```

# Task Dependencies

There can be dependencies between the tasks, this is mainly implemented by the **FGraphEvent** class:

- **SubsequentList** is a list of **FBaseGraphTask** which can't be queued for execution until the current task is finished
```
/** Threadsafe list of subsequents for the event **/
TClosableLockFreePointerListUnorderedSingleConsumer<FBaseGraphTask, 0>	SubsequentList;
```

- **EventsToWaitFor** is a list of **FGraphEvent**, which the current task must wait until it's considered finished
```
/** List of events to wait for until firing. This is not thread safe as it is only legal to fill it in within the context of an executing task. **/
FGraphEventArray EventsToWaitFor;
```

- Note that EventsToWaitFor is a **COMPLETION** dependency, not an **EXECUTION** dependency.
  
  In fact it can only be added while executing the current task, see FGraphEvent::DontCompleteUntil

- If a dependended task is already finished when the depending task is created, it will not be added as a prerequisite

  See TGraphTask::SetupPrereqs and TGraphTask::PrerequisitesComplete

- Examples:
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

# Task Execution

The task execution logic is defined in **TGraphTask::ExecuteTask**:
```
virtual void ExecuteTask(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThread) final override
{
	// Check that the "EventsToWaitFor" list is empty before execution
	if (TTask::GetSubsequentsMode() == ESubsequentsMode::TrackSubsequents)
	{
		Subsequents->CheckDontCompleteUntilIsEmpty(); // we can only add wait for tasks while executing the task
	}
	
	// Call "DoTask" on the custom task class, then call the destructor to allow the task class to clean up
	TTask& Task = *(TTask*)&TaskStorage;
	{
		FScopeCycleCounter Scope(Task.GetStatId(), true); 
		Task.DoTask(CurrentThread, Subsequents);
		Task.~TTask();
		checkThreadGraph(ENamedThreads::GetThreadIndex(CurrentThread) <= ENamedThreads::GetRenderThread() || FMemStack::Get().IsEmpty()); // you must mark and pop memstacks if you use them in tasks! Named threads are excepted.
	}
	
	TaskConstructed = false;

	// Call "DispatchSubsequents" on the FGraphEvent instance
	if (TTask::GetSubsequentsMode() == ESubsequentsMode::TrackSubsequents)
	{
		FPlatformMisc::MemoryBarrier();
		Subsequents->DispatchSubsequents(NewTasks, CurrentThread);
	}

	// Return the memory used by this task instance back to the small allocator or OS
	if (sizeof(TGraphTask) <= FBaseGraphTask::SMALL_TASK_SIZE)
	{
		this->TGraphTask::~TGraphTask();
		FBaseGraphTask::GetSmallTaskAllocator().Free(this);
	}
	else
	{
		delete this;
	}
}
```

- FGraphEvent::DispatchSubsequents
```
void FGraphEvent::DispatchSubsequents(TArray<FBaseGraphTask*>& NewTasks, ENamedThreads::Type CurrentThreadIfKnown)
{
	// Go through the EventsToWaitFor list, see if any event is not finished
	if (EventsToWaitFor.Num())
	{
		// need to save this first and empty the actual tail of the task might be recycled faster than it is cleared.
		FGraphEventArray TempEventsToWaitFor;
		Exchange(EventsToWaitFor, TempEventsToWaitFor);

		bool bSpawnGatherTask = true;

		if (GTestDontCompleteUntilForAlreadyComplete)
		{
			bSpawnGatherTask = false;
			for (FGraphEventRef& Item : TempEventsToWaitFor)
			{
				if (!Item->IsComplete())
				{
					bSpawnGatherTask = true;
					break;
				}
			}
		}


		// If any event is not finished, spawn a new empty task to wait for the events
		if (bSpawnGatherTask)
		{
			// create the Gather...this uses a special version of private CreateTask that "assumes" the subsequent list (which other threads might still be adding too).
			DECLARE_CYCLE_STAT(TEXT("FNullGraphTask.DontCompleteUntil"),
			STAT_FNullGraphTask_DontCompleteUntil,
				STATGROUP_TaskGraphTasks);

			ENamedThreads::Type LocalThreadToDoGatherOn = ENamedThreads::AnyHiPriThreadHiPriTask;
			if (!GIgnoreThreadToDoGatherOn)
			{
				LocalThreadToDoGatherOn = ThreadToDoGatherOn;
			}

			// Note that here TempEventsToWaitFor is used as the prerequisites of the new task
			// Which means the new task won't be executed until all the tasks in TempEventsToWaitFor are finished!
			// Also, the SubsequentList of the current event will be copied over to the new task so that they'll be triggered when the latter is finished
			TGraphTask<FNullGraphTask>::CreateTask(FGraphEventRef(this), &TempEventsToWaitFor, CurrentThreadIfKnown).ConstructAndDispatchWhenReady(GET_STATID(STAT_FNullGraphTask_DontCompleteUntil), LocalThreadToDoGatherOn);

			// Must return here as SubsequentList can't be closed just yet
			return;
		}
	}

	// If there's no completion dependency, call ConditionalQueueTask on all the tasks from SubsequentList
	// Note that closing SubsequentList marks the completion of this task/event!
	SubsequentList.PopAllAndClose(NewTasks);
	for (int32 Index = NewTasks.Num() - 1; Index >= 0 ; Index--) // reverse the order since PopAll is implicitly backwards
	{
		FBaseGraphTask* NewTask = NewTasks[Index];
		checkThreadGraph(NewTask);
		NewTask->ConditionalQueueTask(CurrentThreadIfKnown);
	}
	NewTasks.Reset();
}
```

- FBaseGraphTask::ConditionalQueueTask
```
void ConditionalQueueTask(ENamedThreads::Type CurrentThread)
{
	// If NumberOfPrerequistitesOutstanding reaches 0, it means all the prerequisites are finished and we can be added to the TG for execution
	if (NumberOfPrerequistitesOutstanding.Decrement()==0)
	{
		QueueTask(CurrentThread);
	}
}

void QueueTask(ENamedThreads::Type CurrentThreadIfKnown)
{
	checkThreadGraph(LifeStage.Increment() == int32(LS_Queued));
	FTaskGraphInterface::Get().QueueTask(this, ThreadToExecuteOn, CurrentThreadIfKnown);
}
```

# Task Dispatching

- **ConstructAndDispatchWhenReady** is called the construct the actual task object and try to dispatch it

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

# Task Graph Implementation

- FTaskGraphInterface is implemented by **FTaskGraphImplementation**

- There's a **FWorkerThread** object for each threads in the TG system, this includes both the named and the unnamed threads
```
// per-thread data for all the threads, including the named and unnamed threads
FWorkerThread		WorkerThreads[MAX_THREADS];
```

- There's a **FStallingTaskQueue** object for each thread priority, they're shared between all the unnamed threads
```
// per-priority task queue for the unnamed threads (MAX_THREAD_PRIORITIES is 3)
FStallingTaskQueue<FBaseGraphTask, PLATFORM_CACHE_LINE_SIZE, 2>	IncomingAnyThreadTasks[MAX_THREAD_PRIORITIES];
```

- A **FRunnableThread** object is created for each unnamed thread, managed in FWorkerThread::RunnableThread

- A **FTaskThreadBase** object is created for each TG threads, used for managing the tasks assigned to the threads

- **FNamedTaskThread** implements FTaskThreadBase for the named threads
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

- **FTaskThreadAnyThread** implements FTaskThreadBase for the unnamed threads
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

- **FStallingTaskQueue** is a container class, managing 2 FIFO queues for tasks of different priorities

  - Task is added to the queue with an assigned priority
  ```
  int32 Push(T* InPayload, uint32 Priority)
  ```

  - When popping tasks from FStallingTaskQueue, the high priorities ones are always popped first
