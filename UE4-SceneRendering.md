# FPrimitiveSceneInfo

- See void FPrimitiveSceneInfo::CacheMeshDrawCommands(FRHICommandListImmediate& RHICmdList)

```
class FPrimitiveSceneInfo
{
	/** The primitive's cached mesh draw commands infos for all static meshes. Kept separately from StaticMeshes for cache efficiency inside InitViews. */
	TArray<class FCachedMeshDrawCommandInfo> StaticMeshCommandInfos;

	/** The primitive's static mesh relevances. Must be in sync with StaticMeshes. Kept separately from StaticMeshes for cache efficiency inside InitViews. */
	TArray<class FStaticMeshBatchRelevance> StaticMeshRelevances;

	/** The primitive's static meshes. */
	TArray<class FStaticMeshBatch> StaticMeshes;
}
```

# Draw Call Merging

- FMeshDrawCommand is considered mergable if FMeshDrawCommand::MatchesForDynamicInstancing return true.

- When static primitive is added to the scene, their draw command is divided into "state bucket" based on the mergability against the existing ones

  See FScene::CachedMeshDrawCommandStateBuckets and FCachedPassMeshDrawListContext::FinalizeCommand

- If a command is merged, FCachedMeshDrawCommandInfo::StateBucketId will have a valid Id

- When collecting visible draw command for the frame, a valid StateBucketId ensures that the same FMeshDrawCommand pointer is returned for different FCachedMeshDrawCommandInfo that are batched
  
  See FDrawCommandRelevancePacket::AddCommandsForMesh

- Before rendering the pass, all the visible mesh commands are sent to FMeshDrawCommandPassSetupTask, as FMeshDrawCommandPassSetupTaskContext::MeshDrawCommands

  Note that by default the mesh command list contain duplicated FMeshDrawCommand* for entries that can be merged.

  The final merging happens in BuildMeshDrawCommandPrimitiveIdBuffer.


# FScene
```
class FScene : public FSceneInterface
{
	FCachedPassMeshDrawList CachedDrawLists[EMeshPass::Num];

	/** Instancing state buckets.  These are stored on the scene as they are precomputed at FPrimitiveSceneInfo::AddToScene time. */
	TSet<FMeshDrawCommandStateBucket, MeshDrawCommandKeyFuncs> CachedMeshDrawCommandStateBuckets;
}
```

# FViewCommands
```
class FViewCommands
{
	TStaticArray<FMeshCommandOneFrameArray, EMeshPass::Num> MeshCommands;
	TStaticArray<int32, EMeshPass::Num> NumDynamicMeshCommandBuildRequestElements;
	TStaticArray<TArray<const FStaticMeshBatch*, SceneRenderingAllocator>, EMeshPass::Num> DynamicMeshCommandBuildRequests;
}
```

# Rendering Thread

- FRenderingThread inherits from FRunnable

- The rendering thread is created in StartRenderingThread
```
void StartRenderingThread()
{
	// Create the rendering thread.
	GRenderingThreadRunnable = new FRenderingThread();

	GRenderingThread = FRunnableThread::Create(GRenderingThreadRunnable, *BuildRenderingThreadName(ThreadCount), 0, FPlatformAffinity::GetRenderingThreadPriority(), FPlatformAffinity::GetRenderingThreadMask());
}
```

- The main loop of the rendering thread happens in RenderingThreadMain
```
void RenderingThreadMain( FEvent* TaskGraphBoundSyncEvent )
{
	// ProcessThreadUntilRequestReturn internally loops to process all the tasks added to the rendering thread, until the program exits
	FTaskGraphInterface::Get().ProcessThreadUntilRequestReturn(RenderThread);
}
```

# Render Command

- Render commands can be created from any threads by using the ENQUEUE_RENDER_COMMAND macro, which calls EnqueueUniqueRenderCommand:
```
template<typename TSTR, typename LAMBDA>
FORCEINLINE_DEBUGGABLE void EnqueueUniqueRenderCommand(LAMBDA&& Lambda)
{
	typedef TEnqueueUniqueRenderCommandType<TSTR, LAMBDA> EURCType;

	if (IsInRenderingThread())
	{
		FRHICommandListImmediate& RHICmdList = GetImmediateCommandList_ForRenderCommand();
		Lambda(RHICmdList);
	}
	else
	{
		if (ShouldExecuteOnRenderThread())
		{
			CheckNotBlockedOnRenderThread();
			TGraphTask<EURCType>::CreateTask().ConstructAndDispatchWhenReady(Forward<LAMBDA>(Lambda));
		}
		else
		{
			EURCType TempCommand(Forward<LAMBDA>(Lambda));
			FScopeCycleCounter EURCMacro_Scope(TempCommand.GetStatId());
			TempCommand.DoTask(ENamedThreads::GameThread, FGraphEventRef());
		}
	}
}
```

- If called from the rendering thread, the lambda is executed immediately

- If callled from other threads, a TEnqueueUniqueRenderCommandType task is created and added to the TG system for execution:
```
template<typename TSTR, typename LAMBDA>
class TEnqueueUniqueRenderCommandType : public FRenderCommand
{
public:
	TEnqueueUniqueRenderCommandType(LAMBDA&& InLambda) : Lambda(Forward<LAMBDA>(InLambda)) {}

	void DoTask(ENamedThreads::Type CurrentThread, const FGraphEventRef& MyCompletionGraphEvent)
	{
		FRHICommandListImmediate& RHICmdList = GetImmediateCommandList_ForRenderCommand();
		Lambda(RHICmdList);
	}
```

# Scene Rendering

- The command for rendering the scene is added in FRendererModule::BeginRenderingViewFamily, here's a typical callstack:
```
UE4Editor-Renderer.dll!FRendererModule::BeginRenderingViewFamily(FCanvas * Canvas, FSceneViewFamily * ViewFamily) Line 3604	C++
UE4Editor-Engine.dll!UGameViewportClient::Draw(FViewport * InViewport, FCanvas * SceneCanvas) Line 1512	C++
UE4Editor-Engine.dll!FViewport::Draw(bool bShouldPresent) Line 1593	C++
UE4Editor-Engine.dll!UGameEngine::RedrawViewports(bool bShouldPresent) Line 658	C++
UE4Editor-Engine.dll!UGameEngine::Tick(float DeltaSeconds, bool bIdleMode) Line 1789	C++
UE4Editor.exe!FEngineLoop::Tick() Line 4485	C++
```

- RenderViewFamily_RenderThread looks like this:
```
static void RenderViewFamily_RenderThread(FRHICommandListImmediate& RHICmdList, FSceneRenderer* SceneRenderer)
{
	if(SceneRenderer->ViewFamily.EngineShowFlags.HitProxies)
	{
		// Render the scene's hit proxies.
		SceneRenderer->RenderHitProxies(RHICmdList);
	}
	else
	{
		// Render the scene.
		SceneRenderer->Render(RHICmdList);
	}
}
```

- There're 2 implementations for FSceneRenderer: FDeferredShadingSceneRenderer and FMobileSceneRenderer

# FDeferredShadingSceneRenderer::Render
```
void FDeferredShadingSceneRenderer::Render(FRHICommandListImmediate& RHICmdList)
{
	// Process primitives that have been added/removed since the last frame
	Scene->UpdateAllPrimitiveSceneInfos(RHICmdList);

 	// Clear the scene color from last frame
	SceneContext.ReleaseSceneColor();

	// Wait for the occlusion result from last frame
	WaitOcclusionTests(RHICmdList);


	bDoInitViewAftersPrepass = InitViews(RHICmdList, BasePassDepthStencilAccess, ILCTaskData, UpdateViewCustomDataEvents);

#if RHI_RAYTRACING
	// Gather mesh instances, shaders, resources, parameters, etc. and build ray tracing acceleration structure
	GatherRayTracingWorldInstances(RHICmdList);
#endif

	UpdateGPUScene(RHICmdList, *Scene);

	for (int32 ViewIndex = 0; ViewIndex < Views.Num(); ViewIndex++)
	{
		FViewInfo& View = Views[ViewIndex];
		UploadDynamicPrimitiveShaderDataForView(RHICmdList, *Scene, View);
	}

	const bool bIsOcclusionTesting = DoOcclusionQueries(FeatureLevel) && (!bIsWireframe || bIsViewFrozen || bHasViewParent);

	if (bShouldRenderSkyAtmosphere)
	{
		// Generate the Sky/Atmosphere look up tables
		RenderSkyAtmosphereLookUpTables(RHICmdList);
	}

	RunGPUSkinCacheTransition(RHICmdList, Scene, EGPUSkinCacheTransition::Renderer);

	// Before starting the render, all async task for the Custom data must be completed
	if (UpdateViewCustomDataEvents.Num() > 0)
	{
		QUICK_SCOPE_CYCLE_COUNTER(STAT_FDeferndershaddShadingSceneRenderer_AsyncUpdateViewCustomData_Wait);
		CSV_SCOPED_TIMING_STAT_EXCLUSIVE(AsyncUpdateViewCustomData_Wait);
		FTaskGraphInterface::Get().WaitUntilTasksComplete(UpdateViewCustomDataEvents, ENamedThreads::GetRenderThread());
	}

	if (bNeedsPrePass)
	{
		bDepthWasCleared = RenderPrePass(RHICmdList, AfterTasksAreStarted);
	}

#if RHI_RAYTRACING
	// Must be done after FGlobalDynamicVertexBuffer::Get().Commit() for dynamic geometries to be updated
	DispatchRayTracingWorldUpdates(RHICmdList);
#endif

	{
		SCOPED_GPU_STAT(RHICmdList, SortLights);
		GatherAndSortLights(SortedLightSet);
		ComputeLightGrid(RHICmdList, bComputeLightGrid, SortedLightSet);
	}

	{
		SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_AllocGBufferTargets);
		SceneContext.PreallocGBufferTargets(); // Even if !bShouldRenderVelocities, the velocity buffer must be bound because it's a compile time option for the shader.
		SceneContext.AllocGBufferTargets(RHICmdList);
	}

	// Early occlusion queries
	const bool bOcclusionBeforeBasePass = (EarlyZPassMode == EDepthDrawingMode::DDM_AllOccluders) || (EarlyZPassMode == EDepthDrawingMode::DDM_AllOpaque);

	if (bOcclusionBeforeBasePass)
	{
		RenderShadowDepthMaps(RHICmdList);
	}

	if (GetCustomDepthPassLocation() == 0)
	{
		RenderCustomDepthPassAtLocation(RHICmdList, 0);
	}

	if (bOcclusionBeforeBasePass)
	{
		// Run occlusion query here
	}

	if (IsForwardShadingEnabled(ShaderPlatform))
	{
		RenderForwardShadingShadowProjections(RHICmdList, ForwardScreenSpaceShadowMask);

		RenderIndirectCapsuleShadows(
			RHICmdList, 
			NULL, 
			NULL);
	}

	if (bRenderDeferredLighting)
	{
		SceneContext.AllocateDeferredShadingPathRenderTargets(RHICmdList);
	}

	SceneContext.BeginRenderingGBuffer

	RenderBasePass(RHICmdList, BasePassDepthStencilAccess, ForwardScreenSpaceShadowMask.GetReference(), bDoParallelBasePass, bRenderLightmapDensity);

	SceneContext.FinishGBufferPassAndResolve(RHICmdList, BasePassDepthStencilAccess);

	if (!bAllowReadonlyDepthBasePass)
	{
		SceneContext.ResolveSceneDepthTexture(RHICmdList, FResolveRect(0, 0, FamilySize.X, FamilySize.Y));
	}

	if (!bOcclusionBeforeBasePass)
	{
		// Run occlusion query here
	}

	if (!bUseGBuffer)
	{		
		ResolveSceneColor(RHICmdList);
	}

	// Shadow and fog after base pass
	if (!bOcclusionBeforeBasePass)
	{
		// Before starting the shadow render, all async task for the shadow Custom data must be completed
		if (bDoInitViewAftersPrepass && UpdateViewCustomDataEvents.Num() > 0)
		{
			QUICK_SCOPE_CYCLE_COUNTER(STAT_FDeferredShadingSceneRenderer_AsyncUpdateViewCustomData_Wait);
			FTaskGraphInterface::Get().WaitUntilTasksComplete(UpdateViewCustomDataEvents, ENamedThreads::GetRenderThread());
		}

		RenderShadowDepthMaps(RHICmdList);

		checkSlow(RHICmdList.IsOutsideRenderPass());

		ComputeVolumetricFog(RHICmdList);
		ServiceLocalQueue();
	}

#if RHI_RAYTRACING
	WaitForRayTracingScene(RHICmdList);
#endif	

	// Pre-lighting composition lighting stage
	// e.g. deferred decals, SSAO
	if (FeatureLevel >= ERHIFeatureLevel::SM5)
	{
		GCompositionLighting.ProcessAfterBasePass(RHICmdList, View);
	}

	// Render lighting.
	if (bRenderDeferredLighting)
	{
		RenderDiffuseIndirectAndAmbientOcclusion(RHICmdList);

		// These modulate the scenecolor output from the basepass, which is assumed to be indirect lighting
		RenderIndirectCapsuleShadows(
			RHICmdList, 
			SceneContext.GetSceneColorSurface(), 
			SceneContext.bScreenSpaceAOIsValid ? SceneContext.ScreenSpaceAO->GetRenderTargetItem().TargetableTexture : NULL);

		// These modulate the scenecolor output from the basepass, which is assumed to be indirect lighting
		RenderDFAOAsIndirectShadowing(RHICmdList, SceneContext.SceneVelocity, DynamicBentNormalAO);

		RenderLights(RHICmdList, SortedLightSet, HairDatas);

		// Render diffuse sky lighting and reflections that only operate on opaque pixels
		RenderDeferredReflectionsAndSkyLighting(RHICmdList, DynamicBentNormalAO, SceneContext.SceneVelocity, HairDatas);

		// SSS need the SceneColor finalized as an SRV.
		ResolveSceneColor(RHICmdList);

		ComputeSubsurfaceShim(RHICmdList, Views);
	}

	const bool bShouldRenderSingleLayerWater = ShouldRenderSingleLayerWater(Views, ViewFamily.EngineShowFlags);
	if(bShouldRenderSingleLayerWater)
	{
		...
	}


	// Draw Lightshafts
	if (ViewFamily.EngineShowFlags.LightShafts)
	{
		RenderLightShaftOcclusion(RHICmdList, LightShaftOutput);
	}

	// Draw atmosphere
	if (bCanOverlayRayTracingOutput && ShouldRenderAtmosphere(ViewFamily))
	{
		RenderAtmosphere(RHICmdList, LightShaftOutput);
	}

	// Draw the sky atmosphere
	if (bShouldRenderSkyAtmosphere)
	{
		RenderSkyAtmosphere(RHICmdList);
	}

	// Draw fog.
	if (bCanOverlayRayTracingOutput && ShouldRenderFog(ViewFamily))
	{
		RenderFog(RHICmdList, LightShaftOutput);
	}

	// Render translucency
	{
		...
	}

	// Render postprocess passes
}
```

# FDeferredShadingSceneRenderer::InitViews
```
bool FDeferredShadingSceneRenderer::InitViews(FRHICommandListImmediate& RHICmdList, FExclusiveDepthStencil::Type BasePassDepthStencilAccess, struct FILCUpdatePrimTaskData& ILCTaskData, FGraphEventArray& UpdateViewCustomDataEvents)
{
	// Set up motion blue params
	// TAA jittering
	// Dither fading unifor buffer
	PreVisibilityFrameSetup(RHICmdList);

	ComputeViewVisibility(RHICmdList, BasePassDepthStencilAccess, ViewCommandsPerView, DynamicIndexBufferForInitViews, DynamicVertexBufferForInitViews, DynamicReadBufferForInitViews);

	// Initialise Sky/View resources before the view global uniform buffer is built.
	if (ShouldRenderSkyAtmosphere(Scene, ViewFamily.EngineShowFlags))
	{
		InitSkyAtmosphereForViews(RHICmdList);
	}

	PostVisibilityFrameSetup(ILCTaskData);
}
```

# FSceneRenderer::ComputeViewVisibility
```
void FSceneRenderer::ComputeViewVisibility(FRHICommandListImmediate& RHICmdList, FExclusiveDepthStencil::Type BasePassDepthStencilAccess, FViewVisibleCommandsPerView& ViewCommandsPerView, 
	FGlobalDynamicIndexBuffer& DynamicIndexBuffer, FGlobalDynamicVertexBuffer& DynamicVertexBuffer, FGlobalDynamicReadBuffer& DynamicReadBuffer)
{
	for each view
	{
		// Call "ConditionalUpdateStaticMeshes" on Scene->PrimitivesNeedingStaticMeshUpdateWithoutVisibilityCheck
		// this is normally only done once for the static primitives in the scene

		// Setup View.PrimitiveVisibilityMap, etc
		
		// Add all the lights in the scene to View.VisibleLightInfos
		
		// Run "FrustumCull" for distance and frustum culling (internally uses ParallelFor for better performance)
		// Results stored in View.PrimitiveVisibilityMap

		// Call UpdateStaticMeshes on the combined result of View.PrimitiveVisibilityMap and Scene->PrimitivesNeedingStaticMeshUpdate
		// Normally do not run every frame for static primitives

		if (!bIsInstancedStereo)
		{
			// Call ComputeAndMarkRelevanceForViewParallel
			// Note that only the visible primitives from View.PrimitiveVisibilityMap are processed
		}
	}

	if (bIsInstancedStereo)
	{
		// Combine the "PrimitiveVisibilityMap" from each eyes
		// Call ComputeAndMarkRelevanceForViewParallel
	}

	// Call FSceneRenderer::GatherDynamicMeshElements to gather all the primitive with dynamic relevance
	// Eventually calls GetDynamicMeshElements

	for each view
	{
		// Call SetupMeshPass
	}
}
```

# FPrimitiveSceneInfo::CacheMeshDrawCommands

- Called when caching the mesh draw commands from this primitive to the scene
```
void FPrimitiveSceneInfo::CacheMeshDrawCommands(FRHICommandListImmediate& RHICmdList)
{
	for each FStaticMeshBatch in StaticMeshes
	{
		if (SupportsCachingMeshDrawCommands(Proxy, Mesh))
		{
			for each pass
			{
				if (pass supports mesh command caching)
				{
					// get the per-pass cached draw list from the scene
					FCachedPassMeshDrawList& SceneDrawList = Scene->CachedDrawLists[PassType];

					// create a draw list context
					FCachedPassMeshDrawListContext CachedPassMeshDrawListContext(CommandInfo, SceneDrawList, *Scene);

					// create the mesh processor for this pass
					PassProcessorCreateFunction CreateFunction = FPassProcessorManager::GetCreateFunction(ShadingPath, PassType);
					FMeshPassProcessor* PassMeshProcessor = CreateFunction(Scene, nullptr, &CachedPassMeshDrawListContext);

					// add the mesh batch
					PassMeshProcessor->AddMeshBatch(Mesh, BatchElementMask, Proxy);
				}
			}
		}
	}
}
```
