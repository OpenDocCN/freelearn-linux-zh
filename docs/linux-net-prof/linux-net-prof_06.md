# 第四章：*第四章*：Linux 防火墙

Linux 几乎一直都有集成的防火墙可供管理员使用。使用本机防火墙工具，您可以创建传统的周边防火墙，包括地址转换或代理服务器。然而，在现代数据中心，这些并不是典型的用例。现代基础设施中主机防火墙的典型用例如下：

+   入站访问控制，限制对管理界面的访问

+   入站访问控制，限制对其他安装的服务的访问

+   记录访问，以备后续的事件响应，如安全暴露、违规或其他事件。

尽管出站过滤（出站访问控制）当然是建议的，但这更常见地是在网络边界上实施 - 在 VLAN 之间的防火墙和路由器上，或者面向不太受信任的网络，如公共互联网。

在本章中，我们将重点介绍实施一组规则，以管理对实施通用访问的主机的访问，以及对管理员访问的 SSH 服务。

在本章中，我们将涵盖以下主题：

+   配置 iptables

+   配置 nftables

# 技术要求

为了跟随本章的示例，我们将继续在现有的 Ubuntu 主机或虚拟机上进行。本章将重点介绍 Linux 防火墙，因此可能需要第二台主机来测试防火墙更改。

在我们逐步进行各种防火墙配置时，我们将只使用两个主要的 Linux 命令：

![](img/Table_01.jpg)

# 配置 iptables

在撰写本文时（2021 年），我们对防火墙架构还在变化中。iptables 仍然是许多发行版的默认主机防火墙，包括我们的示例 Ubuntu 发行版。然而，该行业已开始向更新的架构 nftables（Netfilter）迈进。例如，红帽和 CentOS v8（在 Linux 内核 4.18 上）将 nftables 作为默认防火墙。仅供参考，当 iptables 在内核版本 3.13 中引入时（大约在 2014 年），它取代了`ipchains`包（该包在 1999 年的内核版本 2.2 中引入）。转移到新命令的主要原因是朝着更一致的命令集前进，提供更好的 IPv6 支持，并使用 API 提供更好的编程支持进行配置操作。

尽管 nftables 架构确实有一些优势（我们将在本章中介绍），但当前的 iptables 方法已经有数十年的惯性。整个自动化框架和产品都是基于 iptables 的。一旦我们进入语法，您会发现这看起来可能是可行的，但请记住，通常情况下，Linux 主机将被部署并使用数十年之久 - 想想收银机、医疗设备、电梯控制或与制造设备（如 PLC）一起工作的主机。在许多情况下，这些长寿命的主机可能没有配置自动更新，因此根据组织的类型，您可能随时可以轻松地预期使用来自 5 年、10 年或 15 年前的完整操作系统版本的主机。此外，由于这些设备的特性，即使它们连接到网络，它们可能不会被列入“计算机”清单。这意味着，尽管默认防火墙从 iptables 迁移到 nftables 可能在任何特定发行版的新版本上迅速进行，但将有大量的遗留主机将继续使用 iptables 多年。

现在我们知道了 iptables 和 nftables 是什么，让我们开始配置它们，首先是 iptables。

## iptables 的高级概述

iptables 是一个 Linux 防火墙应用程序，在大多数现代发行版中默认安装。如果启用了 iptables，它将管理主机的所有流量。防火墙配置位于文本文件中，与您在 Linux 上所期望的一样，它被组织成包含一组规则的表**chains**。

当数据包匹配规则时，规则的结果将是一个目标。目标可以是另一个链，也可以是三个主要操作之一：

+   **接受**：数据包被传递。

+   **丢弃**：数据包被丢弃；不会被传递。

+   **返回**：阻止数据包穿过此链；告诉它返回到上一个链。

其中一个默认表称为**filter**。这个表有三个默认链：

+   **输入**：控制进入主机的数据包

+   **转发**：处理传入的数据包以转发到其他地方。

+   **输出**：处理离开主机的数据包

另外两个默认表是**NAT**和**Mangle**。

像所有新命令一样，查看 iptables 手册页，并快速查看 iptables 帮助文本。为了更容易阅读，您可以通过`less`命令运行帮助文本，使用`iptables -- help | less`。

默认情况下，iptables 默认情况下未配置。我们可以从“iptables –L -v”（用于“list”）中看到三个默认链中没有规则：

```
robv@ubuntu:~$ sudo iptables -L -v
Chain INPUT (policy ACCEPT 254 packets, 43091 bytes)
 pkts bytes target     prot opt in     out     source               destination 
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination 
Chain OUTPUT (policy ACCEPT 146 packets, 18148 bytes)
 pkts bytes target     prot opt in     out     source               destination
```

我们可以看到服务正在运行，尽管`INPUT`和`OUTPUT`链上的数据包和字节数都不为零且在增加。

为了向链中添加规则，我们使用`-A`参数。这个命令可以带几个参数。一些常用的参数如下：

![](img/Table_02.jpg)

因此，例如，这两条规则将允许来自网络`1.2.3.0/24`的主机连接到我们主机的端口`tcp/22`，并且任何东西都可以连接到`tcp/443`：

```
sudo iptables -A INPUT -i ens33 -p tcp  -s 1.2.3.0/24 --dport 22  -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

端口`tcp/22`是 SSH 服务，`tcp/443`是 HTTPS，但如果选择的话，没有什么可以阻止您在任一端口上运行其他服务。当然，如果这些端口上没有任何运行的东西，规则就毫无意义了。

执行完毕后，让我们再次查看我们的规则集。我们将使用`- -line-numbers`添加行号，并使用“-n”（用于数字）跳过地址的任何 DNS 解析：

```
robv@ubuntu:~$ sudo iptables -L -n -v --line-numbers
Chain INPUT (policy ACCEPT 78 packets, 6260 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 ACCEPT     tcp  --  ens33  *       1.2.3.0/24            0.0.0.0/0            tcp dpt:22
2        0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0             0.0.0.0/0            tcp dpt:443
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
Chain OUTPUT (policy ACCEPT 56 packets, 6800 bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

规则列表按顺序从上到下进行处理，因此如果您希望，例如，仅拒绝一个主机访问我们的`https`服务器但允许其他所有内容，您将在`INPUT`规范符号中添加行号。请注意，我们已经在以下代码块的第二条命令中改变了`List`语法-我们只指定了`INPUT`规则，并且还指定了`filter`表（如果您没有指定任何内容，则默认为 filter）：

```
sudo iptables -I INPUT 2 -i ens33 -p tcp  -s 1.2.3.5 --dport 443 -j DROP
robv@ubuntu:~$ sudo iptables -t filter -L INPUT --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     tcp  --  1.2.3.0/24           anywhere              tcp dpt:ssh
2    DROP       tcp  --  1.2.3.5              anywhere              tcp dpt:https
3    ACCEPT     tcp  --  anywhere             anywhere              tcp dpt:https
```

在前面的例子中，我们使用了“-I”参数在链中的特定位置插入规则。然而，如果您已经计划好并且正在按顺序构建规则集，您可能会发现使用“-A”（追加）参数更容易，它将规则追加到列表底部。

在您的源中，您可以定义主机而不是子网，可以只使用 IP 地址（没有掩码）或一系列地址，例如，`--src-range 192.168.122.10-192.168.122.20`。

这个概念可以用来保护服务器上运行的特定服务。例如，通常您会希望限制对允许管理员访问的端口（例如 SSH）的访问仅限于该主机的管理员，但允许更广泛地访问主机上的主要应用程序（例如 HTTPS）。我们刚刚定义的规则是对此的一个开始，假设服务器的管理员在`1.2.3.0/24`子网上。然而，我们错过了阻止其他子网的人连接到 SSH 的“拒绝”：

```
sudo iptables -I INPUT 2 -i ens33 -p tcp  --dport 22 -j DROP
```

这些规则很快就会变得复杂。习惯于将协议规则“分组”是很好的。在我们的例子中，我们将 SSH 保持相邻并按逻辑顺序排列，HTTPS 规则也是如此。您希望每个协议/端口的默认操作都是每个组中的最后一个，前面是例外情况：

```
sudo iptables –L
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     tcp  --  1.2.3.0/24           anywhere              tcp dpt:ssh
2    DROP       tcp  --  anywhere             anywhere              tcp dpt:ssh
3    DROP       tcp  --  1.2.3.5              anywhere              tcp dpt:https
4    ACCEPT     tcp  --  anywhere             anywhere              tcp dpt:https
```

由于规则是按顺序处理的，出于性能原因，您将希望将最频繁“命中”的规则放在列表的顶部。因此，在我们的示例中，我们可能已经反向放置了规则。在许多服务器上，您可能更愿意将应用程序端口（在本例中为`tcp/443`）放在列表的顶部，而将管理员权限（通常看到较低的流量）放在列表的底部。

通过数字删除特定规则（例如，如果我们有一个`INPUT`规则 5），请使用以下命令：

```
sudo iptables –D INPUT 5
```

由于网络管理员应该在本书中保持对安全性的关注，请记住，使用 iptables 限制流量只是过程的前半部分。除非启用了 iptables 日志记录，否则我们无法回顾过去发生的事情。要记录规则，请向其添加`-j LOG`。除了仅记录外，我们还可以使用`- -log-level`参数添加日志级别，并使用`- -log-prefix 'text goes here'`添加一些描述性文本。您可以从中获得什么？

+   记录允许的 SSH 会话可以帮助我们跟踪可能正在对我们主机上的管理服务进行端口扫描的人员。

+   记录被阻止的 SSH 会话可以跟踪试图从非管理员子网连接到管理服务的人员。

+   记录成功和失败的 HTTPS 连接可以帮助我们在故障排除时将 Web 服务器日志与本地防火墙日志相关联。

要仅记录所有内容，请使用以下命令：

```
sudo iptables –A INPUT –j LOG
```

要仅记录来自一个子网的流量，请使用以下命令：

```
sudo iptables –A input –s 192.168.122.0/24 –j LOG
```

要添加日志级别和一些描述性文本，请使用以下命令：

```
sudo iptables -A INPUT –s 192.168.122.0/24 –j LOG - -log-level 3 –log-prefix '*SUSPECT Traffic Rule 9*'
```

日志存储在哪里？在 Ubuntu（我们的示例操作系统）中，它们被添加到`/var/log/kern.log`。在 Red Hat 或 Fedora 中，可以在`/var/log/messages`中找到它们。

我们还应该考虑做什么？就像信息技术中的其他一切一样，如果您可以构建一个东西并让它自行记录，通常可以避免编写单独的文档（通常在完成后几天就过时了）。要添加注释，只需向任何规则添加`-m comment - -comment "Comment Text Here"`。

因此，对于我们的小型四条规则防火墙表，我们将向每条规则添加注释：

```
sudo iptables -A INPUT -i ens33 -p tcp  -s 1.2.3.0/24 --dport 22  -j ACCEPT -m comment --comment "Permit Admin" 
sudo iptables -A INPUT -i ens33 -p tcp  --dport 22  -j DROP -m comment --comment "Block Admin" 
sudo iptables -I INPUT 2 -i ens33 -p tcp  -s 1.2.3.5 --dport 443 -j DROP -m comment --comment "Block inbound Web"
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT -m comment --comment "Permit all Web Access"
sudo iptables -L INPUT
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  1.2.3.0/24           anywhere              tcp dpt:ssh /* Permit Admin */
DROP       tcp  --  anywhere             anywhere              tcp dpt:ssh /* Block Admin */
DROP       tcp  --  1.2.3.5              anywhere              tcp dpt:https /* Block inbound Web */
ACCEPT     tcp  --  anywhere             anywhere              tcp dpt:https /* Permit all Web Access */
```

关于 iptables 规则的最后说明：在您的链中有一个默认规则，称为`默认策略`，这是最后一个条目。默认值为`ACCEPT`，因此如果数据包一直到列表底部，它将被接受。这通常是期望的行为，如果您计划拒绝一些流量然后允许其余流量 - 例如，如果您正在保护“大多数公共”服务，例如大多数 Web 服务器。

然而，如果所需的行为是允许一些流量然后拒绝其余流量，您可能希望将默认策略更改为`DENY`。要更改`INPUT`链的默认策略，请使用`iptables –P INPUT DENY`命令。`ACCEPT`。

您始终可以添加一个最终规则，允许所有或拒绝所有以覆盖默认策略（无论是什么）。

现在我们已经有了一个基本的规则集，就像许多其他事情一样，您需要记住这个规则集不是永久的 - 它只是在内存中运行，因此不会在系统重新启动后保留。您可以使用`iptables-save`命令轻松保存您的规则。如果在配置中出现错误并希望恢复到保存的表而不重新加载，您可以随时使用`iptables-restore`命令。虽然这些命令在 Ubuntu 发行版中默认安装，但您可能需要安装一个软件包将它们添加到其他发行版中。例如，在基于 Debian 的发行版中，检查或安装`iptables-persistent`软件包，或在基于 Red Hat 的发行版中，检查或安装`iptables-services`软件包。

现在我们已经牢牢掌握了基本的允许和拒绝规则，让我们来探索**网络地址转换**（**NAT**）表。

## NAT 表

NAT 用于转换来自（或前往）一个 IP 地址或子网的流量，并使其看起来像另一个 IP 地址。

这可能是在互联网网关或防火墙中最常见的情况，其中“内部”地址位于 RFC1918 范围中的一个或多个，而“外部”接口连接到整个互联网。在这个例子中，内部子网将被转换为可路由的互联网地址。在许多情况下，所有内部地址都将映射到单个“外部”地址，即网关主机的外部 IP。在这个例子中，这是通过将每个“元组”（源 IP、源端口、目标 IP、目标端口和协议）映射到一个新的元组来实现的，其中源 IP 现在是一个可路由的外部 IP，源端口只是下一个空闲的源端口（目标和协议值保持不变）。

防火墙将这种从内部元组到外部元组的映射保留在内存中的“NAT 表”中。当返回流量到达时，它使用这个表将流量映射回真实的内部源 IP 和端口。如果特定的 NAT 表条目是针对 TCP 会话的，TCP 会话拆除过程将删除该条目的映射。如果特定的 NAT 表条目是针对 UDP 流量的，那么在一段时间的不活动后，该条目通常会被删除。

这在实际中是什么样子？让我们以一个内部网络`192.168.10.0/24`的例子来说明，以及一个 NET 配置，其中所有内部主机都使用网关主机的外部接口的这种“过载 NAT”配置：

![图 4.1–Linux 作为周界防火墙](img/B16336_04_001.jpg)

图 4.1–Linux 作为周界防火墙

让我们更具体一些。我们将添加一个主机，`192.168.10.10`，该主机将向`8.8.8.8`发出 DNS 查询：

![图 4.2–周界防火墙示例，显示 NAT 和状态（会话跟踪或映射）](img/B16336_04_002.jpg)

图 4.2–周界防火墙示例，显示 NAT 和状态（会话跟踪或映射）

因此，使用这个例子，我们的配置是什么样子的？就像以下这样简单：

```
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
```

这告诉网关主机使用`eth1`接口的 IP 地址对离开接口的所有流量进行伪装。`POSTROUTING`关键字告诉它使用`POSTROUTING`链，这意味着这个`MASQERADE`操作发生在数据包路由之后。

当我们开始引入加密时，操作是在路由前还是路由后发生将产生更大的影响。例如，如果我们在 NAT 操作之前或之后加密流量，这可能意味着流量在一个实例中被加密，而在另一个实例中则没有。因此，在这种情况下，出站 NAT 将在路由前或后是相同的。最好开始定义顺序，以避免混淆。

这有数百种变体，但在这一点上重要的是你已经了解了 NAT 的工作原理（特别是映射过程）。让我们离开我们的 NAT 示例，看看混淆表是如何工作的。

## 混淆表

混淆表用于手动调整 IP 数据包在我们的 Linux 主机中传输时的值。让我们考虑一个简短的例子–使用我们上一节中的防火墙示例，如果`eth1`接口上的互联网上行使用`1500`字节的数据包。例如，DSL 链接通常有一些封装开销，而卫星链接则使用较小的数据包（这样任何单个数据包错误都会影响较少的流量）。

“没问题，”你说。“在会话启动时有一个完整的 MTU“发现”过程，通信的两个主机会找出两方之间可能的最大数据包。”然而，特别是对于较旧的应用程序或特定的 Windows 服务，这个过程会中断。可能导致这种情况的另一件事是，如果运营商网络由于某种原因阻止了 ICMP。这可能看起来像是一个极端特例，但实际上，它经常出现。特别是对于传统协议，常见的是发现这个 MTU 发现过程中断。在这种情况下，混淆表就是你的朋友！

这个例子告诉操纵表“当您看到一个`SYN`数据包时，调整这个例子中的`1412`”：

```
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --set-mss 1412
```

如果您正在为实际配置进行计算，如何获得这个“较小的数字”？如果 ICMP 被传递，您可以使用以下命令：

```
ping –M do –s 1400 8.8.8.8
```

这告诉`ping`，“不要分段数据包；发送一个目的地为`8.8.8.8`的`1400`字节大小的数据包。”

通常，查找“真实”大小是一个试错过程。请记住，这个大小包括在这个大小中的 28 个字节的数据包头。

或者如果 ICMP 不起作用，您可以使用`nping`（来自我们的 NMAP 部分）。在这里，我们告诉`nping`使用 TCP，端口`53`，`mtu`值为`1400`，仅持续 1 秒：

```
$ sudo nping --tcp -p 53 -df --mtu 1400 -c 1 8.8.8.8
Starting Nping 0.7.80 ( https://nmap.org/nping ) at 2021-04-22 10:04 PDT
Warning: fragmentation (mtu=1400) requested but the payload is too small already (20)
SENT (0.0336s) TCP 192.168.122.113:62878 > 8.8.8.8:53 S ttl=64 id=35812 iplen=40  seq=255636697 win=1480
RCVD (0.0451s) TCP 8.8.8.8:53 > 192.168.122.113:62878 SA ttl=121 id=42931 iplen=44  seq=1480320161 win=65535 <mss 1430>
```

在这两种情况下（`ping`和`nping`），您都在寻找适用的最大数字（在`nping`的情况下，这将是您仍然看到`RCVD`数据包的最大数字），以确定 MSS 的帮助数字。

从这个例子中可以看出，操纵表的使用非常少。通常，您会在数据包中插入或删除特定的位 - 例如，您可以根据流量类型设置数据包中的**服务类型**（**TOS**）或**区分服务字段代码点**（**DSCP**）位，以告诉上游运营商特定流量可能需要的服务质量。

现在我们已经介绍了一些 iptables 中的默认表，让我们讨论一下在构建复杂表时保持操作顺序的重要性。

## iptables 的操作顺序

已经讨论了一些主要的 iptables，为什么操作顺序很重要？我们已经提到了一个例子 - 如果您正在使用 IPSEC 加密流量，通常会有一个“匹配列表”来定义哪些流量正在被加密。通常情况下，您希望在 NAT 表处理流量之前进行匹配。

同样，您可能正在进行基于策略的路由。例如，您可能希望通过源、目的地和协议匹配流量，并且，例如，将备份流量转发到具有较低每个数据包成本的链路上，并将常规流量转发到具有更好速度和延迟特性的链路上。您通常希望在 NAT 之前做出这个决定。

有几个图表可用于确定 iptables 操作发生的顺序。我通常参考由*Phil Hagen*维护的图表，网址为[`stuffphilwrites.com/wp-content/uploads/2014/09/FW-IDS-iptables-Flowchart-v2019-04-30-1.png`](https://stuffphilwrites.com/wp-content/uploads/2014/09/FW-IDS-iptables-Flowchart-v2019-04-30-1.png)：

![图 4.3 - iptables 的操作顺序](img/B16336_04_003.jpg)

图 4.3 - iptables 的操作顺序

正如您所看到的，配置、处理，尤其是调试 iptables 配置可能变得非常复杂。在本章中，我们专注于输入表，特别是限制或允许在主机上运行的服务的访问。随着我们继续讨论 Linux 上运行的各种服务，您应该能够利用这些知识，看到输入规则可以用来保护您环境中的服务。

接下来您可以用 iptables 做什么？像往常一样，再次查看 man 页面 - 大约有 100 页的语法和示例，如果您想深入了解这个功能，iptables man 页面是一个很好的资源。例如，正如我们讨论过的，您可以使用 iptables 和一些静态路由将 Linux 主机作为路由器或基于 NAT 的防火墙。然而，这些不是常规数据中心的正常用例。在 Linux 主机上运行这些功能是很常见的，但在大多数情况下，您会看到这些功能在预打包的 Linux 发行版上执行，比如 VyOS 发行版或路由器的 FRR/Zebra 软件包，或者 pfSense 或 OPNsense 防火墙发行版。

掌握了 iptables 的基础知识，让我们来解决 nftables 防火墙的配置。

# 配置 nftables

正如我们在本章开头讨论的那样，iptables 正在被弃用，并最终在 Linux 中被 nftables 取代。考虑到这一点，使用 nftables 有什么优势？

部署 nftables 规则比在 iptables 中快得多——在底层，iptables 在添加每条规则时都会修改内核。而 nftables 不会这样做。与此相关，nftables 还有一个 API。这使得使用编排或“网络即代码”工具更容易进行配置。这些工具包括 Terraform、Ansible、Puppet、Chef 和 Salt 等应用程序。这使得系统管理员更容易地自动化主机的部署，因此新的虚拟机可以在几分钟内部署到私有或公共云中，而不是几小时。更重要的是，可能涉及多个主机的应用程序可以并行部署。

nftables 在 Linux 内核中的操作效率也要高得多，因此对于任何给定的规则集，您可以指望 nftables 占用更少的 CPU。对于我们的仅有四条规则的规则集来说，这可能看起来并不重要，但是如果您有 40 条、400 条或 4000 条规则，或者在 400 台虚拟机上有 40 条规则，这可能会很快累积起来！

nftables 使用单个命令进行所有操作——`nft`。虽然您可以使用 iptables 语法进行兼容性，但您会发现没有预定义的表或链，更重要的是，您可以在单个规则中进行多个操作。我们还没有讨论太多关于 IPv6 的内容，但是 iptables 本身无法处理 IPv6（您需要安装一个新的软件包：ip6tables）。

基本知识覆盖后，让我们深入研究命令行和使用`nft`命令配置 nftables 防火墙的细节。

## nftables 基本配置

在这一点上，看一下 nftables 的 man 页面可能是明智的。还要查看主要 nftables 命令`nft`的 man 页面。这个手册比 iptables 更长、更复杂；长达 600 多页。

考虑到这一点，让我们部署与 iptables 相同的示例配置。保护主机的直接`INPUT`防火墙是大多数数据中心中最常见的 Linux 防火墙风格。

首先，请确保记录您已经存在的 iptables 和 ip6tables 规则（`iptables –L`和`ip6tables –L`），然后清除两者（使用`-F`选项）。即使您可以同时运行 iptables 和 nftables，也并不意味着这样做是明智的。考虑一下将管理此主机的下一个人；他们将只看到一个防火墙，认为这就是所有已部署的。为了下一个继承您正在处理的主机的人，配置事物总是明智的！

如果您有现有的 iptables 规则集，特别是如果它是一个复杂的规则集，那么`iptables-translate`命令将把几小时的工作转化为几分钟的工作：

```
robv@ubuntu:~$ iptables-translate -A INPUT -i ens33 -p tcp  -s 1.2.3.0/24 --dport 22  -j ACCEPT -m comment --comment "Permit Admin"
nft add rule ip filter INPUT iifname "ens33" ip saddr 1.2.3.0/24 tcp dport 22 counter accept comment \"Permit Admin\"
```

使用这种语法，我们的 iptables 规则变成了一组非常相似的 nftables 规则：

```
sudo nft add table filter
sudo nft add chain filter INPUT
sudo nft add rule ip filter INPUT iifname "ens33" ip saddr 1.2.3.0/24 tcp dport 22 counter accept comment \"Permit Admin\"
sudo nft add rule ip filter INPUT iifname "ens33" tcp dport 22 counter drop comment \"Block Admin\" 
sudo nft add rule ip filter INPUT iifname "ens33" ip saddr 1.2.3.5 tcp dport 443 counter drop comment \"Block inbound Web\" 
sudo nft add rule ip filter INPUT tcp dport 443 counter accept comment \"Permit all Web Access\"
```

请注意，在添加规则之前，我们首先创建了一个表和一个链。现在来列出我们的规则集：

```
sudo nft list ruleset
table ip filter {
        chain INPUT {
                iifname "ens33" ip saddr 1.2.3.0/24 tcp dport 22 counter packets 0 bytes 0 accept comment "Permit Admin"
                iifname "ens33" tcp dport 22 counter packets 0 bytes 0 drop comment "Block Admin"
                iifname "ens33" ip saddr 1.2.3.5 tcp dport 443 counter packets 0 bytes 0 drop comment "Block inbound Web"
                tcp dport 443 counter packets 0 bytes 0 accept comment "Permit all Web Access"
        }
}
```

就像许多 Linux 网络构造一样，nftables 规则在这一点上并不是持久的；它们只会在下一次系统重新加载（或服务重新启动）之前存在。默认的`nftools`规则集在`/etc/nftools.conf`中。您可以通过将它们添加到此文件中使我们的新规则持久。

特别是在服务器配置中，更新`nftools.conf`文件可能会变得非常复杂。通过将`nft`配置分解为逻辑部分并将其拆分为`include`文件，可以大大简化这一过程。

## 使用包含文件

还可以做什么？您可以设置一个“case”结构，将防火墙规则分段以匹配您的网络段：

```
nft add rule ip Firewall Forward ip daddr vmap {\
      192.168.21.1-192.168.21.254 : jump chain-pci21, \
      192.168.22.1-192.168.22.254 : jump chain-servervlan, \
      192.168.23.1-192.168.23.254 : jump chain-desktopvlan23 \
}
```

在这里，定义的三个链都有自己的入站规则或出站规则集。

您可以看到每个规则都是一个`match`子句，然后将匹配的流量跳转到管理子网的规则集。

与其制作一个单一的、庞大的 nftables 文件，不如使用`include`语句以逻辑方式分隔语句。这样你可以维护一个单一的规则文件，用于所有 web 服务器、SSH 服务器或其他服务器或服务类，这样你最终会得到一些标准的`include`文件。这些文件可以根据需要在每个主机的主文件中以逻辑顺序包含：

```
# webserver ruleset
Include "ipv4-ipv6-webserver-rules.nft"
# admin access restricted to admin VLAN only
Include "ssh-admin-vlan-access-only.nft"
```

或者，你可以使规则变得越来越复杂-到了你基于 IP 头字段的规则，比如**区分服务代码点**（**DSCP**），这是数据包中用于确定或强制执行**服务质量**（**QOS**）的六位，特别是对于语音或视频数据包。你可能还决定在路由前或路由后应用防火墙规则（如果你正在进行 IPSEC 加密，这真的很有帮助）。

## 删除我们的防火墙配置

在我们可以继续下一章之前，我们应该删除我们的示例防火墙配置，使用以下两个命令：

```
$ # first remove the iptables INPUT and FORWARD tables
$ sudo iptables -F INPUT
$ sudo iptables -F FORWARD
$ # next this command will flush the entire nft ruleset
$ sudo nft flush ruleset
```

# 总结

虽然许多发行版仍将 iptables 作为默认防火墙，但随着时间的推移，我们可以预期这种情况会转向更新的 nftables 架构。在这个过渡完成之前还需要一些年头，即使在那时，也会出现一些“意外”，比如你发现了你清单中没有的主机，或者你没有意识到的基于 Linux 的设备-物联网设备，比如恒温器、时钟或电梯控制器。本章让我们对这两种架构有了初步了解。

在 nftables 手册页面中大约有 150 页，iptables 手册页面中有 20 页，这些文档本质上就是一本独立的书。我们已经初步了解了这个工具，但在现代数据中心，为每个主机定义入口过滤器是你最常见的 nftables 用法。然而，当你探索数据中心的安全要求时，出站和过境规则可能确实在你的策略中占据一席之地。我希望这次讨论对你的旅程是一个良好的开始！

如果你发现我们在本章讨论的任何概念都有些模糊，现在是一个很好的时间来复习它们。在下一章中，我们将讨论 Linux 服务器和服务的整体加固方法-当然，Linux 防火墙是这次讨论的关键部分！

# 问题

最后，这里是一些问题列表，供你测试对本章材料的了解。你可以在*附录*的*评估*部分找到答案：

1.  如果你要开始一个新的防火墙策略，你会选择哪种方法？

1.  你会如何实施防火墙的中央标准？

# 进一步阅读

+   iptables 的手册页面：[`linux.die.net/man/8/iptables`](https://https://linux.die.net/man/8/iptables%0D)

+   iptables 处理流程图（Phil Hagen）：

[`stuffphilwrites.com/2014/09/iptables-processing-flowchart/`](https://stuffphilwrites.com/2014/09/iptables-processing-flowchart/)

[`stuffphilwrites.com/wp-content/uploads/2014/09/FW-IDS-iptables-Flowchart-v2019-04-30-1.png`](https://stuffphilwrites.com/wp-content/uploads/2014/09/FW-IDS-iptables-Flowchart-v2019-04-30-1.png)

+   NFT 的手册页面：[`www.netfilter.org/projects/nftables/manpage.html`](https://https://www.netfilter.org/projects/nftables/manpage.html%0D)

+   nftables 维基：[`wiki.nftables.org/wiki-nftables/index.php/Main_Page`](https://https://wiki.nftables.org/wiki-nftables/index.php/Main_Page%0D)

+   *10 分钟内的 nftables*：[`wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes`](https://https://wiki.nftables.org/wiki-nftables/index.php/Quick_reference-nftables_in_10_minutes)
