---
imported: 2026-10-13
origin: "由 2026-09-24 导入的 ELF 那组知识点整理成的概述（当时原文没保留）"
---
# ELF 概述

ELF（Executable and Linkable Format，可执行与可链接格式）是 Unix/Linux 世界最核心的二进制文件格式。它不只是「可执行文件格式」，还是目标文件、共享库、核心转储、内核模块、动态插件、eBPF 对象共用的容器。理解 ELF，就是理解**程序怎么从磁盘上的文件变成进程地址空间里的代码和数据**。

一句话定位：**ELF 是链接器和加载器之间的契约。**

## 阅读顺序

细节都在知识点里，这里只给一条串起来读的路线：

1. [[ELF]]：是什么、链接视图与执行视图、ELF 头、文件类型（`e_type`）
2. [[ELF节]]：链接视图——`.text/.data/.bss/.rodata`、符号表、重定位节、动态链接用的节
3. [[ELF段]]：执行视图——程序头表、`PT_LOAD` 怎么映射进虚拟内存
4. [[重定位]]：编译时不知道地址，链接 / 加载时补上
5. [[动态链接]]：动态链接器、GOT/PLT、延迟绑定、符号版本；例子里有数据库 C 扩展怎么 `dlopen`
6. [[ELF加载过程]]：`execve` 时内核和动态链接器各做什么
7. [[ELF安全机制]]：PIE / ASLR / NX / RELRO / CET 怎么靠 ELF 元数据表达

## 来源与支持范围

ELF 源自 System V ABI，后来由 TIS（Tool Interface Standard）标准化。它支持：

- 32 位 / 64 位
- 小端 / 大端
- 多种 CPU 架构：x86、x86-64、ARM、AArch64、RISC-V、MIPS、PowerPC 等
- 多种文件类型：目标文件、可执行文件、共享库、core dump

## 用在哪里

1. 可执行文件：`/bin/ls`、`/usr/bin/python`
2. 共享库：`libc.so.6`、`libssl.so`、`libstdc++.so`
3. 目标文件：`.o`，编译器（[[汇编器]]）输出、链接器输入
4. 静态库：`.a` 是 `ar` 归档，里面装多个 `.o`
5. 核心转储：`core` 文件，内存快照 + note，用于事后调试
6. 内核模块：Linux `.ko`，模块加载器解析符号和重定位
7. 内核镜像：`vmlinux` 是 ELF，`bzImage` 是压缩引导镜像
8. Android Native 库：`.so`
9. [[eBPF]] 对象：BPF 程序编译为 ELF，由 libbpf 加载
10. [[JIT]] 目标文件：LLVM ORC、MCJIT 可生成 ELF 对象并加载——ELF 在这里是「运行时生成的代码容器」
11. 数据库插件：PostgreSQL、MySQL、SQLite 扩展动态库（见 [[动态链接]] 的例子）
12. 调试符号文件：`.debug`、build-id 对应文件
13. 容器镜像里的二进制：几乎全是 ELF
14. 嵌入式与 RTOS：很多嵌入式 Linux、QNX、VxWorks

## 对操作系统的启发

1. **统一加载接口**：内核只实现一个 ELF 加载器，就能支持可执行文件、共享库、PIE、core
2. **链接与加载分离**：编译时生成可重定位信息，加载时由动态链接器完成地址绑定
3. **虚拟内存映射**：`PT_LOAD` 直接映射到虚拟地址，支持按需分页、写时复制、mmap
4. **动态链接器用户态化**：内核只负责加载解释器，复杂的符号解析放在用户态
5. **进程启动模型**：`fork` + `execve`，`execve` 替换地址空间
6. **模块化**：内核模块 `.ko` 用 ELF，支持动态加载、符号导出、重定位
7. **安全机制**：PIE、ASLR、NX、RELRO、CET 都依赖 ELF 元数据
8. **核心转储**：core 是 ELF
9. **可扩展性**：节/段类型、NOTE、GNU 扩展让格式持续演进
10. **ABI 稳定性**：符号版本、SONAME、动态标签支撑长期二进制兼容

## 对 CSAPP 学习的启发

CSAPP 第 7 章「链接」和第 9 章「虚拟内存」几乎就是 ELF 的教科书：

- **第 7 章 链接**：`.c` → `.s` → `.o` → 可执行文件；`ET_REL` 目标文件、符号表、重定位；静态/动态链接、共享库；GOT/PLT、位置无关代码；库打桩、LD_PRELOAD
- **第 9 章 虚拟内存**：ELF 段映射到虚拟地址；`mmap`、按需分页、写时复制；加载器怎么把可执行文件变成进程
- **第 8 章 异常控制流**：`fork`、`execve`、进程上下文
- **第 3 章 机器级表示**：`.text` 里的机器码、汇编、反汇编
- **第 5 章 优化**：PIC 开销、内联、符号可见性
- **安全实验**：Attack Lab 的缓冲区溢出、NX、ASLR、RELRO 都和 ELF 权限与布局有关

作者原话：读懂 ELF，CSAPP 的链接、加载、虚拟内存、动态库章节会从「概念」变成「可验证的工程现实」。

## 实践工具

```sh
file a.out
readelf -h a.out          # ELF 头
readelf -l a.out          # 程序头 / 段
readelf -S a.out          # 节头 / 节
readelf -s a.out          # 符号表
readelf -d a.out          # 动态段
readelf -r a.out          # 重定位
objdump -d a.out          # 反汇编
objdump -t a.out          # 符号表
objdump -s -j .rodata a.out
nm a.out
ldd a.out
patchelf --print-interpreter a.out
strip a.out
gdb a.out
cat /proc/$(pidof a.out)/maps
LD_DEBUG=libs ./a.out
LD_PRELOAD=./hook.so ./a.out
```

这些工具能把 ELF 的节、段、符号、重定位一一对应起来。
