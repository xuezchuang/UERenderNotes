# Chapter 6: Shadow Rendering

## Goals

By the end of this chapter, you will understand:
- Shadow mapping fundamentals and UE5's implementation
- Cascaded Shadow Maps (CSM) for directional lights
- Virtual Shadow Maps (VSM) - UE5's modern shadow system
- Per-object shadows (point/spot lights)
- Shadow optimization and debugging techniques

## Overview

Shadows are critical for visual realism but also expensive. UE5 implements multiple shadow techniques optimized for different light types and scenarios. Virtual Shadow Maps (VSM), introduced in UE5, provide high-quality, scalable shadows with minimal artist tuning.

**Shadow techniques in UE5**:
- **Cascaded Shadow Maps (CSM)**: For directional lights (sun)
- **Virtual Shadow Maps (VSM)**: Modern, high-resolution shadow system
- **Traditional Shadow Maps**: For point and spot lights
- **Ray-Traced Shadows**: Optional, high-quality (RTX hardware)
- **Contact Shadows**: Screen-space micro-shadows

## Pipeline Steps

### 1. Shadow Setup
**What happens**: Determine which lights cast shadows and their resolution.

**Per-light decisions**:
- Does light cast shadows? (costly, can be disabled)
- Shadow map resolution (higher = sharper, more memory)
- Shadow distance (how far shadows are rendered)

### 2. Shadow Map Rendering
**What happens**: Render scene from light's perspective to depth maps.

**For each shadow-casting light**:
1. Compute light view/projection matrices
2. Render shadow casters (depth-only)
3. Store depth in shadow map texture
4. Optional: Filter shadow map (blur, VSM processing)

### 3. Shadow Projection
**What happens**: During lighting, sample shadow maps to determine visibility.

**Per-lit pixel**:
1. Transform pixel world position to light space
2. Sample shadow map at that position
3. Compare pixel depth with shadow map depth
4. If pixel depth > shadow map depth: in shadow
5. Apply shadow to lighting contribution

### 4. Shadow Filtering
**What happens**: Soften shadow edges for realism.

**Techniques**:
- **PCF (Percentage Closer Filtering)**: Sample shadow map multiple times, average results
- **PCSS (Percentage Closer Soft Shadows)**: Variable penumbra based on distance
- **VSM Filtering**: Efficient filtering using virtual pages

## Key UE Source Files, Classes, and Functions

### Shadow System Core

**`Engine/Source/Runtime/Renderer/Private/ShadowRendering.h`**
- `FProjectedShadowInfo` - Represents a single shadow map
- Shadow rendering interface and utilities

**`Engine/Source/Runtime/Renderer/Private/ShadowRendering.cpp`**
- `FProjectedShadowInfo::RenderDepth()` - Render shadow map depth
- Shadow setup and rendering logic

**`Engine/Source/Runtime/Renderer/Private/ShadowSetup.cpp`**
- `InitProjectedShadowInfo()` - Initialize shadow projection
- Shadow frustum calculation

### Cascaded Shadow Maps

**`Engine/Source/Runtime/Renderer/Private/ShadowSetup.cpp`**
- `SetupWholeSceneProjection()` - Setup CSM cascades
- Cascade split computation

**`Engine/Shaders/Private/ShadowProjectionCommon.ush`**
- CSM sampling and filtering (HLSL)
- Cascade selection logic

### Virtual Shadow Maps

**`Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.h`**
- `FVirtualShadowMapArray` - Manages VSM system
- Virtual page allocation and management

**`Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.cpp`**
- `RenderVirtualShadowMaps()` - Main VSM rendering
- Page marking and caching

**`Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapProjection.ush`**
- VSM sampling (HLSL)
- Page lookup and filtering

### Shadow Mesh Processor

**`Engine/Source/Runtime/Renderer/Private/ShadowDepthRendering.h`**
- `FShadowDepthPassMeshProcessor` - Processes meshes for shadow depth
- Shadow-specific vertex factories

**`Engine/Source/Runtime/Renderer/Private/ShadowDepthRendering.cpp`**
- `AddMeshBatch()` - Add mesh to shadow pass
- Shadow shader selection

### Key Functions

```cpp
// ShadowRendering.cpp
void FProjectedShadowInfo::RenderDepth(
    FRHICommandListImmediate& RHICmdList,
    FSceneRenderer* SceneRenderer,
    const FViewInfo* FoundView,
    bool bDoParallelDispatch);

// ShadowSetup.cpp
void FProjectedShadowInfo::SetupWholeSceneProjection(
    FLightSceneInfo* InLightSceneInfo,
    FViewInfo* InDependentView,
    const FWholeSceneProjectedShadowInitializer& Initializer,
    uint32 InResolutionX,
    uint32 InResolutionY,
    uint32 InBorderSize);

// VirtualShadowMapArray.cpp
void FVirtualShadowMapArray::RenderVirtualShadowMaps(
    FRDGBuilder& GraphBuilder,
    const TArray<FProjectedShadowInfo*, SceneRenderingAllocator>& VirtualSmInvalidatingInstancesArray,
    bool bNaniteEnabled);
```

## Cascaded Shadow Maps (CSM)

### What Are Cascades?

Directional lights (sun) affect the entire scene, but shadow resolution is limited. CSM divides the view frustum into multiple cascades (splits), each with its own shadow map.

**Cascade strategy**:
- Near cascade: High resolution, small area (close to camera)
- Far cascade: Lower resolution, large area (far from camera)
- Typically 2-4 cascades

### Cascade Splitting

**Common split schemes**:

**Logarithmic**:
```cpp
float SplitDistance = Near * pow(Far / Near, i / NumCascades);
```

**Linear**:
```cpp
float SplitDistance = Near + (Far - Near) * (i / NumCascades);
```

**Practical (blend)**:
```cpp
float Log = Near * pow(Far / Near, i / NumCascades);
float Linear = Near + (Far - Near) * (i / NumCascades);
float SplitDistance = lerp(Linear, Log, 0.5); // Blend factor
```

### CSM Rendering

**Per-cascade**:
1. Compute cascade frustum (slice of view frustum)
2. Fit shadow frustum to cascade bounds
3. Render shadow casters to shadow map
4. Store cascade info (view/proj matrices, split distance)

**At runtime**:
```hlsl
// Select cascade based on pixel depth
float PixelDepth = SceneDepth;
int CascadeIndex = SelectCascade(PixelDepth, CascadeSplitDistances);

// Sample shadow map for selected cascade
float4 ShadowPosition = mul(float4(WorldPosition, 1), CascadeViewProj[CascadeIndex]);
float ShadowDepth = ShadowMaps[CascadeIndex].Sample(ShadowPosition.xy);
float Shadow = (ShadowPosition.z > ShadowDepth) ? 0 : 1;
```

### CSM Console Variables

```
r.Shadow.CSM.MaxCascades=4              // Number of cascades (1-4)
r.Shadow.MaxResolution=2048             // Max shadow map resolution per cascade
r.Shadow.DistanceScale=1.0              // Scale shadow distance
r.Shadow.FadeExponent=0.25              // Shadow fade sharpness
r.Shadow.CSMTransitionScale=1.0         // Cascade transition smoothness
```

### CSM Artifacts

**Peter Panning**:
- Shadows detached from objects
- Caused by shadow bias
- Fix: Reduce `r.Shadow.NormalOffsetScale`

**Shadow acne**:
- Self-shadowing artifacts (striping)
- Caused by insufficient bias
- Fix: Increase `r.Shadow.DepthBias`

**Cascade seams**:
- Visible boundaries between cascades
- Fix: Increase `r.Shadow.CSMTransitionScale`

## Virtual Shadow Maps (VSM)

### What Are Virtual Shadow Maps?

VSM is UE5's modern shadow system, providing very high-resolution shadows (up to 16K×16K) with efficient memory usage through virtualization.

**Key features**:
- **Clipmap hierarchy**: Multiple levels of detail (like terrain clipmaps)
- **Page-based allocation**: Only allocate shadow texels that are visible
- **Caching**: Reuse shadow pages across frames
- **Nanite integration**: Seamless with Nanite geometry

### How VSM Works

**Clipmap levels**:
```
Level 0: 16K×16K (highest detail, near camera)
Level 1: 8K×8K (medium detail)
Level 2: 4K×4K (lower detail)
Level 3: 2K×2K (lowest detail, far from camera)
```

**Page system**:
- Each level divided into 128×128 pixel pages
- Pages allocated on-demand (visible shadow regions only)
- Non-visible pages not allocated (saves memory)

**Rendering**:
1. **Mark pages**: Determine which pages are visible
2. **Allocate pages**: Allocate physical memory for marked pages
3. **Render pages**: Render shadow depth to allocated pages
4. **Cache pages**: Reuse pages from previous frame when possible

### VSM Benefits

**Compared to CSM**:
- Much higher resolution (16K vs 2-4K)
- No cascade seams (unified clipmap)
- Better with dynamic objects (per-object caching)
- Scales to many lights (each light gets its own VSM)

**Memory efficiency**:
- Only allocates visible shadow texels
- Typical scenes use 50-100MB for all shadows (vs 100s of MB for high-res CSMs)

### VSM Console Variables

```
r.Shadow.Virtual.Enable=1                     // Enable VSM
r.Shadow.Virtual.ResolutionLodBiasDirectional=-0.5  // Quality bias (lower = sharper)
r.Shadow.Virtual.MaxPhysicalPages=2048        // Physical page pool size
r.Shadow.Virtual.Cache=1                      // Enable page caching
r.Shadow.Virtual.ShowStats=1                  // Show VSM statistics
```

### VSM Debugging

**Visualize VSM**:
```
r.Shadow.Virtual.Visualize=1  // Show VSM pages
```

**Colors indicate**:
- Green: Cached pages (reused from previous frame)
- Red: Newly rendered pages
- Blue: Invalid pages

**Statistics**:
```
stat VSM  // Show VSM rendering statistics
```

Displays:
- Page allocations
- Cache hit rate
- Rendering time

## Point and Spot Light Shadows

### Shadow Cube Maps (Point Lights)

Point lights emit in all directions, requiring 6 shadow maps (cube faces):
```
+X, -X, +Y, -Y, +Z, -Z
```

**Optimization**:
- Only render visible faces (cull based on light/camera positions)
- Lower resolution than directional shadows (typically 512-1024)

**Console variables**:
```
r.Shadow.MaxResolution=1024           // Per-face resolution
r.Shadow.RadiusThreshold=0.01         // Minimum light radius for shadows
```

### Shadow Projections (Spot Lights)

Spot lights use single shadow map (perspective projection):
- Simpler than point lights (one projection)
- Fits cone frustum tightly

### Per-Object Shadows

For movable lights, objects can have individual shadow maps:
- Better quality than whole-scene shadows
- More expensive (more shadow maps)
- Useful for characters, important objects

**Enable**:
```cpp
// On mesh component
StaticMeshComponent->CastShadow = true;
StaticMeshComponent->bCastDynamicShadow = true;
```

## Shadow Filtering

### PCF (Percentage Closer Filtering)

Sample shadow map multiple times, average results:

```hlsl
float PCF_Shadow(
    Texture2D ShadowMap,
    SamplerState ShadowSampler,
    float2 ShadowUV,
    float CompareDepth,
    int FilterSize)
{
    float Shadow = 0;
    int SampleCount = 0;
    
    for (int x = -FilterSize; x <= FilterSize; ++x)
    {
        for (int y = -FilterSize; y <= FilterSize; ++y)
        {
            float2 Offset = float2(x, y) * ShadowTexelSize;
            float SampleDepth = ShadowMap.Sample(ShadowSampler, ShadowUV + Offset).r;
            Shadow += (CompareDepth > SampleDepth) ? 0 : 1;
            SampleCount++;
        }
    }
    
    return Shadow / SampleCount;
}
```

**Quality**:
```
r.Shadow.FilterMethod=1  // 0=Point, 1=PCF, 2=VSM
r.Shadow.Quality=5       // PCF kernel size (0-5)
```

### PCSS (Percentage Closer Soft Shadows)

Variable penumbra (soft shadow) based on blocker distance:

```hlsl
float PCSS_Shadow(...)
{
    // 1. Blocker search: Find average blocker depth
    float AvgBlockerDepth = FindBlockerDepth(...);
    
    // 2. Penumbra size: Compute soft shadow size
    float PenumbraSize = (CompareDepth - AvgBlockerDepth) / AvgBlockerDepth;
    
    // 3. PCF with variable filter size
    return PCF_Shadow(..., PenumbraSize);
}
```

**Result**: Shadows soft near edges, sharp near contact (more realistic).

### Contact Shadows

Screen-space shadows for fine detail:

**Algorithm**:
- Ray-march in screen space from pixel toward light
- Check if ray hits geometry (depth buffer)
- If hit, apply contact shadow

**Benefits**:
- Captures small details (fingers, small objects)
- Complements shadow maps

**Console variables**:
```
r.ContactShadows=1               // Enable contact shadows
r.ContactShadows.Length=1.0      // Ray length (world space)
r.ContactShadows.NonShadowCastingIntensity=0.0  // Intensity for non-shadow-casting objects
```

## Ray-Traced Shadows

### Hardware Ray Tracing

If RTX/DXR hardware is available:
- Trace rays from pixels toward lights
- Check for occlusion
- Very high quality (sharp or soft shadows with accurate penumbra)

**Enable**:
```
r.RayTracing.Shadows=1            // Enable RT shadows
r.RayTracing.Shadows.SamplesPerPixel=1  // Quality vs performance
```

**Limitations**:
- Requires ray-tracing capable GPU
- More expensive than shadow maps
- Can be used selectively (per-light setting)

## Debugging Tips

### 1. Visualize Shadow Maps

**Console commands**:
```
vis ShadowMap           // Show shadow map texture
r.Shadow.Visualize=1    // Overlay shadow coverage
```

**RenderDoc**:
- Capture frame
- View shadow map render targets
- Inspect per-pixel shadow values

### 2. Check Shadow Resolution

**Too low resolution**:
- Blocky, pixelated shadows
- Fix: Increase `r.Shadow.MaxResolution`

**Too high resolution**:
- Performance hit
- Fix: Reduce resolution, use VSM

### 3. Shadow Distance

**Console variable**:
```
r.Shadow.DistanceScale=1.0  // Adjust shadow distance
```

**Visualize**:
```
viewmode LightingOnly  // See where shadows fade out
```

### 4. Profile Shadow Rendering

**Console commands**:
```
stat SceneRendering  // Shows shadow pass timing
profilegpu           // Detailed shadow breakdown
```

**Typical timings**:
- Directional CSM: 2-6ms
- Directional VSM: 1-3ms (with caching)
- Point light shadows: 0.5-2ms per light
- Spot light shadows: 0.3-1ms per light

### 5. Shadow Artifacts

**Flickering shadows**:
- Temporal instability (moving cascades/pages)
- Fix: Increase cascade transition zone, enable VSM caching

**Light bleeding**:
- Shadows too soft, light leaks through
- Fix: Reduce shadow softness, check shadow bias

**Missing shadows**:
- Object outside shadow frustum/distance
- Fix: Increase shadow distance, check "Cast Shadow" setting

### 6. VSM-Specific Debugging

**Show page usage**:
```
r.Shadow.Virtual.Visualize=1
```

**Check cache efficiency**:
```
stat VSM
```

**Look for**:
- Low cache hit rate (< 50%): Many pages re-rendering each frame
- High page allocations: May need more physical pages

## Advanced Topics

### Inset Shadows

Per-object shadows with tighter bounds:
- Higher quality for important objects
- More expensive (additional shadow maps)

**Enable**:
```cpp
// On light component
DirectionalLight->DynamicShadowDistanceMovableLight = 3000;  // Per-object shadow distance
```

### Distance Field Shadows

Use distance fields for soft shadows:
- Pre-computed distance field per mesh
- Ray-march through distance field for shadows
- Soft shadows without expensive filtering

**Enable**:
```
r.DistanceFieldShadowing=1  // Enable distance field shadows
r.GenerateMeshDistanceFields=1  // Generate distance fields
```

### Capsule Shadows

Character-specific shadow approximation:
- Physics capsules cast shadows
- Very cheap
- Good for character self-shadowing

**Enable**:
```cpp
// On skeletal mesh component
SkeletalMeshComponent->bCastCapsuleDirectShadow = true;
```

### Subsurface Shadows

Shadows for subsurface materials (skin, wax):
- Shadows attenuate based on material thickness
- Light penetrates surface slightly

**Transmission**:
```hlsl
float Transmission = saturate(1 - Shadow) * SubsurfaceOpacity;
float3 SubsurfaceColor = BaseColor * LightColor * Transmission;
```

## Performance Considerations

### Shadow Resolution vs. Performance

**General guidelines**:
- Directional: 2048-4096 (CSM) or VSM
- Point: 512-1024 per face
- Spot: 512-2048

**Balance**:
- Higher resolution = sharper shadows, more memory/fill-rate
- VSM provides best quality/performance balance

### Shadow Distance

Shorter distance = better performance:
```
r.Shadow.DistanceScale=0.5  // Reduce shadow distance by 50%
```

**Tune per-project**:
- Open world: Longer distance needed
- Indoor: Shorter distance acceptable

### Shadow Caching

**Static shadows**:
- Bake to lightmaps (no runtime cost)
- Use for static geometry

**Stationary lights**:
- Cache indirect lighting
- Dynamic shadows for movable objects only

**VSM caching**:
- Automatically reuses pages (30-90% cache hit typical)
- Huge performance win for static/mostly-static scenes

### Limit Shadow-Casting Lights

Not all lights need shadows:
- Disable shadows on fill lights
- Use light functions instead of shadows for simple patterns

**Per-light setting**:
```cpp
PointLight->CastShadows = false;  // Disable shadows
```

## Summary

Shadow rendering:
1. **Setup** - Determine lights, resolution, distance
2. **Shadow Map Rendering** - Render depth from light's view
3. **Projection** - Sample shadow maps during lighting
4. **Filtering** - Soften shadows (PCF, PCSS)

Key techniques:
- **CSM**: Cascaded shadows for directional lights
- **VSM**: High-resolution, page-based virtual shadows
- **Shadow Maps**: Traditional approach for point/spot lights

Key classes: `FProjectedShadowInfo`, `FVirtualShadowMapArray`

VSM is recommended for most UE5 projects - better quality and performance than CSM.

## Next Chapter

[Chapter 7: Post Processing](07-postprocess.md) - Learn how UE5 applies post-process effects like tone mapping, bloom, and color grading.
