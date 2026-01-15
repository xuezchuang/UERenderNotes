# UE5 Rendering Notes

A comprehensive Markdown knowledge base documenting Unreal Engine 5's rendering pipeline, from Materials to pixels on screen.

## Overview

This repository provides in-depth, source-code-guided documentation of UE5's rendering architecture. Each chapter breaks down a major rendering system, explaining how it works, which source files implement it, and how to debug it effectively.

**Target Audience**: Graphics programmers, technical artists, and engine developers who want to understand UE5's rendering internals.

## Table of Contents

### Core Documentation
- [Glossary](glossary.md) - Key rendering terms and concepts
- [Reference Paths](reference-paths.md) - Quick lookup for UE5 source files, classes, and functions

### Chapters

1. [Material System and HLSL Conversion](chapters/01-material-system.md)
   - How UE Materials become HLSL shader code
   - Material graph evaluation and code generation
   
2. [Shader Compilation Pipeline](chapters/02-shader-compilation.md)
   - From material graphs to GPU bytecode
   - Shader permutations and the shader map
   
3. [Render Dependency Graph (RDG)](chapters/03-rdg.md)
   - Modern rendering framework
   - Resource management and automatic barriers
   
4. [BasePass Rendering](chapters/04-basepass.md)
   - GBuffer generation
   - Material evaluation in the base pass
   
5. [Lighting System](chapters/05-lighting.md)
   - Deferred and forward lighting
   - Light types and their implementation
   
6. [Shadow Rendering](chapters/06-shadows.md)
   - Shadow mapping techniques
   - Cascaded shadow maps and virtual shadow maps
   
7. [Post Processing](chapters/07-postprocess.md)
   - Post-process pipeline architecture
   - Tone mapping, bloom, and other effects
   
8. [Lumen Global Illumination](chapters/08-lumen.md)
   - Dynamic global illumination system
   - Surface cache and radiance cache
   
9. [Nanite Virtualized Geometry](chapters/09-nanite.md)
   - Virtualized geometry rendering
   - Level of detail and culling systems
   
10. [Temporal Super Resolution (TSR)](chapters/10-tsr.md)
    - Modern temporal upscaling
    - History reprojection and anti-aliasing

## How to Use This Knowledge Base

1. **Start with the Glossary** if you're new to rendering terminology
2. **Follow chapters in order** for a complete understanding of the pipeline
3. **Use Reference Paths** to quickly locate source code when debugging
4. **Each chapter is self-contained** - jump to topics of interest

## Contributing

This knowledge base documents UE5 rendering as of version 5.3+. The engine evolves rapidly, so contributions and updates are welcome.

## Additional Resources

- [Unreal Engine Source Code](https://github.com/EpicGames/UnrealEngine)
- [Official UE Documentation](https://docs.unrealengine.com/)
- [Unreal Engine Graphics Programming Discord](https://discord.gg/unreal-slackers)

## License

This documentation is provided as educational material. Unreal Engine is property of Epic Games.
