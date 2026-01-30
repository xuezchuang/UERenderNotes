# Chapter 9: Nanite Virtualized Geometry

## Goals

By the end of this chapter, you will understand:
- What Nanite is and how virtualized geometry works
- Nanite's hierarchical level of detail (LOD) system
- Visibility buffer rendering technique
- Cluster culling and streaming architecture
- Performance characteristics and best practices

## Overview

Nanite is UE5's virtualized geometry system that renders massive amounts of geometric detail efficiently. It eliminates traditional LOD management, supports billions of triangles, and maintains performance through aggressive culling and streaming.

**Key features**:
- **No manual LODs**: Automatic level of detail
- **Massive detail**: Billions of triangles per scene
- **Pixel-sized triangles**: Geometry detail matching screen resolution
- **Streaming**: Only loads visible clusters
- **Consistent performance**: Scales from simple to complex scenes

**How it works**:
- Meshes divided into small clusters (~128 triangles)
- Hierarchical LOD structure (tree of clusters)
- GPU-driven visibility and culling
- Visibility buffer replaces traditional rasterization

## Pipeline Steps

### 1. Mesh Processing (Offline)
**What happens**: Import mesh is processed into Nanite format.

**Process**:
1. Simplify mesh into LOD hierarchy
2. Divide each LOD into clusters (~128 triangles)
3. Build BVH (Bounding Volume Hierarchy) for culling
4. Compress geometry data
5. Store in Nanite-specific format

### 2. Streaming
**What happens**: Load only necessary clusters from disk/memory.

**Strategy**:
- High-detail clusters near camera
- Low-detail clusters far from camera
- Stream in/out based on visibility
- Asynchronous loading (no hitches)

### 3. Culling (Two-Pass)
**What happens**: Determine which clusters are visible.

**First Pass (Coarse)**:
- Frustum culling (off-screen clusters removed)
- Occlusion culling (HZB - Hierarchical Z-Buffer)
- LOD selection (choose appropriate detail level)

**Second Pass (Fine)**:
- Per-cluster visibility
- Small primitive culling
- Backface culling

### 4. Visibility Buffer Rasterization
**What happens**: Instead of GBuffer, render visibility information.

**Visibility buffer contains**:
- Triangle ID (which triangle)
- Cluster ID (which cluster)
- Depth (Z value)

**No vertex/pixel shaders** during rasterization - just IDs.

### 5. Material Evaluation (Deferred)
**What happens**: Shade pixels using visibility buffer.

**Process**:
1. For each pixel, read triangle/cluster ID from visibility buffer
2. Fetch vertex data for triangle
3. Compute barycentric coordinates
4. Interpolate vertex attributes
5. Evaluate material (same as BasePass, Chapter 4)
6. Write to GBuffer

### 6. Integration with Standard Renderer
**What happens**: Merge Nanite output with non-Nanite geometry.

**Composition**:
- Nanite GBuffer + Standard GBuffer → Combined GBuffer
- Depth buffer merging
- Proceed with lighting (Chapter 5)

## Key UE Source Files, Classes, and Functions

### Nanite System Core

**`Engine/Source/Runtime/Renderer/Private/Nanite/Nanite.h`**
- `Nanite` namespace - Core Nanite functionality
- Nanite constants and configuration

**`Engine/Source/Runtime/Renderer/Private/Nanite/NaniteRender.h`**
- `FNaniteVisibilityResults` - Visibility query results
- Nanite rendering interface

**`Engine/Source/Runtime/Renderer/Private/Nanite/NaniteRender.cpp`**
- `CullRasterize()` - Main Nanite rendering entry point
- Culling and rasterization orchestration

### Nanite Resources

**`Engine/Source/Runtime/Engine/Public/Rendering/NaniteResources.h`**
- `FNaniteResources` - Nanite mesh data
- Cluster data structures

**`Engine/Source/Runtime/Engine/Private/Rendering/NaniteResources.cpp`**
- Resource management and streaming
- Cluster loading

### Culling

**`Engine/Source/Runtime/Renderer/Private/Nanite/NaniteCulling.h`**
- Culling pass declarations

**`Engine/Shaders/Private/Nanite/NaniteCulling.usf`**
- GPU culling shaders (HLSL)
- Hierarchical culling implementation

### Rasterization

**`Engine/Source/Runtime/Renderer/Private/Nanite/NaniteRasterizer.cpp`**
- Software and hardware rasterization

**`Engine/Shaders/Private/Nanite/NaniteRasterize.usf`**
- Rasterization shaders (HLSL)
- Visibility buffer generation

### Material Evaluation

**`Engine/Source/Runtime/Renderer/Private/Nanite/NaniteVisualize.cpp`**
- Nanite visualization modes

**`Engine/Shaders/Private/Nanite/NaniteExportGBuffer.usf`**
- Material shading from visibility buffer
- GBuffer export

### Key Functions

```cpp
// NaniteRender.cpp
void CullRasterize(
    FRDGBuilder& GraphBuilder,
    const FScene& Scene,
    const TArray<FViewInfo>& Views,
    const FCullingContext& CullingContext,
    const FRasterContext& RasterContext,
    const FRasterState& RasterState,
    const TArray<FInstanceDraw>* OptionalInstanceDraws = nullptr);

// NaniteResources.cpp
void FNaniteResources::InitResources(FRHICommandListImmediate& RHICmdList);

// NaniteCulling.usf (HLSL)
[numthreads(64, 1, 1)]
void MainCS(uint GroupID : SV_GroupID, uint GroupThreadID : SV_GroupThreadID)
{
    // Culling logic
}

// NaniteExportGBuffer.usf (HLSL)
void ExportGBuffer(
    uint2 PixelPos,
    uint VisibleClusterIndex,
    uint TriIndex)
{
    // Material evaluation and GBuffer export
}
```

## Nanite Mesh Format

### Cluster Structure

Meshes are divided into small clusters:
```cpp
struct FCluster
{
    // Geometric data
    uint32 NumTriangles;      // Typically ~128
    uint32 NumVertices;       // Typically ~64-96
    
    // Bounding information
    FBounds Bounds;           // AABB for culling
    FSphere LODError;         // LOD selection
    
    // Vertex data (quantized)
    TArray<FVertex> Vertices;
    
    // Index data (compressed)
    TArray<uint32> Indices;
    
    // Material
    uint32 MaterialIndex;
};
```

### Hierarchical LOD

Clusters organized in tree structure:
```
LOD 0 (Highest detail, most clusters)
  |
LOD 1 (Medium detail, fewer clusters)
  |
LOD 2 (Low detail, fewest clusters)
  |
LOD 3 (Lowest detail, single cluster)
```

**LOD selection**:
- Based on screen-space error metric
- Smooth transitions (no popping)
- Per-cluster decision (not per-mesh)

### Vertex Quantization

Vertices compressed aggressively:
- Positions: Quantized to cluster bounds (16-bit)
- Normals: Octahedral encoding (16-bit)
- UVs: Quantized (16-bit)
- Colors: 8-bit per channel

**Benefits**:
- Reduced memory footprint
- Faster streaming
- Better cache utilization

## Culling System

### Two-Pass Culling

**Pass 1 - Coarse Culling** (Compute shader):
```hlsl
// Frustum culling
if (!FrustumIntersects(Cluster.Bounds, ViewFrustum))
    return; // Cull

// HZB occlusion culling
float2 ScreenMin, ScreenMax;
ProjectBoundsToScreen(Cluster.Bounds, ScreenMin, ScreenMax);
float OccluderDepth = HZBTexture.SampleLevel(Sampler, ScreenMin, MipLevel).r;
if (Cluster.Bounds.MaxZ < OccluderDepth)
    return; // Occluded, cull

// LOD selection
float ScreenSize = ComputeScreenSize(Cluster);
if (ScreenSize < Cluster.LODError)
    UseParentCluster(); // Too small, use coarser LOD
else
    VisibleClusters.Append(Cluster);
```

**Pass 2 - Fine Culling** (Per-cluster):
```hlsl
// Backface culling at cluster level
if (dot(Cluster.AverageNormal, ViewDirection) > 0)
    return; // Backfacing cluster

// Small primitive culling
if (Cluster.ScreenSize < MinPixelSize)
    return; // Too small to render
```

### Hierarchical Z-Buffer (HZB)

Occlusion culling uses HZB:
- Mipmap chain of depth buffer
- Each level stores max depth of 2×2 region
- Fast conservative occlusion tests

**Build HZB**:
```hlsl
// Generate HZB mip chain
for (int Mip = 1; Mip < MaxMips; ++Mip)
{
    float4 Depths = GatherDepth(PrevMip, UV);
    float MaxDepth = max(max(Depths.x, Depths.y), max(Depths.z, Depths.w));
    OutputDepth[Mip] = MaxDepth;
}
```

### GPU-Driven Pipeline

Entire culling pipeline runs on GPU:
- No CPU readback (async)
- Indirect draw calls (GPU generates draw commands)
- Fully parallel (thousands of clusters tested simultaneously)

## Visibility Buffer

### What Is Visibility Buffer?

Instead of traditional rasterization (vertex → pixel shader), Nanite writes IDs:

**Traditional rasterization**:
```
Vertex Shader → Interpolation → Pixel Shader → GBuffer
```

**Nanite visibility buffer**:
```
Rasterization → Visibility Buffer (IDs + Depth)
Deferred → Fetch Vertex Data → Evaluate Material → GBuffer
```

### Visibility Buffer Format

**Per-pixel data** (64-bit):
```cpp
struct FVisibilityPixel
{
    uint32 TriangleID : 32;   // Which triangle (within cluster)
    uint32 ClusterID : 32;    // Which cluster
};

// Depth stored in separate buffer
float Depth;
```

### Material Evaluation from Visibility Buffer

**Deferred shading**:
```hlsl
void MaterialShader(uint2 PixelPos)
{
    // Read visibility buffer
    FVisibilityPixel Vis = VisibilityBuffer[PixelPos];
    float Depth = DepthBuffer[PixelPos];
    
    // Fetch cluster and triangle data
    FCluster Cluster = Clusters[Vis.ClusterID];
    FTriangle Triangle = GetTriangle(Cluster, Vis.TriangleID);
    
    // Compute barycentric coordinates
    float3 WorldPos = ReconstructWorldPosition(PixelPos, Depth);
    float3 Bary = ComputeBarycentric(WorldPos, Triangle);
    
    // Interpolate vertex attributes
    float3 Normal = InterpolateNormal(Triangle, Bary);
    float2 UV = InterpolateUV(Triangle, Bary);
    // ... other attributes
    
    // Evaluate material (same as standard BasePass)
    FPixelMaterialInputs Inputs = EvaluateMaterial(UV, Normal, ...);
    
    // Write to GBuffer
    EncodeGBuffer(Inputs, GBufferA, GBufferB, GBufferC, ...);
}
```

**Benefits**:
- Decouples geometry from shading
- Fixed rasterization cost (no pixel overdraw for materials)
- Variable rate shading possible

## Streaming

### Cluster Streaming

Only load visible clusters:
- **Resident**: Clusters in GPU memory
- **Streaming**: Loading from disk/system memory
- **Eviction**: Remove unused clusters

**Request system**:
```cpp
// Each frame
1. Determine required clusters (from culling)
2. Check which are resident
3. Request non-resident clusters
4. Wait for streaming (async)
5. Render with available clusters
```

### Progressive Detail Loading

As camera moves closer:
- Higher detail clusters streamed in
- Lower detail clusters evicted
- Smooth transitions (no popping)

### Fallback Rendering

While waiting for streaming:
- Use lower LOD (parent cluster)
- No holes or missing geometry
- Progressive enhancement

**Console variables**:
```
r.Nanite.Streaming.NumInitialRootPages=64     // Initial streaming budget
r.Nanite.Streaming.MaxPendingPages=32         // Max concurrent requests
r.Nanite.Streaming.MaxPageInstallsPerFrame=4  // Throughput limit
```

## Debugging Tips

### 1. Visualize Nanite

**Console commands**:
```
r.Nanite.Visualize=1              // Enable visualization
r.Nanite.Visualize.Mode=Overview  // Various modes
```

**Visualization modes**:
- **Overview**: General Nanite info
- **Clusters**: Show individual clusters (color-coded)
- **Triangles**: Show triangle density
- **MaterialMode**: Show material assignments
- **LightmapDensity**: Lightmap usage

### 2. Check Nanite Statistics

**Console commands**:
```
stat Nanite           // Nanite-specific stats
stat SceneRendering   // General stats including Nanite
r.Nanite.ShowStats=1  // On-screen Nanite stats
```

**Stats include**:
- Cluster count (total, visible, rendered)
- Triangle count
- Memory usage
- Streaming status

### 3. Profile Nanite

**Console commands**:
```
profilegpu           // GPU profiler with Nanite breakdown
```

**Typical timings**:
- Culling: 0.5-2ms
- Rasterization: 1-4ms
- Material evaluation: 2-6ms
- **Total**: 4-12ms (scene dependent)

### 4. Nanite vs. Non-Nanite Comparison

**Toggle Nanite**:
```
r.Nanite=0   // Disable Nanite (use standard rendering)
r.Nanite=1   // Enable Nanite
```

**Compare**:
- Performance (frame time)
- Visual quality (LOD popping, detail level)
- Memory usage

### 5. Common Issues

**Nanite mesh not rendering**:
- Check "Enable Nanite Support" in Static Mesh settings
- Verify mesh was imported correctly
- Check if mesh is within streaming budget

**Flickering/Instability**:
- Increase `r.Nanite.MaxPixelsPerEdge=1` (reduce rasterization precision)
- Check for overlapping geometry (Z-fighting)

**Performance issues**:
- Too many instances (use Instanced Static Meshes)
- Reduce streaming budget if memory-constrained
- Check material complexity (materials still evaluated per-pixel)

## Advanced Topics

### Nanite and Lumen Integration

Nanite works seamlessly with Lumen (Chapter 8):
- Nanite meshes participate in Lumen Surface Cache
- High-detail geometry improves GI quality
- Combined, they enable unprecedented visual fidelity

### Programmable Rasterization

Nanite uses software rasterization (on GPU):
- More flexible than fixed-function hardware
- Enables custom culling and LOD logic
- Optimized for Nanite's workload

**Fallback to hardware rasterization**:
- For transparent geometry
- For certain platforms (mobile)

### Material Complexity

Nanite decouples geometry from material cost:
- **Geometry complexity**: Constant cost (visibility buffer)
- **Material complexity**: Variable cost (shader evaluation)

**Implication**: Complex materials still expensive - optimize shader complexity.

### Instancing

Nanite supports instancing:
- Thousands of instances of same mesh
- Each instance culled separately
- Efficient memory usage (shared cluster data)

**Use Instanced Static Mesh Component**:
```cpp
UInstancedStaticMeshComponent* ISM = NewObject<UInstancedStaticMeshComponent>();
ISM->SetStaticMesh(MyNaniteMesh);
ISM->AddInstance(Transform1);
ISM->AddInstance(Transform2);
// ... thousands more
```

### Limitations

**Nanite does not support**:
- Vertex animation (skeletal meshes, morph targets)
- World Position Offset (WPO) in materials
- Tessellation
- Certain material features (pixel depth offset)

**Workarounds**:
- Use standard rendering for animated meshes
- Hybrid approach (Nanite + standard in same scene)

## Performance Considerations

### When to Use Nanite

**Ideal for**:
- Static geometry (architecture, rocks, props)
- High-poly meshes (millions of triangles)
- Scenes with many instances
- Far-view rendering (landscapes, cityscapes)

**Not ideal for**:
- Animated characters (use skeletal meshes)
- Transparent objects (Nanite is opaque only)
- Simple geometry (traditional rendering may be faster)

### Memory Usage

**Nanite memory breakdown**:
- **Cluster data**: Largest component (compressed geometry)
- **Streaming pool**: Resident clusters
- **Visibility buffer**: Screen resolution dependent (minimal)

**Typical usage**:
- 100MB-500MB for streaming pool
- Scales with scene complexity

**Console variables**:
```
r.Nanite.MaxCandidateClusters=16777216  // Max clusters in scene
r.Nanite.MaxNodes=2097152               // Max hierarchy nodes
```

### Scalability

Nanite scales well:
- **High-end**: Full detail, large streaming budget
- **Mid-range**: Reduced detail, moderate budget
- **Low-end**: Aggressive LOD, small budget

**Project Settings → Nanite**:
- Quality presets (Low, Medium, High, Epic)
- Per-platform settings

### Triangle Budget

Unlike traditional rendering, Nanite doesn't have a strict triangle limit:
- Performance depends on visibility, not total triangles
- Millions of visible triangles rendered efficiently
- Occlusion culling critical for performance

**Rule of thumb**: Keep per-pixel triangle density reasonable (1-2 triangles per pixel ideal).

## Best Practices

### 1. Enable Nanite for High-Poly Meshes
```cpp
// Static Mesh Editor
- Enable Nanite Support: Checked
- Preserve Area: Unchecked (unless needed)
```

### 2. Use Instancing
- Use Instanced Static Mesh components for repeated objects
- Hierarchical Instanced Static Mesh (HISM) for large counts

### 3. Optimize Materials
- Nanite benefits from simple materials
- Complex materials still pay full pixel shader cost
- Share materials across meshes when possible

### 4. Combine with Lumen
- Nanite + Lumen = Best results
- High geometry detail improves GI quality
- Both systems designed to work together

### 5. Profile and Iterate
- Use `stat Nanite` and `profilegpu`
- Identify bottlenecks (culling, rasterization, materials)
- Adjust settings based on target platform

## Summary

Nanite provides:
1. **Virtualized geometry** - Stream only visible clusters
2. **Automatic LOD** - No manual LOD creation
3. **Massive scale** - Billions of triangles
4. **GPU-driven culling** - Efficient visibility determination
5. **Visibility buffer** - Decouple geometry from shading

Key classes: `FNaniteResources`, `FNaniteVisibilityResults`

Nanite is revolutionary for static geometry, enabling film-quality assets in real-time. Combined with Lumen, it represents a major leap forward in real-time rendering.

## Next Chapter

[Chapter 10: Temporal Super Resolution (TSR)](10-tsr.md) - Learn UE5's advanced temporal upsampling and anti-aliasing technique.
