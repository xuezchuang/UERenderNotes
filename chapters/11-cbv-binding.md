# CBV(Constant Buffer View)绑定机制

## 概述

常量缓冲区视图(Constant Buffer View, CBV)是Direct3D 12中向着色器传递常量数据的核心机制。在渲染管线中，CBV用于存储与渲染无关的常量数据，如变换矩阵、光照参数等。本章节总结D3D12中绑定CBV的各种方式，并对比它们的性能特征和使用场景。

## D3D12中的CBV绑定方式

Direct3D 12提供了三种主要的方式将CBV绑定到图形管线中。这些方式在Root Signature中定义，然后通过命令列表设置。

### 方式1: Root Descriptor - 直接根常量缓冲视图(Root CBV)

根常量缓冲视图直接将CBV放在Root Signature中，无需经过描述符堆。

**Root Signature定义:**
```cpp
// 在Root Signature中定义一个直接的CBV
D3D12_ROOT_PARAMETER rootParameters[1];
rootParameters[0].ParameterType = D3D12_ROOT_PARAMETER_TYPE_CBV;
rootParameters[0].Descriptor.ShaderRegister = 0;  // b0 in HLSL
rootParameters[0].Descriptor.RegisterSpace = 0;
rootParameters[0].ShaderVisibility = D3D12_SHADER_VISIBILITY_ALL;
```

**使用(绑定):**
```cpp
// 在命令列表中设置
commandList->SetGraphicsRootConstantBufferView(
    0,                                      // RootParameterIndex
    constantBuffer->GetGPUVirtualAddress()  // GPU虚拟地址
);
```

**特点:**
- ✅ 开销小，无需描述符堆查询
- ✅ 延迟最低，GPU可直接访问
- ❌ Root Signature空间有限(总计128 DWORDs)
- ❌ 适合频繁更新的常量数据

---

### 方式2: 根常量(Root Constants)

根常量是直接嵌入Root Signature中的32位值的集合，最高效但容量有限。

**Root Signature定义:**
```cpp
// 在Root Signature中定义根常量
D3D12_ROOT_PARAMETER rootParameters[1];
rootParameters[0].ParameterType = D3D12_ROOT_PARAMETER_TYPE_32BIT_CONSTANTS;
rootParameters[0].Constants.ShaderRegister = 0;   // b0 in HLSL
rootParameters[0].Constants.RegisterSpace = 0;
rootParameters[0].Constants.Num32BitValues = 16;  // 16个DWORD = 64字节
rootParameters[0].ShaderVisibility = D3D12_SHADER_VISIBILITY_ALL;
```

**使用(绑定):**
```cpp
// 设置根常量数据
uint32_t constants[4] = { /* data */ };
commandList->SetGraphicsRoot32BitConstants(
    0,              // RootParameterIndex
    4,              // Num32BitValuesToSet
    constants,      // 数据指针
    0               // DestOffsetIn32BitValues
);
```

**特点:**
- ✅ 性能最优，直接CPU->GPU传输
- ✅ 无需缓冲区和描述符
- ❌ 容量有限(总计128 DWORDs限制)
- ❌ 仅适合小数据(通常 < 64字节)

---

### 方式3: 描述符堆中的CBV(Descriptor Heap CBV)

通过描述符堆管理CBV，使用描述符表在Root Signature中引用。

**Root Signature定义:**
```cpp
// 在Root Signature中定义描述符表
D3D12_DESCRIPTOR_RANGE ranges[1];
ranges[0].RangeType = D3D12_DESCRIPTOR_RANGE_TYPE_CBV;
ranges[0].NumDescriptors = 16;              // 最多16个CBV
ranges[0].BaseShaderRegister = 0;           // b0 in HLSL
ranges[0].RegisterSpace = 0;
ranges[0].OffsetInDescriptorsFromTableStart = 0;

D3D12_ROOT_PARAMETER rootParameters[1];
rootParameters[0].ParameterType = D3D12_ROOT_PARAMETER_TYPE_DESCRIPTOR_TABLE;
rootParameters[0].DescriptorTable.NumDescriptorRanges = 1;
rootParameters[0].DescriptorTable.pDescriptorRanges = ranges;
rootParameters[0].ShaderVisibility = D3D12_SHADER_VISIBILITY_ALL;
```

**使用(绑定):**
```cpp
// 第一步: 在堆中创建CBV描述符
D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc = {};
cbvDesc.BufferLocation = constantBuffer->GetGPUVirtualAddress();
cbvDesc.SizeInBytes = constantBufferSize;
device->CreateConstantBufferView(&cbvDesc, descriptorHeapHandle);

// 第二步: 在命令列表中设置描述符表
commandList->SetGraphicsRootDescriptorTable(
    0,                              // RootParameterIndex
    descriptorHeap->GetGPUDescriptorHandleForHeapStart()
);
```

**特点:**
- ✅ 容量大，可管理多个CBV
- ✅ 灵活性高，支持间接寻址
- ✅ 与其他描述符类型兼容
- ❌ 多一次堆查询，延迟稍高
- ❌ 需要管理描述符堆生命周期

---

## D3D12 CBV绑定方式对比表

| 方式 | Root Descriptor | 根常量 | 描述符堆 |
|------|-----------------|--------|---------|
| **初始化** | Root Signature定义 | Root Signature定义 | 堆+描述符定义 |
| **绑定函数** | SetGraphicsRootConstantBufferView | SetGraphicsRoot32BitConstants | SetGraphicsRootDescriptorTable |
| **容量** | 每个受限 | 极小(< 128 DWORDs) | 大 |
| **性能** | 优秀 | 最优 | 良好 |
| **灵活性** | 中等 | 低(仅常量) | 高 |
| **Root Signature开销** | 8 DWORDs | 1 + Num32BitValues DWORDs | 1 DWORDs |
| **使用场景** | 频繁变化的数据 | 小型参数 | 大量/多个缓冲区 |

---

## Unreal Engine的实现策略

### UE的设计选择

Unreal Engine在不同版本和渲染路径中采用了不同的策略:

1. **延迟渲染路径**: 主要使用**描述符堆CBV**方式，支持动态资源绑定和多样化的缓冲区场景
2. **前向渲染路径**: 可能结合使用**根描述符**和**描述符堆**，取决于具体光源数量和场景复杂度
3. **动态缓冲区**: 对于每帧更新的数据(如View/Project矩阵)，采用**根描述符**以获得最低延迟

### UE的根签名设计

UE的根签名通常包含:
- Root Descriptor用于高频变化的数据(View/Project矩阵等)
- 描述符表用于材质资源、纹理采样器等
- 根常量用于小参数(如标志位、计数等)

---

## 后续内容规划

本章将在后续补充以下内容:

1. **UE中CBV的具体设置流程**
   - RootSignature定义方式
   - 常量缓冲区的分配策略
   - 每帧更新的机制

2. **BasePass中的CBV绑定实例**
   - 顶点着色器常量
   - 像素着色器常量
   - 性能优化技巧

3. **动态常量缓冲区管理**
   - 环形缓冲区设计
   - 对齐要求
   - 内存效率

4. **调试与优化**
   - GPU调试工具的使用
   - 常见的绑定错误
   - 性能瓶颈识别

---

## 相关主题

- [BasePass渲染](./04-basepass.md) - 使用CBV的实际应用
- [着色器编译](./02-shader-compilation.md) - HLSL中的常量缓冲区声明
- [RDG系统](./03-rdg.md) - 资源绑定的抽象层

## 参考资源

- Microsoft Learn: [ID3D12GraphicsCommandList::SetGraphicsRootConstantBufferView](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsrootconstantbufferview)
- Microsoft Learn: [ID3D12GraphicsCommandList::SetGraphicsRootDescriptorTable](https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-setgraphicsrootdescriptortable)
- Microsoft Learn: [Root Signatures](https://learn.microsoft.com/en-us/windows/win32/direct3d12/root-signatures)
