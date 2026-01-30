# 引用路径

按渲染系统组织的 UE5 源文件、类和函数的快速查找。

## 材质系统

### 核心类
```
Engine/Source/Runtime/Engine/Classes/Materials/Material.h
  └─ UMaterial - 基础材质资产类

Engine/Source/Runtime/Engine/Classes/Materials/MaterialExpression.h
  └─ UMaterialExpression - 材质节点的基类

Engine/Source/Runtime/Engine/Classes/Materials/MaterialInstance.h
  └─ UMaterialInstance - 带参数覆盖的材质实例
```

### 材质编译
```
Engine/Source/Runtime/Engine/Private/Materials/MaterialCompiler.h
  └─ FMaterialCompiler - 材质编译接口

Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.h
  └─ FHLSLMaterialTranslator - 材质图到 HLSL 的转换器

Engine/Source/Runtime/Engine/Private/Materials/Material.cpp
  └─ UMaterial::CacheShadersForResources() - 触发材质编译
```

### 着色器
```
Engine/Shaders/Private/MaterialTemplate.ush
  └─ 基础材质着色器模板（HLSL）

Engine/Shaders/Private/DeferredShadingCommon.ush
  └─ 共享的延迟着色工具
```

## 着色器编译

### 编译核心
```
Engine/Source/Runtime/ShaderCore/Public/ShaderCompiler.h
  └─ FShaderCompilerInput, FShaderCompilerOutput, FShaderCompileJob

Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.cpp
  └─ CompileShader() - 主编译入口点
  └─ GlobalBeginCompileShader() - 异步编译开始
```

### ShaderCompileWorker
```
Engine/Source/Programs/ShaderCompileWorker/Private/ShaderCompileWorker.cpp
  └─ 独立的着色器编译进程

Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.cpp
  └─ GShaderCompilingManager - 全局编译管理器
  └─ FShaderCompileThreadRunnable - 编译线程管理
```

### 平台特定编译器
```
Engine/Source/Developer/Windows/ShaderFormatD3D/Private/ShaderFormatD3D.cpp
  └─ DirectX 着色器编译

Engine/Source/Developer/ShaderFormatVectorVM/Private/ShaderFormatVectorVM.cpp
  └─ Niagara VectorVM 着色器编译
```

## 渲染依赖图（RDG）

### 核心类
```
Engine/Source/Runtime/RenderCore/Public/RenderGraphBuilder.h
  └─ FRDGBuilder - RDG 构建器主类

Engine/Source/Runtime/RenderCore/Public/RenderGraphResources.h
  └─ FRDGTexture, FRDGBuffer - RDG 资源类型

Engine/Source/Runtime/RenderCore/Private/RenderGraphBuilder.cpp
  └─ FRDGBuilder::Execute() - 执行渲染图
```

### Pass 管理
```
Engine/Source/Runtime/RenderCore/Public/RenderGraphPass.h
  └─ FRDGPass - Pass 基类

Engine/Source/Runtime/RenderCore/Private/RenderGraphPass.cpp
  └─ Pass 执行逻辑
```

## BasePass 渲染

### BasePass 实现
```
Engine/Source/Runtime/Renderer/Private/BasePassRendering.h
  └─ FBasePassMeshProcessor - BasePass 网格处理器

Engine/Source/Runtime/Renderer/Private/BasePassRendering.cpp
  └─ RenderBasePass() - BasePass 渲染入口

Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.cpp
  └─ FDeferredShadingSceneRenderer::RenderBasePass()
```

### BasePass 着色器
```
Engine/Shaders/Private/BasePassPixelShader.usf
  └─ BasePass 像素着色器

Engine/Shaders/Private/BasePassVertexShader.usf
  └─ BasePass 顶点着色器
```

## 光照系统

### 延迟光照
```
Engine/Source/Runtime/Renderer/Private/LightRendering.h
  └─ 光照渲染核心

Engine/Source/Runtime/Renderer/Private/DeferredLightingCommon.h
  └─ 延迟光照共享代码
```

### 光照着色器
```
Engine/Shaders/Private/DeferredLightingCommon.ush
  └─ 延迟光照 HLSL 函数

Engine/Shaders/Private/DeferredLightPixelShaders.usf
  └─ 延迟光照像素着色器
```

## 阴影系统

### 阴影贴图
```
Engine/Source/Runtime/Renderer/Private/ShadowRendering.h
  └─ FShadowDepthDrawingPolicy

Engine/Source/Runtime/Renderer/Private/ShadowRendering.cpp
  └─ 阴影深度渲染实现
```

### 虚拟阴影贴图（VSM）
```
Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/VirtualShadowMapArray.h
  └─ FVirtualShadowMapArray - VSM 管理器

Engine/Source/Runtime/Renderer/Private/VirtualShadowMaps/VirtualShadowMapClipmap.cpp
  └─ VSM 裁剪贴图实现
```

## 后处理

### 后处理管线
```
Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessing.h
  └─ 后处理系统

Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessing.cpp
  └─ AddPostProcessingPasses() - 添加后处理 pass
```

### 色调映射
```
Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessTonemap.cpp
  └─ 色调映射实现

Engine/Shaders/Private/PostProcessTonemap.usf
  └─ 色调映射着色器
```

## Lumen

### Lumen 核心
```
Engine/Source/Runtime/Renderer/Private/Lumen/Lumen.h
  └─ Lumen 全局光照系统

Engine/Source/Runtime/Renderer/Private/Lumen/LumenSceneData.h
  └─ FLumenSceneData - Lumen 场景数据
```

### Surface Cache
```
Engine/Source/Runtime/Renderer/Private/Lumen/LumenSurfaceCache.cpp
  └─ Lumen 表面缓存管理

Engine/Shaders/Private/Lumen/LumenSurfaceCacheFeedback.usf
  └─ Surface cache 着色器
```

### Radiance Cache
```
Engine/Source/Runtime/Renderer/Private/Lumen/LumenRadianceCache.cpp
  └─ 辐射度缓存实现

Engine/Shaders/Private/Lumen/LumenRadianceCache.usf
  └─ Radiance cache 着色器
```

## Nanite

### Nanite 核心
```
Engine/Source/Runtime/Renderer/Private/Nanite/Nanite.h
  └─ Nanite 虚拟化几何系统

Engine/Source/Runtime/Renderer/Private/Nanite/NaniteRender.cpp
  └─ Nanite 渲染实现
```

### 光栅化
```
Engine/Source/Runtime/Renderer/Private/Nanite/NaniteRasterizer.cpp
  └─ Nanite 软件光栅化器

Engine/Shaders/Private/Nanite/NaniteRasterizer.usf
  └─ Nanite 光栅化着色器
```

### 流式加载
```
Engine/Source/Runtime/Engine/Private/Rendering/NaniteStreaming.cpp
  └─ Nanite 流式加载管理
```

## TSR（时间超分辨率）

### TSR 实现
```
Engine/Source/Runtime/Renderer/Private/TemporalSuperResolution.cpp
  └─ 时间超分辨率实现

Engine/Shaders/Private/TemporalSuperResolution.usf
  └─ TSR 着色器
```

### 历史重投影
```
Engine/Source/Runtime/Renderer/Private/PostProcess/PostProcessTemporalCommon.h
  └─ 时间抗锯齿共享代码
```

## 渲染器核心

### 场景渲染器
```
Engine/Source/Runtime/Renderer/Private/SceneRendering.h
  └─ FSceneRenderer - 场景渲染器基类

Engine/Source/Runtime/Renderer/Private/DeferredShadingRenderer.h
  └─ FDeferredShadingSceneRenderer - 延迟着色渲染器
```

### 渲染线程
```
Engine/Source/Runtime/RenderCore/Public/RenderingThread.h
  └─ 渲染线程管理

Engine/Source/Runtime/RenderCore/Private/RenderingThread.cpp
  └─ ENQUEUE_RENDER_COMMAND() - 渲染命令宏
```

## 常用工具类

### 数学
```
Engine/Source/Runtime/Core/Public/Math/Vector.h
  └─ FVector, FVector2D, FVector4

Engine/Source/Runtime/Core/Public/Math/Matrix.h
  └─ FMatrix

Engine/Source/Runtime/Core/Public/Math/Color.h
  └─ FLinearColor, FColor
```

### RHI（渲染硬件接口）
```
Engine/Source/Runtime/RHI/Public/RHI.h
  └─ RHI 主头文件

Engine/Source/Runtime/RHI/Public/RHICommandList.h
  └─ FRHICommandList - GPU 命令列表
```

---

*本参考路径基于 UE5.2+。随着引擎版本更新，路径可能会变化。*
