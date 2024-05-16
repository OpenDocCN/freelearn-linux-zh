# 第七章：老男孩网络

在本章中，我们将涵盖：

+   基本网络入门

+   让我们 ping 一下！

+   列出网络上所有活动的机器

+   通过网络传输文件

+   使用脚本设置以太网和无线局域网

+   使用 SSH 进行无密码自动登录

+   使用 SSH 在远程主机上运行命令

+   在本地挂载点挂载远程驱动器

+   在网络上多播窗口消息

+   网络流量和端口分析

# 介绍

网络是通过网络互连机器并配置网络中的节点具有不同规格的行为。我们使用 TCP/IP 作为我们的网络堆栈，并且所有操作都基于它。网络是每个计算机系统的重要组成部分。网络中连接的每个节点都被分配一个唯一的 IP 地址以进行标识。网络中有许多参数，如子网掩码、路由、端口、DNS 等，需要基本的理解才能跟进。

许多使用网络的应用程序通过打开和连接到防火墙端口来运行。每个应用程序可能提供诸如数据传输、远程 shell 登录等服务。在由许多机器组成的网络上可以执行许多有趣的管理任务。Shell 脚本可用于配置网络中的节点、测试机器的可用性、自动执行远程主机上的命令等。本章重点介绍了介绍有趣的与网络相关的工具或命令的不同配方，以及它们如何用于解决不同的问题。

# 基本网络入门

在基于网络的配方之前，您有必要对设置网络、术语和命令进行基本了解，例如分配 IP 地址、添加路由等。本配方将概述 GNU/Linux 中用于网络的不同命令及其用法。

## 准备工作

网络中的每个节点都需要分配许多参数才能成功工作并与其他机器互连。一些不同的参数包括 IP 地址、子网掩码、网关、路由、DNS 等。

本配方将介绍`ifconfig`、`route`、`nslookup`和`host`命令。

## 如何做...

网络接口用于连接到网络。通常，在类 UNIX 操作系统的上下文中，网络接口遵循 eth0、eth1 的命名约定。此外，还有其他接口，如 usb0、wlan0 等，可用于 USB 网络接口、无线局域网等网络。

`ifconfig`是用于显示网络接口、子网掩码等详细信息的命令。

`ifconfig`位于`/sbin/ifconfig`。当键入`ifconfig`时，一些 GNU/Linux 发行版会显示错误“找不到命令”。这是因为用户的 PATH 环境变量中没有包含`/sbin`。当键入命令时，Bash 会在 PATH 变量中指定的目录中查找。

默认情况下，在 Debian 中，`ifconfig`不可用，因为`/sbin`不在 PATH 中。

`/sbin/ifconfig`是绝对路径，因此尝试使用绝对路径（即`/sbin/ifconfig`）运行 ifconfig。对于每个系统，默认情况下都会有一个名为'lo'的接口，称为环回，指向当前机器。例如：

```
$ ifconfig
lo        Link encap:Local Loopback
inet addr:127.0.0.1  Mask:255.0.0.0
inet6addr: ::1/128 Scope:Host
 UP LOOPBACK RUNNING  MTU:16436  Metric:1
 RX packets:6078 errors:0 dropped:0 overruns:0 frame:0
 TX packets:6078 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:0
 RX bytes:634520 (634.5 KB)  TX bytes:634520 (634.5 KB)

wlan0     Link encap:EthernetHWaddr 00:1c:bf:87:25:d2
inet addr:192.168.0.82  Bcast:192.168.3.255  Mask:255.255.252.0
inet6addr: fe80::21c:bfff:fe87:25d2/64 Scope:Link
 UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
 RX packets:420917 errors:0 dropped:0 overruns:0 frame:0
 TX packets:86820 errors:0 dropped:0 overruns:0 carrier:0
collisions:0 txqueuelen:1000
 RX bytes:98027420 (98.0 MB)  TX bytes:22602672 (22.6 MB)

```

`ifconfig`输出中最左边的列显示了网络接口的名称，右侧列显示了与相应网络接口相关的详细信息。

## 还有更多...

有几个附加命令经常用于查询和配置网络。让我们一起了解基本命令和用法。

### 打印网络接口列表

以下是一个一行命令序列，用于打印系统上可用的网络接口列表。

```
$ ifconfig | cut -c-10 | tr -d ' ' | tr -s '\n'
lo
wlan0

```

`ifconfig`输出的每行的前 10 个字符用于写入网络接口的名称。因此，我们使用`cut`提取每行的前 10 个字符。`tr -d ' '`删除每行中的每个空格字符。现在使用`tr -s '\n'`来压缩`\n`换行符，以产生一个接口名称列表。

### 分配和显示 IP 地址

`ifconfig`命令显示系统上可用的每个网络接口的详细信息。但是，我们可以通过使用以下命令将其限制为特定接口：

```
$ ifconfig iface_name

```

例如：

```
$ ifconfig wlan0
wlan0     Link encap:Ethernet HWaddr 00:1c:bf:87:25:d2
inet addr:192.168.0.82  Bcast:192.168.3.255
 Mask:255.255.252.0

```

从前面提到的命令的输出中，我们感兴趣的是 IP 地址、广播地址、硬件地址和子网掩码。它们如下：

+   `HWaddr 00:1c:bf:87:25:d2`是硬件地址（MAC 地址）

+   `inet addr:192.168.0.82`是 IP 地址

+   `Bcast:192.168.3.255`是广播地址

+   子网掩码：255.255.252.0

在几种脚本上下文中，我们可能需要从脚本中提取这些地址中的任何一个以进行进一步操作。

提取 IP 地址是一个常见的任务。为了从`ifconfig`输出中提取 IP 地址，请使用：

```
$ ifconfig wlan0 | egrep -o "inet addr:[^ ]*" | grep -o "[0-9.]*"
192.168.0.82

```

第一个命令`egrep -o "inet addr:[^ ]*"`将打印`inet addr:192.168.0.82`。

模式以`inet addr:`开头，以一些非空格字符序列（由`[^ ]*`指定）结尾。现在在下一个管道中，它打印数字和'.'的字符组合。

为了设置网络接口的 IP 地址，请使用：

```
# ifconfig wlan0 192.168.0.80

```

您需要以 root 身份运行上述命令。`192.168.0.80`是要设置的地址。

设置子网掩码以及 IP 地址如下：

```
# ifconfig wlan0 192.168.0.80  netmask 255.255.252.0

```

### 欺骗硬件地址（MAC 地址）

在某些情况下，通过使用硬件地址提供对网络上计算机的身份验证或过滤，我们可以使用硬件地址欺骗。硬件地址显示为`ifconfig`输出中的`HWaddr 00:1c:bf:87:25:d2`。

我们可以在软件级别欺骗硬件地址，如下所示：

```
# ifconfig eth0 hw ether 00:1c:bf:87:25:d5

```

在上述命令中，`00:1c:bf:87:25:d5`是要分配的新 MAC 地址。

当我们需要通过 MAC 认证的服务提供商访问互联网以为单个机器提供互联网访问时，这可能是有用的。

### 名称服务器和 DNS（域名服务）

互联网的基本寻址方案是 IP 地址（点分十进制形式，例如`202.11.32.75`）。但是，互联网上的资源（例如网站）是通过称为 URL 或域名的 ASCII 字符组合访问的。例如，[google.com](http://google.com)是一个域名。它实际上对应一个 IP 地址。在浏览器中输入 IP 地址也可以访问 URL`www.google.com`。

将 IP 地址与符号名称抽象化的技术称为**域名服务**（**DNS**）。当我们输入`google.com`时，配置了我们网络的 DNS 服务器将域名解析为相应的 IP 地址。而在本地网络上，我们可以使用本地 DNS 通过主机名对网络上的本地机器进行符号命名。

分配给当前系统的名称服务器可以通过读取`/etc/resolv.conf`来查看。例如：

```
$ cat /etc/resolv.conf
nameserver 8.8.8.8

```

我们可以手动添加名称服务器，如下所示：

```
# echo nameserver IP_ADDRESS >> /etc/resolv.conf

```

我们如何获取相应域名的 IP 地址？

获取 IP 地址的最简单方法是尝试 ping 给定的域名，并查看回显回复。例如：

```
$ ping google.com
PING google.com (64.233.181.106) 56(84) bytes of data.
Here 64.233.181.106 is the corresponding IP address.

```

一个域名可以分配多个 IP 地址。在这种情况下，DNS 服务器将从 IP 地址列表中返回一个地址。要获取分配给域名的所有地址，我们应该使用 DNS 查找实用程序。

### DNS 查找

命令行中有不同的 DNS 查找实用程序可用。这些将请求 DNS 服务器进行 IP 地址解析。`host`和`nslookup`是两个 DNS 查找实用程序。

当执行`host`时，它将列出附加到域名的所有 IP 地址。`nslookup`是另一个类似于`host`的命令，它可以用于查询与 DNS 和名称解析相关的详细信息。例如：

```
$ host google.com
google.com has address 64.233.181.105
google.com has address 64.233.181.99
google.com has address 64.233.181.147
google.com has address 64.233.181.106
google.com has address 64.233.181.103
google.com has address 64.233.181.104

```

它还可以列出 DNS 资源记录，如 MX（邮件交换器）如下：

```
$ nslookup google.com
Server:    8.8.8.8
Address:  8.8.8.8#53

Non-authoritative answer:
Name:  google.com
Address: 64.233.181.105
Name:  google.com
Address: 64.233.181.99
Name:  google.com
Address: 64.233.181.147
Name:  google.com
Address: 64.233.181.106
Name:  google.com
Address: 64.233.181.103
Name:  google.com
Address: 64.233.181.104

Server:    8.8.8.8

```

上面的最后一行对应于用于 DNS 解析的默认名称服务器。

在不使用 DNS 服务器的情况下，可以通过向文件`/etc/hosts`添加条目来将符号名称添加到 IP 地址解析中。

要添加一个条目，请使用以下语法：

```
# echo IP_ADDRESS symbolic_name >> /etc/hosts

```

例如：

```
# echo 192.168.0.9 backupserver.com  >> /etc/hosts

```

添加了这个条目后，每当解析到`backupserver.com`时，它将解析为`192.168.0.9`。

### 设置默认网关，显示路由表信息

当本地网络连接到另一个网络时，需要分配一些机器或网络节点，通过这些节点进行互连。因此，目的地在本地网络之外的 IP 数据包应该被转发到与外部网络相连的节点机器。这个特殊的节点机器，能够将数据包转发到外部网络，被称为网关。我们为每个节点设置网关，以便连接到外部网络。

操作系统维护一个称为路由表的表，其中包含有关数据包如何转发以及通过网络中的哪个机器节点的信息。路由表可以显示如下：

```
$ route
Kernel IP routing table
Destination      Gateway   Genmask      Flags  Metric  Ref  UseIface
192.168.0.0         *      255.255.252.0  U     2      0     0wlan0
link-local          *      255.255.0.0    U     1000   0     0wlan0
default          p4.local  0.0.0.0        UG    0      0     0wlan0

```

或者，您也可以使用：

```
$ route -n
Kernel IP routing table
Destination   Gateway      Genmask       Flags Metric Ref  Use   Iface
192.168.0.0   0.0.0.0      255.255.252.0   U     2     0     0   wlan0
169.254.0.0   0.0.0.0      255.255.0.0     U     1000  0     0   wlan0
0.0.0.0       192.168.0.4  0.0.0.0         UG    0     0     0   wlan0

```

使用`-n`指定显示数字地址。当使用`-n`时，它将显示每个带有数字 IP 地址的条目，否则它将显示 DNS 条目下的符号主机名而不是 IP 地址。

默认网关设置如下：

```
# route add default gw IP_ADDRESS INTERFACE_NAME

```

例如：

```
# route add default gw 192.168.0.1 wlan0

```

### Traceroute

当应用程序通过 Internet 请求服务时，服务器可能位于遥远的位置，并通过任意数量的网关或设备节点连接。数据包通过多个网关传输并到达目的地。有一个有趣的命令`traceroute`，它显示了数据包到达目的地所经过的所有中间网关的地址。`traceroute`信息帮助我们了解每个数据包需要经过多少跳才能到达目的地。中间网关或路由器的数量为连接在大型网络中的两个节点之间的距离提供了一个度量标准。`traceroute`的输出示例如下：

```
$ traceroute google.com
traceroute to google.com (74.125.77.104), 30 hops max, 60 byte packets
1  gw-c6509.lxb.as5577.net (195.26.4.1)  0.313 ms  0.371 ms  0.457 ms
2  40g.lxb-fra.as5577.net (83.243.12.2)  4.684 ms  4.754 ms  4.823 ms
3  de-cix10.net.google.com (80.81.192.108)  5.312 ms  5.348 ms  5.327 ms
4  209.85.255.170 (209.85.255.170)  5.816 ms  5.791 ms 209.85.255.172 (209.85.255.172)  5.678 ms
5  209.85.250.140 (209.85.250.140)  10.126 ms  9.867 ms  10.754 ms
6  64.233.175.246 (64.233.175.246)  12.940 ms 72.14.233.114 (72.14.233.114)  13.736 ms  13.803 ms
7  72.14.239.199 (72.14.239.199)  14.618 ms 209.85.255.166 (209.85.255.166)  12.755 ms 209.85.255.143 (209.85.255.143)  13.803 ms
8  209.85.255.98 (209.85.255.98)  22.625 ms 209.85.255.110 (209.85.255.110)  14.122 ms
* 
9  ew-in-f104.1e100.net (74.125.77.104)  13.061 ms  13.256 ms  13.484 ms

```

## 另请参阅

+   *使用变量和环境变量玩耍* 第一章 ，解释了 PATH 变量

+   *使用 grep 在文件中搜索和挖掘“文本”* 第四章 ，解释了 grep 命令

# 让我们 ping！

`ping`是最基本的网络命令，每个用户都应该首先了解。它是一个通用命令，在主要操作系统上都可以使用。它也是一个用于验证网络上两个主机之间连接性的诊断工具。它可以用来查找网络上哪些机器是活动的。让我们看看如何使用 ping。

## 如何做...

为了检查网络上两个主机的连接性，`ping`命令使用**Internet** **Control** **Message** **Protocol** (**ICMP**)回显数据包。当这些回显数据包发送到主机时，如果主机是可达或活动的，主机将以回复的方式响应。

检查主机是否可达如下：

```
$ ping ADDRESS

```

`ADDRESS`可以是主机名、域名或 IP 地址本身。

`ping`将持续发送数据包，并且回复信息将打印在终端上。通过按下*Ctrl* + *C*来停止 ping。

例如：

+   当主机是可达时，输出将类似于以下内容：

```
$ ping 192.168.0.1 
PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
64 bytes from 192.168.0.1: icmp_seq=1 ttl=64 time=1.44 ms
^C 
--- 192.168.0.1 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.440/1.440/1.440/0.000 ms

$ ping google.com
PING google.com (209.85.153.104) 56(84) bytes of data.
64 bytes from bom01s01-in-f104.1e100.net (209.85.153.104): icmp_seq=1 ttl=53 time=123 ms
^C 
--- google.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 123.388/123.388/123.388/0.000 ms

```

+   当主机不可达时，输出将类似于：

```
$ ping 192.168.0.99
PING 192.168.0.99 (192.168.0.99) 56(84) bytes of data.
From 192.168.0.82 icmp_seq=1 Destination Host Unreachable
From 192.168.0.82 icmp_seq=2 Destination Host Unreachable

```

一旦主机不可达，ping 将返回`Destination Host Unreachable`错误消息。

## 有更多

除了检查网络中两点之间的连接性外，`ping`命令还可以与其他选项一起使用以获得有用的信息。让我们看看`ping`的其他选项。

### 往返时间

`ping`命令可用于查找网络上两个主机之间的**往返时间**（**RTT**）。 RTT 是数据包到达目标主机并返回到源主机所需的时间。 RTT 以毫秒为单位可以从 ping 中获得。示例如下：

```
--- google.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4000ms
rtt min/avg/max/mdev = 118.012/206.630/347.186/77.713 ms

```

这里最小的 RTT 为 118.012ms，平均 RTT 为 206.630ms，最大 RTT 为 347.186ms。 `ping`输出中的`mdev`（77.713ms）参数代表平均偏差。

### 限制要发送的数据包数量

`ping`命令发送回显数据包，并无限期等待`echo`的回复，直到通过按下*Ctrl* + *C*停止。但是，我们可以使用`-c`标志来限制要发送的回显数据包的数量。

用法如下：

```
-c COUNT
```

例如：

```
$ ping 192.168.0.1 -c 2 
PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data. 
64 bytes from 192.168.0.1: icmp_seq=1 ttl=64 time=4.02 ms 
64 bytes from 192.168.0.1: icmp_seq=2 ttl=64 time=1.03 ms 

--- 192.168.0.1 ping statistics --- 
2 packets transmitted, 2 received, 0% packet loss, time 1001ms 
rtt min/avg/max/mdev = 1.039/2.533/4.028/1.495 ms 

```

在上一个示例中，`ping`命令发送两个回显数据包并停止。

当我们需要通过脚本 ping 多台机器并检查其状态时，这是非常有用的。

### ping 命令的返回状态

`ping`命令在成功时返回退出状态 0，并在失败时返回非零。成功意味着目标主机是可达的，而失败意味着目标主机是不可达的。

返回状态可以轻松获得如下：

```
$ ping ADDRESS -c2
if [ $? -eq 0 ];
then
 echo Successful ;
else
 echo Failure
fi

```

# 列出网络上所有活动的机器

当我们处理大型局域网时，我们可能需要检查网络中其他机器的可用性，无论是存活还是不存活。机器可能不存活有两种情况：要么它没有通电，要么由于网络中的问题。通过使用 shell 脚本，我们可以轻松地找出并报告网络上哪些机器是活动的。让我们看看如何做到这一点。

## 准备就绪

在这个配方中，我们使用了两种方法。第一种方法使用`ping`，第二种方法使用`fping`。`fping`不会默认随 Linux 发行版一起提供。您可能需要使用软件包管理器手动安装`fping`。

## 如何做到...

让我们通过脚本找出网络上所有活动的机器以及查找相同的替代方法。

+   **方法 1：**

我们可以使用`ping`命令编写自己的脚本来查询 IP 地址列表，并检查它们是否存活，如下所示：

```
#!/bin/bash
#Filename: ping.sh
# Change base address 192.168.0 according to your network.

for ip in 192.168.0.{1..255} ;
do
  ping $ip -c 2 &> /dev/null ;

  if [ $? -eq 0 ];
  then
    echo $ip is alive
  fi

done
```

输出如下：

```
$ ./ping.sh
192.168.0.1 is alive
192.168.0.90 is alive

```

+   **方法 2：**

我们可以使用现有的命令行实用程序来查询网络上计算机的状态，如下所示：

```
$ fping -a 192.160.1/24 -g 2> /dev/null 
192.168.0.1 
192.168.0.90

```

或者，使用：

```
$ fping -a 192.168.0.1 192.168.0.255 -g

```

## 它是如何工作的...

在方法 1 中，我们使用`ping`命令来查找网络上存活的计算机。我们使用`for`循环来遍历 IP 地址列表。列表生成为`192.168.0.{1..255}`。`{start..end}`符号将扩展并生成 IP 地址列表，例如`192.168.0.1`，`192.168.0.2`，`192.168.0.3`直到`192.168.0.255`。

`ping $ip -c 2 &> /dev/null`将在每次循环执行时对相应的 IP 地址运行`ping`。`-c 2`用于限制要发送的回显数据包的数量为两个数据包。`&> /dev/null`用于将`stderr`和`stdout`重定向到`/dev/null`，以便它不会打印在终端上。使用`$?`我们评估退出状态。如果成功，退出状态为 0，否则为非零。因此，成功的 IP 地址将被打印。我们还可以打印不成功的 IP 地址列表，以提供不可达的 IP 地址列表。

### 注意

这里有一个练习给你。不要在脚本中使用硬编码的 IP 地址范围，而是修改脚本以从文件或`stdin`中读取 IP 地址列表。

在这个脚本中，每个 ping 都是依次执行的。尽管所有 IP 地址彼此独立，但由于是顺序程序，`ping`命令会由于发送两个回显数据包和接收它们的延迟或执行下一个`ping`命令的超时而执行。

当涉及到 255 个地址时，延迟很大。让我们并行运行所有`ping`命令，使其更快。脚本的核心部分是循环体。为了使`ping`命令并行运行，将循环体括在`( )&`中。`( )`括起一组命令以作为子 shell 运行，`&`通过离开当前线程将其发送到后台。例如：

```
(
 ping $ip -c2 &> /dev/null ;

 if [ $? -eq 0 ];
 then
 echo $ip is alive
 fi
)&

wait

```

`for`循环体执行许多后台进程，退出循环并终止脚本。为了使脚本在所有子进程结束之前终止，我们有一个称为`wait`的命令。在脚本的末尾放置一个`wait`，以便它等待直到所有子`( )`子 shell 进程完成。

### 注意

`wait`命令使脚本只有在所有子进程或后台进程终止或完成后才能终止。

查看书中提供的代码中的`fast_ping.sh`。

方法 2 使用了一个名为`fping`的不同命令。它可以同时 ping 一系列 IP 地址并非常快速地响应。`fping`可用的选项如下：

+   `fping`的`-a`选项指定打印所有活动机器的 IP 地址

+   `-u`选项与`fping`一起指定打印所有不可达的机器

+   `-g`选项指定从斜杠子网掩码表示法指定的 IP/mask 或起始和结束 IP 地址生成 IP 地址范围：

```
$ fping -a 192.160.1/24 -g

```

或

```
$ fping -a 192.160.1 192.168.0.255 -g

```

+   `2>/dev/null`用于将由于不可达主机而打印的错误消息转储到空设备

还可以手动指定 IP 地址列表作为命令行参数或通过`stdin`列表。例如：

```
$ fping -a 192.168.0.1 192.168.0.5 192.168.0.6
# Passes IP address as arguments
$ fping -a <ip.list
# Passes a list of IP addresses from a file

```

## 还有更多...

`fping`命令可用于从网络查询 DNS 数据。让我们看看如何做到这一点。

### 使用 fping 进行 DNS 查找

`fping`有一个`-d`选项，通过使用 DNS 查找返回主机名。它将在 ping 回复中打印主机名而不是 IP 地址。

```
$ cat ip.list
192.168.0.86
192.168.0.9
192.168.0.6

$ fping -a -d 2>/dev/null  <ip.list
www.local
dnss.local

```

## 另请参阅

+   *玩转文件描述符和重定向*第一章，解释了数据重定向

+   *比较和测试*第一章，解释数字比较

# 文件传输

计算机网络的主要目的是资源共享。在资源共享中，最突出的用途是文件共享。有多种方法可以在网络上的不同节点之间传输文件。本教程讨论了如何使用常用协议 FTP、SFTP、RSYNC 和 SCP 进行文件传输。

## 准备工作

执行网络文件传输的命令在 Linux 安装中通常是默认可用的。可以使用`lftp`命令通过 FTP 传输文件。可以使用`sftp`通过 SSH 连接传输文件，使用`rsync`命令和使用`scp`通过 SSH 进行传输。

## 如何做...

**文件传输协议**（**FTP**）是一种用于在网络上的机器之间传输文件的旧文件传输协议。我们可以使用`lftp`命令来访问启用 FTP 的服务器进行文件传输。它使用端口 21。只有在远程机器上安装了 FTP 服务器才能使用 FTP。许多公共网站使用 FTP 共享文件。

要连接到 FTP 服务器并在其间传输文件，请使用：

```
$ lftp username@ftphost

```

现在它将提示输入密码，然后显示如下的登录提示：

```
lftp username@ftphost:~> 

```

您可以在此提示中键入命令。例如：

+   要更改目录，请使用`cd directory`

+   要更改本地机器的目录，请使用`lcd`

+   要创建目录，请使用`mkdir`

+   要下载文件，请使用`get filename`如下：

```
lftp username@ftphost:~> get filename

```

+   要从当前目录上传文件，请使用`put filename`如下：

```
lftp username@ftphost:~> put filename

```

+   可以使用`quit`命令退出`lftp`会话

`lftp`提示中支持自动完成。

## 还有更多...

让我们来看一些用于通过网络传输文件的其他技术和命令。

### 自动 FTP 传输

`ftp`是另一个用于基于 FTP 的文件传输的命令。`lftp`更灵活。`lftp`和`ftp`命令打开一个与用户的交互会话（它通过显示消息提示用户输入）。如果我们想要自动化文件传输而不是使用交互模式怎么办？我们可以通过编写一个 shell 脚本来自动化 FTP 文件传输，如下所示：

```
#!/bin/bash
#Filename: ftp.sh
#Automated FTP transfer
HOST='domain.com'
USER='foo'
PASSWD='password'
ftp -i -n $HOST <<EOF
user ${USER} ${PASSWD}
binary
cd /home/slynux
puttestfile.jpg
getserverfile.jpg
quit
EOF
```

上述脚本具有以下结构：

```
<<EOF
DATA
EOF
```

这用于通过`stdin`发送数据到 FTP 命令。第一章中的*Playing with file descriptors and redirection*一节解释了各种重定向到`stdin`的方法。

`ftp`的`-i`选项关闭与用户的交互会话。`user ${USER} ${PASSWD}`设置用户名和密码。`binary`将文件模式设置为二进制。

### SFTP（安全 FTP）

SFTP 是一种类似 FTP 的文件传输系统，运行在 SSH 连接的顶部。它利用 SSH 连接来模拟 FTP 接口。它不需要远程端安装 FTP 服务器来执行文件传输，但需要安装和运行 OpenSSH 服务器。它是一个交互式命令，提供`sftp`提示符。

以下命令用于执行文件传输。对于具有特定 HOST、USER 和 PASSWD 的每个自动化 FTP 会话，所有其他命令保持不变：

```
cd /home/slynux
put testfile.jpg
get serverfile.jpg

```

要运行`sftp`，请使用：

```
$ sftp user@domainname

```

类似于`lftp`，`sftp`会话可以通过输入`quit`命令退出。

SSH 服务器有时不会在默认端口 22 上运行。如果它在不同的端口上运行，我们可以在`sftp`后面指定端口，如`-oPort=PORTNO`。

例如：

```
$ sftp -oPort=422 user@slynux.org

```

### 注意

`-oPort`应该是`sftp`命令的第一个参数。

### RSYNC

rsync 是一个重要的命令行实用程序，广泛用于在网络上复制文件和进行备份快照。这在单独的*使用 rsync 进行备份快照*一节中有更好的解释，解释了`rsync`的用法。

### SCP（安全复制）

SCP 是一种比传统的名为`rcp`的远程复制工具更安全的文件复制技术。文件通过加密通道传输。SSH 用作加密通道。我们可以通过以下方式轻松地将文件传输到远程计算机：

```
$ scp filename user@remotehost:/home/path

```

这将提示输入密码。可以通过使用自动登录 SSH 技术使其无需密码。*使用 SSH 进行无密码自动登录*一节解释了 SSH 自动登录。

因此，使用`scp`进行文件传输不需要特定的脚本。一旦 SSH 登录被自动化，`scp`命令可以在不需要交互式提示输入密码的情况下执行。

这里的`remotehost`可以是 IP 地址或域名。`scp`命令的格式是：

```
$ scp SOURCE DESTINATION

```

`SOURCE`或`DESTINATION`可以采用`username@localhost:/path`的格式，例如：

```
$ scp user@remotehost:/home/path/filename filename

```

上述命令将文件从远程主机复制到当前目录，并给出文件名。

如果 SSH 运行的端口与 22 不同，请使用与`sftp`相同语法的`-oPort`。

### 使用 SCP 进行递归复制

通过使用`scp`，我们可以通过以下方式在网络上的两台计算机之间递归复制目录，使用`-r`参数：

```
$ scp -r /home/slynux user@remotehost:/home/backups
# Copies the directory /home/slynux recursively to remote location

```

`scp`也可以通过使用`-p`参数保留权限和模式来复制文件。

## 另请参阅

+   第一章的*Playing with file descriptors and redirection*一节解释了使用 EOF 的标准输入

# 使用脚本设置以太网和无线局域网

以太网配置简单。由于它使用物理电缆，因此没有特殊要求，如身份验证。然而，无线局域网需要身份验证，例如 WEP 密钥以及要连接的无线网络的 ESSID。让我们看看如何通过编写一个 shell 脚本连接到无线网络和有线网络。

## 准备工作

连接有线网络时，我们需要使用`ifconfig`实用程序分配 IP 地址和子网掩码。但是对于无线网络连接，将需要额外的实用程序，如`iwconfig`和`iwlist`，以配置更多参数。

## 如何做...

为了从有线接口连接到网络，执行以下脚本：

```
#!/bin/bash
#Filename: etherconnect.sh
#Description: Connect Ethernet

#Modify the parameters below according to your settings
######### PARAMETERS ###########

IFACE=eth0
IP_ADDR=192.168.0.5
SUBNET_MASK=255.255.255.0
GW=192.168.0.1
HW_ADDR='00:1c:bf:87:25:d2'
# HW_ADDR is optional
#################################

if [ $UID -ne 0 ];
then
  echo "Run as root"
  exit 1
fi

# Turn the interface down before setting new config
/sbin/ifconfig $IFACE down

if [[ -n $HW_ADDR  ]];
then
  /sbin/ifconfig hw ether $HW_ADDR
 echo Spoofed MAC ADDRESS to $HW_ADDR

fi

/sbin/ifconfig $IFACE $IP_ADDR netmask $SUBNET_MASK

route add default gw $GW $IFACE

echo Successfully configured $IFACE
```

连接到带有 WEP 的无线局域网的脚本如下：

```
#!/bin/bash
#Filename: wlan_connect.sh
#Description: Connect to Wireless LAN

#Modify the parameters below according to your settings
######### PARAMETERS ###########
IFACE=wlan0
IP_ADDR=192.168.1.5
SUBNET_MASK=255.255.255.0
GW=192.168.1.1
HW_ADDR='00:1c:bf:87:25:d2' 
#Comment above line if you don't want to spoof mac address

ESSID="homenet"
WEP_KEY=8b140b20e7 
FREQ=2.462G
#################################

KEY_PART=""

if [[ -n $WEP_KEY ]];
then
  KEY_PART="key $WEP_KEY"
fi

# Turn the interface down before setting new config
/sbin/ifconfig $IFACE down

if [ $UID -ne 0 ];
then
  echo "Run as root"
  exit 1;
fi

if [[ -n $HW_ADDR  ]];
then
  /sbin/ifconfig $IFACE hw ether $HW_ADDR
  echo Spoofed MAC ADDRESS to $HW_ADDR
fi

/sbin/iwconfig $IFACE essid $ESSID $KEY_PART freq $FREQ

/sbin/ifconfig $IFACE $IP_ADDR netmask $SUBNET_MASK

route add default gw $GW $IFACE

echo Successfully configured $IFACE
```

## 它是如何工作的...

命令`ifconfig`、`iwconfig`和`route`需要以 root 用户身份运行。因此，在脚本开始时会检查 root 用户。

以太网连接脚本非常简单，并且使用了食谱中解释的概念，*基本网络入门*。让我们来看看用于连接到无线局域网的命令。

无线局域网需要一些参数，如`essid`、`key`和频率才能连接到网络。`essid`是我们需要连接的无线网络的名称。一些**有线** **等效** **协议**（**WEP**）网络使用 WEP 密钥进行身份验证，而有些网络则不使用。WEP 密钥通常是一个 10 位十六进制密码。接下来是分配给网络的频率。`iwconfig`是用于将无线网卡连接到适当的无线网络、WEP 密钥和频率的命令。

我们可以使用实用程序`iwlist`扫描和列出可用的无线网络。要进行扫描，请使用以下命令：

```
# iwlist scan 
wlan0     Scan completed : 
 Cell 01 - Address: 00:12:17:7B:1C:65
 Channel:11
 Frequency:2.462 GHz (Channel 11) 
 Quality=33/70  Signal level=-77 dBm
 Encryption key:on
 ESSID:"model-2" 

```

`频率`参数可以从扫描结果中提取，从`Frequency:2.462 GHz (Channel 11)`这一行中。

## 另请参阅

+   *比较和测试*第一章中，解释了字符串比较。

# 使用 SSH 进行无密码自动登录

SSH 在自动化脚本中被广泛使用。通过使用 SSH，可以在远程主机上远程执行命令并读取它们的输出。SSH 通过使用用户名和密码进行身份验证。在执行 SSH 命令时会提示输入密码。但是在自动化脚本中，SSH 命令可能会在循环中执行数百次，因此每次提供密码是不切实际的。因此我们需要自动化登录。SSH 具有一个内置功能，可以使用 SSH 密钥进行自动登录。本食谱描述了如何创建 SSH 密钥并实现自动登录。

## 如何做...

SSH 使用基于公钥和私钥的加密技术进行自动身份验证。身份验证密钥有两个元素：公钥和私钥对。我们可以使用`ssh-keygen`命令创建一个身份验证密钥。为了自动化身份验证，公钥必须放置在服务器上（通过将公钥附加到`~/.ssh/authorized_keys`文件），并且其配对的私钥文件应该存在于客户端机器的`~/.ssh`目录中，即您要从中登录的计算机。关于 SSH 的几个配置（例如`authorized_keys`文件的路径和名称）可以通过修改配置文件`/etc/ssh/sshd_config`来配置。

设置使用 SSH 进行自动身份验证有两个步骤。它们是：

1.  从需要登录到远程机器的机器上创建 SSH 密钥。

1.  将生成的公钥传输到远程主机，并将其附加到`~/.ssh/authorized_keys`文件中。

为了创建一个 SSH 密钥，输入以下指定 RSA 加密算法类型的`ssh-keygen`命令：

```
$ ssh-keygen -t rsa
Generating public/private rsa key pair. 
Enter file in which to save the key (/home/slynux/.ssh/id_rsa): 
Created directory '/home/slynux/.ssh'. 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/slynux/.ssh/id_rsa. 
Your public key has been saved in /home/slynux/.ssh/id_rsa.pub. 
The key fingerprint is: 
f7:17:c6:4d:c9:ee:17:00:af:0f:b3:27:a6:9c:0a:05slynux@slynux-laptop 
The key's randomart image is: 
+--[ RSA 2048]----+ 
|           .     | 
|            o . .|
|     E       o o.|
|      ...oo | 
|       .S .+  +o.| 
|      .  . .=....| 
|     .+.o...| 
|      . . + o.  .|
|       ..+       | 
+-----------------+ 

```

生成公私钥对需要输入一个密码。也可以在不输入密码的情况下生成密钥对，但这是不安全的。我们可以编写使用脚本从脚本到多台机器进行自动登录的监控脚本。在这种情况下，应该在运行`ssh-keygen`命令时将密码留空，以防止脚本在运行时要求输入密码。

现在`~/.ssh/id_rsa.pub`和`~/.ssh/id_rsa`已经生成。`id_dsa.pub`是生成的公钥，`id_dsa`是私钥。公钥必须附加到远程服务器上的`~/.ssh/authorized_keys`文件，我们需要从当前主机自动登录。

为了附加一个密钥文件，使用：

```
$ ssh USER@REMOTE_HOST "cat >> ~/.ssh/authorized_keys" < ~/.ssh/id_rsa.pub
Password:

```

在上一个命令中提供登录密码。

自动登录已设置。从现在开始，在执行过程中 SSH 不会提示输入密码。您可以使用以下命令进行测试：

```
$ ssh USER@REMOTE_HOST uname
Linux

```

您将不会被提示输入密码。

# 使用 SSH 在远程主机上运行命令

SSH 是一个有趣的系统管理工具，可以通过登录 shell 来控制远程主机。SSH 代表安全外壳。可以在通过登录到远程主机收到的 shell 上执行命令，就好像我们在本地主机上运行命令一样。它通过加密隧道运行网络数据传输。本教程将介绍在远程主机上执行命令的不同方法。

## 准备工作

SSH 不会默认随所有 GNU/Linux 发行版一起提供。因此，您可能需要使用软件包管理器安装`openssh-server`和`openssh-client`软件包。SSH 服务默认在端口号 22 上运行。

## 如何做...

要连接到运行 SSH 服务器的远程主机，请使用：

```
$ ssh username@remote_host

```

在这个命令中：

+   `username`是存在于远程主机上的用户。

+   `remote_host`可以是域名或 IP 地址。

例如：

```
$ ssh mec@192.168.0.1
The authenticity of host '192.168.0.1 (192.168.0.1)' can't be established.
RSA key fingerprint is 2b:b4:90:79:49:0a:f1:b3:8a:db:9f:73:2d:75:d6:f9.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.0.1' (RSA) to the list of known hosts.
Password:

Last login: Fri Sep  3 05:15:21 2010 from 192.168.0.82
mec@proxy-1:~$

```

它将交互式地要求用户密码，并在成功验证后将返回用户的 shell。

默认情况下，SSH 服务器在端口 22 上运行。但是某些服务器在不同端口上运行 SSH 服务。在这种情况下，使用`ssh`命令的`-p port_no`来指定端口。

为了连接到运行在端口 422 上的 SSH 服务器，请使用：

```
$ ssh user@locahost -p 422

```

您可以在对应于远程主机的 shell 中执行命令。Shell 是一个交互式工具，用户可以在其中输入和运行命令。但是，在 shell 脚本上下文中，我们不需要交互式 shell。我们需要自动化多个任务。我们需要在远程 shell 上执行多个命令，并在本地主机上显示或存储其输出。在自动化脚本中每次输入密码都不切实际，因此应配置 SSH 的自动登录。

*使用 SSH 实现无密码自动登录*中解释了 SSH 命令。

在运行使用 SSH 的自动化脚本之前，请确保已配置自动登录。

要在远程主机上运行命令并在本地主机 shell 上显示其输出，请使用以下语法：

```
$ ssh user@host 'COMMANDS'

```

例如：

```
$ ssh mec@192.168.0.1 'whoami'
Password: 
mec

```

可以通过在命令之间使用分号分隔符来给出多个命令，如下所示：

```
$ ssh user@host 'command1 ; command2 ; command3'

```

命令可以通过`stdin`发送，并且命令的输出将可用于`stdout`。

语法如下：

```
$ ssh user@remote_host  "COMMANDS" > stdout.txt 2> errors.txt

```

`COMMANDS`字符串应该用引号引起来，以防止分号字符在本地主机 shell 中充当分隔符。我们还可以通过`stdin`传递任何涉及管道语句的命令序列到 SSH 命令，如下所示：

```
$ echo  "COMMANDS" | sshuser@remote_host> stdout.txt 2> errors.txt

```

例如：

```
$ ssh mec@192.168.0.1  "echo user: $(whoami);echo OS: $(uname)"
Password: 
user: slynux 
OS: Linux 

```

在此示例中，在远程主机上执行的命令是：

```
echo user: $(whoami);
echo OS: $(uname)

```

它可以概括为：

```
COMMANDS="command1; command2; command3"
$ ssh user@hostname  "$COMMANDS"

```

我们还可以通过使用`( )`子 shell 运算符在命令序列中传递更复杂的子 shell。

让我们编写一个基于 SSH 的 shell 脚本，用于收集一组远程主机的正常运行时间。正常运行时间是系统已经运行的时间。`uptime`命令用于显示系统已经运行多长时间。

假设`IP_LIST`中的所有系统都有一个共同的用户`test`。

```
#!/bin/bash
#Filename: uptime.sh
#Description: Uptime monitor

IP_LIST="192.168.0.1 192.168.0.5 192.168.0.9"
USER="test"

for IP in $IP_LIST;
do
 utime=$(ssh $USER@$IP uptime  | awk '{ print $3 }' )
 echo $IP uptime:  $utime
done

```

预期输出是：

```
$ ./uptime.sh
192.168.0.1 uptime: 1:50,
192.168.0.5 uptime: 2:15,
192.168.0.9 uptime: 10:15,

```

## 还有更多...

`ssh`命令可以使用多个附加选项执行。让我们逐一介绍它们。

### 使用压缩的 SSH

SSH 协议还支持使用压缩进行数据传输，当带宽成为问题时非常有用。使用`ssh`命令的`-C`选项启用压缩，如下所示：

```
$ ssh -C user@hostname COMMANDS

```

### 将数据重定向到远程主机 shell 命令的 stdin

有时我们需要将一些数据重定向到远程 shell 命令的`stdin`中。让我们看看如何做到这一点。一个例子如下：

```
$ echo "text" | ssh user@remote_host 'cat >> list'

```

或：

```
# Redirect data from file as:
$ ssh user@remote_host 'cat >> list'  < file

```

`cat >> list`将通过`stdin`接收的数据附加到文件列表中。此命令在远程主机上执行。但是数据是从本地主机传递到`stdin`。

## 另请参阅

+   *使用 SSH 实现无密码自动登录*，解释了如何配置自动登录以执行命令而无需提示输入密码。

# 在本地挂载点挂载远程驱动器

在进行读写数据传输操作时，拥有一个本地挂载点以访问远程主机文件系统将非常有帮助。SSH 是网络中最常见的传输协议，因此我们可以利用它与`sshfs`一起使用。`sshfs`使您能够将远程文件系统挂载到本地挂载点。让我们看看如何做到这一点。

## 准备工作

`sshfs`在 GNU/Linux 发行版中默认未安装。使用软件包管理器安装`sshfs`。`sshfs`是 fuse 文件系统包的扩展，允许支持的操作系统将各种数据挂载为本地文件系统。

## 如何做...

为了将远程主机上的文件系统位置挂载到本地挂载点，使用：

```
# sshfs user@remotehost:/home/path /mnt/mountpoint
Password:

```

在提示时输入用户密码。

现在，远程主机上`/home/path`的数据可以通过本地挂载点`/mnt/mountpoint`访问。

在完成工作后卸载，使用：

```
# umount /mnt/mountpoint

```

## 参见

+   *使用 SSH 在远程主机上运行命令*，解释了 ssh 命令。

# 在网络上多播窗口消息

网络管理员经常需要向网络上的节点发送消息。在用户的桌面上显示弹出窗口将有助于提醒用户获得信息。使用 shell 脚本和 GUI 工具包可以实现此任务。本教程讨论了如何向远程主机发送自定义消息的弹出窗口。

## 准备工作

要实现 GUI 弹出窗口，可以使用 zenity。Zenity 是一个可脚本化的 GUI 工具包，用于创建包含文本框、输入框等的窗口。SSH 可用于连接到远程主机上的远程 shell。Zenity 在 GNU/Linux 发行版中默认未安装。使用软件包管理器安装 zenity。

## 如何做...

Zenity 是可脚本化的对话框创建工具包之一。还有其他工具包，如 gdialog、kdialog、xdialog 等。Zenity 似乎是一个灵活的工具包，符合 GNOME 桌面环境。

为了使用 zenity 创建信息框，使用：

```
$ zenity --info --text "This is a message"
# It will display a window with "This is a message" as text.

```

Zenity 可以用来创建带有输入框、组合输入、单选按钮、按钮等的窗口。这些不在本教程的范围内。查看 zenity 的 man 页面以获取更多信息。

现在，我们可以使用 SSH 在远程机器上运行这些 zenity 语句。为了在远程主机上通过 SSH 运行此语句，运行：

```
$ ssh user@remotehost 'zenity --info --text "This is a message"'

```

但是这将返回一个错误，如下：

```
(zenity:3641): Gtk-WARNING **: cannot open display: 

```

这是因为 zenity 依赖于 Xserver。Xsever 是一个负责在屏幕上绘制图形元素的守护进程，包括 GUI。裸的 GNU/Linux 系统只包含文本终端或 shell 提示符。

Xserver 使用一个特殊的环境变量`DISPLAY`来跟踪正在系统上运行的 Xserver 实例。

我们可以手动设置`DISPLAY=:0`来指示 Xserver 关于 Xserver 实例。

上一个 SSH 命令可以重写为：

```
$ ssh username@remotehost 'export DISPLAY=:0 ; zenity --info --text "This is a message"'

```

如果具有用户名的用户已登录到任何窗口管理器中，此语句将在`remotehost`上显示一个弹出窗口。

为了将弹出窗口多播到多个远程主机，编写一个 shell 脚本如下：

```
#!/bin/bash
#Filename: multi_cast_window.sh
# Description: Multi-cast window popups

IP_LIST="192.168.0.5 192.168.0.3 192.168.0.23"
USER="username"

COMMAND='export DISPLAY=:0 ;zenity --info --text "This is a message" '
for host in $IP_LIST;
do
  ssh $USER@$host  "$COMMAND" &
done
```

## 它是如何工作的...

在上面的脚本中，我们有一个 IP 地址列表，应该在这些 IP 地址上弹出窗口。使用循环来遍历 IP 地址并执行 SSH 命令。

在 SSH 语句中，最后我们使用了后缀`&`。`&`将 SSH 语句发送到后台。这样做是为了方便执行多个 SSH 语句的并行化。如果不使用`&`，它将启动 SSH 会话，执行 zenity 对话框，并等待用户关闭弹出窗口。除非远程主机上的用户关闭窗口，否则循环中的下一个 SSH 语句将不会被执行。为了避免循环被阻塞，等待 SSH 会话终止，使用了`&`技巧。

## 参见

+   *使用 SSH 在远程主机上运行命令*，解释了 ssh 命令。

# 网络流量和端口分析

网络端口是网络应用程序的基本参数。应用程序在主机上打开端口，并通过远程主机上打开的端口与远程主机通信。了解打开和关闭的端口对于安全上下文至关重要。恶意软件和 root 工具包可能在具有自定义端口和自定义服务的系统上运行，这允许攻击者捕获对数据和资源的未经授权访问。通过获取打开的端口和运行在端口上的服务列表，我们可以分析和保护系统免受 root 工具包的控制，并且该列表有助于有效地将它们移除。打开端口的列表不仅有助于恶意软件检测，还有助于收集有关系统上打开端口的信息，从而能够调试基于网络的应用程序。它有助于分析特定端口连接和端口监听功能是否正常工作。本教程讨论了用于端口分析的各种实用程序。

## 准备就绪

有多种命令可用于监听每个端口上运行的端口和服务（例如，`lsof`和`netstat`）。这些命令默认情况下在所有 GNU/Linux 发行版上都可用。

## 如何做...

要列出系统上所有打开的端口以及每个附加到它的服务的详细信息，请使用：

```
$ lsof -i
COMMAND    PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
firefox-b 2261 slynux   78u  IPv4  63729      0t0  TCP localhost:47797->localhost:42486 (ESTABLISHED)
firefox-b 2261 slynux   80u  IPv4  68270      0t0  TCP slynux-laptop.local:41204->192.168.0.2:3128 (CLOSE_WAIT)
firefox-b 2261 slynux   82u  IPv4  68195      0t0  TCP slynux-laptop.local:41197->192.168.0.2:3128 (ESTABLISHED)
ssh       3570 slynux    3u  IPv6  30025      0t0  TCP localhost:39263->localhost:ssh (ESTABLISHED)
ssh       3836 slynux    3u  IPv4  43431      0t0  TCP slynux-laptop.local:40414->boneym.mtveurope.org:422 (ESTABLISHED)
GoogleTal 4022 slynux   12u  IPv4  55370      0t0  TCP localhost:42486 (LISTEN)
GoogleTal 4022 slynux   13u  IPv4  55379      0t0  TCP localhost:42486->localhost:32955 (ESTABLISHED)

```

`lsof`的输出中的每个条目对应于打开端口进行通信的每个服务。输出的最后一列包含类似于以下行：

```
slynux-laptop.local:34395->192.168.0.2:3128 (ESTABLISHED)
```

在此输出中，`slynux-laptop.local:34395`对应于本地主机部分，`192.168.0.2:3128`对应于远程主机。

`34395`是从当前机器打开的端口，`3128`是服务连接到远程主机的端口。

要列出当前机器上打开的端口，请使用：

```
$ lsof -i | grep ":[0-9]\+->" -o | grep "[0-9]\+" -o  | sort | uniq

```

`:[0-9]\+->`正则表达式用于从`lsof`输出中提取主机端口部分（`:34395->`）。接下来的`grep`用于提取端口号（即数字）。通过同一端口可能发生多个连接，因此同一端口的多个条目可能会发生。为了仅显示每个端口一次，它们被排序并打印唯一的端口。

## 还有更多...

让我们通过其他实用程序来查看打开的端口和与网络流量相关的信息。

### 使用 netstat 列出打开的端口和服务

`netstat` 是用于网络服务分析的另一个命令。解释`netstat`的所有功能不在本教程的范围内。我们现在将看看如何列出服务和端口号。

使用`netstat -tnp`列出打开的端口和服务如下：

```
$ netstat -tnp
(Not all processes could be identified, non-owned process info 
will not be shown, you would have to be root to see it all.) 
Active Internet connections (w/o servers) 
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name 
tcp        0      0 192.168.0.82:38163      192.168.0.2:3128        ESTABLISHED 2261/firefox-bin 
tcp        0      0 192.168.0.82:38164      192.168.0.2:3128        TIME_WAIT   - 
tcp        0      0 192.168.0.82:40414      193.107.206.24:422      ESTABLISHED 3836/ssh 
tcp        0      0 127.0.0.1:42486         127.0.0.1:32955         ESTABLISHED 4022/GoogleTalkPlug 
tcp        0      0 192.168.0.82:38152      192.168.0.2:3128        ESTABLISHED 2261/firefox-bin 
tcp6       0      0 ::1:22                  ::1:39263               ESTABLISHED - 
tcp6       0      0 ::1:39263               ::1:22                  ESTABLISHED 3570/ssh 

```
