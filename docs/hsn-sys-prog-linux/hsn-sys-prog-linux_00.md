# 前言

Linux 操作系统及其嵌入式和服务器应用程序是当今分散和网络化宇宙中关键软件基础设施的关键组成部分。对熟练的 Linux 开发人员的行业需求不断增加。本书旨在为您提供两方面的内容：扎实的理论基础和实用的、与行业相关的信息——通过代码进行说明——涵盖 Linux 系统编程领域。本书深入探讨了 Linux 系统编程的艺术和科学，包括系统架构、虚拟内存、进程内存和管理、信号、定时器、多线程、调度和文件 I/O。

这本书试图超越使用 API X 来实现 Y 的方法；它着力解释了理解编程接口、设计决策以及有经验的开发人员在使用它们时所做的权衡和背后的原理所需的概念和理论。故障排除技巧和行业最佳实践丰富了本书的内容。通过本书，您将具备与 Linux 系统编程接口一起工作所需的概念知识和实践经验。

# 这本书适合谁

《Linux 系统编程实践》是为 Linux 专业人士准备的：系统工程师、程序员和测试人员（QA）。它也适用于学生；任何想要超越使用 API 集合理解强大的 Linux 系统编程 API 背后的理论基础和概念的人。您应该熟悉 Linux 用户级别的知识，包括登录、通过命令行界面使用 shell 以及使用 find、grep 和 sort 等工具。需要具备 C 编程语言的工作知识。不需要有 Linux 系统编程的先前经验。

# 这本书涵盖了什么

《Linux 系统架构》一章涵盖了关键基础知识：Unix 设计理念和 Linux 系统架构。同时，还涉及了其他重要方面——CPU 特权级别、处理器 ABI 以及系统调用的真正含义。

《虚拟内存》一章澄清了关于虚拟内存的常见误解以及为什么它对现代操作系统设计至关重要；还介绍了进程虚拟地址空间的布局。

《资源限制》一章深入探讨了每个进程资源限制以及管理其使用的 API。

《动态内存分配》一章首先介绍了流行的 malloc API 系列的基础知识，然后深入探讨了更高级的方面，如程序断点、malloc 的真正行为、需求分页、内存锁定和保护，以及使用 alloca 函数。

《Linux 内存问题》一章介绍了（不幸地）普遍存在的内存缺陷，这些缺陷由于对内存 API 的正确设计和使用缺乏理解而出现在我们的项目中。涵盖了未定义行为（一般）、溢出和下溢错误、泄漏等缺陷。

《内存问题调试工具》一章展示了如何利用现有工具，包括编译器本身、Valgrind 和 AddressSanitizer，用于检测前一章中出现的内存问题。

《进程凭证》一章是两章中第一章，重点是让您从系统角度思考和理解安全性和特权。在这里，您将了解传统安全模型——一组进程凭证——以及操作它们的 API。重要的是，还深入探讨了 setuid-root 进程及其安全影响。

第八章，*进程能力*，向你介绍了现代 POSIX 能力模型以及当应用程序开发人员学会使用和利用这一模型而不是传统模型时，安全性可以得到的好处。我们还探讨了能力是什么，如何嵌入它们以及安全性的实际设计。

第九章，*进程执行*，是处理广泛的进程管理领域（执行、创建和信号）的四章中的第一章。在本章中，你将学习 Unix exec 公理的行为方式以及如何使用 API 集（exec 家族）来利用它。

第十章，*进程创建*，深入探讨了`fork(2)`系统调用的行为和使用方法；我们通过七条 fork 规则来描述这一过程。我们还描述了 Unix 的 fork-exec-wait 语义（并深入探讨了等待 API），还涵盖了孤儿进程和僵尸进程。

第十一章，*信号-第一部分*，涉及了 Linux 平台上信号的重要主题：信号的含义、原因和方式。我们在这里介绍了强大的`sigaction(2)`系统调用，以及诸如可重入和信号异步安全性、sigaction 标志、信号堆栈等主题。

第十二章，*信号-第二部分*，继续我们对信号的覆盖，因为这是一个庞大的主题。我们将指导你正确地编写一个处理臭名昭著的致命段错误的信号处理程序，以及处理实时信号、向进程发送信号、使用信号进行进程间通信以及处理信号的其他替代方法。

第十三章，*定时器*，教会你如何在现实世界的 Linux 应用程序中设置和处理定时器这一重要（和与信号相关的）主题。我们首先介绍传统的定时器 API，然后迅速转向现代的 POSIX 间隔定时器以及如何使用它们。我们还介绍并演示了两个有趣的小项目。

第十四章，*使用 Pthreads 进行多线程编程第一部分-基础知识*，是关于在 Linux 上使用 pthread 框架进行多线程编程的三部曲中的第一部分。在这里，我们向你介绍了线程究竟是什么，它与进程的区别，以及使用线程的动机（在设计和性能方面）。本章还指导你了解在 Linux 上编写 pthread 应用程序的基础知识，包括线程的创建、终止、加入等。

第十五章，*使用 Pthreads 进行多线程编程第二部分-同步*，是一个专门讨论同步和竞争预防这一非常重要主题的章节。你将首先了解问题的本质，然后深入探讨原子性、锁定、死锁预防等关键主题。接下来，本章将教你如何使用 pthread 同步 API 来处理互斥锁和条件变量。

第十六章，*使用 Pthreads 进行多线程编程第三部分*，完成了我们关于多线程的工作；我们阐明了线程安全、线程取消和清理以及在多线程应用程序中处理信号的关键主题。我们在本章中讨论了多线程的利弊，并解答了一些常见问题。

第十七章，*Linux 上的 CPU 调度*，向您介绍了系统程序员应该了解的与调度相关的主题。我们涵盖了 Linux 进程/线程状态机，实时概念以及 Linux 操作系统提供的三种（最小）POSIX CPU 调度策略。通过利用可用的 API，您将学习如何在 Linux 上编写软实时应用程序。我们最后简要介绍了一个有趣的事实，即 Linux*可以*被打补丁以作为实时操作系统。

第十八章，*高级文件 I/O*，完全专注于在 Linux 上执行更高级的 IO 以获得最佳性能（因为 IO 通常是瓶颈）。您将简要了解 Linux IO 堆栈的架构（页面缓存至关重要），以及向操作系统提供文件访问模式建议的 API。编写性能 IO 代码，正如您将了解的那样，涉及使用诸如 SG-I/O、内存映射、DIO 和 AIO 等技术。

第十九章，*故障排除和最佳实践*，是对 Linux 故障排除关键要点的重要总结。您将了解到使用强大工具，如 perf 和跟踪工具。然后，本章试图总结一般软件工程和特别是 Linux 编程的关键要点，探讨行业最佳实践。我们认为这些对于任何程序员来说都是至关重要的收获。

[附录 A](https://www.packtpub.com/sites/default/files/downloads/File_IO_Essentials.pdf)，文件 I/O 基础知识，向您介绍了如何在 Linux 平台上执行高效的文件 I/O，通过流式（stdio 库层）API 集以及底层系统调用。在此过程中，还涵盖了有关缓冲及其对性能的影响的重要信息。

请参考本章：[`www.packtpub.com/sites/default/files/downloads/File_IO_Essentials.pdf`](https://www.packtpub.com/sites/default/files/downloads/File_IO_Essentials.pdf)。

[附录 B](https://www.packtpub.com/sites/default/files/downloads/Daemon_Processes.pdf)，守护进程，以简洁的方式向您介绍了 Linux 上守护进程的世界。您将了解如何编写传统的 SysV 风格守护进程。还简要介绍了构建现代新风格守护进程所涉及的内容。

请参考本章：[`www.packtpub.com/sites/default/files/downloads/Daemon_Processes.pdf`](https://www.packtpub.com/sites/default/files/downloads/Daemon_Processes.pdf)。

# 为了充分利用本书

正如前面提到的，本书旨在面向 Linux 软件专业人员——无论是开发人员、程序员、架构师还是 QA 人员，以及希望通过 Linux 操作系统的系统编程主题扩展知识和技能的认真学生。

我们假设您熟悉通过命令行界面、shell 使用 Linux 系统。我们还假设您熟悉使用 C 语言进行编程，知道如何使用编辑器和编译器，并熟悉 Makefile 的基础知识。我们*不*假设您对书中涉及的主题有任何先前的知识。

为了充分利用本书——我们非常明确地指出——您不仅必须阅读材料，还必须积极地动手尝试、修改提供的代码示例，并尝试完成作业！为什么？简单：实践才是真正教会您并内化主题的方法；犯错误并加以修正是学习过程中至关重要的一部分。我们始终主张经验主义方法——不要轻信任何东西。实验，亲自尝试并观察。

因此，我们建议您克隆本书的 GitHub 存储库（请参阅以下部分的说明），浏览文件，并尝试它们。显然，为了进行实验，使用**虚拟机**（**VM**）是绝对推荐的（我们已经在 Ubuntu 18.04 LTS 和 Fedora 27/28 上测试了代码）。书的 GitHub 存储库中还提供了在系统上安装的强制和可选软件包的清单；请阅读并安装所有必需的实用程序，以获得最佳体验。

最后，但绝对不是最不重要的，每一章都有一个*进一步阅读*部分，在这里提到了额外的在线链接和书籍（在某些情况下）；我们建议您浏览这些内容。您将在书的 GitHub 存储库上找到每一章的*进一步阅读*材料。

# 下载示例代码文件

您可以从[www.packt.com](http://www.packtpub.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，您可以访问[www.packt.com/support](http://www.packtpub.com/support)并注册，以便直接将文件发送到您的邮箱。

您可以按照以下步骤下载代码文件：

1.  登录或注册[www.packt.com](http://www.packtpub.com/support)。

1.  选择“支持”选项卡。

1.  单击“代码下载和勘误”。

1.  在搜索框中输入书名并按照屏幕上的说明操作。

文件下载后，请确保使用最新版本的解压缩或提取文件夹：

+   WinRAR/7-Zip for Windows

+   Zipeg/iZip/UnRarX for Mac

+   7-Zip/PeaZip for Linux

该书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Hands-on-System-Programming-with-Linux`](https://github.com/PacktPublishing/Hands-on-System-Programming-with-Linux)。我们还有其他丰富书籍和视频代码包可供查看，网址为**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**。请查看。

# 下载彩色图片

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图片。您可以在此处下载：[`www.packtpub.com/sites/default/files/downloads/9781788998475_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/9781788998475_ColorImages.pdf)

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码单词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。例如：“让我们通过我们的`membugs.c`程序的源代码来检查这些。”

代码块设置如下：

```
include <pthread.h>
int pthread_mutexattr_gettype(const pthread_mutexattr_t *restrict attr,     int *restrict type);
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type);
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目会以粗体显示：

```
include <pthread.h>
int pthread_mutexattr_gettype(const pthread_mutexattr_t *restrict attr,      int *restrict type);
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type);
```

任何命令行输入或输出都以以下方式编写：

```
$ ./membugs 3
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词会以这种方式出现在文本中。例如：“通过下拉菜单选择 C 作为语言。”

警告或重要说明会以这种方式出现。

提示和技巧会以这种方式出现。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**：发送电子邮件至`customercare@packtpub.com`，并在主题中提及书名。如果您对本书的任何方面有疑问，请发送电子邮件至`customercare@packtpub.com`与我们联系。

**勘误**：尽管我们已经尽一切努力确保内容的准确性，但错误确实会发生。如果您在本书中发现错误，我们将不胜感激地向我们报告。请访问[www.packt.com/submit-errata](http://www.packtpub.com/submit-errata)，选择您的书，单击勘误提交表单链接，然后输入详细信息。

**盗版**：如果您在互联网上发现我们作品的任何形式的非法复制，请您提供给我们地址或网站名称，我们将不胜感激。请通过`copyright@packt.com`与我们联系，并附上材料链接。

**如果您有兴趣成为作者**：如果您在某个专题上有专业知识，并且有兴趣撰写或为一本书做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。当您阅读并使用了这本书后，为什么不在购买它的网站上留下评论呢？潜在的读者可以看到并使用您公正的意见来做出购买决定，我们在 Packt 可以了解您对我们产品的看法，我们的作者也可以看到您对他们书籍的反馈。谢谢！

有关 Packt 的更多信息，请访问[packt.com](https://www.packtpub.com/)。
