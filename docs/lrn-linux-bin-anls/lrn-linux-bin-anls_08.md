# 第八章。ECFS – 扩展核心文件快照技术

**扩展核心文件快照**（**ECFS**）技术是一款插入 Linux 核心处理程序并创建专门设计用于进程内存取证的特殊进程内存快照的软件。大多数人不知道如何解析进程镜像，更不用说如何检查其中的异常。即使对于专家来说，查看进程镜像并检测感染或恶意软件可能是一项艰巨的任务。

在 ECFS 之前，除了使用大多数 Linux 发行版附带的**gcore**脚本创建的核心文件之外，没有真正的进程镜像快照标准。如前一章简要讨论的那样，常规核心文件对于进程取证分析并不特别有用。这就是 ECFS 核心文件出现的原因——提供一种可以描述进程镜像的每一个细微差别的文件格式，以便可以进行高效分析、轻松导航，并且可以轻松集成到恶意软件分析和进程取证工具中。

在本章中，我们将讨论 ECFS 的基础知识以及如何使用 ECFS 核心文件和**libecfs** API 来快速设计恶意软件分析和取证工具。

# 历史

2011 年，我为 DARPA 合同创建了一个名为 Linux VMA Monitor 的软件原型([`www.bitlackeys.org/#vmavudu`](http://www.bitlackeys.org/#vmavudu))。这个软件旨在查看实时进程内存或进程内存的原始快照。它能够检测各种运行时感染，包括共享库注入、PLT/GOT 劫持和其他指示运行时恶意软件的异常。

最近，我考虑将这个软件重写为更完善的状态，我觉得为进程内存创建一个本地快照格式将是一个非常好的功能。这是开发 ECFS 的最初灵感，尽管我已经取消了重新启动 Linux VMA Monitor 软件的计划，但我仍在继续扩展和开发 ECFS 软件，因为它对其他许多人的项目非常有价值。它甚至被整合到了 Lotan 产品中，这是一款用于通过分析崩溃转储来检测利用尝试的软件([`www.leviathansecurity.com/lotan`](http://www.leviathansecurity.com/lotan))。

# ECFS 的理念

ECFS 的目标是使程序的运行时分析比以往任何时候都更容易。整个过程都封装在一个单一文件中，并且以一种有序和高效的方式组织，以便通过解析部分头来访问有用的数据，如符号表、动态链接数据和取证相关结构，从而实现定位和访问对于检测异常和感染至关重要的数据和代码。 

# 开始使用 ECFS

撰写本章时，完整的 ECFS 项目和源代码可在[`github.com/elfmaster/ecfs`](http://github.com/elfmaster/ecfs)上找到。一旦你用 git 克隆了存储库，你应该按照 README 文件中的说明编译和安装软件。

目前，ECFS 有两种使用模式：

+   将 ECFS 插入核心处理程序

+   ECFS 快照而不终止进程

### 注意

在本章中，术语 ECFS 文件、ECFS 快照和 ECFS 核心文件是可以互换使用的。

## 将 ECFS 插入核心处理程序

首先要做的是将 ECFS 核心处理程序插入 Linux 内核中。`make` install 会为您完成这项工作，但必须在每次重启后进行操作，或者存储在一个`init`脚本中。手动设置 ECFS 核心处理程序的方法是修改`/proc/sys/kernel/core_pattern`文件。

这是激活 ECFS 核心处理程序的命令：

```
echo '|/opt/ecfs/bin/ecfs_handler -t -e %e -p %p -o \ /opt/ecfs/cores/%e.%p' > /proc/sys/kernel/core_pattern
```

### 注意

请注意设置了`-t`选项。这对取证非常重要，而且很少关闭。此选项告诉 ECFS 捕获任何可执行文件或共享库映射的整个文本段。在传统核心文件中，文本图像被截断为 4k。在本章的后面，我们还将研究`-h`选项（启发式），它可以设置为启用扩展启发式以检测共享库注入。

`ecfs_handler`二进制文件将调用`ecfs32`或`ecfs64`，具体取决于进程是 64 位还是 32 位。我们写入 procfs `core_pattern`条目的行前面的管道符（`|`）告诉内核将其产生的核心文件导入到我们的 ECFS 核心处理程序进程的标准输入中。然后 ECFS 核心处理程序将传统核心文件转换为高度定制和出色的 ECFS 核心文件。每当进程崩溃或收到导致核心转储的信号，例如**SIGSEGV**或**SIGABRT**，那么 ECFS 核心处理程序将介入并使用自己的一套特殊程序来创建 ECFS 风格的核心转储。

以下是捕获`sshd`的 ECFS 快照的示例：

```
$ kill -ABRT `pidof sshd`
$ ls -lh /opt/ecfs/cores
-rwxrwx--- 1 root root 8244638 Jul 24 13:36 sshd.1211
$
```

将 ECFS 作为默认的核心文件处理程序非常好，非常适合日常使用。这是因为 ECFS 核心向后兼容传统核心文件，并且可以与诸如 GDB 之类的调试器一起使用。但是，有时用户可能希望捕获 ECFS 快照而无需终止进程。这就是 ECFS 快照工具的用处所在。

## 在不终止进程的情况下进行 ECFS 快照

让我们考虑一个场景，有一个可疑的进程正在运行。它可疑是因为它消耗了大量的 CPU，并且它打开了网络套接字，尽管已知它不是任何类型的网络程序。在这种情况下，可能希望让进程继续运行，以便潜在的攻击者尚未被警告，但仍然具有生成 ECFS 核心文件的能力。在这些情况下应该使用`ecfs_snapshot`实用程序。

`ecfs_snapshot`实用程序最终使用 ptrace 系统调用，这意味着两件事：

+   捕获进程的快照可能需要更长的时间。

+   它可能对使用反调试技术防止 ptrace 附加的进程无效

在这些问题中的任何一个成为问题的情况下，您可能需要考虑使用 ECFS 核心处理程序来处理工作，这种情况下您将不得不终止进程。然而，在大多数情况下，`ecfs_snapshot`实用程序将起作用。

以下是使用快照实用程序捕获 ECFS 快照的示例：

```
$ ecfs_snapshot -p `pidof host` -o host_snapshot
```

这为程序 host 捕获了快照，并创建了一个名为`host_snapshot`的 ECFS 快照。在接下来的章节中，我们将演示 ECFS 的一些实际用例，并使用各种实用程序查看 ECFS 文件。

# libecfs - 用于解析 ECFS 文件的库

ECFS 文件格式非常容易使用传统的 ELF 工具进行解析，比如`readelf`，但是为了构建自定义的解析工具，我强烈建议您使用 libecfs 库。这个库是专门设计用于轻松解析 ECFS 核心文件的。稍后在本章中，我们将演示更多细节，当我们设计高级恶意软件分析工具来检测被感染的进程时。

libecfs 也用于正在开发的`readecfs`实用程序，这是一个用于解析 ECFS 文件的工具，非常类似于众所周知的`readelf`实用程序。请注意，libecfs 包含在 GitHub 存储库上的 ECFS 软件包中。

# readecfs

在本章的其余部分中，将使用`readecfs`实用程序来演示不同的 ECFS 功能。以下是从`readecfs -h`中的工具的概要：

```
Usage: readecfs [-RAPSslphega] <ecfscore>
-a  print all (equiv to -Sslphega)
-s  print symbol table info
-l  print shared library names
-p  print ELF program headers
-S  print ELF section headers
-h  print ELF header
-g  print PLTGOT info
-A  print Auxiliary vector
-P  print personality info
-e  print ecfs specific (auiliary vector, process state, sockets, pipes, fd's, etc.)

-[View raw data from a section]
-R <ecfscore> <section>

-[Copy an ELF section into a file (Similar to objcopy)]
-O <ecfscore> .section <outfile>

-[Extract and decompress /proc/$pid from .procfs.tgz section into directory]
-X <ecfscore> <output_dir>

Examples:
readecfs -e <ecfscore>
readecfs -Ag <ecfscore>
readecfs -R <ecfscore> .stack
readecfs -R <ecfscore> .bss
readecfs -eR <ecfscore> .heap
readecfs -O <ecfscore> .vdso vdso_elf.so
readecfs -X <ecfscore> procfs_dir
```

# 使用 ECFS 检查被感染的进程

在展示 ECFS 在真实案例中的有效性之前，了解一下我们将从黑客的角度使用的感染方法的背景将会很有帮助。对于黑客来说，能够将反取证技术纳入其在受损系统上的工作流程中是非常有用的，这样他们的程序，尤其是那些充当后门等的程序，可以对未经训练的人保持隐藏。

其中一种技术是执行**伪装**进程。这是在现有进程内运行程序的行为，理想情况下是在已知是良性但持久的进程内运行，例如 ftpd 或 sshd。Saruman 反取证执行([`www.bitlackeys.org/#saruman`](http://www.bitlackeys.org/#saruman))允许攻击者将一个完整的、动态链接的 PIE 可执行文件注入到现有进程的地址空间并运行它。

它使用线程注入技术，以便注入的程序可以与主机程序同时运行。这种特定的黑客技术是我在 2013 年想出并设计的，但我毫不怀疑其他类似的工具在地下场景中存在的时间比这长得多。通常，这种类型的反取证技术会不被注意到，并且很难被检测到。

让我们看看通过使用 ECFS 技术分析这样的进程可以实现什么样的效率和准确性。

## 感染主机进程

主机进程是一个良性进程，通常会是像 sshd 或 ftpd 这样的东西，就像之前提到的那样。为了举例，我们将使用一个简单而持久的名为 host 的程序；它只是在屏幕上打印一条消息并在无限循环中运行。然后，我们将使用 Saruman 反取证执行启动程序将远程服务器后门注入到该进程中。

在终端 1 中，运行主机程序：

```
$ ./host
I am the host
I am the host
I am the host
```

在终端 2 中，将后门注入到进程中：

```
$ ./launcher `pidof host` ./server
[+] Thread injection succeeded, tid: 16187
[+] Saruman successfully injected program: ./server
[+] PT_DETACHED -> 16186
$
```

## 捕获和分析 ECFS 快照

现在，如果我们通过使用`ecfs_snapshot`实用程序捕获进程的快照，或者通过向进程发出核心转储信号，我们就可以开始我们的检查了。

### 符号表分析

让我们来看一下`host.16186`快照的符号表分析：

```
 readelf -s host.16186

Symbol table '.dynsym' contains 6 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 00007fba3811e000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00007fba3818de30     0 FUNC    GLOBAL DEFAULT  UND puts
     2: 00007fba38209860     0 FUNC    GLOBAL DEFAULT  UND write
     3: 00007fba3813fdd0     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 00007fba3818c4e0     0 FUNC    GLOBAL DEFAULT  UND fopen

Symbol table '.symtab' contains 6 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000400470    96 FUNC    GLOBAL DEFAULT   10 sub_400470
     1: 00000000004004d0    42 FUNC    GLOBAL DEFAULT   10 sub_4004d0
     2: 00000000004005bd    50 FUNC    GLOBAL DEFAULT   10 sub_4005bd
     3: 00000000004005ef    69 FUNC    GLOBAL DEFAULT   10 sub_4005ef
     4: 0000000000400640   101 FUNC    GLOBAL DEFAULT   10 sub_400640
     5: 00000000004006b0     2 FUNC    GLOBAL DEFAULT   10 sub_4006b0
```

`readelf`命令允许我们查看符号表。请注意，`.dynsym`中存在动态符号的符号表，以及存储在`.symtab`符号表中的本地函数的符号表。ECFS 能够通过访问动态段并找到`DT_SYMTAB`来重建动态符号表。

### 注意

`.symtab`符号表有点棘手，但非常有价值。ECFS 使用一种特殊的方法来解析包含以 dwarf 格式的帧描述条目的`PT_GNU_EH_FRAME`段；这些用于异常处理。这些信息对于收集二进制文件中定义的每个函数的位置和大小非常有用。

在函数被混淆的情况下，诸如 IDA 之类的工具将无法识别二进制或核心文件中定义的每个函数，但 ECFS 技术将成功。这是 ECFS 对逆向工程世界产生的主要影响之一——一种几乎无懈可击的定位和确定每个函数大小并生成符号表的方法。在`host.16186`文件中，符号表被完全重建。这很有用，因为它可以帮助我们检测是否有任何 PLT/GOT 钩子被用来重定向共享库函数，如果是的话，我们可以识别被劫持的函数的实际名称。

### 段头分析

现在，让我们来看一下`host.16186`快照的段头分析。

我的`readelf`版本已经稍作修改，以便它识别以下自定义类型：`SHT_INJECTED`和`SHT_PRELOADED`。如果不对 readelf 进行这种修改，它将只显示与这些定义相关的数值。如果你愿意，可以查看`include/ecfs.h`中的定义，并将它们添加到`readelf`源代码中：

```
$ readelf -S host.16186
There are 46 section headers, starting at offset 0x255464:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00002238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note             NOTE             0000000000000000  000005f0
       000000000000133c  0000000000000000   A       0     0     4
  [ 3] .hash             GNU_HASH         0000000000400298  00002298
       000000000000001c  0000000000000000   A       0     0     4
  [ 4] .dynsym           DYNSYM           00000000004002b8  000022b8
       0000000000000090  0000000000000018   A       5     0     8
  [ 5] .dynstr           STRTAB           0000000000400348  00002348
       0000000000000049  0000000000000018   A       0     0     1
  [ 6] .rela.dyn         RELA             00000000004003c0  000023c0
       0000000000000018  0000000000000018   A       4     0     8
  [ 7] .rela.plt         RELA             00000000004003d8  000023d8
       0000000000000078  0000000000000018   A       4     0     8
  [ 8] .init             PROGBITS         0000000000400450  00002450
       000000000000001a  0000000000000000  AX       0     0     8
  [ 9] .plt              PROGBITS         0000000000400470  00002470
       0000000000000060  0000000000000010  AX       0     0     16
  [10] ._TEXT            PROGBITS         0000000000400000  00002000
       0000000000001000  0000000000000000  AX       0     0     16
  [11] .text             PROGBITS         00000000004004d0  000024d0
       00000000000001e2  0000000000000000           0     0     16
  [12] .fini             PROGBITS         00000000004006b4  000026b4
       0000000000000009  0000000000000000  AX       0     0     16
  [13] .eh_frame_hdr     PROGBITS         00000000004006e8  000026e8
       000000000000003c  0000000000000000  AX       0     0     4
  [14] .eh_frame         PROGBITS         0000000000400724  00002728
       0000000000000114  0000000000000000  AX       0     0     8
  [15] .ctors            PROGBITS         0000000000600e10  00003e10
       0000000000000008  0000000000000008   A       0     0     8
  [16] .dtors            PROGBITS         0000000000600e18  00003e18
       0000000000000008  0000000000000008   A       0     0     8
  [17] .dynamic          DYNAMIC          0000000000600e28  00003e28
       00000000000001d0  0000000000000010  WA       0     0     8
  [18] .got.plt          PROGBITS         0000000000601000  00004000
       0000000000000048  0000000000000008  WA       0     0     8
  [19] ._DATA            PROGBITS         0000000000600000  00003000
       0000000000001000  0000000000000000  WA       0     0     8
  [20] .data             PROGBITS         0000000000601040  00004040
       0000000000000010  0000000000000000  WA       0     0     8
  [21] .bss              PROGBITS         0000000000601050  00004050
       0000000000000008  0000000000000000  WA       0     0     8
  [22] .heap             PROGBITS         0000000000e9c000  00006000
       0000000000021000  0000000000000000  WA       0     0     8
  [23] .elf.dyn.0        INJECTED         00007fba37f1b000  00038000
       0000000000001000  0000000000000000  AX       0     0     8
  [24] libc-2.19.so.text SHLIB            00007fba3811e000  0003b000
       00000000001bb000  0000000000000000   A       0     0     8
  [25] libc-2.19.so.unde SHLIB            00007fba382d9000  001f6000
       00000000001ff000  0000000000000000   A       0     0     8
  [26] libc-2.19.so.relr SHLIB            00007fba384d8000  001f6000
       0000000000004000  0000000000000000   A       0     0     8
  [27] libc-2.19.so.data SHLIB            00007fba384dc000  001fa000
       0000000000002000  0000000000000000   A       0     0     8
  [28] ld-2.19.so.text   SHLIB            00007fba384e3000  00201000
       0000000000023000  0000000000000000   A       0     0     8
  [29] ld-2.19.so.relro  SHLIB            00007fba38705000  0022a000
       0000000000001000  0000000000000000   A       0     0     8
  [30] ld-2.19.so.data   SHLIB            00007fba38706000  0022b000
       0000000000001000  0000000000000000   A       0     0     8
  [31] .procfs.tgz       LOUSER+0         0000000000000000  00254388
       00000000000010dc  0000000000000001           0     0     8
  [32] .prstatus         PROGBITS         0000000000000000  00253000
       00000000000002a0  0000000000000150           0     0     8
  [33] .fdinfo           PROGBITS         0000000000000000  002532a0
       0000000000000ac8  0000000000000228           0     0     4
  [34] .siginfo          PROGBITS         0000000000000000  00253d68
       0000000000000080  0000000000000080           0     0     4
  [35] .auxvector        PROGBITS         0000000000000000  00253de8
       0000000000000130  0000000000000008           0     0     8
  [36] .exepath          PROGBITS         0000000000000000  00253f18
       000000000000001c  0000000000000008           0     0     1
  [37] .personality      PROGBITS         0000000000000000  00253f34
       0000000000000004  0000000000000004           0     0     1
  [38] .arglist          PROGBITS         0000000000000000  00253f38
       0000000000000050  0000000000000001           0     0     1
  [39] .fpregset         PROGBITS         0000000000000000  00253f88
       0000000000000400  0000000000000200           0     0     8
  [40] .stack            PROGBITS         00007fff4447c000  0022d000
       0000000000021000  0000000000000000  WA       0     0     8
  [41] .vdso             PROGBITS         00007fff444a9000  0024f000
       0000000000002000  0000000000000000  WA       0     0     8
  [42] .vsyscall         PROGBITS         ffffffffff600000  00251000
       0000000000001000  0000000000000000  WA       0     0     8
  [43] .symtab           SYMTAB           0000000000000000  0025619d
       0000000000000090  0000000000000018          44     0     4
  [44] .strtab           STRTAB           0000000000000000  0025622d
       0000000000000042  0000000000000000           0     0     1
  [45] .shstrtab         STRTAB           0000000000000000  00255fe4
       00000000000001b9  0000000000000000           0     0     1
```

第二十三部分对我们来说特别重要；它被标记为一个带有注入标记的可疑 ELF 对象：

```
  [23] .elf.dyn.0        INJECTED         00007fba37f1b000  00038000
       0000000000001000  0000000000000000  AX       0     0     8 
```

当 ECFS 启发式检测到一个 ELF 对象可疑，并且在其映射的共享库列表中找不到该特定对象时，它会以以下格式命名该段：

```
.elf.<type>.<count>
```

类型可以是四种之一：

+   `ET_NONE`

+   `ET_EXEC`

+   `ET_DYN`

+   `ET_REL`

在我们的例子中，它显然是`ET_DYN`，表示为`dyn`。计数只是找到的注入对象的索引。在这种情况下，索引是`0`，因为它是在这个特定进程中找到的第一个并且唯一的注入 ELF 对象。

`INJECTED`类型显然表示该部分包含一个被确定为可疑或通过非自然手段注入的 ELF 对象。在这种特殊情况下，进程被 Saruman（前面描述过）感染，它注入了一个**位置无关可执行文件**（**PIE**）。PIE 可执行文件的类型是`ET_DYN`，类似于共享库，这就是为什么 ECFS 将其标记为这种类型。

## 使用 readecfs 提取寄生代码

我们在 ECFS 核心文件中发现了一个与寄生代码相关的部分，这是一个注入的 PIE 可执行文件。下一步是调查代码本身。可以通过以下方式之一来完成：使用`objdump`实用程序或更高级的反汇编器，如 IDA pro，来导航到名为`.elf.dyn.0`的部分，或者首先使用`readecfs`实用程序从 ECFS 核心文件中提取寄生代码：

```
$ readecfs -O host.16186 .elf.dyn.0 parasite_code.exe

- readecfs output for file host.16186
- Executable path (.exepath): /home/ryan/git/saruman/host
- Command line: ./host                                                                          

[+] Copying section data from '.elf.dyn.0' into output file 'parasite_code.exe'
```

现在，我们有了从进程映像中提取的寄生代码的唯一副本，这要归功于 ECFS。要识别这种特定的恶意软件，然后提取它，如果没有 ECFS，这将是一项极其繁琐的任务。现在我们可以将`parasite_code.exe`作为一个单独的文件进行检查，在 IDA 中打开它等等：

```
root@elfmaster:~/ecfs/cores# readelf -l parasite_code.exe
readelf: Error: Unable to read in 0x40 bytes of section headers
readelf: Error: Unable to read in 0x780 bytes of section headers

Elf file type is DYN (Shared object file)
Entry point 0xdb0
There are 9 program headers, starting at offset 64

Program Headers:
 Type        Offset             VirtAddr           PhysAddr
              FileSiz            MemSiz              Flags  Align
 PHDR         0x0000000000000040 0x0000000000000040 0x0000000000000040
              0x00000000000001f8 0x00000000000001f8  R E    8
 INTERP       0x0000000000000238 0x0000000000000238 0x0000000000000238
              0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
 LOAD         0x0000000000000000 0x0000000000000000 0x0000000000000000
              0x0000000000001934 0x0000000000001934  R E    200000
 LOAD         0x0000000000001df0 0x0000000000201df0 0x0000000000201df0
              0x0000000000000328 0x0000000000000330  RW     200000
 DYNAMIC      0x0000000000001e08 0x0000000000201e08 0x0000000000201e08
              0x00000000000001d0 0x00000000000001d0  RW     8
 NOTE         0x0000000000000254 0x0000000000000254 0x0000000000000254
              0x0000000000000044 0x0000000000000044  R      4
 GNU_EH_FRAME 0x00000000000017e0 0x00000000000017e0 0x00000000000017e0
              0x000000000000003c 0x000000000000003c  R      4
  GNU_STACK   0x0000000000000000 0x0000000000000000 0x0000000000000000
              0x0000000000000000 0x0000000000000000  RW     10
  GNU_RELRO   0x0000000000001df0 0x0000000000201df0 0x0000000000201df0
              0x0000000000000210 0x0000000000000210  R      1
readelf: Error: Unable to read in 0x1d0 bytes of dynamic section
```

请注意，`readelf`在前面的输出中抱怨。这是因为我们提取的寄生体没有自己的段头表。将来，`readecfs`实用程序将能够为从整体 ECFS 核心文件中提取的映射 ELF 对象重建一个最小的段头表。

## 分析 Azazel 用户态 rootkit

如第七章中所述，*进程内存取证*，Azazel 用户态 rootkit 是一种通过`LD_PRELOAD`感染进程的用户态 rootkit，其中 Azazel 共享库链接到进程，并劫持各种`libc`函数。在第七章中，*进程内存取证*，我们使用 GDB 和`readelf`来检查这种特定的 rootkit 感染进程。现在让我们尝试使用 ECFS 方法来进行这种类型的进程内省。以下是从已感染 Azazel rootkit 的可执行文件 host2 中的一个进程的 ECFS 快照。

### 重建 host2 进程的符号表

现在，这是 host2 的符号表在进程重建时：

```
$ readelf -s host2.7254

Symbol table '.dynsym' contains 7 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00007f0a0d0ed070     0 FUNC    GLOBAL DEFAULT  UND unlink
     2: 00007f0a0d06fe30     0 FUNC    GLOBAL DEFAULT  UND puts
     3: 00007f0a0d0bcef0     0 FUNC    GLOBAL DEFAULT  UND opendir
     4: 00007f0a0d021dd0     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main
     5: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND fopen

 Symbol table '.symtab' contains 5 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 00000000004004b0   112 FUNC    GLOBAL DEFAULT   10 sub_4004b0
     1: 0000000000400520    42 FUNC    GLOBAL DEFAULT   10 sub_400520
     2: 000000000040060d    68 FUNC    GLOBAL DEFAULT   10 sub_40060d
     3: 0000000000400660   101 FUNC    GLOBAL DEFAULT   10 sub_400660
     4: 00000000004006d0     2 FUNC    GLOBAL DEFAULT   10 sub_4006d0
```

从前面的符号表中我们可以看出，host2 是一个简单的程序，只有少量的共享库调用（这在`.dynsym`符号表中显示）：`unlink`，`puts`，`opendir`和`fopen`。

### 重建 host2 进程的段头表

让我们看看 host2 的段头表在进程重建时是什么样子的：

```
$ readelf -S host2.7254

There are 65 section headers, starting at offset 0x27e1ee:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00002238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note             NOTE             0000000000000000  00000900
       000000000000105c  0000000000000000   A       0     0     4
  [ 3] .hash             GNU_HASH         0000000000400298  00002298
       000000000000001c  0000000000000000   A       0     0     4
  [ 4] .dynsym           DYNSYM           00000000004002b8  000022b8
       00000000000000a8  0000000000000018   A       5     0     8
  [ 5] .dynstr           STRTAB           0000000000400360  00002360
       0000000000000052  0000000000000018   A       0     0     1
  [ 6] .rela.dyn         RELA             00000000004003e0  000023e0
       0000000000000018  0000000000000018   A       4     0     8
  [ 7] .rela.plt         RELA             00000000004003f8  000023f8
       0000000000000090  0000000000000018   A       4     0     8
  [ 8] .init             PROGBITS         0000000000400488  00002488
       000000000000001a  0000000000000000  AX       0     0     8
  [ 9] .plt              PROGBITS         00000000004004b0  000024b0
       0000000000000070  0000000000000010  AX       0     0     16
  [10] ._TEXT            PROGBITS         0000000000400000  00002000
       0000000000001000  0000000000000000  AX       0     0     16
  [11] .text             PROGBITS         0000000000400520  00002520
       00000000000001b2  0000000000000000           0     0     16
  [12] .fini             PROGBITS         00000000004006d4  000026d4
       0000000000000009  0000000000000000  AX       0     0     16
  [13] .eh_frame_hdr     PROGBITS         0000000000400708  00002708
       0000000000000034  0000000000000000  AX       0     0     4
  [14] .eh_frame         PROGBITS         000000000040073c  00002740
       00000000000000f4  0000000000000000  AX       0     0     8
  [15] .ctors            PROGBITS         0000000000600e10  00003e10
       0000000000000008  0000000000000008   A       0     0     8
  [16] .dtors            PROGBITS         0000000000600e18  00003e18
       0000000000000008  0000000000000008   A       0     0     8
  [17] .dynamic          DYNAMIC          0000000000600e28  00003e28
       00000000000001d0  0000000000000010  WA       0     0     8
  [18] .got.plt          PROGBITS         0000000000601000  00004000
       0000000000000050  0000000000000008  WA       0     0     8
  [19] ._DATA            PROGBITS         0000000000600000  00003000
       0000000000001000  0000000000000000  WA       0     0     8
  [20] .data             PROGBITS         0000000000601048  00004048
       0000000000000010  0000000000000000  WA       0     0     8
  [21] .bss              PROGBITS         0000000000601058  00004058
       0000000000000008  0000000000000000  WA       0     0     8
  [22] .heap             PROGBITS         0000000000602000  00005000
       0000000000021000  0000000000000000  WA       0     0     8
  [23] libaudit.so.1.0.0 SHLIB            0000003001000000  00026000
       0000000000019000  0000000000000000   A       0     0     8
  [24] libaudit.so.1.0.0 SHLIB            0000003001019000  0003f000
       00000000001ff000  0000000000000000   A       0     0     8
  [25] libaudit.so.1.0.0 SHLIB            0000003001218000  0003f000
       0000000000001000  0000000000000000   A       0     0     8
  [26] libaudit.so.1.0.0 SHLIB            0000003001219000  00040000
       0000000000001000  0000000000000000   A       0     0     8
  [27] libpam.so.0.83.1\. SHLIB            0000003003400000  00041000
       000000000000d000  0000000000000000   A       0     0     8
  [28] libpam.so.0.83.1\. SHLIB            000000300340d000  0004e000
       00000000001ff000  0000000000000000   A       0     0     8
  [29] libpam.so.0.83.1\. SHLIB            000000300360c000  0004e000
       0000000000001000  0000000000000000   A       0     0     8
  [30] libpam.so.0.83.1\. SHLIB            000000300360d000  0004f000
       0000000000001000  0000000000000000   A       0     0     8
  [31] libutil-2.19.so.t SHLIB            00007f0a0cbf9000  00050000
       0000000000002000  0000000000000000   A       0     0     8
  [32] libutil-2.19.so.u SHLIB            00007f0a0cbfb000  00052000
       00000000001ff000  0000000000000000   A       0     0     8
  [33] libutil-2.19.so.r SHLIB            00007f0a0cdfa000  00052000
       0000000000001000  0000000000000000   A       0     0     8
  [34] libutil-2.19.so.d SHLIB            00007f0a0cdfb000  00053000
       0000000000001000  0000000000000000   A       0     0     8
  [35] libdl-2.19.so.tex SHLIB            00007f0a0cdfc000  00054000
       0000000000003000  0000000000000000   A       0     0     8
  [36] libdl-2.19.so.und SHLIB            00007f0a0cdff000  00057000
       00000000001ff000  0000000000000000   A       0     0     8
  [37] libdl-2.19.so.rel SHLIB            00007f0a0cffe000  00057000
       0000000000001000  0000000000000000   A       0     0     8
  [38] libdl-2.19.so.dat SHLIB            00007f0a0cfff000  00058000
       0000000000001000  0000000000000000   A       0     0     8
  [39] libc-2.19.so.text SHLIB            00007f0a0d000000  00059000
       00000000001bb000  0000000000000000   A       0     0     8
  [40] libc-2.19.so.unde SHLIB            00007f0a0d1bb000  00214000
       00000000001ff000  0000000000000000   A       0     0     8
  [41] libc-2.19.so.relr SHLIB            00007f0a0d3ba000  00214000
       0000000000004000  0000000000000000   A       0     0     8
  [42] libc-2.19.so.data SHLIB            00007f0a0d3be000  00218000
       0000000000002000  0000000000000000   A       0     0     8
  [43] azazel.so.text    PRELOADED        00007f0a0d3c5000  0021f000
       0000000000008000  0000000000000000   A       0     0     8
  [44] azazel.so.undef   PRELOADED        00007f0a0d3cd000  00227000
       00000000001ff000  0000000000000000   A       0     0     8
  [45] azazel.so.relro   PRELOADED        00007f0a0d5cc000  00227000
       0000000000001000  0000000000000000   A       0     0     8
  [46] azazel.so.data    PRELOADED        00007f0a0d5cd000  00228000
       0000000000001000  0000000000000000   A       0     0     8
  [47] ld-2.19.so.text   SHLIB            00007f0a0d5ce000  00229000
       0000000000023000  0000000000000000   A       0     0     8
  [48] ld-2.19.so.relro  SHLIB            00007f0a0d7f0000  00254000
       0000000000001000  0000000000000000   A       0     0     8
  [49] ld-2.19.so.data   SHLIB            00007f0a0d7f1000  00255000
       0000000000001000  0000000000000000   A       0     0     8
  [50] .procfs.tgz       LOUSER+0         0000000000000000  0027d038
       00000000000011b6  0000000000000001           0     0     8
  [51] .prstatus         PROGBITS         0000000000000000  0027c000
       0000000000000150  0000000000000150           0     0     8
  [52] .fdinfo           PROGBITS         0000000000000000  0027c150
       0000000000000ac8  0000000000000228           0     0     4
  [53] .siginfo          PROGBITS         0000000000000000  0027cc18
       0000000000000080  0000000000000080           0     0     4
  [54] .auxvector        PROGBITS         0000000000000000  0027cc98
       0000000000000130  0000000000000008           0     0     8
  [55] .exepath          PROGBITS         0000000000000000  0027cdc8
       000000000000001c  0000000000000008           0     0     1
  [56] .personality      PROGBITS         0000000000000000  0027cde4
       0000000000000004  0000000000000004           0     0     1
  [57] .arglist          PROGBITS         0000000000000000  0027cde8
       0000000000000050  0000000000000001           0     0     1
  [58] .fpregset         PROGBITS         0000000000000000  0027ce38
       0000000000000200  0000000000000200           0     0     8
  [59] .stack            PROGBITS         00007ffdb9161000  00257000
       0000000000021000  0000000000000000  WA       0     0     8
  [60] .vdso             PROGBITS         00007ffdb918f000  00279000
       0000000000002000  0000000000000000  WA       0     0     8
  [61] .vsyscall         PROGBITS         ffffffffff600000  0027b000
       0000000000001000  0000000000000000  WA       0     0     8
  [62] .symtab           SYMTAB           0000000000000000  0027f576
       0000000000000078  0000000000000018          63     0     4
  [63] .strtab           STRTAB           0000000000000000  0027f5ee
       0000000000000037  0000000000000000           0     0     1
  [64] .shstrtab         STRTAB           0000000000000000  0027f22e
       0000000000000348  0000000000000000           0     0     1
```

ELF 的 43 到 46 节都立即引起怀疑，因为它们标记为`PRELOADED`节类型，这表明它们是从使用`LD_PRELOAD`环境变量预加载的共享库的映射：

```
  [43] azazel.so.text    PRELOADED        00007f0a0d3c5000  0021f000
       0000000000008000  0000000000000000   A       0     0     8
  [44] azazel.so.undef   PRELOADED        00007f0a0d3cd000  00227000
       00000000001ff000  0000000000000000   A       0     0     8
  [45] azazel.so.relro   PRELOADED        00007f0a0d5cc000  00227000
       0000000000001000  0000000000000000   A       0     0     8
  [46] azazel.so.data    PRELOADED        00007f0a0d5cd000  00228000
       0000000000001000  0000000000000000   A       0     0     8
```

各种用户态 rootkit，如 Azazel，使用`LD_PRELOAD`作为它们的注入手段。下一步是查看 PLT/GOT（全局偏移表），并检查它是否包含指向各自边界之外的函数的指针。

你可能还记得前面的章节中提到 GOT 包含一个指针值表，应该指向这两者之一：

+   对应的 PLT 条目中的 PLT 存根（记住第二章中的延迟链接概念，*ELF 二进制格式*）

+   如果链接器已经以某种方式（延迟或严格链接）解析了特定的 GOT 条目，那么它将指向可执行文件的`.rela.plt`节中相应重定位条目所表示的共享库函数

### 使用 ECFS 验证 PLT/GOT

手动理解和系统验证 PLT/GOT 的完整性是很繁琐的。幸运的是，使用 ECFS 可以很容易地完成这项工作。如果你喜欢编写自己的工具，那么你应该使用专门为此目的设计的`libecfs`函数：

```
ssize_t get_pltgot_info(ecfs_elf_t *desc, pltgot_info_t **pginfo)
```

该函数分配了一个结构数组，每个元素都与单个 PLT/GOT 条目相关。

名为`pltgot_info_t`的 C 结构具有以下格式：

```
typedef struct pltgotinfo {
   unsigned long got_site; // addr of the GOT entry itself
   unsigned long got_entry_va; // pointer value stored in the GOT entry
   unsigned long plt_entry_va; // the expected PLT address
   unsigned long shl_entry_va; // the expected shared lib function addr
} pltgot_info_t;
```

可以在`ecfs/libecfs/main/detect_plt_hooks.c`中找到使用此函数的示例。这是一个简单的演示工具，用于检测共享库注入和 PLT/GOT 钩子，稍后在本章中进行了展示和注释，以便清晰地理解。`readecfs`实用程序还演示了在传递`-g`标志时使用`get_pltgot_info()`函数。 

### 用于 PLT/GOT 验证的 readecfs 输出

```
- readecfs output for file host2.7254
- Executable path (.exepath): /home/user/git/azazel/host2
- Command line: ./host2
- Printing out GOT/PLT characteristics (pltgot_info_t):
gotsite    gotvalue       gotshlib          pltval         symbol
0x601018   0x7f0a0d3c8c81  0x7f0a0d0ed070   0x4004c6      unlink
0x601020   0x7f0a0d06fe30  0x7f0a0d06fe30   0x4004d6      puts
0x601028   0x7f0a0d3c8d77  0x7f0a0d0bcef0   0x4004e6      opendir
0x601030   0x7f0a0d021dd0  0x7f0a0d021dd0   0x4004f6      __libc_start_main
```

前面的输出很容易解析。`gotvalue`应该有一个地址，与`gotshlib`或`pltval`匹配。然而，我们可以看到，第一个条目，即符号`unlink`，其地址为`0x7f0a0d3c8c81`。这与预期的共享库函数或 PLT 值不匹配。

进一步调查将显示该地址指向`azazel.so`中的一个函数。从前面的输出中，我们可以看到，唯一没有被篡改的两个函数是`puts`和`__libc_start_main`。为了更深入地了解检测过程，让我们看一下一个工具的源代码，该工具作为其检测功能的一部分自动进行 PLT/GOT 验证。这个工具叫做`detect_plt_hooks`，是用 C 编写的。它利用 libecfs API 来加载和解析 ECFS 快照。

请注意，以下代码大约有 50 行源代码，这相当了不起。如果我们不使用 ECFS 或 libecfs，要准确分析共享库注入和 PLT/GOT 钩子的进程映像，大约需要 3000 行 C 代码。我知道这一点，因为我已经做过了，而使用 libecfs 是迄今为止最轻松的方法。

这里有一个使用`detect_plt_hooks.c`的代码示例：

```
#include "../include/libecfs.h"

int main(int argc, char **argv)
{
    ecfs_elf_t *desc;
    ecfs_sym_t *dsyms;
    char *progname;
    int i;
    char *libname;
    long evil_addr = 0;

    if (argc < 2) {
        printf("Usage: %s <ecfs_file>\n", argv[0]);
        exit(0);
    }

    /*
     * Load the ECFS file and creates descriptor
     */
    desc = load_ecfs_file(argv[1]);
    /*
     * Get the original program name
    */
    progname = get_exe_path(desc);

    printf("Performing analysis on '%s' which corresponds to executable: %s\n", argv[1], progname);

    /*
     * Look for any sections that are marked as INJECTED
     * or PRELOADED, indicating shared library injection
     * or ELF object injection.
     */
    for (i = 0; i < desc->ehdr->e_shnum; i++) {
        if (desc->shdr[i].sh_type == SHT_INJECTED) {
            libname = strdup(&desc->shstrtab[desc->shdr[i].sh_name]);
            printf("[!] Found malicously injected ET_DYN (Dynamic ELF): %s - base: %lx\n", libname, desc->shdr[i].sh_addr);
        } else
        if (desc->shdr[i].sh_type == SHT_PRELOADED) {
            libname = strdup(&desc->shstrtab[desc->shdr[i].sh_name]);
            printf("[!] Found a preloaded shared library (LD_PRELOAD): %s - base: %lx\n", libname, desc->shdr[i].sh_addr);
        }
    }
    /*
     * Load and validate the PLT/GOT to make sure that each
     * GOT entry points to its proper respective location
     * in either the PLT, or the correct shared lib function.
     */
    pltgot_info_t *pltgot;
    int gotcount = get_pltgot_info(desc, &pltgot);
    for (i = 0; i < gotcount; i++) {
        if (pltgot[i].got_entry_va != pltgot[i].shl_entry_va &&
            pltgot[i].got_entry_va != pltgot[i].plt_entry_va &&
            pltgot[i].shl_entry_va != 0) {
            printf("[!] Found PLT/GOT hook: A function is pointing at %lx instead of %lx\n",
                pltgot[i].got_entry_va, evil_addr = pltgot[i].shl_entry_va);
     /*
      * Load the dynamic symbol table to print the
      * hijacked function by name.
      */
            int symcount = get_dynamic_symbols(desc, &dsyms);
            for (i = 0; i < symcount; i++) {
                if (dsyms[i].symval == evil_addr) {
                    printf("[!] %lx corresponds to hijacked function: %s\n", dsyms[i].symval, &dsyms[i].strtab[dsyms[i].nameoffset]);
                break;
                }
            }
        }
    }
    return 0;
}
```

# ECFS 参考指南

ECFS 文件格式既简单又复杂！总的来说，ELF 文件格式本身就很复杂，ECFS 从结构上继承了这些复杂性。另一方面，如果你知道它具有哪些特定特性以及要寻找什么，ECFS 可以帮助你轻松地浏览进程映像。

在前面的章节中，我们给出了一些利用 ECFS 的实际例子，展示了它的许多主要特性。然而，重要的是要有一个简单直接的参考，了解这些特性是什么，比如存在哪些自定义节以及它们的确切含义。在本节中，我们将为 ECFS 快照文件提供一个参考。

## ECFS 符号表重建

ECFS 处理程序使用对 ELF 二进制格式甚至是 dwarf 调试格式的高级理解，特别是动态段和`GNU_EH_FRAME`段，来完全重建程序的符号表。即使原始二进制文件已经被剥离并且没有部分头，ECFS 处理程序也足够智能，可以重建符号表。

我个人从未遇到过符号表重建完全失败的情况。它通常会重建所有或大多数符号表条目。可以使用诸如`readelf`或`readecfs`之类的实用程序访问符号表。libecfs API 还具有几个功能：

```
int get_dynamic_symbols(ecfs_elf_t *desc, ecfs_sym_t **syms)
int get_local_symbols(ecfs_elf_t *desc, ecfs_sym_t **syms)
```

一个函数获取动态符号表，另一个获取本地符号表——分别是`.dynsym`和`.symtab`。

以下是使用`readelf`读取符号表：

```
$ readelf -s host.6758

Symbol table '.dynsym' contains 8 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 00007f3dfd48b000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00007f3dfd4f9730     0 FUNC    GLOBAL DEFAULT  UND fputs
     2: 00007f3dfd4acdd0     0 FUNC    GLOBAL DEFAULT  UND __libc_start_main
     3: 00007f3dfd4f9220     0 FUNC    GLOBAL DEFAULT  UND fgets
     4: 0000000000000000     0 NOTYPE  WEAK   DEFAULT  UND __gmon_start__
     5: 00007f3dfd4f94e0     0 FUNC    GLOBAL DEFAULT  UND fopen
     6: 00007f3dfd54bd00     0 FUNC    GLOBAL DEFAULT  UND sleep
     7: 00007f3dfd84a870     8 OBJECT  GLOBAL DEFAULT   25 stdout

Symbol table '.symtab' contains 5 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 00000000004004f0   112 FUNC    GLOBAL DEFAULT   10 sub_4004f0
     1: 0000000000400560    42 FUNC    GLOBAL DEFAULT   10 sub_400560
     2: 000000000040064d   138 FUNC    GLOBAL DEFAULT   10 sub_40064d
     3: 00000000004006e0   101 FUNC    GLOBAL DEFAULT   10 sub_4006e0
     4: 0000000000400750     2 FUNC    GLOBAL DEFAULT   10 sub_400750
```

## ECFS 部分头

ECFS 处理程序重建了程序可能具有的大部分原始部分头。它还添加了一些非常有用的新部分和部分类型，对于取证分析非常有用。部分头由名称和类型标识，并包含数据或代码。

解析部分头非常容易，因此它们对于创建进程内存映像的地图非常有用。通过部分头导航整个进程布局比仅具有程序头（例如常规核心文件）要容易得多，后者甚至没有字符串名称。程序头描述内存段，而部分头为给定段的每个部分提供上下文。部分头有助于为逆向工程师提供更高的分辨率。

| 部分头 | 描述 |
| --- | --- |
| `._TEXT` | 这指向文本段（而不是`.text`部分）。这使得在不必解析程序头的情况下定位文本段成为可能。 |
| `._DATA` | 这指向数据段（而不是`.data`部分）。这使得在不必解析程序头的情况下定位数据段成为可能。 |
| `.stack` | 这指向了几个可能的堆栈段之一，取决于线程的数量。如果没有名为`.stack`的部分，要知道进程的实际堆栈在哪里将会更加困难。您将不得不查看`%rsp`寄存器的值，然后查看哪些程序头段包含与堆栈指针值匹配的地址范围。 |
| `.heap` | 类似于`.stack`部分，这指向堆段，也使得识别堆变得更加容易，特别是在 ASLR 将堆移动到随机位置的系统上。在旧系统上，它总是从数据段扩展的。 |
| `.bss` | 此部分并非 ECFS 的新内容。之所以在这里提到它，是因为对于可执行文件或共享库，`.bss`部分不包含任何内容，因为未初始化的数据在磁盘上不占用空间。然而，ECFS 表示内存，因此`.bss`部分实际上直到运行时才会被创建。ECFS 文件具有一个实际反映进程使用的未初始化数据变量的`.bss`部分。 |
| `.vdso` | 这指向映射到每个 Linux 进程中的[vdso]段，其中包含对于某些`glibc`系统调用包装器调用真实系统调用所必需的代码。 |
| `.vsyscall` | 类似于`.vdso`代码，`.vsyscall`页面包含用于调用少量虚拟系统调用的代码。它已经保留了向后兼容性。在逆向工程中了解此位置可能会很有用。 |
| `.procfs.tgz` | 此部分包含由 ECFS 处理程序捕获的进程`/proc/$pid`的整个目录结构和文件。如果您是一位狂热的取证分析师或程序员，那么您可能已经知道`proc`文件系统中包含的信息有多么有用。对于单个进程，在`/proc/$pid`中有超过 300 个文件。 |

| `.prstatus` | 此部分包含一系列`elf_prstatus`结构的数组。这些结构中存储了有关进程状态和寄存器状态的非常重要的信息：

```
struct elf_prstatus
  {
    struct elf_siginfo pr_info;         /* Info associated with signal.  */
    short int pr_cursig;                /* Current signal.  */
    unsigned long int pr_sigpend;       /* Set of pending signals.  */
    unsigned long int pr_sighold;       /* Set of held signals.  */
    __pid_t pr_pid;
    __pid_t pr_ppid;
    __pid_t pr_pgrp;
    __pid_t pr_sid;
    struct timeval pr_utime;            /* User time.  */
    struct timeval pr_stime;            /* System time.  */
    struct timeval pr_cutime;           /* Cumulative user time.  */
    struct timeval pr_cstime;           /* Cumulative system time.  */
    elf_gregset_t pr_reg;               /* GP registers.  */
    int pr_fpvalid;                     /* True if math copro being used.  */
  };
```

|

| `.fdinfo` | 此部分包含描述进程打开文件、网络连接和进程间通信所使用的文件描述符、套接字和管道的 ECFS 自定义数据。头文件`ecfs.h`定义了`fdinfo_t`类型：

```
typedef struct fdinfo {
        int fd;
        char path[MAX_PATH];
        loff_t pos;
        unsigned int perms;
        struct {
                struct in_addr src_addr;
                struct in_addr dst_addr;
                uint16_t src_port;
                uint16_t dst_port;
        } socket;
        char net;
} fd_info_t;
```

`readecfs`实用程序可以解析并漂亮地显示文件描述符信息，如查看 sshd 的 ECFS 快照时所示：

```
        [fd: 0:0] perms: 8002 path: /dev/null
        [fd: 1:0] perms: 8002 path: /dev/null
        [fd: 2:0] perms: 8002 path: /dev/null
        [fd: 3:0] perms: 802 path: socket:[10161]
        PROTOCOL: TCP
        SRC: 0.0.0.0:22
        DST: 0.0.0.0:0

        [fd: 4:0] perms: 802 path: socket:[10163]
        PROTOCOL: TCP
        SRC: 0.0.0.0:22
        DST: 0.0.0.0:0
```

|

| `.siginfo` | 此部分包含特定信号的信息，例如杀死进程的信号，或者在快照被拍摄之前的最后一个信号代码。`siginfo_t struct`存储在此部分。此结构的格式可以在`/usr/include/bits/siginfo.h`中看到。 |
| --- | --- |
| `.auxvector` | 这包含来自堆栈底部（最高内存地址）的实际辅助向量。辅助向量由内核在运行时设置，它包含传递给动态链接器的运行时信息。这些信息对于高级取证分析人员可能在多种情况下都很有价值。 |
| `.exepath` | 这保存了为该进程调用的原始可执行路径的字符串，即`/usr/sbin/sshd`。 |

| `.personality` | 这包含个性信息，即 ECFS 个性信息。可以使用 8 字节的无符号整数设置任意数量的个性标志：

```
#define ELF_STATIC (1 << 1) // if it's statically linked (instead of dynamically)
#define ELF_PIE (1 << 2)    // if it's a PIE executable
#define ELF_LOCSYM (1 << 3) // was a .symtab symbol table created by ecfs?
#define ELF_HEURISTICS (1 << 4) // were detection heuristics used by ecfs?
#define ELF_STRIPPED_SHDRS (1 << 8) // did the binary have section headers?
```

|

| `.arglist` | 包含存储为数组的原始`'char **argv'`。 |
| --- | --- |

## 将 ECFS 文件用作常规核心文件

ECFS 核心文件格式基本上与常规 Linux 核心文件向后兼容，因此可以像传统方式一样与 GDB 一起用作调试核心文件。

ECFS 文件的 ELF 文件头将其`e_type`（ELF 类型）设置为`ET_NONE`，而不是`ET_CORE`。这是因为核心文件不应该有节头，但 ECFS 文件确实有节头，为了确保它们被诸如`objdump`、`objcopy`等特定实用程序所承认，我们必须将它们标记为非 CORE 文件。在 ECFS 文件中切换 ELF 类型的最快方法是使用随 ECFS 软件套件一起提供的`et_flip`实用程序。

以下是使用 GDB 与 ECFS 核心文件的示例：

```
$ gdb -q /usr/sbin/sshd sshd.1195
Reading symbols from /usr/sbin/sshd...(no debugging symbols found)...done.
"/opt/ecfs/cores/sshd.1195" is not a core dump: File format not recognized
(gdb) quit
```

接下来，以下是将 ELF 文件类型更改为`ET_CORE`并重试的示例：

```
$ et_flip sshd.1195
$ gdb -q /usr/sbin/sshd sshd.1195
Reading symbols from /usr/sbin/sshd...(no debugging symbols found)...done.
[New LWP 1195]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Core was generated by `/usr/sbin/sshd -D'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x00007ff4066b8d83 in __select_nocancel () at ../sysdeps/unix/syscall-template.S:81
81  ../sysdeps/unix/syscall-template.S: No such file or directory.
(gdb)
```

## libecfs API 及其使用方法

libecfs API 是将 ECFS 支持集成到 Linux 恶意软件分析和逆向工程工具中的关键组件。这个库的文档内容太多，无法放入本书的一个章节中。我建议您使用与项目本身一起不断增长的手册：

[`github.com/elfmaster/ecfs/blob/master/Documentation/libecfs_manual.txt`](https://github.com/elfmaster/ecfs/blob/master/Documentation/libecfs_manual.txt)

# 使用 ECFS 进行进程复活

您是否曾经想过能够在 Linux 中暂停和恢复进程？设计 ECFS 后，很快就显而易见，它们包含了足够的关于进程及其状态的信息，可以将它们重新加载到内存中，以便它们可以从上次停止的地方开始执行。这个功能有许多可能的用途，并需要更多的研究和开发。

目前，ECFS 快照执行的实现是基本的，只能处理简单的进程。在撰写本章时，它可以恢复文件流，但不能处理套接字或管道，并且只能处理单线程进程。执行 ECFS 快照的软件可以在 GitHub 上找到：[`github.com/elfmaster/ecfs_exec`](https://github.com/elfmaster/ecfs_exec)。

以下是快照执行的示例：

```
$ ./print_passfile
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin

– interrupted by snapshot -
```

我们现在有了 ECFS 快照文件 print_passfile.6627（其中 6627 是进程 ID）。我们将使用 ecfs_exec 来执行这个快照，它应该会从离开的地方开始执行：

```
$ ecfs_exec ./print_passfile.6627
[+] Using entry point: 7f79a0473f20
[+] Using stack vaddr: 7fff8c752738
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
syslog:x:101:104::/home/syslog:/bin/false
messagebus:x:102:106::/var/run/dbus:/bin/false
usbmux:x:103:46:usbmux daemon,,,:/home/usbmux:/bin/false
dnsmasq:x:104:65534:dnsmasq,,,:/var/lib/misc:/bin/false
avahi-autoipd:x:105:113:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/bin/false
kernoops:x:106:65534:Kernel Oops Tracking Daemon,,,:/:/bin/false
saned:x:108:115::/home/saned:/bin/false
whoopsie:x:109:116::/nonexistent:/bin/false
speech-dispatcher:x:110:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/sh
avahi:x:111:117:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/bin/false
lightdm:x:112:118:Light Display Manager:/var/lib/lightdm:/bin/false
colord:x:113:121:colord colour management daemon,,,:/var/lib/colord:/bin/false
hplip:x:114:7:HPLIP system user,,,:/var/run/hplip:/bin/false
pulse:x:115:122:PulseAudio daemon,,,:/var/run/pulse:/bin/false
statd:x:116:65534::/var/lib/nfs:/bin/false
guest-ieu5xg:x:117:126:Guest,,,:/tmp/guest-ieu5xg:/bin/bash
sshd:x:118:65534::/var/run/sshd:/usr/sbin/nologin
gdm:x:119:128:Gnome Display Manager:/var/lib/gdm:/bin/false
```

这是一个关于`ecfs_exec`如何工作的非常简单的演示。它使用了来自`.fdinfo`部分的文件描述符信息来获取文件描述符号、文件路径和文件偏移量。它还使用了`.prstatus`和`.fpregset`部分来获取寄存器状态，以便可以从离开的地方恢复执行。

# 了解更多关于 ECFS 的信息

扩展核心文件快照技术 ECFS 仍然相对较新。我在 defcon 23 上做了演讲（[`www.defcon.org/html/defcon-23/dc-23-speakers.html#O%27Neill`](https://www.defcon.org/html/defcon-23/dc-23-speakers.html#O%27Neill)），目前这个技术还在不断传播。希望会有一个社区的发展，更多人会开始采用 ECFS 进行日常取证工作和工具。尽管如此，目前已经存在一些关于 ECFS 的资源：

官方 GitHub 页面：[`github.com/elfmaster/ecfs`](https://github.com/elfmaster/ecfs)

+   原始白皮书（已过时）：[`www.leviathansecurity.com/white-papers/extending-the-elf-core-format-for-forensics-snapshots`](http://www.leviathansecurity.com/white-papers/extending-the-elf-core-format-for-forensics-snapshots)

+   POC || GTFO 0x7 的一篇文章：*核心文件的创新*，[`speakerdeck.com/ange/poc-gtfo-issue-0x07-1`](https://speakerdeck.com/ange/poc-gtfo-issue-0x07-1)

# 总结

在本章中，我们介绍了 ECFS 快照技术和快照格式的基础知识。我们使用了几个真实的取证案例来实验 ECFS，甚至编写了一个使用 libecfs C 库来检测共享库注入和 PLT/GOT 钩子的工具。在下一章中，我们将跳出用户空间，探索 Linux 内核、vmlinux 的布局以及内核 rootkit 和取证技术的组合。
