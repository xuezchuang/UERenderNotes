# Chapter 10: Temporal Super Resolution (TSR)

## Goals

By the end of this chapter, you will understand:
- What TSR is and how temporal upsampling works
- History reprojection and rejection
- Anti-aliasing integration
- Performance benefits and quality considerations
- Debugging temporal artifacts

## Overview

Temporal Super Resolution (TSR) is UE5's advanced upsampling technique that renders at lower resolution and reconstructs higher resolution output using temporal data. It provides excellent anti-aliasing while improving performance.

**Key features**:
- **High-quality upsampling**: Better than spatial upsampling (bilinear, bicubic)
- **Integrated anti-aliasing**: Replaces TAA with better quality
- **Performance gain**: 30-60% faster rendering (depends on scale)
- **Temporal stability**: Reduces flickering, crawling, shimmering

**How it works**:
- Render at lower resolution (e.g., 50-67% of target)
- Use motion vectors to reproject previous frame
- Blend current and history with intelligent rejection
- Output at target resolution with anti-aliasing

## Pipeline Steps

### 1. Low-Resolution Rendering
**What happens**: Render scene at lower resolution (internal resolution).

**Example**:
- Target: 1920×1080 (Full HD)
- Internal: 1280×720 (67% scale)
- ~2.25× fewer pixels to shade

### 2. Motion Vector Generation
**What happens**: Compute per-pixel velocity (motion from previous frame).

**Velocity sources**:
- Camera movement (view matrix delta)
- Object movement (per-object transforms)
- Vertex animation (skeletal meshes, WPO)

**Motion vector buffer**:
```hlsl
float2 Velocity = (CurrentScreenPos - PreviousScreenPos);
```

### 3. History Reprojection
**What happens**: Sample previous frame using motion vectors.

**Process**:
1. For each pixel in current frame
2. Read motion vector
3. Compute previous frame position: `PrevUV = CurrUV - Velocity`
4. Sample history buffer at `PrevUV`

### 4. History Validation
**What happens**: Detect invalid/disoccluded pixels in history.

**Rejection tests**:
- **Depth discontinuity**: Object moved, reveals background
- **Out of bounds**: Pixel wasn't visible last frame
- **Color clamping**: History color doesn't match neighborhood
- **Velocity validation**: Inconsistent motion

### 5. Temporal Accumulation
**What happens**: Blend current and validated history.

**Blending**:
```hlsl
float3 Result = lerp(CurrentColor, HistoryColor, TemporalFeedback);
```

- Valid history: High feedback (0.9-0.95) - strong temporal filtering
- Invalid history: Low feedback (0.0-0.3) - rely on current frame

### 6. Spatial Upsampling
**What happens**: Upsample to target resolution.

**Techniques**:
- Bilinear filtering (fast)
- Catmull-Rom filtering (higher quality)
- Lanczos filtering (highest quality, expensive)

### 7. Anti-Aliasing
**What happens**: TSR provides AA as part of upsampling.

**AA quality**:
- Better than FXAA (post-process AA)
- Better than MSAA (multi-sample AA) - cheaper
- Comparable to TAA (temporal AA) - with upsampling bonus

## Key UE Source Files, Classes, and Functions

### TSR System Core

**`Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalSuperResolution.h`**
- `FTSR` - Main TSR interface
- TSR configuration structures

**`Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalSuperResolution.cpp`**
- `AddTemporalSuperResolutionPasses()` - Add TSR to render graph
- Main TSR orchestration

### History Management

**`Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalAA.h`**
- History buffer management (shared with TAA)
- `FTemporalAAHistory` - Stores previous frame data

### Shaders

**`Engine/Shaders/Private/TemporalSuperResolution/TSRCommon.ush`**
- Shared TSR utilities (HLSL)
- History sampling, velocity calculations

**`Engine/Shaders/Private/TemporalSuperResolution/TSRResolve.usf`**
- Main TSR resolve pass (HLSL)
- History reprojection and blending

**`Engine/Shaders/Private/TemporalSuperResolution/TSRReject.usf`**
- History validation and rejection (HLSL)

**`Engine/Shaders/Private/TemporalSuperResolution/TSRUpdate.usf`**
- History update and anti-aliasing (HLSL)

### Key Functions

```cpp
// TemporalSuperResolution.cpp
void AddTemporalSuperResolutionPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FTSRPassInputs& Inputs,
    FTSROutputs* OutOutputs);

// TSRResolve.usf (HLSL)
void MainCS(
    uint2 GroupId : SV_GroupID,
    uint2 GroupThreadId : SV_GroupThreadID)
{
    // TSR resolve logic
    // - Reproject history
    // - Validate history
    // - Blend with current
    // - Upsample to target resolution
}
```

## History Reprojection

### Motion Vectors

Motion vectors encode pixel movement:

**Generation**:
```hlsl
// Vertex shader
float4 CurrClipPos = mul(float4(WorldPos, 1), CurrViewProj);
float4 PrevClipPos = mul(float4(PrevWorldPos, 1), PrevViewProj);

float2 CurrScreen = CurrClipPos.xy / CurrClipPos.w;
float2 PrevScreen = PrevClipPos.xy / PrevClipPos.w;

float2 Velocity = (CurrScreen - PrevScreen) * float2(0.5, -0.5);
```

**Storage**:
- R16G16_FLOAT or R16G16_SNORM
- Stored in velocity buffer (generated during BasePass/Velocity pass)

### Reprojection

**Sample history using velocity**:
```hlsl
float2 CurrentUV = PixelPos * View.BufferSizeAndInvSize.zw;
float2 Velocity = VelocityTexture.Sample(Sampler, CurrentUV).xy;
float2 HistoryUV = CurrentUV - Velocity;

// Sample history
float3 HistoryColor = HistoryTexture.Sample(Sampler, HistoryUV).rgb;
```

**Challenges**:
- Velocity may be incorrect (object boundaries, thin objects)
- History may be invalid (disocclusion, new pixels)
- Bilinear sampling insufficient for quality

### Bilateral Filtering

Higher quality history sampling:

```hlsl
float3 SampleHistoryBilateral(float2 HistoryUV, float CurrDepth)
{
    float3 Result = 0;
    float TotalWeight = 0;
    
    // Sample 2×2 or 3×3 neighborhood
    for (int x = -1; x <= 1; ++x)
    {
        for (int y = -1; y <= 1; ++y)
        {
            float2 Offset = float2(x, y) * HistoryTexelSize;
            float2 SampleUV = HistoryUV + Offset;
            
            // Sample history color and depth
            float3 Color = HistoryTexture.Sample(Sampler, SampleUV).rgb;
            float Depth = HistoryDepth.Sample(Sampler, SampleUV).r;
            
            // Depth-aware weight (bilateral)
            float DepthWeight = exp(-abs(CurrDepth - Depth) * DepthSigma);
            float SpatialWeight = GaussianWeight(Offset);
            float Weight = DepthWeight * SpatialWeight;
            
            Result += Color * Weight;
            TotalWeight += Weight;
        }
    }
    
    return Result / max(TotalWeight, 0.001);
}
```

## History Rejection

### Why Rejection Is Needed

History can be invalid:
- **Disocclusion**: Moving object reveals background
- **Lighting changes**: Dynamic lights turn on/off
- **New geometry**: Objects enter frame
- **Incorrect velocity**: Velocity estimation failure

Invalid history causes ghosting and artifacts.

### Rejection Techniques

**1. Neighborhood Clamping**:
```hlsl
// Compute min/max of current frame's neighborhood
float3 NeighborMin = 99999;
float3 NeighborMax = -99999;

for (int i = 0; i < 9; ++i)
{
    float3 Neighbor = CurrentNeighbor[i];
    NeighborMin = min(NeighborMin, Neighbor);
    NeighborMax = max(NeighborMax, Neighbor);
}

// Clamp history to neighborhood bounds
HistoryColor = clamp(HistoryColor, NeighborMin, NeighborMax);
```

**2. Variance Clipping**:
```hlsl
// Compute mean and variance of neighborhood
float3 Mean = ComputeMean(Neighborhood);
float3 Variance = ComputeVariance(Neighborhood, Mean);
float3 StdDev = sqrt(Variance);

// Clamp history within N standard deviations
float3 ClampMin = Mean - StdDev * ClampThreshold;
float3 ClampMax = Mean + StdDev * ClampThreshold;
HistoryColor = clamp(HistoryColor, ClampMin, ClampMax);
```

**3. Depth Rejection**:
```hlsl
// Compare current and reprojected depth
float CurrDepth = SceneDepth.Sample(Sampler, CurrentUV).r;
float HistoryDepth = PrevSceneDepth.Sample(Sampler, HistoryUV).r;

float DepthDiff = abs(CurrDepth - HistoryDepth);
if (DepthDiff > DepthThreshold)
{
    // Reject history (disocclusion detected)
    TemporalFeedback = 0.0; // Don't use history
}
```

**4. Out-of-Bounds Rejection**:
```hlsl
// Check if history UV is valid
if (any(HistoryUV < 0) || any(HistoryUV > 1))
{
    // History out of screen bounds
    TemporalFeedback = 0.0;
}
```

### Adaptive Blending

Adjust blend weight based on history confidence:

```hlsl
float TemporalFeedback = 0.9; // Default: high confidence

// Reduce confidence for rejected history
if (DepthRejection)
    TemporalFeedback *= 0.1;

if (ColorRejection)
    TemporalFeedback *= 0.3;

if (OutOfBounds)
    TemporalFeedback = 0.0;

// Blend
float3 Result = lerp(CurrentColor, HistoryColor, TemporalFeedback);
```

## Anti-Aliasing

### Temporal Anti-Aliasing (TAA)

TSR includes TAA functionality:

**Sub-pixel jitter**:
```cpp
// Jitter camera projection to sample sub-pixels
float2 Jitter = HaltonSequence(FrameIndex % 8) * 2 - 1; // [-1, 1]
Jitter *= PixelSize * 0.5; // Half-pixel offset

// Apply to projection matrix
ProjectionMatrix = AddJitter(ProjectionMatrix, Jitter);
```

**Accumulation**:
- Each frame samples different sub-pixel position
- Temporal accumulation averages samples
- Result: super-sampled anti-aliasing

### Compared to Other AA Techniques

**FXAA (Fast Approximate AA)**:
- ✗ Post-process blur (loses detail)
- ✓ Fast (1 pass)

**MSAA (Multi-Sample AA)**:
- ✓ High quality (geometric edges)
- ✗ Expensive (multiple samples per pixel)
- ✗ Doesn't handle shader aliasing

**TAA (Temporal AA)**:
- ✓ High quality (all aliasing types)
- ✗ No upsampling (same resolution)
- ✓ Good temporal stability

**TSR**:
- ✓ High quality AA
- ✓ Upsampling (performance gain)
- ✓ Excellent temporal stability
- ✗ Requires motion vectors

## Performance Benefits

### Rendering Resolution vs. Output Resolution

**Example scenario**:
- **Target**: 3840×2160 (4K)
- **Render**: 2560×1440 (67% scale)
- **Savings**: 2.25× fewer pixels

**Frame time improvement**:
- BasePass: ~40% faster
- Lighting: ~40% faster
- Post-processing: ~30% faster
- **Total**: ~30-50% faster (depends on bottleneck)

### Quality vs. Performance

**TSR quality settings**:
- **High (67% scale)**: Best quality, moderate perf gain
- **Medium (50-58%)**: Balanced
- **Low (50%)**: Maximum performance, acceptable quality
- **Ultra Performance (33%)**: Extreme upsampling (quality suffers)

**Console variables**:
```
r.TSR.ShadingRejection.Flickering=0.5     // Rejection threshold
r.TSR.History.R11G11B10F=1                 // Compressed history format
r.TSR.Velocity.Extrapolation=1             // Velocity extrapolation
r.TemporalAA.Upsampling=1                  // Enable upsampling
r.TemporalAA.Quality=2                     // 0=Low, 1=Medium, 2=High
```

### Memory Overhead

**TSR requires**:
- History buffers (1-2 frames)
- Velocity buffer
- Intermediate targets

**Typical overhead**: 50-100MB (resolution-dependent)

## Debugging Tips

### 1. Visualize TSR Components

**Console commands**:
```
r.TSR.Debug.ArraySize=1                    // Show debug views
r.TSR.Visualize=1                          // Enable visualization
```

**Debug views**:
- Input: Current frame
- History: Reprojected history
- Velocity: Motion vectors
- Rejection: Where history was rejected

### 2. Compare TSR On/Off

**Toggle TSR**:
```
r.TemporalAA.Upsampling=0   // Disable TSR
r.TemporalAA.Upsampling=1   // Enable TSR
```

**Compare**:
- Performance (frame time)
- Quality (sharpness, aliasing, stability)

### 3. Check Velocity Buffer

**Visualize velocity**:
```
vis Velocity  // Show motion vectors
```

**Look for**:
- Incorrect velocity (objects with wrong motion)
- Missing velocity (should move but doesn't)
- Velocity discontinuities (sharp changes)

### 4. Adjust Rejection Thresholds

**If too much ghosting**:
```
r.TSR.ShadingRejection.SpatialFilter=1     // More aggressive rejection
r.TSR.Velocity.WeightClamp=0.1             // Lower history weight
```

**If too much flickering**:
```
r.TSR.ShadingRejection.Flickering=0.7      // More lenient rejection
r.TSR.History.SampleCount=2                // More history samples
```

### 5. Profile TSR

**Console commands**:
```
stat SceneRendering  // Shows TSR timing
profilegpu           // Detailed breakdown
```

**Typical timings**:
- TSR resolve: 1-3ms
- History update: 0.5-1ms
- Total: ~1.5-4ms (depends on resolution, quality)

### 6. Common Artifacts

**Ghosting**:
- Trails behind moving objects
- Fix: Increase rejection sensitivity, improve velocity

**Flickering**:
- Temporal instability (shimmering)
- Fix: Increase history weight, reduce rejection

**Blurriness**:
- Overly soft image
- Fix: Reduce history weight, increase sharpness, use higher internal resolution

**Crawling**:
- Edges seem to "crawl" or shimmer
- Fix: Adjust sub-pixel jitter pattern, improve AA quality

## Advanced Topics

### Velocity Dilution

Dilate velocity buffer to fill holes:

```hlsl
// Find max velocity magnitude in neighborhood
float2 MaxVelocity = VelocityBuffer[PixelPos].xy;
float MaxMagnitude = length(MaxVelocity);

for (int i = 0; i < NeighborCount; ++i)
{
    float2 NeighborVelocity = VelocityBuffer[PixelPos + Offset[i]].xy;
    float Magnitude = length(NeighborVelocity);
    
    if (Magnitude > MaxMagnitude)
    {
        MaxVelocity = NeighborVelocity;
        MaxMagnitude = Magnitude;
    }
}

// Use max velocity (covers object boundaries better)
return MaxVelocity;
```

### Responsive AA

TSR can prioritize responsiveness over stability:

```cpp
// High responsiveness: Lower history weight (less ghosting, more flicker)
r.TSR.History.ResponseiveStencil=1  // Use stencil to mark responsive areas
```

Useful for:
- Fast-paced games (FPS)
- User input areas (crosshairs, UI)

### Sharpening

TSR can apply sharpening:

```
r.TSR.Sharpness=0.3  // 0=None, 1=Maximum sharpness
```

**Implementation**:
- Unsharp masking or similar technique
- Applied during or after upsampling
- Counteracts temporal blur

### Dynamic Resolution

TSR works with dynamic resolution:
- Adjust internal resolution based on GPU load
- TSR upsamples to fixed output resolution
- Maintains stable output res while varying internal res

**Console variables**:
```
r.DynamicRes.OperationMode=2               // Enable dynamic resolution
r.DynamicRes.TargetedGPUHeadRoom=10        // Target GPU headroom (ms)
```

### VRS (Variable Rate Shading)

Combine TSR with VRS:
- Render different regions at different rates
- TSR upsamples all to final resolution
- Further performance gains

## Best Practices

### 1. Always Generate Velocity

Ensure all objects output correct velocity:
- Static meshes: Automatic (transform changes)
- Skeletal meshes: Automatic (bone transforms)
- Custom vertex animation: Output velocity manually
- Particles: May need custom velocity

### 2. Choose Appropriate Resolution Scale

**Guidelines**:
- **4K output**: 67% scale (2560×1440) - excellent quality
- **1440p output**: 67-75% scale - good balance
- **1080p output**: 75-85% scale - quality focus

**Lower scales**:
- For weaker hardware
- When baseline rendering is slow

### 3. Balance with Other Features

TSR performance gain allows:
- Higher quality settings (shadows, GI)
- More post-process effects
- Better lighting

**Don't**:
- Use lowest scale and call it a day
- Sacrifice base quality excessively

### 4. Test Temporal Stability

**Test scenarios**:
- Fast camera movement
- Moving objects
- Particles and effects
- Lighting changes

**Look for**:
- Ghosting
- Flickering
- Loss of detail

### 5. Platform-Specific Tuning

**Console** (PS5, Xbox Series):
- 67% scale typical
- High quality settings

**PC High-End**:
- 75-85% scale (already fast)
- Focus on quality

**PC Mid-Range**:
- 50-67% scale
- Balance quality/performance

**Mobile** (if TSR supported):
- 50% or lower scale
- Performance priority

## Summary

TSR provides:
1. **Temporal upsampling** - Render lower, output higher
2. **High-quality AA** - Better than TAA alone
3. **Performance gains** - 30-60% faster rendering
4. **Temporal stability** - Reduced flickering and aliasing

Key techniques:
- Motion vectors for reprojection
- History validation and rejection
- Adaptive temporal blending
- Spatial upsampling

Key classes: `FTSR`, `FTemporalAAHistory`

TSR is essential for modern UE5 projects, providing both quality and performance. It's especially valuable for 4K rendering and demanding scenes with Lumen and Nanite.

---

## Conclusion

This completes the 10-chapter journey through UE5's rendering pipeline. You've learned:

1. **Materials → HLSL** (Chapter 1)
2. **Shader Compilation** (Chapter 2)
3. **RDG** (Chapter 3)
4. **BasePass** (Chapter 4)
5. **Lighting** (Chapter 5)
6. **Shadows** (Chapter 6)
7. **Post-Processing** (Chapter 7)
8. **Lumen** (Chapter 8)
9. **Nanite** (Chapter 9)
10. **TSR** (Chapter 10)

Together, these systems form the foundation of UE5's cutting-edge rendering capabilities. Continue exploring the source code, experiment with settings, and build amazing real-time graphics!
