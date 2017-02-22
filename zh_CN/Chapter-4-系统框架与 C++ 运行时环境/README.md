## Chapter 4: 系统框架与 C++ 运行时

#### 内核的 C++ 运行时

内核可以用 C++ 写也可以用 C 写，倾向于用 C++ 是因为一些它语言机制的诱惑，比如运行时支持，构造函数等。

编译器默认假设 C++ 的运行时环境是完整的，但是我们没把 libsupc++ 链接到 C++ 内核中，所以需要增加一些基本函数。这些函数可以在 [kernel/runtime/cxx.cc](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/cxx.cc) 文件中找到。

**注意:** 操作符 `new` and `delete` 在虚拟内存和分页未初始化前是不能使用的。

#### 基本的 C/C++ 函数

内核代码不能使用标准库中的函数，所以我们需要增加一些基本函数来挂历内存和字符串。

```cpp
void 	itoa(char *buf, unsigned long int n, int base);

void *	memset(char *dst,char src, int n);
void *	memcpy(char *dst, char *src, int n);

int 	strlen(char *s);
int 	strcmp(const char *dst, char *src);
int 	strcpy(char *dst,const char *src);
void 	strcat(void *dest,const void *src);
char *	strncpy(char *destString, const char *sourceString,int maxLength);
int 	strncmp( const char* s1, const char* s2, int c );
```

这些函数是在 [kernel/runtime/string.cc](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/string.cc), [kernel/runtime/memory.cc](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/memory.cc), [kernel/runtime/itoa.cc](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/runtime/itoa.cc) 中定义的。

#### C 类型

下一步，我们来定义一下将在代码中用到的不同类型。大部分变量类型是 unsigned 的。这意味着它们的所有位（bit）都用来存储整型数据。signed 变量用第一位来标志类型。

```cpp
typedef unsigned char 	u8;
typedef unsigned short 	u16;
typedef unsigned int 	u32;
typedef unsigned long long 	u64;

typedef signed char 	s8;
typedef signed short 	s16;
typedef signed int 		s32;
typedef signed long long	s64;
```

#### 编译内核

编译内核跟编译一个可执行的 linux 系统是不同的，我们不能用标准库也不能依赖系统。

[kernel/Makefile](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/kernel/Makefile) 文件定义了编译和链接内核的过程

x86 架构中，gcc/g++/ld 将使用以下参数：

```
# 链接器
LD=ld
LDFLAG= -melf_i386 -static  -L ./  -T ./arch/$(ARCH)/linker.ld

# C++ 编译器
SC=g++
FLAG= $(INCDIR) -g -O2 -w -trigraphs -fno-builtin  -fno-exceptions -fno-stack-protector -O0 -m32  -fno-rtti -nostdlib -nodefaultlibs 

# 汇编编译器
ASM=nasm
ASMFLAG=-f elf -o
```
