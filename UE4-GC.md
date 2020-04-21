
# CollectGarbage
```
void CollectGarbage(EObjectFlags KeepFlags, bool bPerformFullPurge)
{
	// No other thread may be performing UObject operations while we're running
	AcquireGCLock();

	// Perform actual garbage collection
	CollectGarbageInternal(KeepFlags, bPerformFullPurge);

	// Other threads are free to use UObjects
	ReleaseGCLock();
}
```

# CollectGarbageInternal

- Flush async loading
```
// Flush streaming before GC if requested
if (GFlushStreamingOnGC)
{
	if (IsAsyncLoading())
	{
		UE_LOG(LogGarbage, Log, TEXT("CollectGarbageInternal() is flushing async loading"));
	}
	FGCCSyncObject::Get().GCUnlock();
	FlushAsyncLoading();
	FGCCSyncObject::Get().GCLock();
}
```

- Purge previously collected garbage
```
if (GObjIncrementalPurgeIsInProgress || GObjPurgeIsRequired)
{
	IncrementalPurgeGarbage(false);
	FMemory::Trim();
}
```

- Perform reachability analysis
```
{
	const double StartTime = FPlatformTime::Seconds();
	FRealtimeGC TagUsedRealtimeGC;
	TagUsedRealtimeGC.PerformReachabilityAnalysis(KeepFlags, bForceSingleThreadedGC);
	UE_LOG(LogGarbage, Log, TEXT("%f ms for GC"), (FPlatformTime::Seconds() - StartTime) * 1000);
}
```

- Reconstruct clusters if needed
```
if (GUObjectClusters.ClustersNeedDissolving())
{
	const double StartTime = FPlatformTime::Seconds();
	GUObjectClusters.DissolveClusters();
	UE_LOG(LogGarbage, Log, TEXT("%f ms for dissolving GC clusters"), (FPlatformTime::Seconds() - StartTime) * 1000);
}
```

- GatherUnreachableObjects
```
GatherUnreachableObjects(bForceSingleThreadedGC);
```

- UnhashUnreachableObjects
```
if (bPerformFullPurge || !GIncrementalBeginDestroyEnabled)
{
	UnhashUnreachableObjects(/**bUseTimeLimit = */ false);
	FScopedCBDProfile::DumpProfile();
}
```

- IncrementalPurgeGarbage
```
// Perform a full purge by not using a time limit for the incremental purge. The Editor always does a full purge.
if (bPerformFullPurge || GIsEditor)
{
	IncrementalPurgeGarbage(false);
}
```

## PerformReachabilityAnalysis

- Add UGCObjectReferencer (collection of all FGCObject)

- MarkObjectsAsUnreachable

- PerformReachabilityAnalysisOnObjects

### MarkObjectsAsUnreachable

- Use "ParallelFor" to do the work on multiple threads

- Add objects from the root set to the serialize list
  
  Add cluster root or cluster objects to the KeepClusterRefsList

- Add objects with the keep flag to the serialize list

- Add cluster root with the keep flag to the KeepClusterRefsList

- Add pending kill cluster root to the ClustersToDissolveList

- Set EInternalObjectFlags::Unreachable on all the other objects

- Call "DissolveClusterAndMarkObjectsAsUnreachable" on objects from the ClustersToDissolveList

- For each cluster object in the KeepClusterRefsList:

  1. Set EInternalObjectFlags::ReachableInCluster  

  2. Clear EInternalObjectFlags::Unreachable from its cluster root

  3. Call FGCReferenceProcessorSinglethreaded::MarkReferencedClustersAsReachable on the cluster root

- For each cluster root in the KeepClusterRefsList:

  1. Call FGCReferenceProcessorSinglethreaded::MarkReferencedClustersAsReachable

### PerformReachabilityAnalysisOnObjects
```
virtual void PerformReachabilityAnalysisOnObjects(FGCArrayStruct* ArrayStruct, bool bForceSingleThreaded) override
{
	if (!bForceSingleThreaded)
	{
		FGCReferenceProcessorMultithreaded ReferenceProcessor;
		TFastReferenceCollector<true, FGCReferenceProcessorMultithreaded, FGCCollectorMultithreaded, FGCArrayPool> ReferenceCollector(ReferenceProcessor, FGCArrayPool::Get());
		ReferenceCollector.CollectReferences(*ArrayStruct);
	}
	else
	{
		FGCReferenceProcessorSinglethreaded ReferenceProcessor;
		TFastReferenceCollector<false, FGCReferenceProcessorSinglethreaded, FGCCollectorSinglethreaded, FGCArrayPool> ReferenceCollector(ReferenceProcessor, FGCArrayPool::Get());
		ReferenceCollector.CollectReferences(*ArrayStruct);
	}
}
```

#### TFastReferenceCollector::CollectReferences

- Call ProcessObjectArray in single-threaded mode

- Use FCollectorTaskQueue to divide the work load in multi-threaded mode, which calls ProcessObjectArray in each task with a subset of the work load

##### TFastReferenceCollector::ProcessObjectArray

```
void ProcessObjectArray(FGCArrayStruct& InObjectsToSerializeStruct, const FGraphEventRef& MyCompletionGraphEvent)
```

- Iterate through the objects in ObjectsToSerialize

- Call "FPlatformMisc::PrefetchBlock" to prefetch the next object

- Call "AssembleReferenceTokenStream" on the object class if needed

- Process each token from the object class' ReferenceTokenStream

  1. Call HandleTokenStreamObjectReference. This function does 2 things: 

     Nulling out references of pending kill objects and adding referenced objects to the serialize list.

  2. Spawn new task to process NewObjectsToSerialize if needed

- Spawn new task to process NewObjectsToSerialize if needed

- Swap ObjectsToSerialize and NewObjectsToSerialize, then repeat

# IncrementalPurgeGarbage
```
void IncrementalPurgeGarbage(bool bUseTimeLimit, float TimeLimit)
```

- Call UnhashUnreachableObjects

- Call IncrementalDestroyGarbage

## UnhashUnreachableObjects

- Iterate objects in GUnreachableObjects

- Call ConditionalBeginDestroy 

## IncrementalDestroyGarbage

- Iterate on GUnreachableObjects
```
// Only proceed with destroying the object if the asynchronous cleanup started by BeginDestroy has finished.
if(Object->IsReadyForFinishDestroy())
{
	// Send FinishDestroy message.
	Object->ConditionalFinishDestroy();
}
else
{
	// Add the object index to our list of objects to revisit after we process everything else
	GGCObjectsPendingDestruction.Add(Object);
	GGCObjectsPendingDestructionCount++;
}
```

- If all GUnreachableObjects are processed, do another pass on GGCObjectsPendingDestruction

- If GObjFinishDestroyHasBeenRoutedToAllObjects
   
  1. Call FAsyncPurge::BeginPurge

  2. Call FAsyncPurge::TickPurge

### FAsyncPurge::TickPurge
```
void TickPurge(bool bUseTimeLimit, float TimeLimit, double StartTime)
```

- Call TickDestroyObjects if not threaded

- Call WaitForAsyncDestructionToFinish if threaded and not time limit

- Call TickDestroyGameThreadObjects

#### FAsyncPurge::TickDestroyObjects

- Iterate GUnreachableObjects
```
if (!Thread || Object->IsDestructionThreadSafe())
{
	Object->~UObject();
	GUObjectAllocator.FreeUObject(Object);
}
else
{
	// This object will be destroyed incrementally on the game thread
	GameThreadObjects.Push(Object);
}
```

#### FAsyncPurge::TickDestroyGameThreadObjects

- Iterate GameThreadObjects
```
Object->~UObject();
GUObjectAllocator.FreeUObject(Object);
```
