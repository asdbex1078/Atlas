---
name: ELF 安全机制（PIE/ASLR/NX/RELRO）
field: 计算机系统
type: 机制
aliases:
  - PIE
  - ASLR
  - RELRO
  - NX
  - BIND_NOW
  - CET
tags:
  - ELF
  - 安全
  - ASLR
  - PIE
  - RELRO
  - NX
desc: PIE+ASLR 随机基址、GNU_STACK 不可执行栈、RELRO 重定位后 GOT 只读、BIND_NOW 缩短 GOT 可写窗口、CET 控制流保护——全都靠 ELF 元数据表达
learned: 2026-09-24
sources:
  - "[[articles/计算机系统/ELF]]"
---
# ELF 安全机制（PIE/ASLR/NX/RELRO）

## 描述

现代 Linux 的二进制加固几乎都不是额外的机制，而是写在 ELF 元数据里、由加载器和动态链接器照着执行的约定。这也是为什么现代可执行文件的 `e_type` 通常是 `ET_DYN`：它们是 PIE，才能随机基址。

## 核心内容

- **PIE / ASLR**：可执行文件本身也编成位置无关，加载基址随机化；内部指针靠 `R_X86_64_RELATIVE` [[重定位]]修正
- **NX / GNU_STACK**：`PT_GNU_STACK` 标记栈为 RW、不可执行，栈上注入的 shellcode 跑不了（对应[[栈内存]]）
- **RELRO**：`PT_GNU_RELRO` 让 GOT 等区域在重定位完成后变只读
- **BIND_NOW**：立即绑定，所有符号启动时解析完，减少 GOT 可写的时间（配合 Full RELRO）
- **CET / GNU_PROPERTY**：`.note.gnu.property` / `PT_GNU_PROPERTY` 声明 IBT、SHSTK 等硬件控制流保护
- **符号可见性**：`-fvisibility=hidden` 减少导出符号
- **版本脚本**：控制符号导出与版本

`AT_RANDOM`（加载时放进 auxv 的随机数）用于栈保护等。

### 个人理解

这些机制和 [[动态链接]] 的延迟绑定天然冲突：延迟绑定需要 GOT 在运行期可写，而可写的函数指针表正是攻击面——所以 RELRO + BIND_NOW 是拿启动速度换安全。CSAPP Attack Lab 里的缓冲区溢出、NX、ASLR、RELRO 都和 ELF 的权限与布局有关。

## 参考资料

- CSAPP Attack Lab
- `readelf -l`（看 GNU_STACK / GNU_RELRO）、`readelf -d`（看 BIND_NOW / FLAGS）

## 待办

- [ ] 用 `checksec` 或 `readelf` 对比 `-z norelro`、`-z relro`、`-z relro -z now` 三种产物

## 关系
- 依赖:: [[ELF段]] — PT_GNU_STACK / PT_GNU_RELRO / PT_GNU_PROPERTY
- 相关:: [[动态链接]] — RELRO、BIND_NOW 限制的正是 GOT 的可写窗口
- 相关:: [[栈内存]] — NX：栈不可执行
