# 前言

关于当今的 Linux 环境，本书中解释的大多数主题已经可用并且有详细的介绍。本书还涵盖了大量信息，并帮助创建许多观点。当然，本书中还介绍了一些关于各种主题的很好的书籍，并在这里，您将找到对它们的引用。然而，本书的范围并不是再次呈现这些信息，而是在传统的嵌入式开发过程中与 Yocto 项目使用的方法之间进行对比。

本书还介绍了您在嵌入式 Linux 中可能遇到的各种挑战，并为其提出了解决方案。尽管本书旨在面向对其基本 Yocto 和 Linux 技能相当自信并试图改进它们的开发人员，但我相信那些在这个领域没有真正经验的人也可以在这里找到一些有用的信息。

本书围绕您在嵌入式 Linux 之旅中会遇到的各种重要主题构建而成。除此之外，还向您提供了技术信息和许多练习，以确保尽可能多地向您传递信息。在本书结束时，您应该对 Linux 生态系统有一个清晰的认识。

# 本书涵盖的内容

第一章，“介绍”，试图呈现嵌入式 Linux 软件和硬件架构的样子。它还向您介绍了 Linux 和 Yocto 的好处，并提供了示例。它解释了 Yocto 项目的架构以及它是如何集成到 Linux 环境中的。

第二章，“交叉编译”，为您提供了工具链的定义、其组件以及获取方式。之后，向您提供了有关 Poky 存储库的信息，并与组件进行了比较。

第三章，“引导加载程序”，为您提供了引导顺序、U-Boot 引导加载程序以及如何为特定板构建它的信息。之后，它提供了从 Poky 获取 U-Boot 配方的访问权限，并展示了它的使用方法。

第四章，“Linux 内核”，解释了 Linux 内核和源代码的特性。它为您提供了构建内核源代码和模块的信息，然后继续解释 Yocto 内核的配方，并展示了内核引导后发生的相同事情。

第五章，“Linux 根文件系统”，为您提供了有关根文件系统目录和设备驱动程序的组织的信息。它解释了各种文件系统、BusyBox 以及最小文件系统应包含的内容。它将向您展示如何在 Yocto 项目内外编译 BusyBox，以及如何使用 Poky 获取根文件系统。

第六章，“Yocto 项目的组件”，概述了 Yocto 项目的可用组件，其中大部分在 Poky 之外。它提供了每个组件的简介和简要介绍。在本章之后，这些组件中的一些将被更详细地解释。

第七章，“ADT Eclipse 插件”，展示了如何设置 Yocto 项目 Eclipse IDE，为交叉开发和使用 Qemu 进行调试进行设置，并自定义图像并与不同工具进行交互。

第八章，“Hob，Toaster 和 Autobuilder”，介绍了这些工具的每一个，并解释了它们各自的用途，提到了它们的好处。

第九章, *Wic 和其他工具*，解释了如何使用另一组工具，这些工具与前一章提到的工具非常不同。

第十章, *实时*，展示了 Yocto Project 的实时层，它们的目的和附加值。还提供了有关 Preempt-RT、NoHz、用户空间 RTOS、基准测试和其他实时相关功能的文档信息。

第十一章, *安全*，解释了 Yocto Project 的安全相关层，它们的目的以及它们如何为 Poky 增加价值。在这里，您还将获得有关 SELinux 和其他应用程序的信息，例如 bastille、buck-security、nmap 等。

第十二章, *虚拟化*，解释了 Yocto Project 的虚拟化层，它们的目的以及它们如何为 Poky 增加价值。您还将获得有关虚拟化相关软件包和倡议的信息。

第十三章, *CGL 和 LSB*，为您提供了 Carrier Graded Linux (CGL)的规范和要求的信息，以及 Linux Standard Base (LSB)的规范、要求和测试。最后，将与 Yocto Project 提供的支持进行对比。

# 阅读本书需要什么

在阅读本书之前，对嵌入式 Linux 和 Yocto 的先验知识将会有所帮助，尽管不是强制性的。在本书中，有许多练习可供选择，为了完成这些练习，对 GNU/Linux 环境的基本理解将会很有用。此外，一些练习是针对特定的开发板，另一些涉及使用 Qemu。拥有这样的开发板和对 Qemu 的先验知识是一个加分项，但不是强制性的。

在整本书中，有一些章节包含各种练习，需要读者已经具备 C 语言、Python 和 Shell 脚本的知识。如果读者在这些领域有经验，那将会很有帮助，因为它们是当今大多数 Linux 项目中使用的核心技术。我希望这些信息不会在阅读本书内容时让您感到沮丧，希望您会喜欢它。

# 这本书是为谁准备的

这本书是针对 Yocto 和 Linux 爱好者的，他们想要构建嵌入式 Linux 系统，也许还想为社区做出贡献。背景知识应该包括 C 编程技能，以及将 Linux 作为开发平台的经验，对软件开发流程有基本的了解。如果您之前阅读过《使用 Yocto Project 进行嵌入式 Linux 开发》，那也是一个加分项。

看一下技术趋势，Linux 是下一个大事件。它提供了访问尖端开源产品的机会，每天都有更多的嵌入式系统投入使用。Yocto Project 是与嵌入式设备交互的任何项目的最佳选择，因为它提供了丰富的工具集，帮助您将大部分精力和资源投入到产品开发中，而不是重新发明。

# 约定

在本书中，您会发现一些区分不同信息类型的文本样式。以下是一些样式的示例及其含义的解释。

文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下: "一个`maintainers`文件提供了特定板支持的贡献者列表。"

代码块设置如下:

```
sudo add-apt-repository "deb http://archive.ubuntu.com/ubuntu $(lsb_release -sc) universe"
sudo apt-get update
sudo add-apt-repository "deb http://people.linaro.org/~neil.williams/lava jessie main"
sudo apt-get update

sudo apt-get install postgresql
sudo apt-get install lava
sudo a2dissite 000-default
sudo a2ensite lava-server.conf
sudo service apache2 restart
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目将以粗体显示:

```
sudo add-apt-repository "deb http://archive.ubuntu.com/ubuntu $(lsb_release -sc) universe"
sudo apt-get update
sudo add-apt-repository "deb http://people.linaro.org/~neil.williams/lava jessie main"
sudo apt-get update

sudo apt-get install postgresql
sudo apt-get install lava
sudo a2dissite 000-default
sudo a2ensite lava-server.conf
sudo service apache2 restart
```

任何命令行输入或输出都将按照以下格式编写：

```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.1 LTS"

```

**新术语**和**重要单词**以粗体显示。您在屏幕上看到的单词，例如在菜单或对话框中，会以这种方式出现在文本中："如果出现此警告消息，请按**确定**并继续"

### 注意

警告或重要说明会以这种方式出现在框中。

### 提示

技巧和窍门会以这种方式出现。

# 读者反馈

我们的读者的反馈总是受欢迎的。让我们知道您对本书的看法——您喜欢或不喜欢什么。读者的反馈对我们很重要，因为它帮助我们开发您真正能够充分利用的标题。

要向我们发送一般反馈，只需简单地发送电子邮件`<feedback@packtpub.com>`，并在消息主题中提及书名。

如果您在某个专题上有专业知识，并且有兴趣撰写或为书籍做出贡献，请参阅我们的作者指南[www.packtpub.com/authors](http://www.packtpub.com/authors)。

# 客户支持

现在您是 Packt 书籍的自豪所有者，我们有一些事情可以帮助您充分利用您的购买。

## 勘误

尽管我们已经尽一切努力确保内容的准确性，但错误还是会发生。如果您在我们的书中发现错误——可能是文本或代码中的错误——我们将不胜感激，如果您能向我们报告。通过这样做，您可以帮助其他读者避免挫折，并帮助我们改进本书的后续版本。如果您发现任何勘误，请访问[`www.packtpub.com/submit-errata`](http://www.packtpub.com/submit-errata)报告，选择您的书，点击**勘误提交表**链接，并输入您的勘误详情。一旦您的勘误经过验证，您的提交将被接受，并且勘误将被上传到我们的网站或添加到该书标题的勘误部分下的任何现有勘误列表中。

要查看先前提交的勘误，请转到[`www.packtpub.com/books/content/support`](https://www.packtpub.com/books/content/support)并在搜索字段中输入书名。所需信息将出现在**勘误**部分下。

## 盗版

在互联网上盗版受版权保护的材料是所有媒体的持续问题。在 Packt，我们非常重视版权和许可的保护。如果您在互联网上以任何形式发现我们作品的非法副本，请立即向我们提供位置地址或网站名称，以便我们采取补救措施。

请通过`<copyright@packtpub.com>`与我们联系，并附上涉嫌盗版材料的链接。

我们感谢您帮助保护我们的作者和我们为您提供有价值的内容的能力。

## 问题

如果您对本书的任何方面有问题，可以通过`<questions@packtpub.com>`与我们联系，我们将尽力解决问题。
