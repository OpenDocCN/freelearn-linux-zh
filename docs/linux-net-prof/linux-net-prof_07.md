# 第五章：具有实际示例的 Linux 安全标准

在本章中，我们将探讨为什么 Linux 主机，就像任何主机一样，在初始安装后需要一些关怀 - 事实上，在它们的整个生命周期中 - 以进一步加固它们。在此过程中，我们将涵盖各种主题，以建立加固 Linux 主机的最终“大局”。

本章将涵盖以下主题：

+   为什么我需要保护我的 Linux 主机？

+   云特定的安全考虑

+   常见的行业特定安全标准

+   互联网安全中心的关键控制

+   互联网安全中心基准

+   SELinux 和 AppArmor

# 技术要求

在本章中，我们将涵盖许多主题，但技术上的重点将放在加固 SSH 服务上，使用我们当前的 Linux 主机或虚拟机。与上一章一样，您可能会发现在进行更改时测试您的更改是否有用，但这并不是必须的。

# 为什么我需要保护我的 Linux 主机？

就像几乎所有其他操作系统一样，Linux 安装经过简化，使安装过程尽可能简单，尽可能少地出现问题。正如我们在前几章中看到的，这通常意味着没有启用防火墙的安装。此外，操作系统版本和软件包版本当然会与安装媒体匹配，而不是与每个软件包的最新版本匹配。在本章中，我们将讨论 Linux 中的默认设置通常不是大多数人认为安全的设置，作为行业，我们如何通过立法、法规和建议来解决这个问题。

至于初始安装是否过时，幸运的是，大多数 Linux 发行版都启用了自动更新过程。这由`/etc/apt/apt.conf.d/20auto-upgrades`文件中的两行控制：

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

这两个设置默认都设置为`1`（启用）。这两行都很容易理解 - 第一行控制包列表是否更新，第二行控制自动更新的开启或关闭。对于可能处于“巡航控制”管理方法的桌面或服务器来说，这个默认设置并不是一个坏设置。但需要注意的是，`Unattended-Upgrade`行只启用安全更新。

在大多数良好管理的环境中，您会期望看到安排的维护窗口，先在不太关键的服务器上进行升级和测试，然后再部署到更重要的主机上。在这些情况下，您将希望将自动更新设置为`0`，并使用手动或脚本化的更新过程。对于 Ubuntu，手动更新过程涉及两个命令，按以下顺序执行：

![](img/B16336_05_Table_01.jpg)

这些可以合并成一行（请参见下一行代码），但在升级步骤中您将需要回答一些“是/否”提示 - 首先是批准整个过程和数据量。此外，如果您的任何软件包在版本之间更改了默认行为，您将被要求做出决定：

```
# sudo apt-get update && sudo apt-get upgrade
```

`&&`运算符按顺序执行命令。第二个命令只有在第一个成功完成（返回代码为零）时才执行。

但等等，您可能会说，我的一些主机在云中 - 那它们呢？在下一节中，您将发现无论您在何处安装，Linux 都是 Linux，而且在某些情况下，您的云实例可能比您的“数据中心”服务器模板更不安全。无论您的操作系统是什么，或者您在哪里部署，更新都将是您安全程序的关键部分。

# 云特定的安全考虑

如果您在任何主要云中启动虚拟机并使用它们的默认镜像，从安全的角度来看，有一些事情需要考虑：

+   有些云启用了自动更新；有些没有。然而，每个操作系统的每个镜像总是有些过时。在您启动虚拟机之后，您将需要更新它，就像您会更新独立主机一样。

+   大多数云服务镜像也有主机防火墙，在某些限制模式下启用。对于您来说，这两个防火墙问题意味着，当您首次启动新的 Linux 虚拟机时，不要指望能够“ping”它，直到您查看了主机防火墙配置（请记住上一章 - 一定要检查`iptables`和`nftables`）。

+   许多云服务镜像默认情况下允许直接从公共互联网进行管理访问。在 Linux 的情况下，这意味着通过`tcp/22`进行 SSH。虽然这种访问的默认设置不像各种云服务提供商早期那样常见，但检查您是否对整个互联网开放了 SSH（`tcp/22`）仍然是明智的。

+   通常，您可能正在使用云“服务”，而不是实际的服务器实例。例如，无服务器数据库实例很常见，您可以完全访问和控制您的数据库，但承载它的服务器对您的用户或应用程序不可见。潜在的服务器可能专用于您的实例，但更有可能是跨多个组织共享的。

现在我们已经讨论了本地部署和云 Linux 部署之间的一些差异，让我们讨论一下不同行业之间的安全要求的差异。

# 常见的行业特定安全标准

有许多行业特定的指导和监管要求，其中一些即使您不在该行业，您可能也熟悉。由于它们是行业特定的，我们将对每个进行高层描述 - 如果其中任何一个适用于您，您将知道每个都值得一本书（或几本书）来描述。

![](img/B16336_05_Table_02a.jpg)![](img/B16336_05_Table_02b.jpg)

尽管这些标准和监管或法律要求各自具有行业特定的重点，但许多基本建议和要求都非常相似。当没有提供良好安全指导的一套法规时，**互联网安全中心**（**CIS**）的“关键控制”通常被使用。事实上，这些控制通常与监管要求一起使用，以提供更好的整体安全姿态。

# 互联网安全中心关键控制

虽然 CIS 的关键控制并不是合规标准，但它们无疑是任何组织的一个很好的基础和一个良好的工作模型。关键控制非常实用 - 它们不是以合规性为驱动，而是专注于现实世界的攻击和对抗。理解是，如果您专注于这些控制，特别是按顺序专注于它们，那么您的组织将很好地抵御“野外”中看到的更常见的攻击。例如，仅仅通过查看顺序，就可以明显地看出，您无法保护您的主机（**＃3**），除非您知道您网络上有哪些主机（**＃1**）。同样，日志记录（**＃8**）没有主机和应用程序清单（**＃2**和**＃3**）是无效的。当组织按照列表逐步进行工作时，它很快就会达到不成为“群体中最慢的羚羊”的目标。

与 CIS 基准一样，关键控制由志愿者编写和维护。它们也会随着时间进行修订 - 这是关键的，因为世界随着时间、操作系统和攻击的不断发展而发生变化。虽然 10 年前的威胁仍然存在，但现在我们有新的威胁，我们有新的工具，恶意软件和攻击者使用的方法与 10 年前不同。本节描述了关键控制的第 8 版（于 2021 年发布），但如果您在组织中使用这些控制做决策，请务必参考最新版本。

关键控制（第 8 版）分为三个实施组：

**实施组 1（IG1）-基本控制**

这些控制是组织通常开始的地方。如果这些都已经就位，那么您可以确保您的组织不再是“群中最慢的羚羊”。这些控制针对较小的 IT 团队和商业/现成的硬件和软件。

**实施组 2（IG2）- 中型企业**

实施组 2 的控制扩展了 IG1 的控制，为更具体的配置和技术流程提供了技术指导。这组控制针对较大的组织，在那里有一个人或一小组负责信息安全，或者有法规合规要求。

**实施组 3（IG3）- 大型企业**

这些控制针对已建立的安全团队和流程的较大环境。这些控制中的许多与组织有关 - 与员工和供应商合作，并为事件响应、事件管理、渗透测试和红队演习制定政策和程序。

每个实施组都是前一个的超集，因此 IG3 包括组 1 和 2。每个控制都有几个子部分，正是这些子部分由实施组分类。有关每个控制和实施组的完整描述，关键控制的源文件可免费下载[`www.cisecurity.org/controls/cis-controls-list/`](https://www.cisecurity.org/controls/cis-controls-list/)，其中包括点击以获取描述以及详细的 PDF 和 Excel 文档。

![](img/B16336_05_Table_03a.jpg)![](img/B16336_05_Table_03b.jpg)![](img/B16336_05_Table_03c.jpg)![](img/B16336_05_Table_03d.jpg)

现在我们已经讨论了关键控制措施，那么如何将其转化为保护 Linux 主机或您组织中可能看到的基于 Linux 的基础架构呢？让我们看一些具体的例子，从关键控制措施 1 和 2（硬件和软件清单）开始。

## 开始 CIS 关键安全控制 1 和 2

对网络上的主机和每台主机上运行的软件的准确清单几乎是每个安全框架的关键部分 - 思想是如果您不知道它在那里，就无法保护它。

让我们探讨如何在我们的 Linux 主机上使用零预算方法来实施关键控制 1 和 2。

### 关键控制 1 - 硬件清单

让我们使用本机 Linux 命令来探索关键控制 1 和 2 - 硬件和软件清单。

硬件清单很容易获得 - 许多系统参数都可以作为文件轻松获得，位于`/proc`目录中。`proc`文件系统是虚拟的。`/proc`中的文件不是真实的文件；它们反映了机器的操作特性。例如，您可以通过查看正确的文件来获取 CPU（此输出仅显示第一个 CPU）：

```
$ cat /proc/cpuinfo
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
model           : 158
model name      : Intel(R) Xeon(R) CPU E3-1505M v6 @ 3.00GHz
stepping        : 9
microcode       : 0xde
cpu MHz         : 3000.003
cache size      : 8192 KB
physical id     : 0
siblings        : 1
core id         : 0
cpu cores       : 1
…
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon nopl xtopology tsc_reliable nonstop_tsc cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch cpuid_fault invpcid_single pti ssbd ibrs ibpb stibp fsgsbase tsc_adjust bmi1 avx2 smep bmi2 invpcid rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 xsaves arat md_clear flush_l1d arch_capabilities
bugs            : cpu_meltdown spectre_v1 spectre_v2 spec_store_bypass l1tf mds swapgs itlb_multihit srbds
bogomips        : 6000.00
…
```

内存信息也很容易找到：

```
$ cat /proc/meminfo
MemTotal:        8025108 kB
MemFree:         4252804 kB
MemAvailable:    6008020 kB
Buffers:          235416 kB
Cached:          1486592 kB
SwapCached:            0 kB
Active:          2021224 kB
Inactive:         757356 kB
Active(anon):    1058024 kB
Inactive(anon):     2240 kB
Active(file):     963200 kB
Inactive(file):   755116 kB
…
```

深入挖掘`/proc`文件系统，我们可以在`/proc/sys/net/ipv4`中的许多单独的文件中找到各种 IP 或 TCP 参数的设置（此列表已完整并格式化以便更轻松地查看）。

硬件方面，有多种方法可以获取操作系统版本：

```
$ cat /proc/version
Linux version 5.8.0-38-generic (buildd@lgw01-amd64-060) (gcc (Ubuntu 9.3.0-17ubuntu1~20.04) 9.3.0, GNU ld (GNU Binutils for Ubuntu) 2.34) #43~20.04.1-Ubuntu SMP Tue Jan 12 16:39:47 UTC 2021
$ cat /etc/issue
Ubuntu 20.04.1 LTS \n \l
$ uname -v
#43~20.04.1-Ubuntu SMP Tue Jan 12 16:39:47 UTC 2021
```

大多数组织选择将操作系统信息放入硬件清单中，尽管将其放入该机器的软件清单中同样正确。在几乎每个操作系统中，安装的应用程序将以比操作系统更频繁的频率更新，这就是为什么硬件清单如此频繁地被选择的原因。重要的是它记录在一个清单中。在大多数系统中，硬件和软件清单系统是同一个系统，所以这样的讨论就很好地解决了。

`lshw`命令是一个很好的“给我所有东西”的命令，用于硬件清单——`lshw`的 man 页面为我们提供了更深入挖掘或更有选择性地显示此命令的附加选项。不过，这个命令可能收集太多信息——你需要有选择地进行收集！

组织通常通过编写脚本来找到一个很好的折衷方案，以收集他们的硬件清单所需的确切信息——例如，下面的简短脚本对基本的硬件和操作系统清单非常有用。它使用了我们迄今为止使用过的几个文件和命令，并通过使用一些新命令进行了扩展：

+   `fdisk`用于磁盘信息

+   `dmesg`和`dmidecode`用于系统信息：

```
echo -n "Basic Inventory for Hostname: "
uname -n
#
echo =====================================
dmidecode | sed -n '/System Information/,+2p' | sed 's/\x09//'
dmesg | grep Hypervisor
dmidecode | grep "Serial Number" | grep -v "Not Specified" | grep -v None
#
echo =====================================
echo "OS Information:"
uname -o -r
if [ -f /etc/redhat-release ]; then
    echo -n "  "
    cat /etc/redhat-release
fi
if [ -f /etc/issue ]; then
    cat /etc/issue
fi
#
echo =====================================
echo "IP information: "
ip ad | grep inet | grep -v "127.0.0.1" | grep -v "::1/128" | tr -s " " | cut -d " " -f 3
# use this line if legacy linux
# ifconfig | grep "inet" | grep -v "127.0.0.1" | grep -v "::1/128" | tr -s " " | cut -d " " -f 3
#
echo =====================================
echo "CPU Information: "
cat /proc/cpuinfo | grep "model name\|MH\|vendor_id" | sort -r | uniq
echo -n "Socket Count: "
cat /proc/cpuinfo | grep processor | wc -l
echo -n "Core Count (Total): "
cat /proc/cpuinfo | grep cores | cut -d ":" -f 2 | awk '{ sum+=$1} END {print sum}'
#
echo =====================================
echo "Memory Information: "
grep MemTotal /proc/meminfo | awk '{print $2,$3}'
#
echo =====================================
echo "Disk Information: "
fdisk -l | grep Disk | grep dev
```

你的实验 Ubuntu 虚拟机的输出可能如下所示（此示例是虚拟机）。请注意，我们使用了`sudo`（主要是为了`fdisk`命令，它需要这些权限）：

```
$ sudo ./hwinven.sh
Basic Inventory for Hostname: ubuntu
=====================================
System Information
Manufacturer: VMware, Inc.
Product Name: VMware Virtual Platform
[    0.000000] Hypervisor detected: VMware
        Serial Number: VMware-56 4d 5c ce 85 8f b5 52-65 40 f0 92 02 33 2d 05
=====================================
OS Information:
5.8.0-45-generic GNU/Linux
Ubuntu 20.04.2 LTS \n \l
=====================================
IP information:
192.168.122.113/24
fe80::1ed6:5b7f:5106:1509/64
=====================================
CPU Information:
vendor_id       : GenuineIntel
model name      : Intel(R) Xeon(R) CPU E3-1505M v6 @ 3.00GHz
cpu MHz         : 3000.003
Socket Count: 2
Core Count (Total): 2
=====================================
Memory Information:
8025036 kB
=====================================
Disk Information:
Disk /dev/loop0: 65.1 MiB, 68259840 bytes, 133320 sectors
Disk /dev/loop1: 55.48 MiB, 58159104 bytes, 113592 sectors
Disk /dev/loop2: 218.102 MiB, 229629952 bytes, 448496 sectors
Disk /dev/loop3: 217.92 MiB, 228478976 bytes, 446248 sectors
Disk /dev/loop5: 64.79 MiB, 67915776 bytes, 132648 sectors
Disk /dev/loop6: 55.46 MiB, 58142720 bytes, 113560 sectors
Disk /dev/loop7: 51.2 MiB, 53501952 bytes, 104496 sectors
Disk /dev/fd0: 1.42 MiB, 1474560 bytes, 2880 sectors
Disk /dev/sda: 40 GiB, 42949672960 bytes, 83886080 sectors
Disk /dev/loop8: 32.28 MiB, 33845248 bytes, 66104 sectors
Disk /dev/loop9: 51.4 MiB, 53522432 bytes, 104536 sectors
Disk /dev/loop10: 32.28 MiB, 33841152 bytes, 66096 sectors
Disk /dev/loop11: 32.28 MiB, 33841152 bytes, 66096 sectors
```

有了填充硬件清单所需的信息，接下来让我们看看我们的软件清单。

### 关键控制 2——软件清单

要清点所有已安装的软件包，可以使用`apt`或`dpkg`命令。我们将使用这个命令来获取已安装软件包的列表：

```
$ sudo apt list --installed | wc -l
WARNING: apt does not have a stable CLI interface. Use with caution in scripts.
1735
```

请注意，有这么多软件包时，最好要么知道自己在寻找什么并提出具体的请求（也许使用`grep`命令），要么收集多台主机的所有信息，然后使用数据库找出在某一方面不匹配的主机。

`dpkg`命令将为我们提供类似的信息：

```
dpkg -
Name                                 Version                 Description
====================================================================================
acpi-support               0.136.1                scripts for handling many ACPI events
acpid                      1.0.10-5ubuntu2.1      Advanced Configuration and Power Interfacee
adduser                    3.112ubuntu1           add and remove users and groups
adium-theme-ubuntu         0.1-0ubuntu1           Adium message style for Ubuntu
adobe-flash-properties-gtk 10.3.183.10-0lucid1    GTK+ control panel for Adobe Flash Player pl
.... and so on ....
```

要获取软件包中包含的文件，使用以下命令：

```
robv@ubuntu:~$ dpkg -L openssh-client
/.
/etc
/etc/ssh
/etc/ssh/ssh_config
/etc/ssh/ssh_config.d
/usr
/usr/bin
/usr/bin/scp
/usr/bin/sftp
/usr/bin/ssh
/usr/bin/ssh-add
/usr/bin/ssh-agent
….
```

要列出大多数 Red Hat 发行版中安装的所有软件包，请使用以下命令：

```
$ rpm -qa
libsepol-devel-2.0.41-3.fc13.i686
wpa_supplicant-0.6.8-9.fc13.i686
system-config-keyboard-1.3.1-1.fc12.i686
libbeagle-0.3.9-5.fc12.i686
m17n-db-kannada-1.5.5-4.fc13.noarch
pptp-1.7.2-9.fc13.i686
PackageKit-gtk-module-0.6.6-2.fc13.i686
gsm-1.0.13-2.fc12.i686
perl-ExtUtils-ParseXS-2.20-121.fc13.i686
... (and so on)
```

要获取有关特定软件包的更多信息，使用`rpm -qi`：

```
$ rpm -qi python
Name        : python                       Relocations: (not relocatable)
Version     : 2.6.4                             Vendor: Fedora Project
Release     : 27.fc13                       Build Date: Fri 04 Jun 2010 02:22:55 PM EDT
Install Date: Sat 19 Mar 2011 08:21:36 PM EDT      Build Host: x86-02.phx2.fedoraproject.org
Group       : Development/Languages         Source RPM: python-2.6.4-27.fc13.src.rpm
Size        : 21238314                         License: Python
Signature   : RSA/SHA256, Fri 04 Jun 2010 02:36:33 PM EDT, Key ID 7edc6ad6e8e40fde
Packager    : Fedora Project
URL         : http://www.python.org/
Summary     : An interpreted, interactive, object-oriented programming language
Description :
Python is an interpreted, interactive, object-oriented programming
....
(and so on)
```

要获取所有软件包的更多信息（可能是太多信息），使用`rpm -qia`。

这些清单，正如你所看到的，非常细致和完整。你可以选择清点所有东西——即使是完整的文本清单（没有数据库）也是有价值的。如果你有两台相似的主机，你可以使用`diff`命令来查看两台相似工作站之间的差异（一台工作，一台不工作）。

或者，如果你在进行故障排除，通常会检查已安装的版本与已知的错误、文件日期与已知的安装日期等是否匹配。

迄今为止讨论的清单方法都是 Linux 本地的，但并不适合管理一大批主机，甚至不适合很好地管理一台主机。让我们来探索 OSQuery，这是一个管理软件包，可以简化在许多关键控制和/或任何你可能需要遵守的监管框架上取得进展。

## OSQuery——关键控制 1 和 2，添加控制 10 和 17

与维护成千上万行文本文件作为清单不同，更常见的方法是使用实际的应用程序或平台来维护你的清单——可以是在主机上实时进行，可以是在数据库中，也可以是两者的结合。OSQuery 是一个常见的平台。它为管理员提供了一个类似数据库的接口，用于目标主机上的实时信息。

OSQuery 是一个常见的选择，因为它可以处理最流行的 Linux 和 Unix 变体、macOS 和 Windows，都在一个界面中。让我们深入了解这个流行平台的 Linux 部分。

首先，要安装 OSQuery，我们需要添加正确的存储库。对于 Ubuntu，使用以下命令：

```
$ echo "deb [arch=amd64] https://pkg.osquery.io/deb deb main" | sudo tee /etc/apt/sources.list.d/osquery.list
```

接下来，导入存储库的签名密钥：

```
$ sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 1484120AC4E9F8A1A577AEEE97A80C63C9D8B80B
```

然后，更新软件包列表：

```
$ sudo apt update
```

最后，我们可以安装`osquery`：

```
$ sudo apt-get install osquery
```

OSQuery 有三个主要组件：

![](img/B16336_05_Table_04.jpg)

安装完成后，让我们来探索交互式 shell。请注意，如果没有设置守护程序并“连接”你的各种主机，我们使用的是一个虚拟数据库，只查看我们的本地主机：

```
robv@ubuntu:~$ osqueryi
Using a virtual database. Need help, type '.help'
osquery> .help
Welcome to the osquery shell. Please explore your OS!
You are connected to a transient 'in-memory' virtual database.
.all [TABLE]     Select all from a table
.bail ON|OFF     Stop after hitting an error
.echo ON|OFF     Turn command echo on or off
.exit            this program
.features        List osquery's features and their statuses
.headers ON|OFF  Turn display of headers on or off
.help            Show this message
….
```

接下来让我们来看看我们可以使用的数据库表：

```
osquery> .tables
  => acpi_tables
  => apparmor_events
  => apparmor_profiles
  => apt_sources
  => arp_cache
  => atom_packages
  => augeas
  => authorized_keys
  => azure_instance_metadata
  => azure_instance_tags
  => block_devices
  => bpf_process_events
  => bpf_socket_events
….
```

有数十个表格来跟踪各种系统参数。让我们来看看操作系统版本，例如：

```
osquery> select * from os_version;
+--------+---------------------------+-------+-------+-------+-------+----------+---------------+----------+--------+
| name   | version                   | major | minor | patch | build | platform | platform_like | codename | arch   |
+--------+---------------------------+-------+-------+-------+-------+----------+---------------+----------+--------+
| Ubuntu | 20.04.1 LTS (Focal Fossa) | 20    | 4     | 0     |       | ubuntu   | debian        | focal    | x86_64 |
```

或者，要收集本地接口 IP 和子网掩码，不包括环回接口，使用以下命令：

```
osquery> select interface,address,mask from interface_addresses where interface NOT LIKE '%lo%';
+-----------+---------------------------------+-----------------------+
| interface | address                         | mask                  |
+-----------+---------------------------------+-----------------------+
| ens33     | 192.168.122.170                 | 255.255.255.0         |
| ens33     | fe80::1ed6:5b7f:5106:1509%ens33 | ffff:ffff:ffff:ffff:: |
+-----------+---------------------------------+-----------------------+
```

或者，要检索本地 ARP 缓存，请使用以下命令：

```
osquery> select * from arp_cache;
+-----------------+-------------------+-----------+-----------+
| address         | mac               | interface | permanent |
+-----------------+-------------------+-----------+-----------+
| 192.168.122.201 | 3c:52:82:15:52:1b | ens33     | 0         |
| 192.168.122.1   | 00:0c:29:3b:73:cb | ens33     | 0         |
| 192.168.122.241 | 40:b0:34:72:48:e4 | ens33     | 0         |
```

或者，列出已安装的软件包（请注意，此输出限制为 2）：

```
osquery> select * from deb_packages limit 2;
+-----------------+--------------------------+--------+------+-------+-------------------+----------------------+-----------------------------------------------------------+---------+----------+
| name            | version                  | source | size | arch  | revision          | status               | maintainer                                                | section | priority |
+-----------------+--------------------------+--------+------+-------+-------------------+----------------------+-----------------------------------------------------------+---------+----------+
| accountsservice | 0.6.55-0ubuntu12~20.04.4 |        | 452  | amd64 | 0ubuntu12~20.04.4 | install ok installed | Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com> | admin   | optional |
| acl             | 2.2.53-6                 |        | 192  | amd64 | 6                 | install ok installed | Ubuntu Developers <ubuntu-devel-discuss@lists.ubuntu.com> | utils   | optional |
+-----------------+--------------------------+--------+------+-------+-------------------+----------------------+-----------------------------------------------------------+---------+----------+
```

您还可以查询运行中的进程（显示限制为 10）：

```
osquery> SELECT pid, name FROM processes order by start_time desc limit 10;
+-------+----------------------------+
| pid   | name                       |
+-------+----------------------------+
| 34790 | osqueryi                   |
| 34688 | sshd                       |
| 34689 | bash                       |
| 34609 | sshd                       |
| 34596 | systemd-resolve            |
| 34565 | dhclient                   |
| 34561 | kworker/0:3-cgroup_destroy |
| 34562 | kworker/1:3-events         |
| 34493 | kworker/0:0-events         |
| 34494 | kworker/1:2-events         |
+-------+----------------------------+
```

我们可以向我们的进程列表添加额外的信息。让我们为每个进程添加`SHA256`哈希值。哈希是一种可以唯一标识数据的数学函数。例如，如果您有两个具有不同名称但相同哈希的文件，它们很可能是相同的。虽然总会有一小部分可能性会出现哈希“碰撞”（两个非相同文件的相同哈希），但使用不同算法再次对它们进行哈希处理可以消除任何不确定性。在取证中广泛使用哈希数据工件-特别是在收集证据以证明责任链完整性方面。

即使在取证分析中，单个哈希值通常足以确定唯一性（或不确定性）。

这对运行进程意味着什么？如果您的恶意软件在每个实例中使用随机名称以规避检测，那么在所有 Linux 主机的 RAM 中对进程进行哈希处理，可以让您找到在不同主机上以不同名称运行的相同进程：

```
osquery> SELECT DISTINCT h.sha256, p.name, u.username
    ...> FROM processes AS p
    ...> INNER JOIN hash AS h ON h.path = p.path
    ...> INNER JOIN users AS u ON u.uid = p.uid
    ...> ORDER BY start_time DESC
    ...> LIMIT 5;
+------------------------------------------------------------------+-----------------+----------+
| sha256                                                           | name            | username |
+------------------------------------------------------------------+-----------------+----------+
| 45fc2c2148bdea9cf7f2313b09a5cc27eead3460430ef55d1f5d0df6c1d96 ed4 | osqueryi        | robv     |
| 04a484f27a4b485b28451923605d9b528453d6c098a5a5112bec859fb5f2 eea9 | bash            | robv     |
| 45368907a48a0a3b5fff77a815565195a885da7d2aab8c4341c4ee869af4 c449 | gvfsd-metadata  | robv     |
| d3f9c91c6bbe4c7a3fdc914a7e5ac29f1cbfcc3f279b71e84badd25b313f ea45 | update-notifier | robv     |
| 83776c9c3d30cfc385be5d92b32f4beca2f6955e140d72d857139d2f7495 af1e | gnome-terminal- | robv     |
+------------------------------------------------------------------+-----------------+----------+
```

这个工具在事件响应情况下特别有效。通过我们在这几页中列出的查询，我们可以快速找到具有特定操作系统或软件版本的主机-换句话说，我们可以找到容易受到特定攻击的主机。此外，我们可以收集所有运行进程的哈希值，以找到可能伪装成良性进程的恶意软件。所有这些都可以通过只进行几次查询来完成。

最后一部分将关键控件中的高级指令转换为 Linux 中的“实用”命令，以实现这些目标。让我们看看这与更具规范性和操作系统或应用程序特定的安全指导有何不同-在这种情况下，将 CIS 基准应用于主机实施。

# 互联网安全中心基准

CIS 发布描述任何数量基础设施组件的安全配置的安全基准。这包括几种不同 Linux 发行版的所有方面，以及可能部署在 Linux 上的许多应用程序。这些基准非常“规范”-基准中的每个建议都描述了问题，如何使用操作系统命令或配置解决问题，以及如何对当前设置的状态进行审计。

CIS 基准的一个非常吸引人的特点是，它们是由一群行业专家编写和维护的，他们自愿投入时间使互联网变得更安全。虽然供应商参与制定这些文件，但它们是团体努力的最终建议需要团体的共识。最终结果是一个与供应商无关、共识和社区驱动的具有非常具体建议的文件。

CIS 基准旨在构建更好的平台（无论平台是什么），并且可以进行审计，因此每个建议都有补救和审计部分。对每个基准的详细解释至关重要，这样管理员不仅知道他们在改变什么，还知道为什么。这一点很重要，因为并非所有建议都适用于每种情况，事实上，有时建议会相互冲突，或者导致目标系统上的特定事物无法正常工作。这些情况在文件中描述，但这强调了不要将所有建议都最大程度地实施的重要性！这也清楚地表明，在审计情况下，追求“100%”并不符合任何人的最佳利益。

这些基准的另一个关键特点是它们通常是两个基准合二为一-对“常规”组织会有建议，对更高安全性环境则有更严格的建议。

CIS 确实维护一个审计应用程序**CIS-CAT**（**配置评估工具**），它将根据其基准评估基础架构，但许多行业标准工具，如安全扫描仪（如 Nessus）和自动化工具（如 Ansible、Puppet 或 Chef），将根据适用的 CIS 基准评估目标基础架构。

现在我们了解了基准的目的，让我们来看看 Linux 基准，特别是其中一组建议。

## 应用 CIS 基准-在 Linux 上保护 SSH

在保护服务器、工作站或基础架构平台时，有一个想要保护的事项清单以及如何实现的清单会很有帮助。这就是 CIS 基准的用途。正如讨论的那样，您可能永远不会完全在任何一个主机上实施所有 CIS 基准的建议-安全建议通常会损害或禁用您可能需要的服务，并且有时建议会相互冲突。这意味着基准通常会经过仔细评估，并被用作组织特定构建文档的主要输入。

让我们使用 Ubuntu 20.04 的 CIS 基准来保护我们主机上的 SSH 服务。SSH 是远程连接和管理 Linux 主机的主要方法。这使得在 Linux 主机上保护 SSH 服务器成为一项重要任务，并且通常是建立网络连接后的第一个配置任务。

首先，下载基准-所有平台的基准文档位于[`www.cisecurity.org/cis-benchmarks/`](https://www.cisecurity.org/cis-benchmarks/)。如果您不是运行 Ubuntu 20.04，请下载与您的发行版最接近的基准。您会发现 SSH 是一种非常常见的服务，用于保护 SSH 服务的建议在发行版之间非常一致，并且在非 Linux 平台上通常有相匹配的建议。

在开始之前，更新存储库列表并升级操作系统软件包-再次注意我们如何同时运行两个命令。在命令上使用单个`&`终止符会将其在后台运行，但使用`&&`会按顺序运行两个命令，第二个命令在第一个成功完成时执行（也就是说，如果它具有零的“返回值”）：

```
$ sudo apt-get update && sudo apt-get upgrade
```

您可以在`bash man`页面上了解更多信息（执行`man bash`）。

现在，操作系统组件已更新，让我们安装 SSH 守护程序，因为它在 Ubuntu 上默认情况下未安装：

```
$ sudo apt-get install openssh-server
```

在现代 Linux 发行版中，这将安装 SSH 服务器，然后进行基本配置并启动服务。

现在让我们开始保护它。在 Ubuntu 基准中查看 SSH 部分，我们看到了 22 个不同的配置设置的建议：

+   5.2 配置 SSH 服务器。

+   5.2.1 确保配置了`/etc/ssh/sshd_config`的权限。

+   5.2.2 确保配置了 SSH 私有主机密钥文件的权限。

+   5.2.3 确保配置了 SSH 公共主机密钥文件的权限。

+   5.2.4 确保 SSH`LogLevel`适当。

+   5.2.5 确保禁用 SSH X11 转发。

+   5.2.6 确保 SSH`MaxAuthTries`设置为`4`或更少。

+   5.2.7 确保启用了 SSH`IgnoreRhosts`。

+   5.2.8 确保禁用了 SSH`HostbasedAuthentication`。

+   5.2.9 确保 SSH 根登录已禁用。

+   5.2.10 确保禁用了 SSH`PermitEmptyPasswords`。

+   5.2.11 确保禁用了 SSH`PermitUserEnvironment`。

+   5.2.12 确保仅使用强大的密码。

+   5.2.13 确保仅使用强大的 MAC 算法。

+   5.2.14 确保仅使用强大的密钥交换算法。

+   5.2.15 确保配置了 SSH 空闲超时间隔。

+   5.2.16 确保 SSH`LoginGraceTime`设置为一分钟或更短。

+   5.2.17 确保限制了 SSH 访问。

+   5.2.18 确保配置了 SSH 警告横幅。

+   5.2.19 确保启用了 SSH PAM。

+   5.2.20 确保禁用 SSH`AllowTcpForwarding`。

+   5.2.21 确保配置了 SSH`MaxStartups`。

+   5.2.22 确保限制了 SSH`MaxSessions`。

为了说明这些工作原理，让我们更详细地看一下两个建议 - 禁用 root 用户的直接登录（5.2.9）和确保我们的加密密码是字符串（5.2.12）。

### 确保 SSH root 登录已禁用（5.2.9）

这个建议是确保用户都使用他们的命名帐户登录 - 用户“root”不应该直接登录。这确保了任何可能指示配置错误或恶意活动的日志条目将附有一个真实的人名。

这个术语叫做“不可否认性” - 如果每个人都有自己的命名帐户，而且没有“共享”帐户，那么在发生事故时，没有人可以声称“每个人都知道那个密码，不是我”。

这个审计命令是运行以下命令：

```
$ sudo sshd -T | grep permitrootlogin
permitrootlogin without-password
```

这个默认设置是不符合要求的。我们希望这个是“no”。`without-password`值表示您可以使用非密码方法（如使用证书）以 root 用户身份登录。

为了解决这个问题，我们将在补救部分查找。这告诉我们编辑`/etc/ssh/sshd_config`文件，并添加`PermitRootLogin no`这一行。`PermitRootLogin`被注释掉了（用`#`字符），所以我们要么取消注释，要么更好的是直接在注释值下面添加我们的更改，如下所示：

![图 5.1 - 对 sshd_config 文件的编辑，以拒绝 SSH 上的 root 登录](img/B16336_05_001.jpg)

图 5.1 - 对 sshd_config 文件的编辑，以拒绝 SSH 上的 root 登录

现在我们将重新运行我们的审计检查，我们会看到我们现在符合要求了：

```
$ sudo sshd -T | grep permitrootlogin
permitrootlogin no
```

实施了这个建议后，让我们看看我们在 SSH 密码上的情况（CIS 基准建议 5.2.12）。

### 确保只使用强密码（5.2.12）

这个检查确保只使用强密码来加密实际的 SSH 流量。审计检查表明我们应该再次运行`sshd –T`，并查找“ciphers”行。我们希望确保我们只启用已知的字符串密码，目前这是一个短列表：

+   `aes256-ctr`

+   `aes192-ctr`

+   `aes128-ctr`

特别是，SSH 的已知弱密码包括任何`DES`或`3DES`算法，或任何块密码（附加了`cbc`）。

让我们检查我们当前的设置：

```
$ sudo sshd -T | grep Ciphers
ciphers chacha20-poly1305@openssh.com,aes128-ctr,aes192-ctr,aes256-ctr,aes128-gcm@openssh.com,aes256-gcm@openssh.com
```

虽然我们在列表中有已知符合要求的密码，但我们也有一些不符合要求的密码。这意味着攻击者可以在适当的位置“降级”协商的密码为一个不太安全的密码，当会话建立时。

在补救部分，我们被指示查看同一个文件并更新“ciphers”行。在文件中，根本没有“Ciphers”行，只有一个`Ciphers and keyring`部分。这意味着我们需要添加那一行，如下所示：

```
# Ciphers and keying
Ciphers aes256-ctr,aes192-ctr,aes128-ctr
```

保持注释不变。例如，如果以后需要密钥环，那么可以在那里找到它们的占位符。尽可能保留或添加尽可能多的注释是明智的 - 保持配置尽可能“自我记录”是使下一个可能需要排除您刚刚做出的更改的人的工作变得容易的好方法。特别是，如果多年过去了，那么下一个人是您自己的未来版本！

接下来，我们将重新加载`sshd`守护程序，以确保我们所有的更改都生效：

```
$ sudo systemctl reload sshd
```

最后，重新运行我们的审计检查：

```
$ cat sshd_config | grep Cipher
# Ciphers and keying
Ciphers aes256-ctr,aes192-ctr,aes128-ctr
```

成功！

我们如何在我们的主机上检查密码支持？这个密码更改是一个重要的设置，很可能需要在许多系统上设置，其中一些可能没有一个可以直接编辑的 Linux 命令行或`sshd_config`文件。回想一下上一章。我们将使用`nmap`从远程系统检查这个设置，使用`ssh2-enum-algos.nse`脚本。我们将查看密码的`Encryption Algorithms`脚本输出部分：

```
$ sudo nmap -p22 -Pn --open 192.168.122.113 --script ssh2-enum-algos.nse
Starting Nmap 7.80 ( https://nmap.org ) at 2021-02-08 15:22 Eastern Standard Time
Nmap scan report for ubuntu.defaultroute.ca (192.168.122.113)
Host is up (0.00013s latency).
PORT   STATE SERVICE
22/tcp open  ssh
| ssh2-enum-algos:
|   kex_algorithms: (9)
|       curve25519-sha256
|       curve25519-sha256@libssh.org
|       ecdh-sha2-nistp256
|       ecdh-sha2-nistp384
|       ecdh-sha2-nistp521
|       diffie-hellman-group-exchange-sha256
|       diffie-hellman-group16-sha512
|       diffie-hellman-group18-sha512
|       diffie-hellman-group14-sha256
|   server_host_key_algorithms: (5)
|       rsa-sha2-512
|       rsa-sha2-256
|       ssh-rsa
|       ecdsa-sha2-nistp256
|       ssh-ed25519
|   encryption_algorithms: (3)
|       aes256-ctr
|       aes192-ctr
|       aes128-ctr
|   mac_algorithms: (10)
|       umac-64-etm@openssh.com
|       umac-128-etm@openssh.com
|       hmac-sha2-256-etm@openssh.com
|       hmac-sha2-512-etm@openssh.com
|       hmac-sha1-etm@openssh.com
|       umac-64@openssh.com
|       umac-128@openssh.com
|       hmac-sha2-256
|       hmac-sha2-512
|       hmac-sha1
|   compression_algorithms: (2)
|       none
|_      zlib@openssh.com
MAC Address: 00:0C:29:E2:91:BC (VMware)
Nmap done: 1 IP address (1 host up) scanned in 4.09 seconds
```

使用第二个工具来验证您的配置是一个重要的习惯 - 虽然 Linux 是一个可靠的服务器和工作站平台，但是会出现错误。此外，很容易在进行更改后退出时不小心没有保存配置更改 - 使用另一个工具进行双重检查是确保一切都如预期的好方法！

最后，如果您曾经接受过审计，安排进行渗透测试，或者在您的网络上实际上有恶意软件，那么在每种情况下都很可能进行网络扫描以寻找弱算法（或更糟糕的是，明文 Telnet 或`rsh`）。如果您使用与攻击者（或审计人员）相同的工具和方法，您更有可能捕捉到被忽略的那一台主机或者那一组具有您意料之外的 SSH 漏洞的主机！

您应该检查哪些其他关键事项？虽然值得检查 SSH 的所有设置，但在每种情况和环境中，其中一些是关键的：

+   检查您的 SSH 日志级别，以便知道谁从哪个 IP 地址登录（5.2.4）。

+   密钥交换和 MAC 算法检查与密码检查类似；它们加强了协议本身（5.2.13 和 5.2.14）。

+   您需要设置一个空闲超时（5.2.15）。这很重要，因为无人看管的管理员登录可能是一件危险的事情，例如，如果管理员忘记锁定屏幕。此外，如果有人习惯于关闭他们的 SSH 窗口而不是注销，那么在许多平台上这些会话不会关闭。如果达到最大会话数（经过几个月后），下一次连接尝试将失败。要解决这个问题，您需要到物理屏幕和键盘上解决这个问题（例如重新启动 SSHD）或重新加载系统。

+   您需要设置**MaxSessions 限制**（5.2.22）。特别是如果您的主机面临敌对网络（这是现在的每个网络），一个简单地开始数百个 SSH 会话的攻击可能会耗尽主机上的资源，影响其他用户可用的内存和 CPU。

尽管如此，应该审查和评估基准的每个部分中的每个建议，以查看它对您的环境是否合适。在这个过程中，通常会为您的环境创建一个构建文档，一个可以用作模板来克隆生产主机的“金像”主机，以及一个审计脚本或加固脚本，以帮助维护正在运行的主机。

# SELinux 和 AppArmor

Linux 有两个常用的 Linux 安全模块（LSMs），它们为系统添加了额外的安全策略、控制和更改默认行为。在许多情况下，它们修改了 Linux 内核本身。它们都适用于大多数 Linux 发行版，并且在实施时都带有一定程度的风险 - 您需要在实施之前做一些准备工作，以评估实施其中一个可能产生的影响。不建议同时实施两者，因为它们很可能会发生冲突。

SELinux 可以说更加完整，而且管理起来肯定更加复杂。它是一组添加到基本安装中的内核修改和工具。在高层次上，它分离了安全策略的配置和执行。控制包括强制访问控制、强制完整性控制、基于角色的访问控制（RBAC）和类型强制。

SELinux 的功能包括以下内容：

+   将安全策略的定义与执行分开。

+   定义安全策略的明确定义的接口（通过工具和 API）。

+   允许应用程序查询策略定义或特定访问控制。一个常见的例子是允许`crond`在正确的上下文中运行计划任务。

+   支持修改默认策略或创建全新的自定义策略。

+   保护系统完整性（域完整性）和数据保密性（多级安全）的措施。

+   对进程初始化、执行和继承进行控制。

+   对文件系统、目录、文件和打开文件描述符（例如管道或套接字）的额外安全控制。

+   套接字、消息和网络接口的安全控制。

+   对“能力”（RBAC）的使用进行控制。

+   在可能的情况下，策略中不允许的任何内容都将被拒绝。这种“默认拒绝”方法是 SELinux 的根本设计原则之一。

**AppArmor**具有与 SELinux 许多相同的功能，但它使用文件路径而不是对文件应用标签。它还实施了强制访问控制。您可以为任何应用程序分配安全配置文件，包括文件系统访问、网络权限和执行规则。这个列表也很好地概述了 AppArmor 也实施了 RBAC。

由于 AppArmor 不使用文件标签，这使得它在文件系统方面是不可知的，如果文件系统不支持安全标签，这使得它成为唯一的选择。另一方面，这也意味着这种架构决策限制了它匹配 SELinux 所有功能的能力。

AppArmor 的功能包括对以下内容的限制：

+   文件访问控制

+   库加载控制

+   进程执行控制

+   对网络协议的粗粒度控制

+   命名套接字

+   对象上的粗粒度所有者检查（需要 Linux 内核 2.6.31 或更新版本）

两种 LVM 都有学习选项：

+   SELinux 有一个宽容模式，这意味着策略已启用但未强制执行。这种模式允许您测试应用程序，然后检查 SELinux 日志，以查看在强制执行策略时您的应用程序可能受到的影响。可以通过编辑`/etc/selinux/config`文件并将`selinux`行更改为**enforcing**、**permissive**或**disabled**来控制 SELinux 模式。更改后需要重新启动系统。

+   AppArmor 的学习模式称为`aa-complain`。要为所有配置文件的应用程序激活此模式，命令是`aa-complain/etc/apparmor.d/*`。激活学习模式后，然后测试一个应用程序，您可以使用`aa-logprof`命令查看 AppArmor 可能如何影响该应用程序（对于此命令，您需要配置文件和日志的完整路径）。

要检查 LVM 的状态，命令如下：

+   对于 SELinux，命令是`getenforce`，或者更详细的输出是`sestatus`。

+   对于 AppArmor，类似的命令是`apparmor status`和`aa-status`。

总之，AppArmor 和 SELinux 都是复杂的系统。 SELinux 被认为更加复杂，但也更加完整。如果您选择其中一种方法，您应该首先在测试系统上进行测试。在部署之前，最好在生产主机的克隆上尽可能多地测试和构建您的生产配置。这两种解决方案都可以显著提高主机和应用程序的安全性，但都需要大量的设置工作，以及持续的努力来确保主机和应用程序在随着时间的推移而变化时能够正常运行。

这两个系统的更完整的解释超出了本书的范围-如果您希望更全面地探索其中任何一个，它们都有几本专门的书籍。 

# 总结

我们讨论的一切的最终目标——监管框架、关键控制和安全基准——都是为了更容易地更好地保护您的主机和数据中心。在这些指导结构中的关键是为您提供足够的指导，使您能够达到所需的目标，而无需成为安全专家。每一个都变得越来越具体。监管框架通常非常广泛，在如何完成任务方面留下了相当大的自由裁量权。关键控制更为具体，但仍然允许在部署解决方案和实现最终目标方面有相当大的灵活性。CIS 基准非常具体，为您提供了实现目标所需的确切命令和配置更改。

我希望通过本章我们所走过的旅程，您对如何在您的组织中结合这些各种指导方法来更好地保护您的 Linux 基础设施有一个良好的了解。

在下一章中，我们将讨论在 Linux 上实施 DNS 服务。如果您希望继续了解如何更具体地保护您的主机，不用担心——随着我们实施新服务，这个安全讨论会一次又一次地出现。

# 问题

随着我们的结束，这里有一些问题供您测试对本章材料的了解。您将在*附录*的*评估*部分找到答案：

1.  在 IT 实施中，使用哪些美国立法来定义隐私要求？

1.  你能按照 CIS 关键控制进行审计吗？

1.  为什么您会经常使用多种方法来检查一个安全设置——例如 SSH 的加密算法？

# 进一步阅读

有关本章涵盖的主题的更多信息，您可以查看以下链接：

+   PCIDSS: [`www.pcisecuritystandards.org/`](https://www.pcisecuritystandards.org/)

+   HIPAA: [`www.hhs.gov/hipaa/index.html`](https://www.hhs.gov/hipaa/index.html)

+   NIST: [`csrc.nist.gov/publications/sp800`](https://csrc.nist.gov/publications/sp800)

+   FEDRAMP: [`www.fedramp.gov/`](https://www.fedramp.gov/)

+   DISA STIGs: [`public.cyber.mil/stigs/`](https://public.cyber.mil/stigs/)

+   GDPR: [`gdpr-info.eu/`](https://gdpr-info.eu/)

+   PIPEDA: [`www.priv.gc.ca/en/privacy-topics/privacy-laws-in-canada/the-personal-information-protection-and-electronic-documents-act-pipeda/`](https://www.priv.gc.ca/en/privacy-topics/privacy-laws-in-canada/the-personal-information-protection-and-electronic-documents-act-pipeda/)

+   CIS: [`www.cisecurity.org/controls/`](https://www.cisecurity.org/controls/)

[`isc.sans.edu/forums/diary/Critical+Control+2+Inventory+of+Authorized+and+Unauthorized+Software/11728/`](https://isc.sans.edu/forums/diary/Critical+Control+2+Inventory+of+Authorized+and+Unauthorized+Software/11728/)

+   CIS 基准：[`www.cisecurity.org/cis-benchmarks/`](https://www.cisecurity.org/cis-benchmarks/)

+   OSQuery: [`osquery.readthedocs.io/en/stable/`](https://osquery.readthedocs.io/en/stable/)

+   SELinux: [`www.selinuxproject.org/page/Main_Page`](http://www.selinuxproject.org/page/Main_Page)

+   AppArmor: [`apparmor.net/`](https://apparmor.net/)
