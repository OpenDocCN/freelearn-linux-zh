# 第四章：故障排除性能问题

在第三章中，我们通过使用第一章中介绍的故障排除方法论，以及第二章中找到的几个基本故障排除命令和资源，来解决了 Web 应用程序的问题。

# 性能问题

在本章中，我们将继续在第三章中涵盖的情景，我们是一家新公司的新系统管理员。当我们到达开始我们的一天时，一位同事要求我们调查一个服务器变慢的问题。

在要求详细信息时，我们的同事只能提供主机名和被认为“慢”的服务器的 IP。我们的同行提到一个用户报告了这个问题，而用户并没有提供太多细节。

在这种情况下，与第三章中讨论的情况不同，我们没有太多信息可以开始。似乎我们也无法向用户提出故障排除问题。作为系统管理员，需要用很少的信息来排除问题并不罕见。事实上，这种类型的情况非常普遍。

## 它很慢

“它很慢”很难排除故障。关于服务器或服务变慢的投诉最大的问题是，“慢”是相对于遇到问题的用户而言的。

在处理任何关于性能的投诉时，重要的区别是环境设计的基准。在某些环境中，系统以 30%的 CPU 利用率运行可能是一种常规活动，而其他环境可能会保持系统以 10%的 CPU 利用率运行，30%的利用率会表示问题。

在排除故障和调查性能问题时，重要的是回顾系统的历史性能指标，以确保您对收集到的测量值有上下文。这将有助于确定当前系统利用率是否符合预期或异常。

# 性能

一般来说，性能问题可以分为五个领域：

+   应用程序

+   CPU

+   内存

+   磁盘

+   网络

任何一个领域的瓶颈通常也会影响其他领域；因此，了解每个领域是一个好主意。通过了解如何访问和交互每个资源，您将能够找到消耗多个资源的问题的根本原因。

由于报告的问题没有包括任何性能问题的细节，我们将探索和了解每个领域。完成后，我们将查看收集的数据并查看历史统计数据，以确定性能是否符合预期，或者系统性能是否真的下降了。

## 应用程序

在创建性能类别列表时，我按照我经常看到的领域进行了排序。每个环境都是不同的，但根据我的经验，应用程序通常是性能问题的主要来源。

虽然本章旨在涵盖性能问题，第九章，“使用系统工具排除应用程序”专门讨论了使用系统工具排除应用程序问题，包括性能问题。在本章中，我们将假设我们的问题与应用程序无关，并专注于系统性能。

## CPU

CPU 是一个非常常见的性能瓶颈。有时，问题严格基于 CPU，而其他时候，增加的 CPU 使用率是另一个问题的症状。

调查 CPU 利用率最常见的命令是 top 命令。这个命令的主要作用是识别进程的 CPU 利用率。在第二章，“故障排除命令和有用信息的来源”中，我们讨论了使用`ps`命令进行这种活动。在本节中，我们将使用 top 和 ps 来调查 CPU 利用率，以解决我们的速度慢的问题。

### Top – 查看所有内容的单个命令

`top`命令是系统管理员和用户运行的第一批命令之一，用于查看整体系统性能。原因在于 top 不仅显示了负载平均值、CPU 和内存的详细情况，还显示了利用这些资源的进程的排序列表。

`top`最好的部分是，当不带任何标志运行时，这些详细信息每 3 秒更新一次。

以下是不带任何标志运行时`top`输出的示例。

```
top - 17:40:43 up  4:07,  2 users,  load average: 0.32, 0.43, 0.44
Tasks: 100 total,   2 running,  98 sleeping,   0 stopped,   0 zombie
%Cpu(s): 37.3 us,  0.7 sy,  0.0 ni, 62.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:    469408 total,   228112 used,   241296 free,      764 buffers
KiB Swap:  1081340 total,        0 used,  1081340 free.    95332 cached Mem

 PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
 3023 vagrant   20   0    7396    720    504 S 37.6  0.2  91:08.04 lookbusy
 11 root      20   0       0      0      0 R  0.3  0.0   0:13.28 rcuos/0
 682 root      20   0  322752   1072    772 S  0.3  0.2   0:05.60 VBoxService
 1 root      20   0   50784   7256   2500 S  0.0  1.5   0:01.39 systemd
 2 root      20   0       0      0      0 S  0.0  0.0   0:00.00 kthreadd
 3 root      20   0       0      0      0 S  0.0  0.0   0:00.24 ksoftirqd/0
 5 root       0 -20       0      0      0 S  0.0  0.0   0:00.00 kworker/0:0H
 6 root      20   0       0      0      0 S  0.0  0.0   0:00.04 kworker/u2:0
 7 root      rt   0       0      0      0 S  0.0  0.0   0:00.00 migration/0
 8 root      20   0       0      0      0 S  0.0  0.0   0:00.00 rcu_bh
 9 root      20   0       0      0      0 S  0.0  0.0   0:00.00 rcuob/0
 10 root      20   0       0      0      0 S  0.0  0.0   0:05.44 rcu_sched

```

`top`的默认输出中显示了相当多的信息。在本节中，我们将专注于 CPU 利用率信息。

```
%Cpu(s): 37.3 us,  0.7 sy,  0.0 ni, 62.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st

```

`top`命令输出的第一部分显示了当前 CPU 利用率的详细情况。列表中的每一项代表了 CPU 的不同使用方式。为了更好地理解输出结果，让我们来看看每个数值的含义：

+   **us – User**：这个数字表示用户模式中进程所消耗的 CPU 百分比。在这种模式下，应用程序无法访问底层硬件，必须使用系统 API（也称为系统调用）来执行特权操作。在执行这些系统调用时，执行将成为系统 CPU 利用率的一部分。

+   **sy – System**：这个数字表示内核模式执行所消耗的 CPU 百分比。在这种模式下，系统可以直接访问底层硬件；这种模式通常保留给受信任的操作系统进程。

+   **ni – Nice user processes**：这个数字表示由设置了 nice 值的用户进程所消耗的 CPU 时间百分比。`us%`值特指那些未修改过 niceness 值的进程。

+   **id – Idle**：这个数字表示 CPU 空闲的时间百分比。基本上，它是 CPU 未被利用的时间。

+   **wa – Wait**：这个数字表示 CPU 等待的时间百分比。当有很多进程在等待 I/O 设备时，这个值通常很高。I/O 等待状态不仅指硬盘，而是指所有 I/O 设备，包括硬盘。

+   **hi – Hardware interrupts**：这个数字表示由硬件中断所消耗的 CPU 时间百分比。硬件中断是来自系统硬件（如硬盘或网络设备）的信号，发送给 CPU。这些中断表示有事件需要 CPU 时间。

+   **si - 软件中断**：这个数字是被软件中断消耗的 CPU 时间的百分比。软件中断类似于硬件中断；但是，它们是由运行进程发送给内核的信号触发的。

+   **st - 被窃取**：这个数字特别适用于作为虚拟机运行的 Linux 系统。这个数字是被主机从这台机器上窃取的 CPU 时间的百分比。当主机机器本身遇到 CPU 争用时，通常会出现这种情况。在一些云环境中，这也可能发生，作为强制执行资源限制的一种方法。

我之前提到`top`的输出默认每 3 秒刷新一次。CPU 百分比行也每 3 秒刷新一次；`top`将显示自上次刷新间隔以来每个状态的 CPU 时间百分比。

#### 这个输出告诉我们关于我们的问题的什么？

如果我们回顾之前`top`命令的输出，我们可以对这个系统了解很多。

```
%Cpu(s): 37.3 us,  0.7 sy,  0.0 ni, 62.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st

```

从前面的输出中，我们可以看到 CPU 时间的`37.3%`被用户模式下的进程消耗。另外`0.7%`的 CPU 时间被内核执行模式下的进程使用；这是基于`us`和`sy`的值。`id`值告诉我们剩下的 CPU 没有被利用，这意味着总体上，这台服务器上有充足的 CPU 可用。

`top`命令显示的另一个事实是 CPU 时间没有花在等待 I/O 上。我们可以从`wa`值为`0.0`看出。这很重要，因为它告诉我们报告的性能问题不太可能是由于高 I/O。在本章后面，当我们开始探索磁盘性能时，我们将深入探讨 I/O 等待。

#### 来自 top 的单个进程

`top`命令输出中的 CPU 行是整个服务器的摘要，但 top 还包括单个进程的 CPU 利用率。为了更清晰地聚焦，我们可以再次执行 top，但这次，让我们专注于正在运行的`top`进程。

```
$ top -n 1
top - 15:46:52 up  3:21,  2 users,  load average: 1.03, 1.11, 1.06
Tasks: 108 total,   3 running, 105 sleeping,   0 stopped,   0 zombie
%Cpu(s): 34.1 us,  0.7 sy,  0.0 ni, 65.1 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st
KiB Mem:    502060 total,   220284 used,   281776 free,      764 buffers
KiB Swap:  1081340 total,        0 used,  1081340 free.    92940 cached Mem

 PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 3001 vagrant   20   0    7396    720    504 R  98.4  0.1 121:08.67 lookbusy
 3002 vagrant   20   0    7396    720    504 S   6.6  0.1  19:05.12 lookbusy
 1 root      20   0   50780   7264   2508 S   0.0  1.4   0:01.69 systemd
 2 root      20   0       0      0      0 S   0.0  0.0   0:00.01 kthreadd
 3 root      20   0       0      0      0 S   0.0  0.0   0:00.97 ksoftirqd/0
 5 root       0 -20       0      0      0 S   0.0  0.0   0:00.00 kworker/0:0H
 6 root      20   0       0      0      0 S   0.0  0.0   0:00.00 kworker/u4:0
 7 root      rt   0       0      0      0 S   0.0  0.0   0:00.67 migration/0

```

这次执行`top`命令时，使用了`-n`（数字）标志。这个标志告诉`top`只刷新指定次数，这里是 1 次。在尝试捕获`top`的输出时，这个技巧可能会有所帮助。

如果我们回顾上面`top`命令的输出，我们会看到一些非常有趣的东西。

```
 PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 3001 vagrant   20   0    7396    720    504 R  98.4  0.1 121:08.67 lookbusy

```

默认情况下，`top`命令按照进程利用的 CPU 百分比对进程进行排序。这意味着列表中的第一个进程是在该时间间隔内消耗 CPU 最多的进程。

如果我们看一下进程 ID 为`3001`的顶部进程，我们会发现它正在使用 CPU 时间的`98.4%`。然而，根据 top 命令的系统范围 CPU 统计数据，CPU 时间的`65.1%`处于空闲状态。这种情况实际上是许多系统管理员困惑的常见原因。

```
%Cpu(s): 34.1 us,  0.7 sy,  0.0 ni, 65.1 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st

```

一个单个进程如何使用几乎 100%的 CPU 时间，而系统本身显示 CPU 时间的 65%是空闲的？答案其实很简单；当`top`在其标题中显示 CPU 利用率时，比例是基于整个系统的。然而，对于单个进程，CPU 利用率的比例是针对一个 CPU 的。这意味着我们的进程 3001 实际上几乎使用了一个完整的 CPU，而我们的系统很可能有多个 CPU。

通常会看到能够利用多个 CPU 的进程显示的百分比高于 100%。例如，完全利用三个 CPU 的进程将显示 300%。这也可能会让不熟悉`top`命令服务器总体和每个进程输出差异的用户感到困惑。

### 确定可用 CPU 数量

先前，我们确定了这个系统必须有多个可用的 CPU。我们没有确定的是有多少个。确定可用 CPU 数量的最简单方法是简单地读取`/proc/cpuinfo`文件。

```
# cat /proc/cpuinfo
processor  : 0
vendor_id  : GenuineIntel
cpu family  : 6
model    : 58
model name  : Intel(R) Core(TM) i7-3615QM CPU @ 2.30GHz
stepping  : 9
microcode  : 0x19
cpu MHz    : 2348.850
cache size  : 6144 KB
physical id  : 0
siblings  : 2
core id    : 0
cpu cores  : 2
apicid    : 0
initial apicid  : 0
fpu    : yes
fpu_exception  : yes
cpuid level  : 5
wp    : yes
flags    : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl pni ssse3 lahf_lm
bogomips  : 4697.70
clflush size  : 64
cache_alignment  : 64
address sizes  : 36 bits physical, 48 bits virtual
power management:

processor  : 1
vendor_id  : GenuineIntel
cpu family  : 6
model    : 58
model name  : Intel(R) Core(TM) i7-3615QM CPU @ 2.30GHz
stepping  : 9
microcode  : 0x19
cpu MHz    : 2348.850
cache size  : 6144 KB
physical id  : 0
siblings  : 2
core id    : 1
cpu cores  : 2
apicid    : 1
initial apicid  : 1
fpu    : yes
fpu_exception  : yes
cpuid level  : 5
wp    : yes
flags    : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx rdtscp lm constant_tsc rep_good nopl pni ssse3 lahf_lm
bogomips  : 4697.70
clflush size  : 64
cache_alignment  : 64
address sizes  : 36 bits physical, 48 bits virtual
power management:

```

`/proc/cpuinfo`文件包含了关于系统可用 CPU 的大量有用信息。它显示了 CPU 的类型到型号，可用的标志，CPU 的速度，最重要的是可用的 CPU 数量。

系统中每个可用的 CPU 都将在`cpuinfo`文件中列出。这意味着您可以简单地在`cpuinfo`文件中计算处理器的数量，以确定服务器可用的 CPU 数量。

从上面的例子中，我们可以确定这台服务器有 2 个可用的 CPU。

#### 线程和核心

使用`cpuinfo`来确定可用 CPU 数量的一个有趣的注意事项是，当使用具有多个核心并且是超线程的 CPU 时，细节有点误导。`cpuinfo`文件将 CPU 上的核心和线程都报告为它可以利用的处理器。这意味着即使您的系统上安装了一个物理芯片，如果该芯片是一个四核超线程 CPU，`cpuinfo`文件将显示八个处理器。

#### lscpu – 查看 CPU 信息的另一种方法

虽然`/proc/cpuinfo`是许多管理员和用户用来确定 CPU 信息的方法；在基于 RHEL 的发行版上，还有另一条命令也会显示这些信息。

```
$ lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                2
On-line CPU(s) list:   0,1
Thread(s) per core:    1
Core(s) per socket:    2
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 58
Model name:            Intel(R) Core(TM) i7-3615QM CPU @ 2.30GHz
Stepping:              9
CPU MHz:               2348.850
BogoMIPS:              4697.70
L1d cache:             32K
L1d cache:             32K
L2d cache:             6144K
NUMA node0 CPU(s):     0,1

```

`/proc/cpuinfo`和`lscpu`命令之间的一个区别是，`lscpu`使得很容易识别核心、插槽和线程的数量。从`/proc/cpuinfo`文件中识别这些信息通常会有点困难。

### ps – 通过 ps 更深入地查看单个进程

虽然`top`命令可以用来查看单个进程，但我个人认为`ps`命令更适合用于调查运行中的进程。在第二章中，我们介绍了`ps`命令以及它如何用于查看运行进程的许多不同方面。

在本章中，我们将使用`ps`命令更深入地查看我们用`top`命令确定为利用最多 CPU 时间的进程`3001`。

```
$ ps -lf 3001
F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY TIME CMD
1 S vagrant   3001  3000 73  80   0 -  1849 hrtime 01:34 pts/1 892:23 lookbusy --cpu-mode curve --cpu-curve-peak 14h -c 20-80

```

在第二章中，我们讨论了使用`ps`命令来显示运行中的进程。在前面的例子中，我们指定了两个标志，这些标志在第二章中显示，`-l`（长列表）和`–f`（完整格式）。在本章中，我们讨论了这些标志如何为显示的进程提供额外的细节。

为了更好地理解上述进程，让我们分解一下这两个标志提供的额外细节。

+   当前状态：`S`（可中断睡眠）

+   用户：`vagrant`

+   进程 ID：`3001`

+   父进程 ID：`3000`

+   优先级值：`80`

+   优先级级别：`0`

+   正在执行的命令：`lookbusy –cpu-mode-curve –cpu-curve-peak 14h –c 20-80`

早些时候，使用`top`命令时，这个进程几乎使用了一个完整的 CPU，这意味着这个进程是导致报告的缓慢的嫌疑对象。通过查看上述细节，我们可以确定这个进程的一些情况。

首先，它是进程`3000`的子进程；这是我们通过父进程 ID 确定的。其次，当我们运行`ps`命令时，它正在等待一个任务完成；我们可以通过进程当前处于可中断睡眠状态来确定这一点。

除了这两项之外，我们还可以看出该进程没有高调度优先级。我们可以通过查看优先级值来确定这一点，在这种情况下是 80。调度优先级系统的工作方式如下：数字越高，进程在系统调度程序中的优先级越低。

我们还可以看到 niceness 级别设置为`0`，即默认值。这意味着用户没有调整 niceness 级别以获得更高（或更低）的优先级。

这些都是收集有关进程的重要数据点，但单独来看，它们并不能回答这个进程是否是报告的缓慢的原因。

#### 使用 ps 来确定进程的 CPU 利用率

由于我们知道进程`3001`是进程`3000`的子进程，我们不仅应该查看进程`3000`的相同信息，还应该使用`ps`来确定进程`3000`利用了多少 CPU。我们可以通过使用`-o`（选项）标志和`ps`来一次完成所有这些。这个标志允许您指定自己的输出格式；它还允许您查看通过常见的`ps`标志通常不可见的字段。

在下面的命令中，使用`-o`标志来格式化`ps`命令的输出，使用前一次运行的关键字段并包括`%cpu`字段。这个额外的字段将显示进程的 CPU 利用率。该命令还将使用`-p`标志来指定进程`3000`和进程`3001`。

```
$ ps -o state,user,pid,ppid,nice,%cpu,cmd -p 3000,3001
S USER       PID  PPID  NI %CPU CMD
S vagrant   3000  2980   0  0.0 lookbusy --cpu-mode curve --cpu- curve-peak 14h -c 20-80
R vagrant   3001  3000   0 71.5 lookbusy --cpu-mode curve --cpu- curve-peak 14h -c 20-80

```

虽然上面的命令非常长，但它展示了`-o`标志有多么有用。在给定正确的选项的情况下，只用`ps`命令就可以找到大量关于进程的信息。

从上面命令的输出中，我们可以看到进程`3000`是`lookbusy`命令的另一个实例。我们还可以看到进程`3000`是进程`2980`的子进程。在进一步进行之前，我们应该尝试识别与进程`3001`相关的所有进程。

我们可以使用`ps`命令和`--forest`标志来做到这一点，该标志告诉`ps`以树状格式打印父进程和子进程。当提供`-e`（所有）标志时，`ps`命令将以这种树状格式打印所有进程。

### 提示

默认情况下，`ps`命令只会打印与执行命令的用户相关的进程。`-e`标志改变了这种行为，以打印所有可能的进程。

下面的输出被截断，以特别识别`lookbusy`进程。

```
$ ps --forest -eo user,pid,ppid,%cpu,cmd
root      1007     1  0.0 /usr/sbin/sshd -D
root      2976  1007  0.0  \_ sshd: vagrant [priv]
vagrant   2979  2976  0.0      \_ sshd: vagrant@pts/1
vagrant   2980  2979  0.0          \_ -bash
vagrant   3000  2980  0.0              \_ lookbusy --cpu-mode curve - -cpu-curve-peak 14h -c 20-80
vagrant   3001  3000 70.4                  \_ lookbusy --cpu-mode curve --cpu-curve-peak 14h -c 20-80
vagrant   3002  3000 14.6                  \_ lookbusy --cpu-mode curve --cpu-curve-peak 14h -c 20-80

```

从上面的`ps`输出中，我们可以看到 ID 为`3000`的`lookbusy`进程产生了两个进程，分别是`3001`和`3002`。我们还可以看到当前通过 SSH 登录的 vagrant 用户启动了`lookbusy`进程。

由于我们还使用了`-o`标志和`ps`来显示 CPU 利用率，我们可以看到进程`3002`正在利用单个 CPU 的`14.6%`。

### 提示

重要的是要注意，`ps`命令还显示了单个处理器的 CPU 时间百分比，这意味着利用多个处理器的进程可能具有高于 100%的值。

## 把它们都放在一起

现在我们已经通过命令来识别系统的 CPU 利用率，让我们把它们放在一起总结一下找到的东西。

### 用 top 快速查看

我们识别与 CPU 性能相关的问题的第一步是执行`top`命令。

```
$ top

top - 01:50:36 up 23:41,  2 users,  load average: 0.68, 0.56, 0.48
Tasks: 107 total,   4 running, 103 sleeping,   0 stopped,   0 zombie
%Cpu(s): 34.5 us,  0.7 sy,  0.0 ni, 64.9 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:    502060 total,   231168 used,   270892 free,      764 buffers
KiB Swap:  1081340 total,        0 used,  1081340 free.    94628 cached Mem

 PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 3001 vagrant   20   0    7396    724    508 R  68.8  0.1 993:06.80 lookbusy
 3002 vagrant   20   0    7396    724    508 S   1.0  0.1 198:58.16 lookbusy
 12 root      20   0       0      0      0 S   0.3  0.0   3:47.55 rcuos/0
 13 root      20   0       0      0      0 R   0.3  0.0   3:38.85 rcuos/1
 2718 vagrant   20   0  131524   2536   1344 R   0.3  0.5   0:02.28 sshd

```

从`top`的输出中，我们可以识别以下内容：

+   总体而言，系统大约 60%–70%的时间处于空闲状态

+   有两个正在运行`lookbusy`命令/程序的进程，其中一个似乎正在使用单个 CPU 的 70%

+   鉴于这个单独进程的 CPU 利用率和系统 CPU 利用率，所涉及的服务器很可能有多个 CPU

+   我们可以用`lscpu`命令确认存在多个 CPU

+   进程 3001 和 3002 是该系统上利用 CPU 最多的两个进程

+   CPU 等待状态百分比为 0，这意味着问题不太可能与磁盘 I/O 有关

#### 通过 ps 深入挖掘

由于我们从`top`命令的输出中确定了进程`3001`和`3002`是可疑的，我们可以使用`ps`命令进一步调查这些进程。为了快速进行调查，我们将使用`ps`命令和`-o`和`--forest`标志来用一个命令识别最大可能的信息。

```
$ ps --forest -eo user,pid,ppid,%cpu,cmd
root      1007     1  0.0 /usr/sbin/sshd -D
root      2976  1007  0.0  \_ sshd: vagrant [priv]
vagrant   2979  2976  0.0      \_ sshd: vagrant@pts/1
vagrant   2980  2979  0.0          \_ -bash
vagrant   3000  2980  0.0              \_ lookbusy --cpu-mode curve --cpu-curve-peak 14h -c 20-80
vagrant   3001  3000 69.8                  \_ lookbusy --cpu-mode curve --cpu-curve-peak 14h -c 20-80
vagrant   3002  3000 13.9                  \_ lookbusy --cpu-mode curve --cpu-curve-peak 14h -c 20-80

```

从这个输出中，我们可以确定以下内容：

+   进程 3001 和 3002 是进程 3000 的子进程

+   进程 3000 是由`vagrant`用户启动的

+   `lookbusy`命令似乎是一个利用大量 CPU 的命令

+   启动`lookbusy`的方法并不表明这是一个系统进程，而是一个用户运行的临时命令。

根据上述信息，`vagrant`用户启动的`lookbusy`进程有可能是性能问题的根源。如果这个系统通常的 CPU 利用率较低，这是一个合理的根本原因的假设。然而，考虑到我们对这个系统不太熟悉，`lookbusy`进程几乎使用了整个 CPU 也是可能的。

考虑到我们对系统的正常运行条件不太熟悉，我们应该在得出结论之前继续调查性能问题的其他可能来源。

## 内存

在应用程序和 CPU 利用率之后，内存利用率是性能下降的一个非常常见的来源。在 CPU 部分，我们广泛使用了`top`，虽然`top`也可以用来识别系统和进程的内存利用率，但在这一部分，我们将使用其他命令。

### free – 查看空闲和已用内存

如第二章中所讨论的，*故障排除命令和有用信息来源* `free`命令只是简单地打印系统当前的内存可用性和使用情况。

当没有标志时，`free`命令将以千字节为单位输出其值。为了使输出以兆字节为单位，我们可以简单地使用`-m`（兆字节）标志执行`free`命令。

```
$ free -m
 total       used       free     shared    buffers     cached
Mem:     490         92        397          1          0         17
-/+ buffers/cache:         74        415
Swap:         1055         57        998

```

`free`命令显示了关于这个系统以及内存使用情况的大量信息。为了更好地理解这个命令，让我们对输出进行一些分解。

由于输出中有多行，我们将从输出标题之后的第一行开始：

```
Mem:         490        92       397         1         0         17

```

这一行中的第一个值是系统可用的**物理内存**总量。在我们的情况下，这是 490 MB。第二个值是系统使用的**内存**量。第三个值是系统上**未使用**的内存量；请注意，我使用了“未使用”而不是“可用”这个术语。第四个值是用于**共享内存**的内存量；除非您的系统经常使用共享内存，否则这通常是一个较低的数字。

第五个值是用于**缓冲区**的内存量。Linux 通常会尝试通过将频繁使用的磁盘信息放入物理内存来加快磁盘访问速度。缓冲区内存通常是文件系统元数据。**缓存内存**，也就是第六个值，是经常访问文件的内容。

#### Linux 内存缓冲区和缓存

Linux 通常会尝试使用“未使用”的内存来进行缓冲和缓存。这意味着为了提高效率，Linux 内核将频繁访问的文件数据和文件系统元数据存储在未使用的内存中。这使得系统能够利用本来不会被使用的内存来增强磁盘访问，而磁盘访问通常比系统内存慢。

这就是为什么第三个值“未使用”内存通常比预期的数字要低的原因。

然而，当系统的未使用内存不足时，Linux 内核将根据需要释放缓冲区和缓存内存。这意味着即使从技术上讲，用于缓冲区和缓存的内存被使用了，但在需要时它从技术上讲是可用的。

这将我们带到了 free 输出的第二行。

```
-/+ buffers/cache:         74        415

```

第二行有两个值，第一个是**Used**列的一部分，第二个是**Free**或“未使用”列的一部分。这些值是在考虑缓冲区和缓存内存的可用或未使用内存值之后得出的。

简单来说，第二行的已使用值是从第一行的已使用内存值减去缓冲区和缓存值得到的结果。对于我们的示例，这是 92 MB（已使用）减去 17 MB（cached）。

第二行的 free 值是第一行的 Free 值加上缓冲区和缓存内存的结果。使用我们的示例数值，这将是 397 MB（free）加上 17 MB（cached）。

#### 交换内存

`free`命令的输出的第三行是用于交换内存的。

```
Swap:         1055         57        998

```

在这一行中，有三列：可用、已使用和空闲。交换内存的值相当容易理解。可用交换值是系统可用的交换内存量，已使用值是当前分配的交换量，而空闲值基本上是可用交换减去已分配的交换量。

有许多环境不赞成分配大量的交换空间，因为这通常是系统内存不足并使用交换空间来补偿的指标。

#### free 告诉我们关于我们系统的信息

如果我们再次查看 free 的输出，我们可以确定关于这台服务器的很多事情。

```
$ free -m
 total       used       free     shared    buffers     cached
Mem:       490        105        385          1          0         25
-/+ buffers/cache:         79        410
Swap:         1055         56        999

```

我们可以确定实际上只使用了很少的内存（79 MB）。这意味着总体上，系统应该有足够的内存可用于进程。

然而，还有一个有趣的事实，在第三行显示，**56** MB 的内存已被写入交换空间。尽管系统当前有大量可用内存，但已经有 56 MB 被写入交换空间。这意味着在过去的某个时刻，这个系统可能内存不足，足够低到系统不得不将内存页面从物理内存交换到交换内存。

### 检查 oomkill

当 Linux 系统的物理内存耗尽时，它首先尝试重用分配给缓冲区和缓存的内存。如果没有额外的内存可以从这些来源中回收，那么内核将从物理内存中获取旧的内存页面并将它们写入交换内存。一旦物理内存和交换内存都被分配，内核将启动**内存不足杀手**（**oomkill**）进程。`oomkill`进程旨在找到使用大量内存的进程并将其杀死（停止）。

一般来说，在大多数环境中，`oomkill`进程是不受欢迎的。一旦调用，`oomkill`进程可以杀死许多不同类型的进程。无论进程是系统的一部分还是用户级别的，`oomkill`都有能力杀死它们。

对于可能影响内存利用的性能问题，检查`oomkill`进程最近是否被调用是一个很好的主意。确定`oomkill`最近是否运行的最简单方法是简单地查看系统的控制台，因为这个进程的启动会直接记录在系统控制台上。然而，在云和虚拟环境中，控制台可能不可用。

另一个确定最近是否调用了`oomkill`的好方法是搜索`/var/log/messages`日志文件。我们可以通过执行`grep`命令并搜索字符串`Out of memory`来做到这一点。

```
# grep "Out of memory" /var/log/messages

```

对于我们的示例系统，最近没有发生`oomkill`调用。如果我们的系统调用了`oomkill`进程，我们可能会收到类似以下消息：

```
# grep "Out of memory" /var/log/messages
Feb  7 19:38:45 localhost kernel: Out of memory: Kill process 3236 (python) score 838 or sacrifice child

```

在第十一章中，*从常见故障中恢复*，我们将再次调查内存问题，并深入了解`oomkill`及其工作原理。对于本章，我们可以得出结论，系统尚未完全耗尽其可用内存。

### ps - 检查单个进程的内存利用率

到目前为止，系统上的内存使用似乎很小，但是我们从 CPU 验证步骤中知道，运行`lookbusy`的进程是可疑的，可能导致性能问题。由于我们怀疑`lookbusy`进程存在问题，我们还应该查看这些进程使用了多少内存。为了做到这一点，我们可以再次使用带有`-o`标志的`ps`命令。

```
$ ps -eo user,pid,ppid,%mem,rss,vsize,comm | grep lookbusy
vagrant   3000  2980  0.0     4   7396 lookbusy
vagrant   3001  3000  0.0   296   7396 lookbusy
vagrant   3002  3000  0.0   220   7396 lookbusy
vagrant   5380  2980  0.0     8   7396 lookbusy
vagrant   5381  5380  0.0   268   7396 lookbusy
vagrant   5382  5380  0.0   268   7396 lookbusy
vagrant   5383  5380 40.7 204812 212200 lookbusy
vagrant   5531  2980  0.0    40   7396 lookbusy
vagrant   5532  5531  0.0   288   7396 lookbusy
vagrant   5533  5531  0.0   288   7396 lookbusy
vagrant   5534  5531 34.0 170880 222440 lookbusy

```

然而，这一次我们以稍有不同的方式运行了我们的`ps`命令，因此得到了不同的结果。这一次执行`ps`命令时，我们使用了`-e`（everything）标志来显示所有进程。然后将结果传输到`grep`，以便将它们缩小到只匹配`lookbusy`模式的进程。

这是使用`ps`命令的一种非常常见的方式；事实上，这比在命令行上指定进程 ID 更常见。除了使用`grep`之外，这个`ps`命令示例还介绍了一些新的格式选项。

+   **%mem**：这是进程正在使用的系统内存的百分比。

+   **rss**：这是进程的常驻集大小，基本上是指进程使用的不可交换内存量。

+   **vsize**：这是虚拟内存大小，它包含进程完全使用的内存量，无论这些内存是物理内存的一部分还是交换内存的一部分。

+   **comm**：此选项类似于 cmd，但不显示命令行参数。

`ps`示例显示了有趣的信息，特别是以下几行：

```
vagrant   5383  5380 40.7 204812 212200 lookbusy
vagrant   5534  5531 34.0 170880 222440 lookbusy

```

似乎已经启动了几个额外的`lookbusy`进程，并且这些进程正在利用系统内存的 40%和 34%（通过使用`%mem`列）。从 rss 列中，我们可以看到这两个进程正在使用总共 490MB 物理内存中的约 374MB。

看起来这些进程在我们开始调查后开始利用大量内存。最初，我们的 free 输出表明只使用了 70MB 内存；然而，这些进程似乎利用了更多。我们可以通过再次运行 free 来确认这一点。

```
$ free -m
 total       used       free     shared    buffers     cached
Mem:       490        453         37          0          0          3
-/+ buffers/cache:        449         41
Swap:         1055        310        745

```

事实上，我们的系统现在几乎利用了所有的内存；事实上，我们还使用了 310MB 的交换空间。

### vmstat - 监控内存分配和交换

由于这个系统的内存利用率似乎有所波动，有一个非常有用的命令可以定期显示内存分配和释放以及换入和换出的页面数。这个命令叫做`vmstat`。

```
$ vmstat -n 10 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
5  0 204608  31800      0   7676    8    6    12     6  101  131 44 1 55  0  0
1  0 192704  35816      0   2096 1887  130  4162   130 2080 2538 53 6 39  2  0
1  0 191340  32324      0   3632 1590   57  3340    57 2097 2533 54 5 41  0  0
4  0 191272  32260      0   5400  536    2  2150     2 1943 2366 53 4 43  0  0
3  0 191288  34140      0   4152  392    0   679     0 1896 2366 53 3 44  0  0

```

在上面的示例中，`vmstat`命令是使用`-n`（一个标题）标志执行的，后面跟着延迟时间（10 秒）和要生成的报告数（5）。这些选项告诉`vmstat`仅为此次执行输出一个标题行，而不是为每个报告输出一个新的标题行，每 10 秒运行一次报告，并将报告数量限制为 5。如果省略了报告数量的限制，`vmstat`将简单地持续运行，直到使用*CTRL*+*C*停止。

`vmstat`的输出一开始可能有点压倒性，但如果我们分解输出，就会更容易理解。`vmstat`的输出有六个输出类别，即进程、内存、交换、IO、系统和 CPU。在本节中，我们将专注于这两个类别：内存和交换。

+   **内存**

+   `swpd`：写入交换的内存量

+   `free`：未使用的内存量

+   `buff`：用作缓冲区的内存量

+   `cache`：用作缓存的内存量

+   `inact`：非活动内存量

+   `active`：活动内存量

+   **交换**

+   `si`：从磁盘交换的内存量

+   `so`：交换到磁盘的内存量

现在我们已经了解了这些值的定义，让我们看看`vmstat`的输出告诉我们关于这个系统内存使用情况的信息。

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 5  0 204608  31800      0   7676    8    6    12     6  101  131 44  1 55  0  0
 1  0 192704  35816      0   2096 1887  130  4162   130 2080 2538 53  6 39  2  0

```

如果我们比较`vmstat`输出的第一行和第二行，我们会看到一个相当大的差异。特别是，我们可以看到在第一个间隔中，缓存内存是`7676`，而在第二个间隔中，这个值是 2096。我们还可以看到第一行中的`si`或交换入值是 8，而第二行中是 1887。

这种差异的原因是，`vmstat`的第一个报告总是自上次重启以来的统计摘要，而第二个报告是自上一个报告以来的统计摘要。每个后续的报告将总结前一个报告，这意味着第三个报告将总结自第二个报告以来的统计数据。`vmstat`的这种行为经常会让新的系统管理员和用户感到困惑；因此，它通常被认为是一种高级故障排除工具。

由于`vmstat`生成第一个报告的方法，通常的做法是丢弃它并从第二个报告开始。我们将遵循这一原则，特别关注第二个和第三个报告。

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 5  0 204608  31800      0   7676    8    6    12     6  101  131 44  1 55  0  0
 1  0 192704  35816      0   2096 1887  130  4162   130 2080 2538 53  6 39  2  0
 1  0 191340  32324      0   3632 1590   57  3340    57 2097 2533 54  5 41  0  0

```

在第二个和第三个报告中，我们可以看到一些有趣的数据。

最引人注目的是，从第一个报告的生成时间到第二个报告的生成时间，交换了 1,887 页，交换出了 130 页。第二个报告还显示，只有 35 MB 的内存是空闲的，缓冲区中没有内存，缓存中有 2 MB 的内存。根据 Linux 内存的利用方式，这意味着系统上实际上只有 37 MB 的可用内存。

这种低可用内存量解释了为什么我们的系统已经交换了大量页面。我们可以从第三行看到这种趋势正在持续，我们继续交换了相当多的页面，我们的可用内存已经减少到大约 35 MB。

从这个`vmstat`的例子中，我们可以看到我们的系统现在已经用尽了物理内存。因此，我们的系统正在从物理 RAM 中取出内存页面并将其写入我们的交换设备。

### 把所有东西放在一起

现在我们已经探索了用于故障排除内存利用的工具，让我们把它们都放在一起来解决系统性能缓慢的问题。

#### 用 free 查看系统的内存利用

给我们提供系统内存利用快照的第一个命令是`free`命令。这个命令将为我们提供在哪里进一步查找任何内存利用问题的想法。

```
$ free -m
 total       used       free     shared    buffers     cached
Mem:       490        293        196          0          0         18
-/+ buffers/cache:        275        215
Swap:         1055        183        872

```

从`free`的输出中，我们可以看到目前有 215 MB 的内存可用。我们可以通过第二行的`free`列看到这一点。我们还可以看到，总体上，这个系统有 183 MB 的内存已经被交换到我们的交换设备。

#### 观察 vmstat 的情况

由于系统在某个时候已经进行了交换（或者说分页），我们可以使用`vmstat`命令来查看系统当前是否正在进行交换。

这次执行`vmstat`时，我们将不指定报告值的数量，这将导致`vmstat`持续报告内存统计，类似于 top 命令的输出。

```
$ vmstat -n 10
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 4  0 188008 200320      0  19896   35    8    61     9  156    4 44 1 55  0  0
 4  0 188008 200312      0  19896    0    0     0     0 1361 1314 36 2 62  0  0
 2  0 188008 200312      0  19896    0    0     0     0 1430 1442 37 2 61  0  0
 0  0 188008 200312      0  19896    0    0     0     0 1431 1418 37 2 61  0  0
 0  0 188008 200280      0  19896    0    0     0     0 1414 1416 37 2 61  0  0
 2  0 188008 200280      0  19896    0    0     0     0 1456 1480 37 2 61  0  0

```

这个`vmstat`输出与我们之前的执行不同。从这个输出中，我们可以看到虽然有相当多的内存被交换，但系统目前并没有进行交换。我们可以通过`si`（交换入）和 so（交换出）列中的 0 值来确定这一点。

实际上，在这次`vmstat`运行期间，内存利用率似乎很稳定。每个`vmstat`报告中，`free`内存值都相当一致，缓存和缓冲内存统计也是如此。

#### 使用 ps 找到内存利用最多的进程

我们的系统有 490MB 的物理内存，`free`和`vmstat`都显示大约 215MB 的可用内存。这意味着我们系统内存的一半以上目前被使用；在这种使用水平下，找出哪些进程正在使用我们系统的内存是一个好主意。即使没有别的，这些数据也将有助于显示系统当前的状态。

要识别使用最多内存的进程，我们可以使用`ps`命令以及 sort 和 tail。

```
# ps -eo rss,vsize,user,pid,cmd | sort -nk 1 | tail -n 5
 1004 115452 root      5073 -bash
 1328 123356 root      5953 ps -eo rss,vsize,user,pid,cmd
 2504 525652 root       555 /usr/sbin/NetworkManager --no-daemon
 4124  50780 root         1 /usr/lib/systemd/systemd --switched-root --system --deserialize 23
204672 212200 vagrant  5383 lookbusy -m 200MB -c 10

```

上面的例子使用管道将`ps`的输出重定向到 sort 命令。sort 命令执行数字（`-n`）对第一列（`-k 1`）的排序。这将对输出进行排序，将具有最高`rss`大小的进程放在底部。在`sort`命令之后，输出也被管道传递到`tail`命令，当指定了`-n`（数字）标志后跟着一个数字，将限制输出只包括指定数量的结果。

### 提示

如果将命令与管道一起链接的概念是新的，我强烈建议练习这一点，因为它对日常的`sysadmin`任务以及故障排除非常有用。我们将在本书中多次讨论这个概念，并提供示例。

```
204672 212200 vagrant  5383 lookbusy -m 200MB -c 10

```

从`ps`的输出中，我们可以看到进程 5383 正在使用大约 200MB 的内存。我们还可以看到该进程是另一个`lookbusy`进程，再次由 vagrant 用户生成。

从`free`，`vmstat`和`ps`的输出中，我们可以确定以下内容：

+   系统当前大约有 200MB 的可用内存

+   虽然系统目前没有交换，但过去曾经有过，根据我们之前从`vmstat`看到的情况，我们知道它最近进行了交换

+   我们发现进程`5383`正在使用大约 200MB 的内存

+   我们还可以看到进程`5383`是由`vagrant`用户启动的，并且正在运行`lookbusy`进程

+   使用`free`命令，我们可以看到这个系统有 490MB 的物理内存

根据以上信息，似乎由`vagrant`用户执行的`lookbusy`进程不仅是 CPU 的可疑使用者，还是内存的可疑使用者。

## 磁盘

磁盘利用率是另一个常见的性能瓶颈。一般来说，性能问题很少是由于磁盘空间的问题。虽然我曾经看到由于大量文件或大文件的性能问题，但一般来说，磁盘性能受到写入和读取磁盘的限制。因此，在故障排除性能问题时，了解文件系统是否已满很重要，但仅仅根据文件系统的使用情况并不总是能指示是否存在问题。

### iostat - CPU 和设备输入/输出统计

`iostat`命令是用于故障排除磁盘性能问题的基本命令，类似于 vmstat，它提供的使用和信息都是相似的。像`vmstat`一样，执行`iostat`命令后面跟着两个数字，第一个是报告生成的延迟，第二个是要生成的报告数。

```
$ iostat -x 10 3
Linux 3.10.0-123.el7.x86_64 (blog.example.com)   02/08/2015 _x86_64_  (2 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 43.58    0.00    1.07    0.16    0.00   55.19

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda              12.63     3.88    8.47    3.47   418.80   347.40 128.27     0.39   32.82    0.80  110.93   0.47   0.56
dm-0              0.00     0.00   16.37    3.96    65.47    15.82 8.00     0.48   23.68    0.48  119.66   0.09   0.19
dm-1              0.00     0.00    4.73    3.21   353.28   331.71 172.51     0.39   48.99    1.07  119.61   0.54   0.43

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 20.22    0.00   20.33   22.14    0.00   37.32

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.10    13.67  764.97  808.68 71929.34 78534.73 191.23    62.32   39.75    0.74   76.65   0.42  65.91
dm-0              0.00     0.00    0.00    0.10     0.00     0.40 8.00     0.01   70.00    0.00   70.00  70.00   0.70
dm-1              0.00     0.00  765.27  769.76 71954.89 78713.17 196.31    64.65   42.25    0.74   83.51   0.43  66.46

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 18.23    0.00   15.56   29.26    0.00   36.95

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.10     7.10  697.50  440.10 74747.60 42641.75 206.38    74.13   66.98    0.64  172.13   0.58  66.50
dm-0              0.00     0.00    0.00    0.00     0.00     0.00 0.00     0.00    0.00    0.00    0.00   0.00   0.00
dm-1              0.00     0.00  697.40  405.00 74722.00 40888.65 209.74    75.80   70.63    0.66  191.11   0.61  67.24

```

在上面的例子中，提供了`-x`（扩展统计）标志以打印扩展统计信息。扩展统计非常有用，并提供了额外的信息，对于识别性能瓶颈至关重要。

#### CPU 详情

`iostat`命令将显示 CPU 统计信息以及 I/O 统计信息。这是另一个可以用来排除 CPU 利用率的命令。当 CPU 利用率指示高 I/O 等待时间时，这是特别有用的。

```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 20.22    0.00   20.33   22.14    0.00   37.32

```

以上是从`top`命令显示的相同信息；在 Linux 中找到多个输出类似信息的命令并不罕见。由于这些细节已在 CPU 故障排除部分中涵盖，我们将专注于`iostat`命令的 I/O 统计部分。

#### 审查 I/O 统计

要开始审查 I/O 统计，让我们从前两份报告开始。我在下面包括了 CPU 利用率，以帮助指示每份报告的开始位置，因为它是每份统计报告中的第一项。

```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 43.58    0.00    1.07    0.16    0.00   55.19

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda              12.63     3.88    8.47    3.47   418.80   347.40 128.27     0.39   32.82    0.80  110.93   0.47   0.56
dm-0              0.00     0.00   16.37    3.96    65.47    15.82 8.00     0.48   23.68    0.48  119.66   0.09   0.19
dm-1              0.00     0.00    4.73    3.21   353.28   331.71 172.51     0.39   48.99    1.07  119.61   0.54   0.43

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 20.22    0.00   20.33   22.14    0.00   37.32

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.10    13.67  764.97  808.68 71929.34 78534.73 191.23    62.32   39.75    0.74   76.65   0.42  65.91
dm-0              0.00     0.00    0.00    0.10     0.00     0.40 8.00     0.01   70.00    0.00   70.00  70.00   0.70
dm-1              0.00     0.00  765.27  769.76 71954.89 78713.17 196.31    64.65   42.25    0.74   83.51   0.43  66.46

```

通过比较前两份报告，我们发现它们之间存在很大的差异。在第一个报告中，`sda`设备的`％util`值为`0.56`，而在第二个报告中为`65.91`。

这种差异的原因是，与`vmstat`一样，第一次执行`iostat`的统计是基于服务器最后一次重启的时间。第二份报告是基于第一份报告之后的时间。这意味着第二份报告的输出是基于第一份报告生成之间的 10 秒。这与`vmstat`中看到的行为相同，并且是其他收集性能统计信息的工具的常见行为。

与`vmstat`一样，我们将丢弃第一个报告，只看第二个报告。

```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 20.22    0.00   20.33   22.14    0.00   37.32

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.10    13.67  764.97  808.68 71929.34 78534.73 191.23    62.32   39.75    0.74   76.65   0.42  65.91
dm-0              0.00     0.00    0.00    0.10     0.00     0.40 8.00     0.01   70.00    0.00   70.00  70.00   0.70
dm-1              0.00     0.00  765.27  769.76 71954.89 78713.17 196.31    64.65   42.25    0.74   83.51   0.43  66.46

```

从上面，我们可以确定这个系统的几个情况。最重要的是 CPU 行中的`％iowait`值。

```
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 20.22    0.00   20.33   22.14    0.00   37.32

```

早些时候在执行 top 命令时，等待 I/O 的时间百分比相当小；然而，在运行`iostat`时，我们可以看到 CPU 实际上花了很多时间等待 I/O。虽然 I/O 等待并不一定意味着等待磁盘，但这个输出的其余部分似乎表明磁盘活动相当频繁。

```
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.10    13.67  764.97  808.68 71929.34 78534.73 191.23    62.32   39.75    0.74   76.65   0.42  65.91
dm-0              0.00     0.00    0.00    0.10     0.00     0.40 8.00     0.01   70.00    0.00   70.00  70.00   0.70
dm-1              0.00     0.00  765.27  769.76 71954.89 78713.17 196.31    64.65   42.25    0.74   83.51   0.43  66.46

```

扩展统计输出有许多列，为了使这个输出更容易理解，让我们分解一下这些列告诉我们的内容。

+   **rrqm/s**：每秒合并和排队的读取请求数

+   **wrqm/s**：每秒合并和排队的写入请求数

+   **r/s**：每秒完成的读取请求数

+   **w/s**：每秒完成的写入请求数

+   **rkB/s**：每秒读取的千字节数

+   **wkB/s**：每秒写入的千字节数

+   **avgr-sz**：发送到设备的请求的平均大小（以扇区为单位）

+   **avgqu-sz**：发送到设备的请求的平均队列长度

+   **await**：请求等待服务的平均时间（毫秒）

+   **r_await**：读取请求等待服务的平均时间（毫秒）

+   **w_await**：写入请求等待服务的平均时间（毫秒）

+   **svctm**：此字段无效，将被删除；不应被信任或使用

+   **％util**：在此设备服务 I/O 请求时所花费的 CPU 时间百分比。设备最多只能利用 100％

对于我们的示例，我们将专注于`r/s`，`w/s`，`await`和`％util`值，因为这些值将告诉我们关于这个系统的磁盘利用率的很多信息，同时保持我们的示例简单。

经过审查`iostat`输出后，我们可以看到`sda`和`dm-1`设备都具有最高的`％util`值，这意味着它们最接近达到容量。

```
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.10    13.67  764.97  808.68 71929.34 78534.73 191.23    62.32   39.75    0.74   76.65   0.42  65.91
dm-1              0.00     0.00  765.27  769.76 71954.89 78713.17 196.31    64.65   42.25    0.74   83.51   0.43  66.46

```

从这份报告中，我们可以看到`sda`设备平均完成了 764 次读取（`r/s`）和 808 次写入（`w/s`）每秒。我们还可以确定这些请求平均需要 39 毫秒（等待时间）来完成。虽然这些数字很有趣，但并不一定意味着系统处于异常状态。由于我们对这个系统不熟悉，我们并不一定知道读取和写入的水平是否出乎意料。然而，收集这些信息是很重要的，因为这些统计数据是故障排除过程中数据收集阶段的重要数据。

从`iostat`中我们可以看到另一个有趣的统计数据是，`sda`和`dm-1`设备的`%util`值都约为 66%。这意味着在第一次报告生成和第二次报告之间的 10 秒内，66%的 CPU 时间都是在等待`sd`或`dm-1`设备。

#### 识别设备

对于磁盘设备来说，66%的利用率通常被认为是很高的，虽然这是非常有用的信息，但它并没有告诉我们是谁或什么在利用这个磁盘。为了回答这些问题，我们需要弄清楚`sda`和`dm-1`到底被用来做什么。

由于`iostat`命令输出的设备通常是磁盘设备，识别这些设备的第一步是运行`mount`命令。

```
$ mount
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime,seclabel)
devtmpfs on /dev type devtmpfs (rw,nosuid,seclabel,size=244828k,nr_inodes=61207,mode=755)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,seclabel)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,seclabel,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,seclabel,mode=755)
tmpfs on /sys/fs/cgroup type tmpfs (rw,nosuid,nodev,noexec,seclabel,mode=755)
configfs on /sys/kernel/config type configfs (rw,relatime)
/dev/mapper/root on / type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,seclabel)
mqueue on /dev/mqueue type mqueue (rw,relatime,seclabel)
debugfs on /sys/kernel/debug type debugfs (rw,relatime)
/dev/sda1 on /boot type xfs (rw,relatime,seclabel,attr2,inode64,noquota)

```

`mount`命令在没有任何选项的情况下运行时，将显示所有当前挂载的文件系统。`mount`输出中的第一列是已经挂载的设备。在上面的输出中，我们可以看到`sda`设备实际上是一个磁盘设备，并且它有一个名为`sda1`的分区，挂载为`/boot`。

然而，我们没有看到`dm-1`设备。由于这个设备没有出现在`mount`命令的输出中，我们可以通过另一种方式，在`/dev`文件夹中查找`dm-1`设备。

系统上的所有设备都被呈现为`/dev`文件夹结构中的一个文件。`dm-1`设备也不例外。

```
$ ls -la /dev/dm-1
brw-rw----. 1 root disk 253, 1 Feb  1 18:47 /dev/dm-1

```

虽然我们已经找到了`dm-1`设备的位置，但我们还没有确定它的用途。然而，关于这个设备，有一件事引人注目，那就是它的名字`dm-1`。当设备以`dm`开头时，这表明该设备是由设备映射器创建的逻辑设备。

设备映射器是一个 Linux 内核框架，允许系统创建虚拟磁盘设备，这些设备“映射”回物理设备。这个功能用于许多特性，包括软件 RAID、磁盘加密和逻辑卷。

设备映射器框架中的一个常见做法是为这些特性创建符号链接，这些符号链接指向单个逻辑设备。由于我们可以用`ls`命令看到`dm-1`是一个块设备，通过输出的第一列的“b”值（`brw-rw----.`），我们知道`dm-1`不是一个符号链接。我们可以利用这些信息以及 find 命令来识别任何指向`dm-1`块设备的符号链接。

```
# find -L /dev -samefile /dev/dm-1
/dev/dm-1
/dev/rhel/root
/dev/disk/by-uuid/beb5220d-5cab-4c43-85d7-8045f870ba7d
/dev/disk/by-id/dm-uuid-LVM-qj3iMeektIlL3Z0g4WMPMJRbzacnpS9IVOCzB60GSHCEgbRKYW9ZKXR5prUPEE1e
/dev/disk/by-id/dm-name-root
/dev/block/253:1
/dev/mapper/root

```

在前面的章节中，我们使用 find 命令来识别配置和日志文件。在上面的例子中，我们使用了`-L`（跟随链接）标志，后面跟着`/dev`路径和`--samefile`标志，告诉 find 搜索`/dev`文件夹结构，搜索任何符号链接的文件，以识别任何与`/dev/dm-1`相同的文件。

`--samefile`标志标识具有相同`inode`号的文件。当命令中包含`-L`标志时，输出包括符号链接，而这个例子似乎返回了几个结果。最引人注目的符号链接文件是`/dev/mapper/root`；这个文件之所以引人注目，是因为它也出现在挂载命令的输出中。

```
/dev/mapper/root on / type xfs (rw,relatime,seclabel,attr2,inode64,noquota)

```

看起来`/dev/mapper/root`似乎是一个逻辑卷。在 Linux 中，逻辑卷本质上是存储虚拟化。这个功能允许您创建伪设备（作为设备映射器的一部分），它映射到一个或多个物理设备。

例如，可以将四个不同的硬盘组合成一个逻辑卷。逻辑卷然后可以用作单个文件系统的磁盘。甚至可以在以后通过使用逻辑卷添加另一个硬盘。

确认`/dev/mapper/root`设备实际上是一个逻辑卷，我们可以执行`lvdisplay`命令，该命令用于显示系统上的逻辑卷。

```
# lvdisplay
 --- Logical volume ---
 LV Path                /dev/rhel/swap
 LV Name                swap
 VG Name                rhel
 LV UUID                y1ICUQ-l3uA-Mxfc-JupS-c6PN-7jvw-W8wMV6
 LV Write Access        read/write
 LV Creation host, time localhost, 2014-07-21 23:35:55 +0000
 LV Status              available
 # open                 2
 LV Size                1.03 GiB
 Current LE             264
 Segments               1
 Allocation             inherit
 Read ahead sectors     auto
 - currently set to     256
 Block device           253:0

 --- Logical volume ---
 LV Path                /dev/rhel/root
 LV Name                root
 VG Name                rhel
 LV UUID                VOCzB6-0GSH-CEgb-RKYW-9ZKX-R5pr-UPEE1e
 LV Write Access        read/write
 LV Creation host, time localhost, 2014-07-21 23:35:55 +0000
 LV Status              available
 # open                 1
 LV Size                38.48 GiB
 Current LE             9850
 Segments               1
 Allocation             inherit
 Read ahead sectors     auto
 - currently set to     256
 Block device           253:1

```

从`lvdisplay`的输出中，我们可以看到一个名为`/dev/rhel/root`的有趣路径，这个路径也存在于我们的`find`命令的输出中。让我们用`ls`命令来查看这个设备。

```
# ls -la /dev/rhel/root
lrwxrwxrwx. 1 root root 7 Aug  3 16:27 /dev/rhel/root -> ../dm-1

```

在这里，我们可以看到`/dev/rhel/root`是一个指向`/dev/dm-1`的符号链接；这证实了`/dev/rhel/root`与`/dev/dm-1`是相同的，这些实际上是逻辑卷设备，这意味着这些并不是真正的物理设备。

要显示这些逻辑卷背后的物理设备，我们可以使用`pvdisplay`命令。

```
# pvdisplay
 --- Physical volume ---
 PV Name               /dev/sda2
 VG Name               rhel
 PV Size               39.51 GiB / not usable 3.00 MiB
 Allocatable           yes (but full)
 PE Size               4.00 MiB
 Total PE              10114
 Free PE               0
 Allocated PE          10114
 PV UUID               n5xoxm-kvyI-Z7rR-MMcH-1iJI-D68w-NODMaJ

```

我们可以从`pvdisplay`的输出中看到，`dm-1`设备实际上映射到`sda2`，这解释了为什么`dm-1`和`sda`的磁盘利用率非常接近，因为对`dm-1`的任何活动实际上都是在`sda`上执行的。

### 谁在向这些设备写入？

现在我们已经找到了 I/O 的利用情况，我们需要找出谁在利用这个 I/O。找出哪些进程最多地写入磁盘的最简单方法是使用`iotop`命令。这个工具是一个相对较新的命令，现在默认包含在 Red Hat Enterprise Linux 7 中。然而，在以前的 RHEL 版本中，这个命令并不总是可用的。

在采用`iotop`之前，查找使用 I/O 最多的进程的方法涉及使用`ps`命令并浏览`/proc`文件系统。

#### ps - 使用 ps 命令识别利用 I/O 的进程

在收集与 CPU 相关的数据时，我们涵盖了`ps`命令的输出中的状态字段。我们没有涵盖的是进程可能处于的各种状态。以下列表包含了`ps`命令将显示的七种可能的状态：

+   不间断睡眠（`D`）：进程通常在等待 I/O 时处于睡眠状态

+   **运行或可运行**（`R`）：运行队列上的进程

+   **可中断睡眠**（`S`）：等待事件完成但不阻塞 CPU 或 I/O 的进程

+   **已停止**（`T`）：被作业控制系统停止的进程，如 jobs 命令

+   **分页**（`P`）：当前正在分页的进程；但是，在较新的内核上，这不太相关

+   **死亡**（`X`）：已经死亡的进程，不应该出现在运行`ps`时

+   **僵尸**（`Z`）：已终止但保留在不死状态的僵尸进程

在调查 I/O 利用率时，重要的是要识别状态列为`D`的**不间断睡眠**。由于这些进程通常在等待 I/O，它们是最有可能过度利用磁盘 I/O 的进程。

为了做到这一点，我们将使用`ps`命令和`-e`（所有）、`-l`（长格式）和`-f`（完整格式）标志。我们还将再次使用管道将输出重定向到`grep`命令，并将输出过滤为只显示具有`D`状态的进程。

```
# ps -elf | grep " D "
1 D root     13185     2  2  80   0 -     0 get_re 00:21 ? 00:01:32 [kworker/u4:1]
4 D root     15639 15638 30  80   0 -  4233 balanc 01:26 pts/2 00:00:02 bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp

```

从上面的输出中，我们看到有两个进程目前处于不间断睡眠状态。一个进程是`kworker`，这是一个内核系统进程，另一个是`bonnie++`，是由 root 用户启动的进程。由于`kworker`进程是一个通用的内核进程，我们将首先关注`bonnie++`进程。

为了更好地理解这个过程，我们将再次运行`ps`命令，但这次使用`--forest`选项。

```
# ps -elf –forest
4 S root      1007     1  0  80   0 - 20739 poll_s Feb07 ? 00:00:00 /usr/sbin/sshd -D
4 S root     11239  1007  0  80   0 - 32881 poll_s Feb08 ? 00:00:00  \_ sshd: vagrant [priv]
5 S vagrant  11242 11239  0  80   0 - 32881 poll_s Feb08 ? 00:00:02      \_ sshd: vagrant@pts/2
0 S vagrant  11243 11242  0  80   0 - 28838 wait   Feb08 pts/2 00:00:01          \_ -bash
4 S root     16052 11243  0  80   0 - 47343 poll_s 01:39 pts/2 00:00:00              \_ sudo bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp
4 S root     16053 16052 32  80   0 - 96398 hrtime 01:39 pts/2 00:00:03                  \_ bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp

```

通过审查上述输出，我们可以看到`bonnie++`进程实际上是进程`16052`的子进程，后者是`11243`的另一个子进程，后者是`vagrant`用户的 bash shell。

前面的`ps`命令告诉我们，进程 ID 为`16053`的`bonnie++`进程正在等待 I/O 任务。但是，这并没有告诉我们这个进程正在使用多少 I/O；为了确定这一点，我们可以读取`/proc`文件系统中的一个特殊文件，名为`io`。

```
# cat /proc/16053/io
rchar: 1002448848
wchar: 1002438751
syscr: 122383
syscw: 122375
read_bytes: 1002704896
write_bytes: 1002438656
cancelled_write_bytes: 0

```

每个运行的进程在`/proc`中都有一个与进程`id`同名的子文件夹；对于我们的示例，这是`/proc/16053`。这个文件夹由内核维护，用于每个运行的进程，在这些文件夹中存在许多包含有关运行进程信息的文件。

这些文件非常有用，它们实际上是`ps`命令信息的来源之一。其中一个有用的文件名为`io`；`io`文件包含有关进程执行的读取和写入次数的统计信息。

从 cat 命令的输出中，我们可以看到这个进程已经读取和写入了大约 1GB 的数据。虽然这看起来很多，但可能是在很长一段时间内完成的。为了了解这个进程向磁盘写入了多少数据，我们可以再次读取这个文件以捕捉差异。

```
# cat /proc/16053/io
cat: /proc/16053/io: No such file or directory

```

然而，当我们第二次执行 cat 命令时，我们收到了一个错误，即`io`文件不再存在。如果我们再次运行`ps`命令并使用`grep`在输出中搜索`bonnie++`进程，我们会发现`bonnie++`进程正在运行；但是，它是一个新的进程，具有新的进程`ID`。

```
# ps -elf | grep bonnie
4 S root     17891 11243  0  80   0 - 47343 poll_s 02:34 pts/2 00:00:00 sudo bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp
4 D root     17892 17891 33  80   0 -  4233 sleep_ 02:34 pts/2 00:00:02 bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp

```

由于`bonnie++`子进程是短暂的进程，通过读取`io`文件来跟踪 I/O 统计可能会非常困难。

### iotop - 一个用于磁盘 I/O 的类似 top 的命令

由于这些进程频繁启动和停止，我们可以使用`iotop`命令来确定哪些进程最多地利用了 I/O。

```
# iotop
Total DISK READ :     102.60 M/s | Total DISK WRITE :      26.96 M/s
Actual DISK READ:     102.60 M/s | Actual DISK WRITE:      42.04 M/s
 TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO> COMMAND
16395 be/4 root        0.00 B/s    0.00 B/s  0.00 % 45.59 % [kworker/u4:0]
18250 be/4 root      101.95 M/s   26.96 M/s  0.00 % 42.59 % bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp

```

在`iotop`的输出中，我们可以看到一些有趣的 I/O 统计信息。通过`iotop`，我们不仅可以看到系统范围的统计信息，比如每秒的**总磁盘读取**和**总磁盘写入**，还可以看到单个进程的许多统计信息。

从每个进程的角度来看，我们可以看到`bonnie++`进程正在以 101.96 MBps 的速度从磁盘读取数据，并以 26.96 MBps 的速度向磁盘写入数据。

```
16395 be/4 root        0.00 B/s    0.00 B/s  0.00 % 45.59 % [kworker/u4:0]
18250 be/4 root      101.95 M/s   26.96 M/s  0.00 % 42.59 % bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp

```

`iotop`命令与 top 命令非常相似，它会每隔几秒刷新报告的结果。这样做的效果是实时显示 I/O 统计信息。

### 提示

诸如`top`和`iotop`之类的命令在书本格式中很难展示。我强烈建议在具有这些命令的系统上执行这些命令，以了解它们的工作方式。

### 整合起来

现在我们已经介绍了一些用于故障排除磁盘性能和利用率的工具，让我们在解决报告的缓慢时将它们整合起来。

#### 使用 iostat 来确定是否存在 I/O 带宽问题

我们将首先运行的命令是`iostat`，因为这将首先为我们验证是否确实存在问题。

```
# iostat -x 10 3
Linux 3.10.0-123.el7.x86_64 (blog.example.com)   02/09/2015 _x86_64_  (2 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 38.58    0.00    3.22    5.46    0.00   52.75

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda              10.86     4.25  122.46  118.15 11968.97 12065.60 199.78    13.27   55.18    0.67  111.67   0.51  12.21
dm-0              0.00     0.00   14.03    3.44    56.14    13.74 8.00     0.42   24.24    0.51  121.15   0.46   0.80
dm-1              0.00     0.00  119.32  112.35 11912.79 12051.98 206.89    13.52   58.33    0.68  119.55   0.52  12.16

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 7.96    0.00   14.60   29.31    0.00   48.12

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.70     0.80  804.49  776.85 79041.12 76999.20 197.35    64.26   41.41    0.54   83.73   0.42  66.38
dm-0              0.00     0.00    0.90    0.80     3.59     3.19 8.00     0.08   50.00    0.00  106.25  19.00   3.22
dm-1              0.00     0.00  804.29  726.35 79037.52 76893.81 203.75    64.68   43.03    0.53   90.08   0.44  66.75

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
 5.22    0.00   11.21   36.21    0.00   47.36

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               1.10     0.30  749.40  429.70 84589.20 43619.80 217.47    76.31   66.49    0.43  181.69   0.58  68.32
dm-0              0.00     0.00    1.30    0.10     5.20     0.40 8.00     0.00    2.21    1.00   18.00   1.43   0.20
dm-1              0.00     0.00  749.00  391.20 84558.40 41891.80 221.80    76.85   69.23    0.43  200.95   0.60  68.97

```

从`iostat`的输出中，我们可以确定以下信息：

+   该系统的 CPU 目前花费了相当多的时间在等待 I/O，占 30%–40%。

+   看起来`dm-1`和`sda`设备是利用率最高的设备

+   从`iostat`来看，这些设备的利用率为 68%，这个数字似乎相当高

基于这些数据点，我们可以确定存在潜在的 I/O 利用率问题，除非 68%的利用率是预期的。

#### 使用 iotop 来确定哪些进程正在消耗磁盘带宽

现在我们已经确定了大量的 CPU 时间被用于等待 I/O，我们现在应该关注哪些进程最多地利用了磁盘。为了做到这一点，我们将使用`iotop`命令。

```
# iotop
Total DISK READ :     100.64 M/s | Total DISK WRITE :      23.91 M/s
Actual DISK READ:     100.67 M/s | Actual DISK WRITE:      38.04 M/s
 TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO> COMMAND
19358 be/4 root        0.00 B/s    0.00 B/s  0.00 % 40.38 % [kworker/u4:1]
20262 be/4 root      100.35 M/s   23.91 M/s  0.00 % 33.65 % bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp
 363 be/4 root        0.00 B/s    0.00 B/s  0.00 %  2.51 % [xfsaild/dm-1]
 32 be/4 root        0.00 B/s    0.00 B/s  0.00 %  1.74 % [kswapd0]

```

从`iotop`命令中，我们可以看到进程`20262`，它正在运行`bonnie++`命令，具有高利用率以及大量的磁盘读写值。

从`iotop`中，我们可以确定以下信息：

+   系统的每秒总磁盘读取量为 100.64 MBps

+   系统的每秒总磁盘写入量为 23.91 MBps

+   运行`bonnie++`命令的进程`20262`正在读取 100.35 MBps，写入 23.91 MBps

+   比较总数，我们发现进程`20262`是磁盘读写的主要贡献者

鉴于上述情况，似乎我们需要更多地了解进程`20262`的信息。

#### 使用 ps 来更多地了解进程

现在我们已经确定了一个使用大量 I/O 的进程，我们可以使用`ps`命令来调查这个进程的详细信息。我们将再次使用带有`--forest`标志的`ps`命令来显示父进程和子进程的关系。

```
# ps -elf --forest
1007  0  80   0 - 32881 poll_s Feb08 ?        00:00:00  \_ sshd: vagrant [priv]
5 S vagrant  11242 11239  0  80   0 - 32881 poll_s Feb08 ? 00:00:05      \_ sshd: vagrant@pts/2
0 S vagrant  11243 11242  0  80   0 - 28838 wait   Feb08 pts/2 00:00:02          \_ -bash
4 S root     20753 11243  0  80   0 - 47343 poll_s 03:52 pts/2 00:00:00              \_ sudo bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp
4 D root     20754 20753 52  80   0 -  4233 sleep_ 03:52 pts/2 00:00:01                  \_ bonnie++ -n 0 -u 0 -r 239 -s 478 -f -b -d /tmp

```

使用`ps`命令，我们可以确定以下内容：

+   用`iotop`识别的`bonnie++`进程`20262`不见了；然而，其他`bonnie++`进程存在

+   `vagrant`用户已经通过使用`sudo`命令启动了父`bonnie++`进程

+   `vagrant`用户与早期观察中讨论的 CPU 和内存部分的用户相同

鉴于上述细节，似乎`vagrant`用户是我们性能问题的嫌疑人。

## 网络

性能问题的最后一个常见资源是网络。有许多工具可以用来排除网络问题；然而，这些命令中很少有专门针对网络性能的。大多数这些工具都是为深入的网络故障排除而设计的。

由于第五章，*网络故障排除*专门用于解决网络问题，本节将专门关注性能。

### ifstat - 查看接口统计

在网络方面，有大约四个指标可以用来衡量吞吐量。

+   **接收数据包**：接口接收的数据包数量

+   **发送数据包**：接口发送的数据包数量

+   **接收数据**：接口接收的数据量

+   **发送数据**：接口发送的数据量

有许多命令可以提供这些指标，从`ifconfig`或`ip`到`netstat`都有。一个非常有用的专门输出这些指标的实用程序是`ifstat`命令。

```
# ifstat
#21506.1804289383 sampling_interval=5 time_const=60
Interface   RX Pkts/Rate   TX Pkts/Rate   RX Data/Rate   TX Data/Rate
 RX Errs/Drop   TX Errs/Drop   RX Over/Rate   TX Coll/Rate
lo              47 0            47 0         4560 0          4560 0
 0 0             0 0            0 0             0 0
enp0s3       70579 1         50636 0      17797K 65        5520K 96
 0 0             0 0            0 0             0 0
enp0s8       23034 0            43 0       2951K 18          7035 0
 0 0             0 0            0 0             0 0

```

与`vmstat`或`iostat`类似，`ifstat`生成的第一个报告是基于服务器上次重启以来的统计数据。这意味着上面的报告表明`enp0s3`接口自上次重启以来已接收了 70,579 个数据包。

当第二次执行`ifstat`时，结果将与第一个报告有很大的差异。原因是第二个报告是基于自第一个报告以来的时间。

```
# ifstat
#21506.1804289383 sampling_interval=5 time_const=60
Interface   RX Pkts/Rate    TX Pkts/Rate   RX Data/Rate  TX Data/Rate
 RX Errs/Drop    TX Errs/Drop   RX Over/Rate  TX Coll/Rate
lo                0 0             0 0             0 0             0 0
 0 0             0 0             0 0             0 0
enp0s3           23 0            18 0         1530 59         1780 80
 0 0             0 0             0 0             0 0
enp0s8            1 0             0 0           86 10             0 0
 0 0             0 0             0 0             0 0

```

在上面的例子中，我们可以看到我们的系统通过`enp0s3`接口接收了 23 个数据包（RX Pkts）并发送了 18 个数据包（`TX Pkts`）。

通过`ifstat`命令，我们可以确定以下关于我们的系统的内容：

+   目前的网络利用率相当小，不太可能对整个系统造成影响

+   早期显示的`vagrant`用户的进程不太可能利用大量网络资源

根据`ifstat`所见的统计数据，在这个系统上几乎没有网络流量，不太可能导致感知到的缓慢。

## 对我们已经确定的内容进行快速回顾

在继续之前，让我们回顾一下到目前为止我们从性能统计数据中学到的东西：

### 注意

`vagrant`用户一直在启动运行`bonnie++`和`lookbusy`应用程序的进程。

`lookbusy`应用程序似乎要么一直占用整个系统 CPU 的 20%–30%。

这个服务器有两个 CPU，`lookbusy`似乎一直占用一个 CPU 的大约 60%。

`lookbusy`应用程序似乎也一直使用大约 200 MB 的内存；然而，在故障排除期间，我们确实看到这些进程几乎使用了系统的所有内存，导致系统交换。

在启动`bonnie++`进程时，`vagrant`用户的系统经历了高 I/O 等待时间。

在运行时，`bonnie++`进程利用了大约 60%–70%的磁盘吞吐量。

`vagrant`用户正在执行的活动似乎对网络利用率几乎没有影响。

# 比较历史指标

从迄今为止我们了解到的所有事实来看，我们下一个最佳行动方案似乎是建议联系`vagrant`用户，以确定`lookbusy`和`bonnie++`应用程序是否应该以如此高的资源利用率运行。

尽管先前的观察显示了高资源利用率，但这种利用率水平可能是预期的。在开始联系用户之前，我们应该首先审查服务器的历史性能指标。在大多数环境中，都会有一些服务器性能监控软件，如 Munin、Cacti 或许多云 SaaS 提供商之一，用于收集和存储系统统计信息。

如果您的环境使用了这些服务，您可以使用收集的性能数据来将以前的性能统计与我们刚刚收集到的信息进行比较。例如，在过去 30 天中，CPU 性能从未超过 10%，那么`lookbusy`进程可能在那个时候没有运行。

即使您的环境没有使用这些工具之一，您仍然可以执行历史比较。为此，我们将使用一个默认安装在大多数 Red Hat Enterprise Linux 系统上的工具；这个工具叫做`sar`。

## sar – 系统活动报告

在第二章，*故障排除命令和有用信息来源*中，我们简要讨论了使用`sar`命令来查看历史性能统计信息。

当安装了部署`sar`实用程序的`sysstat`软件包时，它将部署`/etc/cron.d/sysstat`文件。在这个文件中有两个`cron`作业，运行`sysstat`命令，其唯一目的是收集系统性能统计信息并生成收集信息的报告。

```
$ cat /etc/cron.d/sysstat
# Run system activity accounting tool every 10 minutes
*/2 * * * * root /usr/lib64/sa/sa1 1 1
# 0 * * * * root /usr/lib64/sa/sa1 600 6 &
# Generate a daily summary of process accounting at 23:53
53 23 * * * root /usr/lib64/sa/sa2 -A

```

当执行这些命令时，收集的信息将存储在`/var/log/sa/`文件夹中。

```
# ls -la /var/log/sa/
total 1280
drwxr-xr-x. 2 root root   4096 Feb  9 00:00 .
drwxr-xr-x. 9 root root   4096 Feb  9 03:17 ..
-rw-r--r--. 1 root root  68508 Feb  1 23:20 sa01
-rw-r--r--. 1 root root  40180 Feb  2 16:00 sa02
-rw-r--r--. 1 root root  28868 Feb  3 05:30 sa03
-rw-r--r--. 1 root root  91084 Feb  4 20:00 sa04
-rw-r--r--. 1 root root  57148 Feb  5 23:50 sa05
-rw-r--r--. 1 root root  34524 Feb  6 23:50 sa06
-rw-r--r--. 1 root root 105224 Feb  7 23:50 sa07
-rw-r--r--. 1 root root 235312 Feb  8 23:50 sa08
-rw-r--r--. 1 root root 105224 Feb  9 06:00 sa09
-rw-r--r--. 1 root root  56616 Jan 23 23:00 sa23
-rw-r--r--. 1 root root  56616 Jan 24 20:10 sa24
-rw-r--r--. 1 root root  24648 Jan 30 23:30 sa30
-rw-r--r--. 1 root root  11948 Jan 31 23:20 sa31
-rw-r--r--. 1 root root  44476 Feb  5 23:53 sar05
-rw-r--r--. 1 root root  27244 Feb  6 23:53 sar06
-rw-r--r--. 1 root root  81094 Feb  7 23:53 sar07
-rw-r--r--. 1 root root 180299 Feb  8 23:53 sar08

```

`sysstat`软件包生成的数据文件使用遵循“`sa<两位数的日期>`”格式的文件名。例如，在上面的输出中，我们可以看到“`sa24`”文件是在 1 月 24 日生成的。我们还可以看到这个系统有从 1 月 23 日到 2 月 9 日的文件。

`sar`命令是一个允许我们读取这些捕获的性能指标的命令。本节将向您展示如何使用`sar`命令来查看与`iostat`、`top`和`vmstat`等命令之前查看的相同统计信息。然而，这次`sar`命令将提供最近和历史信息。

### CPU

要使用`sar`命令查看 CPU 统计信息，我们可以简单地使用`–u`（CPU 利用率）标志。

```
# sar -u
Linux 3.10.0-123.el7.x86_64 (blog.example.com)   02/09/2015   _x86_64_  (2 CPU)

12:00:01 AM     CPU     %user     %nice   %system   %iowait    %steal %idle
12:10:02 AM     all      7.42      0.00     13.46     37.51      0.00 41.61
12:20:01 AM     all      7.59      0.00     13.61     38.55      0.00 40.25
12:30:01 AM     all      7.44      0.00     13.46     38.50      0.00 40.60
12:40:02 AM     all      8.62      0.00     15.71     31.42      0.00 44.24
12:50:02 AM     all      8.77      0.00     16.13     29.66      0.00 45.44
01:00:01 AM     all      8.88      0.00     16.20     29.43      0.00 45.49
01:10:01 AM     all      7.46      0.00     13.64     37.29      0.00 41.61
01:20:02 AM     all      7.35      0.00     13.52     37.79      0.00 41.34
01:30:01 AM     all      7.40      0.00     13.36     38.60      0.00 40.64
01:40:01 AM     all      7.42      0.00     13.53     37.86      0.00 41.19
01:50:01 AM     all      7.44      0.00     13.58     38.38      0.00 40.60
04:20:02 AM     all      7.51      0.00     13.72     37.56      0.00 41.22
04:30:01 AM     all      7.34      0.00     13.36     38.56      0.00 40.74
04:40:02 AM     all      7.40      0.00     13.41     37.94      0.00 41.25
04:50:01 AM     all      7.45      0.00     13.81     37.73      0.00 41.01
05:00:02 AM     all      7.49      0.00     13.75     37.72      0.00 41.04
05:10:01 AM     all      7.43      0.00     13.30     39.28      0.00 39.99
05:20:02 AM     all      7.24      0.00     13.17     38.52      0.00 41.07
05:30:02 AM     all     13.47      0.00     11.10     31.12      0.00 44.30
05:40:01 AM     all     67.05      0.00      1.92      0.00      0.00 31.03
05:50:01 AM     all     68.32      0.00      1.85      0.00      0.00 29.82
06:00:01 AM     all     69.36      0.00      1.76      0.01      0.00 28.88
06:10:01 AM     all     70.53      0.00      1.71      0.01      0.00 27.76
Average:        all     14.43      0.00     12.36     33.14      0.00 40.07

```

如果我们从上面的头信息中查看，我们可以看到带有`-u`标志的`sar`命令与`iostat`和 top CPU 详细信息相匹配。

```
12:00:01 AM     CPU     %user     %nice   %system   %iowait    %steal %idle

```

从`sar -u`的输出中，我们可以发现一个有趣的趋势：从 00:00 到 05:30，CPU I/O 等待时间保持在 30%–40%。然而，从 05:40 开始，I/O 等待时间减少，但用户级 CPU 利用率增加到 65%–70%。

尽管这两个测量并没有明确指向任何一个过程，但它们表明 I/O 等待时间最近已经减少，而用户 CPU 时间已经增加。

为了更好地了解历史统计信息，我们需要查看前一天的 CPU 利用率。幸运的是，我们可以使用`–f`（文件名）标志来做到这一点。`–f`标志将允许我们为`sar`命令指定一个历史文件。这将允许我们有选择地查看前一天的统计信息。

```
# sar -f /var/log/sa/sa07 -u
Linux 3.10.0-123.el7.x86_64 (blog.example.com)   02/07/2015 _x86_64_  (2 CPU)

12:00:01 AM     CPU     %user     %nice   %system   %iowait    %steal %idle
12:10:01 AM     all     24.63      0.00      0.71      0.00      0.00 74.66
12:20:01 AM     all     25.31      0.00      0.70      0.00      0.00 73.99
01:00:01 AM     all     27.59      0.00      0.68      0.00      0.00 71.73
01:10:01 AM     all     29.64      0.00      0.71      0.00      0.00 69.65
05:10:01 AM     all     44.09      0.00      0.63      0.00      0.00 55.28
05:20:01 AM     all     60.94      0.00      0.58      0.00      0.00 38.48
05:30:01 AM     all     62.32      0.00      0.56      0.00      0.00 37.12
05:40:01 AM     all     63.74      0.00      0.56      0.00      0.00 35.70
05:50:01 AM     all     65.08      0.00      0.56      0.00      0.00 34.35
0.00     76.07
Average:        all     37.98      0.00      0.65      0.00      0.00 61.38

```

在 2 月 7 日的报告中，我们可以看到 CPU 利用率与我们之前的故障排除所发现的情况有很大的不同。一个突出的问题是，在 7 日的报告中，没有 CPU 时间花费在 I/O 等待状态。

然而，我们看到用户 CPU 时间根据一天中的时间波动从 20%到 65%不等。这可能表明预期会有更高的用户 CPU 时间利用率。

### 内存

要显示内存统计信息，我们可以使用带有`-r`（内存）标志的`sar`命令。

```
# sar -r
Linux 3.10.0-123.el7.x86_64 (blog.example.com)   02/09/2015 _x86_64_  (2 CPU)

12:00:01 AM kbmemfree kbmemused  %memused kbbuffers  kbcached kbcommit   %commit  kbactive   kbinact   kbdirty
12:10:02 AM     38228    463832     92.39         0    387152 446108     28.17    196156    201128         0
12:20:01 AM     38724    463336     92.29         0    378440 405128     25.59    194336    193216     73360
12:30:01 AM     38212    463848     92.39         0    377848 405128     25.59      9108    379348     58996
12:40:02 AM     37748    464312     92.48         0    387500 446108     28.17    196252    201684         0
12:50:02 AM     33028    469032     93.42         0    392240 446108     28.17    196872    205884         0
01:00:01 AM     34716    467344     93.09         0    380616 405128     25.59    195900    195676     69332
01:10:01 AM     31452    470608     93.74         0    384092 396660     25.05    199100    196928     74372
05:20:02 AM     38756    463304     92.28         0    387120 399996     25.26    197184    198456         4
05:30:02 AM    187652    314408     62.62         0     19988 617000     38.97    222900     22524         0
05:40:01 AM    186896    315164     62.77         0     20116 617064     38.97    223512     22300         0
05:50:01 AM    186824    315236     62.79         0     20148 617064     38.97    223788     22220         0
06:00:01 AM    182956    319104     63.56         0     24652 615888     38.90    226744     23288         0
06:10:01 AM    176992    325068     64.75         0     29232 615880     38.90    229356     26500         0
06:20:01 AM    176756    325304     64.79         0     29480 615884     38.90    229448     26588         0
06:30:01 AM    176636    325424     64.82         0     29616 615888     38.90    229516     26820         0
Average:        77860    424200     84.49         0    303730 450102     28.43    170545    182617     29888

```

再次，如果我们查看`sar`的内存报告标题，我们可以看到一些熟悉的值。

```
12:00:01 AM kbmemfree kbmemused  %memused kbbuffers  kbcached kbcommit   %commit  kbactive   kbinact   kbdirty

```

从这份报告中，我们可以看到在 05:40 时，系统突然释放了 150MB 的物理内存。从`kbcached`列可以看出，这 150MB 的内存被分配给了磁盘缓存。这是基于 05:40 时，缓存内存从 196MB 下降到 22MB 的事实。

有趣的是，这与 CPU 利用率的变化在 05:40 也是一致的。如果我们希望回顾历史内存利用情况，我们也可以使用带有`-f`（文件名）标志和`-r`（内存）标志。然而，由于我们可以看到 05:40 有一个相当明显的趋势，我们现在将重点放在这个时间上。

### 磁盘

要显示今天的磁盘统计信息，我们可以使用`-d`（块设备）标志。

```
# sar -d
Linux 3.10.0-123.el7.x86_64 (blog.example.com)   02/09/2015 _x86_64_  (2 CPU)

12:00:01 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
12:10:02 AM    dev8-0   1442.64 150584.15 146120.49    205.67 82.17     56.98      0.51     74.17
12:10:02 AM  dev253-0      1.63     11.11      1.96      8.00 0.06     34.87     19.72      3.22
12:10:02 AM  dev253-1   1402.67 150572.19 146051.96    211.47 82.73     58.98      0.53     74.68
04:20:02 AM    dev8-0   1479.72 152799.09 150240.77    204.80 81.27     54.89      0.50     73.86
04:20:02 AM  dev253-0      1.74     10.98      2.96      8.00 0.06     31.81     14.60      2.54
04:20:02 AM  dev253-1   1438.57 152788.11 150298.01    210.69 81.84     56.83      0.52     74.38
05:30:02 AM  dev253-0      1.00      7.83      0.17      8.00 0.00      3.81      2.76      0.28
05:30:02 AM  dev253-1   1170.61 123647.27 122655.72    210.41 69.12     59.04      0.53     62.20
05:40:01 AM    dev8-0      0.08      1.00      0.34     16.10 0.00      1.88      1.00      0.01
05:40:01 AM  dev253-0      0.11      0.89      0.00      8.00 0.00      1.57      0.25      0.00
05:40:01 AM  dev253-1      0.05      0.11      0.34      8.97 0.00      2.77      1.17      0.01
05:50:01 AM    dev8-0      0.07      0.49      0.28     11.10 0.00      1.71      1.02      0.01
05:50:01 AM  dev253-0      0.06      0.49      0.00      8.00 0.00      2.54      0.46      0.00
05:50:01 AM  dev253-1      0.05      0.00      0.28      6.07 0.00      1.96      0.96      0.00

Average:          DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz avgqu-sz     await     svctm     %util
Average:       dev8-0   1215.88 125807.06 123583.62    205.11 66.86     55.01      0.50     60.82
Average:     dev253-0      2.13     12.48      4.53      8.00 0.10     44.92     17.18      3.65
Average:     dev253-1   1181.94 125794.56 123577.42    210.99 67.31     56.94      0.52     61.17

```

默认情况下，`sar`命令将打印设备名称为“`dev<major>-<minor>`”，这可能有点令人困惑。如果添加了`-p`（持久名称）标志，设备名称将使用持久名称，与挂载命令中的设备匹配。

```
# sar -d -p
Linux 3.10.0-123.el7.x86_64 (blog.example.com)   08/16/2015 _x86_64_  (4 CPU)

01:46:42 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
01:48:01 AM       sda      0.37      0.00      3.50      9.55 0.00      1.86      0.48      0.02
01:48:01 AM rhel-swap      0.00      0.00      0.00      0.00 0.00      0.00      0.00      0.00
01:48:01 AM rhel-root      0.37      0.00      3.50      9.55 0.00      2.07      0.48      0.02

```

即使名称以不可识别的格式显示，我们也可以看到`dev253-1`似乎在 05:40 之前有相当多的活动，磁盘`tps`（每秒事务）从 1170 下降到 0.11。磁盘 I/O 利用率的大幅下降似乎表明今天在 05:40 发生了相当大的变化。

### 网络

要显示网络统计信息，我们需要使用带有`-n DEV`标志的`sar`命令。

```
# sar -n DEV
Linux 3.10.0-123.el7.x86_64 (blog.example.com)   02/09/2015 _x86_64_  (2 CPU)

12:00:01 AM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s rxcmp/s   txcmp/s  rxmcst/s
12:10:02 AM    enp0s3      1.51      1.18      0.10      0.12 0.00      0.00      0.00
12:10:02 AM    enp0s8      0.14      0.00      0.02      0.00 0.00      0.00      0.07
12:10:02 AM        lo      0.00      0.00      0.00      0.00 0.00      0.00      0.00
12:20:01 AM    enp0s3      0.85      0.85      0.05      0.08 0.00      0.00      0.00
12:20:01 AM    enp0s8      0.18      0.00      0.02      0.00 0.00      0.00      0.08
12:20:01 AM        lo      0.00      0.00      0.00      0.00 0.00      0.00      0.00
12:30:01 AM    enp0s3      1.45      1.16      0.10      0.11 0.00      0.00      0.00
12:30:01 AM    enp0s8      0.18      0.00      0.03      0.00 0.00      0.00      0.08
12:30:01 AM        lo      0.00      0.00      0.00      0.00 0.00      0.00      0.00
05:20:02 AM        lo      0.00      0.00      0.00      0.00 0.00      0.00      0.00
05:30:02 AM    enp0s3      1.23      1.02      0.08      0.11 0.00      0.00      0.00
05:30:02 AM    enp0s8      0.15      0.00      0.02      0.00 0.00      0.00      0.04
05:30:02 AM        lo      0.00      0.00      0.00      0.00 0.00      0.00      0.00
05:40:01 AM    enp0s3      0.79      0.78      0.05      0.14 0.00      0.00      0.00
05:40:01 AM    enp0s8      0.18      0.00      0.02      0.00 0.00      0.00      0.08
05:40:01 AM        lo      0.00      0.00      0.00      0.00 0.00      0.00      0.00
05:50:01 AM    enp0s3      0.76      0.75      0.05      0.13 0.00      0.00      0.00
05:50:01 AM    enp0s8      0.16      0.00      0.02      0.00 0.00      0.00      0.07
05:50:01 AM        lo      0.00      0.00      0.00      0.00 0.00      0.00      0.00
06:00:01 AM    enp0s3      0.67      0.60      0.04      0.10 0.00      0.00      0.00

```

在网络统计报告中，我们看到整天都没有变化。这表明，总体上，这台服务器从未出现与网络性能瓶颈相关的问题。

## 通过比较历史统计数据来回顾我们所学到的内容

通过使用`sar`查看历史统计数据和使用`ps`、`iostat`、`vmstat`和`top`等命令查看最近的统计数据后，我们可以得出关于我们的“性能慢”的以下结论。

由于我们被同事要求调查这个问题，我们的结论将以电子邮件回复的形式发送给这位同事。

*嗨鲍勃！*

*我调查了一个用户说服务器“慢”的服务器。看起来用户 vagrant 一直在运行两个主要程序的多个实例。第一个是 lookbusy 应用程序，似乎始终使用大约 20%–40%的 CPU。然而，至少有一个实例中，lookbusy 应用程序还使用了大量内存，耗尽了物理内存并迫使系统大量交换。然而，这个过程并没有持续很长时间。*

*第二个程序是 bonnie++应用程序，似乎利用了大量的磁盘 I/O 资源。当 vagrant 用户运行 bonnie++应用程序时，它占用了大约 60%的 dm-1 和 sda 磁盘带宽，导致了大约 30%的高 I/O 等待。通常，这个系统的 I/O 等待为 0%（通过 sar 确认）。*

*看起来 vagrant 用户可能正在运行超出预期水平的应用程序，导致其他用户的性能下降。*

# 总结

在本章中，我们开始使用一些高级的 Linux 命令，这些命令在第二章中进行了探索，例如`iostat`和`vmstat`。我们还对 Linux 中的一个基本实用程序`ps`命令非常熟悉，同时解决了一个模糊的性能问题。

在第三章中，*故障排除 Web 应用程序*，我们能够从数据收集到试错的完整故障排除过程，而在本章中，我们的行动主要集中在数据收集和建立假设阶段。发现自己只是在解决问题而不是执行纠正措施是非常常见的。有许多问题应该由系统的用户而不是系统管理员来解决，但管理员的角色仍然是识别问题的来源。

在第五章中，*网络故障排除*，我们将解决一些非常有趣的网络问题。网络对于任何系统都至关重要；问题有时可能很简单，而有时则非常复杂。在下一章中，我们将探讨网络和如何使用诸如`netstat`和`tcpdump`之类的工具来排除网络问题。
