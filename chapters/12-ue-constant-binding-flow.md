# UE 常量数据绑定流程（D3D12 / RHIThread）

## 目标

记录 UE 在 D3D12 下将 shader constant 数据从 `RenderThread` 传到 `RHIThread`，并最终绑定到 GPU 的流程。

## 总体结论（先看这个）

- `FD3D12StateCache::SetConstantBuffer` 的核心职责是：把常量数据版本化并拷贝到可供 GPU 访问的显存（常见是 upload/动态分配区域），然后把对应的 GPU 地址写入 `CBVCache`。
- `FD3D12StateCache::ApplyConstants` 的核心职责是：把前面缓存好的 GPU 地址真正绑定到 root parameter（例如 `SetGraphicsRootConstantBufferView`）。
- 因此可以把它理解为：
  - `SetConstantBuffer` = 数据上传 + 缓存状态更新
  - `ApplyConstants` = 实际发 D3D12 绑定命令

## RHIThread 侧调用链（Draw 时）

以图形绘制路径为例：

```cpp
FD3D12CommandContext::RHIDrawPrimitive
    -> FD3D12CommandContext::CommitNonComputeShaderConstants
        -> FD3D12StateCache::SetConstantBuffer
    -> FD3D12StateCache::ApplyState(ED3D12PipelineType::Graphics)
        -> FD3D12StateCache::ApplyConstants
```

## `FD3D12StateCache::SetConstantBuffer` 记录（重点）

你关注的点是对的：这里有资源池/线性分配器式的管理思路，行为上可类比 DirectX-Graphics-Samples 里的 `LinearAllocator`。

### 关键过程（按代码语义拆解）

1. 创建临时 `FD3D12ResourceLocation`
2. 调用 `Buffer.Version(Location, bDiscardSharedConstants)`
3. `Version(...)` 成功时：
   - 将常量数据拷贝到本次版本对应的显存位置（通常是 CPU 可写、GPU 可读的动态区域）
   - 得到该版本的 `GPUVirtualAddress`
4. 把 `GPUVirtualAddress` 写入 `PipelineState.Common.CBVCache`
5. 记录 `ResidencyHandle`
6. 标记 dirty slot，等待 `ApplyConstants` 阶段真正绑定

### 代码片段（你给出的重点）

```cpp
D3D12_STATE_CACHE_INLINE void SetConstantBuffer(
    EShaderFrequency ShaderFrequency,
    FD3D12ConstantBuffer& Buffer,
    bool bDiscardSharedConstants)
{
    FD3D12ResourceLocation Location(GetParentDevice());

    if (Buffer.Version(Location, bDiscardSharedConstants))
    {
        // Note: Code assumes the slot index is always 0.
        const uint32 SlotIndex = 0;

        FD3D12ConstantBufferCache& CBVCache = PipelineState.Common.CBVCache;
        D3D12_GPU_VIRTUAL_ADDRESS& CurrentGPUVirtualAddress =
            CBVCache.CurrentGPUVirtualAddress[ShaderFrequency][SlotIndex];
        check(Location.GetGPUVirtualAddress() != CurrentGPUVirtualAddress);
        CurrentGPUVirtualAddress = Location.GetGPUVirtualAddress();
        CBVCache.ResidencyHandles[ShaderFrequency][SlotIndex] =
            &Location.GetResource()->GetResidencyHandle();
        FD3D12ConstantBufferCache::DirtySlot(
            CBVCache.DirtySlotMask[ShaderFrequency], SlotIndex);

#if USE_STATIC_ROOT_SIGNATURE
        CBVCache.CBHandles[ShaderFrequency][SlotIndex] = Buffer.GetOfflineCpuHandle();
#endif
    }
}
```

### 这一层的理解（结论）

- `Version(...)` 可以理解为“生成/获取当前常量数据的一个可绑定版本”。
- 这个版本对应一段真实的 GPU 可访问内存位置（`Location`）。
- `SetConstantBuffer` 并没有直接调用 `SetGraphicsRootConstantBufferView`。
- 它只是把“待绑定的 GPU 地址”准备好并缓存到 `CBVCache`。

## `FD3D12StateCache::ApplyConstants`

这一阶段才会把 `CBVCache` 里的地址绑定到命令列表。

- 本质是绑定 GPU 地址（Root CBV）
- 对应 D3D12 调用可理解为：

```cpp
ID3D12GraphicsCommandList::SetGraphicsRootConstantBufferView(...)
```

也就是说：

- `SetConstantBuffer` 解决“数据放哪”
- `ApplyConstants` 解决“把哪个地址绑定到哪个 root slot”

## 线程拆分：RenderThread vs RHIThread

上面 `SetConstantBuffer / ApplyConstants` 是在 **RHIThread** 执行的。

在 **RenderThread**，主要发生的是“参数收集与入队”。

### RenderThread：参数写入批处理器

典型入口：

```cpp
VertexShader->SetParameters(...)
```

这里会把变量写入/拷贝到 `FRHIParameterBatcher`（批处理缓存）中。

### RenderThread：Draw 前把 batched 参数变成 RHI 命令

```cpp
RHICmdList.DrawIndexedPrimitive(...)
    -> FRHICommandList::PreDraw()
        -> FRHIParameterBatcher::PreDraw()
```

`FRHIParameterBatcher::PreDraw`（你给出的代码）：

```cpp
void FRHIParameterBatcher::PreDraw(FRHICommandList& InCommandList)
{
    for (uint32 Index = 0; Index < SF_NumGraphicsFrequencies; Index++)
    {
        InCommandList.SetBatchedShaderParameters(
            GetBatchedGraphicsShader(Index),
            AllBatchedShaderParameters[Index]);
        AllBatchedShaderParameters[Index].Reset();
    }
}
```

可以理解为：

- `AllBatchedShaderParameters` 中暂存的是 RenderThread 收集到的 shader 参数
- `PreDraw()` 会把它们通过 `SetBatchedShaderParameters(...)` 转成 RHI 命令（内部会 `ALLOC_COMMAND(...)` 入队）
- 随后本地 batcher 立刻 `Reset()`，为下一次 draw 准备

## RHIThread：消费批处理参数并落到 D3D12 常量绑定路径

RHIThread 执行入队命令时（如 `FRHICommandSetShaderParameters<FRHIGraphicsShader>`），会进入类似 `SetShaderParametersOnContext` 的路径，把参数写入 D3D12 上下文对应的常量缓存结构。

之后在真正 draw（`RHIDrawPrimitive`）时：

1. `CommitNonComputeShaderConstants`
2. `SetConstantBuffer`（上传/生成版本并缓存 GPU 地址）
3. `ApplyState(Graphics)`
4. `ApplyConstants`（绑定 Root CBV）

## 一句话总结

UE 的 constant 数据绑定不是在 `SetParameters` 时直接绑定 GPU，而是先在 RenderThread 批处理入队，再由 RHIThread 在 draw 提交阶段完成“上传 + 缓存地址 + Root CBV 绑定”。

