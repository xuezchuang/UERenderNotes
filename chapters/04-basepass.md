# Chapter 4: BasePass Rendering

## Goals

By the end of this chapter, you will understand:
- What the BasePass is and its role in deferred rendering
- How the GBuffer (Geometry Buffer) is structured
- Material evaluation during BasePass
- Vertex factories and mesh rendering
- How to debug BasePass rendering issues

## Overview

The BasePass is the first major rendering pass in UE5's deferred renderer. It renders all opaque geometry to the GBuffer, storing material properties (base color, normals, roughness, metallic, etc.) for later use by lighting passes.

**BasePass outputs** (GBuffer):
- **GBufferA**: World normal (RGB), Per-object data (A)
- **GBufferB**: Metallic (R), Specular (G), Roughness (B), Shading model (A)
- **GBufferC**: Base color (RGB), Indirect irradiance (A)
- **GBufferD**: Custom data (varies by shading model)
- **GBufferE**: Precomputed shadow factors
- **Depth/Stencil**: Depth and stencil buffer

## Pipeline Steps

### 1. Scene Preparation
**What happens**: Build list of meshes to render in BasePass.

**Culling**:
- Frustum culling (off-screen objects removed)
- Occlusion culling (occluded objects removed)
- Distance culling (based on LOD settings)

**Sorting**:
- By depth (front-to-back) for early-Z optimization
- By material (reduce state changes)
- By mesh type (static, skeletal, etc.)

### 2. Render Target Setup
**What happens**: Allocate and clear GBuffer render targets.

**GBuffer allocation**:
```cpp
// Typically:
// GBufferA: R8G8B8A8 or R10G10B10A2
// GBufferB: R8G8B8A8
// GBufferC: R8G8B8A8
// Depth: D24S8 or D32
```

### 3. Depth Pre-Pass (Optional)
**What happens**: Render depth-only to populate depth buffer.

**Benefits**:
- Early-Z rejection in BasePass (skip hidden pixels)
- Reduces pixel shader invocations
- Especially useful for complex materials

**Console variable**:
```
r.EarlyZPass=1  // 0=off, 1=on, 2=full depth prepass, 3=masked only
```

### 4. BasePass Draw Calls
**What happens**: Draw each mesh, evaluating materials and writing to GBuffer.

**Per-mesh**:
1. Bind vertex/index buffers
2. Set material shader and parameters
3. Draw mesh triangles
4. Pixel shader evaluates material
5. Write material attributes to GBuffer

### 5. Decals (Optional)
**What happens**: Apply decals that modify GBuffer.

**DBuffer decals**:
- Rendered before BasePass
- Blended into BasePass geometry

**GBuffer decals**:
- Rendered after BasePass
- Modify existing GBuffer data

## Key UE Source Files, Classes, and Functions

### BasePass Rendering

**`Engine/Source/Runtime/Renderer/Private/BasePassRendering.h`**
- `TBasePassPixelShaderBaseType` - BasePass pixel shader base class
- `TBasePassVertexShaderBaseType` - BasePass vertex shader base class
- BasePass shader permutations

**`Engine/Source/Runtime/Renderer/Private/BasePassRendering.cpp`**
- `FBasePassMeshProcessor` - Processes meshes for BasePass
- `AddMeshBatch()` - Adds a mesh batch to BasePass

**`Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp`**
- `FDeferredShadingSceneRenderer::RenderBasePass()` - Main BasePass entry point
- Orchestrates BasePass execution

### GBuffer

**`Engine/Source/Runtime/Renderer/Private/SceneRendering.h`**
- `FSceneRenderTargets` - Manages all scene render targets including GBuffer
- `GetSceneColor()`, `GetGBufferA()`, etc. - Accessors

**`Engine/Source/Runtime/Renderer/Private/SceneRenderTargets.cpp`**
- `FSceneRenderTargets::AllocateCommonDepthTargets()` - Allocate depth buffer
- `FSceneRenderTargets::AllocateGBufferTargets()` - Allocate GBuffer

### Mesh Processing

**`Engine/Source/Runtime/Renderer/Private/MeshPassProcessor.h`**
- `FMeshPassProcessor` - Base class for mesh pass processors
- `AddMeshBatch()` - Add mesh to pass
- `BuildMeshDrawCommands()` - Build draw commands

**`Engine/Source/Runtime/Renderer/Private/SceneVisibility.cpp`**
- `FSceneRenderer::ComputeViewVisibility()` - Compute visible primitives
- Frustum culling, occlusion culling

### Vertex Factories

**`Engine/Source/Runtime/Engine/Public/VertexFactory.h`**
- `FVertexFactory` - Base class for vertex factories
- Defines how vertex data is fetched and processed

**`Engine/Source/Runtime/Engine/Public/LocalVertexFactory.h`**
- `FLocalVertexFactory` - Standard static mesh vertex factory

**`Engine/Source/Runtime/Engine/Public/GPUSkinVertexFactory.h`**
- `FGPUSkinVertexFactory` - Skeletal mesh vertex factory (GPU skinning)

### Key Functions

```cpp
// DeferredShadingRenderer.cpp
bool FDeferredShadingSceneRenderer::RenderBasePass(
    FRDGBuilder& GraphBuilder,
    FSceneTextures& SceneTextures,
    const FDBufferTextures& DBufferTextures,
    FExclusiveDepthStencil::Type BasePassDepthStencilAccess,
    FRDGTextureRef ForwardShadowMaskTexture);

// BasePassRendering.cpp
void FBasePassMeshProcessor::AddMeshBatch(
    const FMeshBatch& RESTRICT MeshBatch,
    uint64 BatchElementMask,
    const FPrimitiveSceneProxy* RESTRICT PrimitiveSceneProxy,
    int32 StaticMeshId = -1);

// MeshPassProcessor.cpp
void FMeshPassProcessor::BuildMeshDrawCommands(
    const FMeshBatch& MeshBatch,
    uint64 BatchElementMask,
    const FPrimitiveSceneProxy* PrimitiveSceneProxy,
    const FMaterialRenderProxy& MaterialRenderProxy,
    const FMaterial& Material,
    const FMeshPassProcessorRenderState& DrawRenderState,
    FMeshDrawCommandPrimitiveIdInfo PrimitiveIdInfo,
    EMeshDrawCommandType DrawCommandType,
    FStaticMeshBatchInfo* StaticMeshBatchInfo);
```

## GBuffer Layout

### UE5 GBuffer Structure

**GBufferA** (WorldNormal):
```cpp
R8G8B8A8_UNORM or R10G10B10A2_UNORM
RGB: World-space normal (encoded)
A: Per-object data / AO
```

**GBufferB** (Material Properties):
```cpp
R8G8B8A8_UNORM
R: Metallic (0 = dielectric, 1 = metal)
G: Specular (typically 0.5 for non-metals)
B: Roughness (0 = smooth, 1 = rough)
A: Shading model ID + per-model data
```

**GBufferC** (BaseColor):
```cpp
R8G8B8A8_UNORM
RGB: Base color (albedo)
A: Ambient occlusion / indirect lighting
```

**GBufferD** (Custom Data):
```cpp
R8G8B8A8_UNORM
Contents vary by shading model:
- Subsurface: Subsurface color/opacity
- Clear coat: Clear coat amount/roughness
- Cloth: Fuzz color/amount
- etc.
```

**GBufferE** (Precomputed Shadow Factors):
```cpp
R16G16B16A16_UNORM
Stores precomputed shadow masks for static lighting
```

### Accessing GBuffer in Shaders

**In pixel shader**:
```hlsl
// Read from GBuffer
void MyDeferredPS(
    in float4 ScreenPosition : SV_Position,
    out float4 OutColor : SV_Target0)
{
    // Get screen UV
    float2 ScreenUV = ScreenPosition.xy * View.BufferSizeAndInvSize.zw;
    
    // Sample GBuffer
    float4 GBufferA = Texture2DSample(SceneTexturesStruct.GBufferATexture, 
                                      SceneTexturesStruct.GBufferATextureSampler, 
                                      ScreenUV);
    float4 GBufferB = Texture2DSample(SceneTexturesStruct.GBufferBTexture, 
                                      SceneTexturesStruct.GBufferBTextureSampler, 
                                      ScreenUV);
    float4 GBufferC = Texture2DSample(SceneTexturesStruct.GBufferCTexture, 
                                      SceneTexturesStruct.GBufferCTextureSampler, 
                                      ScreenUV);
    
    // Decode GBuffer
    FGBufferData GBuffer = DecodeGBufferData(GBufferA, GBufferB, GBufferC, ...);
    
    // Access material properties
    float3 BaseColor = GBuffer.BaseColor;
    float Roughness = GBuffer.Roughness;
    float Metallic = GBuffer.Metallic;
    float3 WorldNormal = GBuffer.WorldNormal;
}
```

## Material Evaluation

### Material Shader Execution

During BasePass, for each pixel:
```hlsl
// 1. Vertex shader transforms geometry
FVertexFactoryOutput VertexOutput = VertexFactory.GetVertexElements(...);

// 2. Interpolate vertex data to pixel
FMaterialPixelParameters MaterialParameters = GetMaterialPixelParameters(...);

// 3. Evaluate material graph (generated from Material Editor)
void CalcPixelMaterialInputs(
    FMaterialPixelParameters Parameters,
    inout FPixelMaterialInputs PixelMaterialInputs)
{
    // Material graph code here (generated HLSL)
    PixelMaterialInputs.BaseColor = <material graph result>;
    PixelMaterialInputs.Roughness = <material graph result>;
    PixelMaterialInputs.Metallic = <material graph result>;
    PixelMaterialInputs.Normal = <material graph result>;
    // ...
}

// 4. Encode material properties to GBuffer
EncodeGBuffer(PixelMaterialInputs, GBufferA, GBufferB, GBufferC, ...);
```

### Shading Models

UE5 supports multiple shading models:
- **Default Lit**: Standard PBR (Metallic/Roughness)
- **Subsurface**: For skin, wax, marble
- **Preintegrated Skin**: Optimized skin shading
- **Clear Coat**: For car paint, lacquer
- **Subsurface Profile**: Artist-controllable subsurface
- **Two-Sided Foliage**: For leaves, grass
- **Hair**: For realistic hair rendering
- **Cloth**: For fabric materials
- **Eye**: For realistic eyes
- **Thin Translucent**: For thin translucent surfaces

Each model may use GBuffer differently.

## Vertex Factories

### What Are Vertex Factories?

Vertex factories abstract how vertex data is fetched:
- **Static meshes**: Simple position/normal/UV
- **Skeletal meshes**: Bones, weights, skinning
- **Procedural meshes**: Generated data
- **Instances**: Per-instance transforms
- **Landscape**: Heightmap-based

### Vertex Factory Interface

Key methods:
```cpp
class FVertexFactory
{
    // Get vertex shader code
    virtual void ModifyCompilationEnvironment(
        const FVertexFactoryShaderPermutationParameters& Parameters,
        FShaderCompilerEnvironment& OutEnvironment);
    
    // Declare vertex inputs
    virtual void GetStreams(
        ERHIFeatureLevel::Type FeatureLevel,
        EVertexInputStreamType VertexStreamType,
        FVertexDeclarationElementList& Elements);
};
```

### Static Mesh Vertex Factory

**Location**: `Engine/Source/Runtime/Engine/Public/LocalVertexFactory.h`

**Vertex streams**:
- Position (float3)
- Normal (packed)
- Tangent (packed)
- UV coordinates (float2 × N channels)
- Color (optional, R8G8B8A8)

### Skeletal Mesh Vertex Factory

**Location**: `Engine/Source/Runtime/Engine/Public/GPUSkinVertexFactory.h`

**Additional data**:
- Bone indices (up to 4-8 per vertex)
- Bone weights (normalized)
- Bone matrices (from animation)

**Skinning in vertex shader**:
```hlsl
float3 SkinnedPosition = 0;
float3 SkinnedNormal = 0;

for (int i = 0; i < 4; i++)
{
    int BoneIndex = Input.BlendIndices[i];
    float BoneWeight = Input.BlendWeights[i];
    
    float4x4 BoneMatrix = BoneMatrices[BoneIndex];
    SkinnedPosition += mul(float4(Input.Position, 1), BoneMatrix).xyz * BoneWeight;
    SkinnedNormal += mul(Input.Normal, (float3x3)BoneMatrix) * BoneWeight;
}
```

## Debugging Tips

### 1. Visualize GBuffer Channels

**Console commands**:
```
viewmode LightingOnly        // See lighting without base color
viewmode Unlit               // See base color only
viewmode DetailLighting      // Detailed view of lighting
viewmode SpecularColor       // Visualize specular
viewmode Metallic            // Visualize metallic
viewmode Roughness           // Visualize roughness
viewmode SubsurfaceColor     // Visualize subsurface
```

**Custom visualization**:
```cpp
// In your own post-process shader
float Roughness = GBuffer.Roughness;
OutColor = float4(Roughness.xxx, 1);  // Visualize as grayscale
```

### 2. Check GBuffer Contents

**Enable buffer visualization**:
- Editor → Show → Visualize → GBuffer Hints (A, B, C, D, E)

**RenderDoc**:
- Capture frame
- View GBuffer render targets
- Inspect per-pixel values

### 3. Profile BasePass

**Console commands**:
```
stat SceneRendering   // Shows BasePass timing
stat RHI              // Shows draw calls
profilegpu            // Detailed GPU breakdown
```

**Typical timing**:
- BasePass: 2-8ms (depends on scene complexity)
- Early-Z prepass: 0.5-2ms

### 4. Debug Material Complexity

**Shader complexity view**:
```
viewmode ShaderComplexity
```

Colors indicate cost:
- Green: Cheap
- Yellow: Moderate
- Red: Expensive
- White/Pink: Very expensive

**Identify expensive materials**:
- Check for excessive texture samples
- Look for complex math operations
- Verify normal maps are used efficiently

### 5. Occlusion Debugging

**Disable occlusion culling**:
```
r.HZBOcclusion=0  // Disable hierarchical Z-buffer occlusion
```

**Visualize occlusion**:
```
r.VisualizeOccludedPrimitives=1
```

### 6. Common BasePass Issues

**Black surfaces**:
- Check material base color is not zero
- Verify normal map is correct (should be bluish)
- Check GBuffer is being written correctly

**Incorrect lighting**:
- Verify GBuffer normals are in world space
- Check metallic/roughness values are reasonable (0-1 range)
- Ensure shading model is correct

**Performance issues**:
- Too many draw calls (use instancing)
- Complex materials (reduce shader instructions)
- Overdraw (optimize mesh density, use LODs)

## Advanced Topics

### BasePass Variants

**Atmospheric fog**:
- Additional shader permutation for atmospheric fog in BasePass
- Increases shader count

**Velocity**:
- Some BasePass shaders output velocity for motion blur/TAA
- Requires previous frame transform data

**Editor selections**:
- Editor-specific BasePass variant for selection highlighting

### Mobile BasePass

Mobile uses a simplified BasePass:
- Fewer GBuffer targets (often just 1-2)
- Forward rendering more common than deferred
- Shader complexity reduced

### Forward Shading BasePass

**When forward shading is used**:
- Transparent materials
- Materials with too many attributes for GBuffer
- VR (lower latency)

**Forward BasePass**:
- Evaluates lighting during BasePass
- No GBuffer output
- Directly outputs lit color

Enable forward shading:
```cpp
// Project Settings → Rendering → Forward Shading
r.ForwardShading=1
```

### Nanite and BasePass

Nanite meshes have their own specialized visibility buffer pass, which replaces traditional BasePass for Nanite geometry (see Chapter 9).

## Performance Considerations

### Draw Call Optimization

**Instancing**:
- Use instanced static meshes (ISM)
- Use hierarchical instanced static meshes (HISM)
- Reduces draw calls dramatically

**Mesh merging**:
- Merge static meshes in-editor
- Actor merging tool (Window → Developer Tools → Merge Actors)

### Overdraw Reduction

**Check overdraw**:
```
viewmode QuadOverdraw
```

**Reduce overdraw**:
- Use LODs appropriately
- Occlusion culling
- Backface culling
- Early-Z rejection

### Material Optimization

**Reduce instruction count**:
- Simplify material graphs
- Use material functions for reusability
- Avoid redundant calculations

**Texture sampling**:
- Combine textures into channels (pack roughness/metallic/AO)
- Use compressed formats (BC1, BC3, BC5)
- Appropriate texture sizes (don't oversample)

## Summary

BasePass rendering:
1. **Preparation** - Cull, sort, prepare meshes
2. **Depth Prepass** - Optionally render depth first
3. **GBuffer Generation** - Evaluate materials, write to GBuffer
4. **Decals** - Apply decals to GBuffer

Key outputs: GBuffer (normals, base color, roughness, metallic, etc.)

Key classes: `FBasePassMeshProcessor`, `FSceneRenderTargets`, `FVertexFactory`

The BasePass is the foundation of deferred rendering, capturing all material properties needed for lighting.

## Next Chapter

[Chapter 5: Lighting System](05-lighting.md) - Learn how UE5 computes lighting using the GBuffer data.
