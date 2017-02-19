## Chapter 5: 管理 x86 架构的基本类

现在我们知道了怎么编译 C++ 内核以及用 GRUB 来引导二进制文件，我们可以开始用 C/C++ 做些帅气的事情了。

#### 向命令行窗口打印

我们将使用 VGA 默认模式（03h）来给用户显示文字。屏幕可以从 0xB8000 处的视频内存来直接访问。这个屏幕分辨率方案是 80x25，每个字符通过 2 个字节来定义：一个是字符代码，一个是样式标志。这意味着视频内存总大小是 4000B (80B*25B*2B)。

在 IO 类([io.cc](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc))中:
* **x,y**: 定义屏幕光标位置
* **real_screen**: 定义视频内存的指针
* **putc(char c)**: 在屏幕上打印一个字符并管理光标位置
* **printf(char* s, ...)**: 打印一个字符串

我们给 [IO 类](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc) 增加了一个 **putc** 方法，它用来在屏幕上放置一个字符并更新（x,y）位置。

```cpp
/* 在屏幕上打印字符 */
void Io::putc(char c){
	kattr = 0x07;
	unsigned char *video;
	video = (unsigned char *) (real_screen+ 2 * x + 160 * y);
	// 换行
	if (c == '\n') {
		x = 0;
		y++;
	// 退格符
	} else if (c == '\b') {
		if (x) {
			*(video + 1) = 0x0;
			x--;
		}
	// 制表符
	} else if (c == '\t') {
		x = x + 8 - (x % 8);
	// 回车
	} else if (c == '\r') {
		x = 0;
	} else {
		*video = c;
		*(video + 1) = kattr;

		x++;
		if (x > 79) {
			x = 0;
			y++;
		}
	}
	if (y > 24)
		scrollup(y - 24);
}
```

我们还增加了一个有用且非常熟悉的方法: [printf](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/arch/x86/io.cc#L155)

```cpp
/* 在屏幕上打印字符串 */
void Io::print(const char *s, ...){
	va_list ap;

	char buf[16];
	int i, j, size, buflen, neg;

	unsigned char c;
	int ival;
	unsigned int uival;

	va_start(ap, s);

	while ((c = *s++)) {
		size = 0;
		neg = 0;

		if (c == 0)
			break;
		else if (c == '%') {
			c = *s++;
			if (c >= '0' && c <= '9') {
				size = c - '0';
				c = *s++;
			}

			if (c == 'd') {
				ival = va_arg(ap, int);
				if (ival < 0) {
					uival = 0 - ival;
					neg++;
				} else
					uival = ival;
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				if (neg)
					print("-%s", buf);
				else
					print(buf);
			}
			 else if (c == 'u') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 10);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print(buf);
			} else if (c == 'x' || c == 'X') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 'p') {
				uival = va_arg(ap, int);
				itoa(buf, uival, 16);
				size = 8;

				buflen = strlen(buf);
				if (buflen < size)
					for (i = size, j = buflen; i >= 0;
					     i--, j--)
						buf[i] =
						    (j >=
						     0) ? buf[j] : '0';

				print("0x%s", buf);
			} else if (c == 's') {
				print((char *) va_arg(ap, int));
			}
		} else
			putc(c);
	}

	return;
}
```

#### 汇编接口

汇编语言里有非常多的指令，但是 C 语言里却并不是一一对应的（比如 cli，sti，in 和 out）,所以我们需要有个调用这些指令的接口。

在 C 语言里，我们能直接用 "asm()" 来导入汇编语言，gcc 用 GAS 这个汇编器来编译汇编语言。

**注意:** GAS 使用 AT&T 语法

```cpp
/* 输出字节 */
void Io::outb(u32 ad, u8 v){
	asmv("outb %%al, %%dx" :: "d" (ad), "a" (v));;
}
/* 输出字 */
void Io::outw(u32 ad, u16 v){
	asmv("outw %%ax, %%dx" :: "d" (ad), "a" (v));
}
/* 输出字 */
void Io::outl(u32 ad, u32 v){
	asmv("outl %%eax, %%dx" : : "d" (ad), "a" (v));
}
/* 输入字节 */
u8 Io::inb(u32 ad){
	u8 _v;       \
	asmv("inb %%dx, %%al" : "=a" (_v) : "d" (ad)); \
	return _v;
}
/* 输入字 */
u16	Io::inw(u32 ad){
	u16 _v;			\
	asmv("inw %%dx, %%ax" : "=a" (_v) : "d" (ad));	\
	return _v;
}
/* 输入字 */
u32	Io::inl(u32 ad){
	u32 _v;			\
	asmv("inl %%dx, %%eax" : "=a" (_v) : "d" (ad));	\
	return _v;
}
```
