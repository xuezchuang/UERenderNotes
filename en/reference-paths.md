# Reference Paths

Quick lookup for UE5 source files, classes, and functions organized by rendering system.

## Material System

### Core Classes
```
Engine/Source/Runtime/Engine/Classes/Materials/Material.h
  └─ UMaterial - Base material asset class

Engine/Source/Runtime/Engine/Classes/Materials/MaterialExpression.h
  └─ UMaterialExpression - Base class for material nodes

Engine/Source/Runtime/Engine/Classes/Materials/MaterialInstance.h
  └─ UMaterialInstance - Material instance with parameter overrides
```

### Material Compilation
```
Engine/Source/Runtime/Engine/Private/Materials/MaterialCompiler.h
  └─ FMaterialCompiler - Material compilation interface

Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.h
  └─ FHLSLMaterialTranslator - Material graph to HLSL translator

Engine/Source/Runtime/Engine/Private/Materials/Material.cpp
  └─ UMaterial::CacheShadersForResources() - Trigger material compilation
```

### Shaders
```
Engine/Shaders/Private/MaterialTemplate.ush
  └─ Base material shader template (HLSL)

Engine/Shaders/Private/DeferredShadingCommon.ush
  └─ Shared deferred shading utilities
```

## Shader Compilation

### Compilation Core
```
Engine/Source/Runtime/ShaderCore/Public/ShaderCompiler.h
  └─ FShaderCompilerInput, FShaderCompilerOutput, FShaderCompileJob

Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.cpp
  └─ CompileShader() - Main compilation entry point
  └─ GlobalBeginCompileShader() - Async compilation start
```

### Shader Types
```
Engine/Source/Runtime/ShaderCore/Public/Shader.h
  └─ FShader - Base shader class
  └─ FGlobalShader - Engine-wide shaders
  └─ FMaterialShader - Material-specific shaders
  └─ FMeshMaterialShader - Mesh material shaders
```

### Shader Maps
```
Engine/Source/Runtime/Engine/Public/MaterialShaderMap.h
  └─ FMaterialShaderMap - Compiled shader variants for material

Engine/Source/Runtime/ShaderCore/Public/ShaderCodeLibrary.h
  └─ FShaderCodeLibrary - Shader storage and loading
```

### Platform Compilers
```
Engine/Source/Developer/Windows/ShaderFormatD3D/Private/ShaderFormatD3D.cpp
  └─ DirectX shader compilation (DXC)

Engine/Source/Developer/ShaderFormatVectorVM/Private/ShaderFormatVectorVM.cpp
  └─ Vector VM shaders (Niagara)
```

## Render Dependency Graph (RDG)

### Core RDG
```
Engine/Source/Runtime/RenderCore/Public/RenderGraphBuilder.h
  └─ FRDGBuilder - Main RDG interface
  └─ AddPass() - Register render pass
  └─ CreateTexture(), CreateBuffer() - Resource creation
  └─ Execute() - Execute render graph

Engine/Source/Runtime/RenderCore/Public/RenderGraphResources.h
  └─ FRDGTexture, FRDGBuffer - RDG resource handles
  └─ FRDGTextureDesc, FRDGBufferDesc - Resource descriptors
```

### RDG Passes
```
Engine/Source/Runtime/RenderCore/Public/RenderGraphPass.h
  └─ FRDGPass - Base pass class

Engine/Source/Runtime/RenderCore/Public/RenderGraphParameters.h
  └─ RDG_TEXTURE_ACCESS(), RDG_BUFFER_ACCESS() - Parameter macros
```

## BasePass Rendering

### BasePass Core
```
Engine/Source/Runtime/Renderer/Private/BasePassRendering.h
  └─ TBasePassPixelShaderBaseType - BasePass pixel shader
  └─ TBasePassVertexShaderBaseType - BasePass vertex shader

Engine/Source/Runtime/Renderer/Private/BasePassRendering.cpp
  └─ FBasePassMeshProcessor - Mesh processing for BasePass

Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp
  └─ FDeferredShadingSceneRenderer::RenderBasePass() - Main BasePass entry
```

### GBuffer
```
Engine/Source/Runtime/Renderer/Private/SceneRendering.h
  └─ FSceneRenderTargets - Manages scene render targets including GBuffer

Engine/Source/Runtime/Renderer/Private/SceneRenderTargets.cpp
  └─ AllocateGBufferTargets() - GBuffer allocation
```

### Vertex Factories
```
Engine/Source/Runtime/Engine/Public/VertexFactory.h
  └─ FVertexFactory - Base vertex factory class

Engine/Source/Runtime/Engine/Public/LocalVertexFactory.h
  └─ FLocalVertexFactory - Static mesh vertex factory

Engine/Source/Runtime/Engine/Public/GPUSkinVertexFactory.h
  └─ FGPUSkinVertexFactory - Skeletal mesh vertex factory
```

### Mesh Processing
```
Engine/Source/Runtime/Renderer/Private/MeshPassProcessor.h
  └─ FMeshPassProcessor - Base mesh pass processor
  └─ AddMeshBatch(), BuildMeshDrawCommands()

Engine/Source/Runtime/Renderer/Private/SceneVisibility.cpp
  └─ ComputeViewVisibility() - Visibility determination
```

## Lighting System

### Deferred Lighting
```
Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp
  └─ FDeferredShadingSceneRenderer::RenderLights() - Main lighting entry

Engine/Source/Runtime/Renderer/Private/LightRendering.cpp
  └─ RenderLight() - Render single light
```

### Light Types
```
Engine/Source/Runtime/Engine/Classes/Components/LightComponent.h
  └─ ULightComponent - Base light class

Engine/Source/Runtime/Engine/Classes/Components/PointLightComponent.h
  └─ UPointLightComponent - Point light

Engine/Source/Runtime/Engine/Classes/Components/SpotLightComponent.h
  └─ USpotLightComponent - Spot light

Engine/Source/Runtime/Engine/Classes/Components/DirectionalLightComponent.h
  └─ UDirectionalLightComponent - Directional light
```

### Light Scene Info
```
Engine/Source/Runtime/Renderer/Private/LightSceneInfo.h
  └─ FLightSceneInfo - Light representation in scene
```

### Lighting Shaders
```
Engine/Shaders/Private/DeferredLightingCommon.ush
  └─ BRDF evaluation functions (HLSL)

Engine/Shaders/Private/DeferredLightPixelShaders.usf
  └─ Deferred light pixel shaders

Engine/Shaders/Private/BRDF.ush
  └─ BRDF implementations (GGX, Lambert, etc.)

Engine/Shaders/Private/ReflectionEnvironmentShaders.usf
  └─ IBL and reflection capture sampling
```

### Light Culling
```
Engine/Source/Runtime/Renderer/Private/LightGridInjection.cpp
  └─ Clustered/tiled light culling
```

## Shadow Rendering

### Shadow Core
```
Engine/Source/Runtime/Renderer/Private/ShadowRendering.h
  └─ FProjectedShadowInfo - Shadow map representation

Engine/Source/Runtime/Renderer/Private/ShadowRendering.cpp
  └─ FProjectedShadowInfo::RenderDepth() - Render shadow map

Engine/Source/Runtime/Renderer/Private/ShadowSetup.cpp
  └─ InitProjectedShadowInfo() - Shadow projection setup
```

### Cascaded Shadow Maps
```
Engine/Source/Runtime/Renderer/Private/ShadowSetup.cpp
  └─ SetupWholeSceneProjection() - CSM cascade setup

Engine/Shaders/Private/ShadowProjectionCommon.ush
  └─ CSM sampling and filtering (HLSL)
```

### Virtual Shadow Maps
```
Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.h
  └─ FVirtualShadowMapArray - VSM management

Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.cpp
  └─ RenderVirtualShadowMaps() - VSM rendering

Engine/Shaders/Private/VirtualShadowMaps/VirtualShadowMapProjection.ush
  └─ VSM sampling (HLSL)
```

### Shadow Depth
```
Engine/Source/Runtime/Renderer/Private/ShadowDepthRendering.h
  └─ FShadowDepthPassMeshProcessor - Shadow depth mesh processing

Engine/Source/Runtime/Renderer/Private/ShadowDepthRendering.cpp
  └─ Shadow depth rendering implementation
```

## Post-Processing

### Post-Process Pipeline
```
Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessing.h
  └─ FPostProcessing - Post-process orchestration

Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessing.cpp
  └─ AddPostProcessingPasses() - Build post-process graph
```

### Tone Mapping
```
Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessTonemap.h
  └─ Tone mapping interface

Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessTonemap.cpp
  └─ AddTonemapPass() - Add tone mapping

Engine/Shaders/Private/PostProcessTonemap.usf
  └─ Tone mapping shaders (ACES, Reinhard, etc.)
```

### Bloom
```
Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessBloom.h
  └─ Bloom interface

Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessBloom.cpp
  └─ AddBloomPass() - Bloom implementation
```

### Depth of Field
```
Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessDOF.h
  └─ DOF interface

Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessDOF.cpp
  └─ AddDepthOfFieldPasses() - DOF implementation
```

### Motion Blur
```
Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessMotionBlur.h
  └─ Motion blur interface

Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessMotionBlur.cpp
  └─ AddMotionBlurPass() - Motion blur implementation
```

### Ambient Occlusion
```
Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessAmbientOcclusion.h
  └─ SSAO interface

Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessAmbientOcclusion.cpp
  └─ AddAmbientOcclusionPasses() - SSAO implementation
```

## Lumen Global Illumination

### Lumen Core
```
Engine/Source/Runtime/Renderer/Private/Lumen/Lumen.h
  └─ FLumenSceneData - Lumen scene representation

Engine/Source/Runtime/Renderer/Private/Lumen/Lumen.cpp
  └─ InitializeLumenSceneData() - Lumen initialization
```

### Surface Cache
```
Engine/Source/Runtime/Renderer/Private/Lumen/LumenSurfaceCache.h
  └─ FLumenSurfaceCacheData - Surface cache management

Engine/Source/Runtime/Renderer/Private/Lumen/LumenSurfaceCache.cpp
  └─ UpdateSurfaceCache() - Surface cache update

Engine/Shaders/Private/Lumen/LumenSurfaceCache.usf
  └─ Surface cache shaders (HLSL)
```

### Radiance Cache
```
Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadianceCache.h
  └─ FLumenRadianceCacheInputs - Radiance cache config

Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadianceCache.cpp
  └─ RenderRadianceCache() - Radiance cache update

Engine/Shaders/Private/Lumen/LumenRadianceCache.usf
  └─ Radiance cache shaders (HLSL)
```

### Screen Probes
```
Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeGather.h
  └─ Screen probe system

Engine/Source/Runtime/Renderer/Private/Lumen/LumenScreenProbeGather.cpp
  └─ RenderScreenProbeGather() - Screen probe gathering
```

### Reflections
```
Engine/Source/Runtime/Renderer/Private/Lumen/LumenReflections.cpp
  └─ RenderLumenReflections() - Lumen reflection rendering
```

### Ray Tracing
```
Engine/Shaders/Private/Lumen/LumenRadianceCacheHardwareRayTracing.usf
  └─ Hardware ray tracing shaders (RTX/DXR)

Engine/Shaders/Private/Lumen/LumenSceneLighting.usf
  └─ Software ray tracing (distance fields)
```

## Nanite Virtualized Geometry

### Nanite Core
```
Engine/Source/Runtime/Renderer/Private/Nanite/Nanite.h
  └─ Nanite namespace - Core functionality

Engine/Source/Runtime/Renderer/Private/Nanite/NaniteRender.h
  └─ FNaniteVisibilityResults - Visibility query results

Engine/Source/Runtime/Renderer/Private/Nanite/NaniteRender.cpp
  └─ CullRasterize() - Main Nanite rendering entry
```

### Nanite Resources
```
Engine/Source/Runtime/Engine/Public/Rendering/NaniteResources.h
  └─ FNaniteResources - Nanite mesh data

Engine/Source/Runtime/Engine/Private/Rendering/NaniteResources.cpp
  └─ Resource management and streaming
```

### Culling
```
Engine/Source/Runtime/Renderer/Private/Nanite/NaniteCulling.h
  └─ Culling pass declarations

Engine/Shaders/Private/Nanite/NaniteCulling.usf
  └─ GPU culling shaders (HLSL)
```

### Rasterization
```
Engine/Source/Runtime/Renderer/Private/Nanite/NaniteRasterizer.cpp
  └─ Software/hardware rasterization

Engine/Shaders/Private/Nanite/NaniteRasterize.usf
  └─ Rasterization shaders (HLSL)
```

### Material Evaluation
```
Engine/Source/Runtime/Renderer/Private/Nanite/NaniteVisualize.cpp
  └─ Nanite visualization

Engine/Shaders/Private/Nanite/NaniteExportGBuffer.usf
  └─ Material shading from visibility buffer
```

## Temporal Super Resolution (TSR)

### TSR Core
```
Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalSuperResolution.h
  └─ FTSR - Main TSR interface

Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalSuperResolution.cpp
  └─ AddTemporalSuperResolutionPasses() - TSR passes
```

### History Management
```
Engine/Source/Runtime/Renderer/Private/PostProcess/TemporalAA.h
  └─ FTemporalAAHistory - History buffer management
```

### TSR Shaders
```
Engine/Shaders/Private/TemporalSuperResolution/TSRCommon.ush
  └─ Shared TSR utilities (HLSL)

Engine/Shaders/Private/TemporalSuperResolution/TSRResolve.usf
  └─ TSR resolve pass (HLSL)

Engine/Shaders/Private/TemporalSuperResolution/TSRReject.usf
  └─ History validation (HLSL)

Engine/Shaders/Private/TemporalSuperResolution/TSRUpdate.usf
  └─ History update and AA (HLSL)
```

## Scene Rendering

### Scene Renderer
```
Engine/Source/Runtime/Renderer/Private/SceneRendering.h
  └─ FSceneRenderer - Base scene renderer
  └─ FViewInfo - View information

Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.h
  └─ FDeferredShadingSceneRenderer - Deferred renderer

Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp
  └─ Render() - Main rendering entry point
```

### Scene Management
```
Engine/Source/Runtime/Engine/Public/SceneInterface.h
  └─ FSceneInterface - Scene interface

Engine/Source/Runtime/Renderer/Private/Scene.h
  └─ FScene - Scene representation

Engine/Source/Runtime/Renderer/Private/PrimitiveSceneInfo.h
  └─ FPrimitiveSceneInfo - Primitive in scene
```

## Console Variables

Key console variable files:
```
Engine/Source/Runtime/Renderer/Private/RendererModule.cpp
  └─ Many rendering CVars registered here

Engine/Config/BaseEngine.ini
  └─ Default engine configuration

Engine/Config/BaseScalability.ini
  └─ Scalability settings and presets
```

## Commonly Used Shader Files

```
Engine/Shaders/Private/Common.ush
  └─ Common utilities and definitions

Engine/Shaders/Private/Definitions.usf
  └─ Global shader definitions

Engine/Shaders/Private/MaterialTemplate.ush
  └─ Material shader template

Engine/Shaders/Private/DeferredShadingCommon.ush
  └─ Deferred shading utilities

Engine/Shaders/Private/BRDF.ush
  └─ BRDF implementations

Engine/Shaders/Private/ShadingModels.ush
  └─ Shading model implementations
```

---

## Navigation Tips

### Finding Source Code

**By feature name**:
1. Search for class name in Visual Studio/Rider
2. Use "Go to File" (Ctrl+Shift+N in Rider)
3. Check corresponding header file first

**By console variable**:
1. Search for CVar name in source
2. Usually registered near implementation

**By shader code**:
1. Shader files in `Engine/Shaders/Private/`
2. Use extension: `.usf` (shader), `.ush` (header)

### Building UE5 from Source

Required to explore and modify these files:
1. Get source from Epic Games GitHub
2. Generate project files
3. Build in Visual Studio (Windows) or Xcode (Mac)
4. Debug in engine for deep understanding

### Documentation

- Official docs: https://docs.unrealengine.com/
- Source code comments: Often detailed
- This knowledge base: Cross-reference chapters

---

This reference guide provides starting points for deep dives into any rendering system. The UE5 source code is well-structured - once you understand the patterns, navigation becomes intuitive.
