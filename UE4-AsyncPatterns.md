
# FRunnable and FRunnableThread
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
