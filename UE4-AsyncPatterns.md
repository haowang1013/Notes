
# FRunnable and FRunnableThread

Best for recursive work that runs for a long time.

```
class FMyRunnable : public FRunnable
{
public:
	uint32 Run() override
	{
		while(bRunning)
		{
			// Do work here
		}

		return 0;
	}

	void Stop() override
	{
		bRunning = false;
	}

private:
	TAtomic<bool> bRunning = true;
}

// Start the thread
Runnable = new FMyRunnable();
RunnableThread = FRunnableThread::Create(Runnable, TEXT("Thread Name"));

// Stop the thread, this will call FRunnable::Stop
RunnableThread->Kill(true);
RunnableThread = nullptr;

// Finally delete the runnable instance
delete Runnable;
```

Notes:
- The user code must delete the runnable instances.


# IQueuedWork and FQueuedThreadPool

Best for the producer-consumer pattern with no dependency between the work.

```
class FMyQueuedWork : public IQueuedWork
{
public:
	void DoThreadedWork() override
	{
		// Do work here

		delete this;
	}

	void Abandon() override
	{
		// Abandon is only called if DoThreadedWork has not been called yet
		delete this;
	}
}

// Start the thread pool
ThreadPool = FQueuedThreadPool::Allocate();
if (!ThreadPool->Create(NumThreads))
{
	delete ThreadPool;
	ThreadPool = nullptr;
	return;
}

// Add work
auto Work = new FMyQueuedWork();
ThreadPool->AddQueuedWork(Work);

// Stop the thread pool
delete ThreadPool;
```

Notes:
- When the thread pool object is deleted, it calls Destroy() internally, which:
  
  - Calls Abandon() on all the queued work that hasn't started yet.

  - Waits for all the started work to finish.

- The queued work should delete itself when finished or abandoned since the system doesn't do it for you.


# FAutoDeleteAsyncTask

Best for one-off tasks.

```
class FMyTask : public FNonAbandonableTask
{
public:
	FMyTask(const FString& InData) : Data(InData)
	{}

	void DoWork()
	{
		UE_LOG(LogTemp, Log, TEXT("%s"), *Data);
	}

	FORCEINLINE TStatId GetStatId() const
	{
		RETURN_QUICK_DECLARE_CYCLE_STAT(FMyTask, STATGROUP_ThreadPoolAsyncTasks);
	}

private:
	FString Data;
};

// Create and dispatch the task
auto Task = new FAutoDeleteAsyncTask<FMyTask>("This is an auto-delete task!");
Task->StartBackgroundTask();
```

# FAsyncTask
```
class FMyTask : public FNonAbandonableTask
{
public:
	FMyTask(const FString& InData) : Data(InData)
	{}

	void DoWork()
	{
		UE_LOG(LogTemp, Log, TEXT("%s"), *Data);
	}

	FORCEINLINE TStatId GetStatId() const
	{
		RETURN_QUICK_DECLARE_CYCLE_STAT(FMyTask, STATGROUP_ThreadPoolAsyncTasks);
	}

private:
	FString Data;
};

// Create and dispatch the task
auto Task = new FAsyncTask<FMyTask>("This is an auto-delete task!");
Task->StartBackgroundTask();

// Check if the work is done
Task->IsWorkDone();

// Cancel the task
if (Task->Cancel())
{
	delete Task;
}

// Make sure the task is completed
Task->EnsureCompletion(true);

// Wait for completion with a timeout
Task->WaitCompletionWithTimeout(1.f);
```

Notes:
- The task must be deleted by the user code when finished/cancelled.

# Async Functions

Best for one-off stuff.

```
FString Data = ...;	
TFuture<int> Result = Async(EAsyncExecution::TaskGraph|TaskGraphMainThread|Thread|ThreadPool, [Data]()
{
	// Do some work with "Data"
	return 123;
});

...

// Wait for the result
while(1)
{
	if (Result.IsReady())
	{
		// Note that Get() will block until the result is ready!
		auto Number = Result.Get();
		...
	}
}
```

Or

```
AsyncTask(ENamedThreads::GameThread, []()
{
	// Do work here
	// This will be called by the task graph
});
```
