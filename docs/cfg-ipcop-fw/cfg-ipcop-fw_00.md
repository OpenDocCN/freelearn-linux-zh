# 前言

IPCop 是一个基于 Linux 的有状态防火墙发行版，位于您的互联网连接和网络之间，并使用您制定的一组规则来指导流量。它提供了大多数您期望现代防火墙具有的功能，最重要的是它以高度自动化和简化的方式为您设置了所有这些功能。

本书是一个易于阅读的指南，介绍了 IPCop 在网络中的各种不同角色。本书以非常友好的方式编写，使这个复杂的主题易于阅读和愉快。它首先涵盖了基本的 IPCop 概念，然后介绍了基本的 IPCop 配置，然后涵盖了 IPCop 的高级用法。本书适用于有经验和新手 IPCop 用户。

# 本书内容

第一章简要介绍了一些防火墙和网络概念。该章介绍了几种常见网络设备的角色，并解释了防火墙如何适应其中。

第二章介绍了 IPCop 软件包本身，讨论了 IPCop 的红/橙/蓝/绿接口如何适应网络拓扑。然后涵盖了 IPCop 在其他常见角色中的配置，比如作为 web 代理、DHCP、DNS、时间和 VPN 服务器。

第三章涵盖了三个示例场景，我们将学习如何部署 IPCop，以及 IPCop 接口如何相互连接以及与整个网络的连接。

第四章涵盖了安装 IPCop。它概述了运行 IPCop 所需的系统配置，并解释了使 IPCop 运行所需的配置。

第五章解释了如何使用 IPCop 提供的各种工具来管理、操作、排除故障和监控我们的 IPCop 防火墙。

第六章从解释我们系统中 IDS 的需求开始，然后继续解释如何在 IPCop 中使用 SNORT IDS。

第七章介绍了 VPN 概念，并解释了如何为系统设置 IPSec VPN 配置。特别关注配置蓝区——一个增强无线网络安全性的安全无线网络，即使已经使用 WEP 或 WPA。

第八章演示了如何使用 IPCop 管理带宽，利用流量整形技术和缓存管理。该章还涵盖了 Squid 网页代理和缓存系统的配置。

第九章着重介绍了可用于配置 IPCop 以满足我们需求的各种插件。我们将了解如何安装插件，然后更多地了解像 SquidGuard、Enhanced Filtering、Blue Access、LogSend 和 CopFilter 这样的常见插件。

第十章涵盖了 IPCop 的安全风险，补丁管理以及一些安全和审计工具和测试。

第十一章概述了 IPCop 用户在邮件列表和 IRC 形式上的支持。

# 本书所需内容

IPCop 在专用盒上运行，并且*完全接管硬盘*，因此不要使用有价值数据的硬盘。它可以在旧的或“过时”的硬件上运行，比如 386 处理器，32Mb 的 RAM 和 300Mb 硬盘。但是，如果您计划使用 IPCop 的一些功能，比如缓存网页代理或入侵检测日志，您将需要更多的 RAM，更多的磁盘空间和更快的处理器。

至少需要一个网络接口卡 NIC 来*连接*绿色接口。如果您将通过电缆调制解调器连接到互联网，您将需要两个网卡。

一旦安装，您无需连接显示器或键盘到 IPCop 盒子，因为它作为*无头*服务器运行，并通过网络用 Web 浏览器进行管理。

# 约定

在本书中，您将找到一些文本样式，用于区分不同类型的信息。以下是一些这些样式的示例，以及它们的含义解释。

代码有三种样式。文本中的代码单词显示如下："在 Windows 中，`ipconfig`命令还允许用户释放和更新 DHCP 信息。"

代码块将设置如下：

```
james@horus: ~ $ sudo nmap 10.10.2.32 -T Insane -O
Starting nmap 3.81 ( http://www.insecure.org/nmap/ ) at 2006-05-02 21:36 BST
Interesting ports on 10.10.2.32:
(The 1662 ports scanned but not shown below are in state: closed)
PORT STATE SERVICE
22/tcp open ssh
MAC Address: 00:30:AB:19:23:A9 (Delta Networks)
Device type: general purpose
Running: Linux 2.4.X|2.5.X|2.6.X
OS details: Linux 2.4.18 - 2.6.7
Uptime 0.034 days (since Tue May 2 20:47:15 2006)
Nmap finished: 1 IP address (1 host up) scanned in 8.364 seconds

```

任何命令行输入和输出都将以以下方式显示：

```
# mv /addons /addons.bak
# tar xzvf /addons-2.3-CLI-b2.tar.gz -C /
# cd /addons
# ./addoncfg -u
# ./addoncfg -i 

```

**新术语**和**重要单词**以粗体字体引入。您在屏幕上看到的单词，例如菜单或对话框中的单词，会以这样的方式出现："然后我们返回插件页面，点击**浏览**按钮，浏览到刚刚下载的文件，点击**上传**，插件就安装到服务器上了。"

### 注意

警告或重要提示会以这样的方式出现。

### 注意

技巧和窍门会以这样的方式出现。

# 读者反馈

我们始终欢迎读者的反馈。请让我们知道您对本书的看法，您喜欢或不喜欢的地方。读者的反馈对我们开发能让您真正受益的书籍至关重要。

要向我们发送一般反馈，只需发送电子邮件至`<feedback@packtpub.com>`，请确保在主题中提及书名。

如果有您需要的书籍，并希望我们出版，请在[www.packtpub.com](http://www.packtpub.com)的**建议书名**表单中给我们留言，或发送电子邮件至`<suggest@packtpub.com>`。

如果有您擅长的话题，并且您有兴趣撰写或为一本书做出贡献，请查看我们的作者指南[www.packtpub.com/authors](http://www.packtpub.com/authors)。

# 客户支持

现在您是 Packt 书籍的自豪所有者，我们有一些事情可以帮助您充分利用您的购买。

## 下载书籍示例代码

访问[`www.packtpub.com/support`](http://www.packtpub.com/support)，并从书籍列表中选择本书，以下载本书的示例代码或额外资源。然后将显示可供下载的文件。

可下载的文件包含如何使用它们的说明。

## 勘误

尽管我们已经尽最大努力确保内容的准确性，但错误难免会发生。如果您在我们的书籍中发现错误——可能是文字或代码上的错误——我们将不胜感激，如果您能向我们报告。通过这样做，您可以帮助其他读者避免挫败感，并有助于改进本书的后续版本。如果您发现任何勘误，请访问[`www.packtpub.com/support`](http://www.packtpub.com/support)，选择您的书籍，点击**提交勘误**链接，并输入您的勘误详情。一旦您的勘误经过验证，您的提交将被接受，并将勘误添加到现有勘误列表中。您可以通过选择您的书名从[`www.packtpub.com/support`](http://www.packtpub.com/support)查看现有的勘误。

## 问题

如果您在阅读本书的过程中遇到问题，请联系我们的`<questions@packtpub.com>`，我们将尽力解决。
