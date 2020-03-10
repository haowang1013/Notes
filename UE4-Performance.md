# Scalability

## Resolution

* `r.MobileContentScaleFactor`

Controls the resolution scaling of the final frame buffer (after UI is rendered). 0 means full resolution on iOS.

* `r.ScreenPercentage`

Controls the resolution scaling of the 3D rendering buffer, this won't affect UI resolution. 

100 means no scaling, otherwise there will be a upscale pass before the UI is rendered, which can be several ms on the mobile devices.

## Draw Distance

### Max Draw Distance

Use **CullDistanceVolume** or **DirectiveCullDistanceVolume** to automatically set the max draw distance for a group of objects. 

The latter is preferred as it scales the draw distance based on the size of the object, rather than using pre-defined buckets.

Note that the cull distance volume only affects static components. Moveable components need to have their max draw distance set manually.

To preview distance based culling in the editor, run **ShowFlag.DistanceCulledPrimitives 0**

* `r.ViewDistanceScale`

This variable affects the draw distance of all the objects uniformly. Larger value = more view distance.

### Foliage

* `foliage.MinimumScreenSize`



## Geometry

### Static Mesh

There's no cvar to scale the LOD globally but the **Minimum LOD** property can be set on the mesh itself, with different value for each platform.

### Skeletal Mesh

* `r.SkeletalMeshLODBias`

### Foliage

* `foliage.MinLOD`

* `foliage.LODDistanceScale`

* `foliage.MaxTrianglesToRender`

### Landscape

* `r.LandscapeLODBias`

* `r.LandscapeLODDistributionScale`



## Detail Mode

Each primitive component has a **Detail Mode** property which can be used to hide the component for any platform with a lower setting.

* `r.DetailMode`

Set the detail mode for the current platform.

## Feature Level Switch

Use the **Feature Level Switch** node in the material to create different quality/cost code path for different platforms.

# Content Optimization

## HLOD

HLOD is great for creating proxy mesh for a group of meshes that need to be seen from great distance. It combines the drawcalls, reduced the geometry complexity and often can use a much simpler material.

Smaller detailed objects should be excluded from HLOD as they don't contribute to the look meaningfully. This also means they need to have proper max draw distance set (see above), otherwise they'll be rendered all the time!

 - To verify HLOD is working in the game, run **show hlodcoloration**

 - To enable/disable HLOD, run **r.hlod 1/0**

 - To force HLOD level, run **r.hlod force <LEVEL>**

## Texture Mips

Most of the textures should have the mips generated, with exception for the ones used by UI. Make sure the textures use the mip-gen setting from its texture group.

## Mesh LODs

Most of the meshes should have LODs generated, as vertex processing can be quite expensive on mobile platforms. 

The cooker is able to validate the presence of LODs if the project is configured like this:

```
[/Script/Engine.MeshLODGenerationSettings]
; Missing LODs will be treated as error so the build can fail!
bLogErrorForMissingLODs=True

; Static mesh LOD requirement
StaticMeshMinVertices=200
StaticMeshNumRequiredLODs=3

; Static mesh auto LOD generation settings
StaticMeshDefaultAutoComputeLODScreenSizeFactor=0.5
StaticMeshDefaultNumOfLODLevelsToCastShadow=2

; Skeletal mesh LOD requirement
SkeletalMeshMinVertices=200
SkeletalMeshNumRequiredLODs=3
```

- To verify LOD is working in the game, run **show lodcoloration**
