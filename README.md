如何制作操作系统
=======================================
[SamyPesse/How-to-Make-a-Computer-Operating-System](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System) 项目的中文翻译

## 内容:
	- Chapter 1: 架构及本系统介绍  【Completed】
	- Chapter 2: 安装开发环境 【Completed】
	- Chapter 3: 通过 GRUB 启动 【Completed】
	- Chapter 4: 系统框架与 C++ 运行时
	- Chapter 5: 管理 x86 架构的基本类
	- Chapter 6: GDT
	- Chapter 7: IDT 和 中断
	- Chapter 8: 内存管理：物理内存与虚拟内存
	- Chapter 9: 内存管理：实现

关于如何用 C++ 来从零制作操作系统的电子书

**注意**: 这个仓库是我以前课程成果的修改版。它是几年前我在高中时写的其中一个[项目](https://github.com/SamyPesse/devos)，我仍然在重构某些部分。原始课程是法语的，而我的母语也并非英语。我准备业余时间继续提高这门课程。

**书籍**: 
线上版本可以在这里获取[http://samypesse.gitbooks.io/how-to-create-an-operating-system/](http://samypesse.gitbooks.io/how-to-create-an-operating-system/) (PDF, Mobi 和 ePub 格式)，它是用[GitBook](https://www.gitbook.com/) 生成的。

**源码**: 所有系统源码都将存在 [src](https://github.com/SamyPesse/How-to-Make-a-Computer-Operating-System/tree/master/src) 目录下。每一步都会包含指向相关文件的链接。

**贡献**: 这个课程开放贡献，欢迎提 issues 或 PR。

**提问**: 欢迎通过 issues 或评论来提问题。

Twitter [@SamyPesse](https://twitter.com/SamyPesse) / [GitHub](https://github.com/SamyPesse).

### 我们在创建什么类型的操作系统？

我们的目标是用 C++ 来创建一个简单的类 UNIX 的操作系统，但它并非只是概念型的。它可以启动，运行用户态的 shell，并且可扩展

![Screen](./preview.png)
