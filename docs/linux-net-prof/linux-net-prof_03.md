# 第二章：*第二章*：基本 Linux 网络配置和操作-处理本地接口

在本章中，我们将探讨如何在 Linux 主机上显示和配置本地接口和路由。尽可能地，我们将讨论执行这些操作的新旧命令。这将包括显示和修改 IP 地址、本地路由和其他接口参数。在此过程中，我们将讨论如何使用二进制方法构建 IP 地址和子网地址。

本章应该为我们在后续章节中涵盖的主题，如故障排除网络问题、加固我们的主机和安装安全服务，奠定坚实的基础。

本章涵盖的主题如下：

+   处理网络设置-两组命令

+   显示接口 IP 信息

+   IPv4 地址和子网掩码

+   为接口分配 IP 地址

# 技术要求

在本章和其他每一章中，当我们讨论各种命令时，鼓励您在自己的计算机上尝试。本书中的命令都是在 Ubuntu Linux 20 版（长期支持版）上演示的，但在几乎任何 Linux 发行版上，这些命令应该基本相同或非常相似。

# 处理网络设置-两组命令

在大多数人熟悉的 Linux 寿命中，**ifconfig**（接口配置）和相关命令一直是 Linux 操作系统的主要组成部分，以至于现在在大多数发行版中已经被弃用，但仍然是许多系统和网络管理员的常用命令。

为什么要替换这些旧的网络命令？有几个原因。一些新硬件（特别是 InfiniBand 网络适配器）不受旧命令的良好支持。此外，随着 Linux 内核多年来的变化，旧命令的操作随着时间的推移变得越来越不一致，但是在向后兼容性方面的压力使得解决这个问题变得困难。

旧命令在`net-tools`软件包中，新命令在`iproute2`软件包中。新管理员应该专注于新命令，但熟悉旧命令仍然是一件好事。仍然很常见的是发现运行 Linux 的旧计算机，这些机器可能永远不会更新，仍然使用旧命令。因此，我们将涵盖两种工具集。

这个教训是，在 Linux 世界中，变化是不断的。旧命令仍然可用，但不是默认安装的。

要安装旧命令，请使用此命令：

```
robv@ubuntu:~$ sudo apt install net-tools
 [sudo] password for robv:
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following package was automatically installed and is no longer required:
  libfprint-2-tod1
Use 'sudo apt autoremove' to remove it.
The following NEW packages will be installed:
  net-tools
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 0 B/196 kB of archives.
After this operation, 864 kB of additional disk space will be used.
Selecting previously unselected package net-tools.
(Reading database ... 183312 files and directories currently installed.)
Preparing to unpack .../net-tools_1.60+git20180626.aebd88e-1ubuntu1_amd64.deb ..                                      .
Unpacking net-tools (1.60+git20180626.aebd88e-1ubuntu1) ...
Setting up net-tools (1.60+git20180626.aebd88e-1ubuntu1) ...
Processing triggers for man-db (2.9.1-1) ...
```

您可能会注意到`install`命令及其输出中的一些内容：

+   `sudo`：使用了`sudo`命令-`/etc/sudoers`。在大多数发行版中，默认情况下，在安装操作系统时定义的`userid`会自动包含在该文件中。可以使用`visudo`命令添加其他用户或组。

为什么使用`sudo`？安装软件或更改网络参数以及许多其他系统操作都需要提升的权限-在多用户企业系统上，您不希望非管理员的人进行这些更改。

因此，如果`sudo`如此强大，为什么我们不以 root 身份运行所有操作呢？主要是因为这是一个安全问题。当然，如果您拥有 root 权限，一切都会正常工作。但是，任何错误和打字错误都可能导致灾难性的结果。此外，如果您以正确的权限运行并且碰巧执行了一些恶意软件，那么恶意软件将具有相同的权限，这显然不理想！如果有人问，是的，Linux 恶意软件确实存在，并且从一开始就一直存在于操作系统中。

+   `apt`：使用`apt`命令 - `apt`是 Ubuntu、Debian 和相关发行版上的默认安装程序，但不同发行版之间的软件包管理应用程序会有所不同。除了`apt`及其等效命令外，仍然支持从下载文件进行安装。Debian、Ubuntu 和相关发行版使用`deb`文件，而许多其他发行版使用`rpm`文件。总结如下：

![](img/Table_1.jpg)

所以，现在我们有了一大堆新命令要看，我们如何获取更多关于它们的信息呢？`man`（手册）命令在 Linux 中有大多数命令和操作的文档。例如，`apt`的`man`命令可以使用`man apt`命令打印；输出如下：

![图 2.1 - apt man 页面](img/B16336_02_001.jpg)

图 2.1 - apt man 页面

在本书中介绍新命令时，花点时间使用`man`命令进行复习 - 本书更多是为了指导你的旅程，而不是替代实际操作系统文档。

现在我们已经讨论了现代和传统工具，然后安装了传统的`net-tools`命令，那么这些命令是什么，它们是做什么的呢？

# 显示接口 IP 信息

在 Linux 工作站上显示接口信息是一个常见的任务。特别是如果你的主机适配器被设置为自动配置，例如使用**动态主机配置协议**（**DHCP**）或 IPv6 自动配置。

正如我们讨论过的，有两组命令可以做到这一点。`ip`命令允许我们在新操作系统上显示或配置主机的网络参数。在旧版本中，你会发现使用`ifconfig`命令。

`ip`命令将允许我们显示或更新 IP 地址、路由信息和其他网络信息。例如，要显示当前 IP 地址信息，请使用以下命令：

```
ip address
```

`ip`命令支持`ip addr`或甚至`ip a`都会给你相同的结果：

```
robv@ubuntu:~$ ip ad
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:33:2d:05 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.182/24 brd 192.168.122.255 scope global dynamic noprefixroute ens33
       valid_lft 6594sec preferred_lft 6594sec
    inet6 fe80::1ed6:5b7f:5106:1509/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

你会发现，即使是最简单的命令有时也会返回比你想要的更多的信息。例如，你会看到命令行选项中的`-4`或`-6`：

```
robv@ubuntu:~$ ip -4 ad
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    inet 192.168.122.182/24 brd 192.168.122.255 scope global dynamic noprefixroute ens33
       valid_lft 6386sec preferred_lft 6386sec
```

在这个输出中，你会看到`loopback`接口（一个逻辑的内部接口）的 IP 地址是`127.0.0.1`，以及以太网接口`ens33`的 IP 地址是`192.168.122.182`。

现在是一个绝佳的时机输入`man ip`并复习我们可以使用这个命令做的各种操作：

![图 2.2 - ip man 页面](img/B16336_02_002.jpg)

图 2.2 - ip man 页面

`ifconfig`命令与`ip`命令有非常相似的功能，但正如我们所指出的，它主要出现在旧版本的 Linux 上。传统命令都是有机地发展起来的，功能都是按需添加的。这导致我们处于一个状态，随着显示或配置更复杂的事物，语法变得越来越不一致。而更现代的命令是从头开始设计的，以保持一致性。

让我们使用传统命令来重复我们的努力；要显示接口 IP，只需输入`ifconfig`：

```
robv@ubuntu:~$ ifconfig
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1400
        inet 192.168.122.22  netmask 255.255.255.0  broadcast 192.168.122.255
        inet6 fe80::1ed6:5b7f:5106:1509  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:33:2d:05  txqueuelen 1000  (Ethernet)
        RX packets 161665  bytes 30697457 (30.6 MB)
        RX errors 0  dropped 910  overruns 0  frame 0
        TX packets 5807  bytes 596427 (596.4 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 1030  bytes 91657 (91.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1030  bytes 91657 (91.6 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

正如你所看到的，基本上是以稍微不同的格式显示相同的信息。如果你查看这两个命令的`man`页面，你会发现`ip`命令的选项更一致，并且没有太多的 IPv6 支持 - 例如，原生情况下你无法选择只显示 IPv4 或 IPv6。

## 显示路由信息

在现代网络命令中，我们将使用完全相同的`ip`命令来显示我们的路由信息。而且，正如你所期望的，命令是`ip route`，可以缩写为`ip r`：

```
robv@ubuntu:~$ ip route
default via 192.168.122.1 dev ens33 proto dhcp metric 100
169.254.0.0/16 dev ens33 scope link metric 1000
192.168.122.0/24 dev ens33 proto kernel scope link src 192.168.122.156 metric 100
robv@ubuntu:~$ ip r
default via 192.168.122.1 dev ens33 proto dhcp metric 100
169.254.0.0/16 dev ens33 scope link metric 1000
192.168.122.0/24 dev ens33 proto kernel scope link src 192.168.122.156 metric 100
```

从这个输出中，我们看到我们有一个指向`192.168.122.1`的*默认路由*。默认路由就是这样 - 如果一个数据包被发送到不在路由表中的目的地，主机将把该数据包发送到其默认网关。路由表总是会优先选择“最具体”的路由 - 最接近目的地 IP 的路由。如果没有匹配，那么最具体的路由就会走向默认网关，它的路由是`0.0.0.0 0.0.0.0`（换句话说，如果它不匹配其他任何东西的路由）。主机假设默认网关 IP 属于路由器，然后（希望）知道下一步该把该数据包发送到哪里。

我们还看到了到`169.254.0.0/16`的路由。这被称为**链路本地地址**，在 RFC 3927 中定义。**RFC**代表**请求评论**，它作为互联网标准在开发过程中使用的非正式同行评审的一部分。已发布的 RFC 列表由**IETF**（**互联网工程任务组**）维护，网址为 https://www.ietf.org/standards/rfcs/。

链路本地地址只在当前子网中运行 - 如果主机没有静态配置的 IP 地址，并且 DHCP 没有分配地址，它将使用 RFC 中定义的前两个八位字节（`169.254`），然后计算最后两个八位字节，将它们随机分配。经过 Ping/ARP 测试（我们将在*第三章*中讨论 ARP，*使用 Linux 和 Linux 工具进行网络诊断*），以确保这个计算出的地址实际上是可用的，主机就准备好进行通信了。这个地址只能与同一网络段上的其他 LLA 地址通信，通常使用广播和多播协议，如 ARP，Alljoyn 等来“找到”彼此。为了澄清，这些地址几乎从不在真实网络中使用，它们是在绝对没有其他选择的情况下使用的地址。为了混淆，微软将这些地址称为不同的东西 - **自动私有互联网协议地址**（**APIPA**）。

最后，我们看到了到本地子网的路由，这种情况下是`192.168.122.0/24`。这被称为**连接路由**（因为它连接到该接口）。这告诉主机在自己的子网中与其他主机通信时不需要进行路由。

这组路由在简单网络中非常常见 - 默认网关，本地段，就是这样。在许多操作系统中，除非主机实际上使用链路本地地址，否则你不会看到`169.254.0.0`子网。

在传统方面，有多种方法可以显示当前的路由集。典型的命令是`netstat -rn`用于*网络状态*，显示路由和数字显示。然而，`route`是一个独立的命令（我们稍后会看到为什么）：

```
robv@ubuntu:~$ netstat -rn
Kernel IP routing table
Destination     Gateway         Genmask         Flags   MSS Window  irtt Iface
0.0.0.0         192.168.122.1   0.0.0.0         UG        0 0          0 ens33
169.254.0.0     0.0.0.0         255.255.0.0     U         0 0          0 ens33
192.168.122.0   0.0.0.0         255.255.255.0   U         0 0          0 ens33
robv@ubuntu:~$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.122.1   0.0.0.0         UG    100     0        0 ens33
169.254.0.0     0.0.0.0         255.255.0.0     U     1000    0        0 ens33
192.168.122.0   0.0.0.0         255.255.255.0   U     100     0        0 ens33
```

这些显示了相同的信息，但现在我们有了两个额外的命令 - `netstat`和`route`。传统的网络工具集往往为每个目的都有一个单独的命令，并且在这种情况下，我们看到了其中两个命令有相当大的重叠。对于新手来说，了解所有这些命令并保持它们不同的语法是一项挑战。`ip`命令集使这变得简单得多！

无论您最终使用哪种工具集，现在您都具备了建立和检查 IP 寻址和路由的基础知识，这将为您的主机提供基本的连接性。

# IPv4 地址和子网掩码

在前一节中，我们简要讨论了 IP 地址，但让我们更详细地讨论一下。IPv4 允许您通过为每个设备分配一个地址和子网掩码来唯一地寻址*子网*中的每个设备。例如，在我们的示例中，IPv4 地址是`192.168.122.182`。IPv4 地址中的每个*八位组*的范围可以是`0-255`，子网掩码是`/24`，通常表示为`255.255.255.0`。直到我们将事情分解为二进制表示，这似乎很复杂。`255`的二进制是`11111111`（8 位），其中有 3 个这样的分组组成 24 位。因此，我们的地址和掩码表示的含义是，当进行掩码处理时，地址的网络部分是`192.168.122.0`，地址的主机部分是`182`，范围是`1-254`。

分解如下：

![](img/Table_2.jpg)

如果我们需要一个更大的子网怎么办？我们只需将掩码向左移动几位。例如，对于 20 位子网掩码，我们有以下内容：

![](img/Table_3.jpg)

这使得掩码的第三个八位组为`0b11110000`（注意简写的`0b`表示“二进制”），在十进制中转换为`240`。这将第三个八位组的网络掩码为`0b01110000`或`112`。这将我们的主机地址范围增加到第三个八位组的`0-15`（`0 – 0b1111`），第四个八位组的`0-255`（`0 – 0b11111111`），或者总共`3824`（15 x 255 – 1）（我们将在下一节讨论`-1`）。

您可以看到，保留一个可以进行二进制到十进制转换的计算器应用对于网络专业人员来说是一个方便的事情！确保它也可以进行十六进制（`16 进制`）转换；我们将在几分钟内深入探讨这一点。

现在我们已经掌握了十进制和特别是二进制中的地址和子网掩码的处理技巧，让我们扩展一下，并探讨如何用它来说明其他寻址概念。

## 特定用途地址

有一些需要涵盖的*特殊用途*地址，以进一步探讨 IP 地址在本地子网中的工作方式。首先，如果地址中的所有主机*位*都设置为`1`，那就是**广播**地址。如果您向广播地址发送信息，它将发送到子网中的所有网络接口并被所有网络接口读取。

因此，在我们的两个例子中，`/24`网络的广播如下：

![](img/Table_4.jpg)

换句话说，我们有一个广播地址为`192.168.122.255`。

`/20`网络的广播如下：

![](img/Table_5.jpg)

或者，我们可以转换回十进制，得到广播地址为`192.168.127.255`。

将 IPv4 地址的网络和主机部分之间的边界移动会让人想起**地址类**的概念。当转换为二进制时，前几个字节定义了该地址的所谓**类别**子网掩码。在大多数操作系统中，如果您在 GUI 中设置 IP 地址，通常会默认填入这个类别子网掩码。这些二进制到子网掩码的分配如下：

![](img/Table_6.jpg)

这定义了网络的默认类别子网掩码。我们将在接下来的两节中深入探讨这一点。

从所有这些内容中，您可以看到为什么大多数管理员使用`255.255.255.0`或`255.255.0.0`。任何其他选择都会在每次添加新成员时变得混乱，并可能导致服务器或工作站配置错误。此外，每次需要设置或解释网络地址时都要“做数学”并不吸引大多数人。

我们刚刚触及的第二种特殊地址是**多播**地址。多播地址用于将多个设备包括在一个对话中。例如，您可以使用多播地址将相同的视频流发送到多个网络连接的显示器，或者如果您正在设置语音/视频应用中的电话会议或会议。本地网络的多播地址采用以下形式：

![](img/Table_7.jpg)

最后的 11 位（3+8）通常形成各种多播协议的“众所周知的地址”。一些常见的多播地址如下：

![](img/Table_8.jpg)

已知的完整注册的多播地址列表由**IANA**（**互联网编号分配机构**）维护，网址为 https://www.iana.org/assignments/multicast-addresses/multicast-addresses.xhtml。虽然这可能看起来很全面，但供应商通常会在这个地址空间中创建自己的多播地址。

这是多播地址的基本介绍——它比这复杂得多，甚至有整本书专门讨论它的设计、实施和理论。我们所讨论的足以让你有一个大致的概念，足以开始。

在广播和多播地址已经涵盖的情况下，让我们讨论在你的环境中最有可能使用的 IP 地址“家族”。

## 私有地址——RFC 1918

另一组特殊地址是 RFC 1918 地址空间。RFC 1918 描述了一系列为组织内部使用而分配的 IP 子网。这些地址不能在公共互联网上使用，因此必须在将流量路由到它们或从它们路由到公共互联网之前使用**网络地址转换**（**NAT**）进行转换。

RFC1918 地址如下：

+   `10.0.0.0/8`（A 类）

+   `172.16.0.0`到`172.31.0.0 / 16`（B 类）（这可以总结为`172.16.0.0/12`）

+   `192.168.0.0/16`（C 类）

这些地址为组织提供了一个大的 IP 空间供内部使用，所有这些地址都保证不会与公共互联网上的任何内容发生冲突。

作为一个有趣的练习，你可以使用这些 RFC 1918 子网来验证默认地址类，方法是将每个子网的第一个八位转换为二进制，然后将它们与最后一节中的表进行比较。

RFC 1918 规范在这里完全记录：https://tools.ietf.org/html/rfc1918。

现在我们已经讨论了 IP 地址和子网掩码的二进制方面，以及各种特殊的 IP 地址组，我相信你已经厌倦了理论和数学，想要回到与 Linux 主机的命令行玩耍！好消息是，我们仍然需要讨论 IPv6（IP 版本 6）的寻址位和字节。更好的消息是，它将在附录中，这样我们就可以让你尽快到键盘上！

现在我们已经牢固掌握了显示 IP 参数并对 IP 地址有了很好的理解，让我们配置一个 IP 接口以供使用。

# 为接口分配 IP 地址

分配永久的 IPv4 地址是你可能需要在构建的每台服务器上做的事情。幸运的是，这很简单。在新的命令集中，我们将使用`nmcli`命令（`manual`）。我们将以`nmcli`格式显示网络连接：

```
robv@ubuntu:~$ sudo nmcli connection show
NAME           UUID                           TYPE       DEVICE
Wired connection 1  02ea4abd-49c9-3291-b028-7dae78b9c968  ethernet  ens33
```

我们的连接名称是`有线连接 1`。不过，我们不需要每次都输入这个，我们可以通过输入`Wi`然后按*Tab*来完成名称的选项。另外，请记住`nmcli`将允许缩短命令子句，所以我们可以使用`mod`代替`modify`，`con`代替`connection`等等。让我们继续我们的命令序列（注意最后一个命令中参数是如何缩短的）：

```
$ sudo nmcli connection modify "Wired connection 1" ipv4.addresses 192.168.122.22/24
$
$ sudo nmcli connection modify "Wired connection 1" ipv4.gateway 192.168.122.1
$
$ sudo nmcli connection modify "Wired connection 1" ipv4.dns "8.8.8.8"
$
$ sudo nmcli con mod "Wired connection 1" ipv4.method manual
$
Now, let's save the changes and make them "live":
$ sudo nmcli connection up "Wired connection 1" 
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
$
```

使用传统方法，我们所有的更改都是通过编辑文件完成的。而且仅仅是为了好玩，文件名和位置会因发行版而异。这里显示了最常见的编辑和文件。

要更改 DNS 服务器，请编辑`/etc/resolv.conf`并更改`nameserver`行以反映所需的服务器 IP：

```
nameserver 8.8.8.8
```

要更改 IP 地址、子网掩码等，编辑`/etc/sysconfig/network-scripts/ifcfg-eth0`文件，并按以下方式更新值：

```
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
NETMASK=255.255.255.0
IPADDR=10.0.1.27
```

如果你的默认网关在这个接口上，你可以添加这个：

```
GATEWAY=192.168.122.1
```

再次注意，在不同的发行版上，要编辑的文件可能会有所不同，特别要注意**这种方法不向后兼容**。在现代 Linux 系统上，编辑基本文件进行网络更改的方法基本上不再适用。

现在我们知道如何为接口分配 IP 地址，让我们学习如何在主机上调整路由。

## 添加路由

要添加临时静态路由，`ip`命令再次是我们的首选。在这个例子中，我们告诉我们的主机路由到`192.168.122.10`以到达`10.10.10.0/24`网络：

```
robv@ubuntu:~$ sudo ip route add 10.10.10.0/24 via 192.168.122.10
 [sudo] password for robv:
robv@ubuntu:~$ ip route
default via 192.168.122.1 dev ens33 proto dhcp metric 100
10.10.10.0/24 via 192.168.122.10 dev ens33
169.254.0.0/16 dev ens33 scope link metric 1000
192.168.122.0/24 dev ens33 proto kernel scope link src 192.168.122.156 metric 100
```

您还可以添加`egress`网络接口以在`ip route add`命令的末尾添加`dev <devicename>`。

然而，这只是添加了一个临时路由，如果主机重新启动或网络进程重新启动，它将无法保存。您可以使用`nmcli`命令添加永久静态路由。

首先，我们将以`nmcli`格式显示网络连接：

```
robv@ubuntu:~$ sudo nmcli connection show
NAME                UUID                                  TYPE       DEVICE
Wired connection 1  02ea4abd-49c9-3291-b028-7dae78b9c968  ethernet  ens33
```

接下来，我们将使用`nmcli`为`Wired connection 1`连接添加路由到`10.10.11.0/24`通过`192.168.122.11`：

```
robv@ubuntu:~$ sudo nmcli connection modify "Wired connection 1" +ipv4.routes "10.10.11.0/24 192.168.122.11"
```

同样，让我们保存我们的`nmcli`更改：

```
$ sudo nmcli connection up "Wired connection 1" 
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
$
```

现在，查看我们的路由表，我们看到了我们的两个静态路由：

```
robv@ubuntu:~$ ip route
default via 192.168.122.1 dev ens33 proto dhcp metric 100
10.10.10.0/24 via 192.168.122.10 dev ens33
10.10.11.0/24 via 192.168.122.11 dev ens33 proto static metric 100
169.254.0.0/16 dev ens33 scope link metric 1000
192.168.122.0/24 dev ens33 proto kernel scope link src 192.168.122.156 metric 100
```

然而，如果我们重新加载，我们会发现我们的临时路由现在已经消失，永久路由已经生效：

```
robv@ubuntu:~$ ip route
default via 192.168.122.1 dev ens33 proto dhcp metric 100
10.10.11.0/24 via 192.168.122.11 dev ens33 proto static metric 100
169.254.0.0/16 dev ens33 scope link metric 1000
192.168.122.0/24 dev ens33 proto kernel scope link src 192.168.122.156 metric 100
```

完成了添加路由的基础知识后，让我们看看如何在旧的 Linux 主机上，使用传统的`route`命令完成同样的任务。

## 使用传统方法添加路由

首先，要添加路由，请使用以下命令：

```
$ sudo route add –net 10.10.12.0 netmask 255.255.255.0 gw 192.168.122.12
```

要使这个路由永久生效，事情变得复杂-永久路由存储在文件中，文件名和位置会因发行版而异，这就是为什么`iproute2/nmcli`命令的一致性在现代系统上如此方便。

在旧的 Debian/Ubuntu 发行版上，一个常见的方法是编辑`/etc/network/interfaces`文件并添加以下行：

```
up route add –net 10.10.12.0 netmask 255.255.255.0 gw 192.168.122.12
```

或者，在旧的 Redhat 系列发行版上，编辑`/etc/sysconfig/network-scripts/route-<device name>`文件并添加以下行：

```
10.10.12.0/24 via 192.168.122.12
```

或者，只需将路由作为命令添加，编辑`/etc/rc.local`文件-这种方法在几乎任何 Linux 系统上都可以工作，但通常被认为不够优雅，主要是因为它是下一个管理员会查找设置的最后一个地方（因为它不是一个正确的网络设置文件）。`rc.local`文件只会在系统启动时执行，并运行其中的任何命令。在这种情况下，我们将添加我们的`route add`命令：

```
/sbin/route add –net 10.10.12.0 netmask 255.255.255.0 gw 192.168.122.12
```

在这一点上，我们已经在我们的 Linux 主机上设置了网络。我们已经设置了 IP 地址、子网掩码和路由。尤其是在故障排除或初始设置时，常常需要禁用或启用接口；我们接下来会涵盖这个内容。

## 禁用和启用接口

在新的命令“世界”中，我们使用-你猜对了-`ip`命令。在这里，我们将“弹跳”接口，先将其关闭，然后再次打开：

```
robv@ubuntu:~$ sudo ip link set ens33 down
robv@ubuntu:~$ sudo ip link set ens33 up
```

在旧的命令集中，使用`ifconfig`来禁用或启用接口：

```
robv@ubuntu:~$ sudo ifconfig ens33 down
robv@ubuntu:~$ sudo ifconfig ens33 up
```

在执行接口命令时，始终要记住，你不想*砍掉你坐在的树枝*。如果你是远程连接的（例如使用`ssh`），如果你改变了`ip`地址或路由，或者禁用了一个接口，你很容易在那一点上失去与主机的连接。

在这一点上，我们已经涵盖了大部分你需要在现代网络中配置你的 Linux 主机的任务。然而，网络管理的一个重要部分是诊断和设置配置以适应特殊情况，例如-调整设置以优化流量，可能需要较小或较大的数据包大小。

## 在接口上设置 MTU

在现代系统中越来越常见的一项操作是设置**消息传输单元**（**MTU**）。这是接口将发送或接收的最大**协议数据报单元**（**PDU**，在大多数网络中也称为**帧**）的大小。在以太网上，默认的 MTU 为 1,500 字节，这相当于最大数据包大小为 1,500 字节。媒体的最大数据包大小通常称为**最大段大小**（**MSS**）。对于以太网，这三个值如下：

![表 2.1 - 以太网的帧大小、MTU、数据包大小和 MSS 的关系](img/Table_10.jpg)

表 2.1 - 以太网的帧大小、MTU、数据包大小和 MSS 的关系

为什么我们需要改变这个？ 1,500 是数据包大小的一个不错的折衷方案，因为它足够小，以至于在出现错误时，错误会被快速检测到，并且需要重新传输的数据量相对较小。然而，特别是在数据中心，有一些例外情况。

在处理存储流量，特别是 iSCSI 时，希望使用较大的帧大小，以便数据包大小可以容纳更多数据。在这些情况下，MTU 通常设置在 9,000 左右（通常称为**巨型数据包**）。这些网络通常部署在 1 Gbps、10 Gbps 或更快的网络上。您还会看到更大的数据包用于适应备份或虚拟机迁移（例如：VMware 中的 VMotion 或 Hyper-V 中的 Live Migration）。

在另一端，您也经常会看到需要较小数据包的情况。这是特别重要的，因为并非所有主机都会很好地检测到这一点，许多应用程序会在其流量中设置**DF**（**不分段**）位。在这种情况下，您可能会看到在可能只支持 1,380 字节数据包的介质上设置了一个 1,500 字节的数据包 - 在这种情况下，应用程序将简单地失败，并且错误消息通常不会对故障排除有所帮助。您可能会在哪里看到这种情况？涉及封装数据包的任何链路通常都会涉及到这一点 - 例如隧道或 VPN 解决方案。这些将通过封装引起的开销减少帧大小（和结果数据包大小），这通常很容易计算。卫星链路是另一种常见情况。它们通常默认为 512 字节的帧 - 在这些情况下，大小将由服务提供商发布。

设置 MTU 就像你想的那样简单 - 我们将再次使用`nmcli`。请注意，在此示例中，我们缩短了`nmcli`的命令行参数，并且我们在最后保存了配置更改 - MTU 在最后一个命令之后立即更改。让我们将 MTU 设置为`9000`以优化 iSCSI 流量：

```
$ sudo nmcli con mod "Wired connection 1" 802-3-ethernet.mtu 9000
$ sudo nmcli connection up "Wired connection 1" 
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/5)
$
```

设置了我们的 MTU，`nmcli`命令还能做什么？

## 有关 nmcli 命令的更多信息

`nmcli`命令也可以以交互方式调用，并且可以在实时解释器或 shell 中进行更改。要进入以太网接口的 shell，请使用`nmcli connection edit type ethernet`命令。在 shell 中，`print`命令列出了可以为该接口类型更改的所有`nmcli`参数。请注意，此输出被分成了逻辑组 - 我们已编辑此（非常冗长）输出，以显示您可能需要在各种情况下调整、编辑或故障排除的许多设置：

```
nmcli> print
===============================================================================
                     Connection profile details (ethernet)
===============================================================================
connection.id:                          ethernet
connection.uuid:                        e0b59700-8dcb-4801-9557-9dee5ab7164f
connection.stable-id:                   --
connection.type:                        802-3-ethernet
connection.interface-name:              --
….
connection.lldp:                        default
connection.mdns:                        -1 (default)
connection.llmnr:                       -1 (default)
-------------------------------------------------------------------------------
```

这些是常见的以太网选项：

```
802-3-ethernet.port:                    --
802-3-ethernet.speed:                   0
802-3-ethernet.duplex:                  --
802-3-ethernet.auto-negotiate:          no
802-3-ethernet.mac-address:             --
802-3-ethernet.mtu:                     auto
….
802-3-ethernet.wake-on-lan:             default
802-3-ethernet.wake-on-lan-password:    --
-------------------------------------------------------------------------------
```

这些是常见的 IPv4 选项：

```
ipv4.method:                            auto
ipv4.dns:                               --
ipv4.dns-search:                        --
ipv4.dns-options:                       --
ipv4.dns-priority:                      0
ipv4.addresses:                         --
ipv4.gateway:                           --
ipv4.routes:                            --
ipv4.route-metric:                      -1
ipv4.route-table:                       0 (unspec)
ipv4.routing-rules:                     --
ipv4.ignore-auto-routes:                no
ipv4.ignore-auto-dns:                   no
ipv4.dhcp-client-id:                    --
ipv4.dhcp-iaid:                         --
ipv4.dhcp-timeout:                      0 (default)
ipv4.dhcp-send-hostname:                yes
ipv4.dhcp-hostname:                     --
ipv4.dhcp-fqdn:                         --
ipv4.dhcp-hostname-flags:               0x0 (none)
ipv4.never-default:                     no
ipv4.may-fail:                          yes
ipv4.dad-timeout:                       -1 (default)
-------------------------------------------------------------------------------
```

（IPv6 选项将在此处列出，但已删除以保持此列表的可读性。）

这些是代理设置：

```
-------------------------------------------------------------------------------
proxy.method:                           none
proxy.browser-only:                     no
proxy.pac-url:                          --
proxy.pac-script:                       --
-------------------------------------------------------------------------------
nmcli>
```

如前所述，列表有些简略。我们展示了您可能需要在各种设置或故障排除情况下检查或调整的设置。在您自己的站点上运行该命令以查看完整的列表。

正如我们所示，`nmcli`命令允许我们交互式地或从命令行调整多个接口参数。特别是命令行界面允许我们在脚本中调整网络设置，从而可以一次性调整数十、数百甚至数千个站点的设置。

# 总结

通过本章的学习，您应该对 IP 地址有一个坚实的理解，从二进制的角度来看。通过这个，您应该理解子网寻址和掩码，以及广播和组播寻址。您还对各种 IP 地址类有了很好的掌握。有了这些，您应该能够使用各种不同的命令在 Linux 主机上显示或设置 IP 地址和路由。其他接口操作也应该很容易实现，比如在接口上设置 MTU。

有了这些技能，您就准备好开始我们下一个话题了：使用 Linux 和 Linux 工具进行网络诊断。

# 问题

最后，这里有一些问题供您测试对本章材料的了解。您将在*附录*的*评估*部分找到答案。

1.  默认网关有什么作用？

1.  对于`192.168.25.0/24`网络，子网掩码和广播地址是什么？

1.  对于同一网络，广播地址是如何使用的？

1.  对于同一网络，可能的主机地址是什么？

1.  如果您需要静态设置以太网接口的速度和双工模式，您会使用什么命令？

# 进一步阅读

+   RFC 1918 – 专用互联网地址分配: [`tools.ietf.org/html/rfc1918`](https://https://tools.ietf.org/html/rfc1918%0D)

+   RFC 791 – 互联网协议: [`tools.ietf.org/html/rfc791`](https://https://tools.ietf.org/html/rfc791)
