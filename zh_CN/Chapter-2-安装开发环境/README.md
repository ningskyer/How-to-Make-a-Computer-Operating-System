## Chapter 2: 安装开发环境

第一步就是配置好开发环境。用 Vagrant 和 Virtualbox，你就能在任何平台下编译和测试你的操作系统了。

### 安装 Vagrant

> Vagrant 是一款开源软件，它用来创建和配置虚拟开发环境。它可以看做是 VirtaulBox 的一个封装。

Vagrant 能帮助我们在任何系统上创建一个干净的虚拟开发环境。第一步就是在 http://www.vagrantup.com/ 下载和安装你的系统所对应的 Vagrant。

### 安装 Virtualbox

> Oracle 虚拟机 VirtualBox 是给基于 x86 和 AMD64/Intel64 的电脑提供的虚拟软件的安装包。

Vagrant 需要 Virtualbox 来工作, 在 https://www.virtualbox.org/wiki/Downloads 下载安装 VirtualBox。

### 启动并测试开发环境

Vagrant 和 Virtualbox 安装后，你需要为 Vagrant 下载 ubuntu lucid32 镜像。

```
vagrant box add lucid32 http://files.vagrantup.com/lucid32.box
```

lucid32 镜像准备好后, 我们需要用 *Vagrantfile* 来定义开发环境。[创建一个名为 *Vagrantfile* 的文件](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/Vagrantfile)。这个文件定义了我们环境所需要的依赖：nasm，make，build-essential, grub 和 qemu

启动虚拟机:

```
vagrant up
```

 ssh 连接到 virtual box 来访问虚拟机：

```
vagrant ssh
```

包含 *Vagrantfile* 的文件夹将会默认挂载到虚拟机（本示例中的 Ubuntu Lucid32）的 */vagrant* 目录下：

```
cd /vagrant
```

#### 构建并运行我们的操作系统

[**Makefile**](https://github.com/ningskyer/How-to-Make-a-Computer-Operating-System/blob/master/src/Makefile) 文件定义了构建内核的基础规则，用户 libc 和一些用户空间的程序。

构建:

```
make all
```

用 qemu 测试系统:

```
make run
```
qemu 文档在这里 [QEMU 模拟器文档](http://wiki.qemu.org/download/qemu-doc.html

用 Ctrl-a 退出模拟器
