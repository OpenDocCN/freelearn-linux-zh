# 前言

软件工程是在微处理器上创建一个存在、生活和呼吸的发明的行为。我们称之为程序。逆向工程是发现程序的生存和呼吸方式的行为，而且它是我们如何使用反汇编器和逆向工具的组合来理解、解剖或修改该程序的行为，并依靠我们的黑客直觉来掌握我们正在逆向工程的目标程序。我们必须了解二进制格式、内存布局和给定处理器的指令集的复杂性。因此，我们成为了微处理器上程序生命的真正主人。逆向工程师擅长二进制掌握的艺术。这本书将为您提供成为 Linux 二进制黑客所需的正确课程、见解和任务。当有人称自己为逆向工程师时，他们将自己提升到了不仅仅是工程师的水平。一个真正的黑客不仅可以编写代码，还可以解剖代码，反汇编二进制文件和内存段，以修改软件程序的内部工作方式；这就是力量……

在专业和业余的层面上，我在计算机安全领域使用我的逆向工程技能，无论是漏洞分析、恶意软件分析、杀毒软件、rootkit 检测还是病毒设计。这本书的很多内容将集中在计算机安全方面。我们将分析内存转储、重建进程映像，并探索一些更神秘的二进制分析领域，包括 Linux 病毒感染和二进制取证。我们将解剖感染恶意软件的可执行文件，并感染运行中的进程。这本书旨在解释在 Linux 中进行逆向工程所需的组件，因此我们将深入学习 ELF（可执行和链接格式），这是 Linux 用于可执行文件、共享库、核心转储和目标文件的二进制格式。这本书最重要的方面之一是它深入洞察了 ELF 二进制格式的结构复杂性。ELF 的部分、段和动态链接概念是重要且令人兴奋的知识点。我们将探索黑客 ELF 二进制的深度，并看到这些技能如何应用于广泛的工作领域。

这本书的目标是教会你成为少数具有 Linux 二进制黑客基础的人之一，这将被揭示为一个广阔的主题，为您打开创新研究的大门，并让您处于 Linux 操作系统低级黑客的前沿。您将获得有关 Linux 二进制（和内存）修补、病毒工程/分析、内核取证和 ELF 二进制格式的宝贵知识。您还将对程序执行和动态链接有更多的见解，并对二进制保护和调试内部有更高的理解。

我是一名计算机安全研究人员、软件工程师和黑客。这本书只是对我所做的研究和作为结果产生的基础知识的有组织的观察和记录。

这些知识涵盖了广泛的信息范围，这些信息在互联网上找不到。这本书试图将许多相关主题汇集到一起，以便作为 Linux 二进制和内存黑客主题的入门手册和参考。它绝不是一个完整的参考，但包含了很多核心信息，可以帮助您入门。

# 这本书涵盖了什么

第一章*Linux 环境及其工具*，简要描述了本书中将使用的 Linux 环境及其工具。

第二章《ELF 二进制格式》帮助你了解 Linux 和大多数 Unix 操作系统中使用的 ELF 二进制格式的每个主要组件。

第三章《Linux 进程跟踪》教你如何使用 ptrace 系统调用来读取和写入进程内存并注入代码。

第四章《ELF 病毒技术- Linux/Unix 病毒》是你发现 Linux 病毒的过去、现在和未来，以及它们是如何设计的，以及围绕它们的所有令人惊奇的研究。

第五章《Linux 二进制保护》解释了 ELF 二进制保护的基本内部原理。

第六章《Linux ELF 二进制取证》是你学习如何解剖 ELF 对象以寻找病毒、后门和可疑的代码注入的地方。

第七章《进程内存取证》向你展示如何解剖进程地址空间，以寻找存储在内存中的恶意软件、后门和可疑的代码注入。

第八章《ECFS-扩展核心文件快照技术》是对 ECFS 的介绍，这是一个用于深度进程内存取证的新开源产品。

第九章《Linux /proc/kcore 分析》展示了如何通过对/proc/kcore 进行内存分析来检测 Linux 内核恶意软件。

# 你需要为这本书准备什么

这本书的先决条件如下：我们假设你具有对 Linux 命令行的工作知识、全面的 C 编程技能，以及对 x86 汇编语言的基本了解（这有帮助但不是必需的）。有一句话说，“如果你能读懂汇编语言，那么一切都是开源的。”

# 这本书适合谁

如果你是软件工程师或逆向工程师，并且想要了解更多关于 Linux 二进制分析的知识，这本书将为你提供在安全、取证和防病毒领域实施二进制分析解决方案所需的一切。这本书非常适合安全爱好者和系统级工程师。我们假设你具有一定的 C 编程语言和 Linux 命令行的经验。

# 惯例

在这本书中，你会发现一些文本样式，用以区分不同类型的信息。以下是一些这些样式的例子及其含义的解释。

文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 用户名显示如下：“有七个节头，从偏移量`0x1118`开始。”

代码块设置如下：

```
uint64_t injection_code(void * vaddr)
{
        volatile void *mem;

        mem = evil_mmap(vaddr,
                        8192,
                        PROT_READ|PROT_WRITE|PROT_EXEC,
                        MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS,
                        -1, 0);

        __asm__ __volatile__("int3");
}
```

当我们希望引起你对代码块的特定部分的注意时，相关的行或项目将以粗体显示：

```
0xb755a990] changed to [0x8048376]
[+] Patched GOT with PLT stubs
Successfully rebuilt ELF object from memory
Output executable location: dumpme.out
[Quenya v0.1@ELFWorkshop]
quit
```

任何命令行输入或输出都以以下形式书写：

```
hacker@ELFWorkshop:~/
workshop/labs/exercise_9$ ./dumpme.out

```

### 注意

警告或重要提示会以这样的方式出现在一个框中。

### 提示

提示和技巧会出现在这样的形式。

# 读者反馈

我们非常欢迎读者的反馈。让我们知道你对这本书的看法——你喜欢或不喜欢什么。读者的反馈对我们很重要，因为它帮助我们开发出你真正能从中获益的标题。

要向我们发送一般反馈，只需通过电子邮件发送 `<feedback@packtpub.com>`，并在主题中提及书名。

如果您在某个专题上有专业知识，并且有兴趣撰写或为一本书做出贡献，请参阅我们的作者指南[www.packtpub.com/authors](http://www.packtpub.com/authors)。

# 客户支持

现在您是 Packt 图书的自豪所有者，我们有很多东西可以帮助您充分利用您的购买。

## 下载示例代码

您可以从[`www.packtpub.com`](http://www.packtpub.com)的帐户中下载您购买的所有 Packt Publishing 图书的示例代码文件。如果您在其他地方购买了这本书，您可以访问[`www.packtpub.com/support`](http://www.packtpub.com/support)并注册，以便文件直接通过电子邮件发送给您。

## 勘误表

尽管我们已经尽一切努力确保内容的准确性，但错误确实会发生。如果您在我们的书中发现错误，也许是文本或代码中的错误，我们将不胜感激，如果您能向我们报告。通过这样做，您可以帮助其他读者避免挫折，并帮助我们改进本书的后续版本。如果您发现任何勘误，请访问[`www.packtpub.com/submit-errata`](http://www.packtpub.com/submit-errata)，选择您的书，点击**勘误提交表格**链接，并输入您的勘误详情。一旦您的勘误被验证，您的提交将被接受，并且勘误将被上传到我们的网站或添加到该标题的**勘误**部分下的任何现有勘误列表中。

要查看先前提交的勘误表，请转到[`www.packtpub.com/books/content/support`](https://www.packtpub.com/books/content/support)并在搜索字段中输入书名。所需信息将出现在**勘误表**部分下。

## 盗版

互联网上侵犯版权材料的盗版是所有媒体的持续问题。在 Packt，我们非常重视版权和许可的保护。如果您在互联网上发现我们作品的任何非法副本，请立即向我们提供位置地址或网站名称，以便我们采取补救措施。

请通过`<copyright@packtpub.com>`与我们联系，并附上涉嫌盗版材料的链接。

我们感谢您在保护我们的作者和为您提供有价值内容的能力方面的帮助。

## 问题

如果您对本书的任何方面有问题，可以通过`<questions@packtpub.com>`与我们联系，我们将尽力解决问题。
