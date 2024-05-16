# 前言

欢迎阅读《架构师的 Linux 实战》，深入了解架构师在处理基于 Linux 的解决方案时的思维过程。本书将帮助您达到架构和实施不同 IT 解决方案所需的知识水平。

此外，它将向您展示开源软件的灵活性，通过演示行业中最广泛使用的一些产品，为您提供解决方案并分析每个方面，从设计阶段的最开始，一直到实施阶段，在那里我们将从零开始构建我们设计中提出的基础设施。

深入探讨设计解决方案的技术方面，我们将详细剖析每个方面，以实施和调整基于 Linux 的开源解决方案。

# 这本书是为谁写的

本书面向 Linux 系统管理员、Linux 支持工程师、DevOps 工程师、Linux 顾问以及任何其他类型的希望学习或扩展基于 Linux 和开源软件的架构、设计和实施解决方案的技术专业人士。

# 本书涵盖了什么

第一章，*设计方法论简介*，通过分析提出的问题以及在设计解决方案时应该提出的正确问题，来提取必要的信息以定义正确的问题陈述。

第二章，*定义 GlusterFS 存储*，介绍了 GlusterFS 是什么，并定义了存储集群。

第三章，*设计存储集群*，探讨了使用 GlusterFS 及其各种组件实施集群存储解决方案的设计方面。

第四章，*在云基础设施上使用 GlusterFS*，解释了在云上实施 GlusterFS 所需的配置。

第五章，*分析 Gluster 系统的性能*，详细介绍了先前配置的解决方案，解释了所采取的配置以及对性能进行测试的实施。

第六章，*创建高可用的自愈架构*，讨论了 IT 行业是如何从使用单片应用程序发展为基于云的、容器化的、高可用的微服务的。

第七章，*理解 Kubernetes 集群的核心组件*，探讨了核心 Kubernetes 组件，展示了每个组件以及它们如何帮助我们解决客户的问题。

第八章，*设计 Kubernetes 集群*，深入探讨了 Kubernetes 集群的要求和配置。

第九章，*部署和配置 Kubernetes*，介绍了 Kubernetes 集群的实际安装和配置。

第十章，*使用 ELK Stack 进行监控*，解释了弹性堆栈的每个组件以及它们的连接方式。

第十一章，*设计 ELK Stack*，涵盖了部署弹性堆栈时的设计考虑。

第十二章，*使用 Elasticsearch、Logstash 和 Kibana 管理日志*，描述了弹性堆栈的实施、安装和配置。

第十三章，*使用 Salty 解决管理问题*，讨论了业务需要具有用于基础设施的集中管理实用程序的需求，例如 Salt。

第十四章，*Getting Your Hands Salty*，介绍了如何安装和配置 Salt。

第十五章，*设计最佳实践*，介绍了设计具有弹性和防故障解决方案所需的一些不同最佳实践。

# 充分利用本书

需要一些基本的 Linux 知识，因为本书不解释 Linux 管理的基础知识。

本书中给出的示例可以在云端或本地部署。其中一些设置是在微软的云平台 Azure 上部署的，因此建议您拥有 Azure 帐户以便跟随示例。Azure 提供免费试用以评估和测试部署，更多信息可以在[`azure.microsoft.com/free/`](https://azure.microsoft.com/free/)找到。此外，有关 Azure 的更多信息可以在[`azure.microsoft.com`](https://azure.microsoft.com)找到。

由于本书完全围绕 Linux 展开，因此连接到互联网是必需的。这可以从 Linux 桌面（或笔记本电脑）、macOS 终端或**Windows 子系统**（**WSL**）中完成。

本书中所示的所有示例都使用可以轻松从可用存储库或各自的来源获得的开源软件，无需支付许可证。

一定要访问项目页面以表达一些爱意——开发这些项目需要付出很多努力：

+   [`github.com/gluster/glusterfs`](https://github.com/gluster/glusterfs)

+   [`github.com/zfsonlinux/zfs`](https://github.com/zfsonlinux/zfs)

+   [`github.com/kubernetes/kubernetes`](https://github.com/kubernetes/kubernetes)

+   [`github.com/elastic/elasticsearch`](https://github.com/elastic/elasticsearch)

+   [`github.com/saltstack/salt`](https://github.com/saltstack/salt)

# 下载示例代码文件

您可以从[www.packt.com](http://www.packt.com)的帐户中下载本书的示例代码文件。如果您在其他地方购买了本书，可以访问[www.packt.com/support](http://www.packt.com/support)并注册，以便直接通过电子邮件接收文件。

您可以按照以下步骤下载代码文件：

1.  请登录或注册[www.packt.com](http://www.packt.com)。

1.  选择“支持”选项卡。

1.  单击“代码下载和勘误”。

1.  在搜索框中输入书名，然后按照屏幕上的说明操作。

下载文件后，请确保使用最新版本的解压缩软件解压缩文件夹：

+   Windows 上的 WinRAR/7-Zip

+   Mac 上的 Zipeg/iZip/UnRarX

+   Linux 上的 7-Zip/PeaZip

该书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/-Hands-On-Linux-for-Architects`](https://github.com/PacktPublishing/-Hands-On-Linux-for-Architects)。如果代码有更新，将在现有的 GitHub 存储库上进行更新。

我们还有来自丰富书籍和视频目录的其他代码包，可以在**[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)**上找到。去看看吧！

# 下载彩色图像

我们还提供了一个 PDF 文件，其中包含本书中使用的屏幕截图/图表的彩色图像。您可以在此处下载：[`www.packtpub.com/sites/default/files/downloads/9781789534108_ColorImages.pdf`](https://www.packtpub.com/sites/default/files/downloads/9781789534108_ColorImages.pdf)。

# 使用的约定

本书中使用了许多文本约定。

`CodeInText`：表示文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 用户名。这是一个例子：“该命令中的两个关键点是`address-prefix`标志和`subnet-prefix`标志。”

代码块设置如下：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: gluster-pvc  
spec:
 accessModes:
 - ReadWriteMany      
 resources:
    requests:
      storage: 1Gi  
```

当我们希望引起您对代码块的特定部分的注意时，相关行或项目会以粗体显示：

```
 SHELL ["/bin/bash", "-c"]
 RUN echo "Hello I'm using bash" 
```

任何命令行输入或输出都会以以下方式编写：

```
yum install -y zfs
```

**粗体**：表示一个新术语，一个重要单词，或者您在屏幕上看到的单词。例如，菜单或对话框中的单词会以这种方式出现在文本中。这里有一个例子：“要确认数据是否被发送到集群，请转到 kibana 屏幕上的 Discover”。

警告或重要说明会显示为这样。

提示和技巧会显示为这样。

# 联系我们

我们始终欢迎读者的反馈。

**一般反馈**：如果您对本书的任何方面有疑问，请在邮件主题中提及书名，并发送电子邮件至`customercare@packtpub.com`。

**勘误**：尽管我们已经尽一切努力确保内容的准确性，但错误确实会发生。如果您在这本书中发现了错误，我们将不胜感激，如果您能向我们报告。请访问[www.packt.com/submit-errata](http://www.packt.com/submit-errata)，选择您的书，点击勘误提交表格链接，并输入详细信息。

**盗版**：如果您在互联网上发现我们作品的任何形式的非法副本，我们将不胜感激，如果您能向我们提供位置地址或网站名称。请通过链接`copyright@packt.com`与我们联系。

**如果您有兴趣成为作者**：如果您在某个专题上有专业知识，并且有兴趣撰写或为一本书做出贡献，请访问[authors.packtpub.com](http://authors.packtpub.com/)。

# 评论

请留下评论。一旦您阅读并使用了这本书，为什么不在购买它的网站上留下评论呢？潜在的读者可以看到并使用您的客观意见来做出购买决定，我们在 Packt 可以了解您对我们产品的看法，我们的作者可以看到您对他们的书的反馈。谢谢！

有关 Packt 的更多信息，请访问[packt.com](http://www.packt.com/)。
