## Chapter 7: IDT 和中断

**注：**
IDT 与 PIC 也是在 [kernel/arch/x86/x86.cc](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/x86.cc) 中定义的。

中断是发送给处理器的一种信号，它是被硬件或软件触发的，表示一个需要立即响应的事件。

有三种类型的中断：

- **硬件中断:** 由外设（键盘、鼠标、硬盘，……）发送给处理器。处理器在轮询循环中等待外部事件时会有 CPU 时间的浪费，硬件中断是种减少这种时间浪费的方式。
- **软件中断:** 是被软件自动初始化的。它用来管理系统调用。
- **异常:** 用来处理程序执行时内部出现了无法处理的错误或异常时（比如除以 0 ，缺页故障，……）

#### 键盘例子：

当用户在键盘上按键时，键盘控制器会给中断控制器发送一个中断信号。如果中断没有被屏蔽，中断控制器会把中断信号发送给处理器，处理器将执行一个程序来管理这个中断（如键按下或者释放），比如说执行一个程序来从键盘控制器里获取按下的键并把对应字母打印到屏幕上。一旦字符处理程序执行完后，被中断的任务就能恢复了。

#### 什么是 PIC?

[PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller) (Programmable interrupt controller 可编程中断控制器) 是一个合并几个中断源到一个或多个 CPU 线上的设备，它给中断划分优先级层次。当设备有多个中断要激活时，它会让他们根据相对优先级来激活

最出名的 PIC 是 8259A，每个 8259A 都能处理 8 个设备，但是多数计算机有两个控制器：一个主控制器一个从控制器，这使得计算机能管理 14 个设备上的中断事件。

这一章里，我们需要编写控制器初始化和屏蔽中断的代码。

#### 什么是 IDT?

> 中断描述表（Interrupt Descriptor Table - IDT）是一种 x86 架构中使用的数据结构，它用来实现中断向量表。处理器用 IDT 来决定中断和异常的正确响应。

我们的内核打算用 IDT 来定义中断发生时要执行的不同函数。

像 GDT 一样，IDT 是用 LIDTL 汇编指令来加载的。它会给出 IDT 描述结构的地址：

```cpp
struct idtr {
	u16 limite;
	u32 base;
} __attribute__ ((packed));
```

中断描述表是由 IDT 段组成的，段结构如下：

```cpp
struct idtdesc {
	u16 offset0_15;
	u16 select;
	u16 type;
	u16 offset16_31;
} __attribute__ ((packed));
```

**注意:** 指令 ```__attribute__ ((packed))``` 是告诉 gcc 编译器这个结构应该尽可能地减少占用内存。如果不写这个指令，gcc 会包含一些优化内存对齐和访问的字节。


现在我们需要定义 IDT 表，然后用 LIDTL 指令加载它。IDT 表可以存储到内存的任意地址，只要它的地址能被发送给使用 IDTR 寄存器的进程就可以。

下面是一张常用中断的表格（可屏蔽中断叫做 IRQ）:

| IRQ   |         Description        |
|:-----:| -------------------------- |
| 0 | 可编程定时器中断 |
| 1 | 键盘中断 |
| 2 | Cascade (在 主从两个 PIC 内部使用，不抛出) |
| 3 | COM2 中断(如果使能了) |
| 4 | COM1 中断(如果使能了) |
| 5 | LPT2 中断(if enabled) |
| 6 | 软盘 |
| 7 | LPT1 中断|
| 8 | CMOS 实时时钟 (如果使能了) |
| 9 | 外设 / 传统 SCSI / NIC 中断 |
| 10 | 外设 / 传统 SCSI / NIC 中断 |
| 11 | 外设 / 传统 SCSI / NIC 中断 |
| 12 | PS2 鼠标 |
| 13 | FPU / 协处理器 / 内部中断 |
| 14 | 原 ATA 硬盘 |
| 15 | 副 ATA 硬盘 |

#### 怎么初始化中断？

这是一个定义 IDT 段的简单方法

```cpp
void init_idt_desc(u16 select, u32 offset, u16 type, struct idtdesc *desc)
{
	desc->offset0_15 = (offset & 0xffff);
	desc->select = select;
	desc->type = type;
	desc->offset16_31 = (offset & 0xffff0000) >> 16;
	return;
}
```

现在我们可以初始化中断了：

```cpp
#define IDTBASE	0x00000000
#define IDTSIZE 0xFF
idtr kidtr;
```


```cpp
void init_idt(void)
{
	/* 初始化可屏蔽中断 */
	int i;
	for (i = 0; i < IDTSIZE; i++)
		init_idt_desc(0x08, (u32)_asm_schedule, INTGATE, &kidt[i]); //

	/* 向量  0 -> 31 代表异常 */
	init_idt_desc(0x08, (u32) _asm_exc_GP, INTGATE, &kidt[13]);		/* #GP */
	init_idt_desc(0x08, (u32) _asm_exc_PF, INTGATE, &kidt[14]);     /* #PF */

	init_idt_desc(0x08, (u32) _asm_schedule, INTGATE, &kidt[32]);
	init_idt_desc(0x08, (u32) _asm_int_1, INTGATE, &kidt[33]);

	init_idt_desc(0x08, (u32) _asm_syscalls, TRAPGATE, &kidt[48]);
	init_idt_desc(0x08, (u32) _asm_syscalls, TRAPGATE, &kidt[128]); //48

	kidtr.limite = IDTSIZE * 8;
	kidtr.base = IDTBASE;


	/* 把 IDT 拷贝到内存中 */
	memcpy((char *) kidtr.base, (char *) kidt, kidtr.limite);

	/* 加载 IDTR 寄存器 */
	asm("lidtl (kidtr)");
}
```

IDT 初始化后，我们需要通过配置 PIC 激活中断。下面的函数将用处理器的输出端口 ```io.outb``` 来写入内部寄存器，从而实现配置主从两个 PIC。我们用端口来配置 PIC：

* 主 PIC: 0x20 和 0x21
* 从 PIC: 0xA0 和 0xA1

对于 PIC 来说，有两种寄存器：

* ICW (初始化命令字 Initialization Command Word): 重新初始化控制器
* OCW (操作控制字 Operation Control Word): 配置初始化了的控制器（一般用来屏蔽和取消屏蔽中断）

```cpp
void init_pic(void)
{
	/* ICW1 初始化*/
	io.outb(0x20, 0x11);
	io.outb(0xA0, 0x11);

	/* ICW2 初始化*/
	io.outb(0x21, 0x20);	/* start vector = 32 */
	io.outb(0xA1, 0x70);	/* start vector = 96 */

	/* ICW3 初始化*/
	io.outb(0x21, 0x04);
	io.outb(0xA1, 0x02);

	/* ICW4 初始化*/
	io.outb(0x21, 0x01);
	io.outb(0xA1, 0x01);

	/* 屏蔽中断 */
	io.outb(0x21, 0x0);
	io.outb(0xA1, 0x0);
}
```

#### PIC ICW 配置细节

寄存器必须按顺序配置。

**ICW1 (port 0x20 / port 0xA0)**
```
|0|0|0|1|x|0|x|x|
         |   | +--- with ICW4 (1) or without (0)
         |   +----- one controller (1), or cascade (0)
         +--------- triggering by level (level) (1) or by edge (edge) (0)
```

**ICW2 (port 0x21 / port 0xA1)**
```
|x|x|x|x|x|0|0|0|
 | | | | |
 +----------------- base address for interrupts vectors
```

**ICW2 (0x21 端口或 0xA1 端口)**

对于主控制器:
```
|x|x|x|x|x|x|x|x|
 | | | | | | | |
 +------------------ slave controller connected to the port yes (1), or no (0)
```

对于从控制器:
```
|0|0|0|0|0|x|x|x|  pour l'esclave
           | | |
           +-------- Slave ID which is equal to the master port
```

**ICW4 (port 0x21 / port 0xA1)**

它是用来定义控制器应该工作在哪种模式下。

```
|0|0|0|x|x|x|x|1|
       | | | +------ mode "automatic end of interrupt" AEOI (1)
       | | +-------- mode buffered slave (0) or master (1)
       | +---------- mode buffered (1)
       +------------ mode "fully nested" (1)
```

#### 为什么 IDT 段要增加偏移量？

你应该已经注意到，当我初始化 IDT 段时，使用了偏移量来给汇编语言的代码分段。是因为[x86int.asm](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/x86int.asm) 文件里定义了不同的函数，都是类似下面的方式：

```asm
%macro	SAVE_REGS 0
	pushad
	push ds
	push es
	push fs
	push gs
	push ebx
	mov bx,0x10
	mov ds,bx
	pop ebx
%endmacro

%macro	RESTORE_REGS 0
	pop gs
	pop fs
	pop es
	pop ds
	popad
%endmacro

%macro	INTERRUPT 1
global _asm_int_%1
_asm_int_%1:
	SAVE_REGS
	push %1
	call isr_default_int
	pop eax	;;a enlever sinon
	mov al,0x20
	out 0x20,al
	RESTORE_REGS
	iret
%endmacro
```

这些宏将会用来定义中断的段，这能防止不同寄存器的损坏，对于多任务来说是非常有用的。
