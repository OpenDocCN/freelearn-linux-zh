# 第十二章：附录

本章提供了对前几章中提出的所有问题的一套解决方案。如果您已经回答了这些问题，可以检查您的答案的准确性。如果您当时无法找到解决方案，可以参考这里给出的相应章节的答案。

## 第一章：探索 Microsoft Azure 云

1.  您可以虚拟化计算、网络和存储资源。当然，最终，您仍然需要硬件在世界的某个地方运行 hypervisors，可能还需要一个云平台。

1.  虚拟化模拟硬件，容器模拟运行在底层操作系统上的多个容器的操作系统。在虚拟化中，每个虚拟机都有自己的内核；它们不使用 hypervisor/硬件内核。在硬件虚拟化中，一切都被转换为软件。在容器虚拟化中，只有进程被隔离。

1.  这取决于；您是否在同一平台上开发应用程序？如果是这样，那么 PaaS 是适合您的服务类型；否则，请使用 IaaS。SaaS 提供了一个应用程序；它不是一个托管平台。

1.  这取决于。Azure 符合并帮助您遵守法律规定和安全/隐私政策。此外，如果担心数据存储在世界其他地区，还有不同地区的概念。但总是有例外——大多数情况下是公司政策或政府裁决。

1.  对于可伸缩性、性能和冗余性来说非常重要。

1.  这是一个基于云的身份管理服务，用于控制对云和本地混合环境的访问。它允许您登录并访问云和本地环境，而不是使用自己的 AD 服务器并管理它们。

## 第二章：开始使用 Azure 云

1.  它有助于自动化。除此之外，基于 Web 的门户经常更改，而命令行界面更加稳定。在我们看来，这也让您更好地了解底层技术，因为它的工作流程相对严格。

1.  它提供了存储所有数据对象的访问权限。您需要一个用于引导诊断和 Azure Cloud Shell 数据的存储。更多细节可以在*第四章* *管理 Azure*中找到。

1.  在 Azure 中，存储帐户必须是全局唯一的。

1.  报价是由发布者提供的一组相关图像，例如 Ubuntu 服务器。图像是一个特定的图像。

1.  停止的 Azure 虚拟机会保留分配的资源，例如动态公共 IP 地址，并产生成本，而取消分配的虚拟机会释放所有资源，因此停止产生资源成本。但是，两者都会产生存储成本。

1.  基于密钥的身份验证有助于自动化，因为它可以在不在脚本中暴露秘密/密码的情况下使用。

1.  将创建公钥和私钥（如果仍然需要）并存储在您的主目录（`~/.ssh`）中；公钥将被添加到虚拟机中的`authorized_keys`文件中

## 第三章：基本 Linux 管理

1.  `对于用户 Lisa John Karel Carola; useradd $user; done`。

1.  执行`passwd <user>`并输入`welc0meITG`，然后它会要求您再次输入密码以确认，因此再次输入`welc0meITG`。

1.  `getent<user>`。

1.  `groupadd finance;` `groupadd staff`。

1.  `groupmems -g <group_name> -a <user_name>`；或者，`usermod –a –G <group_name> <user_name>`。

1.  要创建目录并设置组所有权，请执行以下操作：

```
mkdir /home/staff
chown staff /home/staff
chgrp staff /home/staff
```

同样，对于`finance`，执行以下命令：

```
mkdir /home/finance
chown finance /home/finance
chgrp finance /home/finance
```

1.  `chmod –R g+r /home/finance`。

1.  默认的获取访问控制列表（`getfacl -d`）将列出用户的 ACL。

## 第四章：管理 Azure

1.  在使用 Azure 门户创建虚拟机时，您不需要任何东西。当您使用命令行时，您需要以下虚拟网络：

资源组

Azure 虚拟网络（VNet）

配置的子网

网络安全组

公共 IP 地址

网络接口

1.  您需要诸如诊断和监控之类的名称服务，这些服务需要存储帐户。

1.  有时（例如，对于存储帐户），名称必须是唯一的。将前缀与随机生成的数字结合在一起是使名称可识别和唯一的好方法。

1.  定义可以在虚拟网络中使用的 IP 范围。

1.  在虚拟网络中创建一个或多个子网，这些子网可以被隔离或路由到彼此，而不必离开虚拟网络。

1.  网络安全组为网络提供 ACL，并为虚拟机或容器提供端口转发。

1.  从虚拟机到互联网的流量通过**源网络地址转换**（**SNAT**）发送。这意味着发出数据包的 IP 地址将被公共 IP 地址替换，这对于 TCP/IP 的出站和入站路由是必需的。

1.  当虚拟机被停止时，动态分配的公共 IP 地址将被释放。当虚拟机再次启动时，它将获得另一个 IP 地址。当必须保持相同的 IP 地址即使服务 IP 地址发生变化时，您可以创建和分配静态公共 IP 地址。

## 第五章：高级 Linux 管理

1.  Linux 内核。

1.  `systemd-udevd`。

1.  `ls /sys/class/net`和`ip link show`。

1.  用于 Linux 的 Azure 代理。

1.  `ls /sys/class/net`和`lsblk`。`lsscsi`命令也可能有所帮助。

1.  使用`RAID0`来提高性能并允许比仅使用单个磁盘更好的吞吐量是一个好主意。

1.  在文件系统级别，使用`mdadm`或**逻辑卷管理器**（**LVM**）（本章未涉及）。

1.  创建 RAID，格式化它，并创建一个挂载点：

```
mdadm --create /dev/md127 --level 0 --raid-devices 3 \    /dev/sd{c,d,e}mkfs.xfs -L myraid /dev/md127 mkdir /mnt/myraid
```

创建一个单元文件，`/etc/systemd/system/mnt-myraid.mount`：

```
[Unit]Description = myRaid volume [Mount]Where = /mnt/myraid What = /dev/md127 Type = xfs [Install]WantedBy = local-fs.mount
```

在启动时启用它：

```
systemctl enable --now mnt-myraid.mount
```

## 第六章：管理 Linux 安全和身份

1.  使用`firewall-cmd`文件或部署`/etc/firewalld`目录。

1.  `--permanent`参数使其在重新启动时持久，并在启动配置期间执行。

1.  在 Linux 中，您可以使用 systemd 中的 ACL 来限制访问。一些应用程序还提供其他主机允许/拒绝选项。在 Azure 中，您可以使用网络安全组和 Azure 防火墙服务。

1.  **自主访问控制**（**DAC**）用于基于用户/组和文件权限限制访问。**强制访问控制**（**MAC**）根据每个资源对象的分类标签进一步限制访问。

1.  如果有人非法访问了一个应用程序或系统，使用 DAC，就没有办法阻止进一步的访问，特别是对于具有相同用户/组所有者和其他用户权限的文件。

1.  每个设备都将有一个唯一的 MAC 地址，您可以使用`ipconfig/ all`找到您的虚拟机的 MAC 地址，然后查找物理地址。

利用 Linux 安全模块的 MAC 框架如下：

SELinux：基于 Red Hat 的发行版和 SUSE

AppArmor：Ubuntu 和 SUSE

较少人知道的 TOMOYO（SUSE）：本书未涉及

1.  除了 SELinux 可以保护更多的资源对象之外，AppArmor 直接与路径一起工作，而 SELinux 通过细粒度访问控制保护整个系统。

1.  加入 AD 域之前，您需要以下先决条件：

用于授权的 Kerberos 客户端

`realm`、`adcli`和`net`命令

## 第七章：部署您的虚拟机

1.  我们使用自动化部署来节省时间，快速建立可重现的环境，并避免手动错误。

1.  除了上一个问题的答案，标准化的工作环境使基于团队的应用程序开发成为可能。

1.  脚本非常灵活。脚本更容易创建，并且可以随时手动调用。自动化过程可以通过事件触发，例如使用`git push`向 Git 添加代码或停止/启动虚拟机。

1.  Azure 资源管理器是最重要的。此外，您可以使用 Terraform、Ansible 和 PowerShell。

1.  Vagrant 在 Azure 中部署工作负载；Packer 创建一个自定义镜像，您可以部署。

1.  由于多种原因，最重要的原因如下：

安全性，使用 CIS 标准加固镜像

当需要对标准镜像进行定制时

不依赖于第三方的提供

捕获现有虚拟机

将快照转换为镜像

1.  您可以通过构建自己的 VHD 文件来创建自己的镜像。以下是这样做的选项：

在 Hyper-V 或 VirtualBox 中创建一个虚拟机，这是 Windows、Linux 和 macOS 上可用的免费 hypervisor。

在 VMware Workstation 或 KVM 中创建虚拟机，并在 Linux qemu-img 中使用它来转换镜像。

## 第八章：探索持续配置自动化

示例脚本可在 GitHub 上找到[`github.com/PacktPublishing/Hands-On-Linux-Administration-on-Azure---Second-Edition/tree/master/chapter12/solutions_chapter08`](https://github.com/PacktPublishing/Hands-On-Linux-Administration-on-Azure---Second-Edition/tree/master/chapter12/solutions_chapter08)。

## 第九章：Azure 中的容器虚拟化

1.  您可以使用容器来打包和分发应用程序，这些应用程序可以是平台无关的。容器消除了虚拟机和操作系统管理的需求，并帮助您实现高可用性和可伸缩性。

1.  如果您有一个需要底层虚拟机所有资源的庞大的单片应用程序，则容器不适用。

1.  **Linux 容器**（**LXCs**）是可以在 Azure 中进行配置的最佳解决方案。

1.  诸如 Buildah 之类的工具使得创建可用于各种解决方案的虚拟机成为可能。Rkt（发音为"rocket"）也支持 Docker 格式。开放容器倡议正在努力创建标准，以使虚拟机的创建变得更加容易。

1.  您可以在 Azure 中开发所有内容，也可以在本地开发，然后推送到远程环境。

1.  它是容器平台无关的，Buildah 工具比其他工具更容易使用。您可以在[`github.com/containers/buildah`](https://github.com/containers/buildah)上进一步探索。

1.  容器可以根据需要构建、替换、停止和销毁，而不会对应用程序或数据产生任何影响，因此不建议在容器中存储任何数据。而是将其存储在卷中。

## 第十章：使用 Azure Kubernetes Service

1.  Pod 是一组具有共享资源（如存储和网络）的容器，以及如何运行容器的规范。

1.  创建多容器 Pod 的一个很好的理由是为了支持主要应用程序的共同管理的辅助进程。

1.  有多种可用的方法，包括 Draft 和 Helm，除了**Azure Kubernetes Service**（**AKS**）还讨论了这些方法。

1.  您可以使用`kubectl`来更新 AKS 中的应用程序。此外，您还可以使用 Helm 和 Draft。

1.  您不需要手动执行此操作；AKS 将自动执行。

1.  当您希望从多个容器同时读/写时，您将需要一个 iSCSI 解决方案和一个集群文件系统。

1.  示例代码可在 GitHub 上找到[`github.com/MicrosoftDocs/azure-docs/blob/master/articles/aks/azure-disks-dynamic-pv.md`](https://github.com/MicrosoftDocs/azure-docs/blob/master/articles/aks/azure-disks-dynamic-pv.md)。

## 第十一章：故障排除和监视工作负载

1.  您可以使用 Azure 串行控制台以 root 身份访问您的虚拟机，除非有特定的阻止。

1.  为了收集所有标准输出、syslog 消息和来自内核、systemd 进程和单元的相关消息。

1.  syslog 使用以下严重性列表（每个应用程序）：

`警报`：必须立即采取行动。

`临界`：临界条件。

`错误`：错误条件。

警告：警告条件。

`注意`：正常但重要的条件。

`信息`：信息消息。

`调试`：调试级别的消息。

1.  0-紧急，1-警报，2-关键，3-错误，4-警告，5-通知，6-信息，7-调试。

1.  使用`logger`或`systemd-cat`。如果应用程序或脚本不支持 syslog，则可以使用它。另一个选项是将日志条目作为变更管理的一部分添加。

1.  Azure Log Analytics 服务用于查看虚拟机的指标。

1.  `top`实用程序有几个缺点；例如，您无法看到短暂的进程。`atop`和`dstat`实用程序是解决这个问题的解决方案。

1.  `sysstat`实用程序提供历史数据；`dstat`提供实时监控。

1.  它使得来自 Azure 虚拟机（工作站）的`tcpdump`数据的收集更易于阅读，并具有很大的分析潜力。
