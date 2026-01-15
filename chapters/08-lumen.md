# Chapter 8: Lumen Global Illumination

## Goals

By the end of this chapter, you will understand:
- What Lumen is and how it works
- Surface Cache and radiance caching architecture
- Hardware and software ray tracing modes
- Screen-space tracing and Distance Field tracing
- Performance optimization and debugging for Lumen

## Overview

Lumen is UE5's fully dynamic global illumination (GI) and reflections system. It eliminates the need for lightmap baking, providing real-time bounce lighting that responds instantly to changes in geometry, materials, and lighting.

**Key features**:
- **Fully dynamic**: No baking, no precomputation
- **Multi-bounce**: Supports multiple indirect light bounces
- **Scalable**: Works on consoles and high-end PCs
- **Material-aware**: Respects emissive materials, colors, roughness

**Lumen components**:
- **Surface Cache**: Stores surface shading for reuse
- **Radiance Cache**: Stores indirect lighting probes
- **Ray Tracing**: Hardware or software ray tracing
- **Screen Space**: Efficient tracing for visible surfaces

## Pipeline Steps

### 1. Surface Cache Update
**What happens**: Render scene geometry from multiple angles to cache surface appearance.

**Process**:
- Mesh cards (simplified geometry representation)
- Capture albedo, normals, emissive
- Update only changed surfaces (incremental)
- Store in atlas textures

### 2. Direct Lighting
**What happens**: Evaluate direct lighting and store in Surface Cache.

**For each surface**:
- Compute lighting from all lights
- Store direct lighting in Surface Cache
- Used as starting point for bounces

### 3. Ray Tracing (Diffuse Indirect)
**What happens**: Trace rays to gather indirect lighting.

**Per-pixel or per-probe**:
- Shoot rays into scene
- Hit Surface Cache
- Retrieve cached lighting
- Accumulate indirect illumination

### 4. Radiance Cache
**What happens**: Store indirect lighting in 3D probe grid.

**Probes**:
- Placed throughout scene volume
- Store irradiance (incoming light from all directions)
- Interpolated for final pixels
- Reused across multiple frames (temporal stability)

### 5. Screen-Space Tracing
**What happens**: Use screen-space data for high-frequency details.

**Process**:
- Trace rays in screen space (depth buffer)
- Fill gaps from radiance cache
- Blend screen-space and world-space results

### 6. Reflection Tracing
**What happens**: Similar to diffuse, but for specular reflections.

**Specular GI**:
- Trace reflection rays
- Sample Surface Cache or hit-light shader
- Rough reflections use wider cones

## Key UE Source Files, Classes, and Functions

### Lumen System Core

**`Engine/Source/Runtime/Renderer/Private/Lumen/Lumen.h`**
- `FLumenSceneData` - Scene representation for Lumen
- Lumen configuration and state

**`Engine/Source/Runtime/Renderer/Private/Lumen/Lumen.cpp`**
- `InitializeLumenSceneData()` - Initialize Lumen for scene
- Lumen system management

### Surface Cache

**`Engine/Source/Runtime/Renderer/Private/Lumen/LumenSurfaceCache.h`**
- `FLumenSurfaceCacheData` - Surface cache management
- Mesh card representation

**`Engine/Source/Runtime/Renderer/Private/Lumen/LumenSurfaceCache.cpp`**
- `UpdateSurfaceCache()` - Incrementally update surface cache
- Atlas allocation and management

**`Engine/Shaders/Private/Lumen/LumenSurfaceCache.usf`**
- Surface cache rendering shaders (HLSL)

### Radiance Cache

**`Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadianceCache.h`**
- `FLumenRadianceCacheInputs` - Radiance cache configuration
- Probe placement and management

**`Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadianceCache.cpp`**
- `RenderRadianceCache()` - Update radiance probes
- Probe interpolation

**`Engine/Shaders/Private/Lumen/LumenRadianceCache.usf`**
- Radiance cache tracing and interpolation (HLSL)

### Screen-Space Tracing

**`Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeGather.h`**
- Screen-space probe system
- Screen probe placement

**`Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeGather.cpp`**
- `RenderScreenProbeGather()` - Gather lighting via screen probes
- Adaptive screen probe placement

### Ray Tracing

**`Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneDirectLighting.cpp`**
- Direct lighting for Lumen
- Shadow tracing

**`Engine/Source/Runtime/Renderer/Private/Lumen/LumenReflections.cpp`**
- `RenderLumenReflections()` - Lumen specular reflections
- Reflection ray tracing

**`Engine/Shaders/Private/Lumen/LumenRadianceCacheHardwareRayTracing.usf`**
- Hardware ray tracing shaders (RTX/DXR)

**`Engine/Shaders/Private/Lumen/LumenSceneLighting.usf`**
- Software ray tracing (distance field, mesh SDF)

### Key Functions

```cpp
// Lumen.cpp
void InitializeLumenSceneData(
    FRDGBuilder& GraphBuilder,
    const FScene* Scene);

// LumenSurfaceCache.cpp
void UpdateLumenSurfaceCache(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    FLumenSceneData& LumenSceneData);

// LumenRadianceCache.cpp
void RenderRadianceCache(
    FRDGBuilder& GraphBuilder,
    const FLumenRadianceCacheInputs& Inputs,
    const FViewInfo& View,
    FLumenRadianceCacheState& RadianceCacheState);

// LumenReflections.cpp
void RenderLumenReflections(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FSceneTextures& SceneTextures,
    const FLumenReflectionInputs& Inputs,
    FRDGTextureRef& OutColor);
```

## Surface Cache

### What Is Surface Cache?

Surface Cache is Lumen's representation of scene geometry. Instead of rendering the full scene for every GI ray, Lumen renders simplified "cards" that cache surface appearance.

**Mesh Cards**:
- Each mesh approximated by several oriented cards (billboards)
- Cards capture major surface directions (e.g., box has 6 cards)
- Cached in atlas textures

**What's Cached**:
- **Albedo**: Base color
- **Opacity**: For foliage/alpha
- **Normals**: Surface orientation
- **Emissive**: Self-illumination
- **Direct lighting**: Pre-lit with direct lights

### Surface Cache Update

**Incremental updates**:
- Only update changed surfaces (moved objects, lighting changes)
- Most surfaces cached across frames (temporal reuse)
- New surfaces rendered on-demand

**Resolution**:
- Depends on mesh importance (size on screen, LOD)
- Distant objects get lower resolution cards
- Near objects get higher resolution

**Console variables**:
```
r.Lumen.SurfaceCache.Resolution=512        // Atlas resolution
r.Lumen.SurfaceCache.MaxSurfaceCacheTiles=2048  // Max tiles in atlas
r.Lumen.SurfaceCache.UpdateFrequency=1     // Update frequency (frames)
```

### Visualize Surface Cache

**Console commands**:
```
r.Lumen.Visualize.SurfaceCache=1           // Show surface cache cards
r.Lumen.Visualize.SurfaceCacheCoverage=1   // Show coverage
```

## Radiance Cache

### What Is Radiance Cache?

Radiance Cache stores indirect lighting in a sparse 3D grid of probes. Instead of tracing rays per-pixel (expensive), trace rays per-probe and interpolate results.

**Probe grid**:
- Probes placed in 3D space (octree structure)
- Denser near geometry, sparser in open space
- Each probe stores incoming radiance from all directions (spherical harmonics or octahedral)

**Benefits**:
- Amortizes ray tracing cost across multiple pixels
- Temporal stability (probes update slowly)
- Spatial filtering (interpolation smooths noise)

### Radiance Cache Update

**Per-frame**:
1. Determine which probes need updating (changed lighting, new views)
2. For each probe, trace rays
3. Accumulate radiance at probe
4. Store in radiance cache texture

**Interpolation**:
```hlsl
float3 SampleRadianceCache(float3 WorldPos, float3 Normal)
{
    // Find surrounding probes (8 nearest in octree)
    float3 Irradiance = 0;
    float TotalWeight = 0;
    
    for (int i = 0; i < 8; ++i)
    {
        FRadianceProbe Probe = GetProbe(ProbeIndices[i]);
        float3 ProbeIrradiance = EvaluateProbe(Probe, Normal);
        float Weight = ComputeProbeWeight(WorldPos, Probe);
        
        Irradiance += ProbeIrradiance * Weight;
        TotalWeight += Weight;
    }
    
    return Irradiance / max(TotalWeight, 0.001);
}
```

### Console Variables

```
r.Lumen.RadianceCache.ProbeResolution=16       // Probe resolution (texels per side)
r.Lumen.RadianceCache.NumProbeTracesBudget=200 // Max probes to trace per frame
r.Lumen.RadianceCache.SpatialFilterProbes=1    // Enable spatial filtering
```

## Ray Tracing Modes

### Hardware Ray Tracing (RTX/DXR)

**Requirements**:
- Ray tracing capable GPU (NVIDIA RTX, AMD RDNA2+, etc.)
- DX12 or Vulkan

**Benefits**:
- Highest quality
- Accurate ray-triangle intersection
- Supports any geometry

**Enable**:
```
r.Lumen.HardwareRayTracing=1               // Enable hardware RT for Lumen
r.RayTracing.Lumen.UseHardwareRayTracing=1 // Force hardware RT
```

### Software Ray Tracing (Distance Fields)

**Fallback** for GPUs without hardware RT:
- Uses signed distance fields (SDF)
- Sphere tracing through SDF
- Lower quality but still reasonable

**Distance Fields**:
- Pre-computed per mesh
- Stored as 3D volume texture
- Approximates mesh surface

**Enable**:
```
r.GenerateMeshDistanceFields=1             // Generate distance fields
r.DistanceFields.MaxPerMeshResolution=128  // SDF resolution
```

### Screen-Space Tracing

**For visible surfaces**:
- Trace rays in screen space (depth buffer)
- Very efficient (no world-space data needed)
- Limited to on-screen geometry

**Hybrid approach**:
- Use screen space for near/visible surfaces
- Fall back to radiance cache for off-screen

## Screen Probe Gather

### Adaptive Screen Probes

Lumen places probes on screen adaptively:
- More probes in high-detail areas
- Fewer probes in flat areas
- Probes trace rays into scene
- Results interpolated per-pixel

**Algorithm**:
1. Place screen probes (grid or adaptive)
2. Each probe traces rays (8-32 rays typical)
3. Probes sample Surface Cache or trace to geometry
4. Accumulate indirect lighting at probe
5. Interpolate between probes for final pixels

**Benefits**:
- Reduces ray count (vs per-pixel)
- Spatial filtering reduces noise
- Temporal accumulation for stability

**Console variables**:
```
r.Lumen.ScreenProbeGather.NumProbes=1024       // Number of screen probes
r.Lumen.ScreenProbeGather.TracingOctahedronResolution=8  // Rays per probe
r.Lumen.ScreenProbeGather.ScreenTracesMaxDistance=8000   // Max trace distance
```

## Reflections

### Lumen Reflections

Lumen provides dynamic specular reflections:
- Works for all roughness values (sharp and rough)
- Supports off-screen reflections (via radiance cache)
- Material-aware (reflects correct colors, emissives)

**Reflection tracing**:
```hlsl
float3 LumenReflection(
    float3 WorldPos,
    float3 WorldNormal,
    float3 ViewDir,
    float Roughness)
{
    float3 R = reflect(-ViewDir, WorldNormal);
    
    // Trace reflection ray
    FLumenRayHit Hit = TraceLumenRay(WorldPos, R, MaxDistance);
    
    if (Hit.bHit)
    {
        // Sample surface cache at hit point
        float3 HitRadiance = SampleSurfaceCache(Hit.Position, Hit.Normal);
        return HitRadiance;
    }
    else
    {
        // Fall back to sky light
        return SampleSkyLight(R, Roughness);
    }
}
```

**Roughness handling**:
- Sharp reflections: Single ray
- Rough reflections: Cone tracing (wider sampling)

### Console Variables

```
r.Lumen.Reflections.Enable=1                   // Enable Lumen reflections
r.Lumen.Reflections.MaxRoughnessToTrace=0.4    // Max roughness for tracing
r.Lumen.Reflections.ScreenSpaceReconstruction=1 // Use screen space
```

## Debugging Tips

### 1. Visualize Lumen Components

**Console commands**:
```
r.Lumen.Visualize.Mode=1              // Overview mode
r.Lumen.Visualize.SurfaceCache=1      // Surface cache cards
r.Lumen.Visualize.RadianceCache=1     // Radiance probes
r.Lumen.Visualize.ScreenProbeGather=1 // Screen probes
```

**Visualization modes**:
- 0: Off
- 1: Overview (shows all components)
- 2: Direct lighting
- 3: Indirect lighting
- 4: Radiance cache

### 2. Check Lumen Settings

**Console commands**:
```
r.Lumen.DumpStats=1                   // Dump Lumen statistics to log
```

**Verify**:
- Lumen enabled: `r.Lumen.Supported=1`
- Scene has distance fields (for software RT)
- Hardware RT detected (if using RTX)

### 3. Profile Lumen

**Console commands**:
```
stat Lumen         // Lumen-specific stats
stat SceneRendering // General rendering stats
profilegpu         // GPU profiler with Lumen breakdown
```

**Typical timings**:
- Surface cache: 1-3ms
- Radiance cache: 2-5ms
- Screen probe gather: 2-6ms
- Reflections: 2-4ms
- **Total**: 8-18ms (depends on settings)

### 4. Reduce Quality for Performance

**Lower settings**:
```
r.Lumen.TraceMeshSDFs=0               // Disable mesh SDFs (use distance fields only)
r.Lumen.ScreenProbeGather.NumProbes=512 // Fewer probes
r.Lumen.RadianceCache.ProbeResolution=8 // Lower probe resolution
```

**Scalability**:
Use Project Settings → Engine → Rendering → Global Illumination → Lumen quality presets.

### 5. Common Issues

**No GI visible**:
- Check `r.DynamicGlobalIlluminationMethod=1` (1=Lumen)
- Verify Post Process Volume has "Lumen Global Illumination" enabled
- Check scene has lights (GI needs direct lighting to bounce)

**Dark scenes**:
- Increase `r.Lumen.SurfaceCache.DirectLightingUpdateFactor=1`
- Verify emissive materials have high intensity
- Check sky light is present and visible

**Flickering/Noise**:
- Increase temporal accumulation: `r.Lumen.ScreenProbeGather.TemporalAccumulation=1`
- Lower roughness threshold: `r.Lumen.Reflections.MaxRoughnessToTrace=0.3`
- Enable radiance cache filtering: `r.Lumen.RadianceCache.SpatialFilterProbes=1`

**Leaking/Light Bleeding**:
- Improve geometry (close gaps, fix normals)
- Increase distance field resolution: `r.DistanceFields.MaxPerMeshResolution=256`
- Adjust bias: `r.Lumen.SurfaceCache.BiasMultiplier=1.0`

## Advanced Topics

### Emissive Materials

Lumen respects emissive materials:
- Emissive captured in Surface Cache
- Contributes to indirect lighting
- No need for separate light actors

**Setup**:
```cpp
// In material
EmissiveColor = BaseColor * Intensity;
```

**Console variable**:
```
r.Lumen.SurfaceCache.CaptureEmissive=1  // Capture emissive in surface cache
```

### Two-Sided Foliage

Foliage needs special handling:
- Two-Sided Foliage shading model
- Alpha-tested geometry in Surface Cache
- Subsurface scattering support

**Material settings**:
- Shading Model: Two-Sided Foliage
- Blend Mode: Masked
- Two Sided: Enabled

### Reflection Hierarchy

Lumen reflections blend with other reflection methods:
1. **Screen-space reflections** (SSR) - Highest detail, visible geometry only
2. **Lumen reflections** - Dynamic, off-screen support
3. **Reflection captures** - Fallback for distant/rough surfaces
4. **Sky light** - Final fallback

### Lumen for VR

Lumen works in VR but requires tuning:
- Lower screen probe count (each eye renders separately)
- Share radiance cache between eyes
- Potentially lower quality settings for 90/120 FPS target

**Console variables**:
```
r.Lumen.ScreenProbeGather.NumProbes=512  // Reduce for VR
vr.Lumen.RadianceCache.Shared=1          // Share between eyes
```

## Performance Considerations

### Scalability

**Hardware tiers**:
- **High-end (RTX 3070+)**: Full quality, hardware RT
- **Mid-range (Console, GTX 1660+)**: Software RT, medium settings
- **Low-end**: Consider disabling Lumen, use baked lighting

### Cost Breakdown

**Typical Lumen cost** (1080p, medium settings):
- Surface Cache: 1-2ms
- Radiance Cache: 2-4ms
- Screen Probes: 3-6ms
- Reflections: 2-3ms
- **Total**: ~10-15ms

**Optimization targets**:
- Reduce screen probe count
- Lower radiance cache probe resolution
- Reduce max trace distance
- Use hardware RT if available (often faster)

### Scene Optimization

**For best Lumen performance**:
- Reasonable poly count (Lumen is geometry-heavy)
- Good distance fields (for software RT)
- Avoid excessive small objects (many cards)
- Use Nanite for high-poly meshes (Chapter 9)

### Memory Usage

**Lumen memory**:
- Surface Cache: 100-300MB (depends on scene)
- Radiance Cache: 50-150MB
- Distance Fields: Varies by scene

**Monitor**:
```
stat Memory  // Shows memory usage
```

## Summary

Lumen provides:
1. **Surface Cache** - Caches scene geometry appearance
2. **Radiance Cache** - Stores indirect lighting in probes
3. **Ray Tracing** - Hardware or software ray tracing
4. **Screen Probes** - Efficient screen-space gathering
5. **Reflections** - Dynamic specular GI

Key classes: `FLumenSceneData`, `FLumenRadianceCacheInputs`

Lumen is a game-changer for UE5 - eliminates lightmap baking while providing high-quality, fully dynamic GI. Performance is good on modern hardware and continues to improve.

## Next Chapter

[Chapter 9: Nanite Virtualized Geometry](09-nanite.md) - Learn UE5's revolutionary virtualized geometry system.
