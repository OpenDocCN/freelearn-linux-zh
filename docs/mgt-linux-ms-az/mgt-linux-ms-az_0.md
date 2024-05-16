# 前言

## 关于

本节简要介绍了作者和审阅者，本书的覆盖范围，您需要开始的技术技能，以及完成所有主题所需的硬件和软件。

## 关于将 Linux 迁移到 Microsoft Azure

随着云采用成为组织数字转型的核心，人们对在云中部署和托管企业业务工作负载的需求很大。《将 Linux 迁移到 Microsoft Azure》提供了一系列可行的见解，可以将 Linux 工作负载部署到 Azure。

您将首先学习 IT 的历史，操作系统，Unix，Linux 和 Windows，然后再转向云和虚拟化之前的情况。这将使那些对 Linux 不太熟悉的人学会掌握接下来章节所需的术语。此外，您将探索流行的 Linux 发行版，包括 RHEL 7，RHEL 8，SLES，Ubuntu Pro，CentOS 7 等。

随着您的进展，您将深入了解 Linux 工作负载的技术细节，如 LAMP，Java 和 SAP。您将学会如何评估当前环境，并通过云治理和运营规划计划迁移到 Azure。

最后，您将进行一个真实的迁移项目，并学会如何分析，调试和解决 Linux 在 Azure 上用户遇到的一些常见问题。

在 Linux 书的最后，您将能够熟练地将 Linux 工作负载有效迁移到 Azure 以供您的组织使用。

### 关于作者

`@rithin-skaria`。

**Toni Willberg**是一位在 Azure 上的 Linux 专家，拥有 25 年的专业 IT 经验。他曾与微软和 Red Hat 合作担任解决方案架构师，帮助客户和合作伙伴进行开源和云之旅。他曾参与 Packt 出版的各种书籍的技术审查。

目前，Toni 担任 Iglu 云业务部门主管，该公司是一家提供专业公共云项目和服务的托管服务提供商。您可以在 Twitter 上与他联系`@ToniWillberg`。

### 关于审阅者

`@Marin Nedea`。

**Micha Wets**是微软 MVP，喜欢谈论 Azure，Powershell 和自动化，并曾在微软会议，国际活动，微软网络研讨会，研讨会等上发表演讲。他拥有超过 15 年的 DevOps 工程师经验，并对混合云和公共云有深入的了解。

今天，Micha 主要专注于 Azure，Powershell，自动化，Azure DevOps，GitHub Actions 和 Windows 虚拟桌面环境，当涉及将这些环境迁移到 Azure 时，他尤其有丰富的知识。Micha 是 Cloud.Architect 的创始人，您可以在 Twitter 上关注他`@michawets`。

### 学习目标

+   探索各种 Linux 发行版的术语和技术

+   了解微软与商业 Linux 供应商之间的技术支持合作

+   通过使用 Azure 迁移来评估当前工作负载

+   计划云治理和运营

+   执行一个真实的迁移项目

+   管理项目，人员配备和客户参与

### 受众

本书旨在使云架构师，云解决方案提供商以及处理将 Linux 工作负载迁移到 Azure 的任何利益相关者受益。熟悉 Microsoft Azure 将是一个优势。

### 方法

《将 Linux 迁移到 Microsoft Azure》使用理论解释和实际示例的理想结合，帮助您为当今企业面临的真实迁移挑战做好准备。

### 硬件和软件要求

**硬件要求**

为了获得最佳的实验室体验，我们建议以下硬件配置：

+   安装了 Hyper-V 角色的 Windows Server 2016，至少 8GB RAM 和 8 核心，用于评估和迁移实验室

**软件要求**

我们还建议您提前进行以下软件配置：

+   Azure 订阅

+   Azure CLI

### 约定

文本中的代码词、数据库名称、文件夹名称、文件名和文件扩展名显示如下。

"您可以使用`wget`命令在 Linux 上下载它，也可以将其下载到您的计算机，然后使用`SFTP/SCP`将其传输到 Linux 机器上。"

以下是一个代码示例：

```
wget --content-disposition https://aka.ms/dependencyagentlinux -O InstallDependencyAgent-Linux64.bin 
sh InstallDependencyAgent-Linux64.bin 
```

由于 Azure 正在以非常快的速度发展，因此可能会出现 Azure 门户中的某些视图或功能与本书中所见的截图不同。我们已经努力确保本书中的截图和所有技术事实在撰写时是正确的。我们在整本书中都提供了官方文档的链接。如果您不确定，请查阅最新的使用指南文档。

### 下载资源

我们还有来自丰富图书和视频目录的其他代码包，可在[`github.com/PacktPublishing/`](https://github.com/PacktPublishing/)上找到。去看看吧！
