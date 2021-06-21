# rvbt: RISC-V 裸金属环境下的 Backtrace 与 Debug 辅助工具

本文主要介绍如何在 RISC-V 与 Rust 开发环境下为裸金属目标 (Baremetal) 提供堆栈追踪能力。

完整的工具 rvbt 在整理完代码后会开源公布在Github上。

## Part I: DWARF 信息与 rvbt 介绍

在 `gcc` 等编译器编译源代码时，如果加上 `-g` 标识，则会对应源代码生成对应的调试信息。
即 DWARF (Debugging With Arbitrary Record Formats)。
但该类信息通常只由 `gdb` 等调试器在调试时读取，而不会真正地在运行时被加载到内存里。
而裸金属机器在一些情况下可能不能很方便的接入 `gdb` 等调试器，例如 FPGA。
rvbt 正是为该种情况而设计的，运行在 RISC-V Machine Mode 权限级别下的 Backtrace 工具。

关于 DWARF 可以参考:

- [调试器工作原理：第三部分 调试信息](https://hanfeng.ink/post/gdb_debug_info/)
- [How debuggers work: Part 3 - Debugging information](https://eli.thegreenplace.net/2011/02/07/how-debuggers-work-part-3-debugging-information)
- [Wikipedia/DWARF](https://en.wikipedia.org/wiki/DWARF)

这里我们只需要将 DWARF 信息从原有的 ELF 文件中读取出来，作为我们自己的段插入二进制文件中。
并在运行时解析它们，在程序出现异常时解析并输出栈帧即可。

## Part II: DWARF 运行时加载

尽管调试信息在编译时已经整合进入了二进制文件，但我们如果像下面这样直接在代码中链接对应的符号是无法工作的。

```linkscript
/* THIS DOES NOT WORK */
SECTIONS
{
  /*...*/
  .debug_abbrev : {
    __my_debug_abbrev_start = .;
    KEEP (*(.debug_abbrev)) *(.debug_abbrev)
    __my_debug_abbrev_end = .;
  }
  /*...*/
}
```

```rust
extern "C" {
  fn __my_debug_abbrev_start();
  fn __my_debug_abbrev_end();
}
```

在链接阶段，你会收到下面的错误信息：

> xxx relocation at 0xdeadbeef for symbol `__my_debug_abbrev_start` out of range
