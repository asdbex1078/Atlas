---
name: ELF 节（Section）：链接视图
field: 计算机系统
type: 概念
aliases:
  - Section
  - 节头表
  - 链接视图
tags:
  - ELF
  - 链接
  - 节头表
  - 符号表
desc: "ELF 按「节」划分的链接视图：.text/.data/.bss/.rodata 装代码数据，.symtab/.rela.* 服务链接，.dynamic/.got/.plt 服务动态链接；给链接器、调试器看，可被 strip"
learned: 2026-09-24
sources:
  - "[[articles/计算机系统/ELF]]"
---
# ELF 节（Section）：链接视图

## 描述

节是 ELF 里最丰富的区域划分，是**链接视图**：给[[链接器]]、调试器、静态分析工具看。节由节头表描述，节头表可以被 `strip` 删掉而程序仍能运行——内核加载器根本不看节，只看 [[ELF段]]。

## 核心内容

### 节头表项字段

- `sh_name`：节名在 `.shstrtab` 中的偏移
- `sh_type`：节类型
- `sh_flags`：可写、可执行、是否分配内存
- `sh_addr`：加载后的虚拟地址
- `sh_offset`：文件偏移
- `sh_size`：大小
- `sh_link` / `sh_info`：关联信息
- `sh_addralign`：对齐
- `sh_entsize`：表项大小

### 代码与数据

| 节名 | 类型 | 典型标志 | 作用 |
|---|---|---|---|
| `.text` | `PROGBITS` | `AX` | 机器指令（即 [[代码段]]） |
| `.data` | `PROGBITS` | `WA` | 已初始化的全局/静态变量（即 [[数据段]]） |
| `.bss` | `NOBITS` | `WA` | 未初始化全局/静态变量，文件不占空间，加载清零 |
| `.rodata` | `PROGBITS` | `A` | 只读常量、字符串字面量、跳转表 |
| `.comment` | `PROGBITS` | `MS` | 编译器版本信息 |

### 链接与符号

| 节名 | 类型 | 作用 |
|---|---|---|
| `.symtab` | `SYMTAB` | 静态符号表，链接器使用，可被 strip |
| `.strtab` | `STRTAB` | `.symtab` 的字符串表 |
| `.shstrtab` | `STRTAB` | 节名字符串表 |
| `.rel.*` / `.rela.*` | `REL` / `RELA` | 重定位信息（见 [[重定位]]） |
| `.debug_*` | `PROGBITS` | DWARF 调试信息 |
| `.line` | `PROGBITS` | 行号信息，旧格式 |
| `.gnu_debuglink` | `PROGBITS` | 指向分离的调试文件 |
| `.note.gnu.build-id` | `NOTE` | 构建 ID，用于调试与符号定位 |

### 动态链接（详见 [[动态链接]]）

| 节名 | 类型 | 作用 |
|---|---|---|
| `.interp` | `PROGBITS` | 动态链接器路径，如 `/lib64/ld-linux-x86-64.so.2` |
| `.dynamic` | `DYNAMIC` | 动态链接所需标签数组 |
| `.dynsym` | `DYNSYM` | 动态符号表 |
| `.dynstr` | `STRTAB` | 动态字符串表 |
| `.hash` / `.gnu.hash` | `HASH` / `GNU_HASH` | 符号哈希表，加速符号查找 |
| `.rela.dyn` | `RELA` | 数据重定位 |
| `.rela.plt` | `RELA` | PLT 重定位 |
| `.got` / `.got.plt` | `PROGBITS` | 全局偏移表 |
| `.plt` / `.plt.got` / `.plt.sec` | `PROGBITS` | 过程链接表，外部函数跳转桩 |
| `.gnu.version` | `VERSYM` | 符号版本索引 |
| `.gnu.version_d` | `VERDEF` | 版本定义 |
| `.gnu.version_r` | `VERNEED` | 版本需求 |

### 初始化、异常与 TLS

| 节名 | 类型 | 作用 |
|---|---|---|
| `.init` / `.fini` | `PROGBITS` | 旧式初始化和析构代码 |
| `.init_array` | `INIT_ARRAY` | 构造函数指针数组 |
| `.fini_array` | `FINI_ARRAY` | 析构函数指针数组 |
| `.preinit_array` | `PREINIT_ARRAY` | main 之前最早执行的初始化 |
| `.eh_frame` | `PROGBITS` | 异常处理与栈展开信息 |
| `.eh_frame_hdr` | `PROGBITS` | `.eh_frame` 的索引 |
| `.gcc_except_table` | `PROGBITS` | C++ 异常表 |
| `.tdata` | `PROGBITS` | 线程局部存储的已初始化数据 |
| `.tbss` | `NOBITS` | 线程局部存储的未初始化数据 |

### 安全与属性

| 节名 | 作用 |
|---|---|
| `.note.gnu.property` | CET、IBT、SHSTK 等硬件安全属性 |
| `.note.ABI-tag` | ABI 版本标签 |
| `.gnu.hash` | GNU 扩展哈希 |
| `.gnu.version_r` | 依赖库符号版本 |

### 个人理解

汇编里写的 `.text` / `.data` 伪指令，最终落成的就是这里的同名节；`.bss` 是 `NOBITS`，这就是为什么大数组不初始化时可执行文件不会变大。

## 参考资料

- MaskRay, *ELF section types*：https://maskray.me/blog/2021-01-17-elf-section-types
- `readelf -S a.out` 查看节头

## 待办

- [ ] 写一个带全局未初始化大数组的程序，对比 `.bss` 在 `readelf -S` 里的 size 和文件大小

## 关系
- 对比:: [[ELF段]] — 链接视图 vs 执行视图
- 包含:: [[代码段]] — .text 节
- 包含:: [[数据段]] — .data 节
- 相关:: [[链接器]] — 节是给链接器看的
