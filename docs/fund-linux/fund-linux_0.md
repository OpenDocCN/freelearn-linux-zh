# 前言

在这本书中，目标是建立一个扎实的基础，学习 Linux 命令行的所有基本要素，让你入门。它的设计强调只学习实际的核心技能和基本的 Linux 知识，这在开始学习这个美妙的操作系统时非常重要。本课程中展示的所有示例都经过精心选择，是初学者或系统管理员在从零开始时可能遇到的日常和现实世界的任务、用例和问题。我们从虚拟化软件开始我们的旅程，并安装 CentOS 7 Linux 作为虚拟机。然后，我们将温柔地向您介绍最基本的命令行操作，如光标移动、命令、选项和参数、历史记录、引用和通配符、文件流和管道，以及获取帮助，然后向您介绍正则表达式的奥妙以及如何处理文件。然后，演示和解释了最基本的日常 Linux 命令，并提供了对 Bash shell 脚本的简洁介绍。最后，读者将介绍高级主题，如网络、如何排除系统故障、高级文件权限、ACL、setuid、setgid 和粘性位。这只是一个起点，关于 Linux 还有很多东西可以学习。

# 这本书是为谁写的

这本书是为那些希望成为 Linux 系统管理员的个人而写的。

# 本书涵盖了什么

《第一章》*Linux 简介*，向您介绍了 Linux 的一般概念。主题涵盖从虚拟化和 VirtualBox 和 CentOS 的安装，到 VirtualBox 的工作动态，以及与 VirtualBox 的 SSH 连接。

《第二章》*Linux 命令行*，阐明了一系列主题，包括 shell 通配符、命令行操作的介绍、在 Linux 文件系统中导航文件和文件夹的中心思想、不同流的中心思想、正则表达式以及 grep、sed 和 awk 等重要命令。

《第三章》*Linux 文件系统*，着重介绍了系统的工作动态，包括文件链接、用户和组、文件权限、文本文件、文本编辑器以及对 Linux 文件系统的理解。

《第四章》*使用命令行*，带您了解基本的 Linux 命令、信号、附加程序、进程和 Bash shell 脚本。

《第五章》*更高级的命令行和概念*，概述了基本的网络概念、服务、ACL、故障排除、setuid、setgid 和粘性位。

# 从这本书中获得最大的收益

你需要一个基本的实验室设置，至少有 8GB 的 RAM 和双核处理器的系统。如果你计划创建一个虚拟环境，那么建议使用具有相同内存和四核处理器的系统。

对于 Windows 系统，VirtualBox 和 VMware 工作站是最佳选择。对于 Mac 系统，可以在 parallels 上运行测试系统。

在整本书中，我们使用了 CentOS 7 minimal 作为操作系统。

# 下载彩色图片

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图片。您可以在这里下载：[`www.packtpub.com/sites/default/files/downloads/FundamentalsofLinux_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/FundamentalsofLinux_ColorImages.pdf)。

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。例如：“第一个 CentOS 7 VM 服务器现在可以使用 IP `127.0.0.1`和端口`2222`访问，第二个端口`2223`，第三个端口`2224`。”

任何命令行输入或输出都以以下形式编写：

```
# yum update -y 
```

**粗体**：表示新术语、重要单词或屏幕上看到的单词。例如，菜单或对话框中的单词会在文本中以这种方式出现。例如：“选择我们的 CentOS 7 服务器 VM，然后单击绿色的 Start 按钮启动它。”

警告或重要说明会以这种形式出现。

提示和技巧会以这种形式出现。

# 联系我们

我们的读者的反馈总是受欢迎的。

**一般反馈**：发送电子邮件至`feedback@packtpub.com`，并在主题中提及书名。如果您对本书的任何方面有疑问，请发送电子邮件至`questions@packtpub.com`。

**勘误**：尽管我们已经尽一切努力确保内容的准确性，但错误确实会发生。如果您在本书中发现错误，我们将不胜感激，如果您能向我们报告。请访问[www.packtpub.com/submit-errata](http://www.packtpub.com/submit-errata)，选择您的书，点击勘误提交表单链接，并输入详细信息。

**盗版**：如果您在互联网上发现我们作品的任何形式的非法副本，我们将不胜感激，如果您能向我们提供位置地址或网站名称。请通过`copyright@packtpub.com`与我们联系，并附上材料链接。

**如果您有兴趣成为作者**：如果您在某个专题上有专业知识，并且有兴趣撰写或为书籍做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。阅读并使用本书后，为什么不在购买它的网站上留下评论呢？潜在读者可以看到并使用您的客观意见来做出购买决定，我们在 Packt 可以了解您对我们产品的看法，我们的作者也可以看到您对他们书籍的反馈。谢谢！

有关 Packt 的更多信息，请访问[packtpub.com](https://www.packtpub.com/)。
