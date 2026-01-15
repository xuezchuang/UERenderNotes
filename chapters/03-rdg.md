# Chapter 3: Render Dependency Graph (RDG)

## Goals

By the end of this chapter, you will understand:
- What RDG is and why UE5 uses it
- How RDG automatically manages GPU resources and barriers
- How to read and write RDG passes
- RDG resource lifetime and aliasing
- Debugging RDG-based rendering code

## Overview

The Render Dependency Graph (RDG) is UE5's modern rendering framework, introduced to replace manual resource management. RDG automatically handles resource allocation, transitions, and synchronization, making rendering code safer and more efficient.

**Key benefits**:
- Automatic memory aliasing (reuse memory for transient resources)
- Automatic barrier insertion (resource state transitions)
- Optimized GPU workload scheduling
- Built-in visualization and debugging tools

## Pipeline Steps

### 1. Graph Construction (Setup Phase)
**What happens**: Declare all rendering passes and their resource dependencies.

**During this phase**:
- Create RDG resources (textures, buffers)
- Register render passes
- Declare pass inputs/outputs
- No GPU work executed yet

### 2. Dependency Analysis (Compilation Phase)
**What happens**: RDG analyzes the graph to optimize execution.

**Analysis includes**:
- Build dependency chains
- Cull unused passes (if output isn't consumed)
- Determine resource lifetimes
- Plan memory aliasing opportunities

### 3. Resource Allocation
**What happens**: Allocate transient GPU memory for resources.

**Strategy**:
- Pool-based allocation for frequently-used sizes
- Memory aliasing: reuse memory when resource lifetimes don't overlap
- Persistent resources allocated separately

### 4. Barrier Insertion
**What happens**: Insert GPU synchronization barriers automatically.

**Barriers inserted for**:
- Resource state transitions (e.g., RenderTarget → ShaderResource)
- Memory dependencies between passes
- Cross-queue synchronization (graphics, compute, copy)

### 5. Execution
**What happens**: Execute passes in dependency order on GPU.

**Execution respects**:
- Producer-consumer relationships
- Resource availability
- GPU queue capabilities

### 6. Resource Release
**What happens**: Release transient resources back to pool.

**After execution**:
- Transient resources returned to pool
- Persistent resources kept alive
- Ready for next frame's graph

## Key UE Source Files, Classes, and Functions

### Core RDG Classes

**`Engine/Source/Runtime/RenderCore/Public/RenderGraphResources.h`**
- `FRDGResource` - Base class for all RDG resources
- `FRDGTexture` - RDG texture handle
- `FRDGBuffer` - RDG buffer handle
- `FRDGTextureRef`, `FRDGBufferRef` - Reference-counted handles

**`Engine/Source/Runtime/RenderCore/Public/RenderGraphBuilder.h`**
- `FRDGBuilder` - Main interface for building render graphs
- `AddPass()` - Register a render/compute pass
- `CreateTexture()`, `CreateBuffer()` - Create RDG resources
- `Execute()` - Execute the graph

**`Engine/Source/Runtime/RenderCore/Public/RenderGraphPass.h`**
- `FRDGPass` - Base class for render graph passes
- Pass parameter structures
- Lambda-based pass execution

### Resource Descriptors

**`Engine/Source/Runtime/RenderCore/Public/RenderGraphResources.h`**
- `FRDGTextureDesc` - Describes texture properties (size, format, flags)
- `FRDGBufferDesc` - Describes buffer properties (size, stride, usage)

### Resource Access

**`Engine/Source/Runtime/RenderCore/Public/RenderGraphParameters.h`**
- `RDG_TEXTURE_ACCESS()` - Macro for texture parameters
- `RDG_BUFFER_ACCESS()` - Macro for buffer parameters
- Access modes: Read, Write, ReadWrite

### Key Functions

```cpp
// RenderGraphBuilder.h
FRDGTextureRef FRDGBuilder::CreateTexture(
    const FRDGTextureDesc& Desc,
    const TCHAR* Name,
    ERDGTextureFlags Flags = ERDGTextureFlags::None);

FRDGBufferRef FRDGBuilder::CreateBuffer(
    const FRDGBufferDesc& Desc,
    const TCHAR* Name,
    ERDGBufferFlags Flags = ERDGBufferFlags::None);

template<typename ParameterStructType, typename ExecuteLambdaType>
FRDGPassRef FRDGBuilder::AddPass(
    FRDGEventName&& Name,
    ParameterStructType* Parameters,
    ERDGPassFlags Flags,
    ExecuteLambdaType&& ExecuteLambda);

void FRDGBuilder::Execute();

// Resource extraction (to use outside RDG)
TRefCountPtr<IPooledRenderTarget> FRDGBuilder::ConvertToExternalTexture(
    FRDGTextureRef Texture);
```

## RDG Resource Types

### Textures

**Create texture**:
```cpp
FRDGTextureDesc Desc = FRDGTextureDesc::Create2D(
    Extent,                          // FIntPoint size
    PF_FloatRGBA,                   // Pixel format
    FClearValueBinding::Black,      // Clear value
    TexCreate_ShaderResource | TexCreate_RenderTargetable);

FRDGTextureRef MyTexture = GraphBuilder.CreateTexture(Desc, TEXT("MyTexture"));
```

**Texture flags**:
- `TexCreate_RenderTargetable` - Can be used as render target
- `TexCreate_ShaderResource` - Can be sampled in shaders
- `TexCreate_UAV` - Can be used as unordered access view
- `TexCreate_Transient` - Hint that resource is short-lived

### Buffers

**Create buffer**:
```cpp
FRDGBufferDesc Desc = FRDGBufferDesc::CreateStructuredDesc(
    sizeof(FMyStruct),   // Stride
    NumElements);        // Element count

FRDGBufferRef MyBuffer = GraphBuilder.CreateBuffer(Desc, TEXT("MyBuffer"));
```

**Buffer types**:
- **Structured buffer**: Fixed-stride elements
- **Byte address buffer**: Raw byte buffer
- **Vertex buffer**: Vertex data
- **Index buffer**: Index data
- **Indirect args**: GPU-driven draw/dispatch arguments

### External Resources

**Register external texture**:
```cpp
// Bring external texture into RDG
FRDGTextureRef ExternalTexture = GraphBuilder.RegisterExternalTexture(
    ExternalPooledTexture,
    TEXT("ExternalTexture"));
```

**Extract texture from RDG**:
```cpp
// Keep texture alive after graph execution
TRefCountPtr<IPooledRenderTarget> OutputTexture;
GraphBuilder.QueueTextureExtraction(RDGTexture, &OutputTexture);
```

## RDG Pass Example

### Simple Render Pass

```cpp
// 1. Define pass parameters
BEGIN_SHADER_PARAMETER_STRUCT(FMyPassParameters, )
    // Input texture
    RDG_TEXTURE_ACCESS(InputTexture, ERHIAccess::SRVGraphics)
    // Output render target
    RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()

// 2. Allocate parameters on stack
FMyPassParameters* PassParameters = GraphBuilder.AllocParameters<FMyPassParameters>();

// 3. Setup parameters
PassParameters->InputTexture = InputTexture;
PassParameters->RenderTargets[0] = FRenderTargetBinding(
    OutputTexture,
    ERenderTargetLoadAction::EClear);

// 4. Add pass to graph
GraphBuilder.AddPass(
    RDG_EVENT_NAME("MyRenderPass"),
    PassParameters,
    ERDGPassFlags::Raster,
    [PassParameters, MyShader](FRHICommandList& RHICmdList)
    {
        // Setup viewport
        RHICmdList.SetViewport(0, 0, 0, Width, Height, 1);
        
        // Set shader
        FGraphicsPipelineStateInitializer GraphicsPSOInit;
        RHICmdList.ApplyCachedRenderTargets(GraphicsPSOInit);
        GraphicsPSOInit.BlendState = TStaticBlendState<>::GetRHI();
        GraphicsPSOInit.RasterizerState = TStaticRasterizerState<>::GetRHI();
        GraphicsPSOInit.DepthStencilState = TStaticDepthStencilState<false, CF_Always>::GetRHI();
        GraphicsPSOInit.BoundShaderState.VertexDeclarationRHI = GFilterVertexDeclaration.VertexDeclarationRHI;
        GraphicsPSOInit.BoundShaderState.VertexShaderRHI = MyShader.GetVertexShader();
        GraphicsPSOInit.BoundShaderState.PixelShaderRHI = MyShader.GetPixelShader();
        GraphicsPSOInit.PrimitiveType = PT_TriangleList;
        SetGraphicsPipelineState(RHICmdList, GraphicsPSOInit);
        
        // Set shader parameters
        SetShaderParameters(RHICmdList, MyShader, PassParameters);
        
        // Draw
        DrawFullscreenQuad(RHICmdList);
    });
```

### Simple Compute Pass

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyComputeParameters, )
    RDG_BUFFER_ACCESS(InputBuffer, ERHIAccess::SRVCompute)
    RDG_BUFFER_ACCESS(OutputBuffer, ERHIAccess::UAVCompute)
    SHADER_PARAMETER(uint32, NumElements)
END_SHADER_PARAMETER_STRUCT()

FMyComputeParameters* PassParameters = GraphBuilder.AllocParameters<FMyComputeParameters>();
PassParameters->InputBuffer = InputBuffer;
PassParameters->OutputBuffer = OutputBuffer;
PassParameters->NumElements = NumElements;

GraphBuilder.AddPass(
    RDG_EVENT_NAME("MyComputePass"),
    PassParameters,
    ERDGPassFlags::Compute,
    [PassParameters, ComputeShader, NumElements](FRHICommandList& RHICmdList)
    {
        SetComputePipelineState(RHICmdList, ComputeShader.GetComputeShader());
        SetShaderParameters(RHICmdList, ComputeShader, PassParameters);
        
        // Dispatch compute shader
        uint32 GroupSize = 64;
        uint32 NumGroups = FMath::DivideAndRoundUp(NumElements, GroupSize);
        DispatchComputeShader(RHICmdList, ComputeShader, NumGroups, 1, 1);
    });
```

## Resource Lifetime and Aliasing

### Transient Resources

**Lifetime**:
- Exist only during graph execution
- Automatically released after last use
- Memory is aliased with other transient resources

**Example**:
```cpp
// This texture only exists during graph execution
FRDGTextureRef TempTexture = GraphBuilder.CreateTexture(Desc, TEXT("TempTexture"));
```

### Extracted Resources

**Lifetime**:
- Persist beyond graph execution
- Must be explicitly extracted
- Not eligible for aliasing

**Example**:
```cpp
TRefCountPtr<IPooledRenderTarget> PersistentTexture;
GraphBuilder.QueueTextureExtraction(RDGTexture, &PersistentTexture);
// PersistentTexture remains valid after Execute()
```

### Memory Aliasing

RDG can reuse memory when resource lifetimes don't overlap:

```cpp
// Pass 1: Write to TextureA
AddPassThatWritesTextureA();

// Pass 2: Read TextureA, write TextureB
AddPassThatReadsAWritesB();

// Pass 3: Read TextureB
AddPassThatReadsB();

// TextureA and TextureB can share memory!
// TextureA not needed after Pass 2, so its memory can be reused for TextureB
```

### Aliasing Benefits

- Reduces peak memory usage
- Especially helpful for large transient buffers/textures
- Automatic - no manual management needed

**Disable aliasing** (for debugging):
```cpp
// Create texture without aliasing
FRDGTextureRef NoAlias = GraphBuilder.CreateTexture(
    Desc, 
    TEXT("NoAlias"),
    ERDGTextureFlags::SkipTracking);
```

## Pass Flags

### ERDGPassFlags

**Raster passes**:
```cpp
ERDGPassFlags::Raster  // Render pass with render targets
```

**Compute passes**:
```cpp
ERDGPassFlags::Compute  // Compute shader dispatch
```

**Copy passes**:
```cpp
ERDGPassFlags::Copy  // Resource copy operations
```

**Async compute**:
```cpp
ERDGPassFlags::AsyncCompute  // Run on async compute queue
```

**Flags combination**:
```cpp
ERDGPassFlags::Compute | ERDGPassFlags::NeverCull  // Compute pass that's never culled
```

### Pass Culling

RDG automatically culls passes whose outputs aren't used:

```cpp
// This pass will be culled if OutputTexture is never read
GraphBuilder.AddPass(...);

// Prevent culling
GraphBuilder.AddPass(
    ...,
    ERDGPassFlags::Compute | ERDGPassFlags::NeverCull,
    ...);
```

## Debugging Tips

### 1. Enable RDG Debugging

**Console variables**:
```
r.RDG.Debug=1                    // Enable RDG debugging
r.RDG.Debug.FlushGPU=1          // Flush after each pass (slow!)
r.RDG.Debug.VerifyTransientResources=1  // Verify resource states
```

### 2. Visualize RDG

**Console command**:
```
r.RDG.DumpGraph=1   // Dump graph to log
```

**Output shows**:
- All passes and their dependencies
- Resource creation/destruction points
- Memory aliasing information

### 3. RDG Insights

**Enable GPU profiling**:
```
profilegpu  // Shows RDG passes in GPU profiler
```

**Pass names** appear in profiler (set via `RDG_EVENT_NAME()`).

### 4. Check Resource Names

Always name your resources:
```cpp
// Good
FRDGTextureRef MyTexture = GraphBuilder.CreateTexture(Desc, TEXT("MyTexture"));

// Bad (hard to debug)
FRDGTextureRef MyTexture = GraphBuilder.CreateTexture(Desc, TEXT(""));
```

Names appear in:
- GPU profiler
- Graphics debugger (RenderDoc, PIX)
- RDG dump output

### 5. Validate Resource Access

**Problem**: Accessing resource in wrong pass
```cpp
// BAD: InputTexture not declared in pass parameters
GraphBuilder.AddPass(..., [InputTexture](...) {
    // Using InputTexture here is WRONG
});

// GOOD: Declare in parameters
BEGIN_SHADER_PARAMETER_STRUCT(FParams, )
    RDG_TEXTURE_ACCESS(InputTexture, ERHIAccess::SRVGraphics)
END_SHADER_PARAMETER_STRUCT()
```

### 6. Common Errors

**"Resource used but not declared"**:
- Add resource to pass parameter struct
- Use `RDG_TEXTURE_ACCESS()` or `RDG_BUFFER_ACCESS()` macros

**"Resource accessed after being released"**:
- Check resource lifetime
- Extract resource if needed beyond graph execution

**"Circular dependency detected"**:
- Pass A depends on B, B depends on A
- Refactor to break cycle

### 7. RenderDoc/PIX Integration

RDG pass names automatically appear in graphics debuggers:
- Open capture in RenderDoc
- Event browser shows RDG pass hierarchy
- Resource viewer shows RDG resource names

## Advanced Topics

### Async Compute

Run compute work on dedicated queue:
```cpp
GraphBuilder.AddPass(
    RDG_EVENT_NAME("AsyncComputePass"),
    PassParameters,
    ERDGPassFlags::AsyncCompute,  // Use async compute queue
    ExecuteLambda);
```

**Benefits**:
- Overlaps with graphics work
- Better GPU utilization
- Faster frame times

**Considerations**:
- Not all GPUs have async compute
- RDG handles synchronization automatically

### Pooled Render Targets

Efficiently reuse render targets across frames:
```cpp
// Create pooled render target
TRefCountPtr<IPooledRenderTarget> PooledRT;
GRenderTargetPool.FindFreeElement(
    RHICmdList,
    Desc,
    PooledRT,
    TEXT("MyPooledRT"));

// Register with RDG
FRDGTextureRef RDGTexture = GraphBuilder.RegisterExternalTexture(PooledRT);
```

### Subresource Tracking

RDG tracks individual mip levels and array slices:
```cpp
// Access specific mip level
FRDGTextureSRVDesc SRVDesc = FRDGTextureSRVDesc::CreateForMipLevel(Texture, MipLevel);
FRDGTextureSRVRef SRV = GraphBuilder.CreateSRV(SRVDesc);
```

### Custom Barriers

Manually insert barriers when needed:
```cpp
AddPass(GraphBuilder, RDG_EVENT_NAME("Barrier"), [Buffer](FRHICommandList& RHICmdList)
{
    FRHITransitionInfo Transition(
        Buffer->GetRHI(),
        ERHIAccess::UAVCompute,
        ERHIAccess::SRVGraphics);
    RHICmdList.Transition(Transition);
});
```

## Best Practices

### 1. Name Everything
```cpp
// Resources, passes, events - always use descriptive names
FRDGTextureRef SceneColor = GraphBuilder.CreateTexture(Desc, TEXT("SceneColor"));
GraphBuilder.AddPass(RDG_EVENT_NAME("BasePass"), ...);
```

### 2. Declare All Dependencies
```cpp
// Include all resources used in pass in parameter struct
BEGIN_SHADER_PARAMETER_STRUCT(FParams, )
    RDG_TEXTURE_ACCESS(InputA, ERHIAccess::SRVGraphics)
    RDG_TEXTURE_ACCESS(InputB, ERHIAccess::SRVGraphics)
    RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
```

### 3. Prefer Transient Resources
```cpp
// Let RDG manage lifetime and aliasing
FRDGTextureRef Temp = GraphBuilder.CreateTexture(Desc, TEXT("Temp"));
// Don't extract unless you need it later
```

### 4. Use Pooled Resources for Cross-Frame Data
```cpp
// For resources that persist across frames
TRefCountPtr<IPooledRenderTarget> HistoryBuffer;
// Re-register each frame
FRDGTextureRef History = GraphBuilder.RegisterExternalTexture(HistoryBuffer);
```

### 5. Avoid Manual Synchronization
```cpp
// BAD: Manual barriers
RHICmdList.Transition(...);

// GOOD: Let RDG handle it
// Just declare access in pass parameters
```

## Performance Considerations

### Memory Usage

**RDG reduces memory**:
- Aliasing reuses memory (30-50% reduction typical)
- Transient allocator pools memory efficiently
- No fragmentation from manual management

**Monitor usage**:
```
stat RHI  // Shows GPU memory stats
```

### Pass Overhead

**Minimal overhead**:
- Lambda capture is lightweight
- Graph compilation is fast (< 1ms typically)
- Execution overhead negligible

**When overhead matters**:
- Thousands of tiny passes
- Consider merging very small passes

### Barriers and Transitions

**RDG optimizes barriers**:
- Batches barriers together
- Eliminates redundant transitions
- Splits barriers when possible (async)

**Better than manual**:
- Manual code often over-synchronizes
- RDG has global view for optimization

## Summary

RDG provides:
1. **Automatic resource management** - No manual allocation/deallocation
2. **Automatic synchronization** - No manual barriers
3. **Memory efficiency** - Automatic aliasing
4. **Debugging tools** - Graph dumps, visualization
5. **Safety** - Compile-time and runtime validation

Key classes: `FRDGBuilder`, `FRDGTexture`, `FRDGBuffer`, `FRDGPass`

RDG is now the standard way to write rendering code in UE5. All major rendering features use it.

## Next Chapter

[Chapter 4: BasePass Rendering](04-basepass.md) - Learn how UE5 renders the GBuffer and evaluates materials.
