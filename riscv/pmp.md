# RISC-V PMP 测试与抽象

物理内存保护（Physical Memory Protection）是 RISC-V 安全性设计的一个重要组成部分。其分为三种保护模式，NA4、NAPOT、TOR，其中 NA4 是 NAPOT 的一种特殊形式。

下面先介绍 PMP 的基础然后分别介绍三种模式的不同之处以及如何抽象。

## PMP 基础知识

PMP 由两类寄存器组成，`pmpaddr` 与 `pmpcfg`，其中 `pmpaddr` 是 16 个 XLEN 长度的寄存器，里面存储地址的 `[55:2]`、`[34:2]` 位，注意到这里最后两位被忽略掉了。因此保护的范围最小为 4 Bytes（NA4 模式）。
`pmpcfg` 是 16 个 8 位寄存器组成，被打包进 2、4 个 XLEN 寄存器中负责决定 PMP 的模式。

```rust
bitflags! {
    #[repr(C)]
    pub struct PmpFlags: u8 {
        /* RWX 代表范围内内存的权限 */
        const READABLE =    1 << 0;
        const WRITABLE =    1 << 1;
        const EXECUTABLE =  1 << 2;
        /* PMP 寄存器地址的模式 */
        const MODE_TOR =    1 << 3;
        const MODE_NA4 =    2 << 3;
        const MODE_NAPOT =  3 << 3;
        /* 当被 LOCKED 时，只有系统重置可以清空该位 */
        const LOCKED =      1 << 7;
    }
}
```

## TOR 模式（Top of range）

TOR 模式十分简单，但比较浪费 PMP 寄存器，它将两个 `pmpaddr` 寄存器作为一对，当 `pmpcfgX` 为 TOR 模式时，`pmpaddr{X-1}` 与 `pmpaddr{X}` 成为一对，在该范围内的地址受到保护（前闭后开）。

```rust
fn to_tor(&self) -> (usize, usize) {
    (self.addr >> 2, (self.addr + self.size) >> 2)
}
```

## NA4 模式（Naturally aligned four bytes）

NA4 模式是 NAPOT 模式的特殊形式，其只保护 `pmpaddrX` 开始的 4 字节地址范围。

```rust
fn to_napot(&self) -> usize {
    if self.size < 2 || self.size > 56 {
        panic!("[ERROR] invalid pmp napot value");
    }
    if self.size == 2 {
        /* NA4 */
        self.addr >> 2
    } else {
        /* NAPOT */
        let mut pmpaddr = self.addr;
        pmpaddr = pmpaddr >> 2;
        pmpaddr.set_bit(self.size-3, false);
        pmpaddr.set_bits(0..(self.size-3), (1 << (self.size-3))-1);
        pmpaddr
    }
}
```

## NAPOT 模式（Naturally aligned power of two）

NAPOT 模式是利用 `pmpaddrX` 的低比特位来编码需要的地址范围，取决于低比特位的连续 1 的个数，当没有连续 1 时，其为 NA4 模式。

其对应方式可以用下面的表格表示。

| `pmpaddr`   | `pmpcfg.A` | Match type and size            |
|-------------|------------|--------------------------------|
| aaaa...aaaa | NA4        | 4-byte NAPOT range             |
| aaaa...aaa0 | NAPOT      | 8-byte NAPOT range             |
| aaaa...aa01 | NAPOT      | 16-byte NAPOT range            |
| aaaa...a011 | NAPOT      | 32-byte NAPOT range            |
| ...         | ...        | ...                            |
| 0111...1111 | NAPOT      | $2^{XLEN+2}$-byte NAPOT range  |

## Rust 对不同模式的写入的方式

```rust
pub fn enforce(&self, index: usize) {
    if !self.enabled {
        return
    }
    if self.pmp_cfg.contains(PmpFlags::MODE_NA4) || self.pmp_cfg.contains(PmpFlags::MODE_NAPOT) {
        pmpaddr_write(index, self.to_napot());
    } else {
        let (s, e) = self.to_tor();
        pmpaddr_write(index-1, s);
        pmpaddr_write(index, e);
    }
    pmpcfg_write(index, self.pmp_cfg.bits());
}

pub fn exempt(&self, index: usize) {
    pmpcfg_write(index, 0x0);
    pmpaddr_write(index, 0x0);
}
```

## 硬件层级 PMP 寄存器读写

```rust
#[cfg(target_arch="riscv64")]
pub(crate) fn pmpcfg_read(index: usize) -> u8 {
    match index {
        0..=7 => (pmpcfg0::read() >> (8 * index)) as u8,
        8..=15 => (pmpcfg2::read() >> (8 * (index - 8))) as u8,
        _ => panic!("pmp does not exist")
    }
}

#[cfg(target_arch="riscv64")]
pub(crate) fn pmpcfg_write(index: usize, value: u8) {
use bit_field::BitField;
    match index {
        0..=7 => {
            let range = index * 8..(index + 1) * 8;
            let mut reg_value = pmpcfg0::read();
            reg_value.set_bits(range, value as usize);
            pmpcfg0::write(reg_value);
        },
        8..=15 => {
            let range = (index - 8) * 8..(index - 7) * 8;
            let mut reg_value = pmpcfg0::read();
            reg_value.set_bits(range, value as usize);
            pmpcfg2::write(reg_value);
        },
        _ => panic!("pmp does not exist")
    }
}
```

## PMP 测试思路

注意到的是 RISC-V 的 `mstatus` 寄存器中提供了 MPRV 位，当该位置 1 时，在 Machine Mode 下对内存的读写也会经过 mmu 翻译与 PMP 检查，因此我们可以设计一个 Runtime 来进行 PMP 软件可靠性的测试。
