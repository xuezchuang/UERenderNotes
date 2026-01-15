# Chapter 5: Lighting System

## Goals

By the end of this chapter, you will understand:
- UE5's deferred and forward lighting pipelines
- Different light types and their rendering techniques
- Light culling and tiled/clustered lighting
- Image-based lighting (IBL) and reflection captures
- How to debug and optimize lighting performance

## Overview

After the BasePass fills the GBuffer with material properties (Chapter 4), the lighting system computes illumination. UE5 uses deferred lighting for most scenes, evaluating lights in screen space using GBuffer data.

**Main lighting components**:
- **Direct lighting**: Point, spot, directional, rect lights
- **Indirect lighting**: Sky light, reflection captures, Lumen (Chapter 8)
- **Emissive**: Self-illuminated materials
- **Light functions**: Projected textures and light masks

## Pipeline Steps

### 1. Light Culling
**What happens**: Determine which lights affect which pixels/tiles.

**Techniques**:
- **Per-pixel**: Check light bounds against pixel depth
- **Tiled**: Divide screen into tiles, cull lights per-tile
- **Clustered**: 3D grid of clusters, assign lights to clusters

### 2. Direct Lighting Pass
**What happens**: For each light, accumulate its contribution.

**Per-light**:
1. Bind light parameters (color, position, attenuation)
2. Determine affected pixels (light bounds)
3. Sample GBuffer for material properties
4. Compute lighting equation (BRDF)
5. Accumulate to lighting buffer

### 3. Indirect Lighting
**What happens**: Apply ambient lighting from environment.

**Sources**:
- Sky light (captures distant environment)
- Reflection captures (local reflections)
- Lightmaps (baked lighting)
- Lumen (dynamic GI, Chapter 8)

### 4. Light Composition
**What happens**: Combine lighting with base color.

**Formula**:
```
FinalColor = BaseColor × (DirectLighting + IndirectLighting) + Emissive
```

### 5. Translucency Lighting
**What happens**: Light transparent objects (forward rendered).

**Per-transparent object**:
- Evaluate lighting in forward shader
- No GBuffer available (transparent)
- Limited light count (expensive)

## Key UE Source Files, Classes, and Functions

### Deferred Lighting

**`Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp`**
- `FDeferredShadingSceneRenderer::RenderLights()` - Main lighting entry point
- `RenderLight()` - Render a single light
- Orchestrates lighting passes

**`Engine/Source/Runtime/Renderer/Private/LightRendering.cpp`**
- Light rendering implementation
- `RenderLight_Direct()` - Direct lighting for a single light
- Light shader parameter setup

**`Engine/Source/Runtime/Renderer/Private/LightSceneInfo.h`**
- `FLightSceneInfo` - Represents a light in the scene
- Contains light properties and render state

### Light Types

**`Engine/Source/Runtime/Engine/Classes/Components/LightComponent.h`**
- `ULightComponent` - Base class for all lights
- Intensity, color, attenuation settings

**`Engine/Source/Runtime/Engine/Classes/Components/PointLightComponent.h`**
- `UPointLightComponent` - Omnidirectional point light

**`Engine/Source/Runtime/Engine/Classes/Components/SpotLightComponent.h`**
- `USpotLightComponent` - Cone-shaped spotlight

**`Engine/Source/Runtime/Engine/Classes/Components/DirectionalLightComponent.h`**
- `UDirectionalLightComponent` - Infinite directional light (sun)

**`Engine/Source/Runtime/Engine/Classes/Components/RectLightComponent.h`**
- `URectLightComponent` - Area light (rectangle shape)

### Light Culling

**`Engine/Source/Runtime/Renderer/Private/LightGridInjection.cpp`**
- `FLightGridInjectionCS` - Compute shader for light grid generation
- Implements clustered/tiled light culling

**`Engine/Source/Runtime/Renderer/Private/VolumetricLightmapVoxelization.cpp`**
- Light voxelization for volumetric effects

### Shaders

**`Engine/Shaders/Private/DeferredLightingCommon.ush`**
- Shared HLSL code for deferred lighting
- BRDF evaluation functions
- Light attenuation calculations

**`Engine/Shaders/Private/DeferredLightPixelShaders.usf`**
- Pixel shaders for deferred lights
- Different shaders per light type

**`Engine/Shaders/Private/ReflectionEnvironmentShaders.usf`**
- Reflection capture sampling
- Image-based lighting

### Key Functions

```cpp
// DeferredShadingRenderer.cpp
void FDeferredShadingSceneRenderer::RenderLights(
    FRDGBuilder& GraphBuilder,
    FMinimalSceneTextures& SceneTextures,
    const FTranslucencyLightingVolumeTextures& TranslucentLightingVolumes,
    FRDGTextureRef LightingChannelsTexture,
    FSortedLightSetSceneInfo& SortedLightSet);

// LightRendering.cpp
void RenderLight(
    FRHICommandList& RHICmdList,
    const FLightSceneInfo* LightSceneInfo,
    const FViewInfo& View,
    const FLightShaderParameters& LightParameters,
    bool bRenderOverlap);

// LightSceneInfo.cpp
void FLightSceneInfo::CreateLightPrimitiveInteraction(
    const FPrimitiveSceneInfo* PrimitiveSceneInfo);
```

## Light Types and Rendering

### Directional Light (Sun)

**Properties**:
- Infinite distance (parallel rays)
- Affects entire scene
- Most common for sun/moon

**Rendering**:
```cpp
// Full-screen pass (affects all pixels)
// No distance attenuation
float3 DirectionalLighting(
    float3 WorldPosition,
    float3 WorldNormal,
    float3 LightDirection,
    float3 LightColor)
{
    float NoL = saturate(dot(WorldNormal, -LightDirection));
    return LightColor * NoL;
}
```

**Performance**:
- Single full-screen pass
- Cheap (no attenuation, simple shadow lookup)

**Console variables**:
```
r.DirectionalLight.Quality=2  // 0-2, higher = better quality
```

### Point Light

**Properties**:
- Omnidirectional (light radiates in all directions)
- Attenuates with distance
- Defined by position and radius

**Rendering**:
```cpp
// Render light sphere geometry
// Only pixels inside sphere are shaded
float3 PointLightShading(
    float3 WorldPosition,
    float3 WorldNormal,
    float3 LightPosition,
    float LightRadius,
    float3 LightColor)
{
    float3 L = LightPosition - WorldPosition;
    float Distance = length(L);
    L /= Distance;
    
    // Distance attenuation
    float Attenuation = saturate(1 - (Distance / LightRadius));
    Attenuation *= Attenuation; // Squared falloff
    
    float NoL = saturate(dot(WorldNormal, L));
    return LightColor * NoL * Attenuation;
}
```

**Optimization**:
- Render sphere geometry (hull) matching light radius
- Only shade pixels inside sphere
- Backface culling for camera inside light

### Spot Light

**Properties**:
- Cone-shaped illumination
- Defined by position, direction, inner/outer cone angles
- Attenuates with distance and angle

**Rendering**:
```cpp
float3 SpotLightShading(
    float3 WorldPosition,
    float3 WorldNormal,
    float3 LightPosition,
    float3 LightDirection,
    float InnerConeAngle,
    float OuterConeAngle,
    float LightRadius,
    float3 LightColor)
{
    float3 L = LightPosition - WorldPosition;
    float Distance = length(L);
    L /= Distance;
    
    // Distance attenuation
    float DistanceAttenuation = saturate(1 - (Distance / LightRadius));
    DistanceAttenuation *= DistanceAttenuation;
    
    // Spot angle attenuation
    float CosOuter = cos(OuterConeAngle);
    float CosInner = cos(InnerConeAngle);
    float SpotAngle = dot(L, -LightDirection);
    float SpotAttenuation = saturate((SpotAngle - CosOuter) / (CosInner - CosOuter));
    
    float NoL = saturate(dot(WorldNormal, L));
    return LightColor * NoL * DistanceAttenuation * SpotAttenuation;
}
```

**Geometry**:
- Render cone geometry matching spotlight volume
- More complex than point light sphere

### Rect Light (Area Light)

**Properties**:
- Rectangular area that emits light
- Soft shadows
- More physically accurate than point lights

**Rendering**:
```cpp
// Representative point technique (approximate)
float3 RectLightShading(
    float3 WorldPosition,
    float3 WorldNormal,
    float3 RectCenter,
    float3 RectNormal,
    float RectWidth,
    float RectHeight,
    float3 LightColor)
{
    // Find closest point on rect to shading point
    float3 ClosestPoint = ClosestPointOnRect(
        WorldPosition, RectCenter, RectNormal, RectWidth, RectHeight);
    
    float3 L = normalize(ClosestPoint - WorldPosition);
    float Distance = length(ClosestPoint - WorldPosition);
    
    // Inverse square falloff
    float Attenuation = 1.0 / (Distance * Distance + 1.0);
    
    float NoL = saturate(dot(WorldNormal, L));
    return LightColor * NoL * Attenuation;
}
```

**Quality**:
- More expensive than point/spot lights
- Can disable on lower-end platforms

**Console variable**:
```
r.RectLight.Enable=1  // 0 = disable rect lights
```

## Light Culling Techniques

### Simple Light Culling

**Per-light**:
- Check if pixel depth is within light bounds
- Skip lighting if outside

```hlsl
// Simple depth-based culling
float PixelDepth = SceneDepth;
float3 WorldPosition = ReconstructWorldPosition(ScreenUV, PixelDepth);
float DistanceToLight = length(LightPosition - WorldPosition);

if (DistanceToLight > LightRadius)
{
    discard; // Or return 0
}
```

### Tiled Deferred Lighting

**Algorithm**:
1. Divide screen into tiles (e.g., 16×16 pixels)
2. For each tile, determine which lights affect it
3. Store light list per tile
4. Shade pixels using their tile's light list

**Benefits**:
- Reduces redundant light culling (cull per-tile, not per-pixel)
- Better memory coherency
- Fewer shader invocations

**Implementation**:
```cpp
// Compute shader pass
[numthreads(TILE_SIZE, TILE_SIZE, 1)]
void TileLightCullingCS(uint3 GroupId : SV_GroupID, uint3 GroupThreadId : SV_GroupThreadID)
{
    // Find tile bounds in view space
    float2 TileMin = GroupId.xy * TILE_SIZE;
    float2 TileMax = TileMin + TILE_SIZE;
    
    float MinDepth = FindMinDepthInTile(TileMin, TileMax);
    float MaxDepth = FindMaxDepthInTile(TileMin, TileMax);
    
    // Cull lights against tile frustum
    for (uint LightIndex = 0; LightIndex < NumLights; ++LightIndex)
    {
        if (LightIntersectsTile(Lights[LightIndex], TileMin, TileMax, MinDepth, MaxDepth))
        {
            AppendTileLightList(LightIndex);
        }
    }
}
```

### Clustered Deferred Lighting

**Algorithm**:
1. Divide view frustum into 3D grid of clusters (x, y, depth)
2. Assign lights to clusters they intersect
3. For each pixel, find its cluster and use that cluster's light list

**Benefits**:
- Better depth precision than tiled
- Handles overlapping lights better
- Scales to many lights

**UE5 implementation**:
```
r.LightCulling.Clustered=1  // Enable clustered lighting
```

**Visualization**:
```
r.LightCulling.Visualize=1  // Visualize light culling grid
```

## Physically-Based Lighting (PBR)

### BRDF (Bidirectional Reflectance Distribution Function)

UE5 uses a microfacet BRDF model:

```hlsl
float3 SpecularGGX(
    float Roughness,
    float3 SpecularColor,
    float3 N, // Normal
    float3 V, // View direction
    float3 L) // Light direction
{
    float3 H = normalize(V + L);
    float NoH = saturate(dot(N, H));
    float NoV = saturate(dot(N, V));
    float NoL = saturate(dot(N, L));
    float VoH = saturate(dot(V, H));
    
    // GGX normal distribution
    float a = Roughness * Roughness;
    float a2 = a * a;
    float d = (NoH * a2 - NoH) * NoH + 1;
    float D = a2 / (PI * d * d);
    
    // Schlick-GGX geometry term
    float k = (Roughness + 1) * (Roughness + 1) / 8;
    float G_V = NoV / (NoV * (1 - k) + k);
    float G_L = NoL / (NoL * (1 - k) + k);
    float G = G_V * G_L;
    
    // Fresnel (Schlick approximation)
    float3 F = SpecularColor + (1 - SpecularColor) * pow(1 - VoH, 5);
    
    return (D * G * F) / (4 * NoV * NoL + 0.001);
}

float3 DiffuseLambert(float3 DiffuseColor, float NoL)
{
    return DiffuseColor * NoL / PI;
}

// Complete lighting
float3 StandardShading(
    float3 DiffuseColor,
    float3 SpecularColor,
    float Roughness,
    float3 N, float3 V, float3 L)
{
    float NoL = saturate(dot(N, L));
    
    float3 Diffuse = DiffuseLambert(DiffuseColor, NoL);
    float3 Specular = SpecularGGX(Roughness, SpecularColor, N, V, L);
    
    return (Diffuse + Specular) * NoL;
}
```

**Shader location**: `Engine/Shaders/Private/BRDF.ush`

### Energy Conservation

PBR ensures energy conservation:
- Reflected light ≤ incoming light
- Rougher surfaces have wider, dimmer highlights
- Metallic surfaces have no diffuse (only specular)

## Image-Based Lighting (IBL)

### Sky Light

**Purpose**: Ambient lighting from sky/environment.

**Implementation**:
1. Capture environment to cubemap
2. Filter cubemap for different roughness levels (mip chain)
3. At runtime, sample based on view reflection vector and roughness

```hlsl
float3 SkyLightReflection(
    float3 WorldNormal,
    float3 ViewDirection,
    float Roughness)
{
    float3 R = reflect(-ViewDirection, WorldNormal);
    
    // Sample pre-filtered environment map
    float MipLevel = Roughness * MaxMipLevel;
    float3 SkylightColor = TextureCubeSampleLevel(
        SkyLightCubemap, 
        SkyLightSampler, 
        R, 
        MipLevel).rgb;
    
    return SkylightColor;
}
```

**Console variables**:
```
r.SkyLight.RealTimeCapture=1      // Capture every frame (expensive)
r.SkyLight.SamplesPerPixel=1      // Quality vs performance
```

### Reflection Captures

**Types**:
- **Sphere capture**: Captures from a point, projects as sphere
- **Box capture**: Captures from a point, projects as box (for rooms)

**Placement**:
- Place in important areas (rooms, corridors)
- Smaller captures override larger ones (based on priority)

**Blending**:
```hlsl
// Blend multiple reflection captures
float3 BlendedReflection = 0;
float TotalWeight = 0;

for (int i = 0; i < NumCaptures; ++i)
{
    float Weight = CalculateCaptureWeight(WorldPosition, Captures[i]);
    BlendedReflection += SampleReflectionCapture(Captures[i], R) * Weight;
    TotalWeight += Weight;
}

BlendedReflection /= max(TotalWeight, 0.001);
```

**Console variables**:
```
r.ReflectionCaptureResolution=128  // Cubemap resolution
r.ReflectionEnvironment=1          // Enable/disable reflections
```

### Screen Space Reflections (SSR)

**Purpose**: High-quality reflections of on-screen content.

**Algorithm**:
1. For each pixel, trace ray in reflection direction
2. March through screen-space depth buffer
3. Sample screen color where ray hits surface

**Limitations**:
- Only reflects visible geometry
- No reflections of off-screen objects
- Falls back to reflection captures

**Console variables**:
```
r.SSR.Quality=3           // 0-4, quality level
r.SSR.MaxRoughness=0.8    // Max roughness for SSR
```

## Debugging Tips

### 1. Visualize Lighting Components

**Console commands**:
```
viewmode LightingOnly       // See lighting without base color
viewmode Lit                // Default lit view
viewmode Unlit              // See base color only
viewmode LightComplexity    // Visualize light overlap
```

### 2. Light Complexity

**Visualize overlapping lights**:
```
viewmode LightComplexity
```

Colors indicate number of overlapping lights:
- Green: 1-2 lights
- Yellow: 3-5 lights
- Red: 6+ lights (expensive!)

**Reduce complexity**:
- Reduce light radii
- Use fewer lights
- Improve light culling

### 3. Individual Light Debugging

**Disable all lights except one**:
```cpp
// In editor, select light
// Details panel → Toggle visibility
```

**Show light bounds**:
```
show Bounds  // Shows all actor bounds including lights
```

### 4. Profile Lighting Passes

**Console commands**:
```
stat SceneRendering  // Shows lighting pass timing
profilegpu           // Detailed GPU timing
```

**Lighting breakdown**:
- Direct lighting: X ms
- Indirect lighting: Y ms
- Light culling: Z ms

### 5. Check Light Parameters

**Common issues**:
- **Too high intensity**: Blown out/white pixels
- **Too large radius**: Performance hit, too many overlapping lights
- **Wrong attenuation**: Falloff looks unnatural

**Verify in editor**:
- Select light
- Check Details panel values
- Use visualization modes to see effect

### 6. Reflection Debugging

**Visualize reflections only**:
```
viewmode ReflectionOverride
```

**Check reflection capture placement**:
```
show ReflectionCaptures  // Shows capture volumes
```

## Advanced Topics

### Light Functions

Project custom patterns onto lights:
- Attach material to light (Light Function)
- Material samples texture/procedural pattern
- Modulates light intensity/color

**Example use cases**:
- Projected textures (logos, patterns)
- Animated lights (flickering)
- Light gobos (theater lighting)

### IES Profiles

Real-world light distribution data:
- Import IES files (industry standard)
- Accurate representation of real fixtures
- Used for archviz, realistic lighting

**Enable**:
```cpp
// Light component → IES Texture
PointLight->IESTexture = LoadObject<UTextureLightProfile>(...);
```

### Light Channels

Separate lights into channels:
- Light affects only objects on same channel
- Up to 3 channels (RGB)
- Useful for separating character lighting from environment

**Setup**:
```cpp
// Light component → Lighting Channels
Light->LightingChannels.bChannel0 = true;
Light->LightingChannels.bChannel1 = false;
Light->LightingChannels.bChannel2 = false;
```

### Light Mobility

**Static**:
- Baked lightmaps only
- No runtime cost
- No dynamic shadows

**Stationary**:
- Direct lighting is dynamic
- Indirect lighting is baked
- Dynamic shadows
- Best balance for most scenes

**Movable**:
- Fully dynamic
- Highest runtime cost
- Can move/animate

## Performance Considerations

### Light Count

**Limits**:
- **Deferred**: Handles many lights well (with culling)
- **Forward**: Limited to ~4-8 lights per object
- **Mobile**: Often 1-2 dynamic lights

**Optimization**:
- Use fewer, larger lights rather than many small ones
- Disable shadows on minor lights
- Use light channels to limit overlap

### Shadow Costs

Shadows are expensive:
- Directional: ~1-5ms (with CSM, see Chapter 6)
- Point: ~0.5-2ms per light
- Spot: ~0.3-1ms per light

**Reduce shadow cost**:
- Lower shadow resolution
- Reduce shadow distance
- Use static/baked shadows where possible

### Attenuation Radius

Smaller radius = better performance:
- Affects fewer pixels
- Better light culling
- Less overlap

**Tune radius**:
- As small as visually acceptable
- Use attenuation falloff exponent to control falloff curve

## Summary

Lighting pipeline:
1. **Culling** - Determine which lights affect which pixels/tiles
2. **Direct Lighting** - Evaluate each light's contribution (BRDF)
3. **Indirect Lighting** - Sky light, reflections, GI
4. **Composition** - Combine with base color

Key light types: Directional, Point, Spot, Rect

Key techniques: Tiled/clustered culling, PBR/BRDF, IBL

Key classes: `FLightSceneInfo`, `FDeferredShadingSceneRenderer`

## Next Chapter

[Chapter 6: Shadow Rendering](06-shadows.md) - Learn how UE5 renders realistic shadows for all light types.
