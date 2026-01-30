# UE5 Rendering Notes

A bilingual (English/Chinese) comprehensive Markdown knowledge base documenting Unreal Engine 5's rendering pipeline, from Materials to pixels on screen.

> **Language / 语言选择**: [English](en/README.md) | [中文](zh/README.md)

## Overview

This repository provides in-depth, source-code-guided documentation of UE5's rendering architecture. Each chapter breaks down a major rendering system, explaining how it works, which source files implement it, and how to debug it effectively.

**Target Audience**: Graphics programmers, technical artists, and engine developers who want to understand UE5's rendering internals.

## Repository Structure

```
UERenderNotes/
├── en/                          # English documentation
│   ├── README.md               # English introduction
│   ├── glossary.md             # Rendering terminology
│   ├── reference-paths.md      # Source code references
│   └── chapters/               # Chapter documents
│       ├── 01-material-system.md
│       ├── 02-shader-compilation.md
│       ├── 03-rdg.md
│       ├── ...
│       └── 10-tsr.md
│
└── zh/                          # 中文文档
    ├── README.md               # 中文介绍
    ├── glossary.md             # 渲染术语表
    ├── reference-paths.md      # 源码引用路径
    └── chapters/               # 章节文档
        ├── 01-material-system.md
        ├── 02-shader-compilation.md
        ├── 03-rdg.md
        ├── ...
        └── 11-shader-compile-worker.md
```

## Quick Start

- **English Version**: Navigate to [en/](en/) folder or read [English README](en/README.md)
- **中文版本**: 进入 [zh/](zh/) 文件夹或阅读[中文 README](zh/README.md)

## Core Topics Covered

### Rendering Pipeline
- Material System and HLSL Conversion
- Shader Compilation Pipeline
- Render Dependency Graph (RDG)
- BasePass Rendering
- Lighting System
- Shadow Rendering

### Advanced Features
- Post Processing Effects
- Lumen Global Illumination
- Nanite Virtualized Geometry
- Temporal Super Resolution (TSR)
- Shader Compile Worker System

## Documentation Philosophy

- **Source-Code Guided**: Every concept is tied to actual UE5 source code
- **Practical Focus**: Emphasis on how systems work and how to debug them
- **Progressive Depth**: Start with overviews, dive into implementation details
- **Examples Included**: Real code snippets from the engine

## Contributing

Contributions to either English or Chinese documentation are welcome! Please ensure:
- Technical accuracy against UE5 source code
- Clear explanations with code examples
- Proper formatting and cross-references
- Version-specific notes when applicable

## License

This documentation is provided as educational material. Unreal Engine is property of Epic Games.

## Additional Resources

- [Unreal Engine Source Code](https://github.com/EpicGames/UnrealEngine)
- [Official UE Documentation](https://docs.unrealengine.com/)
- [Unreal Engine Graphics Programming Discord](https://discord.gg/unreal-slackers)

---

**Note**: The original files in the root `/chapters/` directory are being phased out. Please use the organized `/en/` and `/zh/` directories instead.
