---
name: ELF 加载过程（execve）
field: 计算机系统
type: 机制
aliases:
  - execve 加载
  - 程序加载
  - binfmt_elf
tags:
  - ELF
  - execve
  - 加载器
  - 进程启动
  - CSAPP
desc: execve 时内核读 ELF 头和程序头表，把每个 PT_LOAD 映射进虚拟内存、多出的部分清零、设权限，加载 PT_INTERP 指定的动态链接器并构建带 auxv 的用户栈，再把控制权交出去
learned: 2026-09-24
sources:
  - "[[articles/计算机系统/ELF]]"
---
# ELF 加载过程（execve）

## 描述

一个 ELF 文件怎么变成进程：以 Linux `execve` 为例，内核负责「映射 + 建栈 + 交控制权」，复杂的符号解析交给用户态的动态链接器。内核实现在 `fs/binfmt_elf.c`。进程启动模型是 `fork` + `execve`：`execve` 用新程序替换掉当前地址空间。

## 核心内容

### 步骤

1. 用户调用 `execve(path, argv, envp)`
2. 内核打开文件，读取 ELF 头，检查魔数、架构、类型
3. 内核读取程序头表（[[ELF段]]）
4. 对每个 `PT_LOAD` 段：
   - 把文件偏移 `p_offset` 处的内容映射到虚拟地址 `p_vaddr`
   - 若 `p_filesz < p_memsz`，多余内存清零——这就是 `.bss`
   - 设置权限：R、W、X
5. 若存在 `PT_INTERP`，内核读取动态链接器路径并加载动态链接器
6. 内核构建用户栈：`argc`、`argv`、`envp`、`auxv`
   - `AT_PHDR`：程序头表地址
   - `AT_ENTRY`：入口地址
   - `AT_BASE`：动态链接器基址
   - `AT_RANDOM`：随机数，用于栈保护等
   - `AT_SYSINFO_EHDR`：vDSO 地址
7. 控制权交给动态链接器，或（静态程序）直接交给入口 `e_entry`
8. 动态链接器（见 [[动态链接]]）：
   - 解析 `.dynamic`
   - 加载 `DT_NEEDED` 依赖库
   - 执行重定位、解析符号
   - 运行 `.init_array` / `.init`
   - 跳转到 `main`
9. 程序退出时运行 `.fini_array` / `.fini`

动态链接器本身也是 ELF 共享对象，通常路径为 `/lib64/ld-linux-x86-64.so.2`。

### 个人理解

「程序先存硬盘、加载到内存后被 CPU 执行」这句话的细节全在这里：加载不是把整个文件拷进内存，而是 mmap 映射，页真正被访问时才按需调入。`main` 并不是第一条被执行的指令——前面还有动态链接器和 `.preinit_array` / `.init_array`。

## 参考资料

- `execve(2)`：https://man7.org/linux/man-pages/man2/execve.2.html
- Linux 内核 ELF 加载器源码 `fs/binfmt_elf.c`：https://elixir.bootlin.com/linux/latest/source/fs/binfmt_elf.c
- LWN, *How programs get run: ELF binaries*：https://lwn.net/Articles/631631/
- CSAPP 第 8 章异常控制流、第 9 章虚拟内存

## 待办

- [ ] 用 `LD_SHOW_AUXV=1 /bin/true` 看一眼 auxv 里的这些 AT_* 值
- [ ] 读 `binfmt_elf.c` 里 `load_elf_binary` 的主干

## 关系
- 依赖:: [[ELF段]] — 内核只读程序头表
- 包含:: [[动态链接]] — 第 8 步由动态链接器完成
- 属于:: [[OS]] — 内核的 ELF 加载器（binfmt_elf）
- 产出:: [[程序的内存布局]] — 加载完成后才有进程地址空间里的代码/数据/栈
