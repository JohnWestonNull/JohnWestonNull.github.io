## 基础知识——三种内存地址

在操作系统中，代表内存的方式分为 3 种，逻辑地址，虚拟地址和物理地址。

- 逻辑地址：由 segment selector(16 bits) 和 offset(32 bits) 构成，其中前者会被存储到处理器特有的寄存器当中，例如在 x86 架构中就分为 `cs`，`ss`，`ds` 等
- 虚拟地址：一个 32/64 bits 整数，最多可以有 $2^n$ 块内存单元。
- 物理地址：一个 32/36 bits 整数，用于定位内存芯片上的内存单元。

## 为什么我们需要内存寻址？
在计算机发展的早期，由于内存大小限制等因素影响，计算机大多数只支持简单的线性地址寻址。
在早期实模式下，内存地址的格式通常为 （段基址：段偏移）,其中段基址是一个 16 位的值，存储在段寄存器中，段偏移也是一个 16 位的值，存储在通用寄存器中。将段基址左移 4 位后加上偏移，两者结合构成一个 20 位的地址。

> 例如，若段基址的值为 `0xFF00`，段偏移的值为 `0x0123`，那么最后的地址就是 `0xFF00 << 4 + 0x0123 = 0xFF123`

在这种模式下，逻辑地址简单的加上一个偏移就是物理地址。这样做当然有好处，就是便于理解。但是缺点也很多，例如没有基本的保护机制，内存越界可能导致整个系统崩溃，也无法运行两个占据同一块空间的程序。

例如下图中，右边的 Program A 和 Program B 都想要 `0x0-0x100` 的内存空间，但实际物理地址上只有一块该地址的空间。而在左边，一个名字叫 Bad 的程序正在尝试更改 Kernel 内核所正在使用的内存。
![yz5DYj.png](https://s3.ax1x.com/2021/02/26/yz5DYj.png)

如果有一个机制能够让不同的程序看到的地址相同，但对应的物理地址不同就好了，例如，我们可以将上面 Program A 的 `0x0-0x100` 映射到实际的 `0xA000-0xA100`，将 Program B 的 `0x0-0x100` 映射到实际的 `0xB000-0xB100`。

对于另一个问题，我们也可以设计一套权限机制，实现对于不同的映射，有着不同的权限管理，例如，对于 Kernel 的高地址代码段，将映射的权限设置成普通用户无法访问，这样 Bad 程序就无法随意修改 Kernel 的内存内容了。

## 虚拟地址到物理地址的转换——页表

## RISC-V 架构下的内存寻址
在 RISC-V 架构下，并没有分段的概念，所有由虚拟地址到物理地址的映射均写入 `satp` 寄存器，由硬件 mmu 完成转换。


## Intel x86 架构下的内存寻址

Intel x86 架构下的内存寻址分为两个步骤，segmentation(分段) 与 paging(分页)。

目前 Linux 操作系统主要依赖分页来对内存区域进行划分，因为分段和分页的功能上实际上是重复的，而分段在一些新的 ISA 上并没有被完善的支持（例如 RISC-V 只支持分页）。

### 分页
**等待更新**


