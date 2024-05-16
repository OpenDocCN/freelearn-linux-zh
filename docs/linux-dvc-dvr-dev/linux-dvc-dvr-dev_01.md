# 第一章：内核开发简介

Linux 是 1991 年芬兰学生 Linus Torvalds 的一个业余项目。该项目逐渐增长，现在仍在增长，全球大约有 1000 名贡献者。如今，Linux 在嵌入式系统和服务器上都是必不可少的。内核是操作系统的核心部分，它的开发并不那么明显。

Linux 相对于其他操作系统有很多优势：

+   免费

+   有着完善的文档和庞大的社区

+   在不同平台上可移植

+   提供对源代码的访问

+   大量免费开源软件

这本书试图尽可能通用。有一个特殊的主题，设备树，它还不是完全的 x86 特性。这个主题将专门用于 ARM 处理器，以及所有完全支持设备树的处理器。为什么选择这些架构？因为它们在台式机和服务器（对于 x86）以及嵌入式系统（ARM）上最常用。

本章主要涉及以下内容：

+   开发环境设置

+   获取、配置和构建内核源代码

+   内核源代码组织

+   内核编码风格简介

# 环境设置

在开始任何开发之前，你需要设置一个环境。至少在基于 Debian 的系统上，专门用于 Linux 开发的环境是相当简单的：

```
 $ sudo apt-get update

 $ sudo apt-get install gawk wget git diffstat unzip texinfo \

 gcc-multilib build-essential chrpath socat libsdl1.2-dev \

 xterm ncurses-dev lzop

```

本书中的一些代码部分与 ARM**系统芯片**（**SoC**）兼容。你也应该安装`gcc-arm`：

```
 sudo apt-get install gcc-arm-linux-gnueabihf

```

我正在一台 ASUS RoG 上运行 Ubuntu 16.04，配备英特尔 i7 处理器（8 个物理核心），16GB 内存，256GB 固态硬盘和 1TB 磁性硬盘。我的最爱编辑器是 Vim，但你可以自由选择你最熟悉的编辑器。

# 获取源代码

在早期的内核版本（直到 2003 年），使用了奇数-偶数版本样式；奇数版本是稳定的，偶数版本是不稳定的。当 2.6 版本发布时，版本方案切换为 X.Y.Z，其中：

+   `X`：这是实际内核的版本，也称为主要版本，当有不兼容的 API 更改时会增加。

+   `Y`：这是次要修订版本，当以向后兼容的方式添加功能时增加。

+   `Z`：这也被称为 PATCH，表示与错误修复相关的版本

这被称为语义版本控制，一直使用到 2.6.39 版本；当 Linus Torvalds 决定将版本号提升到 3.0 时，这也意味着 2011 年语义版本控制的结束，然后采用了 X.Y 方案。

当到了 3.20 版本时，Linus 认为他不能再增加 Y 了，并决定切换到任意的版本方案，当 Y 变得足够大以至于他数不过来时，就增加 X。这就是为什么版本从 3.20 直接变成了 4.0 的原因。请看：[`plus.google.com/+LinusTorvalds/posts/jmtzzLiiejc`](https://plus.google.com/+LinusTorvalds/posts/jmtzzLiiejc)。

现在内核使用任意的 X.Y 版本方案，与语义版本控制无关。

# 源代码组织

对于本书的需求，你必须使用 Linus Torvald 的 Github 存储库。

```
 git clone https://github.com/torvalds/linux
 git checkout v4.1
 ls

```

+   `arch/`：Linux 内核是一个快速增长的项目，支持越来越多的架构。也就是说，内核希望尽可能地通用。架构特定的代码与其他代码分开，并放在这个目录中。该目录包含处理器特定的子目录，如`alpha/`，`arm/`，`mips/`，`blackfin/`等。

+   `block/`：这个目录包含块存储设备的代码，实际上是调度算法。

+   `crypto/`：这个目录包含加密 API 和加密算法代码。

+   `Documentation/`：这应该是你最喜欢的目录。它包含了用于不同内核框架和子系统的 API 描述。在向论坛提问之前，你应该先在这里查找。

+   `drivers/`：这是最重的目录，随着设备驱动程序的合并而不断增长。它包含各种子目录中组织的每个设备驱动程序。

+   `fs/`：此目录包含内核实际支持的不同文件系统的实现，如 NTFS，FAT，ETX{2,3,4}，sysfs，procfs，NFS 等。

+   `include/`：这包含内核头文件。

+   `init/`：此目录包含初始化和启动代码。

+   `ipc/`：这包含**进程间通信**（**IPC**）机制的实现，如消息队列，信号量和共享内存。

+   `kernel/`：此目录包含基本内核的与体系结构无关的部分。

+   `lib/`：库例程和一些辅助函数位于此处。它们是：通用**内核对象**（**kobject**）处理程序和**循环冗余码**（**CRC**）计算函数等。

+   `mm/`：这包含内存管理代码。

+   `net/`：这包含网络（无论是什么类型的网络）协议代码。

+   `scripts/`：这包含内核开发期间使用的脚本和工具。这里还有其他有用的工具。

+   `security/`：此目录包含安全框架代码。

+   `sound/`：音频子系统代码位于此处。

+   `usr/：`目前包含 initramfs 实现。

内核必须保持可移植性。任何特定于体系结构的代码应位于`arch`目录中。当然，与用户空间 API 相关的内核代码不会改变（系统调用，`/proc`，`/sys`），因为这会破坏现有的程序。

该书涉及内核 4.1 版本。因此，任何更改直到 v4.11 版本都会被覆盖，至少可以这样说关于框架和子系统。

# 内核配置

Linux 内核是一个基于 makefile 的项目，具有数千个选项和驱动程序。要配置内核，可以使用`make menuconfig`进行基于 ncurse 的界面，或者使用`make xconfig`进行基于 X 的界面。一旦选择，选项将存储在源树的根目录中的`.config`文件中。

在大多数情况下，不需要从头开始配置。在每个`arch`目录中都有默认和有用的配置文件，可以用作起点：

```
 ls arch/<you_arch>/configs/ 

```

对于基于 ARM 的 CPU，这些配置文件位于`arch/arm/configs/`中，对于 i.MX6 处理器，默认文件配置为`arch/arm/configs/imx_v6_v7_defconfig`。同样，对于 x86 处理器，我们在`arch/x86/configs/`中找到文件，只有两个默认配置文件，`i386_defconfig`和`x86_64_defconfig`，分别用于 32 位和 64 位版本。对于 x86 系统来说，这是非常简单的：

```
make x86_64_defconfig 
make zImage -j16 
make modules 
makeINSTALL_MOD_PATH </where/to/install> modules_install

```

给定一个基于 i.MX6 的板，可以从`ARCH=arm make imx_v6_v7_defconfig`开始，然后`ARCH=arm make menuconfig`。使用前一个命令，您将把默认选项存储在`.config`文件中，使用后一个命令，您可以根据需要更新添加/删除选项。

在使用`xconfig`时可能会遇到 Qt4 错误。在这种情况下，应该使用以下命令：

```
sudo apt-get install  qt4-dev-tools qt4-qmake

```

# 构建您的内核

构建内核需要您指定为其构建的体系结构，以及编译器。也就是说，对于本地构建并非必需。

```
ARCH=arm make imx_v6_v7_defconfig

ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make zImage -j16

```

之后，将看到类似以下内容：

```
    [...]

      LZO     arch/arm/boot/compressed/piggy_data

      CC      arch/arm/boot/compressed/misc.o

      CC      arch/arm/boot/compressed/decompress.o

      CC      arch/arm/boot/compressed/string.o

      SHIPPED arch/arm/boot/compressed/hyp-stub.S

      SHIPPED arch/arm/boot/compressed/lib1funcs.S

      SHIPPED arch/arm/boot/compressed/ashldi3.S

      SHIPPED arch/arm/boot/compressed/bswapsdi2.S

      AS      arch/arm/boot/compressed/hyp-stub.o

      AS      arch/arm/boot/compressed/lib1funcs.o

      AS      arch/arm/boot/compressed/ashldi3.o

      AS      arch/arm/boot/compressed/bswapsdi2.o

      AS      arch/arm/boot/compressed/piggy.o

      LD      arch/arm/boot/compressed/vmlinux

      OBJCOPY arch/arm/boot/zImage

      Kernel: arch/arm/boot/zImage is ready

```

从内核构建中，结果将是一个单一的二进制映像，位于`arch/arm/boot/`中。模块使用以下命令构建：

```
 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make modules

```

您可以使用以下命令安装它们：

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make modules_install

```

`modules_install`目标需要一个环境变量`INSTALL_MOD_PATH`，指定应该在哪里安装模块。如果未设置，模块将安装在`/lib/modules/$(KERNELRELEASE)/kernel/`中。这在第二章 *设备驱动程序基础*中讨论过。

i.MX6 处理器支持设备树，这是用来描述硬件的文件（这在第六章中详细讨论），但是，要编译每个`ARCH`设备树，可以运行以下命令：

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make dtbs

```

但是，并非所有支持设备树的平台都支持`dtbs`选项。要构建一个独立的 DTB，您应该使用：

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make imx6d-    sabrelite.dtb

```

# 内核习惯

内核代码试图遵循标准规则。在本章中，我们只是介绍它们。它们都在专门的章节中讨论，从[第三章](http://post)开始，*内核设施和辅助函数*，我们可以更好地了解内核开发过程和技巧，直到[第十三章](http://post1)，*Linux 设备模型*。

# 编码风格

在深入研究本节之前，您应始终参考内核编码风格手册，位于内核源树中的`Documentation/CodingStyle`。这种编码风格是一组规则，您至少应该遵守这些规则，如果需要内核开发人员接受其补丁。其中一些规则涉及缩进、程序流程、命名约定等。

最流行的是：

+   始终使用 8 个字符的制表符缩进，并且每行应为 80 列长。如果缩进阻止您编写函数，那是因为该函数的嵌套级别太多。可以使用内核源代码中的`scripts/cleanfile`脚本调整制表符大小并验证行大小：

```
scripts/cleanfile my_module.c 
```

+   您还可以使用`indent`工具正确缩进代码：

```
      sudo apt-get install indent

 scripts/Lindent my_module.c

```

+   每个未导出的函数/变量都应声明为静态的。

+   在括号表达式（内部）周围不应添加空格。*s = size of (struct file)*；是可以接受的，而*s = size of( struct file )*；是不可以接受的。

+   禁止使用`typdefs`。

+   始终使用`/* this */`注释样式，而不是`// this`

+   +   不好：`// 请不要使用这个`

+   好的：`/* 内核开发人员喜欢这样 */`

+   宏应该大写，但功能宏可以小写。

+   注释不应该替换不可读的代码。最好重写代码，而不是添加注释。

# 内核结构分配/初始化

内核始终为其数据结构和设施提供两种可能的分配机制。

其中一些结构包括：

+   工作队列

+   列表

+   等待队列

+   Tasklet

+   定时器

+   完成

+   互斥锁

+   自旋锁

动态初始化器都是宏，这意味着它们始终大写：`INIT_LIST_HEAD()`，`DECLARE_WAIT_QUEUE_HEAD()`，`DECLARE_TASKLET()`等等。

说到这一点，所有这些都在第三章中讨论，*内核设施和辅助函数*。因此，代表框架设备的数据结构始终是动态分配的，每个数据结构都有自己的分配和释放 API。这些框架设备类型包括：

+   网络

+   输入设备

+   字符设备

+   IIO 设备

+   类

+   帧缓冲

+   调节器

+   PWM 设备

+   RTC

静态对象的作用域在整个驱动程序中可见，并且由此驱动程序管理的每个设备都可见。动态分配的对象仅由实际使用给定模块实例的设备可见。

# 类、对象和 OOP

内核通过设备和类来实现 OOP。内核子系统通过类进行抽象。几乎每个子系统都有一个`/sys/class/`下的目录。`struct kobject`结构是这种实现的核心。它甚至带有一个引用计数器，以便内核可以知道实际使用对象的用户数量。每个对象都有一个父对象，并且在`sysfs`中有一个条目（如果已挂载）。

每个属于特定子系统的设备都有一个指向**操作**（**ops**）结构的指针，该结构公开了可以在此设备上执行的操作。

# 摘要

本章以非常简短和简单的方式解释了如何下载 Linux 源代码并进行第一次构建。它还涉及一些常见概念。也就是说，这一章非常简短，可能不够，但没关系，这只是一个介绍。这就是为什么下一章会更深入地介绍内核构建过程，如何实际编译驱动程序，无论是作为外部模块还是作为内核的一部分，以及在开始内核开发这段漫长旅程之前应该学习的一些基础知识。
