# 第三章：使用 Linux 和 Linux 工具进行网络诊断

在本章中，我们将介绍一些“工作原理”网络基础知识，以及如何在我们的 Linux 工作站中进行网络故障排除。完成本章后，您应该具备故障排除本地和远程网络服务以及“清点”网络及其服务的工具。

特别是，我们将涵盖以下主题：

+   网络基础知识-OSI 模型。

+   第 2 层-使用 ARP 关联 IP 和 MAC 地址，并对 MAC 地址进行更详细的介绍。

+   第 4 层- TCP 和 UDP 端口的工作原理，包括 TCP 的“三次握手”，以及这在 Linux 命令中的表现。

+   本地 TCP 和 UDP 端口枚举，以及这些与运行服务的关系。

+   使用本机工具进行远程端口枚举。

+   使用已安装的扫描程序（尤其是 netcat 和 nmap）进行远程端口枚举。

+   最后，我们将介绍一些无线操作和故障排除的基础知识。

# 技术要求

为了跟随本节中的示例，我们将使用我们现有的 Ubuntu 主机或**虚拟机**（**VM**）。在本章中，我们将涉及一些无线主题，因此如果您的主机或 VM 中没有无线网卡，您将需要一个 Wi-Fi 适配器来完成这些示例。

在我们通过各种故障排除方法时，我们将使用各种工具，首先是一些本地 Linux 命令：

![](img/B16336_Table_01.jpg)

我们还将使用一些已安装的应用程序：

![](img/B16336_Table_02.jpg)

对于 Ubuntu 中未包含的软件包，请确保您有可用的互联网连接，以便您可以使用`apt`命令进行安装。

# 网络基础知识-OSI 模型

以层的术语讨论网络和应用概念很方便，每一层大致负责更高级别和更抽象功能的功能，并且随着您向*下移动*，在更低层有更多的*基本*原语。以下图表以广义的方式描述了 OSI 模型：

![图 3.1-网络通信的 OSI 模型，带有一些描述和示例](img/B16336_03_001.jpg)

图 3.1-网络通信的 OSI 模型，带有一些描述和示例

在常规用法中，层通常按数字引用，从底部开始计数。因此，第 2 层问题通常涉及 MAC 地址和交换机，并且将局限于站点所在的 VLAN（通常意味着本地子网）。第 3 层问题将涉及 IP 地址分配，路由或数据包（因此将涉及路由器和更远网络的相邻子网）。

与任何模型一样，总会有混淆的余地。例如，在第 6 层和第 7 层之间存在一些长期以来的*模糊性*。在第 5 层和第 6 层之间，虽然 IPSEC 绝对是加密，因此属于第 6 层，但它也可以被视为一种隧道协议（取决于您的观点和实现）。即使在第 4 层，TCP 也有会话的概念，因此似乎可能在第 5 层一侧有所涉及-尽管*端口*的概念将其牢牢地放在第 4 层。

当然，总会有幽默的余地-常见的智慧/笑话是在这个模型中，*人*形成第 8 层。因此，第 8 层问题可能涉及求助台电话，预算讨论，或与您组织的管理层开会解决！

在下一个图表中，我们看到的是要牢记的这个模型中最重要的概念。随着数据的接收，它沿着堆栈向上移动，从最原始的构造到更抽象/高级的构造（例如从位到帧到数据包到 API 到应用程序）。发送数据将其从应用层移动到线上的二进制表示（从上层到下层）。

第 1-3 层通常被称为**媒体**或**网络**层，而第 4-7 层通常被称为**主机或应用程序**层：

![图 3.2-上下移动 OSI 堆栈，封装和解封装](img/B16336_03_002.jpg)

图 3.2-上下移动 OSI 堆栈，封装和解封装

这个概念使得供应商能够制造一个开关，它可以与另一个供应商的网络卡进行交互，或者让交换机与路由器一起工作。这也是我们应用生态系统的动力-在大多数情况下，应用程序开发人员不必担心 IP 地址、路由或无线和有线网络之间的差异，所有这些都已经被处理好了-网络可以被视为一个黑盒子，你在一端发送数据，你可以确信它会在另一端以正确的位置和格式出现。

现在我们已经建立了 OSI 模型的基础，让我们通过探索`arp`命令和本地 ARP 表来详细了解数据链路层。

# 第 2 层-使用 ARP 关联 IP 和 MAC 地址

有了 OSI 模型的坚实基础，我们可以看到到目前为止我们围绕 IP 地址的讨论都集中在第 3 层。这是普通人，甚至许多 IT 和网络人员在他们的理解中通常认为网络路径*停止*的地方-他们可以一直跟踪到那里，然后认为剩下的部分是一个黑盒子。但作为一个网络专业人员，第 1 层和第 2 层非常重要-让我们从第 2 层开始。

理论上，MAC 地址是*烧入*到每个网络接口的地址。虽然这通常是正确的，但这也是一个容易改变的事情。MAC 地址是什么呢？它是一个 12 位（6 字节/48 位）地址，通常以十六进制显示。在显示时，每个字节或双字节通常用`.`或`-`分隔。因此，典型的 MAC 地址可能是`00-0c-29-3b-73-cb`或`9a93.5d84.5a69`（显示两种常见表示）。

实际上，这些地址用于在同一 VLAN 或子网中的主机之间进行通信。如果你查看数据包捕获（我们将在本书的后面部分进行，*第十一章*，*Linux 中的数据包捕获和分析*），在 TCP 会话开始时，你会看到发送站发送一个广播（发送到子网中的所有站点的请求）`谁有 IP 地址 x.x.x.x`。`那是我，我的 MAC 地址是 aaaa.bbbb.cccc`。如果目标 IP 地址在不同的子网上，发送方将为该子网的网关进行“ARP 请求”（通常是默认网关，除非定义了本地路由）。

前进时，发送方和接收方使用 MAC 地址进行通信。这两个主机连接的交换机基础设施仅在每个 VLAN 中使用 MAC 地址，这也是交换机比路由器快得多的原因之一。当我们查看实际的数据包（在*数据包捕获*章节中），你会看到每个数据包中的发送和接收 MAC 地址以及 IP 地址。

ARP 请求在每个主机上都被缓存到一个`arp`命令中：

```
$ arp -a
? (192.168.122.138) at f0:ef:86:0f:5d:70 [ether] on ens33
? (192.168.122.174) at 00:c3:f4:88:8b:43 [ether] on ens33
? (192.168.122.5) at 00:5f:86:d7:e6:36 [ether] on ens33
? (192.168.122.132) at 64:f6:9d:e5:ef:60 [ether] on ens33
? (192.168.122.7) at c4:44:a0:2f:d4:c3 [ether] on ens33
_gateway (192.168.122.1) at 00:0c:29:3b:73:cb [ether] on ens33
```

你可以看到这很简单。它只是将第 3 层 IP 地址与第 2 层 MAC 地址关联到第 1 层`/proc`目录：

```
$ cat /proc/sys/net/ipv4/neigh/default/gc_stale_time
60
$ cat /proc/sys/net/ipv4/neigh/ens33/gc_stale_time
60
```

请注意，每个网络适配器都有一个默认值（以秒为单位），这些值通常是匹配的。这对您来说可能看起来很短-交换机上的匹配 MAC 地址表（通常称为 CAM 表）通常为 5 分钟，路由器上的 ARP 表通常为 14,400 秒（4 小时）。这些值都与资源有关。总的来说，工作站有资源频繁发送 ARP 数据包。交换机从流量中（包括 ARP 请求和响应）学习 MAC 地址，因此使得该计时器略长于工作站计时器是有意义的。同样，路由器上的长时间 ARP 缓存计时器可以节省其 CPU 和 NIC 资源。路由器上的计时器之所以如此之长，是因为在过去的几年里，与网络上的其他所有设备相比，路由器受到带宽和 CPU 的限制。尽管在现代时代已经发生了变化，但路由器上 ARP 缓存超时的长默认值仍然存在。在路由器或防火墙迁移期间很容易忘记这一点-我参与了许多此类维护窗口，在迁移后，对正确路由器的`clear arp`命令神奇地“修复了一切”。

我们还没有讨论 Linux 中的`/proc`目录-这是一个包含 Linux 主机上各种设置和状态的当前文件的“虚拟”目录。这些不是“真实”的文件，但它们被表示为文件，因此我们可以使用与文件相同的命令：`cat`、`grep`、`cut`、`sort`、`awk`等。例如，您可以查看网络接口错误和值，例如在`/proc/net/dev`中（请注意此列表中的事物并不完全正确对齐）。

```
$ cat /proc/net/dev
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
    lo:  208116    2234    0    0    0     0          0          0   208116    2234    0    0    0     0       0          0
 ens33: 255945718  383290    0  662    0     0          0         0 12013178  118882    0    0    0     0       0          0
```

您甚至可以查看内存统计信息（请注意`meminfo`包含**更多**信息）：

```
$ cat /proc/meminfo | grep Mem
MemTotal:        8026592 kB
MemFree:         3973124 kB
MemAvailable:    6171664 kB
```

回到 ARP 和 MAC 地址。您可以添加静态 MAC 地址-一个不会过期且可能与您要连接的主机的真实 MAC 不同的地址。这通常是为了故障排除目的。或者您可以清除 ARP 条目，如果路由器已被替换（例如，如果您的默认网关路由器具有相同的 IP 但现在具有不同的 MAC），则您可能经常需要执行此操作。请注意，您无需特殊权限即可查看 ARP 表，但您确实需要修改它！

要添加静态条目，请执行以下操作（请注意在显示时的`PERM`状态）。

```
$ sudo arp -s 192.168.122.200 00:11:22:22:33:33
$ arp -a | grep 192.168.122.200
? (192.168.122.200) at 00:11:22:22:33:33 [ether] PERM on ens33
```

要删除 ARP 条目，请执行以下操作（请注意，对于此命令，通常会跳过`-i interfacename`参数）。

```
$ sudo arp –i ens33 -d 192.168.122.200
```

要伪装成给定的 IP 地址-例如，回答 IP`10.0.0.1`的 ARP 请求-请执行以下操作：

```
$ sudo arp -i eth0 -Ds 10.0.0.2 eth1 pub
```

最后，您还可以轻松更改接口的 MAC 地址。您可能会认为这是为了处理重复的地址，但这种情况非常罕见。

更改 MAC 地址的合法原因可能包括以下内容：

+   您已迁移防火墙，而 ISP 已经将您的 MAC 硬编码。

+   您已迁移主机或主机 NIC，并且上游路由器对您不可访问，但您不能等待该路由器上的 ARP 缓存过期 4 小时。

+   您已迁移主机，并且旧 MAC 地址有 DHCP 保留，但您无法访问“修复”该 DHCP 条目。

+   出于隐私原因，苹果设备会更改其无线 MAC 地址。鉴于追踪个人身份的许多其他（更容易的）方法，这种保护通常并不那么有效。

更改 MAC 地址的恶意原因包括以下内容：

+   您正在攻击无线网络，并且已经发现一旦经过身份验证，接入点所做的唯一检查就是对客户端 MAC 地址。

+   与前一点相同，但针对使用`802.1x`认证的以太网网络，但配置不安全或不完整（我们将在后面的章节中详细介绍）。

+   您正在攻击具有 MAC 地址权限的无线网络。

希望这说明了出于安全目的使用 MAC 地址通常不是明智的决定。

要查找您的 MAC 地址，我们有四种不同的方法：

```
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 00:0c:29:33:2d:05 brd ff:ff:ff:ff:ff:ff
$ ip link show ens33 | grep link
    link/ether 00:0c:29:33:2d:05 brd ff:ff:ff:ff:ff:ff
$ ifconfig
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1400
        inet 192.168.122.22  netmask 255.255.255.0  broadcast 192.168.122.255
        inet6 fe80::1ed6:5b7f:5106:1509  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:33:2d:05  txqueuelen 1000  (Ethernet)
        RX packets 384968  bytes 256118213 (256.1 MB)
        RX errors 0  dropped 671  overruns 0  frame 0
        TX packets 118956  bytes 12022334 (12.0 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 2241  bytes 208705 (208.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2241  bytes 208705 (208.7 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
$ ifconfig ens33 | grep ether
        ether 00:0c:29:33:2d:05  txqueuelen 1000  (Ethernet)
```

要更改 Linux 主机的 MAC 地址，我们有几个选项：

在 Linux GUI 中，您可以通过单击顶部面板上的网络图标，然后选择**设置**来开始。例如，对于具有一个以太网卡的主机，选择“**有线连接**”，然后选择**有线设置**：

![图 3.3 - 从 GUI 更改 MAC 地址，第 1 步](img/B16336_03_003.jpg)

图 3.3 - 从 GUI 更改 MAC 地址，第 1 步

从弹出的界面中，通过单击**+**图标打开**新配置文件**对话框，然后在**克隆地址**字段中简单地添加 MAC：

![图 3.4 - 从 GUI 更改 MAC 地址，第 2 步](img/B16336_03_004.jpg)

图 3.4 - 从 GUI 更改 MAC 地址，第 2 步

或者，您可以通过命令行或使用脚本执行以下操作（当然，使用您自己的接口名称和目标 MAC 地址）：

```
$ sudo ip link set dev ens33 down
$ sudo ip link set dev ens33 address 00:88:77:66:55:44
$ sudo ip link set dev ens33 device here> up
```

还有`macchanger`软件包，您可以使用它将接口的 MAC 地址更改为目标值或伪随机值。

要进行永久的 MAC 地址更改，您可以使用`netplan`及其相关的配置文件。首先，备份配置文件`/etc/netplan./01-network-manager-all.yaml`，然后进行编辑。请注意，要更改 MAC 地址，您需要一个硬件**烧入地址**（**BIA**）MAC 地址值的`match`语句，然后在设置新 MAC 的下一行：

```
network:
    version: 2
    ethernets:
        ens33:
            dhcp4: true
            match:
                macaddress: b6:22:eb:7b:92:44
            macaddress: xx:xx:xx:xx:xx:xx
```

您可以使用`sudo netplan try`测试新配置，并使用`sudo netplan apply`应用它。

或者，您可以创建或编辑`/etc/udev/rules.d/75-mac-spoof.rules`文件，该文件将在每次启动时执行。添加以下内容：

```
ACTION=="add", SUBSYSTEM=="net", ATTR{address}=="XX:XX:XX:XX:XX:XX", RUN+="/usr/bin/ip link set dev ens33 address YY:YY:YY:YY:YY:YY"
```

掌握了 ARP 中 MAC 地址使用的基础知识后，让我们深入了解 MAC 地址及其与各种网络适配器制造商的关系。

## MAC 地址 OUI 值

所以现在我们已经讨论了超时和 ARP，我们是否已经了解了关于第 2 层和 MAC 地址的所有内容？还不完全 - 让我们谈谈**组织唯一标识符**（**OUI**）值。如果您还记得我们关于如何使用子网掩码将 IP 地址分成网络和主机部分的讨论，您会惊讶地知道 MAC 地址中也有类似的分界线！

每个 MAC 地址的前导位应该标识制造商 - 这个值称为 OUI。OUI 已在 IEEE 维护的正式注册表中注册，并发布在[`standards-oui.ieee.org/oui.txt`](http://standards-oui.ieee.org/oui.txt)上。

但是，Wireshark 项目维护了一个更完整的列表，位于[`gitlab.com/wireshark/wireshark/-/raw/master/manuf`](https://gitlab.com/wireshark/wireshark/-/raw/master/manuf)。 

Wireshark 还提供了一个查找 Web 应用程序，用于此列表，网址为[`www.wireshark.org/tools/oui-lookup.html`](https://www.wireshark.org/tools/oui-lookup.html)。

通常，MAC 地址被平均分割，前 3 个字节（6 个字符）分配给 OUI，后 3 个字节分配给唯一标识设备。但是，组织可以购买更长的 OUI（费用更低），这使它们可以分配更少的设备地址。

OUI 在网络故障排除中是有价值的工具 - 当问题出现或未知站点出现在网络上时，OUI 值可以帮助识别这些罪犯。我们将在本章后面讨论网络扫描仪（特别是 Nmap）时，看到 OUI 稍后会出现。

如果您需要 Linux 或 Windows 的命令行 OUI 解析器，我在[`github.com/robvandenbrink/ouilookup`](https://github.com/robvandenbrink/ouilookup)上发布了一个。

这结束了我们在 OSI 模型的第 2 层中的第一次冒险，以及我们对其与第 3 层的关系的考察，所以让我们进入更高的层次，进入第 4 层，看看 TCP 和 UDP 协议及其相关服务。

# 第 4 层 - TCP 和 UDP 端口的工作原理

传输控制协议（TCP）和用户数据报协议（UDP）通常是我们讨论第 4 层通信时所指的内容，特别是它们如何使用*端口*的概念。

当一个站点想要使用其 IP 地址与同一子网中的另一个站点进行*通信*（IP 通常在应用程序或表示层中确定），它将检查其 ARP 缓存，以查看是否有与该 IP 匹配的 MAC 地址。如果没有该 IP 地址的条目，它将向本地广播地址发送 ARP 请求（正如我们在上一节中讨论的那样）。

接下来的步骤是协议（TCP 或 UDP）建立端口到端口的通信。站点选择一个可用的端口，高于`1024`且低于`65535`（最大端口值），称为**临时端口**。然后使用该端口连接到服务器上的固定服务器端口。这些端口的组合，加上每端的 IP 地址和使用的协议（TCP 或 UDP），将始终是唯一的（因为源端口的选择方式），称为**元组**。这个元组概念是可扩展的，特别是在 NetFlow 配置中，其他值可以被“附加”，例如**服务质量**（QOS）、**区分服务代码点**（DSCP）或**服务类型**（TOS）值、应用程序名称、接口名称和路由信息，如**自治系统号**（ASNs）、MPLS 或 VLAN 信息以及发送和接收的流量字节。由于这种灵活性，所有其他值都是基于的基本 5 值元组通常被称为**5 元组**。

前 1024 个端口（编号为`0-1023`）几乎从不用作源端口 - 这些专门被指定为服务器端口，并且需要 root 权限才能使用。`1024`-`49151`范围内的端口被指定为“用户端口”，`49152`-`65535`被称为动态或私有端口。但是，服务器并不被强制使用低于`1024`的端口号（例如，几乎每个数据库服务器都使用高于`1024`的端口号），这只是一个历史惯例，可以追溯到 TCP 和 UDP 正在开发时，所有服务器端口都在`1024`以下。如果你看看那些追溯到那个时期的服务器，你会看到以下模式，例如：

![](img/B16336_Table_03.jpg)

IANA 维护了正式分配的端口的完整列表，并发布在[`www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml`](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)上。

关于这一点的文档在 RFC6335 中。

实际上，“分配”对于这个列表来说是一个强有力的词。虽然将 Web 服务器放在 TCP 端口`53`上或将 DNS 服务器放在 UDP 端口`80`上是愚蠢的，但许多应用程序根本不在这个列表上，因此只需选择一个通常空闲的端口并使用即可。看到供应商选择实际上已分配给其他人的端口，但分配给了一个更隐蔽或不太常用的服务，这并不罕见。因此，在很大程度上，这个列表是一系列强烈的建议，暗示着我们将考虑任何选择一个知名端口用于自己用途的供应商是……可以说是“愚蠢”的。

## 第 4 层 - TCP 和三次握手

UDP 只需从解析 5 元组开始发送数据。接收应用程序负责接收数据，或者检查应用程序的数据包以验证数据是否按顺序到达并进行任何错误检查。事实上，正是因为 UDP 没有这种额外开销，所以它经常用于诸如 VoIP（IP 电话）和视频流等对时间要求严格的应用程序。在这些类型的应用程序中，如果丢失了一个数据包，通常回溯重试会中断数据流并被最终用户注意到，因此错误在某种程度上被简单地忽略。

然而，TCP 协商了一个序列号，并在对话进行中保持一个序列计数。这使得基于 TCP 的应用程序能够跟踪丢失或损坏的数据包，并在应用程序发送和接收更多数据的同时重试这些数据包。这个初始的协商通常被称为“三次握手”，在图形上看起来像这样：

![图 3.5 – TCP 三次握手，建立 TCP 会话](img/B16336_03_005.jpg)

图 3.5 – TCP 三次握手，建立 TCP 会话

工作原理如下：

1.  第一个数据包来自客户端的临时端口，发送到服务器的（通常是）固定端口。它设置了 SYN（同步）位，并具有随机分配的 SEQ（初始序列）号，本例中为 5432。

1.  服务器的回复数据包设置了 ACK（确认）位，编号为 5433，并且还设置了 SYN 位，具有自己的随机 SYN 值，本例中为 6543。此数据包可能已经包含了握手信息以外的数据（所有后续数据包可能都包含数据）。

1.  第三个数据包是对服务器的第一个 SYN 的 ACK，编号为 6544。

1.  接下来，所有的数据包都是发送给对方的 ACK 数据包，以便每个数据包都有一个唯一的序列号和一个方向。

技术上，数据包 2 可以是两个单独的数据包，但通常它们被合并成一个数据包。

对话的优雅结束方式与此完全相同。结束对话的一方发送一个 FIN，另一方回复一个 FIN-ACK，得到第一方的 ACK，然后结束。

对话的不优雅结束通常是由一个 RST（重置）数据包发起的——一旦发送了 RST，一切就结束了，另一方不应该对此发送回复。

我们将在本章后面以及整本书中都会使用这些主题。所以如果你对此还有疑惑，再读一遍，特别是前面的图表，直到你觉得没问题为止。

现在我们对 TCP 和 UDP 端口如何连接以及为什么应用程序可能使用其中之一有了一些了解，让我们来看看您主机上的应用程序如何在各种端口上“监听”。

# 本地端口枚举——我连接到什么？我在监听什么？

在网络中，许多基本的故障排除步骤都在通信链的一端或另一端——即在客户端或服务器主机上。例如，如果无法访问网页服务器，当然有必要查看网页服务器进程是否正在运行，并且是否在适当的端口上“监听”客户端请求。

`netstat`命令是评估本地主机上网络对话和服务状态的传统方法。要列出所有监听端口和连接，请使用以下选项：

![](img/B16336_Table_04.jpg)

所有五个参数如下所示：

```
$ netstat –tuan
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN
tcp        0      0 192.168.122.22:34586    13.33.160.88:443        TIME_WAIT
tcp        0      0 192.168.122.22:60862    13.33.160.97:443        TIME_WAIT
tcp        0      0 192.168.122.22:48468    35.162.157.58:443       ESTABLISHED
tcp        0      0 192.168.122.22:60854    13.33.160.97:443        TIME_WAIT
tcp        0      0 192.168.122.22:50826    72.21.91.29:80          ESTABLISHED
tcp        0      0 192.168.122.22:22       192.168.122.201:3310    ESTABLISHED
tcp        0      0 192.168.122.22:60860    13.33.160.97:443        TIME_WAIT
tcp        0      0 192.168.122.22:34594    13.33.160.88:443        TIME_WAIT
tcp        0      0 192.168.122.22:42502    44.227.121.122:443      ESTABLISHED
tcp        0      0 192.168.122.22:34596    13.33.160.88:443        TIME_WAIT
tcp        0      0 192.168.122.22:34588    13.33.160.88:443        TIME_WAIT
tcp        0      0 192.168.122.22:46292    35.244.181.201:443      ESTABLISHED
tcp        0      0 192.168.122.22:47902    192.168.122.1:22        ESTABLISHED
tcp        0      0 192.168.122.22:34592    13.33.160.88:443        TIME_WAIT
tcp        0      0 192.168.122.22:34590    13.33.160.88:443        TIME_WAIT
tcp        0      0 192.168.122.22:60858    13.33.160.97:443        TIME_WAIT
tcp        0      0 192.168.122.22:60852    13.33.160.97:443        TIME_WAIT
tcp        0      0 192.168.122.22:60856    13.33.160.97:443        TIME_WAIT
tcp6       0      0 :::22                   :::*                    LISTEN
tcp6       0      0 ::1:631                 :::*                    LISTEN
udp        0      0 127.0.0.53:53           0.0.0.0:*
udp        0      0 0.0.0.0:49345           0.0.0.0:*
udp        0      0 0.0.0.0:631             0.0.0.0:*
udp        0      0 0.0.0.0:5353            0.0.0.0:*
udp6       0      0 :::5353                 :::*
udp6       0      0 :::34878                :::*
$
```

注意不同的状态（您可以在`netstat`的`man`页面中查看所有这些状态，使用`man netstat`命令）。您将看到的最常见的状态列在下表中。如果对任何一种状态的描述感到困惑，您可以跳到接下来的几页，使用图表（图 3.6 和 3.7）来解决这个问题：

![](img/B16336_Table_05.jpg)

以下表格显示了较少见的状态（主要是因为这些状态通常只持续很短的时间）。如果您一直看到这些状态中的任何一个，您可能需要解决问题：

![](img/B16336_Table_06.jpg)

这些状态与我们刚讨论的握手有什么关系？让我们把它们放在一个图表中 - 再次注意，大多数情况下，中间步骤应该只存在很短的时间。如果您看到`SYN_SENT`或`SYN_RECVD`状态持续时间超过几毫秒，您可能需要进行一些故障排除：

![图 3.6 - TCP 会话在建立过程中的各个状态](img/B16336_03_006.jpg)

图 3.6 - TCP 会话在建立过程中的各个状态

当拆除 TCP 会话时，您会看到类似的状态。再次注意，许多中间状态应该只持续很短的时间。编写不良的应用程序通常无法正确拆除会话，因此在这种情况下，您可能会看到`CLOSE WAIT`等状态。另一种拆除会话不良的情况是当路径中的防火墙定义了最大 TCP 会话长度。通常情况下，此设置用于处理无法正确关闭或甚至根本不关闭的编写不良的应用程序。然而，最大会话计时器也可能干扰长时间运行的会话，例如旧式备份作业。如果您遇到这种情况，长时间运行的会话无法恢复良好（例如备份作业出错而不是恢复会话），则可能需要与防火墙管理员合作，以增加此计时器，或者与备份管理员合作，考虑使用更现代的备份软件（例如具有多个并行 TCP 会话和更好的错误恢复）：

![图 3.7 - TCP 会话在各个点被"拆除"时的状态](img/B16336_03_007.jpg)

图 3.7 - TCP 会话在各个点被"拆除"时的状态

请注意，在会话初始化时，我们没有两个状态来区分从服务器返回的`SYN`和`ACK` - 关闭会话涉及的状态比建立会话时涉及的状态要多得多。还要注意的是，数据包在几分之一秒内就会传输，所以如果您在`netstat`显示中看到任何 TCP 会话不是`ESTABLISHED`、`LISTENING`、`TIME-WAIT`或（较少见的）`CLOSED`，那么就有些异常。

为了将监听端口与其背后的服务联系起来，我们将使用`l`（用于监听）而不是`a`，并添加`p`选项以获取程序：

```
$ sudo netstat -tulpn
[sudo] password for robv:
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      666/systemd-resolve
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      811/sshd: /usr/sbin
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      4147/cupsd
tcp6       0      0 :::22                   :::*                    LISTEN      811/sshd: /usr/sbin
tcp6       0      0 ::1:631                 :::*                    LISTEN      4147/cupsd
udp        0      0 127.0.0.53:53           0.0.0.0:*                           666/systemd-resolve
udp        0      0 0.0.0.0:49345           0.0.0.0:*                           715/avahi-daemon: r
udp        0      0 0.0.0.0:631             0.0.0.0:*                           4149/cups-browsed
udp        0      0 0.0.0.0:5353            0.0.0.0:*                           715/avahi-daemon: r
udp6       0      0 :::5353                 :::*                                715/avahi-daemon: r
udp6       0      0 :::34878                :::*                                715/avahi-daemon: r
```

有没有`netstat`的替代方案？当然，有很多。

例如，`ss`几乎具有相同的功能。在下表中，您可以看到我们要求的内容：

![](img/B16336_Table_07.jpg)

让我们通过添加`p`选项来添加进程信息：

```
$ sudo ss -tuap
Netid      State          Recv-Q      Send-Q              Local Address:Port                  Peer Address:Port       Process
udp        UNCONN         0           0                   127.0.0.53%lo:domain                     0.0.0.0:*           users:(("systemd-resolve",pid=666,fd=12))
udp        UNCONN         0           0                         0.0.0.0:49345                      0.0.0.0:*           users:(("avahi-daemon",pid=715,fd=14))
udp        UNCONN         0           0                         0.0.0.0:631                        0.0.0.0:*           users:(("cups-browsed",pid=4149,fd=7))
udp        UNCONN         0           0                         0.0.0.0:mdns                       0.0.0.0:*           users:(("avahi-daemon",pid=715,fd=12))
udp        UNCONN         0           0                            [::]:mdns                          [::]:*           users:(("avahi-daemon",pid=715,fd=13))
udp        UNCONN         0           0                            [::]:34878                         [::]:*           users:(("avahi-daemon",pid=715,fd=15))
tcp        LISTEN         0           4096                127.0.0.53%lo:domain                     0.0.0.0:*           users:(("systemd-resolve",pid=666,fd=13))
tcp        LISTEN         0           128                       0.0.0.0:ssh                        0.0.0.0:*           users:(("sshd",pid=811,fd=3))
tcp        LISTEN         0           5                       127.0.0.1:ipp                        0.0.0.0:*           users:(("cupsd",pid=4147,fd=7))
tcp        ESTAB          0           64                 192.168.122.22:ssh                192.168.122.201:3310        users:(("sshd",pid=5575,fd=4),("sshd",pid=5483,fd=4))
tcp        ESTAB          0           0                  192.168.122.22:42502               44.227.121.122:https       users:(("firefox",pid=4627,fd=162))
tcp        TIME-WAIT      0           0                  192.168.122.22:46292               35.244.181.201:https      
tcp        ESTAB          0           0                  192.168.122.22:47902                192.168.122.1:ssh         users:(("ssh",pid=5832,fd=3))
tcp        LISTEN         0           128                          [::]:ssh                           [::]:*           users:(("sshd",pid=811,fd=4))
tcp        LISTEN         0           5                           [::1]:ipp                           [::]:*           users:(("cupsd",pid=4147,fd=6))
```

注意最后一列是如何换行到下一行的？让我们使用`cut`命令仅选择文本显示中的一些字段。让我们要求列 1、2、4、5 和 6（我们将删除`Recv-Q`和`Send-Q`字段）。我们将使用*管道*的概念将一个命令的输出传递给下一个命令。

`cut`命令只有几个选项，通常您会使用`d`（分隔符）或`f`（字段编号）。

在我们的情况下，我们的分隔符是*空格*字符，我们想要字段 1、2、5 和 6。不幸的是，我们的字段之间有多个空格。我们该如何解决？让我们使用`tr`（translate）命令。通常情况下，`tr`会将一个单个字符转换为一个不同的字符，例如`tr 'a' 'b'`将用`b`替换所有出现的`a`。但在我们的情况下，我们将使用`tr`的`s`选项，它将把目标字符的多个出现减少到一个。

我们的最终命令集会是什么样子？看看以下内容：

```
sudo ss -tuap | tr -s ' ' | cut -d ' ' -f 1,2,4,5,6 --output-delimiter=$'\t'
```

第一个命令与上次使用的`ss`命令相同。我们将其发送到`tr`，它将所有重复的空格字符替换为单个空格。`cut`获取此输出并执行以下操作：“使用空格字符分隔符，只给我字段 1、2、5 和 6，使用*制表符*字符在我的结果列之间。”

我们的最终结果呢？让我们看看：

```
sudo ss -tuap | tr -s ' ' | cut -d ' ' -f 1,2,5,6 --output-delimiter=$'\t'
Netid   State   Local   Address:Port
udp     UNCONN  127.0.0.53%lo:domain    0.0.0.0:*
udp     UNCONN  0.0.0.0:49345   0.0.0.0:*
udp     UNCONN  0.0.0.0:631     0.0.0.0:*
udp     UNCONN  0.0.0.0:mdns    0.0.0.0:*
udp     UNCONN  [::]:mdns       [::]:*
udp     UNCONN  [::]:34878      [::]:*
tcp     LISTEN  127.0.0.53%lo:domain    0.0.0.0:*
tcp     LISTEN  0.0.0.0:ssh     0.0.0.0:*
tcp     LISTEN  127.0.0.1:ipp   0.0.0.0:*
tcp     ESTAB   192.168.122.22:ssh      192.168.122.201:3310
tcp     ESTAB   192.168.122.22:42502    44.227.121.122:https
tcp     ESTAB   192.168.122.22:47902    192.168.122.1:ssh
tcp     LISTEN  [::]:ssh        [::]:*
tcp     LISTEN  [::1]:ipp       [::]:*
```

使用制表符作为分隔符给我们更好的机会让结果列对齐。如果这是一个更大的列表，我们可能会将整个输出发送到一个`.tsv`（缩写为**制表符分隔变量**）文件中，这可以直接被大多数电子表格应用程序读取。这将使用一种称为**重定向**的管道变体来完成。

在这个例子中，我们将使用`>`运算符将整个输出发送到一个名为`ports.csv`的文件中，然后使用`cat`（连接）命令来查看文件内容：

```
$ sudo ss -tuap | tr -s ' ' | cut -d ' ' -f 1,2,5,6 --output-delimiter=$'\t' > ports.tsv
$ cat ports.out
Netid   State   Local   Address:Port
udp     UNCONN  127.0.0.53%lo:domain    0.0.0.0:*
udp     UNCONN  0.0.0.0:49345   0.0.0.0:*
udp     UNCONN  0.0.0.0:631     0.0.0.0:*
udp     UNCONN  0.0.0.0:mdns    0.0.0.0:*
udp     UNCONN  [::]:mdns       [::]:*
udp     UNCONN  [::]:34878      [::]:*
tcp     LISTEN  127.0.0.53%lo:domain    0.0.0.0:*
tcp     LISTEN  0.0.0.0:ssh     0.0.0.0:*
tcp     LISTEN  127.0.0.1:ipp   0.0.0.0:*
tcp     ESTAB   192.168.122.22:ssh      192.168.122.201:3310
tcp     ESTAB   192.168.122.22:42502    44.227.121.122:https
tcp     ESTAB   192.168.122.22:47902    192.168.122.1:ssh
tcp     LISTEN  [::]:ssh        [::]:*
tcp     LISTEN  [::1]:ipp       [::]:*
```

最后，有一个特殊的命令叫做`tee`，它可以将输出发送到两个不同的位置。在这种情况下，我们将其发送到`ports.out`文件和特殊的`STDOUT`（标准输出）文件，这基本上意味着“将其输入回我的终端会话中”。为了好玩，让我们使用`grep`命令只选择已建立的会话：

```
$ sudo ss -tuap | tr -s ' ' | cut -d ' ' -f 1,2,5,6 --output-delimiter=$'\t' | grep "EST" | tee ports.out
tcp     ESTAB   192.168.122.22:ssh      192.168.122.201:3310
tcp     ESTAB   192.168.122.22:42502    44.227.121.122:https
tcp     ESTAB   192.168.122.22:47902    192.168.122.1:ssh
```

想要查看一些关于 TCP 对话的更详细的统计信息吗？使用`t`代表 TCP，`o`代表选项：

```
$ sudo ss –to
State    Recv-Q    Send-Q        Local Address:Port            Peer Address:Port     Process
ESTAB    0         64           192.168.122.22:ssh          192.168.122.201:3310      timer:(on,240ms,0)
ESTAB    0         0            192.168.122.22:42502         44.227.121.122:https     timer:(keepalive,6min47sec,0)
ESTAB    0         0            192.168.122.22:47902          192.168.122.1:ssh       timer:(keepalive,104min,0)
```

这种 TCP 选项显示对于排除可能通过防火墙运行的长期 TCP 会话非常有用。由于内存限制，防火墙会定期清除未正确终止的 TCP 会话。由于它们没有终止，在大多数情况下，防火墙将寻找运行时间超过*x*分钟的会话（其中*x*是一个具有默认值并且可以配置的数字）。这种情况经典的错误可能出现在客户端通过防火墙运行备份或传输大文件，可能是备份到云服务或在网络内外传输大文件。如果这些会话超过了超时值，它们当然会在防火墙上被关闭。

在这种情况下，重要的是要看到长时间的 TCP 会话可能持续多久。备份或文件传输可能由几个较短的会话组成，同时并按顺序运行以最大化性能。或者它们可能是一个持续时间与进程一样长的单个传输。这组`ss`选项可以帮助您了解您的进程在幕后的行为，而无需求助于数据包捕获（不要担心，我们将在本书的后面部分介绍数据包捕获）。

让我们再试一次，查看监听端口并将显示与主机上的监听服务相关联：

```
$ sudo netstat -tulpn
[sudo] password for robv:
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      666/systemd-resolve
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      811/sshd: /usr/sbin
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      4147/cupsd
tcp6       0      0 :::22                   :::*                    LISTEN      811/sshd: /usr/sbin
tcp6       0      0 ::1:631                 :::*                    LISTEN      4147/cupsd
udp        0      0 127.0.0.53:53           0.0.0.0:*                           666/systemd-resolve
udp        0      0 0.0.0.0:49345           0.0.0.0:*                           715/avahi-daemon: r
udp        0      0 0.0.0.0:631             0.0.0.0:*                           4149/cups-browsed
udp        0      0 0.0.0.0:5353            0.0.0.0:*                           715/avahi-daemon: r
udp6       0      0 :::5353                 :::*                                715/avahi-daemon: r
udp6       0      0 :::34878                :::*                                715/avahi-daemon: r
```

收集这些信息的另一个经典方法是使用`lsof`（打开文件列表）命令。等一下，我们想要获取网络信息，而不是谁打开了什么文件的列表！这个问题背后缺少的信息是，在 Linux 中，`lsof`用于枚举 TCP 端口`80`和`22`上的连接：

```
$ lsof -i :443
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
firefox 4627 robv  162u  IPv4  93018      0t0  TCP ubuntu:42502->ec2-44-227-121-122.us-west-2.compute.amazonaws.com:https (ESTABLISHED)
$ lsof -i :22
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
ssh     5832 robv    3u  IPv4 103832      0t0  TCP ubuntu:47902->_gateway:ssh (ESTABLISHED)
```

您可以看到相同的信息，以稍微不同的方式表示。这也很方便，因为`lsof`命令明确显示了每个对话的方向，它是从对话中的初始`SYN`数据包中获取的（发送第一个`SYN`数据包的人在任何 TCP 对话中都是客户端）。

为什么我们如此专注于监听端口和进程？一个答案实际上在本章的早些时候已经提到 - 你只能在特定端口上有一个服务监听。这个经典的例子是尝试在 TCP 端口`80`上启动一个新网站，却不知道已经有一个服务在该端口上监听。在这种情况下，第二个服务或进程将简单地无法启动。

现在我们已经探索了本地监听端口以及它们关联的进程，让我们把注意力转向远程监听端口 - 在其他主机上监听的服务。

# 使用本机工具进行远程端口枚举

那么现在我们知道如何处理本地服务和一些流量诊断，我们如何枚举远程主机上的监听端口和服务呢？

简单的方法是使用本机工具-例如`scp`用于 SFTP 服务器，或`ftp`用于 FTP 服务器。但如果是一些我们没有安装客户端的不同服务怎么办。很简单，`telnet`命令可以在这种情况下使用-例如，我们可以 telnet 到打印机的管理端口，运行`http`（`tcp/80`），并对第一页的标题发出`GET`请求。注意列表底部的垃圾字符-这是该页面上图形的表示方式：

```
$ telnet 192.168.122.241 80
Trying 192.168.122.241...
Connected to 192.168.122.241.
Escape character is '^]'.
GET / HTTP/1.1
HTTP/1.1 200 OK
Server: HP HTTP Server; HP PageWide 377dw MFP - J9V80A; Serial Number: CN74TGJ0H7; Built: Thu Oct 15, 2020 01:32:45PM {MAVEDWPP1N001.2042B.00}
Content-Encoding: gzip
Content-Type: text/html
Last-Modified: Thu, 15 Oct 2020 13:32:45 GMT
Cache-Control: max-age=0
Set-Cookie: sid=se2b8d8b3-e51eab77388ba2a8f2612c2106b7764a;path=/;HttpOnly;
Content-Security-Policy: default-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src * 'unsafe-inline'; frame-ancestors 'self'
X-Frame-Options: SAMEORIGIN
X-UA-Compatible: IE=edge
X-XXS-Protection: 1
X-Content-Type-Options: nosniff
Content-Language: en
Content-Length: 667
▒▒▒O▒0▒▒▒W▒
           Hs<M▒´M
▒▒▒q.▒[▒▒l▒▒▒▒▒N▒J+▒"▒}s▒szr}?▒▒▒▒▒▒▒[▒<|▒▒:B{▒3v=▒▒▒ɇs▒n▒▒▒▒i ▒▒"1vƦ?X▒▒9o▒▒I▒
                                                                               2▒▒?ȋ▒ ]▒▒▒)▒^▒uF▒F{ԞN75▒)#▒|
```

即使你不知道要输入什么，通常如果你能用 telnet 连接，那就意味着你尝试的端口是打开的。

然而，这种方法也存在一些问题-如果你不知道要输入什么，这并不是一种确定端口是否打开的万无一失的方法。而且，经常退出会成为一个问题-通常`BYE`，`QUIT`或`EXIT`会起作用，有时按下*^c*（*Ctrl* + *C*）或*^z*会起作用，但这两种方法都不是 100%保证。最后，你可能正在查看多个主机或多个端口，或者这可能只是你故障排除的第一步。所有这些因素综合起来，使得这种方法既笨拙又耗时。

作为对此的回答，我们有专门为此目的而构建的*端口扫描器*工具-`nmap`（我们将在下一节中介绍）是其中最受欢迎的。但是，如果你没有安装其中一个，`nc`（netcat）命令就是你的朋友！

让我们用 netcat 扫描我们的示例 HP 打印机：

```
$ nc -zv 192.168.122.241 80
Connection to 192.168.122.241 80 port [tcp/http] succeeded!
$ nc -zv 192.168.122.241 443
Connection to 192.168.122.241 443 port [tcp/https] succeeded!
```

或者我们测试前`1024`个端口怎么样？假设我们使用以下命令：

```
$ nc -zv 192.168.122.241 1-1024
```

我们得到了许多错误页面，比如以下内容：

```
nc: connect to 192.168.122.241 port 1013 (tcp) failed: Connection refused
```

好的，让我们尝试用我们的朋友`grep`来过滤这些：

```
$ nc -zv 192.168.122.241 1-1024 | grep –v refused
```

这仍然不起作用-为什么呢？关键在于“错误”这个词，Netcat 将错误发送到特殊的`STDERR`（标准错误）文件，这在 Linux 中是正常的（我们稍后将看到为什么对于这个工具来说，成功的连接算作错误）。该文件会回显到控制台，但它不是`STDOUT`，所以我们的`grep`过滤器完全错过了。我们该如何解决这个问题呢？

关于这三个`STD`文件或*流*的一些背景知识-它们各自都有一个与之关联的文件编号：

![](img/B16336_Table_08.jpg)

通过对这些文件编号进行一些游戏，我们可以将`STDERR`重定向到`STDOUT`（这样`grep`现在将为我们工作）：

```
$ nc -zv 192.168.122.241 1-1024 2>&1 | grep -v refused
Connection to 192.168.122.241 80 port [tcp/http] succeeded!
Connection to 192.168.122.241 443 port [tcp/https] succeeded!
Connection to 192.168.122.241 515 port [tcp/printer] succeeded!
Connection to 192.168.122.241 631 port [tcp/ipp] succeeded!
```

这是`0`（在真实网络中可见），但 netcat 在这方面失败了：

```
$ nc -zv 192.168.122.241 0-65535 2>&1 | grep -v refused
nc: port number too small: 0
$ nc -zv 192.168.122.241 1-65535 2>&1 | grep -v refused
Connection to 192.168.122.241 80 port [tcp/http] succeeded!
Connection to 192.168.122.241 443 port [tcp/https] succeeded!
Connection to 192.168.122.241 515 port [tcp/printer] succeeded!
Connection to 192.168.122.241 631 port [tcp/ipp] succeeded!
Connection to 192.168.122.241 3910 port [tcp/*] succeeded!
Connection to 192.168.122.241 3911 port [tcp/*] succeeded!
Connection to 192.168.122.241 8080 port [tcp/http-alt] succeeded!
Connection to 192.168.122.241 9100 port [tcp/*] succeeded!
```

我们也可以为 UDP 复制这个过程：

```
$ nc -u -zv 192.168.122.1 53
Connection to 192.168.122.1 53 port [udp/domain] succeeded!
```

然而，如果我们扫描 UDP 范围，这可能会出现`端口不可达`错误，如果路径中有任何防火墙，这并不总是受支持。让我们看看当针对 UDP 端口时，“前`1024`”扫描需要多长时间（注意我们如何使用分号串联命令）：

```
$ date ; nc -u -zv 192.168.122.241 1-1024 2>&1 | grep succeed ; date
Thu 07 Jan 2021 09:28:17 AM PST
Connection to 192.168.122.241 68 port [udp/bootpc] succeeded!
Connection to 192.168.122.241 137 port [udp/netbios-ns] succeeded!
Connection to 192.168.122.241 138 port [udp/netbios-dgm] succeeded!
Connection to 192.168.122.241 161 port [udp/snmp] succeeded!
Connection to 192.168.122.241 427 port [udp/svrloc] succeeded!
Thu 07 Jan 2021 09:45:32 AM PST
```

是的，整整 18 分钟-这种方法并不是一个速度恶魔！

使用 netcat，你也可以直接与服务交互，就像我们的 telnet 示例一样，但没有 telnet 带来的“终端/光标控制”类型的开销。例如，连接到 web 服务器的语法如下：

```
# nc 192.168.122.241 80
```

但更有趣的是，我们可以建立一个虚假的服务，告诉 netcat 监听指定的端口。如果你需要测试连接，特别是如果你想测试防火墙规则，但还没有构建目标主机或服务，这将非常方便。

这个语法告诉主机监听端口`80`。使用`l`参数告诉 netcat 监听，但当你的远程测试程序或扫描程序连接并断开连接时，netcat 监听器会退出。使用`l`参数是“更努力地监听”的选项，它可以正确处理 TCP 连接和断开连接，使监听器保持在原地。不幸的是，Ubuntu 实现的 netcat 中缺少`l`参数和`-e`（执行）参数。不过我们可以通过其他方法实现这一点-继续阅读！

在此基础上，让我们使用 netcat 搭建一个简单的网站！首先，创建一个简单的文本文件。我们将把我们的`index.html`做成以下的样子：

![](img/B16336_Table_09.jpg)

现在，为了搭建网站，让我们在 netcat 语句中添加一个 1 秒的超时，并将整个内容放入一个循环中，这样当我们退出连接时，netcat 会重新启动：

```
$ while true; do cat index.html | nc -l -p 80 –q 1; done
nc: Permission denied
nc: Permission denied
nc: Permission denied
…. (and so on) ….
```

注意监听端口`80`失败了 - 我们不得不按下*Ctrl* + *C*来退出循环。为什么会这样？（提示：回到本章前面关于 Linux 中端口定义的部分。）让我们再试一次，使用端口`1500`：

```
$ while true; do cat index.html | nc -l -p 1500 -q 1  ; done
```

浏览到我们的新网站（注意它是 HTTP，并注意使用`:1500`来设置目标端口），我们现在看到以下内容：

![图 3.8 - 一个 Netcat 简单网站](img/B16336_03_008.jpg)

图 3.8 - 一个 Netcat 简单网站

回到 Linux 控制台，您会看到 netcat 回显客户端的`GET`请求和浏览器的`User-Agent`字符串。您将看到整个 HTTP 交换（从服务器的角度）：

```
GET / HTTP/1.1
Host: 192.168.122.22:1500
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.88 Safari/537.36 Edg/87.0.664.66
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
```

让我们更加积极一点，让这个网站告诉我们日期和时间：

```
while true; do echo -e "HTTP/1.1 200 OK\n\n $(date)" | nc -l -p 1500 -q 1; done
```

现在浏览该网站，我们可以看到当前的日期和时间：

![图 3.9 - 一个更复杂的 Netcat 网站 - 添加时间和日期](img/B16336_03_009.jpg)

图 3.9 - 一个更复杂的 Netcat 网站 - 添加时间和日期

或者，使用`apt-get`安装`fortune`软件包，我们现在可以添加一句谚语来给我们一些及时的智慧：

![图 3.10 - 向 Netcat 网站添加一句谚语](img/B16336_03_010.jpg)

图 3.10 - 向 Netcat 网站添加一句谚语

我们还可以使用 netcat 传输文件。在接收端，我们将监听端口`1234`，并将输出发送到`out.file`，同样使用重定向：

```
nc -l -p 1234 > received.txt
```

在发送端，我们将连接到该服务 3 秒钟，并发送`sent-file.txt`。我们将通过在相反方向使用重定向来获取我们的输入，使用`<`运算符：

```
nc -w 3 [destination ip address] 1234 < sent-file.txt
```

现在，回到接收端，我们可以使用`cat`命令查看生成的文件：

```
$ cat received.txt
Mr. Watson, come here, I want to see you.
```

这说明 netcat 可以是一个有价值的故障排除工具，但根据您要完成的任务，使用起来可能会比较复杂。我们可以使用 netcat 作为一个简单的代理，作为一个简单的聊天应用程序，或者呈现一个完整的 Linux shell - 所有这些对于网络管理员（或者渗透测试者）来说都是很方便的东西。

这就是 netcat 的基础知识。我们已经使用 netcat 来枚举本地端口，连接和与远程端口交互，搭建了一些相当复杂的本地服务，甚至传输文件。现在让我们来看看 Nmap，这是一种更快速、更优雅的枚举远程端口和服务的方法。

# 远程端口和服务枚举 - nmap

用于扫描网络资源的最广泛使用的工具是 NMAP（简称网络映射器）。NMAP 起初是一个简单的端口扫描工具，但现在已经远远超出了简单的功能范围，具有长长的功能列表。

首先，nmap 在基本的 Ubuntu 工作站上默认情况下是没有安装的（尽管它在许多其他发行版中默认情况下是包含的）。要安装它，请运行`sudo apt-get install nmap`。

在我们继续使用 nmap 进行工作时，请尝试我们在示例中使用的各种命令。您可能会看到类似的结果，并且会在学习的过程中了解这个有价值的工具。在这个过程中，您可能也会对您的网络有很多了解！

重要提示

关于“自己尝试一下”建议的一个非常重要的警告。NMAP 是一个相当无害的工具，它几乎不会引起网络问题。但是，如果您在生产网络上运行它，您将首先要了解该网络的情况。有几类设备的网络堆栈特别“不稳定” - 例如较旧的医疗设备，以及较旧的工业控制系统（ICS）或监控和数据采集（SCADA）设备。

换句话说，如果您在医院、工厂或公用事业部门，要小心！对生产网络运行任何网络映射都可能引起问题。

你可能确实还想这样做，但首先要针对已知的"空闲"设备进行测试，这样当你扫描"真实"网络时，你可以确保不会造成问题。而且，请（**请**），如果你在医疗保健网络上，请**永远**不要扫描任何与人相关的东西！

第二个（合法的）警告-不要未经许可扫描东西。如果你在家里或实验室网络上，那是一个很好的地方来使用 nmap 或更具侵略性的安全评估工具。但是，如果你在工作中，即使你确信不会造成问题，你也需要先书面获得许可。

扫描你不拥有或没有书面许可扫描的互联网主机是非常非法的。许多人认为这是相当无害的，在大多数情况下，扫描只是大多数公司认为的"互联网白噪音"（大多数组织每小时被扫描几十次甚至几百次）。请记住，谚语"罪犯和信息安全专业人员之间的区别在于签署的合同"之所以经常被重复，是因为它是 100%正确的。

有了这一切，让我们更加熟悉这个伟大的工具！尝试运行`man nmap`（还记得`manual`命令吗？）- nmap 的 man 页面中有很多有用的信息，包括完整的文档。不过一旦我们更熟悉这个工具，你可能会发现帮助文本更快。通常你知道（多多少少）你在寻找什么，所以你可以使用`grep`命令搜索，例如：`nmap - -help | grep <my_search_string>`。在 nmap 的情况下，你可以放弃标准的`- - help`选项，因为 nmap 没有参数的默认输出是帮助页面。

因此，要找出如何进行 ping 扫描-也就是说，对范围内的所有内容进行 ping（我总是忘记语法），你可以搜索如下：

```
$ nmap | grep -i ping
  -sn: Ping Scan - disable port scan
  -PO[protocol list]: IP Protocol Ping
```

我们该怎么做？NMAP 想知道你想要映射什么-在这种情况下，我将映射`192.168.122.0/24`子网：

```
$ nmap -sn 192.168.122.0/24
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-05 13:53 PST
Nmap scan report for _gateway (192.168.122.1)
Host is up (0.0021s latency).
Nmap scan report for ubuntu (192.168.122.21)
Host is up (0.00014s latency).
Nmap scan report for 192.168.122.51
Host is up (0.0022s latency).
Nmap scan report for 192.168.122.128
Host is up (0.0027s latency).
Nmap scan report for 192.168.122.241
Host is up (0.0066s latency).
Nmap done: 256 IP addresses (5 hosts up) scanned in 2.49 seconds
```

这是一个快速扫描，告诉我们子网上当前活动的每个 IP。

现在让我们寻找服务。让我们首先寻找任何运行`tcp/443`（你可能会认识为 HTTPS）的东西。我们将使用`nmap –p 443 –open 192.168.122.0/24`命令。在这个命令中有两件事要注意。首先，我们用`-p`选项指定了端口。

默认情况下，NMAP 使用`SYN`扫描来扫描 TCP 端口。nmap 发送一个`SYN`数据包，并等待收到一个`SYN-ACK`数据包。如果它看到了，端口就是打开的。如果它收到一个`端口不可达`的响应，那么端口被认为是关闭的。

如果我们想要进行完整的`connect`扫描（整个三次握手完成），我们可以指定`-sT`。

接下来，我们看到了一个`--open`选项。这表示"只显示给我打开的端口"。如果没有这个选项，我们将看到关闭的端口以及"过滤"的端口（通常意味着初始数据包没有返回任何东西）。

如果我们想要更详细地了解为什么端口可能被认为是打开、关闭或被过滤，我们会移除`--open`选项，然后添加`--reason`。

```
$ nmap -p 443 --open 192.168.122.0/24
 Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-05 13:55 PST
Nmap scan report for _gateway (192.168.122.1)
Host is up (0.0013s latency).
PORT    STATE SERVICE
443/tcp open  https
Nmap scan report for 192.168.122.51
Host is up (0.0016s latency).
PORT    STATE SERVICE
443/tcp open  https
Nmap scan report for 192.168.122.241
Host is up (0.00099s latency).
PORT    STATE SERVICE
443/tcp open  https
Nmap done: 256 IP addresses (5 hosts up) scanned in 2.33 seconds
```

要扫描 UDP 端口，我们会使用相同的语法，但添加`sU`选项。注意到这一点，我们开始看到活动主机的 MAC 地址。这是因为扫描的主机与扫描器在同一个子网中，所以这些信息是可用的。NMAP 使用 MAC 地址的 OUI 部分来识别每个网络卡的供应商：

```
$ nmap -sU -p 53 --open 192.168.122.0/24
You requested a scan type which requires root privileges.
QUITTING!
```

糟糕-因为我们正在扫描 UDP 端口，Nmap 需要以 root 权限运行（使用`sudo`）。这是因为它需要将发送接口置于*混杂模式*，以便捕获可能返回的任何数据包。这是因为 UDP 中没有类似 TCP 中的*会话*的第 5 层概念，因此发送和接收的数据包之间没有第 5 层连接。根据使用的命令行参数（不仅仅是用于 UDP 扫描），Nmap 可能需要提升的权限。在大多数情况下，如果您使用 Nmap 或类似工具，您会发现自己经常使用`sudo`：

```
$ sudo nmap -sU -p 53 --open 192.168.122.0/24
[sudo] password for robv:
Sorry, try again.
[sudo] password for robv:
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-05 14:04 PST
Nmap scan report for _gateway (192.168.122.1)
Host is up (0.00100s latency).
PORT   STATE SERVICE
53/udp open  domain
MAC Address: 00:0C:29:3B:73:CB (VMware)
Nmap scan report for 192.168.122.21
Host is up (0.0011s latency).
PORT   STATE         SERVICE
53/udp open|filtered domain
MAC Address: 00:0C:29:E4:0C:31 (VMware)
Nmap scan report for 192.168.122.51
Host is up (0.00090s latency).
PORT   STATE         SERVICE
53/udp open|filtered domain
MAC Address: 00:25:90:CB:00:18 (Super Micro Computer)
Nmap scan report for 192.168.122.128
Host is up (0.00078s latency).
PORT   STATE         SERVICE
53/udp open|filtered domain
MAC Address: 98:AF:65:74:DF:6F (Unknown)
Nmap done: 256 IP addresses (23 hosts up) scanned in 1.79 seconds
```

关于此扫描还有一些需要注意的事项：

初始扫描尝试失败了-请注意，您需要 root 权限才能在 NMAP 中进行大多数扫描。为了获得它所做的结果，在许多情况下，该工具会自己制作数据包，而不是使用标准的操作系统服务来执行，它通常也需要权限来捕获目标主机返回的数据包-因此 nmap 需要提升的权限来执行这两个操作。

我们看到很多状态指示`open|filtered`端口。UDP 特别容易出现这种情况-因为没有`SYN`/`SYN-ACK`类型的握手，你发送一个`UDP`数据包，可能不会收到任何回应-这并不一定意味着端口关闭，可能意味着你的数据包被远程服务处理了，但没有发送确认（有些协议就是这样）。或者在很多情况下，这可能意味着端口没有开启，主机没有正确返回 ICMP `Port Unreachable`错误消息（ICMP Type 1, Code 3）。

为了获得更多细节，让我们使用`sV`选项，这将探测相关端口并获取有关服务本身的更多信息。在这种情况下，我们将看到`192.168.122.1`被明确识别为开放的，运行`domain`服务，并列出服务版本为`generic dns response: NOTIMP`（这表示服务器不支持*RFC 2136*中描述的 DNS `UPDATE`功能）。服务信息后面的*服务指纹*签名可以帮助进一步识别服务，如果 NMAP 的识别不是最终确定的。

还要注意，对于其他主机，原因被列为`no-response`。如果您了解协议，通常可以在这些情况下做出良好的推断。在扫描 DNS 时，`no-response`表示那里没有 DNS 服务器或端口关闭。（或者可能是打开了一些与 DNS 不同的古怪服务，这是非常不太可能的）。（这是在`192`的一个 DNS 服务器上。）

还要注意，这次扫描花了整整 100 秒，大约是我们最初扫描的 50 倍：

```
$ sudo nmap -sU -p 53 --open -sV --reason 192.168.122.0/24
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-05 14:17 PST
Nmap scan report for _gateway (192.168.122.1)
Host is up, received arp-response (0.0011s latency).
PORT   STATE SERVICE REASON              VERSION
53/udp open  domain  udp-response ttl 64 (generic dns response: NOTIMP)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-UDP:V=7.80%I=7%D=1/5%Time=5FF4E58A%P=x86_64-pc-linux-gnu%r(DNSVe
SF:rsionBindReq,1E,"\0\x06\x81\x85\0\x01\0\0\0\0\0\0\x07version\x04bind\0\
SF:0\x10\0\x03")%r(DNSStatusRequest,C,"\0\0\x90\x04\0\0\0\0\0\0\0\0")%r(NB
SF:TStat,32,"\x80\xf0\x80\x95\0\x01\0\0\0\0\0\0\x20CKAAAAAAAAAAAAAAAAAAAAA
SF:AAAAAAAAA\0\0!\0\x01");
MAC Address: 00:0C:29:3B:73:CB (VMware)
Nmap scan report for 192.168.122.51
Host is up, received arp-response (0.00095s latency).
PORT   STATE         SERVICE REASON      VERSION
53/udp open|filtered domain  no-response
MAC Address: 00:25:90:CB:00:18 (Super Micro Computer)
Nmap scan report for 192.168.122.128
Host is up, received arp-response (0.00072s latency).
PORT   STATE         SERVICE REASON      VERSION
53/udp open|filtered domain  no-response
MAC Address: 98:AF:65:74:DF:6F (Unknown)
Nmap scan report for 192.168.122.171
Host is up, received arp-response (0.0013s latency).
PORT   STATE         SERVICE REASON      VERSION
53/udp open|filtered domain  no-response
MAC Address: E4:E1:30:16:76:C5 (TCT mobile)
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 256 IP addresses (24 hosts up) scanned in 100.78 seconds
```

让我们尝试对`192.168.122.1`的`tcp/443`端口进行`sV`详细服务扫描-我们将看到 NMAP 对运行在该主机上的 Web 服务器进行了相当不错的识别：

```
root@ubuntu:~# nmap -p 443 -sV 192.168.122.1
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-06 09:02 PST
Nmap scan report for _gateway (192.168.122.1)
Host is up (0.0013s latency).
PORT    STATE SERVICE  VERSION
443/tcp open  ssl/http nginx
MAC Address: 00:0C:29:3B:73:CB (VMware)
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.60 seconds
```

尝试对`192.168.122.51`进行相同的操作，我们发现该服务被正确识别为 VMware ESXi 7.0 管理界面：

```
root@ubuntu:~# nmap -p 443 -sV 192.168.122.51
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-06 09:09 PST
Nmap scan report for 192.168.122.51
Host is up (0.0013s latency).
PORT    STATE SERVICE   VERSION
443/tcp open  ssl/https VMware ESXi SOAP API 7.0.0
MAC Address: 00:25:90:CB:00:18 (Super Micro Computer)
Service Info: CPE: cpe:/o:vmware:ESXi:7.0.0
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 140.48 seconds
```

现在我们已经成为使用各种选项扫描端口的专家，让我们来扩展一下。NMAP 允许我们对它发现的任何开放端口运行脚本-这可以节省大量时间！

## NMAP 脚本

到目前为止，我们只是看了端口扫描-NMAP 不仅仅是这样。它提供了一个完整的脚本引擎，可以处理 NMAP 的数据包或输出，基于 Lua（一种基于文本的解释语言）。我们不会在本书中深入研究 LUA，但 NMAP 附带了几个预先编写的脚本，其中一些对网络管理员非常有价值。

例如，考虑 SMB 版本信息。微软多年来一直强烈建议退役 SMBv1，尤其是在 2017 年 WannaCry/Petya/NotPetya 恶意软件系列中使用 EternalBlue 和 EternalRomance 漏洞之前。尽管通过在较新的 Windows 版本中甚至难以启用 SMBv1 来有效地退役了 SMBv1，但我们仍然在企业网络中看到 SMBv1-无论是在较旧的服务器平台上还是在实现 Samba 服务中的较旧的基于 Linux 的设备上。使用`smb-protocols`脚本扫描这一点非常容易。在使用任何脚本之前，打开脚本查看它的确切功能以及 NMAP 如何调用它（可能需要哪些端口或参数）。在这种情况下，`smb-protocols`文本给出了我们的用法，以及输出中可以期望的内容。

```
-- @usage nmap -p445 --script smb-protocols <target>
-- @usage nmap -p139 --script smb-protocols <target>
--
-- @output
-- | smb-protocols:
-- |   dialects:
-- |     NT LM 0.12 (SMBv1) [dangerous, but default]
-- |     2.02
-- |     2.10
-- |     3.00
-- |     3.02
-- |_    3.11
--
-- @xmloutput
-- <table key="dialects">
-- <elem>NT LM 0.12 (SMBv1) [dangerous, but default]</elem>
-- <elem>2.02</elem>
-- <elem>2.10</elem>
-- <elem>3.00</elem>
-- <elem>3.02</elem>
-- <elem>3.11</elem>
-- </table>
```

让我们扫描目标网络中的一些特定主机，以获取更多信息。我们只展示一个运行 SMBv1 协议的示例主机的输出。从主机名来看，它似乎是一个**网络附加存储**（**NAS**）设备，因此很可能在内部是基于 Linux 或 BSD 的。从 OUI 中我们可以看到主机的品牌名称，这给了我们更具体的信息：

```
nmap -p139,445 --open --script=smb-protocols 192.168.123.0/24
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-06 12:27 Eastern Standard Time
Nmap scan report for test-nas.defaultroute.ca (192.168.123.1)
Host is up (0.00s latency).
PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
MAC Address: 00:D0:B8:21:89:F8 (Iomega)
Host script results:
| smb-protocols: 
|   dialects: 
|     NT LM 0.12 (SMBv1) [dangerous, but default]
|     2.02
|     2.10
|     3.00
|     3.02
|_    3.11
```

或者您可以直接使用`smb-vuln-ms17-010.nse`脚本扫描`Eternal*`漏洞（仅显示一个主机作为示例）。扫描同一主机时，我们发现即使启用了 SMBv1，该特定漏洞也没有发挥作用。但仍强烈建议禁用 SMBv1，因为 SMBv1 容易受到一系列漏洞的影响，而不仅仅是`ms17-010`。

在列表中再往下滚动一点，我们的第二个示例主机确实存在该漏洞。从主机名来看，我们可以看到这很可能是一个业务关键的主机（运行 BAAN），因此我们更希望修复这台服务器而不是勒索软件。查看该主机上的生产应用程序，实际上没有理由让大多数用户暴露 SMB-实际上只有系统或应用程序管理员应该映射到该主机，用户应该通过其应用程序端口连接到它。对此的建议显然是修补漏洞（这可能已经有好几年没有做了），但也要将该服务防火墙隔离开大多数用户（或者如果管理员不使用该服务，则禁用该服务）。

```
Starting Nmap 7.80 ( https://nmap.org ) at 2021-01-06 12:32 Eastern Standard Time
Nmap scan report for nas.defaultroute.ca (192.168.123.11)
Host is up (0.00s latency).
PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
MAC Address: 00:D0:B8:21:89:F8 (Iomega)
Nmap scan report for baan02.defaultroute.ca (192.168.123.77)
Host is up (0.00s latency).
PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
MAC Address: 18:A9:05:3B:ED:EC (Hewlett Packard)
Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
```

Nmap 安装了数百个脚本。如果您正在寻找特定内容，特别是如果仅通过端口扫描无法确定，那么使用一个或多个 nmap 脚本通常是最简单的方法。只需记住，如果您正在寻找"流氓"主机，比如 DHCP 服务器，您将找到您的生产主机以及任何不需要的实例。

请注意，许多脚本依赖您在扫描中包含正确的端口号。"广播"风格的脚本通常只会扫描您的扫描器所在的子网，因此扫描远程子网可能意味着"借用"或放置主机在该子网上。本列表中讨论的许多核心网络服务在本书的后续章节中都有涉及，包括 DNS、DHCP 等。

请记住（再次强调），未经授权的扫描从来不符合您的最佳利益-首先获得书面许可！

nmap 附带了数百个脚本，还有数百个可以通过快速的互联网搜索获得。我在生产网络上发现的一些预打包的 nmap 脚本包括以下内容：

![](img/B16336_Table_10.jpg)

**意外的、恶意的或配置错误的网络基础设施**：

![](img/B16336_Table_11.jpg)

**服务器问题和恶意服务**：

![](img/B16336_Table_12.jpg)

**盗版、"影子 IT"、恶意或其他意外服务器**：

![](img/B16336_Table_13a.jpg)![](img/B16336_Table_13b.jpg)

**工作站问题**：

![](img/B16336_Table_14.jpg)

**网络外围问题**：

![](img/B16336_Table_15.jpg)

**其他服务器或工作站问题**：

![](img/B16336_Table_16.jpg)

这总结了 Nmap 的各种用途。Nmap 在较大的网络中表现不佳 - 例如，在`/8`或`/16`网络中，或一些非常大的 IPv6 网络中。对于这些网络，需要更快的工具。让我们探索 MASSCAN 工具的这些用途。

## Nmap 有限制吗？

Nmap 的主要限制是性能。随着网络规模的增长，Nmap（当然）完成任何扫描所需的时间会越来越长。这通常不是问题，但在生产网络中，如果您的扫描从早上 8 点开始，直到第二天结束，那么很可能有相当长的时间设备大多处于关闭状态或断开连接状态，因此扫描的实用性会受到影响。当您在非常大的网络上时，这一点尤为明显 - 例如，当您的子网掩码缩小或网络数量增加时，Nmap 的扫描时间可能会增加到几个小时、几天或几周。同样，在 IPv6 网络中，通常会看到成千上万甚至数百万的地址，这可能会导致 Nmap 扫描时间长达数年甚至数十年。

有两种方法可以帮助解决这个问题。

首先，如果您阅读 NMAP 的`man`页面，有一些参数可以加快速度 - 您可以调整并行性（可以同时运行多少个操作）、主机超时、往返超时和操作之间的延迟等。这些在`man`页面上有详细说明，并且在这里有更深入的讨论：[`nmap.org/book/man-performance.html`](https://nmap.org/book/man-performance.html)。

或者，您可以查看不同的工具。Rob Graham 维护了 MASSCAN 工具，该工具专门用于高性能扫描。有足够的带宽和马力，它可以在不到 10 分钟的时间内扫描整个 IPv4 互联网。该工具的 1.3 版本增加了 IPv6 支持。MASSCAN 的语法类似于 Nmap，但在使用这个更快的工具时有一些需要注意的地方。该工具以及其文档和“陷阱”发布在这里：[`github.com/robertdavidgraham/masscan`](https://github.com/robertdavidgraham/masscan)。

对于非常大的网络，一个常见的方法是使用 MASSCAN（或调整为更快扫描的 Nmap）进行初始扫描。然后可以使用来自初步扫描的输出来“馈送”下一个工具，无论是 Nmap 还是可能是其他工具，也许是诸如 Nessus 或 OpenVAS 之类的安全扫描工具。像这样将工具“链接”在一起，最大程度地发挥各自的优势，以在最短的时间内实现最佳结果。

所有工具都有其局限性，IPv6 网络对于扫描工具仍然是一个挑战。除非您可以以某种方式限制范围，否则 IPv6 将很快达到扫描主机的网络带宽、时间和内存的极限。DNS 收集等工具可以在这方面提供帮助 - 如果您可以在扫描服务之前确定哪些主机实际上是活动的，那么可以将目标地址数量显着减少到可管理的范围内。

端口扫描已经完成，让我们离开有线世界，探索在无线网络上使用 Linux 进行故障排除。

# 无线诊断操作

无线网络中的诊断工具通常关注于发现信号强度低和干扰的区域 - 这些问题会给使用您的无线网络的人造成问题。

有一些基于 Linux 的优秀无线工具，但我们将讨论 Kismet、Wavemon 和 LinSSID。这三个工具都是免费的，并且都可以使用标准的`apt-get install <package name>`命令进行安装。如果将工具搜索范围扩大到包括攻击类型工具或商业产品，那么列表显然会变得更大。

Kismet 是 Linux 上可用的较旧的无线工具之一。我第一次接触它是作为信息安全工具，强调“隐藏”的无线 SSID 实际上根本不是隐藏的！

要运行该工具，请使用以下命令：

```
$ sudo kismet –c <wireless interface name>
```

或者，如果您有一个完全工作的配置，不需要实际的服务器窗口，运行以下命令：

```
$ sudo kismet –c <wireless interface name> &
```

现在，在另一个窗口中（或者如果您在后台运行 Kismet，则在同一个位置），运行 Kismet 客户端：

```
$ kismet_client
```

在显示中，您将看到各种 SSID 以及传输它们的接入点的 BSSID。当您浏览此列表时，您将看到用于每个 SSID 的信道和加密类型，您的笔记本电脑理解它可以在该 SSID 上协商的速度，以及该 SSID 上的所有客户端站点。每个客户端都将显示其 MAC 地址、频率和数据包计数。这些信息都作为每个客户端的关联过程和持续连接的“握手”部分以明文发送。

由于您的无线适配器一次只能在一个 SSID/BSSID 组合上，所以所呈现的信息是通过在信道之间跳转收集的。

在下面的屏幕截图中，我们展示了一个隐藏的 SSID，显示了接入点的 BSSID，以及与该接入点上的该 SSID 关联的八个客户端：

![图 3.11 - 主屏幕上的典型 Kismet 输出](img/B16336_03_011.jpg)

图 3.11 - 主屏幕上的典型 Kismet 输出

在网络上按*Enter*键会给您更多关于从该接入点广播的 SSID 的信息。请注意，我们在这个显示中看到了一个**隐藏的 SSID**：

![图 3.12 - Kismet 输出，接入点/SSID 详细信息](img/B16336_03_012.jpg)

图 3.12 - Kismet 输出，接入点/SSID 详细信息

进一步深入，您可以获取有关客户端活动的详细信息：

![图 3.13 - Kismet 输出，客户端详细信息](img/B16336_03_013.jpg)

图 3.13 - Kismet 输出，客户端详细信息

Kismet 是一款用于侦察和演示的好工具，但菜单相当容易迷失在其中，当排除信号强度故障时，很难专注于追踪我们真正关心的事物。

Wavemon 是一个非常不同的工具。它只监视您的连接，因此您必须将其与一个 SSID 关联起来。它会给出您当前的接入点、速度、信道等信息，如下面的屏幕截图所示。这可能是有用的，但它只是对通常用于故障排除所需的信息的狭窄视图 - 请注意在下面的屏幕截图中，报告的值主要是关于数据吞吐量和信号，从适配器关联到的网络中看到的。因此，Wavemon 工具主要用于故障排除上行问题，并且在故障排除、评估或查看整体无线基础设施方面并不经常使用：

![图 3.14 - Wavemon 显示](img/B16336_03_014.jpg)

图 3.14 - Wavemon 显示

更有用的是**LinSSID**，这是对 MetaGeek 的 Windows 应用 inSSIDer 的一个相当接近的移植。运行应用程序后，屏幕相当空白。选择要用于“嗅探”本地无线网络的无线适配器，然后按**Run**按钮。

显示了两个频谱上可用的信道（2.4 和 5 GHz），每个 SSID 在顶部窗口中表示。在列表中选中的每个 SSID/BSSID 组合都显示在底部窗口中。这样很容易看到列表中每个 AP 的信号强度，以及图形显示中的相对强度。在它们重叠的图形显示中，相互干扰的 SSID 是显而易见的。下面的屏幕截图显示了 5 GHz 频谱情况 - 请注意 AP 似乎都聚集在两个信道周围。其中任何一个都可以通过更改信道来提高性能，在我们的显示中有很多信道可以使用 - 实际上，这正是推动迁移到 5 GHz 的原因。是的，这个频段更快，但更重要的是，它更容易解决来自邻近接入点的任何干扰问题。还要注意，图表上显示的每个信道大约占用 20 GHz（稍后会详细介绍）：

![图 3.15 - LinSSID 输出 - 主屏显示信道分配和强度，文本和图形两种方式](img/B16336_03_015.jpg)

图 3.15 - LinSSID 输出 - 主屏显示信道分配和强度，文本和图形两种方式

2.4 GHz 信道也不好。由于在北美只有 11 个信道可用，通常会看到人们选择信道 1、6 或 11 - 这 3 个信道不会相互干扰。在几乎任何不是农村的环境中，您会看到几个邻居使用您认为是空闲的那 3 个信道！在下面的屏幕截图中，我们看到每个人都选择了信道 11，原因不明：

![图 3.16 - 来自无线邻居的干扰 - 多个无线 BSSID 使用相同的信道](img/B16336_03_016.jpg)

图 3.16 - 来自无线邻居的干扰 - 多个无线 BSSID 使用相同的信道

在第二个例子中（也来自 2.4 GHz 频谱），我们看到人们选择了更宽的信号“足迹”的结果。在 802.11 无线中，您可以将默认的 20 GHz 信道扩展到 40 或 80 GHz。这样做的好处是 - 在没有任何邻居的情况下 - 这肯定会提高吞吐量，特别是对于轻度使用的信道（例如一个或两个客户端）。然而，在相邻接入点有重叠信号的环境中，您会发现增加信道宽度（在 2.4 GHz 频段上）会给每个人带来更多的干扰 - 相邻接入点可能发现自己没有好的信道选择。这种情况通常会影响每个人的信号质量（和吞吐量），包括选择增加信道宽度的那个“坏邻居”。

在 5 GHz 频段，有更多的信道，因此通常可以更安全地增加信道宽度。不过，在选择或扩大接入点上的信道之前，最好先看看您的频谱发生了什么：

![图 3.17 - 在 2.4 GHz 频谱中使用更宽的信道宽度，导致干扰](img/B16336_03_017.jpg)

图 3.17 - 在 2.4 GHz 频谱中使用更宽的信道宽度，导致干扰

在我们讨论过的工具中，LinSSID 特别适用于进行无线站点调查，您需要查看可用的信道，更重要的是，跟踪信号强度并找到“死角”，以最大限度地覆盖建筑物或区域的无线覆盖范围。 LinSSID 也是我们讨论过的工具中最有帮助的，可以找到信道干扰的情况，或者解决信道宽度选择不当的情况。

通过我们讨论和探讨过的工具，您现在应该能够很好地解决 2.4 GHz 和 5 GHz 频段的无线信号强度和干扰问题。您应该能够使用诸如 Kismet 之类的工具来查找隐藏的 SSID，使用诸如 Wavemon 之类的工具来解决您所关联的网络的问题，并使用诸如 LinSSID 之类的工具来全面查看无线频谱，寻找干扰和信号强度问题，以及信道宽度和信道重叠问题。 

# 总结

通过本章的学习，您应该对 OSI 模型中各种网络和应用协议的分层组织有很好的理解。您应该对 TCP 和 UDP 有扎实的了解，特别是这两种协议如何使用端口，以及 TCP 会话如何建立和终止。使用`netstat`或`ss`查看您的主机如何连接到各种远程服务，或者您的主机正在监听哪些服务，这是您可以继续使用的技能。在此基础上，使用端口扫描程序查看您的组织中运行着哪些主机和网络服务应该是您会发现有用的技能。最后，我们讨论的 Linux 无线工具应该有助于故障排除、配置和无线站点调查。所有这些技能将是我们在本书中继续前进时建立的东西，但更重要的是，它们将有助于您在组织中解决应用程序和网络问题。

这结束了我们对使用 Linux 进行网络故障排除的讨论。尽管在大多数章节中我们会重新讨论故障排除-随着我们前进并构建基础设施的每个部分，我们会发现新的潜在问题和故障排除方法。在本节中，我们详细讨论了通信从网络和主机的角度发生的情况。在下一章中，我们将讨论 Linux 防火墙，这是限制和控制通信的好方法。

# 问题

随着我们的结束，这里有一系列问题供您测试对本章材料的了解。您将在*附录*的*评估*部分找到答案：

1.  当您使用`netstat`、`ss`或其他命令评估本地端口时，您是否会看到处于`ESTABLISHED`状态的 UDP 会话？

1.  确定哪些进程监听哪些端口是重要的吗？

1.  确定从任何特定应用程序连接到哪些远程端口是重要的吗？

1.  为什么要扫描除`tcp/443`以外的端口上即将过期或即将过期的证书？

1.  为什么 netcat 需要`sudo`权限才能在端口`80`上启动监听器？

1.  在 2.4 GHz 频段中，哪三个信道是最佳选择以减少干扰？

1.  除了 20 GHz 之外，您何时会使用其他 Wi-Fi 信道宽度？

# 进一步阅读

+   OSI 模型（*ISO/IED 7498-1*）：[`standards.iso.org/ittf/PubliclyAvailableStandards/s020269_ISO_IEC_7498-1_1994(E).zip`](https://standards.iso.org/ittf/PubliclyAvailableStandards/s020269_ISO_IEC_7498-1_1994(E).zip)

+   Nmap：[`nmap.org/`](https://nmap.org/)

+   Nmap 参考指南：[`nmap.org/book/man.html`](https://nmap.org/book/man.html)

[`www.amazon.com/Nmap-Network-Scanning-Official-Discovery/dp/0979958717`](https://www.amazon.com/Nmap-Network-Scanning-Official-Discovery/dp/0979958717)

+   MASSCAN：[`github.com/robertdavidgraham/masscan`](https://github.com/robertdavidgraham/masscan)
