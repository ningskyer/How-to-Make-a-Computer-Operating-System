如何制作操作系统
=======================================
[SamyPesse/How-to-Make-a-Computer-Operating-System](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System) 中文翻译

## TODO:
	- Chapter 1: Introduction to the x86 architecture and about our OS
	- Chapter 2: Setup the development environment
	- Chapter 3: First boot with GRUB
	- Chapter 4: Backbone of the OS and C++ runtime
	- Chapter 5: Base classes for managing x86 architecture
	- Chapter 6: GDT
	- Chapter 7: IDT and interrupts
	- Chapter 8: Theory: physical and virtual memory
	- Chapter 9: Memory management: physical and virtual


How to Make a Computer Operating System
=======================================

Online book about how to write a computer operating system in C/C++ from scratch.

**Caution**: This repository is a remake of my old course. It was written several years ago [as one of my first projects when I was in High School](https://github.com/SamyPesse/devos), I'm still refactoring some parts. The original course was in French and I'm not an English native. I'm going to continue and improve this course in my free-time.

**Book**: An online version is available at [http://samypesse.gitbooks.io/how-to-create-an-operating-system/](http://samypesse.gitbooks.io/how-to-create-an-operating-system/) (PDF, Mobi and ePub). It was generated using [GitBook](https://www.gitbook.com/).

**Source Code**: All the system source code will be stored in the [src](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/tree/master/src) directory. Each step will contain links to the different related files.

**Contributions**: This course is open to contributions, feel free to signal errors with issues or directly correct the errors with pull-requests.

**Questions**: Feel free to ask any questions by adding issues or commenting sections.

You can follow me on Twitter [@SamyPesse](https://twitter.com/SamyPesse) or [GitHub](https://github.com/SamyPesse).

### What kind of OS are we building?

The goal is to build a very simple UNIX-based operating system in C++, not just a "proof-of-concept". The OS should be able to boot, start a userland shell, and be extensible.

![Screen](./preview.png)
