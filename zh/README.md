# UE5 渲染笔记 - 中文版

全面的 Markdown 知识库，记录 Unreal Engine 5 的渲染管线，从材质到屏幕像素的完整流程。

> **语言选择**: [English](../en/README.md) | [中文](README.md)

## 概述

本仓库提供深入的、以源代码为导向的 UE5 渲染架构文档。每个章节都详细分析一个主要的渲染系统，解释其工作原理、实现它的源文件以及如何有效调试。

**目标读者**: 图形程序员、技术美术和引擎开发者，希望深入理解 UE5 渲染内部机制。

## 目录

### 核心文档
- [术语表](glossary.md) - 关键渲染术语和概念
- [引用路径](reference-paths.md) - UE5 源文件、类和函数的快速查找

### 章节

1. [材质系统与 HLSL 转换](chapters/01-material-system.md)
   - UE 材质如何转换为 HLSL 着色器代码
   - 材质图评估和代码生成
   
2. [着色器编译管线](chapters/02-shader-compilation.md)
   - 从材质图到 GPU 字节码
   - 着色器排列组合和 shader map
   
3. [渲染依赖图 (RDG)](chapters/03-rdg.md)
   - 现代渲染框架
   - 资源管理和自动屏障
   
4. [BasePass 渲染](chapters/04-basepass.md)
   - GBuffer 生成
   - 基础通道中的材质评估
   
5. [光照系统](chapters/05-lighting.md)
   - 延迟和前向光照
   - 光源类型及其实现
   
6. [阴影渲染](chapters/06-shadows.md)
   - 阴影贴图技术
   - 级联阴影贴图和虚拟阴影贴图
   
7. [后处理](chapters/07-postprocess.md)
   - 后处理管线架构
   - 色调映射、泛光和其他效果
   
8. [Lumen 全局光照](chapters/08-lumen.md)
   - 动态全局光照系统
   - Surface cache 和 Radiance cache
   
9. [Nanite 虚拟化几何](chapters/09-nanite.md)
   - 虚拟化几何渲染
   - 细节层次和剔除系统
   
10. [时间超分辨率 (TSR)](chapters/10-tsr.md)
    - 现代时间上采样
    - 历史重投影和抗锯齿

11. [Shader 编译 Job 调度与 Worker](chapters/11-shader-compile-worker.md)
    - ShaderCommonCompile 调度机制
    - ShaderCompileWorker 进程工作流程

## 如何使用本知识库

1. **从术语表开始** - 如果你对渲染术语不熟悉
2. **按顺序阅读章节** - 获得对管线的完整理解
3. **使用引用路径** - 调试时快速定位源代码
4. **每个章节都是独立的** - 可以跳转到感兴趣的主题

## 贡献

本知识库记录 UE5 渲染系统（基于 5.2+ 版本）。引擎快速演进，欢迎贡献和更新。

### 贡献指南

- 确保针对 UE5 源代码的技术准确性
- 包含来自引擎的代码示例
- 尽可能引用具体的源文件和行号
- 遵循现有的文档风格
- 当行为特定于某个版本时，请指定 UE 版本

## 额外资源

- [Unreal Engine 源代码](https://github.com/EpicGames/UnrealEngine)
- [官方 UE 文档](https://docs.unrealengine.com/)
- [Unreal Engine Graphics Programming Discord](https://discord.gg/unreal-slackers)
- [实时渲染资源](https://www.realtimerendering.com/)

## 许可证

本文档作为教育材料提供。Unreal Engine 是 Epic Games 的财产。

---

*部分章节正在持续更新中，标记为"待补充"的章节将逐步完善。*
