# 第九章：管理调用

在本章中，我们将涵盖：

+   关于进程的信息收集

+   终止进程并发送或响应信号

+   Which、whereis、file、whatis 和 loadavg 的解释

+   向用户终端发送消息

+   收集系统信息

+   使用/proc – 收集信息

+   使用 cron 进行调度

+   从 Bash 中写入和读取 MySQL 数据库

+   用户管理脚本

+   批量图像调整和格式转换

# 介绍

GNU/Linux 生态系统由运行的程序、服务、连接的设备、文件系统、用户等组成。全面了解整个系统并根据我们的意愿管理操作系统，是系统管理的主要目的。人们应该掌握常用命令和适当的使用方法，以收集系统信息和管理资源，编写脚本和自动化工具来执行管理任务。本章将介绍几个命令和方法，用于收集关于系统的信息，并利用这些命令来编写管理脚本。

# 关于进程的信息收集

进程是程序的运行实例。计算机上运行多个进程，每个进程都被分配一个称为进程 ID 的唯一标识号。它是一个整数。可以同时执行相同名称的程序的多个实例。但它们都将有不同的进程 ID。进程包括多个属性，例如拥有进程的用户、程序使用的内存量、程序使用的 CPU 量等。本教程将介绍如何收集关于进程的信息。

## 准备工作

与进程管理相关的重要命令是`top`、`ps`和`pgrep`。让我们看看如何收集关于进程的信息。

## 如何做…

`ps`是收集关于进程的信息的重要工具。`ps`提供了有关拥有进程的用户、进程启动时间、用于执行进程的命令路径、进程 ID（PID）、它附加的终端（TTY）、进程使用的内存、进程使用的 CPU 等信息。例如：

```
$ ps
 PID TTY          TIME CMD
 1220 pts/0    00:00:00 bash
 1242 pts/0    00:00:00 ps

```

`ps`命令通常与一组参数一起使用。当没有任何参数运行时，`ps`将显示在当前（TTY）终端上运行的进程。第一列显示进程 ID（PID），第二列是 TTY（终端），第三列是进程启动后经过的时间，最后是 CMD（命令）。

为了显示包含更多信息的更多列，使用`-f`（表示完整）如下：

```
$ ps -f
UID        PID  PPID  C STIME TTY          TIME CMD
slynux    1220  1219  0 18:18 pts/0    00:00:00 -bash
slynux    1587  1220  0 18:59 pts/0    00:00:00 ps -f

```

上述的`ps`命令并不实用，因为它没有提供除了附加到当前终端的进程之外的任何信息。为了获取关于系统上运行的每个进程的信息，添加`-e`（每个）选项。`-ax`（全部）选项也会产生相同的输出。

### 注意

`-x`参数与`-a`一起指定了默认情况下由`ps`施加的去除 TTY 限制。通常，使用`ps`而不带参数会打印仅附加到终端的进程。

运行`ps -e`或`ps –ef`，否则运行`ps -ax`或`ps –axf`：

```
$ ps -e | head
 PID TTY          TIME CMD
1 ?        00:00:00 init
2 ?        00:00:00 kthreadd
3 ?        00:00:00 migration/0
4 ?        00:00:00 ksoftirqd/0
5 ?        00:00:00 watchdog/0
6 ?        00:00:00 events/0
7 ?        00:00:00 cpuset
8 ?        00:00:00 khelper
9 ?        00:00:00 netns

```

这将是一个很长的列表。示例使用`head`过滤输出，所以我们只得到前 10 个条目。

`ps`命令支持显示进程名称和进程 ID 以及其他信息。默认情况下，`ps`将信息显示为不同的列。其中大部分对我们来说并不实用。我们实际上可以使用`-o`标志指定要显示的列。因此，我们可以只打印所需的列。下面将讨论与参数相关的不同参数和`-o`的用法。

为了使用`ps`显示所需的输出列，使用：

```
$ ps [OTHER OPTIONS] -o parameter1,parameter2,parameter3 ..

```

### 注意

使用逗号（,）运算符来分隔`-o`的参数。应该注意的是逗号运算符和下一个参数之间没有空格。大多数情况下，`-o`选项与`-e`（every）选项（`-oe`）结合使用，因为它应该列出系统中运行的每个进程。然而，当与`–o`一起使用某些过滤器时，比如用于列出指定用户拥有的进程的过滤器时，`-e`不会与`–o`一起使用。使用带有过滤器的`-e`将使过滤器无效，并显示所有进程条目。

一个例子如下。这里，`comm`代表 COMMAND，`pcpu`代表 CPU 使用率的百分比：

```
$ ps -eo comm,pcpu | head
COMMAND         %CPU
init             0.0
kthreadd         0.0
migration/0      0.0
ksoftirqd/0      0.0
watchdog/0       0.0
events/0         0.0
cpuset           0.0
khelper          0.0
netns            0.0

```

可以使用`-o`选项的不同参数及其描述如下：

| 参数 | 描述 |
| --- | --- |
| `pcpu` | CPU 百分比 |
| `pid` | 进程 ID |
| `ppid` | 父进程 ID |
| `pmem` | 内存百分比 |
| `comm` | 可执行文件名 |
| `cmd` | 简单命令 |
| `user` | 启动进程的用户 |
| `nice` | 优先级（niceness） |
| `time` | 累积 CPU 时间 |
| `etime` | 进程启动后的经过时间 |
| `tty` | 关联的 TTY 设备 |
| `euid` | 有效用户 |
| `stat` | 进程状态 |

## 还有更多...

让我们通过进程操作命令的其他用法示例。

### top

`top`对于系统管理员来说是一个非常重要的命令。`top`命令默认会输出 CPU 消耗最多的进程列表。该命令的使用如下：

```
$ top

```

它将显示与 CPU 消耗最多的进程相关的几个参数。

### 根据参数对 ps 输出进行排序

`ps`命令的输出可以根据指定的列使用`--sort`参数进行排序。

可以通过使用`+`（升序）或`-`（降序）前缀来指定升序或降序排序。

```
$ ps [OPTIONS] --sort -paramter1,+parameter2,parameter3..

```

例如，要列出前 10 个 CPU 消耗最多的进程，请使用：

```
$ ps -eo comm,pcpu --sort -pcpu | head
COMMAND         %CPU
Xorg             0.1
hald-addon-stor  0.0
ata/0            0.0
scsi_eh_0        0.0
gnome-settings-  0.0
init             0.0
hald             0.0
pulseaudio       0.0
gdm-simple-gree  0.0

```

在这里，进程按 CPU 使用率的降序排序，并且应用`head`来提取前 10 个进程。

我们可以使用`grep`来提取与给定进程名称或其他参数相关的`ps`输出条目。为了找出关于运行 bash 进程的条目，请使用：

```
$ ps -eo comm,pid,pcpu,pmem | grep bash
bash             1255  0.0  0.3
bash             1680  5.5  0.3

```

### 查找给定命令名称时的进程 ID

假设正在执行某个命令的几个实例，我们可能需要识别这些进程的进程 ID。可以通过使用`ps`或`pgrep`命令来找到这些信息。我们可以使用`ps`如下：

```
$ ps -C COMMAND_NAME

```

或者：

```
$ ps -C COMMAND_NAME -o pid=

```

在食谱的前面部分描述了`-o`用户定义格式说明符。但是在这里你可以看到`=`与`pid`附加在一起。这是为了在`ps`的输出中移除标题 PID。为了移除每列的标题，将`=`附加到参数上。例如：

```
$ ps -C bash -o pid=
 1255
 1680

```

该命令列出 bash 进程的进程 ID。

另外，还有一个方便的命令叫做`pgrep`。您应该使用`pgrep`来快速获取特定命令的进程 ID 列表。例如：

```
$ pgrep COMMAND
$ pgrep bash
1255
1680

```

### 注意

`pgrep`只需要命令名称的一部分作为其输入参数来提取 Bash 命令，例如，`pgrep ash`或`pgrep bas`也可以工作。但是`ps`需要您输入确切的命令。

`pgrep`接受许多其他输出过滤选项。为了指定一个分隔符字符来输出，而不是使用换行符作为分隔符，请使用：

```
$ pgrep COMMAND -d DELIMITER_STRING
$ pgrep bash -d ":"
1255:1680

```

指定匹配进程的用户所有者列表如下：

```
$ pgrep -u root,slynux COMMAND

```

在这个命令中，`root`和`slynux`是用户。

返回匹配进程的计数如下：

```
$ pgrep -c COMMAND

```

### 使用 ps 进行真实用户或 ID、有效用户或 ID 的过滤

使用`ps`可以根据指定的真实用户和有效用户名称或 ID 对进程进行分组。指定的参数可以用来通过检查每个条目是否属于特定的有效用户或真实用户的列表来过滤`ps`的输出，并仅显示与它们匹配的条目。可以按以下方式完成：

+   通过使用`-u EUSER1, EUSER2`等来指定有效用户列表

+   通过使用`-U RUSER1, RUSER2`等指定真实用户列表

例如：

```
$ ps -u root -U root -o user,pcpu

```

此命令将显示所有以`root`作为有效用户 ID 和真实用户 ID 运行的进程，并且还将显示用户和 CPU 使用率百分比列。

### 提示

通常，我们会在`-eo`中找到`-o`和`-e`。但是当应用筛选器时，`-o`应该像上面提到的那样单独起作用。

### 用于 ps 的 TTY 筛选器

可以通过指定进程附加到的 TTY 来选择`ps`输出。使用`-t`选项来指定 TTY 列表如下：

```
$ ps -t TTY1, TTY2 ..

```

例如：

```
$ ps -t pts/0,pts/1
 PID TTY          TIME CMD
 1238 pts/0    00:00:00 bash
 1835 pts/1    00:00:00 bash
 1864 pts/0    00:00:00 ps

```

### 有关进程线程的信息

通常，关于进程线程的信息在`ps`输出中是隐藏的。我们可以通过添加`-L`选项来在`ps`输出中显示有关线程的信息。然后它将显示两列 NLWP 和 NLP。NLWP 是进程的线程计数，NLP 是 PS 中每个条目的线程 ID。例如：

```
$ ps -eLf

```

或者：

```
$ ps -eLf --sort -nlwp | head
UID        PID  PPID   LWP  C NLWP STIME TTY          TIME CMD
root       647     1   647  0   64 14:39 ?        00:00:00 /usr/sbin/console-kit-daemon --no-daemon
root       647     1   654  0   64 14:39 ?        00:00:00 /usr/sbin/console-kit-daemon --no-daemon
root       647     1   656  0   64 14:39 ?        00:00:00 /usr/sbin/console-kit-daemon --no-daemon
root       647     1   657  0   64 14:39 ?        00:00:00 /usr/sbin/console-kit-daemon --no-daemon
root       647     1   658  0   64 14:39 ?        00:00:00 /usr/sbin/console-kit-daemon --no-daemon
root       647     1   659  0   64 14:39 ?        00:00:00 /usr/sbin/console-kit-daemon --no-daemon
root       647     1   660  0   64 14:39 ?        00:00:00 /usr/sbin/console-kit-daemon --no-daemon
root       647     1   662  0   64 14:39 ?        00:00:00 /usr/sbin/console-kit-daemon --no-daemon
root       647     1   663  0   64 14:39 ?        00:00:00 /usr/sbin/console-kit-daemon --no-daemon

```

此命令列出了具有最大线程数的 10 个进程。

### 指定输出宽度和要显示的列

我们可以使用用户定义的输出格式说明符`-o`来指定要在`ps`输出中显示的列。另一种指定输出格式的方法是使用“标准”选项。根据您的使用风格进行练习。尝试这些选项：

+   `-f ps –ef`

+   `u ps -e u`

+   `ps ps -e w`（w 代表宽输出）

### 显示进程的环境变量

了解进程依赖的环境变量是我们可能需要的非常有用的信息。进程是否工作可能严重依赖于设置的环境变量。我们可以调试并利用环境数据来解决与进程运行相关的几个问题。

为了列出`ps`条目以及环境变量，请使用：

```
$ ps -eo cmd e

```

例如：

```
$ ps -eo pid,cmd  e | tail -n 3
 1162 hald-addon-acpi: listening on acpid socket /var/run/acpid.socket
 1172 sshd: slynux [priv]
 1237 sshd: slynux@pts/0
 1238 -bash USER=slynux LOGNAME=slynux HOME=/home/slynux PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games MAIL=/var/mail/slynux SHELL=/bin/bash SSH_CLIENT=10.211.55.2 49277 22 SSH_CONNECTION=10.211.55.2 49277 10.211.55.4 22 SSH_TTY=/dev/pts/0 TERM=xterm-color LANG=en_IN XDG_SESSION_COOKIE=d1e96f5cc8a7a3bc3a0a73e44c95121a-1286499339.592429-1573657095

```

这种类型的环境跟踪可以派上用场的一个例子是跟踪 apt-get 软件包管理器的问题。如果您使用 HTTP 代理连接到互联网，您可能需要设置环境变量`http_proxy=host:port`。但有时即使设置了，`apt-get`命令也不会选择代理，因此会返回错误。然后您可以实际查看环境变量并跟踪问题。

我们可能需要一些应用程序使用诸如`crontab`之类的调度工具自动运行。但它可能取决于一些环境变量。假设我们想在指定时间打开一个 GUI 窗口应用程序。我们使用`crontab`在指定时间安排它。但是，如果给出以下条目，您会注意到该应用程序不会在指定时间启动：

```
00 10 * * * /usr/bin/windowapp

```

这是因为窗口应用程序始终依赖于 DISPLAY 环境变量。环境变量需要传递给应用程序。

首先手动运行`windowapp`，然后运行`ps -C windowapp -eo cmd e`。

查找环境变量。在`crontab`中出现命令名称之前加上前缀。问题将得到解决。

将条目修改如下：

```
00 10 * * * DISPLAY=:0 /usr/bin/windowapp

```

`DISPLAY=:0`可以从`ps`输出中获取。

## 另请参阅

+   *使用 cron 进行调度*，解释如何安排任务

# 终止进程并发送或响应信号

终止进程是我们经常遇到的重要任务。有时我们可能需要终止程序的所有实例。命令行提供了几种终止程序的选项。关于类 UNIX 环境中进程的一个重要概念是信号。信号是一种用于中断运行进程以执行某些操作的进程间通信机制。程序的终止也是通过使用信号技术来执行的。本文介绍了信号和信号的使用。

## 准备就绪

信号是 Linux 中可用的进程间机制。我们可以通过使用特定的信号来中断一个进程。每个信号都与一个整数值相关联。当一个进程接收到一个信号时，它会通过执行一个信号处理程序来做出响应。在 Shell 脚本中，也可以发送和接收信号，并根据信号做出响应。`KILL`是用于终止进程的信号。诸如*Ctrl* + *C*、*Ctrl* + *Z*之类的事件也是信号的一种。`kill`命令用于向进程发送信号，`trap`命令用于处理接收到的信号。

## 如何做...

为了列出所有可用的信号，使用：

```
$ kill -l

```

它将打印信号编号和信号名称。

终止一个进程如下：

```
$ kill PROCESS_ID_LIST

```

`kill`命令默认发出 TERM 信号。进程 ID 列表应该用空格作为进程 ID 之间的分隔符来指定。

为了指定要通过`kill`命令发送的信号，使用：

```
$ kill -s SIGNAL PID

```

`SIGNAL`参数可以是信号名称或信号编号。虽然有许多为不同目的指定的信号，但我们通常只使用少数信号。它们如下：

+   `SIGHUP 1`——检测控制进程或终端死机

+   `SIGINT 2`——在按下*Ctrl* + *C*时发出的信号

+   `SIGKILL 9`——用于强制终止进程的信号

+   `SIGTERM -15`——默认用于终止进程的信号

+   `SIGTSTP 20`——在按下*Ctrl* + *Z*时发出的信号

我们经常使用强制终止进程。为了强制终止一个进程，使用：

```
$ kill -s SIGKILL PROCESS_ID

```

或者：

```
$ kill -9 PROCESS_ID

```

## 还有更多...

让我们浏览一下用于终止和发送信号的其他命令。

### kill 命令的系列

`kill`命令以进程 ID 作为参数。`kill`系列中还有其他一些命令，它们接受命令名称作为参数并向进程发送信号。

`killall`命令按名称终止进程如下：

```
$ killall process_name

```

为了通过名称发送信号给一个进程使用：

```
$ killall -s SIGNAL process_name

```

为了通过名称强制终止进程使用：

```
$ killall -9 process_name

```

例如：

```
$ killall -9 gedit

```

通过使用指定的用户拥有的名称来指定进程的名称：

```
$ killall -u USERNAME process_name

```

为了在杀死进程之前进行交互式询问，使用`killall`命令和`-i`参数。

`pkill`命令类似于`kill`命令，但默认情况下接受进程名称而不是进程 ID。例如：

```
$ pkill process_name
$ pkill -s SIGNAL process_name

```

`SIGNAL`是信号编号。`SIGNAL`名称不支持`pkill`。

它提供了许多与`kill`命令相同的选项。查看`pkill`手册以获取更多详细信息。

### 捕获和响应信号

`trap`是一个用于在脚本中为信号分配信号处理程序的命令。一旦使用`trap`命令为信号分配了一个函数，当脚本运行并接收到一个信号时，该函数将在接收到相应信号时被执行。

语法如下：

```
trap 'signal_handler_function_name' SIGNAL LIST

```

`SIGNAL LIST`由空格分隔。它可以是信号编号或信号名称。

让我们编写一个 Shell 脚本来响应`SIGINT`信号：

```
#/bin/bash
#Filename: sighandle.sh
#Description: Signal handler 

function handler()
{
  echo Hey, received signal : SIGINT
}

echo My process ID is $$
# $$ is a special variable that returns process ID of current process/script
trap 'handler' SIGINT
#handler is the name of the signal handler function for SIGINT signal

while true;
do
  sleep 1
done
```

在终端中运行这个脚本。当脚本运行时，如果按下*Ctrl* + *C*，它将通过执行与之关联的信号处理程序来显示消息。*Ctrl* + *C*是`SIGINT`信号。

`while`循环用于通过使用无限循环使进程保持运行而不终止。因此，进程被无限地保持运行，以便它可以响应由另一个进程异步发送给进程的信号。用于无限保持进程运行的循环通常被称为事件循环。如果无限循环不可用，脚本将在执行语句后终止。但对于信号处理程序脚本，它必须等待并响应信号。

我们可以通过使用`kill`命令和脚本的进程 ID 向脚本发送信号：

```
$ kill -s SIGINT PROCESS_ID

```

上述脚本的`PROCESS_ID`在执行时将被打印出来。或者您可以使用`ps`命令找到它

如果没有为信号指定信号处理程序，它将调用操作系统分配的默认信号处理程序。通常，按下 *Ctrl* + *C* 将终止程序，因为操作系统提供的默认处理程序将终止进程。但是在这里定义的自定义处理程序指定了收到信号时的自定义操作。

我们可以为任何可用的信号（`kill -l`）定义信号处理程序，使用 `trap` 命令。还可以为多个信号设置单个信号处理程序。

# which、whereis、file、whatis 和 loadavg 解释

这个配方旨在解释我们遇到的一些命令。了解这些命令对用户有帮助。

## 如何做...

让我们逐个命令及其用法示例。

+   **which**

`which` 命令用于查找命令的位置。我们在终端中键入命令时，并不知道可执行文件存储的位置。

当我们键入一个命令时，终端会在一组位置中查找该命令，并在找到位置时执行可执行文件。这组位置是使用环境变量 `PATH` 指定的。例如：

```
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games

```

我们可以导出 `PATH` 并在键入命令名称时添加我们自己的位置以进行搜索。例如，要将 `/home/slynux/bin` 添加到 `PATH`，请使用以下命令：

```
$ export PATH=$PATH:/home/slynux/bin
# /home/slynux/bin is added to PATH

```

`which` 命令输出给定参数的命令位置。例如：

```
$ which ls
/bin/ls

```

+   **whereis**

`whereis` 类似于 `which` 命令。但它不仅返回命令的路径，还会打印 manpage 的位置（如果有的话），以及命令的源代码路径（如果有的话）。例如：

```
$ whereis ls
ls: /bin/ls /usr/share/man/man1/ls.1.gz

```

+   **file**

`file` 命令是一个有趣且经常使用的命令。它用于确定文件类型。

```
$ file FILENAME

```

这将打印有关文件的详细信息，包括文件类型。

一个例子如下：

```
$ file /bin/ls
/bin/ls: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.15, stripped

```

+   **whatis**

`whatis` 命令输出给定参数的命令的一行描述。它从 manpage 中解析信息。例如：

```
$ whatis ls
ls (1)               - list directory contents

```

### 提示

**apropos**

有时我们需要搜索与某个单词相关的命令是否存在。然后我们可以在命令的 manpages 中搜索字符串。为此，我们可以使用：

`apropos COMMAND`

+   **平均负载**

平均负载是运行系统的总负载的重要参数。它指定系统上可运行进程的平均数量。它由三个值指定。第一个值表示一分钟的平均值，第二个表示五分钟的平均值，第三个表示 15 分钟的平均值。

可以通过运行 `uptime` 来获得。例如：

```
$ uptime
 12:40:53 up  6:16,  2 users,  load average: 0.00, 0.00, 0.00

```

# 向用户终端发送消息

系统管理员可能需要向网络上所有机器的每个用户或指定用户的终端屏幕发送消息。这个配方是执行这项任务的指南。

## 准备工作

`wall` 是一个用于在所有已登录用户的终端上写消息的命令。它可以用于向服务器或多个访问机器上的所有已登录用户传达消息。有时向所有用户发送消息可能不太有用。我们可能需要向特定用户或特定终端发送消息。终端在 Linux 系统中被视为设备，因此这些打开的终端将在 `/dev/pts/` 下有一个相应的设备节点文件。向特定设备写入数据将在相应的终端上显示消息。

## 如何做...

为了向所有用户和所有已登录的终端广播消息，请使用：

```
$ cat message | wall

```

或：

```
$ wall < message
Broadcast Message from slynux@slynux-laptop
 (/dev/pts/1) at 12:54 ...

This is a message

```

消息概述将显示谁发送了消息（哪个用户和哪个主机）。如果其他用户发送消息，则消息将被显示到当前终端，只有在启用了“写消息”选项时才会显示。默认情况下，在大多数发行版中，“写消息”默认是启用的。如果消息的发送者是 root，则消息将显示在屏幕上，无论用户是否启用或禁用“写消息”选项。

为了启用写入消息，请使用：

```
$ mesg y

```

为了禁用写入消息，请使用：

```
$ mesg n

```

让我们编写一个专门向给定用户终端发送消息的脚本：

```
#/bin/bash
#Filename: message_user.sh
#Description: Script to send message to specified user logged terminals.
USER=$1

devices=`ls /dev/pts/* -l | awk '{ print $3,$9 }' | grep $USER | awk '{ print $2 }'`
for dev in $devices;
do
  cat /dev/stdin > $dev
done
```

按以下方式运行脚本：

```
./message_user.sh USERNAME < message.txt
# Pass message through stdin and username as argument

```

输出将如下所示：

```
$ cat message.txt
A message to slynux. Happy Hacking!
# ./message_user.sh slynux  < message.txt
# Run message_user.sh as root since the message is to be send to a specifc user.

```

现在，slynux 的终端将接收消息文本。

## 它是如何工作的...

`/dev/pts`目录将包含与系统上每个已登录终端对应的字符设备。我们可以通过查看设备文件的所有者来找出谁登录到哪个终端。`ls -l`输出将包含所有者名称和设备路径。这些信息是通过使用`awk`提取的。然后它使用`grep`仅提取对应于指定用户的行。用户名作为脚本的第一个参数被接受并存储为变量 USER。然后制作给定用户的终端列表。使用`for`循环来迭代每个设备路径。`/dev/stdin`将包含传递给当前进程的标准输入数据。因此，通过读取`/dev/stdin`，数据被读取并重定向到相应的终端（TTY）设备。因此消息被显示。

# 收集系统信息

从命令行收集有关当前系统的信息非常重要，用于记录系统数据。不同的系统信息数据包括主机名、内核版本、Linux 发行版名称、CPU 信息、内存信息、磁盘分区信息等。本教程将向您展示在 Linux 系统中收集有关系统信息的不同来源。

## 如何做...

为了打印当前系统的主机名，使用：

```
$ hostname

```

或者：

```
$ uname -n

```

通过使用以下方式打印有关 Linux 内核版本、硬件架构等的详细信息：

```
$ uname -a

```

为了打印内核版本，使用：

```
$ uname -r

```

按以下方式打印机器类型：

```
$ uname –m

```

为了打印有关 CPU 详细信息，使用：

```
$ cat /proc/cpuinfo

```

为了提取处理器名称，使用：

```
$ cat /proc/cpuinfo | head -n 5 | tail -1

```

第五行包含处理器名称。因此，首先提取前五行。然后提取最后一行以打印处理器名称。

按以下方式打印有关内存或 RAM 的详细信息：

```
$ cat /proc/meminfo

```

按以下方式打印系统上可用的总内存（RAM）：

```
$ cat /proc/meminfo  | head -1
MemTotal:        1026096 kB

```

为了列出系统上可用的分区信息，使用：

```
$ cat /proc/partitions

```

或者：

```
$ fdisk -l

```

按以下方式获取有关系统的完整详细信息：

```
$ lshw

```

# 使用/proc – 收集信息

`/proc`是 GNU/Linux 操作系统上可用的内存伪文件系统。它被引入以提供一个接口，从用户空间读取几个系统参数。这非常有趣，我们可以从中收集大量信息。让我们看看`proc`文件系统提供的一些功能。

## 如何做...

如果您查看`/proc`，您可以看到几个文件和目录。其中一些在本章的另一个教程中已经解释过。您可以简单地`cat`文件和子目录中的文件，以获取信息。所有这些都是格式良好的文本。

在系统上运行的每个进程都将在`/proc`中有一个目录。在`/proc`中进程的目录名称与该进程的进程 ID 相同。

假设对于 Bash，进程 ID 为 4295（`pgrep bash`），`/proc/4295`将存在。与该进程对应的每个目录都将包含有关该进程的大量信息。`/proc/PID`中的一些重要文件如下。

+   `environ`—包含与该进程关联的环境变量。

通过`cat /proc/4295/environ`，我们可以显示传递给该进程的所有环境变量。

+   `cwd`—是进程的工作目录的符号链接。

+   `exe`—是当前进程的运行可执行文件的符号链接。

```
$ readlink /proc/4295/exe
/bin/bash

```

+   `fd`—是由进程使用的文件描述符条目组成的目录。

# 使用 cron 进行调度

在给定时间或给定时间间隔执行脚本是一个常见的需求。GNU/Linux 系统配备了不同的实用程序来安排任务。`cron`就是这样一个实用程序，它允许任务在系统的后台通过`cron`守护程序定期自动运行。`cron`实用程序使用一个名为“cron 表”的文件，其中存储了要执行的脚本或命令的时间表以及它们要执行的时间。这是一个非常有用的实用程序。一个常见的用法是在空闲时间（某些 ISP 提供免费使用的时间，通常是在大多数人睡觉的夜间）安排从互联网下载文件。用户不需要在夜间醒来开始下载。用户可以编写一个 cron 条目并安排下载。您还可以安排在免费使用时间结束时自动断开互联网连接并关闭系统。

## 准备工作

cron 调度实用程序默认随所有 GNU/Linux 发行版一起提供。一旦我们编写了 cron 表条目，命令将在指定的执行时间执行。`crontab`命令用于向 cron 调度域添加调度条目。cron 调度是一个简单的文本文件。每个用户都有自己的 cron 调度。cron 调度通常称为 cron 作业。

## 如何做…

为了安排任务，我们应该知道编写 cron 表的格式。cron 作业指定要执行的脚本或命令的路径以及要执行的时间。每个 cron 表由以下顺序的六个部分组成：

+   分钟（0-59）

+   小时（0-23）

+   天（1-31）

+   月（1-12）

+   工作日（0-6）

+   COMMAND（要在指定时间执行的脚本或命令）

前五个部分指定要执行命令的实例的时间。还有一些其他选项可以指定时间表。

星号（`*`）用于指定命令应在每个时间实例执行。也就是说，如果在 cron 作业的小时字段中写入`*`，则命令将每小时执行一次。同样，如果您想要在特定时间段的多个实例执行命令，请在相应的时间字段中用逗号分隔指定时间段（例如，要在第五分钟和第十分钟运行命令，请在分钟字段中输入`5,10`）。我们还有另一个很好的选项，可以在特定的时间间隔运行命令。在分钟字段中使用`*/5`以在每五分钟运行一次命令。我们可以将此应用于任何时间字段。cron 表条目可以包含一个或多个 cron 作业的行。cron 表条目中的每一行都是一个作业。例如：

+   让我们写一个示例`crontab`条目以进行说明：

```
02 * * * * /home/slynux/test.sh
```

此 cron 作业将在所有小时的所有天的第二分钟执行`test.sh`脚本。

+   为了在所有天的第五、第六和第七小时运行脚本，使用：

```
00 5,6,7 * * /home/slynux/test.sh
```

+   在每个星期日的每个小时执行`script.sh`如下：

```
00 */12 * * 0 /home/slynux/script.sh
```

+   每天凌晨 2 点关闭计算机如下：

```
00 02 * * * /sbin/shutdown -h
```

现在，让我们看看如何安排 cron 作业。您可以以多种方式执行`crontab`命令以安排脚本。

当您手动运行`crontab`时，使用`-e`选项输入 cron 作业：

```
$ crontab –e
02 02 * * * /home/slynux/script.sh

```

输入`crontab -e`后，将打开默认的文本编辑器（通常是 vi），用户可以输入 cron 作业并保存。此 cron 作业将按指定的时间间隔进行调度和执行。

通常在脚本中调用`crontab`命令进行任务调度时，我们通常使用另外两种方法：

1.  创建一个文本文件（例如`task.cron`）并编写 cron 作业。

然后使用文件名作为命令参数运行`crontab`：

```
$ crontab task.cron

```

1.  通过使用下一种方法，我们可以在不创建单独文件的情况下指定内联的 cron 作业。例如：

```
crontab<<EOF
02 * * * * /home/slynux/script.sh
EOF
```

cron 作业需要在`crontab<<EOF 和 EOF`之间编写。

使用`crontab`命令执行 Cron 作业时具有特权。如果需要执行需要更高特权的命令，比如关机命令，需要以 root 身份运行`crontab`命令。

在 cron 作业中指定的命令使用完整路径写入。这是因为 cron 作业执行的环境与我们在终端上执行的环境不同。因此`PATH`环境变量可能未设置。如果您的命令需要设置某些环境变量才能运行，您应该显式设置环境变量。

## 还有更多…

`crontab`命令还有更多选项。让我们看一些。

### 指定环境变量

许多命令要求环境变量正确设置才能执行。我们可以通过在用户的 cron 表中插入一个带有变量赋值语句的行来设置环境变量。

例如，如果您使用代理服务器连接到互联网，要安排使用互联网的命令，您必须设置 HTTP 代理环境变量`http_proxy`。可以按以下方式完成：

```
crontab<<EOF
http_proxy=http://192.168.03:3128
00 * * * * /home/slynux/download.sh
EOF
```

### 查看 cron 表

我们可以使用`-l`选项列出现有的 cron 作业：

```
$ crontab –l
02 05 * * * /home/user/disklog.sh

```

`crontab -l`列出当前用户的 cron 表中的现有条目。

我们还可以通过使用`-u`选项指定用户名来查看其他用户的 cron 表，如下所示：

```
$ crontab –l –u slynux
09 10 * * * /home/slynux/test.sh

```

当使用`-u`选项获取更高特权时，应以 root 身份运行。

### 删除 cron 表

我们可以使用`-r`选项删除当前用户的 cron 表：

```
$ crontab –r

```

为了删除另一个用户的 cron 表，使用：

```
# crontab –u slynux –r

```

以 root 身份运行以获得更高特权。

# 从 Bash 中写入和读取 MySQL 数据库

MySQL 是一个广泛使用的数据库系统。通常，MySQL 数据库用作使用 PHP、Python、C++等语言编写的应用程序的存储系统。从 shell 脚本访问和操作 MySQL 数据库将会很有趣。我们可以编写脚本将文本文件或 CSV（逗号分隔值）中的内容写入表中，并与 MySQL 数据库交互以读取和操作数据。例如，我们可以通过从 shell 脚本运行查询来读取存储在留言板程序数据库中的所有电子邮件地址。在本教程中，我们将看到如何从 Bash 中读取和写入 MySQL 数据库。例如，这是一个示例问题：

我有一个包含学生详细信息的 CSV 文件。我需要将文件内容插入数据库表中。根据这些数据，我需要为每个部门生成一个单独的排名表。

## 准备工作

为了处理 MySQL 数据库，您应该在系统上安装 mysql-server 和 mysql-client 软件包。这些工具不会默认随 Linux 发行版提供。由于 MySQL 带有用于身份验证的用户名和密码，您应该有一个用户名和密码来运行脚本。

## 如何做…

可以使用 Bash 实用程序（如`sort`、`awk`等）解决上述问题。或者，我们可以通过使用 SQL 数据库表来解决。我们将为创建数据库和表、将学生数据插入表中以及从表中读取和显示处理后的数据编写三个脚本。

创建数据库和表的脚本如下：

```
#!/bin/bash
#Filename: create_db.sh
#Description: Create MySQL database and table

USER="user"
PASS="user"

mysql -u $USER -p$PASS <<EOF 2> /dev/null
CREATE DATABASE students;
EOF

[ $? -eq 0 ] && echo Created DB || echo DB already exist 

mysql -u $USER -p$PASS students <<EOF 2> /dev/null
CREATE TABLE students(
id int,
name varchar(100),
mark int,
dept varchar(4)
);
EOF

[ $? -eq 0 ] && echo Created table students || echo Table students already exist 

mysql -u $USER -p$PASS students <<EOF
DELETE FROM students;
EOF
```

插入数据到表的脚本如下：

```
#!/bin/bash
#Filename: write_to_db.sh
#Description: Read from CSV and write to MySQLdb

USER="user"
PASS="user"

if [ $# -ne 1 ];
then
  echo $0 DATAFILE
  echo
  exit 2
fi

data=$1

while read line;
do

  oldIFS=$IFS
  IFS=,
  values=($line)
  values[1]="\"`echo ${values[1]} | tr ' ' '#' `\""
  values[3]="\"`echo ${values[3]}`\""

  query=`echo ${values[@]} | tr ' #' ', ' `
  IFS=$oldIFS

  mysql -u $USER -p$PASS students <<EOF
INSERT INTO students VALUES($query);
EOF

done< $data
echo Wrote data into DB
```

从数据库查询的脚本如下：

```
#!/bin/bash
#Filename: read_db.sh
#Description: Read from the database

USER="user"
PASS="user"

depts=`mysql -u $USER -p$PASS students <<EOF | tail -n +2
SELECT DISTINCT dept FROM students;
EOF`

for d in $depts;
do

echo Department : $d

result="`mysql -u $USER -p$PASS students <<EOF
SET @i:=0;
SELECT @i:=@i+1 as rank,name,mark FROM students WHERE dept="$d" ORDER BY mark DESC;
EOF`"

echo "$result"
echo

done
```

输入 CSV 文件（`studentdata.csv`）的数据如下：

```
1,Navin M,98,CS
2,Kavya N,70,CS
3,Nawaz O,80,CS
4,Hari S,80,EC
5,Alex M,50,EC
6,Neenu J,70,EC
7,Bob A,30,EC
8,Anu M,90,AE
9,Sruthi,89,AE
10,Andrew,89,AE

```

按以下顺序执行脚本：

```
$ ./create_db.sh 
Created DB
Created table students

$ ./write_to_db.sh studentdat.csv
Wrote data into DB

$ ./read_db.sh 
Department : CS
rank  name  mark
1  Navin M  98
2  Nawaz O  80
3  Kavya N  70

Department : EC
rank  name  mark
1  Hari S  80
2  Neenu J 70
3  Alex M  50
4  Bob A   30

Department : AE
rank  name  mark
1  Anu M    90
2  Sruthi   89
3  Andrew   89

```

## 它是如何工作的…

我们现在将逐个查看上述脚本的解释。第一个脚本`create_db.sh`用于创建名为`students`的数据库和其中的名为`students`的表。我们需要 MySQL 用户名和密码来访问或修改 DBMS 中的数据。变量`USER`和`PASS`用于存储用户名和密码。`mysql`命令用于 MySQL 操作。`mysql`命令可以通过`-u`指定用户名，并通过`-pPASSWORD`指定密码。`mysql`命令的另一个命令参数是数据库名称。如果将数据库名称指定为`mysql`命令的参数，它将用于数据库操作，否则我们必须在 SQL 查询中明确指定要使用的数据库名称。`mysql`命令接受要通过标准输入（`stdin`）执行的查询。通过`<<EOF`方法通过`stdin`提供多行的便捷方式。出现在`<<EOF`和`EOF`之间的文本将作为标准输入传递给`mysql`。在`CREATE DATABASE`查询中，我们已将`stderr`重定向到`/dev/null`，以防止显示错误消息。此外，在表创建查询中，我们已将`stderr`重定向到`/dev/null`以忽略任何错误。然后，我们使用退出状态变量`$?`检查`mysql`命令的退出状态，以了解表或数据库是否已经存在。如果数据库或表已经存在，则显示消息通知。否则，我们将创建它们。

下一个脚本`write_to_db.sh`接受一个学生数据 CSV 文件的文件名。我们使用`while`循环读取 CSV 文件的每一行。因此，在每次迭代中，将收到一个逗号分隔值的行。然后，我们需要将行中的值组成一个 SQL 查询。为此，将数据项存储在逗号分隔行中的最简单方法是使用数组。我们知道数组赋值的形式是`array=(val1 val2 val3)`。这里空格字符是**内部字段分隔符**（**IFS**）。我们有一个逗号分隔值的行，因此通过将 IFS 更改为逗号，我们可以轻松地将值分配给数组（`IFS=,`）。逗号分隔行中的数据项是`id`，`name`，`mark`和`department`。`id`和`mark`是整数值，而`name`和`dept`是字符串（字符串必须用引号括起来）。另外，名字中可能包含空格字符。空格可能与内部字段分隔符冲突。因此，我们应该用某个字符（`#`）替换名字中的空格，并在构建查询后再替换它。为了引用字符串，数组中的值用`\"`前缀和后缀。`tr`用于将名字中的空格替换为`#`。最后，通过将空格字符替换为逗号并将`#`替换为空格来形成查询，并执行此查询。

第三个脚本`read_db.sh`用于查找部门并打印每个部门学生的排名列表。第一个查询用于查找部门的不同名称。我们使用`while`循环来遍历每个部门，并运行查询以按最高分显示学生详细信息。`SET @i=0`是一个 SQL 构造，用于设置变量`i=0`。在每一行上，它都会递增，并显示为学生的排名。

# 用户管理脚本

GNU/Linux 是一个多用户操作系统。许多用户可以登录并同时执行多项活动。有几个管理任务是通过用户管理来处理的。任务包括为用户设置默认 shell，禁用用户帐户，禁用 shell 帐户，添加新用户，删除用户，设置密码，为用户帐户设置到期日期等。本文旨在编写一个用户管理工具，可以处理所有这些任务。

## 如何做…

让我们来看一下用户管理脚本：

```
#!/bin/bash
#Filename: user_adm.sh
#Description: A user administration tool

function usage()
{
  echo Usage:
  echo Add a new user
  echo $0 -adduser username password
  echo
  echo Remove an existing user
  echo $0 -deluser username
  echo
  echo Set the default shell for the user
  echo $0 -shell username SHELL_PATH
  echo
  echo Suspend a user account
  echo $0 -disable username
  echo
  echo Enable a suspended user account
  echo $0 -enable username
  echo
  echo Set expiry date for user account
  echo $0 -expiry DATE 
  echo
  echo Change password for user account
  echo $0 -passwd username
  echo
  echo Create a new user group
  echo $0 -newgroup groupname
  echo
  echo Remove an existing user group
  echo $0 -delgroup groupname
  echo
  echo Add a user to a group
  echo $0 -addgroup username groupname
  echo
  echo Show details about a user
  echo $0 -details username
  echo
  echo Show usage
  echo $0 -usage
  echo

  exit
}

if [ $UID -ne 0 ];
then
  echo Run $0 as root.
  exit 2
fi

case $1 in

  -adduser) [ $# -ne 3 ] && usage ; useradd $2 -p $3 -m ;; 
  -deluser) [ $# -ne 2 ] && usage ; deluser $2 --remove-all-files;;
  -shell)    [ $# -ne 3 ] && usage ; chsh $2 -s $3 ;;
  -disable) [ $# -ne 2 ] && usage ; usermod -L $2 ;; 
  -enable) [ $# -ne 2 ] && usage ; usermod -U $2  ;;
  -expiry) [ $# -ne 3 ] && usage ; chage $2 -E $3 ;;
  -passwd) [ $# -ne 2 ] && usage ; passwd $2 ;;
  -newgroup) [ $# -ne 2 ] && usage ; addgroup $2 ;;
  -delgroup) [ $# -ne 2 ] && usage ; delgroup $2 ;;
  -addgroup) [ $# -ne 3 ] && usage ; addgroup $2 $3 ;;
  -details) [ $# -ne 2 ] && usage ; finger $2 ; chage -l $2 ;;
  -usage) usage ;;
  *) usage ;;
esac
```

示例输出如下：

```
# ./user_adm.sh -details test
Login: test                 Name: 
Directory: /home/test                 Shell: /bin/sh
Last login Tue Dec 21 00:07 (IST) on pts/1 from localhost
No mail.
No Plan.
Last password change          : Dec 20, 2010
Password expires          : never
Password inactive         : never
Account expires             : Oct 10, 2010
Minimum number of days between password change    : 0
Maximum number of days between password change    : 99999
Number of days of warning before password expires  : 7

```

## 它是如何工作的…

`user_adm.sh`脚本可用于执行许多用户管理任务。您可以按照`usage()`文本使用脚本。定义了一个`usage()`函数，用于显示如何使用脚本以及用户给出的任何参数出错或运行了`–usage`参数时如何执行脚本。使用`case`语句来匹配命令参数并根据匹配情况执行相应的命令。`user_adm.sh`脚本的有效命令选项包括：`-adduser`、`-deluser`、`-shell`、`-disable`、`-enable`、`-expiry`、`-passwd`、`-newgroup`、`-delgroup`、`-addgroup`、`-details`和`-usage`。当匹配到`*)` case 时，意味着是错误的选项，因此调用`usage()`。对于每个匹配情况，我们使用`[ $# -ne 3 ] && usage`。用于检查参数的数量。如果命令参数的数量不等于所需数量，则调用`usage()`函数，脚本将在不执行更多操作的情况下退出。为了运行用户管理命令，脚本需要以 root 身份运行。因此，会检查用户 ID 0（root 用户 ID 为 0）。如果用户 ID 为非零，则表示正在以非 root 身份执行。因此，会显示一个以 root 身份运行的消息，并退出脚本。

让我们逐一解释每种情况：

+   `-useradd`：

`useradd`命令可用于创建新用户。它的语法是：

```
useradd USER –p PASSWORD
```

`-m`选项用于创建主目录

还可以使用`–c FULLNAME`选项提供用户的全名。

+   `-deluser`:

`deluser`命令可用于删除用户。语法是：

```
deluser USER
```

`--remove-all-files`用于删除与用户相关的所有文件，包括主目录。

+   `-shell`：

`chsh`命令用于更改用户的默认 shell。语法是：

```
chsh USER –s SHELL
```

+   `-disable`和`–enable`：

`usermod`命令用于操纵与用户帐户相关的多个属性。

`usermod –L USER`锁定用户帐户，`usermod –U USER`解锁用户帐户。

+   `-expiry`：

`chage`命令用于操纵用户帐户到期信息。语法是：

`chage –E DATE`

还有其他选项如下：

+   `-m MIN_DAYS`（将密码更改之间的最小天数设置为`MIN_DAYS`）

+   `-M MAX_DAYS`（设置密码有效的最大天数）

+   `-W WARN_DAYS`（设置密码更改所需的警告天数）

+   `-passwd`：

`passwd`命令用于更改用户的密码。语法是：

`passwd USER`

该命令将提示输入新密码。

+   `-newgroup`和`addgroup`：

`addgroup`命令将在系统中添加一个新的用户组。语法是：

`addgroup GROUP`

要将现有用户添加到组中，请使用：

`addgroup USER GROUP`

`-delgroup`

`delgroup`命令将删除用户组。语法是：

`delgroup GROUP`

+   `-details`：

`finger USER`命令将显示用户的用户信息，其中包括用户主目录路径、上次登录时间、默认 shell 等。`chage –l`命令将显示用户帐户到期信息。

# 批量图像调整大小和格式转换

我们所有人都使用数码相机并从相机以及互联网下载照片。当我们需要处理大量图像文件时，我们可以使用脚本轻松地对文件进行批量操作。我们经常遇到的一个常规任务是调整文件大小。此外，还会用到格式转换，将一种图像格式转换为另一种格式（例如，将 JPEG 转换为 PNG）。当我们从相机下载图片时，大分辨率图片会占用大量空间。但我们可能需要尺寸较小的图片，以便在互联网上存储和发送电子邮件。因此，我们将其调整为较低的分辨率。本文将讨论如何使用脚本进行图像管理。

## 准备工作

**Imagemagick**是一个优秀的图像处理工具，可以处理多种图像格式和不同的构造，并提供丰富的选项。大多数 GNU/Linux 发行版都没有预装 Imagemagick。您需要手动安装该软件包。`convert`是我们经常使用的命令。

## 如何做…

为了将一种图像格式转换为另一种图像格式，请使用：

```
$ convert INPUT_FILE OUTPUT_FILE

```

例如：

```
$ convert file1.png file2.png

```

我们可以通过指定缩放百分比或指定输出图像的宽度和高度来将图像大小调整为指定的图像大小。

通过以下方式指定`WIDTH`或`HEIGHT`来调整图像大小：

```
$ convert image.png -resize WIDTHxHEIGHT image.png

```

例如：

```
$ convert image.png -resize 1024x768 image.png

```

需要提供`WIDTH`或`HEIGHT`中的一个，以便另一个将自动计算并调整大小，以保持图像大小比例：

```
$ convert image.png -resize WIDTHx image.png

```

例如：

```
$ convert image.png -resize 1024x image.png

```

通过指定百分比缩放因子来调整图像大小，如下所示：

```
$ convert image.png -resize "50%" image.png

```

让我们看一个图像管理的脚本：

```
#!/bin/bash
#Filename: image_help.sh
#Description: A script for image management

if [ $# -ne 4 -a $# -ne 6 -a $# -ne 8 ];
then
  echo Incorrect number of arguments
  exit 2
fi

while [ $# -ne 0 ];
do

  case $1 in
  -source) shift; source_dir=$1 ; shift ;;
  -scale) shift; scale=$1 ; shift ;;
  -percent) shift; percent=$1 ; shift ;;
  -dest) shift ; dest_dir=$1 ; shift ;;
  -ext) shift ; ext=$1 ; shift ;;
  *) echo Wrong parameters; exit 2 ;;
  esac;

done

for img in `echo $source_dir/*` ;
do
  source_file=$img
  if [[ -n $ext ]];
  then
    dest_file=${img%.*}.$ext
  else
    dest_file=$img
  fi

  if [[ -n $dest_dir ]];
  then
    dest_file=${dest_file##*/}
    dest_file="$dest_dir/$dest_file"
  fi

  if [[ -n $scale ]];
  then
    PARAM="-resize $scale"
  elif [[ -n $percent ]];
  then
    PARAM="-resize $percent%"	
  fi

  echo Processing file : $source_file
  convert $source_file $PARAM $dest_file

done
```

以下是一个示例输出，将目录`sample_dir`中的图像缩放到`20%`大小：

```
$ ./image_help.sh -source sample_dir -percent 20%
Processing file :sample/IMG_4455.JPG
Processing file :sample/IMG_4456.JPG
Processing file :sample/IMG_4457.JPG
Processing file :sample/IMG_4458.JPG

```

为了将图像缩放到宽度 1024，请使用：

```
$ ./image_help.sh -source sample_dir –scale 1024x

```

将文件更改为 PNG 格式，方法是在上述命令中添加`-ext png`。

按照以下方式指定目标目录来缩放或转换文件：

```
$ ./image_help.sh -source sample -scale 50% -ext png -dest newdir
# newdir is the new destination directory

```

## 工作原理…

上述`image_help.sh`脚本可以接受多个命令行参数，例如`-source`，`-percent`，`-scale`，`-ext`和`-dest`。以下是对每个参数的简要解释：

+   `-source`参数用于指定图像的源目录。

+   `-percent`参数用于指定缩放百分比，`-scale`用于指定缩放宽度和高度。

+   要么使用`-percent`，要么使用`-scale`。它们两者不会同时出现。

+   `-ext`参数用于指定目标文件格式。`-ext`是可选的；如果未指定，将不执行格式转换。

+   `-dest`参数用于指定缩放或转换图像文件的目标目录。`-dest`是可选的。如果未指定`-dest`，目标目录将与源目录相同。在脚本的第一步中，它检查给定给脚本的命令参数数量是否正确。可以出现 4 个、6 个或 8 个参数。

现在，通过使用`while`循环和`case`语句，我们将解析与变量对应的命令行参数。`$#`是一个特殊变量，返回参数的数量。`shift`命令将命令参数向左移动一个位置，这样在每次执行`shift`时，我们可以通过相同的`$1`变量访问一个一个的命令参数，而不是使用`$1`，`$2`，`$3`等。`case`语句匹配`$1`的值。这类似于 C 编程语言中的 switch 语句。当匹配到一个 case 时，执行相应的语句。每个匹配的 case 语句以`;;`结束。一旦所有参数都被解析为变量`percent`，`scale`，`source_dir`，`ext`和`dest_dir`，就会使用`for`循环来遍历源目录中每个文件的路径，并执行相应的操作来转换文件。

如果变量`ext`已定义（如果命令参数中给出了`-ext`），则将目标文件的扩展名从`source_file.extension`更改为`source_file.$ext`。在下一条语句中，它检查是否提供了`-dest`参数。如果指定了目标目录，则通过使用文件名切片，将源路径中的目录替换为目标目录来构建目标文件路径。在下一条语句中，它为`convert`命令构建参数以执行调整大小（`-resize widthx`或`-resize perc%`）。参数构建完成后，使用正确的参数执行`convert`命令。

## 另请参阅

+   基于第二章的扩展名切片，解释了如何提取文件名的部分
