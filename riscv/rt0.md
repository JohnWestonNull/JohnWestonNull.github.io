# Coffer 移植哪吒 D1 开发板小记

近日为哪吒 D1 移植 RISC-V 可信执行环境 Coffer，一些值得注意的点，写了一个小记。

## D1 SDK OpenSBI 打包位置

对于 D1 开发板，全志定制了一整套工具链以支持平头哥自研的一些指令，幸运的是，其对于 SBI 的 OpenSBI 部分并没有使用定制指令，
而是使用普通的 RISC-V 工具链（似乎用定制工具链编译不过去xs）。
因为采用普通工具链的缘故，官方 `make` 脚本甚至不会检查，而是直接从 OpenSBI 的文件夹复制过去。（还复制了好几回，无语。）

OpenSBI 中最初的编译产物位置

`lichee/brandy-2.0/opensbi/build/platform/thead/c910/firmware`

第一次复制位置

`device/config/chips/d1/bin/opensbi_sun20iw1p1.bin`

第二次复制位置

`out/d1_nezha/image/opensbi.fex`

一般来说复制到第一次复制的目录，然后执行 `pack` 就好，`pack` 会执行第二次复制。

## 嵌入式程序初始化

由于 QEMU 这类的模拟器会提供默认的寄存器初始化与内存初始化等功能，因此不需要类似的初始化流程。 而嵌入式主板不同，因此需要一套简单的流程让从 Flash 闪存中加载的程序能够正常运行。

### 寄存器初始化

```asm
  li sp, 0
  li gp, 0
  li tp, 0
  li t0, 0
  li t1, 0
  li t2, 0
  li s0, 0
  li s1, 0
  li a3, 0
  li a4, 0
  li a5, 0
  li a6, 0
  li a7, 0
  li s2, 0
  li s3, 0
  li s4, 0
  li s5, 0
  li s6, 0
  li s7, 0
  li s8, 0
  li s9, 0
  li s10, 0
  li s11, 0
  li t3, 0
  li t4, 0
  li t5, 0
  li t6, 0
  csrw mscratch, 0
```

### 清空 `.bss` 段和初始化 `.data` 段

从内存中复制过来的 `.bss` 段不确保完全为 0，直接使用会有各种奇怪的问题。

每个 LOADABLE/ALLOCATABLE 段通常有两个地址，VMA（运行时虚拟地址） 与 LMA（加载时地址），两者在嵌入式环境不一定相等，可能需要重定位。

利用 Rust 的 `r0` 库直接搞定。

```rust
extern "C" {
        static mut _bss_start: u32;
        static mut _bss_end: u32;
        static mut _data_start: u32;
        static mut _data_end: u32;
        static _flash_data: u32;
    }
    unsafe {
        r0::zero_bss(&mut _bss_start, &mut _bss_end);
        r0::init_data(&mut _data_start,&mut _data_end , &_flash_data);
    }
```

### 全志自定义 Header

全志非常有创意的定制了一块自定义 Header，我也不知道有啥用。 只能原样在 Coffer 中重构了一个一样的以防止出现问题，根据注释是放置设备树等信息的位置。

```c
// lichee/brandy-2.0/opensbi/platform/thead/c910/private_opensbi.h
struct private_opensbi_head {
        unsigned int  jump_instruction; /* jumping to real code */
        unsigned char magic[8];                 /* ="opensbi" */
        unsigned int  dtb_base;                 /* the address of dtb base*/
        unsigned int  uboot_base;               /* the address of dtb base*/
        unsigned int  res3;
        unsigned int  res4;
        unsigned char res5[8];
        unsigned char res6[8];
        unsigned int opensbi_base;              /* the address of opensbi base*/
};

// lichee/brandy-2.0/opensbi/platform/thead/c910/opensbi_head.c
#define BROM_FILE_HEAD_SIZE             (0x400 & 0x00FFFFF)
#define BROM_FILE_HEAD_BIT_10_1         ((BROM_FILE_HEAD_SIZE & 0x7FE) >> 1)
#define BROM_FILE_HEAD_BIT_11           ((BROM_FILE_HEAD_SIZE & 0x800) >> 11)
#define BROM_FILE_HEAD_BIT_19_12        ((BROM_FILE_HEAD_SIZE & 0xFF000) >> 12)
#define BROM_FILE_HEAD_BIT_20           ((BROM_FILE_HEAD_SIZE & 0x100000) >> 20)

#define BROM_FILE_HEAD_SIZE_OFFSET      ((BROM_FILE_HEAD_BIT_20 << 31) | \
                                        (BROM_FILE_HEAD_BIT_10_1 << 21) | \
                                        (BROM_FILE_HEAD_BIT_11 << 20) | \
                                        (BROM_FILE_HEAD_BIT_19_12 << 12))
#define JUMP_INSTRUCTION                (BROM_FILE_HEAD_SIZE_OFFSET | 0x6f)
// 这里直接硬编码了一个 RISC-V 的跳转指令
// 0x400 是该自定义头的大小
// j 0x4000_0400

struct private_opensbi_head  opensbi_head __attribute__ ((section(".head_data"))) =
{
                JUMP_INSTRUCTION,
                "opensbi",
                0, 
                0,
                0,
                0,
                {0},
                {0},
                FW_TEXT_START
};
```

我的 Rust 复刻，注意 Rust 的 `char` 类型和 C 语言的不一致。

```rust
#[repr(C)]
struct SunxiHead {
    pub jump_inst: u32,
    pub magic: [u8; 8],
    pub dtb_base: u32,
    pub uboot_base: u32,
    pub res3: u32,
    pub res4: u32,
    pub res5: [u8; 8],
    pub res6: [u8; 8],
    pub opensbi_base: u32,
}

#[link_section = ".head_data"]
static SUNXI_HEAD: SunxiHead = SunxiHead {
    jump_inst: 0x4000_006f, // j 0x4000_0400
    magic: *b"opensbi",
    dtb_base: 0,
    uboot_base: 0,
    res3: 0,
    res4: 0,
    res5: [0; 8],
    res6: [0; 8],
    opensbi_base: 0x4000_0000,
};
```
