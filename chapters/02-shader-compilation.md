# Chapter 2: Shader Compilation Pipeline

## Goals

By the end of this chapter, you will understand:
- How UE5 compiles HLSL shader code to GPU bytecode
- The shader permutation system and why it exists
- Shader maps, shader caches, and Derived Data Cache (DDC)
- How to optimize shader compilation times
- Debugging shader compilation failures

## Overview

After Materials generate HLSL code (Chapter 1), that code must be compiled for each target platform. UE5's shader compilation system manages this complex process, handling thousands of shader variants, cross-platform compilation, and caching for iteration speed.

## Pipeline Steps

### 1. Shader Job Creation
**What happens**: The engine determines which shader permutations need compilation.

**Key decisions**:
- Which shader types? (Vertex, Pixel, Compute, etc.)
- Which feature levels? (ES3.1, SM5, SM6, etc.)
- Which quality settings? (Low, Medium, High, Epic)
- Which static switches are enabled?

### 2. Shader Compilation Job Dispatch
**What happens**: Compilation jobs are queued and distributed.

**Distribution options**:
- **Local**: Compile on current machine
- **XGE** (Incredibuild): Distribute across network
- **FASTBuild**: Open-source distributed compilation
- **SN-DBS**: PlayStation compilation distribution

### 3. Platform-Specific Compilation
**What happens**: HLSL is cross-compiled for target platform.

**Per-platform compilers**:
- **DirectX (Windows)**: DXC (DirectX Shader Compiler)
- **Vulkan**: DXC → SPIR-V (via SPIRV-Cross)
- **Metal (macOS/iOS)**: DXC → Metal IR
- **PlayStation**: HLSL → PSSL
- **Switch**: Custom toolchain

### 4. Shader Optimization
**What happens**: Compiled shaders are optimized for performance.

**Optimizations**:
- Dead code elimination
- Constant folding
- Register allocation
- Instruction scheduling

### 5. Shader Map Assembly
**What happens**: Compiled shader variants are packaged into a shader map.

**Shader map contains**:
- All vertex/pixel/geometry/compute shader variants
- Shader resource bindings (textures, buffers, constants)
- Shader metadata (instruction counts, resource usage)

### 6. Caching
**What happens**: Results stored in DDC for fast iteration.

**Cache locations**:
- **Local DDC**: `<Project>/Saved/DDC/`
- **Shared DDC**: Network location (S3, file share)
- **Derived Data**: Regenerated if source changes

## Key UE Source Files, Classes, and Functions

### Shader Compilation Core

**`Engine/Source/Runtime/ShaderCore/Public/ShaderCompiler.h`**
- `FShaderCompilerEnvironment` - Compilation settings and defines
- `FShaderCompilerInput` - Input to shader compiler (HLSL code, entry point, etc.)
- `FShaderCompilerOutput` - Compiled shader bytecode and metadata
- `FShaderCompileJob` - Represents a single shader compilation task

**`Engine/Source/Runtime/ShaderCore/Private/ShaderCompiler.cpp`**
- `CompileShader()` - Main entry point for shader compilation
- `GlobalBeginCompileShader()` - Starts async shader compilation

### Shader Types

**`Engine/Source/Runtime/ShaderCore/Public/Shader.h`**
- `FShader` - Base class for all shader types
- `FGlobalShader` - Engine-wide shaders (not material-specific)
- `FMaterialShader` - Material-specific shaders
- `FMeshMaterialShader` - Shaders that render meshes with materials

**`Engine/Source/Runtime/RenderCore/Public/ShaderParameters.h`**
- `FShaderParameterMap` - Maps shader parameters to binding slots
- Shader parameter structures (uniform buffers, loose parameters)

### Shader Map

**`Engine/Source/Runtime/Engine/Public/MaterialShaderMap.h`**
- `FMaterialShaderMap` - Contains all compiled shader variants for a Material
- `GetShader<ShaderType>()` - Retrieves specific shader variant
- `IsComplete()` - Checks if all required shaders compiled

**`Engine/Source/Runtime/ShaderCore/Public/ShaderCodeLibrary.h`**
- `FShaderCodeLibrary` - Manages shader code storage and loading
- Handles shader library packaging for shipping builds

### Platform Compilation

**`Engine/Source/Developer/Windows/ShaderFormatD3D/Private/ShaderFormatD3D.cpp`**
- Windows DirectX shader compilation

**`Engine/Source/Developer/ShaderFormatVectorVM/Private/ShaderFormatVectorVM.cpp`**
- Vector VM shader compilation (Niagara)

### Key Functions

```cpp
// ShaderCompiler.cpp
void CompileShader(
    const TArray<const IShaderFormat*>& ShaderFormats,
    FShaderCompilerInput& Input,
    FShaderCompilerOutput& Output,
    const FString& WorkingDirectory);

// Material.cpp
bool UMaterial::CacheShadersForResources(
    EShaderPlatform Platform,
    const TArray<FMaterialResource*>& ResourcesToCache,
    bool bApplyCompletedShaderMapForRendering);

// MaterialShaderMap.cpp
bool FMaterialShaderMap::Compile(
    FMaterial* Material,
    const FMaterialShaderMapId& ShaderMapId,
    TRefCountPtr<FShaderCompilerEnvironment> MaterialEnvironment,
    const FMaterialCompilationOutput& InMaterialCompilationOutput,
    EShaderPlatform Platform,
    EMaterialQualityLevel::Type QualityLevel,
    ERHIFeatureLevel::Type FeatureLevel);
```

## Shader Permutations

### Why Permutations Exist

A single Material can generate hundreds of shader variants:
- Different vertex factories (static mesh, skeletal mesh, etc.)
- Different lighting modes (forward, deferred)
- Different quality settings
- Static switch parameters
- Feature toggles (Lumen, Nanite, etc.)

### Permutation Domain

Shaders declare permutation dimensions:
```cpp
class FMyShaderPS : public FGlobalShader
{
    DECLARE_GLOBAL_SHADER(FMyShaderPS);
    
    class FEnableLightingDim : SHADER_PERMUTATION_BOOL("ENABLE_LIGHTING");
    class FNumSamplesDim : SHADER_PERMUTATION_RANGE_INT("NUM_SAMPLES", 1, 16);
    
    using FPermutationDomain = TShaderPermutationDomain<
        FEnableLightingDim,
        FNumSamplesDim
    >;
    
    static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
    {
        // Return false to skip unnecessary permutations
        return true;
    }
};
```

This creates: 2 (bool) × 16 (range) = **32 permutations**.

### Reducing Permutation Count

**1. Use ShouldCompilePermutation()**:
```cpp
static bool ShouldCompilePermutation(const FGlobalShaderPermutationParameters& Parameters)
{
    // Don't compile expensive variant for mobile
    if (Parameters.Platform == SP_METAL && 
        Parameters.PermutationVector.Get<FEnableExpensiveFeature>())
    {
        return false;
    }
    return true;
}
```

**2. Use Shared Shaders**:
Share shaders across materials when possible (FGlobalShader, FMeshMaterialShader).

**3. Dynamic Branching**:
Use runtime branches instead of static permutations for rarely-changing features.

## Shader Caching and DDC

### Derived Data Cache (DDC)

**Purpose**: Avoid recompiling unchanged shaders.

**How it works**:
1. Compute hash of shader source + compilation settings
2. Check DDC for cached result
3. If hit: Load compiled shader
4. If miss: Compile and store result

**DDC Hash Includes**:
- Shader source code
- Include files
- Compilation flags
- Platform settings
- Engine version

### Local vs. Shared DDC

**Local DDC** (`<Project>/Saved/DerivedDataCache/`):
- Fast access
- Per-machine
- Can be deleted to free space

**Shared DDC** (Network/Cloud):
- Shared across team
- Reduces compilation for other team members
- Configure in `Engine/Config/BaseEngine.ini`:

```ini
[InstalledDerivedDataBackendGraph]
MinimumDaysToKeepFile=7
Root=(Type=KeyLength, Length=120, Inner=AsyncPut)
AsyncPut=(Type=AsyncPut, Inner=Hierarchy)
Hierarchy=(Type=Hierarchical, Inner=Boot, Inner=Pak, Inner=EnginePak, Inner=Local, Inner=Shared)
Shared=(Type=FileSystem, ReadOnly=false, Clean=false, Flush=false, PurgeTransient=true, DeleteUnused=true, UnusedFileAge=25, FoldersToClean=-1, Path=\\network\path\DDC, EnvPathOverride=UE-SharedDataCachePath)
```

### Shader Working Directory

During development, shaders are compiled to:
```
<Project>/Saved/ShaderDebugInfo/<Platform>/
```

Files include:
- `.usf` - HLSL source
- `.d3dasm` - Disassembly (DX)
- `.preprocessed` - Preprocessed source

## Compilation Optimization Strategies

### 1. Incremental Shader Compilation

**Console variable**:
```
r.ShaderCompiler.JobCache=1  // Enable job caching (default)
```

Only recompiles changed shaders.

### 2. Asynchronous Compilation

**Console variables**:
```
r.ShaderCompiler.MaxShaderJobBatchSize=10  // Batch size
r.Shaders.Optimize=1                        // Enable optimization
r.Shaders.KeepDebugInfo=0                   // Disable debug info (smaller/faster)
```

### 3. Distributed Compilation

**XGE (Incredibuild)**:
```
[DevOptions.Shaders]
bAllowDistributedCompilation=true
JobExecutor=XGE
```

**FASTBuild**:
```
[DevOptions.Shaders]
bAllowDistributedCompilation=true
JobExecutor=FASTBuild
```

### 4. Precompiling Shaders

**For packaging**:
- Compile all material shaders during cook
- Generate global shader library
- Use `-precompileglobalshaders` in UnrealPak

### 5. Shader Development Mode

**For iteration**:
```
r.ShaderDevelopmentMode=1      // Faster iteration, less optimization
r.Shaders.Optimize=0           // Skip time-consuming optimizations
r.Shaders.SkipCompression=1    // Skip shader compression
```

**For shipping**:
```
r.ShaderDevelopmentMode=0
r.Shaders.Optimize=1
r.Shaders.SkipCompression=0
```

## Debugging Tips

### 1. View Shader Compilation Progress

**Editor UI**: 
- Bottom-right corner shows "Compiling Shaders (X / Y)"
- Click to expand and see details

**Console**:
```
stat shadercompiling  // Shows compilation statistics
```

### 2. Dump Shader Debug Info

**Enable**:
```
r.ShaderDevelopmentMode=1
r.DumpShaderDebugInfo=1
```

**Output location**:
```
<Project>/Saved/ShaderDebugInfo/<Platform>/<ShaderHash>/
```

### 3. Shader Compilation Errors

**Common error patterns**:

**"Undefined identifier"**:
- Missing `#include`
- Typo in variable/function name
- Check MaterialTemplate.ush for available parameters

**"Type mismatch"**:
- Verify material expression output types match shader input types
- Check Material parameter types

**"Cannot map shader parameter"**:
- Parameter declared but not bound
- Check FShaderParameterMap binding

### 4. Force Shader Recompilation

**Specific Material**:
```cpp
// In C++ code
Material->CacheShaders();
```

**All Materials**:
```
recompileshaders changed    // Recompile modified shaders
recompileshaders all        // Recompile all shaders
recompileshaders material <MaterialName>  // Recompile specific material
```

### 5. Shader Profiling

**Instruction count**:
Material Stats window shows estimated instruction count.

**GPU profiler**:
```
profilegpu  // Shows per-shader GPU timings
stat gpu    // GPU frame time breakdown
```

### 6. Platform-Specific Issues

**DirectX**:
- Install latest DXC compiler
- Check `Engine/Binaries/ThirdParty/ShaderConductor/Win64/`

**Vulkan**:
- Verify SPIR-V tools installed
- Check validation layers: `r.Vulkan.EnableValidation=1`

**Metal**:
- Update Xcode for latest Metal compiler
- Check Metal shader validation in Xcode

## Advanced Topics

### Shader Conductor

UE5 uses Shader Conductor for cross-platform HLSL:
- HLSL → SPIR-V → Platform-specific IR
- Enables writing shaders once for all platforms

**Location**: `Engine/Source/ThirdParty/ShaderConductor/`

### Shader Cache Invalidation

When to clear DDC:
- Engine version upgrade
- Shader compilation settings changed
- Persistent unexplained rendering issues

**Clear DDC**:
```bash
# Delete local cache
rm -rf <Project>/Saved/DerivedDataCache/

# Clear in Editor
Edit → Editor Preferences → General → Derived Data Cache → Delete Local Derived Data Cache
```

### Custom Shader Formats

Create custom shader formats for proprietary platforms:
1. Implement `IShaderFormat` interface
2. Register format in module startup
3. Provide cross-compilation pipeline

Example: `Engine/Source/Developer/ShaderFormatVectorVM/`

### Shader Bytecode Serialization

Compiled shaders are serialized to:
- `.ushaderbytecode` files
- Packed into shader libraries for shipping
- Format is platform-specific

## Performance Considerations

### Compilation Time vs. Runtime Performance

**Development** (fast iteration):
- `r.Shaders.Optimize=0` - Faster compile, slower runtime
- Fewer permutations (skip non-essential variants)

**Shipping** (fast runtime):
- `r.Shaders.Optimize=1` - Slower compile, faster runtime
- Precompile all permutations needed

### Memory Usage

Each shader variant consumes:
- Bytecode (typically 1-50 KB per variant)
- Metadata (bindings, reflection data)

Excessive permutations can bloat game packages.

### Cook Times

Large projects can have 30-60 minute shader compilation during cooking.

**Optimizations**:
- Shared DDC across team
- Distributed compilation (XGE)
- Reduce material permutations
- Share materials/material instances

## Summary

Shader compilation pipeline:
1. **Job Creation** - Determine required permutations
2. **Dispatch** - Queue compilation jobs
3. **Compilation** - HLSL → platform bytecode
4. **Optimization** - Optimize shader performance
5. **Assembly** - Package into shader maps
6. **Caching** - Store in DDC for reuse

Key classes: `FShaderCompilerInput`, `FShaderCompilerOutput`, `FMaterialShaderMap`

The system handles thousands of shader variants efficiently through permutation management, caching, and distributed compilation.

## Next Chapter

[Chapter 3: Render Dependency Graph (RDG)](03-rdg.md) - Learn UE5's modern rendering framework that manages GPU resources and scheduling.
