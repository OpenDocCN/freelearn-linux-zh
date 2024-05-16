# 第九章。启动- init 程序

我在第四章中看到了内核如何引导到启动第一个程序`init`的点，在第五章中，*构建根文件系统*和第六章中，*选择构建系统*，我看了创建不同复杂性的根文件系统，其中都包含了`init`程序。现在是时候更详细地看看`init`程序，并发现它对系统的重要性。

`init`有许多可能的实现。我将在本章中描述三种主要的实现：BusyBox `init`，System V `init`和`systemd`。对于每种实现，我将概述其工作原理和最适合的系统类型。其中一部分是在复杂性和灵活性之间取得平衡。

# 内核引导后

我们在第四章中看到了*移植和配置内核*，内核引导代码如何寻找根文件系统，要么是`initramfs`，要么是内核命令行上指定的文件系统`root=`，然后执行一个程序，默认情况下是`initramfs`的`/init`，常规文件系统的`/sbin/init`。`init`程序具有根特权，并且由于它是第一个运行的进程，它具有进程 ID（`PID`）为 1。如果由于某种原因`init`无法启动，内核将会恐慌。

`init`程序是所有其他进程的祖先，如`pstree`命令所示，它是大多数发行版中`psmisc`软件包的一部分：

```
# pstree -gn

init(1)-+-syslogd(63)
        |-klogd(66)
        |-dropbear(99)
        `-sh(100)---pstree(109)
```

`init`程序的工作是控制系统并使其运行。它可能只是一个运行 shell 脚本的 shell 命令-在第五章的开头有一个示例，*构建根文件系统*—但在大多数情况下，您将使用专用的`init`守护程序。它必须执行的任务如下：

+   在启动时，它启动守护程序，配置系统参数和其他必要的东西，使系统进入工作状态。

+   可选地，它启动守护程序，比如在允许登录 shell 的终端上启动`getty`。

+   它接管因其直接父进程终止而变成孤儿的进程，并且没有其他进程在线程组中。

+   它通过捕获信号`SIGCHLD`并收集返回值来响应`init`的任何直接子进程的终止，以防止它们变成僵尸进程。我将在第十章中更多地讨论僵尸进程，*了解进程和线程*。

+   可选地，它重新启动那些已经终止的守护进程。

+   它处理系统关闭。

换句话说，`init`管理系统的生命周期，从启动到关闭。目前的想法是`init`很适合处理其他运行时事件，比如新硬件和模块的加载和卸载。这就是`systemd`的作用。

# 介绍 init 程序

在嵌入式设备中，您最有可能遇到的三种`init`程序是 BusyBox `init`，System V `init`和`systemd`。Buildroot 有选项可以构建所有三种，其中 BusyBox `init`是默认选项。Yocto Project 允许您在 System V `init`和`systemd`之间进行选择，System `V init`是默认选项。

以下表格提供了比较这三种程序的一些指标：

| | BusyBox init | System V init | systemd |
| --- | --- | --- | --- |
| --- | --- | --- | --- |
| 复杂性 | 低 | 中等 | 高 |
| 启动速度 | 快 | 慢 | 中等 |
| 所需的 shell | ash | ash 或 bash | 无 |
| 可执行文件数量 | 0 | 4 | 50(*) |
| libc | 任何 | 任何 | glibc |
| 大小（MiB） | 0 | 0.1 | 34(*) |

(*)基于`system`的 Buildroot 配置。

总的来说，从 BusyBox `init`到`systemd`，灵活性和复杂性都有所增加。

# BusyBox init

BusyBox 有一个最小的`init`程序，使用配置文件`/etc/inittab`来定义在启动时启动程序的规则，并在关闭时停止它们。通常，实际工作是由 shell 脚本完成的，按照惯例，这些脚本放在`/etc/init.d`目录中。

`init`首先通过读取配置文件`/etc/inittab`来开始。其中包含要运行的程序列表，每行一个，格式如下：

`<id>::<action>:<program>`

这些参数的作用如下：

+   `id`：命令的控制终端

+   `action`：运行此命令的条件，如下一段所示

+   `program`：要运行的程序

以下是操作步骤：

+   `sysinit`：当`init`启动时运行程序，先于其他类型的操作。

+   `respawn`：运行程序并在其终止时重新启动。用于将程序作为守护进程运行。

+   `askfirst`：与`respawn`相同，但在控制台上打印消息**请按 Enter 键激活此控制台**，并在按下*Enter*后运行程序。用于在终端上启动交互式 shell 而无需提示用户名或密码。

+   `once`：运行程序一次，但如果终止则不尝试重新启动。

+   `wait`：运行程序并等待其完成。

+   `restart`：当`init`接收到信号`SIGHUP`时运行程序，表示应重新加载`inittab`文件。

+   `ctrlaltdel`：当`init`接收到信号`SIGINT`时运行程序，通常是在控制台上按下*Ctrl* + *Alt* + *Del*的结果。

+   `shutdown`：当`init`关闭时运行程序。

以下是一个小例子，它挂载`proc`和`sysfs`，并在串行接口上运行 shell：

```
null::sysinit:/bin/mount -t proc proc /proc
null::sysinit:/bin/mount -t sysfs sysfs /sys
console::askfirst:-/bin/sh
```

对于简单的项目，您希望启动少量守护进程并可能在串行终端上启动登录 shell，手动编写脚本很容易，如果您正在创建一个**RYO**（**roll your own**）嵌入式 Linux，这是合适的。但是，随着需要配置的内容增加，您会发现手写的`init`脚本很快变得难以维护。它们往往不太模块化，因此每次添加新组件时都需要更新。

## Buildroot init 脚本

多年来，Buildroot 一直在有效地使用 BusyBox `init`。Buildroot 在`/etc/init.d`中有两个脚本，名为`rcS`和`rcK`。第一个在启动时启动，并遍历所有以大写`S`开头后跟两位数字的脚本，并按数字顺序运行它们。这些是启动脚本。`rcK`脚本在关闭时运行，并遍历所有以大写`K`开头后跟两位数字的脚本，并按数字顺序运行它们。这些是关闭脚本。

有了这个，Buildroot 软件包可以轻松提供自己的启动和关闭脚本，使用两位数字来规定它们应该运行的顺序，因此系统变得可扩展。如果您正在使用 Buildroot，这是透明的。如果没有，您可以将其用作编写自己的 BusyBox `init`脚本的模型。

# System V init

这个`init`程序受 UNIX System V 的启发，可以追溯到 20 世纪 80 年代中期。在 Linux 发行版中最常见的版本最初是由 Miquel van Smoorenburg 编写的。直到最近，它被认为是引导 Linux 的方式，显然包括嵌入式系统，而 BusyBox `init`是 System V `init`的精简版本。

与 BusyBox `init`相比，System V `init`有两个优点。首先，引导脚本以众所周知的模块化格式编写，使得在构建时或运行时轻松添加新包。其次，它具有运行级别的概念，允许通过从一个运行级别切换到另一个运行级别来一次性启动或停止一组程序。

有从 0 到 6 编号的 8 个运行级别，另外还有 S：

+   **S**：单用户模式

+   **0**：关闭系统

+   **1 至 5**：通用使用

+   **6**：重新启动系统

级别 1 到 5 可以随您的意愿使用。在桌面 Linux 发行版中，它们通常分配如下：

+   **1**：单用户

+   **2**：无需网络配置的多用户

+   **3**：带网络配置的多用户

+   **4**：未使用

+   **5**：带图形登录的多用户

`init`程序启动由`/etc/inittab`中的`initdefault`行给出的默认`runlevel`。您可以使用`telinit [runlevel]`命令在运行时更改运行级别，该命令向`init`发送消息。您可以使用`runlevel`命令找到当前运行级别和先前的运行级别。以下是一个示例：

```
# runlevel
N 5
# telinit 3
INIT: Switching to runlevel: 3
# runlevel
5 3
```

在第一行上，`runlevel`的输出是`N 5`，这意味着没有先前的运行级别，因为自启动以来`runlevel`没有改变，当前的`runlevel`是`5`。在改变`runlevel`后，输出是`5 3`，显示已从`5`转换到`3`。`halt`和`reboot`命令分别切换到`0`和`6`的运行级别。您可以通过在内核命令行上给出不同的单个数字`0`到`6`，或者`S`表示单用户模式，来覆盖默认的`runlevel`。例如，要强制`runlevel`为单用户，您可以在内核命令行上附加`S`，看起来像这样：

```
console=ttyAMA0 root=/dev/mmcblk1p2 S

```

每个运行级别都有一些停止事物的脚本，称为`kill`脚本，以及另一组启动事物的脚本，称为`start`脚本。进入新的`runlevel`时，`init`首先运行`kill`脚本，然后运行`start`脚本。在新的`runlevel`中运行守护进程，如果它们既没有`start`脚本也没有`kill`脚本，那么它们将收到`SIGTERM`信号。换句话说，切换`runlevel`的默认操作是终止守护进程，除非另有指示。

事实上，在嵌入式 Linux 中并不经常使用运行级别：大多数设备只是启动到默认的`runlevel`并保持在那里。我有一种感觉，部分原因是大多数人并不知道它们。

### 提示

运行级别是在不同模式之间切换的一种简单方便的方式，例如，从生产模式切换到维护模式。

System V `init`是 Buildroot 和 Yocto Project 的一个选项。在这两种情况下，init 脚本已经被剥离了任何 bash 特定的内容，因此它们可以与 BusyBox ash shell 一起工作。但是，Buildroot 通过用 SystemV `init`替换 BusyBox `init`程序并添加模仿 BusyBox 行为的`inittab`来作弊。Buildroot 不实现运行级别，除非切换到级别 0 或 6 会停止或重新启动系统。

接下来，让我们看一些细节。以下示例取自 Yocto Project 的 fido 版本。其他发行版可能以稍有不同的方式实现`init`脚本。

## inittab

`init`程序首先读取`/etc/inttab`，其中包含定义每个`runlevel`发生的事情的条目。格式是我在前一节中描述的 BusyBox `inittab`的扩展版本，这并不奇怪，因为 BusyBox 首先从 System V 借鉴了它！

`inittab`中每行的格式如下：

```
id:runlevels:action:process
```

字段如下所示：

+   `id`：最多四个字符的唯一标识符。

+   `runlevels`：应执行此条目的运行级别。（在 BusyBox `inittab`中留空）

+   `action`：以下给出的关键字之一。

+   `process`：要运行的命令。

这些操作与 BusyBox `init`的操作相同：`sysinit`，`respawn`，`once`，`wait`，`restart`，`ctrlaltdel`和`shutdown`。但是，System V `init`没有`askfirst`，这是 BusyBox 特有的。

例如，这是 Yocto Project 目标 core-image-minimal 提供的完整的`inttab`：

```
# /etc/inittab: init(8) configuration.
# $Id: inittab,v 1.91 2002/01/25 13:35:21 miquels Exp $

# The default runlevel.
id:5:initdefault:

# Boot-time system configuration/initialization script.
# This is run first except when booting in emergency (-b) mode.
si::sysinit:/etc/init.d/rcS

# What to do in single-user mode.
~~:S:wait:/sbin/sulogin
# /etc/init.d executes the S and K scripts upon change
# of runlevel.
#
# Runlevel 0 is halt.
# Runlevel 1 is single-user.
# Runlevels 2-5 are multi-user.
# Runlevel 6 is reboot.

l0:0:wait:/etc/init.d/rc 0
l1:1:wait:/etc/init.d/rc 1
l2:2:wait:/etc/init.d/rc 2
l3:3:wait:/etc/init.d/rc 3
l4:4:wait:/etc/init.d/rc 4
l5:5:wait:/etc/init.d/rc 5
l6:6:wait:/etc/init.d/rc 6
# Normally not reached, but fallthrough in case of emergency.
z6:6:respawn:/sbin/sulogin
AMA0:12345:respawn:/sbin/getty 115200 ttyAMA0
# /sbin/getty invocations for the runlevels.
#
# The "id" field MUST be the same as the last
# characters of the device (after "tty").
#
# Format:
#  <id>:<runlevels>:<action>:<process>
#

1:2345:respawn:/sbin/getty 38400 tty1
```

第一个条目`id:5:initdefault`将默认的`runlevel`设置为`5`。接下来的条目`si::sysinit:/etc/init.d/rcS`在启动时运行脚本`rcS`。稍后会有更多关于这个的内容。稍后，有一组六个条目，以`l0:0:wait:/etc/init.d/rc 0`开头。它们在运行级别发生变化时运行脚本`/etc/init.d/rc`：这个脚本负责处理`start`和`kill`脚本。还有一个运行级别`S`的条目，运行单用户登录程序。

在`inittab`的末尾，有两个条目，当进入运行级别 1 到 5 时，它们运行一个`getty`守护进程在设备`/dev/ttyAMA0`和`/dev/tty1`上生成登录提示，从而允许你登录并获得交互式 shell：

```
AMA0:12345:respawn:/sbin/getty 115200 ttyAMA0
1:2345:respawn:/sbin/getty 38400 tty1
```

设备`ttyAMA0`是我们用 QEMU 模拟的 ARM Versatile 板上的串行控制台，对于其他开发板来说可能会有所不同。Tty1 是一个虚拟控制台，通常映射到图形屏幕，如果你的内核使用了`CONFIG_FRAMEBUFFER_CONSOLE`或`VGA_CONSOLE`。桌面 Linux 通常在虚拟终端 1 到 6 上生成六个`getty`进程，你可以用组合键*Ctrl* + *Alt* + *F1*到*Ctrl* + *Alt* + *F6*来选择，虚拟终端 7 保留给图形屏幕。嵌入式设备上很少使用虚拟终端。

由`sysinit`条目运行的脚本`/etc/init.d/rcS`几乎只是进入运行级别`S`：

```
#!/bin/sh

[...]
exec /etc/init.d/rc S
```

因此，第一个进入的运行级别是`S`，然后是默认的`runlevel` `5`。请注意，`runlevel` `S`不会被记录，也不会被`runlevel`命令显示为先前的运行级别。

## init.d 脚本

需要响应`runlevel`变化的每个组件都有一个在`/etc/init.d`中执行该变化的脚本。脚本应该期望两个参数：`start`和`stop`。稍后我会举一个例子。

`runlevel`处理脚本`/etc/init.d/rc`以`runlevel`作为参数进行切换。对于每个`runlevel`，都有一个名为`rc<runlevel>.d`的目录：

```
# ls -d /etc/rc*
/etc/rc0.d  /etc/rc2.d  /etc/rc4.d  /etc/rc6.d
/etc/rc1.d  /etc/rc3.d  /etc/rc5.d  /etc/rcS.d
```

在那里你会找到一组以大写`S`开头后跟两位数字的脚本，你也可能会找到以大写`K`开头的脚本。这些是`start`和`kill`脚本：Buildroot 使用了相同的想法，从这里借鉴过来：

```
# ls /etc/rc5.d
S01networking   S20hwclock.sh   S99rmnologin.sh S99stop-bootlogd
S15mountnfs.sh  S20syslog
```

实际上，这些是指向`init.d`中适当脚本的符号链接。`rc`脚本首先运行所有以`K`开头的脚本，添加`stop`参数，然后运行以`S`开头的脚本，添加`start`参数。再次强调，两位数字代码用于指定脚本应该运行的顺序。

## 添加新的守护进程

假设你有一个名为`simpleserver`的程序，它是作为传统的 Unix 守护进程编写的，换句话说，它会分叉并在后台运行。你将需要一个像这样的`init.d`脚本：

```
#! /bin/sh

case "$1" in
  start)
    echo "Starting simpelserver"
    start-stop-daemon -S -n simpleserver -a /usr/bin/simpleserver
    ;;
  stop)
    echo "Stopping simpleserver"
    start-stop-daemon -K -n simpleserver
    ;;
  *)
    echo "Usage: $0 {start|stop}"
  exit 1
esac

exit 0
```

`Start-stop-daemon`是一个帮助函数，使得更容易操作后台进程。它最初来自 Debian 安装程序包`dpkg`，但大多数嵌入式系统使用的是 BusyBox 中的版本。它使用`-S`参数启动守护进程，确保任何时候都不会有多个实例在运行，并使用`-K`按名称查找守护进程，并默认发送信号`SIGTERM`。将此脚本放在`/etc/init.d/simpleserver`中并使其可执行。

然后，从你想要从中运行这个程序的每个运行级别添加`symlinks`，在这种情况下，只有默认的`runlevel`，`5`：

```
# cd /etc/init.d/rc5.d
# ln -s ../init.d/simpleserver S99simpleserver
```

数字`99`表示这将是最后启动的程序之一。请记住，可能会有其他以`S99`开头的链接，如果是这样，`rc`脚本将按照词法顺序运行它们。

在嵌入式设备中很少需要过多担心关机操作，但如果有需要做的事情，可以在 0 和 6 级别添加`kill symlinks`：

```
# cd /etc/init.d/rc0.d
# ln -s ../init.d/simpleserver K01simpleserver
# cd /etc/init.d/rc6.d
# ln -s ../init.d/simpleserver K01simpleserver
```

## 启动和停止服务

您可以通过直接调用`/etc/init.d`中的脚本与之交互，例如，控制`syslogd`和`klogd`守护进程的`syslog`脚本：

```
# /etc/init.d/syslog --help
Usage: syslog { start | stop | restart }

# /etc/init.d/syslog stop
Stopping syslogd/klogd: stopped syslogd (pid 198)
stopped klogd (pid 201)
done

# /etc/init.d/syslog start
Starting syslogd/klogd: done
```

所有脚本都实现了`start`和`stop`，并且应该实现`help`。有些还实现了`status`，它会告诉您服务是否正在运行。仍在使用 System V `init`的主流发行版有一个名为 service 的命令，用于启动和停止服务，并隐藏直接调用脚本的细节。

# systemd

`systemd`将自己定义为系统和服务管理器。该项目由 Lennart Poettering 和 Kay Sievers 于 2010 年发起，旨在创建一套集成的工具，用于管理 Linux 系统，包括`init`守护程序。它还包括设备管理（`udev`）和日志记录等内容。有人会说它不仅仅是一个`init`程序，它是一种生活方式。它是最先进的，仍在快速发展。`systemd`在桌面和服务器 Linux 发行版上很常见，并且在嵌入式 Linux 系统上也变得越来越受欢迎，特别是在更复杂的设备上。那么，它比 System V `init`在嵌入式系统上更好在哪里呢？

+   配置更简单更合乎逻辑（一旦你理解了它），而不是 System V `init`有时候复杂的 shell 脚本，`systemd`有单元配置文件来设置参数

+   服务之间有明确的依赖关系，而不是仅仅设置脚本运行顺序的两位数代码

+   为每个服务设置权限和资源限制很容易，这对安全性很重要

+   `systemd`可以监视服务并在需要时重新启动它们

+   每个服务和`systemd`本身都有看门狗

+   服务并行启动，减少启动时间

在这里，不可能也不合适对`systemd`进行完整描述。与 System V `init`一样，我将专注于嵌入式用例，并以 Yocto Fido 生成的配置为例，该配置具有`systemd`版本 219。我将进行快速概述，然后向您展示一些具体示例。

## 使用 Yocto Project 和 Buildroot 构建 systemd

Yocto Fido 中的默认`init`是 System V。要选择`systemd`，请在配置中添加这些行，例如，在`conf/local.conf`中：

```
DISTRO_FEATURES_append = " systemd"
VIRTUAL-RUNTIME_init_manager = "systemd"
```

请注意，前导空格很重要！然后重新构建。

Buildroot 将`systemd`作为第三个`init`选项。它需要 glibc 作为 C 库，并且需要启用特定一组配置选项的内核版本为 3.7 或更高。在`systemd`源代码的顶层的`README`文件中有完整的依赖项列表。

### 介绍目标、服务和单元

在我描述`systemd init`如何工作之前，我需要介绍这三个关键概念。

首先，目标是一组服务，类似于但更一般化的 SystemV `runlevel`。有一个默认目标，它是在启动时启动的服务组。

其次，服务是可以启动和停止的守护进程，非常类似于 SystemV `service`。

最后，一个单元是一个描述`target`，`service`和其他几个东西的配置文件。单元是包含属性和值的文本文件。

您可以使用`systemctl`命令更改状态并了解发生了什么。

#### 单元

配置的基本项是单元文件。单元文件位于三个不同的位置：

+   `/etc/systemd/system`：本地配置

+   `/run/systemd/system`：运行时配置

+   `/lib/systemd/system`：分发范围内的配置

在寻找单元时，`systemd`按照这个顺序搜索目录，一旦找到匹配项就停止，这样可以通过在`/etc/systemd/system`中放置同名单元来覆盖分发范围内单元的行为。您可以通过创建一个空的本地文件或链接到`/dev/null`来完全禁用一个单元。

所有单元文件都以标有`[Unit]`的部分开头，其中包含基本信息和依赖项，例如：

```
[Unit]
Description=D-Bus System Message Bus
Documentation=man:dbus-daemon(1)
Requires=dbus.socket
```

单元依赖关系通过`Requires`、`Wants`和`Conflicts`来表达：

+   `Requires`: 此单元依赖的单元列表，当此单元启动时启动

+   `Wants`: `Requires`的一种较弱形式：列出的单元被启动，但如果它们中的任何一个失败，当前单元不会停止

+   `冲突`: 一个负依赖：列出的单元在此单元启动时停止，反之亦然

处理依赖关系会产生一个应该启动（或停止）的单元列表。关键字`Before`和`After`确定它们启动的顺序。停止的顺序只是启动顺序的相反：

+   `Before`: 在列出的单元之前应启动此单元

+   `After`: 在列出的单元之后应启动此单元

在以下示例中，`After`指令确保网络后启动 Web 服务器：

```
[Unit]
Description=Lighttpd Web Server
After=network.target
```

在没有`Before`或`After`指令的情况下，单元将并行启动或停止，没有特定的顺序。

#### 服务

服务是可以启动和停止的守护进程，相当于 System V 的`service`。服务是以`.service`结尾的一种单元文件，例如`lighttpd.service`。

服务单元有一个描述其运行方式的`[Service]`部分。以下是`lighttpd.service`的相关部分：

```
[Service]
ExecStart=/usr/sbin/lighttpd -f /etc/lighttpd/lighttpd.conf -D
ExecReload=/bin/kill -HUP $MAINPID
```

这些是启动服务和重新启动服务时要运行的命令。您可以在这里添加更多配置点，因此请参考`systemd.service`的手册页。

#### 目标

目标是另一种将服务（或其他类型的单元）分组的单元类型。它是一种只有依赖关系的单元类型。目标的名称以`.target`结尾，例如`multi-user.target`。目标是一种期望状态，起到与 System V 运行级别相同的作用。

## systemd 如何引导系统

现在我们可以看到`systemd`如何实现引导。`systemd`由内核作为`/sbin/init`的符号链接到`/lib/systemd/systemd`而运行。它运行默认目标`default.target`，它始终是一个指向期望目标的链接，例如文本登录的`multi-user.target`或图形环境的`graphical.target`。例如，如果默认目标是`multi-user.target`，您将找到此符号链接：

```
/etc/systemd/system/default.target -> /lib/systemd/system/multi-user.target
```

默认目标可以通过在内核命令行上传递`system.unit=<new target>`来覆盖。您可以使用`systemctl`来查找默认目标，如下所示：

```
# systemctl get-default
multi-user.target
```

启动诸如`multi-user.target`之类的目标会创建一个依赖树，将系统带入工作状态。在典型系统中，`multi-user.target`依赖于`basic.target`，后者依赖于`sysinit.target`，后者依赖于需要早期启动的服务。您可以使用`systemctl list-dependencies`打印图形。

您还可以使用`systemctl list-units --type service`列出所有服务及其当前状态，以及使用`systemctl list-units --type target`列出目标。

## 添加您自己的服务

使用与之前相同的`simpleserver`示例，这是一个服务单元：

```
[Unit]
Description=Simple server

[Service]
Type=forking
ExecStart=/usr/bin/simpleserver

[Install]
WantedBy=multi-user.target
```

`[Unit]`部分只包含一个描述，以便在使用`systemctl`和其他命令列出时正确显示。没有依赖关系；就像我说的，它非常简单。

`[Service]`部分指向可执行文件，并带有一个指示它分叉的标志。如果它更简单并在前台运行，`systemd`将为我们进行守护进程，`Type=forking`将不需要。

`[Install]`部分使其依赖于`multi-user.target`，这样我们的服务器在系统进入多用户模式时启动。

一旦单元保存在`/etc/systemd/system/simpleserver.service`中，您可以使用`systemctl start simpleserver`和`systemctl stop simpleserver`命令启动和停止它。您可以使用此命令查找其当前状态：

```
# systemctl status simpleserver
  simpleserver.service - Simple server
  Loaded: loaded (/etc/systemd/system/simpleserver.service; disabled)
  Active: active (running) since Thu 1970-01-01 02:20:50 UTC; 8s ago
  Main PID: 180 (simpleserver)
  CGroup: /system.slice/simpleserver.service
           └─180 /usr/bin/simpleserver -n

Jan 01 02:20:50 qemuarm systemd[1]: Started Simple server.
```

此时，它只会按命令启动和停止，如所示。要使其持久化，您需要向目标添加永久依赖项。这就是单元中`[Install]`部分的目的，它表示当启用此服务时，它将依赖于`multi-user.target`，因此将在启动时启动。您可以使用`systemctl enable`来启用它，如下所示：

```
# systemctl enable simpleserver
Created symlink from /etc/systemd/system/multi-user.target.wants/simpleserver.service to /etc/systemd/system/simpleserver.service.
```

现在您可以看到如何在运行时添加依赖项，而无需编辑任何单元文件。一个目标可以有一个名为`<target_name>.target.wants`的目录，其中可以包含指向服务的链接。这与在目标中的`[Wants]`列表中添加依赖单元完全相同。在这种情况下，您会发现已创建了此链接：

```
/etc/systemd/system/multi-user.target.wants/simpleserver.service
/etc/systemd/system/simpleserver.service
```

如果这是一个重要的服务，如果失败，您可能希望重新启动。您可以通过向`[Service]`部分添加此标志来实现：

`Restart=on-abort`

`Restart`的其他选项是`on-success`、`on-failure`、`on-abnormal`、`on-watchdog`、`on-abort`或`always`。

## 添加看门狗

看门狗是嵌入式设备中的常见要求：如果关键服务停止工作，通常需要采取措施重置系统。在大多数嵌入式 SoC 中，有一个硬件看门狗，可以通过`/dev/watchdog`设备节点访问。看门狗在启动时使用超时进行初始化，然后必须在该期限内进行复位，否则看门狗将被触发，系统将重新启动。与看门狗驱动程序的接口在内核源代码中的`Documentation/watchdog`中有描述，驱动程序的代码在`drivers/watchdog`中。

如果有两个或更多需要由看门狗保护的关键服务，就会出现问题。`systemd`有一个有用的功能，可以在多个服务之间分配看门狗。

`systemd`可以配置为期望从服务接收定期的保持活动状态的调用，并在未收到时采取行动，换句话说，每个服务的软件看门狗。为了使其工作，您必须向守护程序添加代码以发送保持活动状态的消息。它需要检查`WATCHDOG_USEC`环境变量中的非零值，然后在此期间内调用`sd_notify(false, "WATCHDOG=1")`（建议使用看门狗超时的一半时间）。`systemd`源代码中有示例。

要在服务单元中启用看门狗，向`[Service]`部分添加类似以下内容：

```
WatchdogSec=30s
Restart=on-watchdog
StartLimitInterval=5min
StartLimitBurst=4
StartLimitAction=reboot-force
```

在这个例子中，该服务期望每 30 秒进行一次保持活动状态的检查。如果未能交付，该服务将被重新启动，但如果在五分钟内重新启动超过四次，`systemd`将强制立即重新启动。再次，在`systemd`手册中有关于这些设置的完整描述。

像这样的看门狗负责个别服务，但如果`systemd`本身失败，或者内核崩溃，或者硬件锁定。在这些情况下，我们需要告诉`systemd`使用看门狗驱动程序：只需将`RuntimeWatchdogSec=NN`添加到`/etc/systemd/system.conf`。`systemd`将在该期限内重置看门狗，因此如果`systemd`因某种原因失败，系统将重置。

## 嵌入式 Linux 的影响

`systemd`在嵌入式 Linux 中有许多有用的功能，包括我在这个简要描述中没有提到的许多功能，例如使用切片进行资源控制（参见`systemd.slice(5)`和`systemd.resource-control(5)`的手册页）、设备管理（`udev(7)`）和系统日志记录设施（`journald(5)`）。

您必须权衡其大小：即使只构建了核心组件`systemd`、`udevd`和`journald`，其存储空间也接近 10 MiB，包括共享库。

您还必须记住，`systemd`的开发与内核紧密相关，因此它不会在比`systemd`发布时间早一年或两年的内核上工作。

# 进一步阅读

以下资源提供了有关本章介绍的主题的进一步信息：

+   systemd 系统和服务管理器：[`www.freedesktop.org/wiki/Software/systemd/`](http://www.freedesktop.org/wiki/Software/systemd/)（该页面底部有许多有用的链接）

# 总结

每个 Linux 设备都需要某种类型的`init`程序。如果您正在设计一个系统，该系统只需在启动时启动少量守护程序并在此后保持相对静态，那么 BusyBox`init`就足够满足您的需求。如果您使用 Buildroot 作为构建系统，通常这是一个不错的选择。

另一方面，如果您的系统在启动时或运行时服务之间存在复杂的依赖关系，并且您有存储空间，那么`systemd`将是最佳选择。即使没有复杂性，`systemd`在处理看门狗、远程日志记录等方面也具有一些有用的功能，因此您应该认真考虑它。

很难仅凭其自身的优点支持 System V`init`，因为它几乎没有比简单的 BusyBox`init`更多的优势。尽管如此，它仍将长期存在，仅仅因为它存在。例如，如果您正在使用 Yocto Project 进行构建，并决定不使用`systemd`，那么 System V`init`就是另一种选择。

在减少启动时间方面，`systemd`比 System V`init`更快，但是，如果您正在寻找非常快速的启动，没有什么能比得上简单的 BusyBox`init`和最小的启动脚本。

本章是关于一个非常重要的进程，`init`。在下一章中，我将描述进程的真正含义，它与线程的关系，它们如何合作以及它们如何被调度。如果您想创建一个健壮且易于维护的嵌入式系统，了解这些内容是很重要的。
