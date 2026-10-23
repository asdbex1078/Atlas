---
name: ELF（可执行与可链接格式）
field: 计算机系统
type: 概念
aliases:
  - Executable and Linkable Format
  - ELF 文件
tags:
  - 二进制格式
  - 链接
  - 加载
  - Linux
  - CSAPP
desc: Unix/Linux 最核心的二进制容器格式：目标文件、可执行文件、共享库、core dump、内核模块、eBPF 对象都用它；核心设计是链接视图（节）与执行视图（段）分离
learned: 2026-09-24
sources:
  - "[[articles/计算机系统/ELF]]"
---
# ELF（可执行与可链接格式）

## 描述

ELF（Executable and Linkable Format，可执行与可链接格式）是 Unix/Linux 世界最核心的二进制文件格式。它不只是「可执行文件格式」，还是目标文件、共享库、核心转储、内核模块、动态插件、eBPF 对象共用的容器。理解 ELF，就是理解**程序怎么从磁盘上的文件变成进程地址空间里的代码和数据**。

一句话定位：**ELF 是链接器和加载器之间的契约。**

## 核心内容

（全景——ELF 用在哪些地方、对操作系统和 CSAPP 学习的启发、实践工具清单——见原文概述 `articles/计算机系统/ELF.md`，按阅读顺序串起了 ELF 这一组点。）

### 核心设计：两种视图

- **链接视图**：以「节（section）」为单位，给链接器、调试器、静态分析工具看 → [[ELF节]]
- **执行视图**：以「段（segment）」为单位，给操作系统加载器和动态链接器看 → [[ELF段]]

节头表可以被 `strip` 删掉，程序照样能跑；程序头表对可执行文件和共享库的加载必不可少。

### 顶层结构

```text
+----------------------+
| ELF Header           |  e_ident, e_type, e_machine, e_entry, e_phoff, e_shoff ...
+----------------------+
| Program Header Table |  描述「段」，加载器使用
+----------------------+
| Sections / Segments  |  实际代码、数据、符号、重定位、调试信息
+----------------------+
| Section Header Table |  描述「节」，链接器/调试器使用
+----------------------+
```

ELF 头位于文件开头，关键字段：

- `e_ident`：魔数 `0x7F 'E' 'L' 'F'`，以及位数、字节序、OSABI 等
- `e_type`：文件类型
- `e_machine`：目标架构
- `e_entry`：入口虚拟地址
- `e_phoff` / `e_shoff`：程序头表 / 节头表在文件中的偏移
- `e_phentsize` / `e_phnum`、`e_shentsize` / `e_shnum`：两张表的表项大小与数量
- `e_shstrndx`：节名字符串表索引

### 文件类型（`e_type`）

| 类型 | 值 | 含义 | 典型文件 |
|---|---:|---|---|
| `ET_NONE` | 0 | 未知 | 很少见 |
| `ET_REL` | 1 | 可重定位目标文件 | `.o` |
| `ET_EXEC` | 2 | 可执行文件，固定地址 | 传统非 PIE 可执行文件 |
| `ET_DYN` | 3 | 共享目标文件 / PIE 可执行文件 | `.so`、PIE 程序 |
| `ET_CORE` | 4 | 核心转储 | `core` |

注意：现代 Linux 的可执行文件通常是 `ET_DYN`，也就是 PIE，为了支持 ASLR（见 [[ELF安全机制]]）。

## 例子

（补充）同一个 hello world，用 `readelf -h` 看 ELF 头，PIE 和非 PIE 的差别就在 `e_type` 上：

```sh
$ gcc hello.c -o hello_pie                # 现代发行版默认就是 PIE
$ readelf -h hello_pie | grep -E 'Magic|Class|Type|Entry'
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Type:                              DYN (Position-Independent Executable file)
  Entry point address:               0x1060

$ gcc -no-pie hello.c -o hello_nopie
$ readelf -h hello_nopie | grep Type
  Type:                              EXEC (Executable file)
```

（补充）怎么读：

- `Magic` 开头 4 个字节 `7f 45 4c 46` 就是 `0x7F 'E' 'L' 'F'`；第 5 个字节 `02` 表示 64 位（`e_ident[EI_CLASS]`），第 6 个 `01` 表示小端。
- PIE 是 `ET_DYN`，入口地址 `0x1060` 很小——它是相对加载基址的偏移，真正的地址要等 ASLR 选好基址才定（见 [[ELF安全机制]]）；较老的 binutils 会把它显示成 `DYN (Shared object file)`，`file` 命令则显示 `pie executable`。
- 非 PIE 是 `ET_EXEC`，入口地址是 `0x401xxx` 这样的固定虚拟地址，加载器照这个地址映射（见 [[ELF加载过程]]）。

## 参考资料

- System V ABI, ELF 规范：https://refspecs.linuxfoundation.org/elf/gabi4+/ch4.intro.html
- TIS ELF Specification 1.2：https://refspecs.linuxfoundation.org/elf/elf.pdf
- `elf(5)`：https://man7.org/linux/man-pages/man5/elf.5.html
- CSAPP 第 7 章链接、第 9 章虚拟内存
- Ian Lance Taylor, *Linkers and Loaders*
- Wikipedia, *Executable and Linkable Format*
- GNU Binutils BFD 文档：https://sourceware.org/binutils/docs/bfd/

## 待办

- [ ] 拿一个 hello world 分别编成非 PIE / PIE，用 `readelf -h` 对比 `e_type`
- [ ] 对照 CSAPP 第 7 章把 `.c → .s → .o → a.out` 每一步的产物跑一遍

## 关系
- 包含:: [[ELF节]] — 链接视图
- 包含:: [[ELF段]] — 执行视图
- 产出自:: [[汇编器]] — 汇编器把 .s 翻成 ET_REL 目标文件 .o
- 相关:: [[eBPF]] — eBPF 程序编译成 ELF 对象（程序、map、BTF），由 libbpf 加载进内核
- 相关:: [[JIT]] — LLVM ORC / MCJIT 生成 ELF 对象在运行时加载，数据库表达式 JIT 常用
