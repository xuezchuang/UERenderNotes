# 🔧 Shader Compile Job 调度与 ShaderCompileWorker 详解（UE5.2）

本文档为草稿，目的是记录在 Unreal Engine 5.2 中 shader 编译调度（以 `ShaderCommonCompile` 为中心概念）如何分配编译任务（job），以及 `ShaderCompileWorker` 进程/模块的职责与工作流程。后续我会把需要精确代码引用的地方标注为 TODO，等你把理解补上或让我进一步抓取源代码路径并填充具体实现细节。

## 🎯 目标读者

- 渲染/引擎工程师，已经熟悉 UE 的编译流程，想了解 shader 编译任务如何被排队、分派以及如何与远程/本地 worker 协作。

## 📋 概览

- 在 UE5.2 中，shader 编译是一系列独立但互相关联的编译任务（job）。这些 job 通常由上层系统（Material/Shader 系统）创建，随后进入到统一的调度系统。
- 调度系统负责：任务排队、优先级管理、合并/分批提交、在本地线程池上执行，或派发到独立的 `ShaderCompileWorker` 进程/远程 worker（用于并行与分布式编译）。

## 📦 主要概念/数据结构（术语）

- Job：表示一次具体的 shader 编译请求（通常包含 shader 源、编译选项、目标平台、变体信息等）。
- FShaderCompileJob（或类似命名）：引擎内部表示单个编译请求的结构体/类。
- 编译队列：一个或多个队列用于缓存待处理的 job，支持优先级与批处理。
- Worker：执行编译的实体，可为：本地线程（线程池）、本地独立进程（`ShaderCompileWorker`）、或远程机器上的进程（分布式/远程编译）。

## 🔄 调度流程详解（基于 UE5.2 源码分析）

### 1️⃣ 全局着色器验证与编译触发

在 `ShaderCompiler.cpp` 中，`VerifyGlobalShaders()` 函数负责验证并触发全局着色器的编译：

```cpp
// ShaderCompiler.cpp
void VerifyGlobalShaders(...)
{
    // 对每个 GlobalShader 调用 ShouldCompilePermutation
    // 判断是否需要编译该排列组合
}
```

当自定义全局着色器时，通常需要定义 `ShouldCompilePermutation` 静态函数：

```cpp
class FSimpleShader : public FGlobalShader
{
public:
    // 决定是否编译该着色器排列
    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
    {
        // 返回是否需要编译
        return true;
    }
};
```

💡 **调试技巧**：在 `VerifyGlobalShaders` 处打断点，可以在外层强制触发重新编译。这对调试很有用，因为引擎默认使用缓存系统（DDC）来避免重复编译。

### 2️⃣ 编译任务的创建与提交

#### 📝 2.1 任务填充

验证通过后，编译任务会被填充到任务数组中：

```cpp
TArray<FShaderCommonCompileJobPtr> GlobalShaderJobs;
// 填充需要编译的 shader jobs
```

#### 📤 2.2 任务提交流程

任务提交经过两个关键函数：

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

### 3️⃣ 任务队列管理

在 `FShaderCompileJobCollection` 中，使用以下变量记录当前待处理的任务：

```cpp
class FShaderCompileJobCollection
{
    /** 尚未分配给 worker 的任务队列 */
    FShaderCommonCompileJob* PendingJobs[NumShaderCompileJobPriorities];
    
    /** 每个优先级的待处理任务数量 */
    int32 NumPendingJobs[NumShaderCompileJobPriorities];
};
```

这里按优先级组织任务，确保高优先级任务（如当前编辑的材质）能优先编译。

### 4️⃣ 编译线程与 Worker 管理

#### 🧵 4.1 编译线程

`FShaderCompileThreadRunnable` 是一个专门的编译线程，负责管理所有的编译 worker：

```cpp
class FShaderCompileThreadRunnable : public FRunnable
{
    // Worker 信息数组
    TArray<FShaderCompileWorkerInfo*> WorkerInfos;
    
    // 主循环
    int32 CompilingLoop();
};
```

💻 **实际案例**：在 14700K（14核）上，引擎会初始化 **14 个编译 worker**，每个 worker 对应一个独立的 `ShaderCompileWorker` 进程，充分利用多核性能。

#### ⚖️ 4.2 Worker 任务分配

每个 `FShaderCompileWorkerInfo` 维护自己的任务队列：

```cpp
class FShaderCompileWorkerInfo
{
    // 分配给该 worker 的任务
    TArray<FShaderCommonCompileJobPtr> QueuedJobs;
};
```

编译线程会自动将 `PendingJobs` 分配给空闲的 worker，实现负载均衡。

### 5️⃣ Worker 进程通信机制

#### 🔁 5.1 编译循环

编译的核心循环在 `CompilingLoop()` 中：

```cpp
int32 FShaderCompileThreadRunnable::CompilingLoop()
{
    while (!bForceFinish)
    {
        // 1. 将新任务写入文件
        WriteNewTasks();
        
        // 2. 读取已完成的编译结果
        ReadAvailableResults();
        
        // 3. 等待/休眠
        // ...
    }
}
```

#### 📡 5.2 基于文件的 IPC 通信

主进程与 `ShaderCompileWorker` 进程之间通过**文件交换**进行通信：

1️⃣ **WriteNewTasks()**：
   - 主进程将待编译任务序列化写入临时文件
   - 文件路径通常在 `Intermediate/ShaderCompileWorker/` 下
   - 每个 worker 有独立的输入文件

2️⃣ **Worker 处理**：
   - `ShaderCompileWorker` 进程通过 `OpenInputFile()` 轮询等待输入文件
   - 检测到文件后立即读取（`CreateFileReader` 成功）
   - 反序列化编译任务（从 `FArchive` 读取 Job 数据）
   - 调用平台特定的着色器编译器（DXC、FXC 等）
   - 编译完成后序列化结果并写入输出文件

3️⃣ **ReadAvailableResults()**：
   - 主进程定期检查结果文件
   - 读取编译结果（成功/失败、字节码、错误信息）
   - 将结果返回给调用方并存入 DDC

📊 **通信流程图**：
```
主进程                          Worker 进程
  |                                | while(true)
  |                                |   OpenInputFile()
  |                                |   轮询等待... (Sleep 10ms)
  | WriteNewTasks()               |
  |---> 写入任务文件 ------------>| CreateFileReader 成功!
  |                                | 读取 & 反序列化任务
  |                                | 调用编译器 (DXC/FXC)
  |                                | 编译着色器
  |                                | 序列化 & 写入结果文件
  | ReadAvailableResults()        |
  |<--- 读取结果文件 <------------|
  |                                | 继续 while 循环
  | 返回结果 + 存入 DDC           | 等待下一个任务...      |
  | 返回结果 + 存入 DDC           |
```

### 6️⃣ 执行路径选择

调度器根据不同场景选择执行方式：

- **本地线程池**：适合快速、少量任务（开发时小规模编译）
- **独立 Worker 进程**：隔离编译器运行，防止崩溃影响主进程，支持多核并行
- **远程分布式**：在大型团队中，可将任务分发到远程机器集群

### 7️⃣ 结果收集与缓存

编译完成后：
- 结果返回给调度器
- 成功的编译产物存入 **DDC（Derived Data Cache）**
- 失败的任务记录错误日志
- 下次相同输入会直接使用缓存，避免重复编译

## ⚙️ ShaderCompileWorker 的职责与实现

### 📋 职责概览

- 接收编译请求（通过 IPC / 网络 /文件交换），反序列化 job 的输入。
- 在隔离环境中运行实际的 shader 编译器（例如调用平台特定的 HLSL 编译器或引擎封装的编译步骤）。
- 记录并返回编译日志、错误信息和编译产物（字节码、反射数据等）。
- 管理临时文件、超时与重试策略。
- 在分布式场景下，能够同时处理多个并发 job，并按协议返回给主调度端。

### 🔁 Worker 进程核心循环

`ShaderCompileWorker` 作为独立进程运行，其主循环在 `FWorkLoop()` 中实现：

```cpp
// ShaderCompileWorker.cpp
void FWorkLoop()
{
    while (true)
    {
        // 1. 打开输入文件（阻塞等待）
        FArchive* InputFilePtr = OpenInputFile();
        
        if (!InputFilePtr)
        {
            break; // 引擎退出
        }
        
        // 2. 反序列化编译任务
        // ...
        
        // 3. 执行编译
        // ...
        
        // 4. 写入结果文件
        // ...
        
        // 5. 清理并准备下一次循环
        delete InputFilePtr;
    }
}
```

### ⏳ 文件等待机制

Worker 通过**轮询文件系统**来检测新任务，这是一个阻塞式的等待循环：

```cpp
/** 打开输入文件，必要时多次尝试 */
FArchive* OpenInputFile()
{
    FArchive* InputFile = nullptr;
    bool bFirstOpenTry = true;
    
    while (!InputFile && !IsEngineExitRequested())
    {
        // 尝试打开输入文件
        InputFile = IFileManager::Get().CreateFileReader(
            *InputFilePath, 
            FILEREAD_Silent
        );
        
        if (!InputFile && !bFirstOpenTry)
        {
            // 检查退出条件
            CheckExitConditions();
            
            // 让出 CPU 时间，避免空转
            FPlatformProcess::Sleep(0.01f); // 10ms
        }
        
        bFirstOpenTry = false;
    }
    
    return InputFile;
}
```

🔍 **工作原理**：
1. **死循环等待**：Worker 进程在 `while` 循环中持续尝试打开输入文件
2. **非阻塞检测**：使用 `FILEREAD_Silent` 标志避免文件不存在时的错误输出
3. **CPU 友好**：每次失败后 `Sleep(0.01f)` 让出 CPU，避免 100% 占用
4. **退出检测**：通过 `IsEngineExitRequested()` 和 `CheckExitConditions()` 监测主进程状态
5. **文件到达**：一旦主进程写入任务文件，`CreateFileReader` 成功，循环退出并开始处理

⚡ **性能考量**：
- 10ms 的轮询间隔在响应速度和 CPU 使用之间取得平衡
- 多个 Worker 进程独立轮询各自的输入文件，互不干扰
- 文件 I/O 相比网络 IPC 更简单可靠，适合本地多进程通信

## ♻️ 失败与重试策略（常见做法）

- 本地重试：在本地线程编译失败时，可能会简单重试或降级为更保守的编译选项。
- 切换执行器：某些错误可能会触发将 job 从本地线程切换到 `ShaderCompileWorker` 以隔离问题（或反之）。
- 超时与回收：长时间未完成的 job 会被回收并记录为超时，必要时回滚或重新提交。

## 🚀 性能与可扩展性考虑

- 批量化（batching）和去重（dedup）是减少总编译量的关键。
- 把编译交给 多核/多机 的 worker 可以线性扩展吞吐，但会引入网络/序列化开销。
- 本地进程隔离（使用 `ShaderCompileWorker` 进程）能防止编译器崩溃影响主进程，并允许为 worker 指定不同的进程环境或资源限制。

## 📁 关键源码路径（UE5.2）

### 🖥️ 主进程端

```
Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.cpp
  └─ VerifyGlobalShaders() - 全局着色器验证
  └─ FShaderCompilingManager - 编译管理器主类
  └─ FShaderCompilingManager::SubmitJobs() - 提交编译任务

Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.h
  └─ FShaderCompileJob - 编译任务结构
  └─ FShaderCommonCompileJobPtr - 任务智能指针
  └─ FShaderCompileJobCollection - 任务集合管理

Engine/Source/Runtime/ShaderCore/Private/ShaderCompilerCore.cpp
  └─ FShaderCompileThreadRunnable - 编译线程
  └─ FShaderCompileThreadRunnable::CompilingLoop() - 编译主循环
  └─ WriteNewTasks() - 写入任务到文件
  └─ ReadAvailableResults() - 读取编译结果

Engine/Source/Runtime/ShaderCore/Private/ShaderCompilerCore.h
  └─ FShaderCompileWorkerInfo - Worker 信息管理
  └─ QueuedJobs - Worker 任务队列
```

### 🔨 Worker 进程端

```FWorkLoop() - Worker 主循环（无限循环）
  └─ OpenInputFile() - 轮询等待输入文件（阻塞式）
  └─ CreateFileReader() - 文件 I/O 接口
  └─ 任务反序列化与编译
Engine/Source/Programs/ShaderCompileWorker/Private/ShaderCompileWorker.cpp
  └─ main() - Worker 进程入口
  └─ 文件监测与任务处理循环
  └─ 调用平台编译器接口
```

### 🎨 平台特定编译器

```
Engine/Source/Developer/Windows/ShaderFormatD3D/Private/ShaderFormatD3D.cpp
  └─ DirectX 着色器编译（DXC/FXC）
```

## 🎯 编译结果到 RHI Shader 的完整生命周期

本章节详细追踪 ShaderCompileWorker 的编译结果如何一步步演变为 FRHIVertexShader，并最终绑定到 PSO（Pipeline State Object）。

### 📍 起点：全局着色器的使用

在上层代码中，我们通常这样使用全局着色器：

```cpp
FGlobalShaderMap* GlobalShaderMap = GetGlobalShaderMap(GMaxRHIFeatureLevel);
TShaderMapRef<FSimpleShaderVS> VertexShader(GlobalShaderMap);
TShaderMapRef<FSimpleShaderPS> PixelShader(GlobalShaderMap);

// 最终赋值给 PSO
GraphicsPSOInit.BoundShaderState.VertexShaderRHI = VertexShader.GetVertexShader();
```

其中 `VertexShader` 的类型为 `TShaderMapRef<FSimpleShaderVS>`，继承自模板基类 `TShaderRefBase<ShaderType, PointerTableType>`。

### 🔍 第一层：TShaderRefBase 的接口

`TShaderRefBase` 提供了获取 RHI 层 Shader 指针的接口：

```cpp
template<typename ShaderType, typename PointerTableType>
class TShaderRefBase
{
public:
    // 获取 Vertex Shader 的 RHI 指针
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

    // 核心成员
    ShaderType* ShaderContent;           // 该 Shader 的描述信息
    const FShaderMapBase* ShaderMap;     // 全局 ShaderMap（一个 .usf 对应一个）
};
```

**关键点**：
- `ShaderMap` 是全局唯一的，对应一个 `.usf` 文件（如 `SimpleShader.usf`）
- `ShaderContent` 是该 ShaderMap 中某个具体 Shader 的描述信息（元数据），不是实际的编译字节码
- 真正的编译字节码来自 `GetResourceChecked().GetShader(...)` 返回的 RHI 对象

### 🏗️ 第二层：FShaderMapBase 的内部结构

在 `TShaderRefBase` 调用 `GetResourceChecked()` 时，返回的是 `FShaderMapResource`。而 `FShaderMapBase` 内部维护了三个关键的数据结构：

```cpp
class FShaderMapBase
{
private:
    TRefCountPtr<FShaderMapResource> Resource;       // RHI 资源对象
    TRefCountPtr<FShaderMapResourceCode> Code;       // 编译字节码（最核心！）
    TMemoryImageObject<FShaderMapContent> Content;   // Shader 元数据
};
```

**核心理解**：
- **Code**：存储从 ShaderCompileWorker 接收到的编译字节码，本质上对应 D3D12 创建 PSO 时需要的 Shader 字节码
- **Resource**：RHI 层资源对象，包装了 Code，并负责创建具体的 RHI Shader（如 FD3D12VertexShader）
- **Content**：记录该 ShaderMap 中所有 Shader 的元数据和索引信息

### 💾 第三层：编译结果的收集与存储

编译完成后，主进程接收 ShaderCompileWorker 的编译结果。这个过程涉及两个关键阶段：

#### 📥 阶段1：ProcessCompiledJob

```cpp
// 伪代码流程
void ProcessCompiledJob(const FShaderCompileJob& CurrentJob)
{
    // 1. 创建或获取对应的 FGlobalShaderMapSection
    FGlobalShaderMapSection* Section = 
        GGlobalShaderMap[Platform]->FindOrAddSection(ShaderType);
    
    // 2. 将编译输出添加到 Code 中
    Section->GetResourceCode()->AddShaderCompilerOutput(
        CurrentJob.Output,           // 包含字节码、反射数据等
        CurrentJob.Key.ToString()
    );
}
```

此时，ShaderCompileWorker 的编译输出已经被存入 `FShaderMapResourceCode` 对象中。

#### 🔄 阶段2：FinalizeContent 与资源初始化

随后在 `SaveGlobalShaderMapToDerivedDataCache()` 中，执行最终初始化：

```cpp
void FShaderMapBase::FinalizeContent()
{
    // ... 元数据处理 ...
}

void FShaderMapBase::InitResource()
{
    Resource.SafeRelease();
    
    if (Code)
    {
        // 1. 对 Code 执行最终化处理
        Code->Finalize();
        
        // 2. 创建 RHI 资源对象
        Resource = new FShaderMapResource_InlineCode(
            GetShaderPlatform(), 
            Code
        );
        
        // 3. 向 RHI 系统注册资源
        BeginInitResource(Resource);
    }
    
    PostFinalizeContent();
}
```

**转折点**：此时字节码已经被 Finalize 并绑定到 RHI Resource 对象中。

### 🎮 第四层：PSO 创建时的 Shader 提取

当创建 PSO 时，调用链如下：

```cpp
// 1. 上层 PSO 初始化
GraphicsPSOInit.BoundShaderState.VertexShaderRHI = VertexShader.GetVertexShader();

// 2. GetVertexShader() → GetRHIShaderBase(SF_Vertex)
//    → GetResourceChecked().GetShader(ShaderContent->GetResourceIndex())

// 3. FShaderMapResource::GetShader() 返回之前缓存的 RHI Shader
FRHIShader* FShaderMapResource::GetShader(uint32 ShaderIndex)
{
    // 如果尚未创建，则调用 CreateRHIShaderOrCrash()
    return ShaderArray[ShaderIndex];
}
```

### 🔧 真实的平台实现：FD3D12VertexShader

在 D3D12 RHI 中，`FRHIVertexShader*` 的真实类型是 `FD3D12VertexShader`：

```cpp
class FD3D12VertexShader
    : public FRHIVertexShader              // 基类指针（外部使用）
    , public FD3D12ShaderData              // 包含平台特定数据
{
public:
    enum { StaticFrequency = SF_Vertex };
};

class FD3D12ShaderData
{
private:
    TArray<uint8> Code;    // 实际的 Shader 字节码！
    // 其他平台相关的数据...
};
```

**类型擦除设计**：
- 上层代码仅通过 `FRHIVertexShader*` 基类指针使用
- 实际对象是平台特定的 `FD3D12VertexShader`
- `FD3D12ShaderData::Code` 存储最终的编译字节码

### 📊 创建 PSO 时的字节码提取

在 D3D12 中创建 Pipeline State 时，通过以下宏从 Shader 中提取字节码：

```cpp
static FD3D12LowLevelGraphicsPipelineStateDesc GetLowLevelGraphicsPipelineStateDesc(...)
{
    #define COPY_SHADER(Initial, Name) \
        if (FD3D12##Name##Shader* Shader = \
            (FD3D12##Name##Shader*)Initializer.BoundShaderState.Get##Name##Shader()) \
        { \
            Desc.Desc.Initial##S = Shader->GetShaderBytecode();      // 获取字节码 \
            Desc.Initial##SHash = Shader->GetBytecodeHash();         // 获取哈希值 \
        }
    
    COPY_SHADER(V, Vertex);    // 提取 Vertex Shader 字节码
    COPY_SHADER(P, Pixel);     // 提取 Pixel Shader 字节码
    
    #undef COPY_SHADER
}
```

这些字节码随后被填入 D3D12_GRAPHICS_PIPELINE_STATE_DESC，由 `ID3D12Device::CreateGraphicsPipelineState()` 使用。

### 🔗 完整链路总结

```
ShaderCompileWorker 编译结果
    ↓
ProcessCompiledJob()
    ↓
FShaderMapResourceCode::AddShaderCompilerOutput()
    ↓
FShaderMapBase::InitResource()
    ↓
FShaderMapResource_InlineCode 创建 (绑定 Code)
    ↓
FShaderMapResource::GetShader() 返回 FRHIShader*
    ↓
投射为 FD3D12VertexShader* (真实类型)
    ↓
提取 FD3D12ShaderData::Code (字节码)
    ↓
填入 D3D12_GRAPHICS_PIPELINE_STATE_DESC
    ↓
ID3D12Device::CreateGraphicsPipelineState()
    ↓
最终的 ID3D12PipelineState（GPU 可执行的管线）
```

### ⚡ 关键设计模式

1. **延迟初始化**：ShaderMap 直到 `InitResource()` 调用才真正创建 RHI 对象
2. **类型擦除**：上层统一使用 `FRHIShader*` 基类指针，具体类型隐藏在 RHI 实现中
3. **缓存优化**：同一个 ShaderMap 组合只创建一次 RHI 对象，反复使用相同的 `FD3D12VertexShader*`
4. **平台隔离**：字节码提取逻辑在 RHI 层，主程序不需要了解平台细节

## 📝 TODO / 待补充

- TODO: 补充具体的超时/重试参数来源与默认值
- TODO: 补充远程分布式编译的网络协议细节
- TODO: 补充 DDC 存储格式和缓存键生成机制
- TODO: 补充 FShaderMapResource_InlineCode 的具体实现细节
- TODO: 补充 FD3D12ShaderData 中其他平台相关数据的含义

## 📚 参考

- 引擎源码：`Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.cpp`
- 引擎源码：`Engine/Source/Programs/ShaderCompileWorker/`
- 相关文档：[着色器编译管线](02-shader-compilation.md)

---

我已经把核心结构和待补充项写成草稿。你接下来可以：

- 把你自己的理解直接写到本文件中（在补充 TODO 部分或对应章节）；
- 或者让我去线上/本地搜索 UE5.2 源代码并把具体函数/文件引用、代码片段填进 TODO 位置（我可以继续完成）。

底部新增了 TODO 标记以便补全。欢迎指示下一步。
