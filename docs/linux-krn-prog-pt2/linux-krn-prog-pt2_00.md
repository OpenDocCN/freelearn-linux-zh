# 前言

本书旨在帮助您以实际、实践的方式学习 Linux 字符设备驱动程序开发的基础知识，同时提供必要的理论背景，使您对这个广阔而有趣的主题领域有一个全面的了解。为了充分涵盖这些主题，本书的范围故意保持在（大部分）学习如何在 Linux 操作系统上编写`misc`类字符设备驱动程序。这样，您将能够深入掌握基本和必要的驱动程序编写技能，然后能够相对轻松地处理不同类型的 Linux 驱动程序项目。

重点是通过强大的**可加载内核模块**（**LKM**）框架进行实际驱动程序开发；大多数内核驱动程序开发都是以这种方式进行的。重点是在与驱动程序代码的实际操作中保持关注，必要时在足够深的层面上理解内部工作，并牢记安全性。

我们强烈推荐的一点是：要真正学习和理解细节，**最好先阅读并理解本书的伴侣《Linux 内核编程》**。它涵盖了各个关键领域 - 从源代码构建内核，通过 LKM 框架编写内核模块，内核内部包括内核架构，内存系统，内存分配/释放 API，CPU 调度等等。这两本书的结合将为您提供确定和深入的优势。

这本书没有浪费时间 - 第一章让您学习了 Linux 驱动程序框架的细节以及如何编写一个简单但完整的 misc 类字符设备驱动程序。接下来，您将学习如何做一些非常必要的事情：使用各种技术有效地与用户空间进程进行接口（其中一些还可以作为调试/诊断工具！）。然后介绍了理解和处理硬件（外围芯片）I/O 内存。接下来是详细介绍处理硬件中断。这包括学习和使用几种现代驱动程序技术 - 使用线程中断请求，利用资源管理的 API 进行驱动程序，I/O 资源分配等。它涵盖了顶部/底部是什么，使用任务队列和软中断，以及测量中断延迟。接下来是您通常会使用的内核机制 - 使用内核定时器，设置延迟，创建和管理内核线程和工作队列。

本书的剩余两章涉及一个相对复杂但对于现代专业级驱动程序或内核开发人员至关重要的主题：理解和处理内核同步。

本书使用了最新的，即写作时的 5.4 **长期支持**（**LTS**）Linux 内核。这是一个将从 2019 年 11 月一直维护（包括错误和安全修复）到 2025 年 12 月的内核！这是一个关键点，确保本书的内容在未来几年仍然保持当前和有效！

我们非常相信实践经验的方法：本书的 GitHub 存储库上的 20 多个内核模块（以及一些用户应用程序和 shell 脚本）使学习变得生动，有趣且有用。

我们真诚希望您从这本书中学到并享受到知识。愉快阅读！

# 这本书是为谁准备的

这本书主要是为刚开始学习设备驱动程序开发的 Linux 程序员准备的。Linux 设备驱动程序开发人员希望克服频繁和常见的内核/驱动程序开发问题，以及理解和学习执行常见驱动程序任务 - 现代**Linux 设备模型**（**LDM**）框架，用户-内核接口，执行外围 I/O，处理硬件中断，处理并发等等 - 将受益于本书。需要基本了解 Linux 内核内部（和常见 API），内核模块开发和 C 编程。

# 本书涵盖了什么

第一章，“编写简单的杂项字符设备驱动程序”，首先介绍了非常基础的内容 - 驱动程序应该做什么，设备命名空间，sysfs 和 LDM 的基本原则。然后我们深入讨论了编写简单字符设备驱动程序的细节；在此过程中，您将了解框架 - 实际上是“如果不是一个进程，它就是一个文件”哲学/架构的内部实现！您将学习如何使用各种方法实现杂项类字符设备驱动程序；几个代码示例有助于加深概念。还涵盖了在用户空间和内核空间之间复制数据的基本方法。还涵盖了关键的安全问题以及如何解决这些问题（在这种情况下）；实际上演示了一个“坏”驱动程序引发特权升级问题！

第二章，“用户空间和内核通信路径”，涵盖了如何在内核和用户空间之间进行通信，这对于您作为内核模块/驱动程序的作者来说至关重要。在这里，您将了解各种通信接口或路径。这是编写内核/驱动程序代码的重要方面。采用了几种技术：通过传统的 procfs 进行通信，通过 sysfs 进行驱动程序的更好方式，以及其他几种方式，通过 debugfs，netlink 套接字和 ioctl(2)系统调用。

第三章，“处理硬件 I/O 内存”，涵盖了驱动程序编写的一个关键方面 - 访问外围设备或芯片的硬件内存（映射内存 I/O）的问题和解决方案。我们涵盖了使用常见的内存映射 I/O（MMIO）技术以及（通常在 x86 上）端口 I/O（PIO）技术进行硬件 I/O 内存访问和操作。还展示了来自现有内核驱动程序的几个示例。

第四章，“处理硬件中断”，详细介绍了如何处理和处理硬件中断。我们首先简要介绍内核如何处理硬件中断，然后介绍了您如何“分配”IRQ 线（涵盖现代资源管理的 API），以及如何正确实现中断处理程序。然后涵盖了使用线程处理程序的现代方法（以及原因），不可屏蔽中断（NMI）等。还涵盖了在代码中使用“顶半部分”和“底半部分”中断机制的原因以及使用方式，以及有关硬件中断处理的 dos 和 don'ts 的关键信息。使用现代[e]BPF 工具集和 Ftrace 测量中断延迟，结束了这一关键章节。

第五章，“使用内核定时器、线程和工作队列”，涵盖了如何使用一些有用的（通常由驱动程序使用）内核机制 - 延迟、定时器、内核线程和工作队列。它们在许多实际情况下都很有用。如何执行阻塞和非阻塞延迟（根据情况），设置和使用内核定时器，创建和使用内核线程，理解和使用内核工作队列都在这里涵盖。几个示例模块，包括一个简单的加密解密（sed）示例驱动程序的三个版本，用于说明代码中学到的概念。

第六章，“内核同步-第一部分”，首先介绍了关于关键部分、原子性、锁概念的实现以及非常重要的原因。然后我们涵盖了在 Linux 内核中工作时的并发性问题；这自然地引出了重要的锁定准则，死锁的含义以及预防死锁的关键方法。然后深入讨论了两种最流行的内核锁技术 - 互斥锁和自旋锁，以及几个（驱动程序）代码示例。

第七章，*内核同步-第二部分*，继续探讨内核同步的内容。在这里，您将学习关键的锁定优化-使用轻量级原子和（更近期的）引用计数操作符来安全地操作整数，使用 RMW 位操作符来安全地执行位操作，以及使用读者-写者自旋锁而不是常规自旋锁。还讨论了缓存“错误共享”等固有风险。然后介绍了无锁编程技术的概述（重点是每 CPU 变量及其用法，并附有示例）。接着介绍了关键的主题，锁调试技术，包括内核强大的 lockdep 锁验证器的使用。该章节最后简要介绍了内存屏障（以及现有内核网络驱动程序对内存屏障的使用）。

我们再次强调，本书是为新手内核程序员编写设备驱动程序而设计的；本书不涵盖一些 Linux 驱动程序主题，包括其他类型的设备驱动程序（除了字符设备）、设备树等。Packt 提供了其他有价值的指南，帮助您在这些主题领域取得进展。本书将是一个很好的起点。

# 为了充分利用本书

为了充分利用本书，我们希望您具有以下知识和经验：

+   熟悉 Linux 系统的命令行操作。

+   C 编程语言。

+   了解如何通过**可加载内核模块**（LKM）框架编写简单的内核模块

+   了解（至少基本的）关键的 Linux 内核内部概念：内核架构，内存管理（以及常见的动态内存分配/释放 API），以及 CPU 调度。

+   这不是强制性的，但是具有 Linux 内核编程概念和技术的经验将会有很大帮助。

理想情况下，我们强烈建议先阅读本书的伴侣《Linux 内核编程》。

本书的硬件和软件要求以及其安装细节如下：

| **章节编号** | **所需软件（版本）** | **免费/专有** | **软件下载链接** | **硬件规格** | **所需操作系统** |
| --- | --- | --- | --- | --- | --- |

| 所有章节 | 最新的 Linux 发行版；我们使用 Ubuntu 18.04 LTS（以及 Fedora 31 / Ubuntu 20.04 LTS）；任何一个都可以。建议您将 Linux 操作系统安装为**虚拟机**（VM），使用 Oracle VirtualBox 6.x（或更高版本）作为 hypervisor | 免费（开源） | Ubuntu（桌面版）：[`ubuntu.com/download/desktop`](https://ubuntu.com/download/desktop)Oracle VirtualBox：[`www.virtualbox.org/wiki/Downloads`](https://www.virtualbox.org/wiki/Downloads) | *必需：*一台现代化的相对强大的 PC 或笔记本电脑，配备至少 4GB RAM（最少；越多越好），25GB 的可用磁盘空间和良好的互联网连接。*可选：*我们还使用树莓派 3B+作为测试平台。 | Linux 虚拟机在 Windows 主机上 -或-

Linux 作为独立的操作系统 |

详细的安装步骤（软件方面）：

1.  在 Windows 主机系统上安装 Linux 作为虚拟机；按照以下教程之一进行操作：

+   *在 Windows 中使用 VirtualBox 安装 Linux，Abhishek Prakash（It's FOSS！，2019 年 8 月）*：[`itsfoss.com/install-linux-in-virtualbox/`](https://itsfoss.com/install-linux-in-virtualbox/)

+   或者，这里有另一个教程可以帮助您完成相同的操作：*在 Oracle VirtualBox 上安装 Ubuntu*：[`brb.nci.nih.gov/seqtools/installUbuntu.html`](https://brb.nci.nih.gov/seqtools/installUbuntu.html)

1.  在 Linux 虚拟机上安装所需的软件包：

1.  登录到您的 Linux 虚拟机客户端，并首先在终端窗口（shell）中运行以下命令：

```
sudo apt update
sudo apt install gcc make perl
```

1.  1.  现在安装 Oracle VirtualBox Guest Additions。参考：*如何在 Ubuntu 中安装 VirtualBox Guest Additions*：[`www.tecmint.com/install-virtualbox-guest-additions-in-ubuntu/`](https://www.tecmint.com/install-virtualbox-guest-additions-in-ubuntu/)

（此步骤仅适用于使用 Oracle VirtualBox 作为 hypervisor 应用程序的 Ubuntu 虚拟机。）

1.  要安装软件包，请按以下步骤操作：

1.  在 Ubuntu 虚拟机中，首先运行`sudo apt update`命令

1.  现在，在一行中运行`sudo apt install git fakeroot build-essential tar ncurses-dev tar xz-utils libssl-dev bc stress python3-distutils libelf-dev linux-headers-$(uname -r) bison flex libncurses5-dev util-linux net-tools linux-tools-$(uname -r) exuberant-ctags cscope sysfsutils curl perf-tools-unstable gnuplot rt-tests indent tree pstree smem hwloc bpfcc-tools sparse flawfinder cppcheck tuna hexdump trace-cmd virt-what`命令。

1.  有用的资源：

+   Linux 内核官方在线文档：[`www.kernel.org/doc/html/latest/`](https://www.kernel.org/doc/html/latest/)。

+   Linux 驱动程序验证（LDV）项目，特别是*在线 Linux 驱动程序验证服务*页面：[`linuxtesting.org/ldv/online?action=rules`](http://linuxtesting.org/ldv/online?action=rules)。

+   SEALS - 简单嵌入式 ARM Linux 系统：[`github.com/kaiwan/seals/`](https://github.com/kaiwan/seals/)。

+   本书的每一章还有一个非常有用的*进一步阅读*部分，详细介绍更多资源。

1.  本书的伴随指南*Linux 内核编程，Kaiwan N Billimoria，Packt Publishing*的*第一章，内核工作区设置*中描述了详细的说明，以及其他有用的项目，安装 ARM 交叉工具链等。

我们已经在这些平台上测试了本书中的所有代码（它也有自己的 GitHub 存储库）：

+   x86_64 Ubuntu 18.04 LTS 客户操作系统（在 Oracle VirtualBox 6.1 上运行）

+   x86_64 Ubuntu 20.04.1 LTS 客户操作系统（在 Oracle VirtualBox 6.1 上运行）

+   x86_64 Ubuntu 20.04.1 LTS 本机操作系统

+   ARM Raspberry Pi 3B+（运行其发行版内核以及我们的自定义 5.4 内核）；轻度测试。

**如果您使用本书的数字版本，我们建议您自己输入代码，或者更好地，通过 GitHub 存储库访问代码（链接在下一节中可用）。这样做将帮助您避免与复制和粘贴代码相关的任何潜在错误。**

对于本书，我们将以名为`llkd`的用户登录。我强烈建议您遵循*经验主义方法：不要轻信任何人的话，而是尝试并亲身体验。*因此，本书为您提供了许多实践实验和内核驱动程序代码示例，您可以并且必须自己尝试；这将极大地帮助您取得实质性进展，并深入学习和理解 Linux 驱动程序/内核开发的各个方面。

## 下载示例代码文件

您可以从 GitHub 下载本书的示例代码文件，网址为[`github.com/PacktPublishing/Linux-Kernel-Programming-Part-2`](https://github.com/PacktPublishing/Linux-Kernel-Programming-Part-2)。如果代码有更新，将在现有的 GitHub 存储库上进行更新。

我们还提供来自我们丰富的图书和视频目录的其他代码包，可在**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**上找到。去看看吧！

## 下载彩色图片

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在此处下载：[`www.packtpub.com/sites/default/files/downloads/9781801079518_ColorImages.pdf`](http://www.packtpub.com/sites/default/files/downloads/9781801079518_ColorImages.pdf)。

## 使用的约定

本书中使用了许多文本约定。

`CodeInText`：指示文本中的代码字词，数据库表名，文件夹名，文件名，文件扩展名，路径名，虚拟 URL，用户输入和 Twitter 句柄。这是一个例子：“`ioremap()` API 返回`void *`类型的 KVA（因为它是一个地址位置）。”

代码块设置如下：

```
static int __init miscdrv_init(void)
{
    int ret;
    struct device *dev;
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目将以粗体显示：

```
#define pr_fmt(fmt) "%s:%s(): " fmt, KBUILD_MODNAME, __func__
[...]
#include <linux/miscdevice.h>
#include <linux/fs.h>             
[...]
```

任何命令行输入或输出都以以下方式编写：

```
pi@raspberrypi:~ $ sudo cat /proc/iomem
```

**粗体**：表示一个新术语，一个重要词，或者你在屏幕上看到的词。例如，菜单或对话框中的单词会以这种方式出现在文本中。这是一个例子：“从管理面板中选择“系统信息”。”

警告或重要说明看起来像这样。

提示和技巧看起来像这样。

# 取得联系

我们的读者的反馈总是受欢迎的。

**一般反馈**：如果您对本书的任何方面有疑问，请在消息主题中提及书名，并发送电子邮件至`customercare@packtpub.com`。

**勘误**：尽管我们已经尽一切努力确保内容的准确性，但错误确实会发生。如果您在这本书中发现了错误，我们将不胜感激，如果您能向我们报告。请访问[www.packtpub.com/support/errata](https://www.packtpub.com/support/errata)，选择您的书，点击勘误提交表格链接，并输入详细信息。

**盗版**：如果您在互联网上发现我们作品的任何形式的非法副本，我们将不胜感激，如果您能向我们提供位置地址或网站名称。请通过`copyright@packt.com`与我们联系，并附上材料的链接。

**如果您有兴趣成为作者**：如果有一个您在某个专题上有专业知识，并且您有兴趣写作或为一本书做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

## 评论

请留下评论。一旦您阅读并使用了这本书，为什么不在购买它的网站上留下评论呢？潜在的读者可以看到并使用您的公正意见来做出购买决定，我们在 Packt 可以了解您对我们产品的看法，我们的作者可以看到您对他们的书的反馈。谢谢！

有关 Packt 的更多信息，请访问[packt.com](http://www.packt.com/)。
