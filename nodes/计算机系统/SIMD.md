---
name: SIMD
field: 计算机系统
type: 机制
year: 1997
aliases:
  - Single Instruction Multiple Data
  - MMX
  - SSE
  - AVX
tags:
  - 指令集扩展
  - MMX
  - SSE
  - AVX
  - 向量化
desc: 单指令多数据：一条指令同时处理一组数据。x86 上一路从 MMX（64 位）到 SSE（128 位 XMM）、AVX（256 位 YMM）再到 AVX-512，宽度翻了又翻
learned: 2026-10-12
sources:
  - articles/硬件/CPU
layer: 体系结构
---
# SIMD

## 描述

SIMD（Single Instruction, Multiple Data）：一条指令对一个寄存器里打包的多个数据同时做同一种运算。图像、音频、矩阵运算里大量「对每个元素做同样的事」的循环，用 SIMD 可以几倍几十倍地加速。

## 核心内容

x86 上的演进（原文）：

| 扩展 | 首发 | 寄存器 | 能力 |
|---|---|---|---|
| **MMX** | 1997 | 64 位 MMX 寄存器 | 多媒体整数 SIMD |
| **SSE** | Pentium III（1999） | 128 位 XMM | 单精度浮点 SIMD |
| **SSE2** | Pentium 4（2000） | XMM | 扩展到双精度浮点和整数 |
| **AVX** | Sandy Bridge（2011） | 256 位 YMM | 每周期可处理 8 个单精度浮点数 |

- 64 位扩展时又新增了 8 个 128 位 SSE 寄存器 XMM8～XMM15（见 [[x86-64]]）。
- Ice Lake / Tiger Lake 进一步集成了 AI 加速指令（如 DP4A）。
- （补充）**MMX 的坑**：MMX 寄存器和 x87 浮点寄存器是同一组物理寄存器，用完 MMX 必须执行 `EMMS` 才能再做浮点运算——这也是 SSE 改用独立 XMM 寄存器的原因。
- （补充）**ARM 上对应的是 NEON**（128 位）和 SVE / SVE2（向量长度可变）；GPU 则把「同一条指令处理一大批数据」推到了极致（SIMT），见 [[GPU]]。

## 例子

- （补充）两个 float 数组逐元素相加 `c[i] = a[i] + b[i]`：标量代码一次加 1 个；SSE 的 `addps` 一次加 4 个；AVX 的 `vaddps ymm` 一次加 8 个。现代编译器开 `-O3` 会自动把这种循环向量化。

## 关系
- 属于:: [[x86]] — x86 的 MMX / SSE / AVX 扩展
- 对比:: [[GPU]] — GPU 把数据并行推到极致
