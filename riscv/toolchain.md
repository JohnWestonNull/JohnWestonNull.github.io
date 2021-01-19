# RISC-V 工具链 & QEMU 虚拟机

本文主要关注 RISC-V 工具链的搭建以及相应的 QEMU Linux 虚拟机的配置启动上。

## Part I: RISC-V 工具链获取 & 安装

#### 拉取代码

首先克隆主仓库

```bash
git clone https://github.com/riscv/riscv-gnu-toolchain.git
```

由于 RISC-V 工具链的仓库使用了 `git submodule` 的形式，因此仓库内含有对多个子仓库的引用，主要如下。

- `riscv-binutils-gdb`
- `riscv-dejagnu`
- `riscv-gcc`
- `riscv-gdb`
- `riscv-glibc`
- `riscv-newlib`

因此在子仓库无法拉取的时候，可以直接在主仓库内执行

```bash
git clone https://github.com/riscv/riscv-binutils-gdb.git # 这里以 riscv-binutils-gdb 举例
```

如果你因为国内网络环境导致无法下载，我这里提供了完整的包（不确保更新）

#### 前置需求安装

如果是 Ubuntu，执行

```bash
sudo apt-get install autoconf automake autotools-dev curl python3 libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev libexpat-dev
```

如果是 Fedora/CentOS/RHEL OS

```shell
sudo yum install autoconf automake python3 libmpc-devel mpfr-devel gmp-devel gawk  bison flex texinfo patchutils gcc gcc-c++ zlib-devel expat-devel
```

如果是 Arch 系，我相信你有足够的背景知识安装。

#### 编译 & 安装

如果只是单纯需要运行 Linux，只编译 `linux` 编译目标即可，虽然建议两个都编译。

```bash
./configure --prefix=/opt/riscv --enable-multilib # 安装目录为 /opt/riscv (需要 root 权限) & 启动 32 位支持
sudo make linux # 安装 linux 交叉编译器，sudo 是由于需要安装到 /opt 下
sudo make       # 安装 newlib 交叉编译器，sudo 同理
```

## Part II: QEMU 虚拟机安装

QEMU 安装比较简单，直接执行下面命令即可，我同样附了安装包。

```bash
git clone https://github.com/riscv/riscv-qemu.git
cd riscv-qemu
./configure --target-list=riscv64-softmmu,riscv32-softmmu
make
```

> Remark: 使用系统包管理器安装的可能不支持 `softmmu` 会导致运行是出现 memory_overlap 问题，编译安装可以解决。

## Part III: BusyBear 完整 Linux 虚拟环境

我们如果需要一个 Linux 虚拟环境，至少需要下面几个东西

- Linux 内核
- `rootfs` 镜像
- BusyBox 工具箱

但这些工具的准备是十分的 teadious 的，不过有人准备好了一整套的环境，即 `BusyBear`。

#### 拉取仓库

```bash
git clone --recursive https://github.com/michaeljclark/busybear-linux.git
```

#### 修改内核版本

如果你使用的是最新的工具链（例如上面安装的 gcc），那么编译时会出现 `redefined symbol` 的问题，此时需要修改 `BusyBear` 所使用的内核版本，将其从 `v5.0` 改到 `v5.10`

在 `conf/busybear.conf` 下，注意第三行的修改。

```bash
BUSYBOX_VERSION=1.30.1
DROPBEAR_VERSION=2018.76
LINUX_KERNEL_VERSION=5.10

ARCH=riscv64
ABI=lp64d
CROSS_COMPILE=riscv64-unknown-linux-gnu-

IMAGE_FILE=busybear.bin
IMAGE_SIZE=100
ROOT_PASSWORD=busybear
```

#### 编译执行

```bash
cd busybear-linux
make
```

如果一切顺利，此时执行

```bash
scripts/start-qemu.sh
```

你就能看到 Linux 的启动界面了