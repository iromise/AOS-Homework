# The UNIX Time-Sharing System

[TOC]

这篇论文主要是简单地介绍了一下 UNIX 早期的基本功能以及架构，并主要介绍了其中的

- 文件系统
- 进程与镜像
- Shell
- Traps

文章本身并没有太多的创新点，主要在于向大家介绍了 UNIX 系统，介绍了其中相关的部件是如何设计的，以及设计者对于设计这样系统的观点。通过这样的学习，我们可以看出 UNIX 在早期设计时就已经融合了简洁性，文档化，开源共享等基本思想。与此同时，受限于当时的内存和磁盘空间大小，设计者也进行了一些妥协。

## 介绍

在这篇文章发表时，已经有了三个版本的 UNIX，主要包含

* 最早期版本，运行在 PDP-7/PDP-9

* 第二个版本，运行在 PDP-11/20

* 第三个版本，运行在 PDP-11/40/45

作者所要介绍的 UNIX 就是第三个版本。作者指出，第三个版本之所以会产生主要是源于用户对于新特性的需求。在第三个版本中，UNIX 包含了各种软件，而且相关软件的文档都是格式化存储的。

硬件方面，这个版本的 UNIX 是16位的，内存一共有 144 K 字节，其中 UNIX 内核占据了 42K 个字节。在磁盘方面主要有

* 1 个 1M 的固定头的磁盘，主要用于文件存储和 swapping。
* 4 个 2.5M 的可移动头部的可卸载的 disk cartridges。
* 1 个 40M 的可移动头部的可卸载的 disk pack。
软件方面，早期的核心功能使用汇编实现，在当前版本中使用了 C 进行重写。当然，重写有优点也有缺点，优点就是程序可以更容易地被理解，同时支持了更好的特性，如多道程序，可重复代码。缺点就是程序比原来大了 1/3。

## The File System

通过阅读这一部分，我发现，其实在这一版本的 UNIX 中，文件系统已经和目前的 Linux 的宏观文件系统结构非常像了。从用户的角度出发，用户一般会考虑到三种文件

* 普通文件，主要包含文本文件和二进制文件。虽然有些文件可能是有结构的，但是，这些结构是由使用相应文件的程序控制的，和系统并没有任何关系。这在一定程度上增加了可移植性。
* 目录，其实提供了当前目录下的文件名到文件本身的映射。在任何一个目录下都会有两个特殊的项：`.` 和 `..`。具体的目录可以再从两个角度出发，
    * 用户目录，一个用户可以拥有自己的目录，可以读自己的目录，也可以在对应的目录下创建目录与内容。
    * 系统目录，主要指根目录还有存放通用程序的目录。
* 特殊的文件，主要是指与 IO 设备关联的目录，均位于 /dev 目录下。操作这种类型的文件时与操作正常文件几乎没有区别除了但是会激活相关的设备。
而且，令我惊讶的是，在那个时候人们就已经有了 `mount` 的概念了。虽然文件系统的根需要存储在一个设备上，但是整个文件系统并不需要存储在同一个设备上。我们只需要使用 `mount` 命令就可以将一个磁盘挂载到某个路径下。但是呢，在那个时候，人们从空间使用以及复杂度的方面进行考虑，并不运行两个文件系统之间存在链接。

在文件保护方面，那时候每个文件就已经有了 7 个保护位，其中包括

* 三个拥有者的读写执行权限位。
* 三个其他用户的读写执行权限我
* 改变用户的标识符，即目前我们所说的 set-user-id 位。

已经非常接近于目前的 Linux 的文件保护了。**但令我疑惑的是，在当时，创建空目录竟然是特权操作**。

很自然的，对于文件我们自然免不了对文件进行读写，当时，就有了一些 IO 调用，如

* filep = open (name, flag)，打开文件。
    * name 表示文件名。
    * flag 用于标记读，写，更新（同时读写）等操作。
* create，创建文件。
* location = seek(filep, base, offset)，相对于 base 指到文件的某个位置。
比较有意思的是，当时创建文件竟然是一个单独的函数调用，而且竟然有专门的标记来表示同时进行读写。

## Processes and Images

本文中，作者将 image 定义为计算机的执行环境，从我的角度看，它和我们目前所说的虚拟地址空间非常像，因为对于用户来说，其主要有三大部分

* 代码段，从最低地址0开始，进程间可以共享，具有写保护权限。
* 可写数据段，在 8K 地址之后， 可以不断扩大。
* 栈部分，从最高地址向下生长。这一点与目前的虚拟空间仍然保持一致。

作者所考虑的进程与我们现在提到的进程几乎一样，主要包含以下系统调用

* processid = fork (label)，与目前的 fork 不同的是，子进程的控制流会传递到 label。这一点文中并没有进行更加细节地介绍。
* filep = pipe( )，进行进程间的通信，一个祖先进程最初会把对应的管道创建完毕。
* execute(file, arg1, arg2, ..., argn)，执行一个程序，其中 arg1 是文件名。
* processid = wait( )，用于进程同步。父进程会等待子进程结束，返回结束进程的 id。这里并没有说明是否会支持等待所有子进程。
* 
* exit (status)，他会终止进程，关闭其打开的文件。在其退出时，其最近祖先进程会注意到相关的信息。
## The Shell

这个 Shell 与我们目前所熟悉的 Bash 类似，用于用户和计算机交互。其基本执行顺序是

1. 大多数时间，shell 等待用户输入命令
2. 用户输入回车
3. shell 分析命令，使其可以正常执行 execute
4. 执行 fork 命令
5. 子进程尝试根据给定参数执行 execute 命令
6. 父进程等待消亡

在这样的执行过程中，每个 shell 启动的程序都有两个文件描述符，分别表示输入和输出，同时也会有以下的特性

- `|`，管道

* `;`，命令分隔
* `&`，后台运行
* `()`，作为一个子进程运行。
此外，需要注意的是 shell 其实是其它进程的子进程，即默认情况下 UNIX 的初始化的最后一步就是执行 init 函数，从而为每一个 typewriter 建立一个进程，进而为当前用户打开一个 shell。而系统也可以指定在用户登录后执行的第一个程序，比如编辑器。

## Traps

与目前的错误检测机制类似，当系统发现程序

* 引用不存在的内存
* 未实现的指令
* 使用奇地址
时，就会终止进程，报告错误，将用户的镜像写到当前的 core 文件中。用户可以选择使用 debugger 来寻找出错的原因。

当然用户也可以使用中断信号，即输入 delete 字符来中断程序，但是这样并不会产生 core 文件。此外，用户也可以使用 quit 信号强制生成 core 文件。这一点与目前的 Linux 有所不同。目前我们一般都是使用 `Ctrl+C` 的方式来中断程序，同时使用某个开关来控制程序崩溃时是否生成 core 文件，同时还可以配置 Core 文件的名字。

比较有意思的是，当时已经允许这些错误被忽略或者被进程捕获了。

## Perspective

从设计者的角度出发，他们认为 UNIX 之所以非常成功是因为

- UNIX 的成功并不是因为满足了所有先前定义的功能
- 只是为了搭建一个更加好用的环境，使程序员可以更好地工作。
- 不需要为别人而做，都是为了实现自己的想法，比较自由。

而且，他们在设计 UNIX 时遵循了以下基本原则

* 简单易用。因为系统和系统中的程序最后都是让程序员来实现和测试的，所以就尽量保持简单易用，同时设计了很好的交互式程序，以便于提高效率。
* 限制大小。在当时那个环境里，内存和磁盘都非常小，因此在设计时尽可能使得软件优雅，以便于节省空间。
* 很多人可以维护，有想法就可以提出。**我觉得这一点非常重要，这就和目前的开源软件的观念非常像，大家都可以为自己喜欢的东西而一起奋斗。**
## Statistics
根据作者的实验，该系统的效率和稳定性都比较好。我想，这也是 Linux 目前吸引人的一个原因吧。