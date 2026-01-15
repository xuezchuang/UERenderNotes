# Chapter 7: Post Processing

## Goals

By the end of this chapter, you will understand:
- UE5's post-process pipeline architecture
- Major post-process effects (tone mapping, bloom, DOF, motion blur)
- Post-process material system
- Performance optimization for post-process effects
- How to create custom post-process effects

## Overview

Post-processing applies screen-space effects after the scene is rendered. These effects enhance visual quality, mood, and realism. UE5's post-process system is highly flexible, supporting both built-in effects and custom material-based effects.

**Common post-process effects**:
- **Tone mapping**: HDR → LDR conversion
- **Auto exposure**: Automatic brightness adjustment
- **Bloom**: Light glow/bleeding
- **Depth of Field (DOF)**: Camera focus simulation
- **Motion Blur**: Movement blur
- **Screen Space Ambient Occlusion (SSAO)**: Contact shadows
- **Color Grading**: Color correction and artistic look

## Pipeline Steps

### 1. Post-Process Input Gathering
**What happens**: Collect scene color, depth, GBuffer, and velocity buffers.

**Inputs typically include**:
- Scene color (HDR)
- Depth buffer
- GBuffer (normals, roughness, etc.)
- Velocity buffer (for motion blur, TAA)

### 2. Screen Space Effects
**What happens**: Apply effects that use screen-space data.

**Examples**:
- SSAO (uses depth and normals)
- SSR (screen-space reflections, Chapter 5)
- Screen-space GI (SSGI)

### 3. Temporal Effects
**What happens**: Effects that use previous frame data.

**Examples**:
- Temporal Anti-Aliasing (TAA)
- Temporal Super Resolution (TSR, Chapter 10)
- Temporal upsampling

### 4. Tone Mapping and Color Grading
**What happens**: Convert HDR to display-ready LDR.

**Steps**:
1. Calculate scene luminance (auto exposure)
2. Apply tone mapping curve
3. Apply color grading LUT (Look-Up Table)
4. Output to display format

### 5. Additional Effects
**What happens**: Apply bloom, DOF, motion blur, etc.

**Order matters**:
- Some effects before tone mapping (bloom)
- Some after tone mapping (film grain, vignette)

### 6. Final Output
**What happens**: Prepare for display or further processing.

**Output**:
- sRGB color space
- Gamma correction applied
- Ready for display or capture

## Key UE Source Files, Classes, and Functions

### Post-Process Pipeline

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessing.h`**
- `FPostProcessing` - Main post-process orchestration
- Post-process pass definitions

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessing.cpp`**
- `AddPostProcessingPasses()` - Builds post-process pass graph
- Main entry point for post-processing

### Tone Mapping

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessTonemap.h`**
- `FPostProcessTonemapPS` - Tone mapping pixel shader
- Tone mapper selection

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessTonemap.cpp`**
- `AddTonemapPass()` - Add tone mapping to RDG
- Auto exposure integration

**`Engine/Shaders/Private/PostProcessTonemap.usf`**
- Tone mapping HLSL implementation
- Multiple tone curve options (ACES, Reinhard, etc.)

### Bloom

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessBloom.h`**
- `FPostProcessBloomSetupPS` - Bloom threshold and setup

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessBloom.cpp`**
- `AddBloomPass()` - Bloom downsampling and blurring
- Lens flares integration

### Depth of Field

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessDOF.h`**
- Depth of field implementations (Gaussian, Bokeh)

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessDOF.cpp`**
- `AddDepthOfFieldPasses()` - DOF effect passes
- Circle of confusion calculation

### Motion Blur

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessMotionBlur.h`**
- `FPostProcessMotionBlurPS` - Motion blur shader

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessMotionBlur.cpp`**
- `AddMotionBlurPass()` - Motion blur using velocity buffer

### Ambient Occlusion

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessAmbientOcclusion.h`**
- SSAO implementation

**`Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessAmbientOcclusion.cpp`**
- `AddAmbientOcclusionPasses()` - SSAO computation

### Post-Process Materials

**`Engine/Source/Runtime/Engine/Classes/Materials/MaterialInstance.h`**
- `UMaterialInstanceDynamic` - Runtime material creation
- Post-process material interface

### Key Functions

```cpp
// PostProcessing.cpp
void AddPostProcessingPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FPostProcessingInputs& Inputs);

// PostProcessTonemap.cpp
FScreenPassTexture AddTonemapPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FTonemapInputs& Inputs);

// PostProcessBloom.cpp
FScreenPassTexture AddBloomPass(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FBloomInputs& Inputs);

// PostProcessDOF.cpp
void AddDepthOfFieldPasses(
    FRDGBuilder& GraphBuilder,
    const FViewInfo& View,
    const FSceneTextureInputs& SceneTextures,
    FScreenPassTexture& OutColor);
```

## Tone Mapping

### HDR to LDR Conversion

HDR scenes have luminance values beyond display range (0-1). Tone mapping compresses this range while preserving details.

**Common tone mapping curves**:

**ACES (Academy Color Encoding System)**:
```hlsl
float3 ACESFilm(float3 x)
{
    float a = 2.51;
    float b = 0.03;
    float c = 2.43;
    float d = 0.59;
    float e = 0.14;
    return saturate((x * (a * x + b)) / (x * (c * x + d) + e));
}
```

**Reinhard**:
```hlsl
float3 Reinhard(float3 color, float exposure)
{
    color *= exposure;
    return color / (1 + color);
}
```

**Uncharted 2 (John Hable)**:
```hlsl
float3 Uncharted2Tonemap(float3 x)
{
    float A = 0.15;
    float B = 0.50;
    float C = 0.10;
    float D = 0.20;
    float E = 0.02;
    float F = 0.30;
    return ((x*(A*x+C*B)+D*E)/(x*(A*x+B)+D*F))-E/F;
}
```

### Auto Exposure

Simulates camera/eye adaptation to brightness:

**Algorithm**:
1. Compute average scene luminance (downsampling, histogram)
2. Calculate target exposure based on luminance
3. Smoothly adapt exposure over time (temporal)
4. Apply exposure multiplier to scene color

**Console variables**:
```
r.DefaultFeature.AutoExposure=1           // Enable auto exposure
r.DefaultFeature.AutoExposure.Method=2    // 0=Basic, 1=Histogram, 2=Basic+Bias
r.EyeAdaptation.MinBrightness=0.1        // Min adaptation brightness
r.EyeAdaptation.MaxBrightness=2.0        // Max adaptation brightness
r.EyeAdaptation.SpeedUp=3.0              // Adaptation speed (dark→bright)
r.EyeAdaptation.SpeedDown=1.0            // Adaptation speed (bright→dark)
```

### Color Grading

Apply artistic color adjustments:

**LUT (Look-Up Table)**:
- 3D color transformation table
- Pre-computed color corrections
- Very efficient at runtime

**Process**:
1. Artist creates color grading in DCC tool (DaVinci, Photoshop)
2. Export as LUT texture (typically 32×32×32 3D texture)
3. At runtime, use input color as 3D coordinate into LUT
4. Sample LUT to get corrected color

**In shader**:
```hlsl
float3 ColorGrade(float3 Color, Texture3D LUT)
{
    // Scale color to LUT coordinate space
    float3 LUTCoord = saturate(Color) * (LUT_SIZE - 1) / LUT_SIZE;
    LUTCoord += 0.5 / LUT_SIZE;  // Center of texel
    
    // Sample LUT
    return LUT.SampleLevel(LUTSampler, LUTCoord, 0).rgb;
}
```

**Settings**:
```cpp
// Post Process Volume → Color Grading
Temperature, Tint, Saturation, Contrast, Gamma, Gain, Offset
```

## Bloom

### What Is Bloom?

Bloom simulates light bleeding/glow around bright areas, mimicking camera lens and eye behavior.

**Algorithm**:
1. **Threshold**: Extract bright pixels (above threshold)
2. **Downsample**: Create mip chain (blur happens naturally)
3. **Blur**: Apply gaussian blur at each mip level
4. **Upsample**: Recombine mip levels
5. **Composite**: Add bloom to scene color

### Implementation

**Extract bright pixels**:
```hlsl
float3 ExtractBloom(float3 SceneColor, float Threshold)
{
    float Brightness = dot(SceneColor, float3(0.299, 0.587, 0.114));
    float Contribution = saturate(Brightness - Threshold);
    return SceneColor * Contribution;
}
```

**Gaussian blur**:
```hlsl
float3 GaussianBlur(Texture2D Input, float2 UV, float2 TexelSize, float2 Direction)
{
    // 9-tap Gaussian kernel
    const float Weights[5] = {0.227027, 0.1945946, 0.1216216, 0.054054, 0.016216};
    
    float3 Result = Input.Sample(Sampler, UV).rgb * Weights[0];
    for (int i = 1; i < 5; ++i)
    {
        float2 Offset = Direction * TexelSize * i;
        Result += Input.Sample(Sampler, UV + Offset).rgb * Weights[i];
        Result += Input.Sample(Sampler, UV - Offset).rgb * Weights[i];
    }
    return Result;
}
```

**Composite**:
```hlsl
float3 FinalColor = SceneColor + BloomColor * BloomIntensity;
```

### Console Variables

```
r.BloomQuality=5                  // 0-5, quality level
r.Bloom.Threshold=1.0             // Brightness threshold
r.Bloom.ScaleMultiplier=1.0       // Bloom intensity
```

## Depth of Field

### Types of DOF

**Gaussian DOF**:
- Fast approximation
- Uniform blur based on circle of confusion

**Bokeh DOF**:
- Higher quality
- Simulates camera aperture shape
- More expensive

### Circle of Confusion (CoC)

Determines blur amount per pixel:

```hlsl
float CalculateCoC(
    float SceneDepth,
    float FocusDistance,
    float FocalLength,
    float Aperture)
{
    // Thin lens equation
    float FocusedDepth = (FocalLength * FocusDistance) / (FocalLength + FocusDistance);
    float CoC = Aperture * FocalLength * abs(SceneDepth - FocusedDepth) / (SceneDepth * (FocusedDepth - FocalLength));
    return CoC;
}
```

### Bokeh DOF Algorithm

1. **CoC calculation**: Determine blur radius per pixel
2. **Separate layers**: Near, far, and focused layers
3. **Scatter/Gather**: Simulate light gathering on sensor
4. **Composite**: Blend layers based on CoC

### Console Variables

```
r.DepthOfFieldQuality=4           // 0-4, quality level
r.DOF.Gather.PostfilterMethod=1   // 0=None, 1=Median
r.DOF.Gather.RingCount=5          // Number of sample rings (quality)
```

## Motion Blur

### Per-Object Motion Blur

Uses velocity buffer (from BasePass):

**Velocity calculation**:
```hlsl
// Vertex shader
float4 PrevClipPos = mul(float4(WorldPos, 1), PrevViewProj);
float4 CurrClipPos = mul(float4(WorldPos, 1), CurrViewProj);

float2 PrevScreenPos = PrevClipPos.xy / PrevClipPos.w;
float2 CurrScreenPos = CurrClipPos.xy / CurrClipPos.w;

float2 Velocity = (CurrScreenPos - PrevScreenPos) * 0.5;
```

**Motion blur pixel shader**:
```hlsl
float3 MotionBlur(Texture2D SceneColor, Texture2D Velocity, float2 UV)
{
    float2 Vel = Velocity.Sample(Sampler, UV).xy;
    
    // Sample along velocity vector
    float3 Result = 0;
    const int Samples = 8;
    for (int i = 0; i < Samples; ++i)
    {
        float t = (float)i / Samples;
        float2 SampleUV = UV + Vel * (t - 0.5);
        Result += SceneColor.Sample(Sampler, SampleUV).rgb;
    }
    return Result / Samples;
}
```

### Console Variables

```
r.MotionBlurQuality=4             // 0-4, quality level
r.MotionBlurSampleCount=8         // Number of samples
r.MotionBlur.Max=5.0              // Max blur amount (pixels)
r.MotionBlur.Scale=1.0            // Overall intensity
```

## Ambient Occlusion (SSAO)

### Screen Space Ambient Occlusion

Approximates ambient occlusion using screen-space data:

**Algorithm**:
1. For each pixel, sample surrounding depth
2. Check if samples are occluded (above surface)
3. Accumulate occlusion factor
4. Apply to lighting

**Implementation**:
```hlsl
float ComputeSSAO(float2 UV, float3 ViewPos, float3 ViewNormal)
{
    float Occlusion = 0;
    const int SampleCount = 16;
    
    for (int i = 0; i < SampleCount; ++i)
    {
        // Random sample in hemisphere around normal
        float3 SampleDir = GetRandomHemisphereSample(i, ViewNormal);
        float3 SamplePos = ViewPos + SampleDir * AORadius;
        
        // Project to screen space
        float2 SampleUV = ViewPosToScreenUV(SamplePos);
        
        // Get depth at sample position
        float SampleDepth = DepthTexture.Sample(Sampler, SampleUV).r;
        float3 SampleViewPos = ScreenUVToViewPos(SampleUV, SampleDepth);
        
        // Check if sample is occluded
        float RangeCheck = smoothstep(0, 1, AORadius / abs(ViewPos.z - SampleViewPos.z));
        Occlusion += (SampleViewPos.z >= SamplePos.z ? 1.0 : 0.0) * RangeCheck;
    }
    
    return 1.0 - (Occlusion / SampleCount);
}
```

### Console Variables

```
r.AmbientOcclusionLevels=-1       // -1=Auto, 0-3=Quality levels
r.AmbientOcclusionRadiusScale=1.0 // AO radius multiplier
r.AmbientOcclusion.Compute=0      // 0=Pixel shader, 1=Compute shader
```

## Post-Process Materials

### Creating Post-Process Materials

Custom post-process effects using Material Editor:

**Steps**:
1. Create new Material
2. Set Material Domain = "Post Process"
3. Use `SceneTexture` nodes to sample scene color, depth, etc.
4. Implement effect logic
5. Assign to Post Process Volume

**Example - Edge Detection**:
```cpp
// In Material Editor:
// 1. Sample SceneColor at current UV
// 2. Sample SceneColor at neighboring UVs (offset)
// 3. Compute difference (edge detection)
// 4. Blend with original color

float3 EdgeColor = float3(1, 0, 0);  // Red edges
float EdgeThreshold = 0.1;

float3 C = SceneTexture(SceneColor, UV);
float3 L = SceneTexture(SceneColor, UV + float2(-TexelSize.x, 0));
float3 R = SceneTexture(SceneColor, UV + float2(+TexelSize.x, 0));
float3 T = SceneTexture(SceneColor, UV + float2(0, -TexelSize.y));
float3 B = SceneTexture(SceneColor, UV + float2(0, +TexelSize.y));

float EdgeX = length(R - L);
float EdgeY = length(B - T);
float Edge = saturate((EdgeX + EdgeY) / EdgeThreshold);

return lerp(C, EdgeColor, Edge);
```

### Post-Process Material Settings

**Blendable Location**:
- **Before Tonemapping**: Access HDR values
- **After Tonemapping**: LDR, after color grading
- **Replacing Tonemapping**: Custom tone mapper

**Priority**:
- Multiple post-process materials blend by priority (0-1)

### Console Variables

```
r.PostProcessAAQuality=6          // Anti-aliasing quality
r.PostProcessing.PropagateAlpha=0 // Propagate alpha through post-process
```

## Debugging Tips

### 1. Visualize Post-Process Effects

**Disable post-processing**:
```
showflag.PostProcessing 0  // Disable all post-process
```

**Disable individual effects**:
```
showflag.Bloom 0
showflag.DepthOfField 0
showflag.MotionBlur 0
showflag.AmbientOcclusion 0
showflag.Tonemapper 0
```

### 2. Check Post-Process Volume

**Verify volume settings**:
- Select Post Process Volume in scene
- Check "Unbound" or ensure camera is inside volume
- Verify effect settings (intensity, etc.)

### 3. Profile Post-Process

**Console commands**:
```
stat SceneRendering  // Shows post-process timing
profilegpu           // Detailed breakdown
```

**Typical timings**:
- Tone mapping: 0.1-0.5ms
- Bloom: 1-3ms
- DOF: 2-5ms (depends on quality)
- Motion blur: 1-2ms
- SSAO: 1-3ms

### 4. Debug Buffers

**Visualize inputs**:
```
viewmode SceneDepth        // Depth buffer
viewmode WorldNormal       // Normals
vis Velocity               // Motion vectors
```

### 5. Post-Process Material Debugging

**Check material**:
- Ensure Material Domain is "Post Process"
- Verify Blendable Location
- Check if assigned to Post Process Volume

**Test in isolation**:
- Disable other post-process effects
- Verify material compiles without errors

## Advanced Topics

### Custom Tonemapper

Replace UE's tonemapper with custom:

**Create post-process material**:
- Blendable Location = "Replacing Tonemapper"
- Implement custom tone curve
- Sample scene color, apply curve, output

### Multi-Pass Effects

Complex effects may need multiple passes:
- Use render targets
- Ping-pong between textures
- Build pass graph manually

### Temporal Effects

Access previous frame data:
```cpp
// SceneTexture node → "PrevFrameColor"
```

Useful for:
- Temporal smoothing
- Frame blending effects
- History-based techniques

### Compute Shader Post-Process

Some effects benefit from compute shaders:
- Better memory access patterns
- Shared memory for local data
- More control over thread groups

**Example**: `r.AmbientOcclusion.Compute=1`

## Performance Considerations

### Resolution Scaling

Many effects don't need full resolution:
- Bloom: Often rendered at half resolution
- DOF: Can be lower resolution
- SSAO: Quarter resolution common

**Benefits**: Reduces fill-rate, memory bandwidth

### Effect Quality Settings

Scalability settings for post-process:

**Project Settings → Engine → Rendering → Post Processing**:
- Low: Minimal effects, low sample counts
- Medium: Balanced
- High: High quality
- Epic: Maximum quality
- Cinematic: Offline quality

### Disable Expensive Effects

On lower-end hardware:
```
r.DepthOfFieldQuality=0     // Disable DOF
r.MotionBlurQuality=0       // Disable motion blur
r.AmbientOcclusionLevels=0  // Disable SSAO
```

### Bloom Optimization

```
r.BloomQuality=2            // Lower quality (0-5)
r.Bloom.Cross=0             // Disable lens flares
```

## Summary

Post-processing pipeline:
1. **Screen-space effects** - SSAO, SSR
2. **Temporal effects** - TAA, TSR
3. **Tone mapping** - HDR → LDR
4. **Additional effects** - Bloom, DOF, motion blur
5. **Final output** - Display-ready image

Key effects: Tone mapping, bloom, DOF, motion blur, SSAO

Key classes: `FPostProcessing`, `FScreenPassTexture`

Post-processing is critical for visual quality but must be balanced with performance. UE5 provides excellent built-in effects and extensibility via post-process materials.

## Next Chapter

[Chapter 8: Lumen Global Illumination](08-lumen.md) - Learn UE5's dynamic global illumination system.
