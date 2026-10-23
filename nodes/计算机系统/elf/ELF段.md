---
name: ELF 段（Segment）：执行视图
field: 计算机系统
type: 概念
aliases:
  - Segment
  - 程序头表
  - Program Header
  - 执行视图
tags:
  - ELF
  - 加载
  - 程序头表
  - 虚拟内存
desc: ELF 按「段」划分的执行视图：程序头表里的 PT_LOAD 等条目告诉内核把文件哪部分以什么权限映射到哪个虚拟地址；p_filesz < p_memsz 的差额就是清零的 .bss
learned: 2026-09-24
sources:
  - "[[articles/计算机系统/ELF]]"
---
# ELF 段（Segment）：执行视图

## 描述

内核加载器不关心「节」，只关心「段」。段由程序头表描述，是 ELF 的**执行视图**：给操作系统加载器和动态链接器看。对可执行文件和共享库，程序头表必不可少（和 [[ELF节]] 可被 strip 相反）。一个段通常装着若干个节，`readelf -l` 会打印「Section to Segment mapping」。

## 核心内容

### 程序头表项字段

- `p_type`：段类型，如 `PT_LOAD`、`PT_DYNAMIC`
- `p_flags`：权限，`PF_R`、`PF_W`、`PF_X`
- `p_offset`：文件偏移
- `p_vaddr`：虚拟地址
- `p_filesz`：文件中大小
- `p_memsz`：内存中大小
- `p_align`：对齐

`p_filesz < p_memsz` 的典型例子是 `.bss`：文件里不占空间，加载时清零。

### 常见段类型

| 段类型 | 作用 |
|---|---|
| `PT_LOAD` | 可加载段，映射到[[虚拟内存]]，权限由 `p_flags` 决定 |
| `PT_DYNAMIC` | 指向 `.dynamic`，动态链接信息 |
| `PT_INTERP` | 动态链接器路径 |
| `PT_NOTE` | 注释信息，如 build-id、ABI、属性 |
| `PT_PHDR` | 程序头表自身在内存中的位置 |
| `PT_TLS` | 线程局部存储模板 |
| `PT_GNU_EH_FRAME` | 异常处理帧信息 |
| `PT_GNU_STACK` | 标记栈权限，通常 RW、不可执行 |
| `PT_GNU_RELRO` | 重定位后只读区域 |
| `PT_GNU_PROPERTY` | 安全属性 |
| `PT_SHLIB` | 保留，很少使用 |
| `PT_SUNWBSS` / `PT_SUNWSTACK` | Solaris 平台相关 |

### 一个典型动态可执行文件的段

- 一个 `PT_LOAD`：R + X，映射代码
- 一个 `PT_LOAD`：R，映射只读数据
- 一个 `PT_LOAD`：RW，映射数据、GOT、动态段
- `PT_INTERP`：指定动态链接器
- `PT_DYNAMIC`：动态链接信息
- `PT_GNU_RELRO`：把 GOT 等区域在重定位后设为只读
- `PT_GNU_STACK`：不可执行栈

### 个人理解

节按「用途」分（符号表、字符串表、重定位表……），段按「权限」分（RX / R / RW）——加载器只需要知道哪块内存该有什么权限，所以几十个节被合并成三四个 `PT_LOAD`。这也是[[程序的内存布局]]里代码段、数据段在运行时的真正来源；栈和堆则是加载时/运行时另行建立的。

## 参考资料

- `readelf -l a.out` 查看段与节到段的映射
- LWN, *How programs get run: ELF binaries*：https://lwn.net/Articles/631631/

## 待办

- [ ] `readelf -l` 看一个动态程序，数它有几个 `PT_LOAD`、各是什么权限
- [ ] 对照 `/proc/<pid>/maps` 验证每个 `PT_LOAD` 映射到了哪里

## 关系
- 依赖:: [[虚拟内存]] — PT_LOAD 直接映射到虚拟地址，按需分页、写时复制
- 支撑:: [[程序的内存布局]] — 代码段、数据段在运行时的实际来源是 PT_LOAD 映射
