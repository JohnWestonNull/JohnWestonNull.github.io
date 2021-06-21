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

这是因为在编译阶段，类似于 `auipc` 等基于当前 `pc` 寄存器寻址的命令的操作数都会为 0, 在链接阶段再由链接器填入。
而 `extern "C"` 引用的符号正是以此机制链入的，而上面的 DWARF 段距离 `pc` 的距离过远，该寻址手段最多只支持 `+-2M` 范围内的寻址。
而调试信息由于运行时不需要，通常都会被编译器放置在较远的位置，因此当链接器发现需要链接的符号过远而无法链接时就会报上面的错误。

**解决方案: 将调试信息复制一份置入自定义段，让自定义段距离 `.rodata` 段足够近即可。**

遗憾的是，根据下面的描述 [StackOverflow](https://stackoverflow.com/questions/31125217/how-to-make-duplicate-sections-in-elf-file)，链接脚本并不支持**复制**这种操作。

> If a file name matches more than one wildcard pattern, or
> if a file name appears explicitly and is also matched by a wildcard pattern,
> the linker will use the first match in the linker script.

链接脚本在任何时候只会根据优先匹配原则将一段放入一块区域中。 但我们可以利用 `objcopy` 来实现对段的操作。

为了完成复制段的操作，我们首先需要在链接脚本中留出足够的空间，可以利用下面的方法。

```linkscript
  .rvbt_addr : {
    _rvbt_addr_start = .;
    . += _debug_addr_end - _debug_addr_start;
    _rvbt_addr_end = .;
  }
```

然后我们先将 `.debug_{section}` 的段全部导出到临时文件中，再重新导入进我们将在运行时读取的段 `.rvbt_{section}`。

```bash
copy-debug:
  for sec in 'abbrev' 'addr' 'aranges' 'info' 'line' 'line_str' 'ranges' 'rnglists' 'str' 'str_offsets'; do \
    # dump needed section
    rust-objcopy {{PAYLOAD}} --dump-section .debug_$sec=tmp_$sec; \
    # update
    riscv64-unknown-elf-objcopy {{PAYLOAD}} --update-section .rvbt_$sec=tmp_$sec; \
  done
  rm tmp*; 
```

这样完成后的 ELF 中就可以直接访问到链接脚本中定义的符号了。
