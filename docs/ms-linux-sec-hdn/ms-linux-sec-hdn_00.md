# 前言

在这本书中，我们将介绍适用于任何基于 Linux 的服务器或工作站的安全和加固技术。我们的目标是让坏人更难对您的系统进行恶意操作。

# 这本书是为谁准备的

我们的目标读者是一般的 Linux 管理员，无论他们是否专门从事 Linux 安全工作。我们提出的技术可以用于 Linux 服务器或 Linux 工作站。

我们假设我们的目标读者在 Linux 命令行上有一些实践经验，并且具备 Linux 基础知识。

# 这本书涵盖了什么

第一章，*在虚拟环境中运行 Linux*，概述了 IT 安全领域，并告知读者为什么学习 Linux 安全是一个不错的职业选择。我们还将介绍如何建立一个进行实践练习的实验室环境。我们还将展示如何建立一个虚拟实验室环境来进行实践练习。

第二章，*保护用户账户*，介绍了始终使用根用户账户的危险，并将介绍使用 sudo 的好处。然后我们将介绍如何锁定普通用户账户，并确保用户使用高质量的密码。

第三章，*使用防火墙保护服务器*，涉及使用各种类型的防火墙工具。

第四章，*加密和 SSH 加固*，确保重要信息——无论是静态还是传输中的——都受到适当的加密保护。对于传输中的数据，默认的 Secure Shell 配置并不安全，如果保持原样可能会导致安全漏洞。本章介绍了如何解决这个问题。

第五章，*掌握自主访问控制*，介绍了如何设置文件和目录的所有权和权限。我们还将介绍 SUID 和 SGID 对我们有什么作用，以及使用它们的安全影响。最后我们将介绍扩展文件属性。

第六章，*访问控制列表和共享目录管理*，解释了普通的 Linux 文件和目录权限设置并不是非常精细。通过访问控制列表，我们可以只允许特定人访问文件，或者允许多人以不同的权限访问文件。我们还将整合所学知识来管理一个共享目录给一个群组使用。

第七章，*使用 SELinux 和 AppArmor 实施强制访问控制*，讨论了 SELinux，这是包含在 Red Hat 类型的 Linux 发行版中的强制访问控制技术。我们将在这里简要介绍如何使用 SELinux 防止入侵者破坏系统。AppArmor 是另一种包含在 Ubuntu 和 Suse 类型的 Linux 发行版中的强制访问控制技术。我们将在这里简要介绍如何使用 AppArmor 防止入侵者破坏系统。

第八章，*扫描、审计和加固*，讨论了病毒对 Linux 用户来说还不是一个巨大的问题，但对 Windows 用户来说是。如果您的组织有 Windows 客户端访问 Linux 文件服务器，那么这一章就是为您准备的。您可以使用 auditd 来审计对文件、目录或系统调用的访问。它不会防止安全漏洞，但会让您知道是否有未经授权的人试图访问敏感资源。SCAP，即安全内容应用协议，是由国家标准与技术研究所制定的合规性框架。开源实现的 OpenSCAP 可以用来将一个硬化策略应用到 Linux 计算机上。

第九章，《漏洞扫描和入侵检测》，解释了如何扫描我们的系统，以查看我们是否遗漏了任何内容，因为我们已经学会了如何为最佳安全性配置我们的系统。我们还将快速查看入侵检测系统。

第十章，《忙碌蜜蜂的安全提示和技巧》，解释了由于你正在处理安全问题，我们知道你很忙碌。因此，本章向您介绍了一些快速提示和技巧，以帮助您更轻松地完成工作。

# 为了充分利用本书

为了充分利用本书，您不需要太多。但是，以下内容将非常有帮助：

1.  对基本 Linux 命令和如何在 Linux 文件系统中导航有一定的了解。

1.  对于诸如 less 和 grep 之类的工具有基本的了解。

1.  熟悉命令行编辑工具，如 vim 或 nano。

1.  对使用 systemctl 命令控制 systemd 服务有基本的了解。

对于硬件，您不需要任何花哨的东西。您只需要一台能够运行 64 位虚拟机的机器。因此，您可以使用几乎任何一台配备现代 Intel 或 AMD CPU 的主机机器。（这个规则的例外是 Intel Core i3 和 Core i5 CPU。尽管它们是 64 位 CPU，但它们缺乏运行 64 位虚拟机所需的硬件加速。具有讽刺意味的是，远远更老的 Intel Core 2 CPU 和 AMD Opteron CPU 可以正常工作。）对于内存，我建议至少 8GB。

您可以在主机上运行任何三种主要操作系统，因为我们将使用的虚拟化软件适用于 Windows、MacOS 和 Linux。

# 下载彩色图像

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在这里下载：[`www.packtpub.com/sites/default/files/downloads/MasteringLinuxSecurityandHardening_ColorImages.pdf`](http://www.packtpub.com/sites/default/files/downloads/MasteringLinuxSecurityandHardening_ColorImages.pdf)。

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：指示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄。这是一个例子：“让我们使用`getfacl`来查看`acl_demo.txt`文件上是否已经设置了任何访问控制列表。”

代码块设置如下：

```
   [base]
        name=CentOS-$releasever - Base
        mirrorlist=http://mirrorlist.centos.org/?
        release=$releasever&arch=$basearch&repo=os&infra=$infra
          #baseurl=http://mirror.centos.org/centos/
           $releasever/os/$basearch/
        gpgcheck=1
        gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
        priority=1
```

任何命令行输入或输出都以以下方式编写：

```
[donnie@localhost ~]$ tar cJvf new_perm_dir_backup.tar.xz new_perm_dir/ --acls
new_perm_dir/
new_perm_dir/new_file.txt
[donnie@localhost ~]$
```

**粗体**：表示一个新术语、一个重要词或屏幕上看到的词。例如，菜单或对话框中的单词会以这样的方式出现在文本中。这是一个例子：“点击 Network 菜单项，并将 Attached to 设置从 NAT 更改为 Bridged Adapter。”

警告或重要提示会以这种方式出现。

提示和技巧会以这种方式出现。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**：发送电子邮件至`feedback@packtpub.com`，并在主题中提及书名。如果您对本书的任何方面有疑问，请发送电子邮件至`questions@packtpub.com`与我们联系。

**勘误**：尽管我们已经非常小心地确保了内容的准确性，但错误是难免的。如果您在本书中发现了错误，我们将不胜感激，如果您能向我们报告。请访问[www.packtpub.com/submit-errata](http://www.packtpub.com/submit-errata)，选择您的书，点击勘误提交表格链接，并输入详细信息。

**盗版**：如果您在互联网上发现我们作品的任何非法副本，我们将不胜感激，如果您能向我们提供位置地址或网站名称。请通过`copyright@packtpub.com`与我们联系，并附上材料链接。

如果您有兴趣成为一名作者：如果您在某个专业领域有专长，并且有兴趣撰写或为一本书做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。当您阅读并使用了这本书之后，为什么不在购买书籍的网站上留下评论呢？潜在的读者可以看到并使用您的客观意见来做出购买决定，我们在 Packt 可以了解您对我们产品的看法，而我们的作者也可以看到您对他们书籍的反馈。谢谢！

有关 Packt 的更多信息，请访问[packtpub.com](https://www.packtpub.com/)。
