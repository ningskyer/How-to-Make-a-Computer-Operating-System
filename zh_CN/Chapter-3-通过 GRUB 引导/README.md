## Chapter 3: 通过 GRUB 引导

#### 引导是如何工作的？

当基于 x86 的计算机开机时，它通过一个复杂的路径来最终实现将控制权转移给内核的 “main” 程序。本节课程中，我们考虑的是 BIOS 引导方法而非它的继任者（UEFI——它是一种新的主板引导项，正被看做是有近20多年历史的BIOS 的继任者）。

BIOS 启动过程是：RAM 检测 -> 硬件检测与初始化 -> 引导过程（Boot sequence）

```
译者按：BIOS 主要功能如下——
1）POST
加电自检，检测CPU各寄存器、计时芯片、中断芯片、DMA控制器等
2）Initial
枚举设备，初始化寄存器，分配中断、IO端口、DMA资源等
3）Setup
进行系统设置，存于CMOS中。一般开机时按Del或者F2进入到BIOS的设置界面。
4）常驻程序
开机自检程序运行完后，将撤出内存，故BIOS提供了一组常驻程序INT 10h、INT 13h、INT 15h等，提供给操作系统或应用程序调用。
5）启动自举程序
在POST过程结束后，将调用INT 19h，启动自举程序，自举程序将读取引导记录，装载操作系统。
```
最重要的一步是 “引导过程”，在这里 BIOS 完成初始化并尝试将控制转移给引导程序的另一个 stage。

在 “Boot sequence” 期间，BIOS 会尝试选择一个引导设备（比如软盘、硬盘、CD、USB 闪存设备或网络）。我们的操作系统会优先从硬盘引导（但是未来也可以从 CD 或者 USB 闪存启动）。当一个设备的引导区在 511 和 512 地址上分别包含 `0x55` 和 `0xAA` 信号字节，那么它就被认为是可引导的。这两个魔术字节叫做主引导记录（Master Boot Record，也叫 MBR）。这个信号通过二进制表示为 0b1010101001010101。这种位交替方式是种预防几种固定错误（驱动或控制）的方式。如果这个记录被篡改了或者 0x00（表示分区没有设置引导标志），这个设备就不是可引导的。

BIOS 通过依次把每个设备的引导区的前 512 个字节加载到物理内存中，来枚举搜索引导设备。加载的 512 个字节放在 `0x7C00` (？？) 起始位置上。当检测到合法的信号字节时，BIOS 将控制权交给 `0x7C00` 内存地址（通过跳跃指令，如JMP xxx）来执行引导区的代码。

通过这个过程 CPU 已经在 16 位实模式（Real Mode）下运行了。16 位实模式是 x86 CPU 的默认模式，这是为了维护向后兼容性。要在内核里执行 32 位指令，需要一个引导加载器（bootloader）来将 CPU 切换到保护模式（Protected Mode）。

#### GRUB 是什么？

> GNU GRUB (GNU GRand Unified Bootloader 的简写)是 GNU 项目里的一个引导加载程序。GRUB 是 Free Software Foundation 的 Multiboot Specification 的一个实现，它给用户提供选择机制来启动多系统中的一个 或者在某个系统分区里选择一个指定的内核配置。

为了简单，GRUB 是机器第一个引导的东西，而它会简化存在硬盘上的内核的加载。

#### 为什么用 GRUB?

* GRUB 使用简单
* 使得不用 16 位的代码来加载 32 位内核变得简单
* 可以启动 Linux, Windows 和其他系统
* 使得加载外部模块到内存中变得简单

#### GRUB 怎么使用?

GRUB 使用 Multiboot 规范，可执行二进制文件是 32 位的，在前 8192 字节中包含了一个特殊头部（multiboot 头部）。我们的内核就会成为一个 ELF 可执行文件（("Executable and Linkable Format"，是多数 UNIX 系统中可执行文件的通用标准文件格式）。

内核的第一个引导过程是用汇编语言编写的：[kernel/arch/x86/start.asm](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/start.asm)，我们用一个链接文件来定义我们的可执行文件结构：[kernel/arch/x86/linker.ld](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/linker.ld)

这个引导过程也初始化了一部分 C++ 运行时，下一章再讲。

Multiboot 头部结构:

```cpp
struct multiboot_info {
	u32 flags;
	u32 low_mem;
	u32 high_mem;
	u32 boot_device;
	u32 cmdline;
	u32 mods_count;
	u32 mods_addr;
	struct {
		u32 num;
		u32 size;
		u32 addr;
		u32 shndx;
	} elf_sec;
	unsigned long mmap_length;
	unsigned long mmap_addr;
	unsigned long drives_length;
	unsigned long drives_addr;
	unsigned long config_table;
	unsigned long boot_loader_name;
	unsigned long apm_table;
	unsigned long vbe_control_info;
	unsigned long vbe_mode_info;
	unsigned long vbe_mode;
	unsigned long vbe_interface_seg;
	unsigned long vbe_interface_off;
	unsigned long vbe_interface_len;
};
```

你可以用命令 ```mbchk kernel.elf``` 来检测验证 kernel.elf 文件的 multiboot 标准。你也可以用命令 ```nm -n kernel.elf``` 来验证 ELF 二进制文件中不同对象的地址偏移量。

#### 给内核和 GRUB 创建硬盘镜像

这个脚本 [sdk/diskimage.sh](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/sdk/diskimage.sh) 将会生成一个能被 QEMU 用的硬盘镜像。

第一步就是用 qemu-img 来创建硬盘镜像（c.img）:

```
qemu-img create c.img 2M
```

我们需要用 fdisk 来分区，以下是 fdisk 的常用命令：

```bash
fdisk ./c.img

# 切换到 Expert 命令
Switch to Expert commands
> x

# 改变扇面的数目(1-1048576)
> c
> 4

# 改变扇区的数目(1-256, default 16):
> h
> 16

# 改变磁道的数目
> s
> 63

# 返回主菜单
> r

# 增加一个分区
> n

# 选择主分区
> p

# 选择分区数量
> 1

# 指定分区的起始扇区(1-4, 默认 1)
> 1

# 指定分区的终止扇区，根据前面的提示我们可以做出相应的选择+sectors 或 +size{K,M,G}(1-4,默认 4)
> 4

# 切换可引导标志位
> a

# 给可引导标志选择第一个分区
> 1

# 保存写入硬盘并退出
> w
```

我们现在需要将创建好的分区用 losetup 命令设置成循环设备。它能使我们像块设备一样访问一个文件。losetup 需要将这个分区的偏移量当做一个参数传入。这个偏移量这样来计算： **地址偏移量= 起始扇区 * 扇区字节数**

或者直接用 ```fdisk -l -u c.img``` 来获取偏移量， 63 * 512 = 32256.

```bash
losetup -o 32256 /dev/loop1 ./c.img
```

用以下命令在这个新设备（循环设备）上创建一个 EXT2 文件系统：

```bash
mke2fs /dev/loop1
```

从一个挂载的硬盘中拷贝我们的文件：

```bash
mount  /dev/loop1 /mnt/
cp -R bootdisk/* /mnt/
umount /mnt/
```

在硬盘上安装 GRUB:

```bash
grub --device-map=/dev/null << EOF
device (hd0) ./c.img
geometry (hd0) 4 16 63
root (hd0,0)
setup (hd0)
quit
EOF
```

最后卸载循环设备：

```bash
losetup -d /dev/loop1
```

```
译者按：
在类 UNIX 系统里，loop 设备是一种伪设备(pseudo-device)，或者也可以说是仿真设备。它能使我们像块设备一样访问一个文件。在使用之前，一个 loop 设备必须要和一个文件进行连接。这种结合方式给用户提供了一个替代块特殊文件的接口。因此，如果这个文件包含有一个完整的文件系统，那么这个文件就可以像一个磁盘设备一样被 mount 起来。上面说的文件格式，我们经常见到的是 CD 或 DVD 的 ISO 光盘镜像文件或者是软盘(硬盘)的 *.img 镜像文件。通过这种 loop mount (回环mount)的方式，这些镜像文件就可以被 mount 到当前文件系统的一个目录下。losetup 命令用来设置循环设备，籍此来模拟整个文件系统，让用户得以将其视为硬盘驱动器，光驱或软驱等设备，并挂入当作目录来使用。

```

#### 扩展资料

* [GNU GRUB on Wikipedia](http://en.wikipedia.org/wiki/GNU_GRUB)
* [Multiboot specification](https://www.gnu.org/software/grub/manual/multiboot/multiboot.html)
