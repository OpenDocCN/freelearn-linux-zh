# 前言

## 关于

本节简要介绍了作者、本课程的覆盖范围、开始所需的技术技能，以及完成所有包含的活动和练习所需的硬件和软件要求。

## 关于《在 Azure 上进行实践 Linux 管理，第二版》

由于微软 Azure 在提供可扩展的云解决方案方面具有灵活性，它是管理所有工作负载的合适平台。你可以使用它来实现 Linux 虚拟机和容器，并使用开放 API 在开源语言中创建应用程序。

这本 Linux 管理书首先带领你了解 Linux 和 Azure 的基础知识，为后面更高级的 Linux 功能做准备。通过真实世界的例子，你将学习如何在 Azure 中部署虚拟机（VMs），扩展其功能，并有效地管理它们。你将管理容器并使用它们可靠地运行应用程序，在最后一章中，你将探索使用各种开源工具进行故障排除的技术。

通过本书，你将熟练掌握在 Azure 上管理 Linux 和利用部署所需的工具。

### 关于作者

Kamesh Ganesan 是一位云倡导者，拥有近 23 年的 IT 经验，涵盖了 Azure、AWS、GCP 和阿里云等主要云技术。他拥有超过 45 个 IT 认证，包括 5 个 AWS、3 个 Azure 和 3 个 GCP 认证。他担任过多个角色，包括认证的多云架构师、云原生应用架构师、首席数据库管理员和程序分析员。他设计、构建、自动化并交付了高质量、关键性和创新性的技术解决方案，帮助企业、商业和政府客户取得了巨大成功，并显著提高了他们的业务价值，采用了多云策略。

Rithin Skaria 是一位开源倡导者，在 Azure、AWS 和 OpenStack 中管理开源工作负载方面拥有超过 7 年的经验。他目前在微软工作，并参与了微软内部进行的几项开源社区活动。他是认证的微软培训师、Linux 基金会工程师和管理员、Kubernetes 应用程序开发人员和管理员，也是认证的 OpenStack 管理员。在 Azure 方面，他拥有 4 个认证，包括解决方案架构、Azure 管理、DevOps 和安全，他还在 Office 365 管理方面也有认证。他在多个开源部署以及这些工作负载迁移到云端的管理和迁移中发挥了重要作用。

Frederik Vos 居住在荷兰阿姆斯特丹附近的普尔梅伦德市，是一位高级虚拟化技术培训师，专注于 Citrix XenServer 和 VMware vSphere 等虚拟化技术。他专长于数据中心基础设施（虚拟化、网络和存储）和云计算（CloudStack、CloudPlatform、OpenStack 和 Azure）。他还是一位 Linux 培训师和倡导者。他具有教师的知识和系统管理员的实际经验。在过去的 3 年中，他一直在 ITGilde 合作社内担任自由培训师和顾问，为 Linux 基金会提供了许多 Linux 培训课程，比如针对 Azure 的 Linux 培训。

### 学习目标

通过本课程，你将能够：

+   掌握虚拟化和云计算的基础知识

+   了解文件层次结构并挂载新文件系统

+   在 Azure Kubernetes 服务中维护应用程序的生命周期

+   使用 Azure CLI 和 PowerShell 管理资源

+   管理用户、组和文件系统权限

+   使用 Azure 资源管理器重新部署虚拟机

+   实施配置管理以正确配置 VM

+   使用 Docker 构建容器

### 观众

如果您是 Linux 管理员或微软专业人士，希望在 Azure 中部署和管理工作负载，那么这本书适合您。虽然不是必需的，但了解 Linux 和 Azure 将有助于理解核心概念。

### 方法

本书提供了实践和理论知识的结合。它涵盖了引人入胜的现实场景，展示了 Linux 管理员如何在 Azure 平台上工作。每一章都旨在促进每项新技能的实际应用。

### 硬件要求

为了获得最佳的学生体验，我们建议以下硬件配置：

+   处理器：Intel Core i5 或同等级

+   内存：4 GB RAM（建议 8 GB）

+   存储：35 GB 可用空间

### 软件要求

我们还建议您提前准备以下内容：

+   安装有 Linux、Windows 10 或 macOS 操作系统的计算机

+   互联网连接，以便连接到 Azure

### 约定

文本中的代码词、数据库表名、文件夹名、文件名、文件扩展名、路径名、虚拟 URL、用户输入和 Twitter 句柄显示如下：

“以下代码片段创建一个名为`MyResource1`的资源组，并指定 SKU 为`Standard_LRS`，该 SKU 代表了此上下文中的冗余选项。”

以下是一个代码示例块：

```
New-AzStorageAccount -Location westus '
  -ResourceGroupName MyResource1'
  -Name "<NAME>" -SkuName Standard_LRS
```

在许多情况下，我们使用了尖括号，`<>`。您需要用实际参数替换它，而不要在命令中使用这些括号。

### 下载资源

本书的代码包也托管在 GitHub 上，网址为[`github.com/PacktPublishing/Hands-On-Linux-Administration-on-Azure---Second-Edition`](https://github.com/PacktPublishing/Hands-On-Linux-Administration-on-Azure---Second-Edition)。您可以在相关实例中找到本书使用的 YAML 和其他文件。

我们还有其他代码包，来自我们丰富的图书和视频目录，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)上找到。去看看吧！
