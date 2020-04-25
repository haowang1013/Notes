
# Synchronization Primitives

## FEvent
```
// Thread A
SyncEvent = FGenericPlatformProcess::GetSynchEventFromPool();
...
SyncEvent->Trigger();

// Thread B
if (SyncEvent->Wait(Time))
{
	// Event is triggered
}
else
{
	// The wait times out
}
```

## FCriticalSection

```
FCriticalSection CS;
{
	FScopeLock Lock(&CS);
	// Do stuff with a shared data structure that can be accessed in multiple threads
}
```

## TAtomic

## FThreadSafeBool and FThreadSafeCounter

## Others

See LockFreeList.h

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

# IQueuedWork and FQueuedThreadPool Internals

- FQueuedThreadPool is implemented by FQueuedThreadPoolBase
```
/**
 * Implementation of a queued thread pool.
 */
class FQueuedThreadPoolBase : public FQueuedThreadPool
{
protected:

	/** The work queue to pull from. */
	TArray<IQueuedWork*> QueuedWork;
	
	/** The thread pool to dole work out to. */
	TArray<FQueuedThread*> QueuedThreads;

	/** All threads in the pool. */
	TArray<FQueuedThread*> AllThreads;

	/** The synchronization object used to protect access to the queued work. */
	FCriticalSection* SynchQueue;

	/** If true, indicates the destruction process has taken place. */
	bool TimeToDie;
```

- When work is added to the thread pool, the thread pool checks if there're free thread available

  - If not, the work is added to the "QueuedWork" array

  - If a thread is available, it's popped from the end of the "QueuedThreads" array and the work is added to the thread by calling "DoWork"


- The "threads" in the thread pool is implemented by FQueuedThread
```
class FQueuedThread
	: public FRunnable
{
protected:

	/** The event that tells the thread there is work to do. */
	FEvent* DoWorkEvent;

	/** If true, the thread should exit. */
	TAtomic<bool> TimeToDie;

	/** The work this thread is doing. */
	IQueuedWork* volatile QueuedWork;

	/** The pool this thread belongs to. */
	class FQueuedThreadPool* OwningThreadPool;

	/** My Thread  */
	FRunnableThread* Thread;
```

- In the thread loop, FQueuedThread:

  - Wait for the "DoWorkEvent"

  - Do the work via calling "DoThreadedWork"

  - Then return itself to the pool by calling "ReturnToPoolOrGetNextJob"

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
