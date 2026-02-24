# 🔧 Shader Compile Job 调度与 ShaderCompileWorker 详解（UE5.2）

本文档梳理 Unreal Engine 5.2 中 Shader 编译任务（Job）的调度机制，以及 `ShaderCompileWorker` 的职责与工作流程。目标是在不陷入实现细枝末节的前提下，把“任务如何流动、结果如何落地”这条主链条讲清楚。

## 🎯 文档目标与范围

- 解释 Shader Job 如何被创建、排队、分配、执行和回收。
- 解释主进程与 `ShaderCompileWorker` 之间的文件 IPC 通信模型。
- 解释编译结果如何进入 ShaderMap / RHI，并最终参与 PSO 创建。

## 👥 目标读者

- 渲染/引擎工程师（已熟悉 UE 基础编译流程），希望进一步理解 Shader 编译调度与 Worker 协作机制。

## 📋 一页概览

- UE5.2 的 Shader 编译可视为一组相互独立但有依赖关系的 Job。
- Job 通常由上层 Material/Shader 逻辑生成，随后进入统一调度器。
- 调度器负责：
  - 任务排队与优先级管理。
  - 批处理提交与负载分发。
  - 在本地线程、本地独立 Worker 或远程节点上执行。

## 📦 核心术语与数据结构

- `Job`：一次具体 Shader 编译请求（源码、宏/选项、目标平台、变体信息等）。
- `FShaderCompileJob`（或同类结构）：引擎内部表示单个请求的数据结构。
- 编译队列：缓存待处理 Job 的队列体系，通常带优先级。
- `Worker`：执行编译的实体，可为本地线程池、本地 `ShaderCompileWorker` 进程、远程机器进程。

## 🔄 调度主流程（主进程视角）

### 1️⃣ 全局着色器验证与编译触发

`ShaderCompiler.cpp` 中 `VerifyGlobalShaders()` 负责验证并触发全局着色器编译：

```cpp
// ShaderCompiler.cpp
void VerifyGlobalShaders(...)
{
    // 对每个 GlobalShader 调用 ShouldCompilePermutation
    // 判断是否需要编译该排列组合
}
```

自定义全局着色器时，通常需要实现：

```cpp
class FSimpleShader : public FGlobalShader
{
public:
    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
    {
        // 返回该排列是否需要编译
        return true;
    }
};
```

💡 调试建议：在 `VerifyGlobalShaders` 断点观察触发路径，可区分“真实重编译”与“DDC 命中”。

### 2️⃣ Job 创建与提交

验证通过后，任务被写入 Job 数组，例如：

```cpp
TArray<FShaderCommonCompileJobPtr> GlobalShaderJobs;
// 填充需要编译的 shader jobs
```

典型提交链路分两层：

```cpp
// 第一层：编译管理器提交
void FShaderCompilingManager::SubmitJobs(
    TArray<FShaderCommonCompileJobPtr>& NewJobs,
    ...)
{
    // 将 jobs 提交到编译队列
}

// 第二层：任务集合提交
void FShaderCompileJobCollection::SubmitJobs(
    const TArray<FShaderCommonCompileJobPtr>& NewJobs)
{
    // 将任务加入待处理队列
}
```

### 3️⃣ 队列管理与优先级

`FShaderCompileJobCollection` 维护待处理任务：

```cpp
class FShaderCompileJobCollection
{
    /** 尚未分配给 worker 的任务队列 */
    FShaderCommonCompileJob* PendingJobs[NumShaderCompileJobPriorities];

    /** 每个优先级的待处理任务数量 */
    int32 NumPendingJobs[NumShaderCompileJobPriorities];
};
```

按优先级组织后，高优先级任务（例如当前正在编辑材质相关任务）可更快出结果。

### 4️⃣ Worker 管理与任务分派

调度线程由 `FShaderCompileThreadRunnable` 驱动：

```cpp
class FShaderCompileThreadRunnable : public FRunnable
{
    // Worker 信息数组
    TArray<FShaderCompileWorkerInfo*> WorkerInfos;

    // 编译主循环
    int32 CompilingLoop();
};
```

每个 Worker 的本地任务缓存：

```cpp
class FShaderCompileWorkerInfo
{
    // 分配给该 worker 的任务
    TArray<FShaderCommonCompileJobPtr> QueuedJobs;
};
```

调度线程会将 `PendingJobs` 分配给空闲 Worker，以实现吞吐与延迟的平衡。

### 5️⃣ 主进程与 Worker 的文件 IPC

#### 📌 5.1 编译主循环

```cpp
int32 FShaderCompileThreadRunnable::CompilingLoop()
{
    while (!bForceFinish)
    {
        // 1) 将新任务写入文件
        WriteNewTasks();

        // 2) 读取已完成结果
        ReadAvailableResults();

        // 3) 等待/休眠
        // ...
    }
}
```

#### 📡 5.2 通信步骤

1. `WriteNewTasks()`
- 主进程将待编译任务序列化并写入临时文件。
- 文件通常位于 `Intermediate/ShaderCompileWorker/`。
- 每个 Worker 使用独立输入文件。

2. Worker 执行
- `ShaderCompileWorker` 通过 `OpenInputFile()` 轮询输入文件。
- 读取成功后反序列化 Job（`FArchive`）。
- 调用平台编译器（DXC/FXC 等）进行编译。
- 将结果序列化后写回输出文件。

3. `ReadAvailableResults()`
- 主进程轮询结果文件。
- 读取状态、字节码、错误信息。
- 将结果回传调用方并写入 DDC。

通信示意：

```text
主进程                          Worker 进程
  |                                | while(true)
  |                                |   OpenInputFile()
  |                                |   轮询等待... (Sleep 10ms)
  | WriteNewTasks()               |
  |---> 写入任务文件 ------------>| CreateFileReader 成功
  |                                | 读取并反序列化任务
  |                                | 调用编译器 (DXC/FXC)
  |                                | 编译并写入结果文件
  | ReadAvailableResults()        |
  |<--- 读取结果文件 <------------|
  | 返回结果并写入 DDC            | 继续等待下一个任务
```

### 6️⃣ 执行路径选择

- **本地线程池**：适合快速、小规模任务。
- **独立 Worker 进程**：隔离编译器崩溃风险，适合多核并行。
- **远程分布式**：适合团队级吞吐扩展（代价是网络/序列化开销）。

### 7️⃣ 结果收集与缓存

- 调度器接收编译结果并回传调用方。
- 成功产物写入 `DDC`（Derived Data Cache）。
- 失败任务记录错误日志。
- 相同输入可直接命中缓存，减少重复编译。

## ⚙️ ShaderCompileWorker 的职责与实现

### 📋 职责概览

- 接收并反序列化编译请求（文件 IPC / 网络等）。
- 在隔离环境中调用编译器执行 Shader 编译。
- 回传日志、错误信息、字节码与反射数据。
- 处理临时文件、超时、重试等运行时问题。

### 🔁 Worker 主循环

`ShaderCompileWorker` 的核心循环位于 `FWorkLoop()`：

```cpp
// ShaderCompileWorker.cpp
void FWorkLoop()
{
    while (true)
    {
        // 1. 打开输入文件（等待任务）
        FArchive* InputFilePtr = OpenInputFile();

        if (!InputFilePtr)
        {
            break; // 引擎退出或收到结束信号
        }

        // 2. 反序列化编译任务
        // ...

        // 3. 执行编译
        // ...

        // 4. 写入结果文件
        // ...

        // 5. 清理并进入下一轮
        delete InputFilePtr;
    }
}
```

### ⏳ OpenInputFile 轮询等待机制

```cpp
/** 打开输入文件，必要时多次尝试 */
FArchive* OpenInputFile()
{
    FArchive* InputFile = nullptr;
    bool bFirstOpenTry = true;

    while (!InputFile && !IsEngineExitRequested())
    {
        InputFile = IFileManager::Get().CreateFileReader(
            *InputFilePath,
            FILEREAD_Silent
        );

        if (!InputFile && !bFirstOpenTry)
        {
            CheckExitConditions();
            FPlatformProcess::Sleep(0.01f); // 10ms
        }

        bFirstOpenTry = false;
    }

    return InputFile;
}
```

实现取舍：

- 轮询模型简单、稳定，适合本地多进程。
- `FILEREAD_Silent` 能降低“文件尚未到达”时的日志噪音。
- `Sleep(0.01f)` 避免空转抢占 CPU。

## ♻️ 失败与重试策略（常见做法）

- 本地重试：失败后重试，或降级到更保守编译参数。
- 执行器切换：在本地线程与 `ShaderCompileWorker` 间切换以隔离问题。
- 超时回收：长时间未完成的 Job 标记为超时，必要时重新提交。

## 🚀 性能与可扩展性考虑

- 批量化（batching）+ 去重（dedup）是降低总编译量的核心手段。
- 多核/多机 Worker 可扩展吞吐，但会引入额外通信与序列化成本。
- 本地进程隔离提高稳定性，减少编译器异常对主进程影响。

## 📁 关键源码路径（UE5.2）

### 🖥️ 主进程端

```text
Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.cpp
  - VerifyGlobalShaders()
  - FShaderCompilingManager
  - FShaderCompilingManager::SubmitJobs()

Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.h
  - FShaderCompileJob
  - FShaderCommonCompileJobPtr
  - FShaderCompileJobCollection

Engine/Source/Runtime/ShaderCore/Private/ShaderCompilerCore.cpp
  - FShaderCompileThreadRunnable
  - FShaderCompileThreadRunnable::CompilingLoop()
  - WriteNewTasks()
  - ReadAvailableResults()

Engine/Source/Runtime/ShaderCore/Private/ShaderCompilerCore.h
  - FShaderCompileWorkerInfo
  - QueuedJobs
```

### 🔨 Worker 进程端

```text
Engine/Source/Programs/ShaderCompileWorker/Private/ShaderCompileWorker.cpp
  - main()
  - FWorkLoop()
  - OpenInputFile()
  - 任务反序列化与编译逻辑
```

### 🎨 平台特定编译器（Windows / D3D）

```text
Engine/Source/Developer/Windows/ShaderFormatD3D/Private/ShaderFormatD3D.cpp
  - DXC/FXC 编译入口
```

## 🎯 从编译结果到 RHI Shader 的生命周期

### 1️⃣ 上层使用入口

```cpp
FGlobalShaderMap* GlobalShaderMap = GetGlobalShaderMap(GMaxRHIFeatureLevel);
TShaderMapRef<FSimpleShaderVS> VertexShader(GlobalShaderMap);
TShaderMapRef<FSimpleShaderPS> PixelShader(GlobalShaderMap);

GraphicsPSOInit.BoundShaderState.VertexShaderRHI = VertexShader.GetVertexShader();
```

### 2️⃣ `TShaderRefBase` 如何取到 RHI Shader

```cpp
template<typename ShaderType, typename PointerTableType>
class TShaderRefBase
{
public:
    inline FRHIVertexShader* GetVertexShader() const
    {
        return static_cast<FRHIVertexShader*>(GetRHIShaderBase(SF_Vertex));
    }

private:
    inline FRHIShader* GetRHIShaderBase(EShaderFrequency Frequency) const
    {
        FRHIShader* RHIShader = nullptr;
        if (ShaderContent)
        {
            checkSlow(ShaderContent->GetFrequency() == Frequency);
            RHIShader = GetResourceChecked().GetShader(ShaderContent->GetResourceIndex());
            checkSlow(RHIShader->GetFrequency() == Frequency);
        }
        return RHIShader;
    }

    ShaderType* ShaderContent;
    const FShaderMapBase* ShaderMap;
};
```

要点：

- `ShaderContent` 是元数据描述，不是字节码本体。
- 真正字节码由 `FShaderMapResource` 提供的 RHI Shader 承载。

### 3️⃣ `FShaderMapBase` 的关键成员

```cpp
class FShaderMapBase
{
private:
    TRefCountPtr<FShaderMapResource> Resource;
    TRefCountPtr<FShaderMapResourceCode> Code;
    TMemoryImageObject<FShaderMapContent> Content;
};
```

- `Code`：编译字节码容器。
- `Resource`：RHI 资源包装与创建入口。
- `Content`：Shader 元数据与索引。

### 4️⃣ 编译结果收集与资源初始化

```cpp
// 伪代码
void ProcessCompiledJob(const FShaderCompileJob& CurrentJob)
{
    FGlobalShaderMapSection* Section =
        GGlobalShaderMap[Platform]->FindOrAddSection(ShaderType);

    Section->GetResourceCode()->AddShaderCompilerOutput(
        CurrentJob.Output,
        CurrentJob.Key.ToString()
    );
}

void FShaderMapBase::InitResource()
{
    Resource.SafeRelease();

    if (Code)
    {
        Code->Finalize();
        Resource = new FShaderMapResource_InlineCode(GetShaderPlatform(), Code);
        BeginInitResource(Resource);
    }

    PostFinalizeContent();
}
```

### 5️⃣ PSO 创建阶段提取 Shader

```cpp
GraphicsPSOInit.BoundShaderState.VertexShaderRHI = VertexShader.GetVertexShader();

FRHIShader* FShaderMapResource::GetShader(uint32 ShaderIndex)
{
    return ShaderArray[ShaderIndex];
}
```

### 6️⃣ D3D12 下的真实类型与字节码承载

```cpp
class FD3D12VertexShader
    : public FRHIVertexShader
    , public FD3D12ShaderData
{
public:
    enum { StaticFrequency = SF_Vertex };
};

class FD3D12ShaderData
{
private:
    TArray<uint8> Code;
};
```

PSO 创建时，D3D12 会从具体 Shader 对象提取字节码，填入 `D3D12_GRAPHICS_PIPELINE_STATE_DESC`。

### 🔗 完整链路（摘要）

```text
ShaderCompileWorker 编译结果
  -> ProcessCompiledJob()
  -> FShaderMapResourceCode::AddShaderCompilerOutput()
  -> FShaderMapBase::InitResource()
  -> FShaderMapResource_InlineCode
  -> FShaderMapResource::GetShader()
  -> FRHIShader*（实际为 FD3D12VertexShader*）
  -> 提取字节码
  -> D3D12_GRAPHICS_PIPELINE_STATE_DESC
  -> ID3D12Device::CreateGraphicsPipelineState()
```

### ⚡ 关键设计点

1. 延迟初始化：调用 `InitResource()` 前不创建完整 RHI 资源。
2. 类型擦除：上层通过 `FRHIShader*` 访问，平台细节隐藏于 RHI。
3. 缓存复用：同类 ShaderMap 组合尽量复用已创建资源。
4. 平台隔离：字节码提取逻辑集中在 RHI 层。

## 📝 TODO / 待补充

- 补充超时/重试参数的来源与默认值。
- 补充远程分布式编译的网络协议细节。
- 补充 DDC 存储格式与缓存键生成机制。
- 补充 `FShaderMapResource_InlineCode` 的实现细节。
- 补充 `FD3D12ShaderData` 其余字段的语义。

## 📚 参考

- `Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.cpp`
- `Engine/Source/Programs/ShaderCompileWorker/`
- [着色器编译管线](02-shader-compilation.md)
