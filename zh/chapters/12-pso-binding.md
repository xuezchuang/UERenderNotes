# PSO和RootSignature的绑定

## 概述

PSO（Pipeline State Object）是GPU渲染管线的核心配置对象，包含了着色器、混合状态、光栅化状态等所有渲染状态。在Unreal Engine中，PSO和RootSignature的创建、绑定过程涉及多个层次的抽象，体现了UE RHI的设计哲学。本章详细剖析PSO的绑定过程和RootSignature的创建机制。

## UE中的PSO架构设计

### 结构体层次关系

UE采用**双层继承+共享数据类**的设计模式，体现了引擎的RHI抽象风格：

```cpp
// 顶层RHI接口（平台无关）
class FRHIGraphicsPipelineState : public FRHIResource
{
    // 平台无关的PSO接口
};

// 通用平台接口
class FGraphicsPipelineState : public FPipelineState
{
    TRefCountPtr<FRHIGraphicsPipelineState> RHIPipeline;  // 持有具体平台实现
};

// D3D12平台相关的PSO对象
struct FD3D12PipelineState 
{
    TRefCountPtr<ID3D12PipelineState> PipelineState;  // Direct3D12的原生PSO
};

// D3D12的PSO公共数据（RootSignature等）
struct FD3D12PipelineStateCommonData
{
    const FD3D12RootSignature* const RootSignature;  // RootSignature绑定
    FD3D12PipelineState* PipelineState;              // PSO绑定
};

// D3D12的最终实现类
struct FD3D12GraphicsPipelineState : public FRHIGraphicsPipelineState, FD3D12PipelineStateCommonData
{
    // 继承两个父类：
    // 1. FRHIGraphicsPipelineState - 提供RHI接口
    // 2. FD3D12PipelineStateCommonData - 提供D3D12具体实现数据
};
```

### 设计模式分析

这个架构体现了**接口分离 + 实现聚合**的设计思想：

| 层级 | 职责 | 特点 |
|------|------|------|
| `FRHIGraphicsPipelineState` | 定义RHI接口规范 | 平台无关，抽象 |
| `FGraphicsPipelineState` | 通用的PSO容器 | 持有RHI实现指针 |
| `FD3D12PipelineStateCommonData` | 聚合D3D12相关数据 | 平台相关，可复用 |
| `FD3D12GraphicsPipelineState` | 最终实现类 | 同时满足两个接口 |

**关键设计特点**：
- **分离性**：`FD3D12PipelineStateCommonData`作为独立的数据聚合类，不参与接口继承
- **复用性**：共享数据类可被多个不相关的类组合使用
- **解耦性**：上层代码通过`FGraphicsPipelineState`持有`FRHIGraphicsPipelineState`指针，不直接依赖D3D12实现

## PSO的绑定流程

### 1. 顶层API调用

从RenderThread发起PSO绑定：

```cpp
void SetGraphicsPipelineState(FGraphicsMinimalPipelineStateInitializer& PipelineState, ...)
{
    // 第一步：从缓存中获取或创建PSO
    FGraphicsPipelineState* GraphicsPipelineState = 
        PipelineStateCache::GetAndOrCreateGraphicsPipelineState(
            PipelineState, 
            // ... 其他参数
        );
    
    // 第二步：提交PSO绑定命令到RHICmdList
    RHICmdList.SetGraphicsPipelineState(GraphicsPipelineState, ...);
}
```

**流程分析**：
1. 根据Shader、渲染状态等信息计算PSO的哈希值
2. 在缓存中查询是否已存在该PSO
3. 如果不存在，创建新的PSO对象
4. 将PSO绑定命令加入命令列表

### 2. 命令提交和执行

命令列表在RHIThread中执行：

```cpp
// RHIThread执行
void FD3D12StateCache::SetGraphicsPipelineState(FD3D12GraphicsPipelineState* GraphicsPipelineState)
{
    // 更新缓存中的当前PSO
    PipelineState.Graphics.CurrentPipelineStateObject = GraphicsPipelineState;
    
    // 提交到D3D12
    InternalSetPipelineState(GraphicsPipelineState->PipelineState);
}

void FD3D12StateCache::InternalSetPipelineState(FD3D12PipelineState* PipelineState)
{
    // 调用D3D12原生API
    GetCommandList()->SetPipelineState(PipelineState->PipelineState.GetReference());
}
```

**执行步骤**：
1. 保存PSO到状态缓存
2. 获取FD3D12GraphicsPipelineState中的FD3D12PipelineState
3. 从FD3D12PipelineState中提取ID3D12PipelineState指针
4. 调用D3D12 CommandList的SetPipelineState方法

## RootSignature的创建

### RootSignature的作用

RootSignature定义了着色器访问资源的方式，包括：
- CBV（常量缓冲视图）的位置
- SRV（着色资源视图）的位置
- UAV（无序访问视图）的位置
- 采样器的配置

RootSignature**必须在创建PSO之前创建**，D3D12的PSO构造时需要绑定RootSignature。

### RootSignature的获取流程

```cpp
const FD3D12RootSignature* FD3D12Adapter::GetRootSignature(const FBoundShaderStateInput& BSS)
{
    #if USE_STATIC_ROOT_SIGNATURE
        // 方案A：使用全局静态RootSignature
        return &StaticGraphicsRootSignature;
    
    #else
        // 方案B：根据Shader进行量化和动态创建
        
        // 第一步：创建量化的BSS对象
        FD3D12QuantizedBoundShaderState QBSS{};
        
        // 第二步：检查是否需要输入布局（IAInputLayout）
        QBSS.bAllowIAInputLayout = (BSS.VertexDeclarationRHI != nullptr);
        
        // 第三步：获取硬件的资源绑定等级
        const D3D12_RESOURCE_BINDING_TIER ResourceBindingTier = GetResourceBindingTier();
        
        // 第四步：对各个着色阶段进行量化处理
        QuantizeBoundShaderStateCommon(QBSS, 
            FD3D12DynamicRHI::ResourceCast(BSS.GetVertexShader()),
            ResourceBindingTier, 
            SV_Vertex);
        
        #if PLATFORM_SUPPORTS_MESH_SHADERS
        QuantizeBoundShaderStateCommon(QBSS,
            FD3D12DynamicRHI::ResourceCast(BSS.GetMeshShader()),
            ResourceBindingTier,
            SV_Mesh);
        
        QuantizeBoundShaderStateCommon(QBSS,
            FD3D12DynamicRHI::ResourceCast(BSS.GetAmplificationShader()),
            ResourceBindingTier,
            SV_Amplification);
        #endif
        
        QuantizeBoundShaderStateCommon(QBSS,
            FD3D12DynamicRHI::ResourceCast(BSS.GetPixelShader()),
            ResourceBindingTier,
            SV_Pixel,
            true /*bAllowUAVs*/);
        
        QuantizeBoundShaderStateCommon(QBSS,
            FD3D12DynamicRHI::ResourceCast(BSS.GetGeometryShader()),
            ResourceBindingTier,
            SV_Geometry);
        
        // 第五步：验证BindlessRendering的兼容性（可选）
        #if DO_CHECK && PLATFORM_SUPPORTS_BINDLESS_RENDERING
        if (QBSS.bUseDirectlyIndexedResourceHeap || QBSS.bUseDirectlyIndexedSamplerHeap)
        {
            // 验证所有Shader都支持Bindless
            struct FGenericShaderPair
            {
                const FD3D12ShaderData* Data;
                const FRHIGraphicsShader* RHI;
            };
            
            const FGenericShaderPair ShaderDatas[] =
            {
                { FD3D12DynamicRHI::ResourceCast(BSS.GetVertexShader()), BSS.GetVertexShader() },
                #if PLATFORM_SUPPORTS_MESH_SHADERS
                { FD3D12DynamicRHI::ResourceCast(BSS.GetMeshShader()), BSS.GetMeshShader() },
                { FD3D12DynamicRHI::ResourceCast(BSS.GetAmplificationShader()), BSS.GetAmplificationShader() },
                #endif
                { FD3D12DynamicRHI::ResourceCast(BSS.GetPixelShader()), BSS.GetPixelShader() },
                { FD3D12DynamicRHI::ResourceCast(BSS.GetGeometryShader()), BSS.GetGeometryShader() },
            };
            
            for (const FGenericShaderPair& ShaderPair : ShaderDatas)
            {
                if (ShaderPair.RHI)
                {
                    if (QBSS.bUseDirectlyIndexedResourceHeap)
                    {
                        checkf(IsCompatibleWithBindlessResources(ShaderPair.Data),
                            TEXT("Mismatched dynamic resource usage. %s doesn't support binding"),
                            ShaderPair.RHI->GetShaderName());
                    }
                    
                    if (QBSS.bUseDirectlyIndexedSamplerHeap)
                    {
                        checkf(IsCompatibleWithBindlessSamplers(ShaderPair.Data),
                            TEXT("Mismatched dynamic resource usage. %s doesn't support binding"),
                            ShaderPair.RHI->GetShaderName());
                    }
                }
            }
        }
        #endif
        
        // 第六步：从缓存获取或创建RootSignature
        return RootSignatureManager.GetRootSignature(QBSS);
    
    #endif
}
```

### RootSignature创建的关键概念

#### 1. 两种策略

| 策略 | 特点 | 适用场景 |
|------|------|---------|
| **USE_STATIC_ROOT_SIGNATURE** | 使用全局统一的RootSignature | 简单场景，兼容性好 |
| **动态创建** | 根据Shader资源需求创建 | 复杂场景，灵活性高 |

#### 2. 量化（Quantization）机制

**目的**：将不同的BoundShaderState映射到同一个RootSignature

**工作原理**：
- 不记录具体的资源绑定细节
- 只记录关键特征（资源类型、数量、阶段）
- 多个接近的Shader组合可以复用同一个RootSignature
- 减少创建的RootSignature数量，提高效率

```cpp
// 量化的特征示例
struct FD3D12QuantizedBoundShaderState
{
    bool bAllowIAInputLayout;              // 是否需要顶点输入
    uint32 MaxBoundResourceCBVs;           // 最大CBV数量
    uint32 MaxBoundResourceSRVs;           // 最大SRV数量
    uint32 MaxBoundResourceUAVs;           // 最大UAV数量
    bool bUseDirectlyIndexedResourceHeap;  // 是否使用Bindless资源堆
    bool bUseDirectlyIndexedSamplerHeap;   // 是否使用Bindless采样器堆
    // ... 其他量化特征
};
```

#### 3. 资源绑定等级（Resource Binding Tier）

D3D12支持三个资源绑定等级，影响RootSignature的生成：

```
Tier 1: 基础支持，受限的资源绑定
Tier 2: 改进的绑定灵活性
Tier 3: 完全的动态索引支持（Bindless）
```

不同的GPU支持不同的等级，RootSignature创建时需要适配硬件能力。

#### 4. Bindless兼容性检查

使用Bindless渲染时，所有Shader必须都支持：
- **Bindless Resources**：动态索引资源堆
- **Bindless Samplers**：动态索引采样器堆

否则会导致运行时错误。

## 完整的绑定时序

```
RenderThread
    ↓
SetGraphicsPipelineState()
    ↓
PipelineStateCache::GetAndOrCreateGraphicsPipelineState()
    ├─ 计算PSO哈希值
    ├─ 查询缓存
    └─ 如果不存在：创建新PSO
        ├─ GetRootSignature() - 创建或获取RootSignature
        │  ├─ 对所有Shader进行量化
        │  ├─ 验证Bindless兼容性
        │  └─ 从RootSignatureManager获取/创建
        └─ 创建ID3D12PipelineState
    ↓
RHICmdList.SetGraphicsPipelineState(PipelineState, ...)
    ↓
RHIThread
    ↓
FD3D12StateCache::SetGraphicsPipelineState()
    ├─ 更新当前PSO缓存
    └─ InternalSetPipelineState()
        ↓
        CommandList→SetPipelineState(ID3D12PipelineState*)
```

## 性能考虑

### PSO缓存策略

```cpp
// PipelineStateCache中的缓存：
// 键：PSO状态的哈希值（Shader + RenderState）
// 值：FGraphicsPipelineState对象

// 优点：避免重复创建相同的PSO
// 成本：缓存查询的哈希计算
```

### RootSignature复用

- **量化机制**降低了需要的RootSignature数量
- **静态RootSignature**进一步提高性能（无动态创建开销）
- **动态RootSignature**权衡灵活性和性能

### 创建时机

- RootSignature创建**延迟到首次使用**时进行
- PSO创建也是**延迟创建**
- 避免在关键渲染路径中创建大量PSO

## 总结

| 概念 | 职责 | 缓存情况 |
|------|------|---------|
| **PSO** | 定义完整的渲染管线状态 | 缓存在PipelineStateCache |
| **RootSignature** | 定义资源访问方式 | 缓存在RootSignatureManager |
| **量化** | 聚合相似的Shader配置 | 减少创建数量 |
| **绑定时序** | RootSignature → PSO → SetPipelineState | 严格依赖顺序 |

UE的PSO和RootSignature设计体现了：
1. **分层抽象**：从通用接口到平台实现的清晰划分
2. **缓存优化**：大量使用缓存减少创建开销
3. **灵活配置**：同时支持静态和动态两种策略
4. **验证机制**：编译时检查确保兼容性

这套系统使得Unreal Engine能够高效地在D3D12上运行复杂的渲染逻辑。
