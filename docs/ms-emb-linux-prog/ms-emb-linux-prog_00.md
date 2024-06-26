# 前言

嵌入式系统是一种内部带有计算机的设备，看起来不像计算机。洗衣机、电视、打印机、汽车、飞机和机器人都由某种类型的计算机控制，在某些情况下甚至不止一个。随着这些设备变得更加复杂，以及我们对它们的期望不断扩大，控制它们的强大操作系统的需求也在增长。越来越多的情况下，Linux 是首选的操作系统。

Linux 的力量来自于其鼓励代码共享的开源模式。这意味着来自许多不同背景的软件工程师，通常是竞争公司的雇员，可以合作创建一个最新的操作系统内核，并跟踪硬件的发展。从这个代码库中，支持从最大的超级计算机到手表的各种设备。Linux 只是操作系统的一个组成部分。还需要许多其他组件来创建一个工作系统，从基本工具，如命令行，到具有网页内容和与云服务通信的图形用户界面。Linux 内核以及其他大量的开源组件允许您构建一个可以在各种角色中发挥作用的系统。

然而，灵活性是一把双刃剑。虽然它为系统设计者提供了多种解决特定问题的选择，但也带来了如何知道哪种选择最佳的问题。本书的目的是详细描述如何使用免费、开源项目构建嵌入式 Linux 系统，以产生稳健、可靠和高效的系统。它基于作者多年作为顾问和培训师的经验，使用示例来说明最佳实践。

# 本书涵盖的内容

《精通嵌入式 Linux 编程》按照典型嵌入式 Linux 项目的生命周期进行组织。前六章告诉您如何设置项目以及 Linux 系统的构建方式，最终选择适当的 Linux 构建系统。接下来是必须就系统架构和设计选择做出某些关键决策的阶段，包括闪存存储器、设备驱动程序和`init`系统。随后是编写应用程序以利用您构建的嵌入式平台的阶段，其中有两章关于进程、线程和内存管理。最后，我们来到了调试和优化平台的阶段，这在第 12 和 13 章中讨论。最后一章描述了如何为实时应用程序配置 Linux。

第一章，“起步”，通过描述项目开始时系统设计者的选择，为整个故事铺垫。

第二章，“了解工具链”，描述了工具链的组件，重点介绍交叉编译。它描述了在哪里获取工具链，并提供了如何从源代码构建工具链的详细信息。

第三章，“关于引导加载程序”，解释了引导加载程序初始化设备硬件的作用，并以 U-Boot 和 Bareboot 为例进行了说明。它还描述了设备树，这是一种编码硬件配置的方法，用于许多嵌入式系统。

第四章，“移植和配置内核”，提供了如何为嵌入式系统选择 Linux 内核并为设备内部的硬件进行配置的信息。它还涵盖了如何将 Linux 移植到新的硬件上。

第五章，“构建根文件系统”，通过逐步指南介绍了嵌入式 Linux 实现中用户空间部分的概念，以及如何配置根文件系统的方法。

第六章，“选择构建系统”，涵盖了两个嵌入式 Linux 构建系统，它们自动化了前四章描述的步骤，并结束了本书的第一部分。

第七章，“创建存储策略”，讨论了管理闪存存储带来的挑战，包括原始闪存芯片和嵌入式 MMC 或 eMMC 封装。它描述了适用于每种技术类型的文件系统，并涵盖了如何在现场更新设备固件的技术。

第八章，“介绍设备驱动程序”，描述了内核设备驱动程序如何与硬件交互，并提供了简单驱动程序的示例。它还描述了从用户空间调用设备驱动程序的各种方法。

第九章，“启动 - init 程序”，展示了第一个用户空间程序`init`如何启动其余系统。它描述了`init`程序的三个版本，每个版本适用于不同的嵌入式系统组，从 BusyBox `init`到 systemd 的复杂性逐渐增加。

第十章，“了解进程和线程”，从应用程序员的角度描述了嵌入式系统。本章介绍了进程和线程、进程间通信和调度策略。

第十一章，“内存管理”，介绍了虚拟内存背后的思想，以及地址空间如何划分为内存映射。它还涵盖了如何检测正在使用的内存和内存泄漏。

第十二章，“使用 GDB 调试”，向您展示如何使用 GNU 调试器 GDB 交互式调试用户空间和内核代码。它还描述了内核调试器`kdb`。

第十三章，“性能分析和跟踪”，介绍了可用于测量系统性能的技术，从整个系统概要开始，然后逐渐聚焦于导致性能不佳的特定领域。它还描述了 Valgrind 作为检查应用程序对线程同步和内存分配正确性的工具。

第十四章，“实时编程”，提供了关于 Linux 上实时编程的详细指南，包括内核和实时内核补丁的配置，还提供了测量实时延迟的工具描述。它还涵盖了如何通过锁定内存来减少页面错误的信息。

# 本书所需内容

本书使用的软件完全是开源的。在大多数情况下，使用的版本是写作时可用的最新稳定版本。虽然我尽力以不特定于特定版本的方式描述主要特性，但其中的命令示例不可避免地包含一些在较新版本中无法使用的细节。我希望随附的描述足够详细，以便您可以将相同的原则应用于软件包的较新版本。

创建嵌入式系统涉及两个系统：用于交叉编译软件的主机系统和运行软件的目标系统。对于主机系统，我使用了 Ubuntu 14.04，但大多数 Linux 发行版都可以进行少量修改后使用。同样，我不得不选择一个目标系统来代表嵌入式系统。我选择了两个：BeagelBone Black 和 QEMU CPU 模拟器，模拟 ARM 目标。后一个目标意味着您可以尝试示例，而无需投资于实际目标设备的硬件。同时，应该可以将示例应用于广泛的目标，只需根据具体情况进行适应，例如设备名称和内存布局。

目标主要软件包的版本为 U-Boot 2015.07、Linux 4.1、Yocto Project 1.8 "Fido"和 Buildroot 2015.08。

# 这本书适合谁

这本书非常适合已经熟悉嵌入式系统并想要了解如何创建最佳设备的 Linux 开发人员和系统程序员。需要基本的 C 编程理解和系统编程经验。

# 约定

在这本书中，您将找到许多文本样式，用于区分不同类型的信息。以下是这些样式的一些示例以及它们的含义解释。

文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下："我们可以使用流 I/O 函数`fopen(3)`、`fread(3)`和`fclose(3)`。"

代码块设置如下：

```
static struct mtd_partition omap3beagle_nand_partitions[] = {
  /* All the partition sizes are listed in terms of NAND block size */
  {
    .name        = "X-Loader",
    .offset      = 0,
    .size        = 4 * NAND_BLOCK_SIZE,
    .mask_flags  = MTD_WRITEABLE,  /* force read-only */
  }
}
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目会以粗体显示：

```
static struct mtd_partition omap3beagle_nand_partitions[] = {
  /* All the partition sizes are listed in terms of NAND block size */
  {
    .name        = "X-Loader",
    .offset      = 0,
    .size         = 4 * NAND_BLOCK_SIZE,
    .mask_flags  = MTD_WRITEABLE,  /* force read-only */
  }
}
```

任何命令行输入或输出都以以下方式编写：

```
# flash_erase -j /dev/mtd6 0 0
# nandwrite /dev/mtd6 rootfs-sum.jffs2

```

**新术语**和**重要单词**以粗体显示。您在屏幕上看到的单词，例如菜单或对话框中的单词，会以这种方式出现在文本中："第二行在控制台上打印消息**请按 Enter 键激活此控制台**。"

### 注意

警告或重要说明会显示在这样的框中。

### 提示

提示和技巧会显示为这样。

# 读者反馈

我们始终欢迎读者的反馈。让我们知道您对这本书的看法——您喜欢或不喜欢的地方。读者的反馈对我们很重要，因为它可以帮助我们开发出您真正能够充分利用的标题。

要向我们发送一般反馈，只需发送电子邮件至`<feedback@packtpub.com>`，并在邮件主题中提及书名。

如果您在某个专题上有专业知识，并且有兴趣撰写或为书籍做出贡献，请参阅我们的作者指南[www.packtpub.com/authors](http://www.packtpub.com/authors)。

# 客户支持

现在您是 Packt 书籍的自豪所有者，我们有一些事情可以帮助您充分利用您的购买。

## 下载示例代码

您可以从[`www.packtpub.com`](http://www.packtpub.com)的帐户中下载示例代码文件，适用于您购买的所有 Packt Publishing 图书。如果您在其他地方购买了这本书，您可以访问[`www.packtpub.com/support`](http://www.packtpub.com/support)并注册，以便文件直接通过电子邮件发送给您。

## 勘误

尽管我们已经尽最大努力确保内容的准确性，但错误确实会发生。如果您在我们的书籍中发现错误——可能是文本或代码中的错误，我们将不胜感激，如果您能向我们报告。通过这样做，您可以帮助其他读者避免挫折，并帮助我们改进本书的后续版本。如果您发现任何勘误，请访问[`www.packtpub.com/submit-errata`](http://www.packtpub.com/submit-errata)，选择您的书籍，点击**勘误提交表格**链接，并输入您的勘误详情。一旦您的勘误经过验证，您的提交将被接受，并且勘误将被上传到我们的网站或添加到该标题的勘误部分的任何现有勘误列表中。

要查看先前提交的勘误表，请访问[`www.packtpub.com/books/content/support`](https://www.packtpub.com/books/content/support)，并在搜索框中输入书名。所需信息将显示在**勘误表**部分下。

## 盗版

互联网上侵犯版权材料的盗版是跨媒体的持续问题。在 Packt，我们非常重视版权和许可的保护。如果您在互联网上发现我们作品的任何非法副本，请立即向我们提供位置地址或网站名称，以便我们采取补救措施。

请通过`<copyright@packtpub.com>`与我们联系，并提供涉嫌盗版材料的链接。

我们感谢您帮助保护我们的作者和我们提供有价值内容的能力。

## 问题

如果您对本书的任何方面有问题，可以通过`<questions@packtpub.com>`与我们联系，我们将尽力解决问题。
