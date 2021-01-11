# 基础知识——三种内存地址

在操作系统中，代表内存的方式分为 3 种，逻辑地址，虚拟地址和物理地址。

- 逻辑地址：由 segment selector(16 bits) 和 offset(32 bits) 构成，其中前者会被存储到处理器特有的寄存器当中，例如在 x86 架构中就分为 `cs`，`ss`，`ds` 等
- 虚拟地址：一个 32/64 bits 整数，最多可以有 $2^n$ 块内存单元。
- 物理地址：一个 32/36 bits 整数，用于定位内存芯片上的内存单元。

# Intel x86 架构下的内存寻址 (Memory Addressing in x86)

Intel x86 架构下的内存寻址分为两个步骤，segmentation(分段) 与 paging(分页)。

目前 Linux 操作系统主要依赖分页来对内存区域进行划分，因为分段和分页的功能上实际上是重复的，而分段在一些新的 ISA 上并没有被完善的支持（例如 RISC-V 只支持分页）。

### 分页
