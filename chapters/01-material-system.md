# Chapter 1: Material System and HLSL Conversion

## Goals

By the end of this chapter, you will understand:
- How UE5 Material graphs are structured and evaluated
- The process of translating Material nodes to HLSL code
- Key classes and systems involved in Material → shader code generation
- How to debug Material compilation issues

## Overview

Unreal Engine's Material system provides a visual, node-based interface for authoring shaders. Behind the scenes, the Material graph is translated into HLSL shader code that runs on the GPU. This chapter explores that transformation pipeline.

## Pipeline Steps

### 1. Material Graph Construction
**What happens**: Artists create Materials using the Material Editor, connecting nodes to define surface properties.

**Key concepts**:
- Material nodes represent operations (math, textures, parameters)
- Nodes connect via pins (inputs/outputs)
- The graph flows from inputs → operations → material attributes output

### 2. Material Translation
**What happens**: The Material graph is walked and translated into intermediate expression trees.

**Process**:
1. Graph validation (checking connections, types)
2. Node compilation (each node generates HLSL code snippets)
3. Expression tree building (dependency tracking)
4. Dead code elimination (unused branches removed)

### 3. HLSL Code Generation
**What happens**: Expression trees are serialized into actual HLSL shader code.

**Output includes**:
- Vertex shader code (world position offsets, vertex manipulation)
- Pixel shader code (material attributes: base color, roughness, normal, etc.)
- Material parameter declarations (uniforms, textures, samplers)
- Custom HLSL code nodes (if used)

### 4. Shader Compilation
**What happens**: Generated HLSL is passed to the shader compiler (covered in Chapter 2).

## Key UE Source Files, Classes, and Functions

### Core Material Classes

**`Engine/Source/Runtime/Engine/Classes/Materials/Material.h`**
- `UMaterial` - The base Material asset class
- Stores the Material graph and compilation results
- Contains Material properties (blend mode, shading model, etc.)

**`Engine/Source/Runtime/Engine/Classes/Materials/MaterialExpression.h`**
- `UMaterialExpression` - Base class for all Material nodes
- Each node type inherits from this (UMaterialExpressionTextureSample, UMaterialExpressionAdd, etc.)
- `Compile()` method generates HLSL for each node

**`Engine/Source/Runtime/Engine/Classes/Materials/MaterialInstance.h`**
- `UMaterialInstance` - Instance of a Material with overridden parameters
- Inherits shader code from parent but can change parameter values

### Material Compilation

**`Engine/Source/Runtime/Engine/Private/Materials/MaterialCompiler.h`**
- `FMaterialCompiler` - Interface for compiling Material expressions
- Translates nodes to intermediate shader code

**`Engine/Source/Runtime/Engine/Private/Materials/HLSLMaterialTranslator.h`**
- `FHLSLMaterialTranslator` - Main class that converts Material graph → HLSL
- `Translate()` - Entry point for translation
- Generates the shader code strings

**`Engine/Source/Runtime/Engine/Private/Materials/Material.cpp`**
- `UMaterial::CacheShadersForResources()` - Triggers Material compilation
- `UMaterial::BeginCacheForCookedPlatformData()` - Cooking-time compilation

### Key Functions

```cpp
// Material.cpp
bool UMaterial::CacheShadersForResources(
    EShaderPlatform Platform,
    const TArray<FMaterialResource*>& ResourcesToCache,
    bool bApplyCompletedShaderMapForRendering);

// MaterialCompiler.h
int32 FMaterialCompiler::Add(int32 A, int32 B);
int32 FMaterialCompiler::Mul(int32 A, int32 B);
int32 FMaterialCompiler::TextureSample(
    int32 Texture,
    int32 Coordinate,
    EMaterialSamplerType SamplerType);

// HLSLMaterialTranslator.cpp
FString FHLSLMaterialTranslator::GetMaterialShaderCode();
void FHLSLMaterialTranslator::GetMaterialEnvironment(
    EShaderPlatform Platform,
    FShaderCompilerEnvironment& OutEnvironment);
```

## Material Node Examples

### Simple Node Compilation
When you connect two values with an Add node:
```
[Constant(0.5)] → Add → [Output]
[Constant(0.3)] → Add
```

Generated HLSL:
```hlsl
float Local0 = 0.5;
float Local1 = 0.3;
float Local2 = Local0 + Local1; // Result: 0.8
```

### Texture Sample Node
```
[Texture2D] → TextureSample → [Output]
[UV Coordinates] → TextureSample
```

Generated HLSL:
```hlsl
MaterialFloat2 Local0 = Parameters.TexCoords[0].xy;
MaterialFloat4 Local1 = Texture2DSample(
    Material.Texture2D_0,
    Material.Texture2D_0Sampler,
    Local0);
```

## Material Parameters and Shader Code

### Material Parameters
Materials expose parameters that can be changed at runtime without recompilation:

**Scalar Parameters**:
```hlsl
// Declared in generated shader
float Material_ScalarParam_0; // Roughness multiplier
```

**Texture Parameters**:
```hlsl
Texture2D Material_Texture2D_0;
SamplerState Material_Texture2D_0Sampler;
```

**Vector Parameters**:
```hlsl
float4 Material_VectorParam_0; // Tint color
```

### Shader Code Structure

Generated pixel shader structure:
```hlsl
void CalcPixelMaterialInputs(
    FMaterialPixelParameters Parameters,
    inout FPixelMaterialInputs PixelMaterialInputs)
{
    // Material graph code goes here
    // Computes BaseColor, Roughness, Metallic, etc.
    
    PixelMaterialInputs.BaseColor = <compiled expression>;
    PixelMaterialInputs.Roughness = <compiled expression>;
    PixelMaterialInputs.Metallic = <compiled expression>;
    // ... other attributes
}
```

## Debugging Tips

### 1. View Generated Shader Code

**Engine Console Commands**:
```
r.ShaderDevelopmentMode 1    // Enable shader debugging
r.DumpShaderDebugInfo 1      // Dump shader files to Saved/ShaderDebugInfo
```

After enabling, compiled shaders are saved to:
```
<Project>/Saved/ShaderDebugInfo/<Platform>/<ShaderHash>/
```

Files include:
- `.usf` - Generated HLSL code
- `.d` - Dependency information
- Compilation metadata

### 2. Material Stats

**Material Editor → Window → Stats**
- Shows instruction count estimates
- Texture sample counts
- Complexity metrics

### 3. Shader Complexity View Modes

**In Editor Viewport**:
- `Alt + 8` - Shader Complexity (green = cheap, red = expensive)
- `Alt + 9` - Quad Overdraw

**Console Commands**:
```
viewmode ShaderComplexity
viewmode QuadComplexity
```

### 4. Common Compilation Errors

**Error: "Missing connection to Material Output"**
- Fix: Ensure all required material attributes are connected
- Required attributes vary by shading model

**Error: "Texture coordinate out of range"**
- Fix: UE supports 8 UV channels (0-7), check your UV index

**Error: "Cyclic dependency detected"**
- Fix: Material graph contains a circular reference, break the loop

### 5. Debug Material Expressions

Add `UE_LOG` statements in node `Compile()` methods:
```cpp
// In MaterialExpressionCustom.cpp
int32 UMaterialExpressionCustom::Compile(
    class FMaterialCompiler* Compiler,
    int32 OutputIndex)
{
    UE_LOG(LogMaterial, Warning, 
        TEXT("Compiling Custom node: %s"), *Code);
    // ... compilation code
}
```

### 6. Material Instance Issues

If a Material Instance renders incorrectly:
1. Check parent Material compiled successfully
2. Verify parameter overrides are correct type
3. Check "Static Switch" parameters trigger recompilation
4. Try "Reimport" on the Material Instance

## Advanced Topics

### Static Switches
Static switches create shader permutations:
```
StaticSwitch → [Branch A] (if true)
             → [Branch B] (if false)
```

Each state generates a different shader variant, increasing compile time and memory.

### Material Layers
UE5 supports material layering for reusable material functions:
- Material Layer Blend (MLB) system
- Layer stacks combine multiple materials
- Compiled as single unified shader

### Custom HLSL Nodes
Custom nodes allow arbitrary HLSL:
- Use with caution (bypasses graph validation)
- Must manually declare inputs/outputs
- Useful for advanced effects not available via nodes

## Summary

The Material → HLSL pipeline:
1. **Graph Construction** - Visual node graph
2. **Translation** - Nodes → expression trees
3. **Code Generation** - Expression trees → HLSL strings
4. **Compilation** - HLSL → GPU bytecode (Chapter 2)

Key classes: `UMaterial`, `UMaterialExpression`, `FHLSLMaterialTranslator`

The system is extensible - you can create custom Material nodes by inheriting from `UMaterialExpression` and implementing the `Compile()` method.

## Next Chapter

[Chapter 2: Shader Compilation Pipeline](02-shader-compilation.md) - Learn how generated HLSL becomes GPU-executable bytecode.
