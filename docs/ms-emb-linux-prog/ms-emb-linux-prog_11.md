# 第十一章：管理内存

本章涵盖了与内存管理相关的问题，这对于任何 Linux 系统都是一个重要的话题，但对于嵌入式 Linux 来说尤其重要，因为系统内存通常是有限的。在简要回顾了虚拟内存之后，我将向您展示如何测量内存使用情况，如何检测内存分配的问题，包括内存泄漏，以及当内存用尽时会发生什么。您必须了解可用的工具，从简单的工具如`free`和`top`，到复杂的工具如 mtrace 和 Valgrind。

# 虚拟内存基础知识

总之，Linux 配置 CPU 的内存管理单元，向运行的程序呈现一个虚拟地址空间，从零开始，到 32 位处理器上的最高地址`0xffffffff`结束。该地址空间被分成 4 KiB 的页面（也有一些罕见的系统使用其他页面大小）。

Linux 将这个虚拟地址空间分为一个称为用户空间的应用程序区域和一个称为内核空间的内核区域。这两者之间的分割由一个名为`PAGE_OFFSET`的内核配置参数设置。在典型的 32 位嵌入式系统中，`PAGE_OFFSET`是`0xc0000000`，将低 3 GiB 分配给用户空间，将顶部 1 GiB 分配给内核空间。用户地址空间是针对每个进程分配的，因此每个进程都在一个沙盒中运行，与其他进程分离。内核地址空间对所有进程都是相同的：只有一个内核。

这个虚拟地址空间中的页面通过**内存管理单元**（**MMU**）映射到物理地址，后者使用页表执行映射。

每个虚拟内存页面可能是：

+   未映射，访问将导致`SIGSEGV`

+   映射到进程私有的物理内存页面

+   映射到与其他进程共享的物理内存页面

+   映射并与设置了`写时复制`标志的共享：写入被内核捕获，内核复制页面并将其映射到原始页面的进程中，然后允许写入发生

+   映射到内核使用的物理内存页面

内核可能还会将页面映射到保留的内存区域，例如，以访问设备驱动程序中的寄存器和缓冲内存

一个明显的问题是，为什么我们要这样做，而不是像典型的 RTOS 那样直接引用物理内存？

虚拟内存有许多优点，其中一些在这里描述：

+   无效的内存访问被捕获，并通过`SIGSEGV`通知应用程序

+   进程在自己的内存空间中运行，与其他进程隔离

+   通过共享公共代码和数据来有效利用内存，例如在库中

+   通过添加交换文件来增加物理内存的表面数量的可能性，尽管在嵌入式目标上进行交换是罕见的

这些都是有力的论据，但我们必须承认也存在一些缺点。很难确定应用程序的实际内存预算，这是本章的主要关注点之一。默认的分配策略是过度承诺，这会导致棘手的内存不足情况，我稍后也会讨论。最后，内存管理代码在处理异常（页面错误）时引入的延迟使系统变得不太确定，这对实时程序很重要。我将在第十四章 *实时编程*中介绍这一点。

内核空间和用户空间的内存管理是不同的。以下部分描述了基本的区别和你需要了解的事情。

# 内核空间内存布局

内核内存的管理方式相当直接。它不是按需分页的，这意味着对于每个使用`kmalloc()`或类似函数进行的分配，都有真正的物理内存。内核内存从不被丢弃或分页出去。

一些体系结构在内核日志消息中显示了启动时内存映射的摘要。这个跟踪来自一个 32 位 ARM 设备（BeagleBone Black）：

```
Memory: 511MB = 511MB total
Memory: 505980k/505980k available, 18308k reserved, 0K highmem
Virtual kernel memory layout:
  vector  : 0xffff0000 - 0xffff1000   (   4 kB)
  fixmap  : 0xfff00000 - 0xfffe0000   ( 896 kB)
  vmalloc : 0xe0800000 - 0xff000000   ( 488 MB)
  lowmem  : 0xc0000000 - 0xe0000000   ( 512 MB)
  pkmap   : 0xbfe00000 - 0xc0000000   (   2 MB)
  modules : 0xbf800000 - 0xbfe00000   (   6 MB)
    .text : 0xc0008000 - 0xc0763c90   (7536 kB)
    .init : 0xc0764000 - 0xc079f700   ( 238 kB)
    .data : 0xc07a0000 - 0xc0827240   ( 541 kB)
     .bss : 0xc0827240 - 0xc089e940   ( 478 kB)
```

505980 KiB 可用的数字是内核在开始执行但在开始进行动态分配之前看到的空闲内存量。

内核空间内存的使用者包括以下内容：

+   内核本身，换句话说，从内核映像文件在启动时加载的代码和数据。这在前面的代码中显示在段`.text`、`.init`、`.data`和`.bss`中。一旦内核完成初始化，`.init`段就被释放。

+   通过 slab 分配器分配的内存，用于各种内核数据结构。这包括使用`kmalloc()`进行的分配。它们来自标记为`lowmem`的区域。

+   通过`vmalloc()`分配的内存，通常比通过`kmalloc()`可用的内存大。这些位于 vmalloc 区域。

+   用于设备驱动程序访问属于各种硬件部分的寄存器和内存的映射，可以通过阅读`/proc/iomem`来查看。这些来自 vmalloc 区域，但由于它们映射到主系统内存之外的物理内存，它们不占用任何真实的内存。

+   内核模块，加载到标记为模块的区域。

+   其他低级别的分配在其他地方没有被跟踪。

## 内核使用多少内存？

不幸的是，对于这个问题并没有一个完整的答案，但接下来的内容是我们能得到的最接近的。

首先，你可以在之前显示的内核日志中看到内核代码和数据占用的内存，或者你可以使用`size`命令，如下所示：

```
$ arm-poky-linux-gnueabi-size vmlinux
text      data     bss       dec       hex       filename
9013448   796868   8428144   18238460  1164bfc   vmlinux

```

通常，与总内存量相比，大小很小。如果不是这样，你需要查看内核配置，并删除那些你不需要的组件。目前正在努力允许构建小内核：搜索 Linux-tiny 或 Linux Kernel Tinification。后者有一个项目页面[`tiny.wiki.kernel.org/`](https://tiny.wiki.kernel.org/)。

你可以通过阅读`/proc/meminfo`来获取有关内存使用情况的更多信息：

```
# cat /proc/meminfo
MemTotal:         509016 kB
MemFree:          410680 kB
Buffers:            1720 kB
Cached:            25132 kB
SwapCached:            0 kB
Active:            74880 kB
Inactive:           3224 kB
Active(anon):      51344 kB
Inactive(anon):     1372 kB
Active(file):      23536 kB
Inactive(file):     1852 kB
Unevictable:           0 kB
Mlocked:               0 kB
HighTotal:             0 kB
HighFree:              0 kB
LowTotal:         509016 kB
LowFree:          410680 kB
SwapTotal:             0 kB
SwapFree:              0 kB
Dirty:                16 kB
Writeback:             0 kB
AnonPages:         51248 kB
Mapped:            24376 kB
Shmem:              1452 kB
Slab:              11292 kB
SReclaimable:       5164 kB
SUnreclaim:         6128 kB
KernelStack:        1832 kB
PageTables:         1540 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:      254508 kB
Committed_AS:     734936 kB
VmallocTotal:     499712 kB
VmallocUsed:       29576 kB
VmallocChunk:     389116 kB
```

这些字段的描述在`proc(5)`的 man 页面中。内核内存使用是以下内容的总和：

+   **Slab**：由 slab 分配器分配的总内存

+   **KernelStack**：执行内核代码时使用的堆栈空间

+   **PageTables**：用于存储页表的内存

+   **VmallocUsed**：由`vmalloc()`分配的内存

在 slab 分配的情况下，你可以通过阅读`/proc/slabinfo`来获取更多信息。类似地，在`/proc/vmallocinfo`中有 vmalloc 区域的分配细分。在这两种情况下，你需要对内核及其子系统有详细的了解，以确切地看到哪个子系统正在进行分配以及原因，这超出了本讨论的范围。

使用模块，你可以使用`lsmod`来查找代码和数据占用的内存空间：

```
# lsmod
Module            Size  Used by
g_multi          47670  2
libcomposite     14299  1 g_multi
mt7601Usta       601404  0
```

这留下了低级别的分配，没有记录，这阻止我们生成一个准确的内核空间内存使用情况。当我们把我们知道的所有内核和用户空间分配加起来时，这将出现为缺失的内存。

# 用户空间内存布局

Linux 采用懒惰的分配策略，只有在程序访问时才映射物理内存页面。例如，使用`malloc(3)`分配 1 MiB 的缓冲区返回一个内存地址块的指针，但没有实际的物理内存。在页表条目中设置一个标志，以便内核捕获任何读取或写入访问。这就是所谓的页错误。只有在这一点上，内核才尝试找到一个物理内存页，并将其添加到进程的页表映射中。值得用一个简单的程序来演示这一点：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/resource.h>
#define BUFFER_SIZE (1024 * 1024)

void print_pgfaults(void)
{
  int ret;
  struct rusage usage;
  ret = getrusage(RUSAGE_SELF, &usage);
  if (ret == -1) {
    perror("getrusage");
  } else {
    printf ("Major page faults %ld\n", usage.ru_majflt);
    printf ("Minor page faults %ld\n", usage.ru_minflt);
  }
}

int main (int argc, char *argv[])
{
  unsigned char *p;
  printf("Initial state\n");
  print_pgfaults();
  p = malloc(BUFFER_SIZE);
  printf("After malloc\n");
  print_pgfaults();
  memset(p, 0x42, BUFFER_SIZE);
  printf("After memset\n");
  print_pgfaults();
  memset(p, 0x42, BUFFER_SIZE);
  printf("After 2nd memset\n");
  print_pgfaults();
  return 0;
}
```

当你运行它时，你会看到这样的东西：

```
Initial state
Major page faults 0
Minor page faults 172
After malloc
Major page faults 0
Minor page faults 186
After memset
Major page faults 0
Minor page faults 442
After 2nd memset
Major page faults 0
Minor page faults 442
```

在初始化程序环境时遇到了 172 个次要页面错误，并在调用`getrusage(2)`时遇到了 14 个次要页面错误（这些数字将根据您使用的体系结构和 C 库的版本而变化）。重要的部分是填充内存时的增加：442-186 = 256。缓冲区为 1 MiB，即 256 页。第二次调用`memset(3)`没有任何区别，因为现在所有页面都已映射。

正如您所看到的，当内核捕获到对未映射的页面的访问时，将生成页面错误。实际上，有两种页面错误：次要和主要。次要错误时，内核只需找到一个物理内存页面并将其映射到进程地址空间，如前面的代码所示。主要页面错误发生在虚拟内存映射到文件时，例如使用`mmap(2)`，我将很快描述。从该内存中读取意味着内核不仅需要找到一个内存页面并将其映射进来，还需要从文件中填充数据。因此，主要错误在时间和系统资源方面要昂贵得多。

# 进程内存映射

您可以通过`proc`文件系统查看进程的内存映射。例如，这是`init`进程的 PID 1 的映射：

```
# cat /proc/1/maps
00008000-0000e000 r-xp 00000000 00:0b 23281745   /sbin/init
00016000-00017000 rwxp 00006000 00:0b 23281745   /sbin/init
00017000-00038000 rwxp 00000000 00:00 0          [heap]
b6ded000-b6f1d000 r-xp 00000000 00:0b 23281695   /lib/libc-2.19.so
b6f1d000-b6f24000 ---p 00130000 00:0b 23281695   /lib/libc-2.19.so
b6f24000-b6f26000 r-xp 0012f000 00:0b 23281695   /lib/libc-2.19.so
b6f26000-b6f27000 rwxp 00131000 00:0b 23281695   /lib/libc-2.19.so
b6f27000-b6f2a000 rwxp 00000000 00:00 0
b6f2a000-b6f49000 r-xp 00000000 00:0b 23281359   /lib/ld-2.19.so
b6f4c000-b6f4e000 rwxp 00000000 00:00 0
b6f4f000-b6f50000 r-xp 00000000 00:00 0          [sigpage]
b6f50000-b6f51000 r-xp 0001e000 00:0b 23281359   /lib/ld-2.19.so
b6f51000-b6f52000 rwxp 0001f000 00:0b 23281359   /lib/ld-2.19.so
beea1000-beec2000 rw-p 00000000 00:00 0          [stack]
ffff0000-ffff1000 r-xp 00000000 00:00 0          [vectors]
```

前三列显示每个映射的开始和结束虚拟地址以及权限。权限在这里显示：

+   `r` = 读

+   `w` = 写

+   `x` = 执行

+   `s` = 共享

+   `p` = 私有（写时复制）

如果映射与文件相关联，则文件名将出现在最后一列，第四、五和六列包含从文件开始的偏移量，块设备号和文件的 inode。大多数映射都是到程序本身和它链接的库。程序可以分配内存的两个区域，标记为`[heap]`和`[stack]`。使用`malloc(3)`分配的内存来自前者（除了非常大的分配，我们稍后会讨论）；堆栈上的分配来自后者。两个区域的最大大小由进程的`ulimit`控制：

+   **堆**：`ulimit -d`，默认无限制

+   **堆栈**：`ulimit -s`，默认 8 MiB

超出限制的分配将被`SIGSEGV`拒绝。

当内存不足时，内核可能决定丢弃映射到文件且只读的页面。如果再次访问该页面，将导致主要页面错误，并从文件中重新读取。

# 交换

交换的想法是保留一些存储空间，内核可以将未映射到文件的内存页面放置在其中，以便它可以释放内存供其他用途使用。它通过交换文件的大小增加了物理内存的有效大小。这并非是万能药：将页面复制到交换文件和从交换文件复制页面都会产生成本，这在承载工作负载的真实内存太少的系统上变得明显，并开始*磁盘抖动*。

在嵌入式设备上很少使用交换，因为它与闪存存储不兼容，常量写入会迅速磨损。但是，您可能希望考虑交换到压缩的 RAM（zram）。

## 交换到压缩内存（zram）

zram 驱动程序创建名为`/dev/zram0`、`/dev/zram1`等的基于 RAM 的块设备。写入这些设备的页面在存储之前会被压缩。通过 30%至 50%的压缩比，您可以预期整体空闲内存增加约 10%，但会增加更多的处理和相应的功耗。它在一些低内存的 Android 设备上使用。

要启用 zram，请使用以下选项配置内核：

```
CONFIG_SWAP
CONFIG_CGROUP_MEM_RES_CTLR
CONFIG_CGROUP_MEM_RES_CTLR_SWAP
CONFIG_ZRAM
```

然后，通过将以下内容添加到`/etc/fstab`来在启动时挂载 zram：

```
/dev/zram0 none swap defaults zramsize=<size in bytes>,swapprio=<swap partition priority>
```

您可以使用以下命令打开和关闭交换：

```
# swapon /dev/zram0
# swapoff /dev/zram0
```

# 使用 mmap 映射内存

进程开始时，一定数量的内存映射到程序文件的文本（代码）和数据段，以及它链接的共享库。它可以在运行时使用`malloc(3)`在堆上分配内存，并通过局部作用域变量和通过`alloca(3)`分配的内存在堆栈上分配内存。它还可以在运行时动态加载库使用`dlopen(3)`。所有这些映射都由内核处理。但是，进程还可以使用`mmap(2)`以显式方式操纵其内存映射：

```
void *mmap(void *addr, size_t length, int prot, int flags,
  int fd, off_t offset);
```

它从具有描述符`fd`的文件中的`offset`开始映射`length`字节的内存，并在成功时返回映射的指针。由于底层硬件以页面为单位工作，`length`被舍入到最接近的整页数。保护参数`prot`是读、写和执行权限的组合，`flags`参数至少包含`MAP_SHARED`或`MAP_PRIVATE`。还有许多其他标志，这些标志在 man 页面中有描述。

mmap 有许多用途。以下是其中一些。

## 使用 mmap 分配私有内存

您可以使用 mmap 通过设置`MAP_ANONYMOUS`标志和`fd`文件描述符为`-1`来分配一个私有内存区域。这类似于使用`malloc(3)`从堆中分配内存，只是内存是按页对齐的，并且是页的倍数。内存分配在与库相同的区域。事实上，出于这个原因，一些人称该区域为 mmap 区域。

匿名映射更适合大型分配，因为它们不会用内存块固定堆，这会增加碎片化的可能性。有趣的是，您会发现`malloc(3)`（至少在 glibc 中）停止为超过 128 KiB 的请求从堆中分配内存，并以这种方式使用 mmap，因此在大多数情况下，只使用 malloc 是正确的做法。系统将选择满足请求的最佳方式。

## 使用 mmap 共享内存

正如我们在第十章中看到的，*了解进程和线程*，POSIX 共享内存需要使用 mmap 来访问内存段。在这种情况下，您设置`MAP_SHARED`标志，并使用`shm_open()`的文件描述符：

```
int shm_fd;
char *shm_p;

shm_fd = shm_open("/myshm", O_CREAT | O_RDWR, 0666);
ftruncate(shm_fd, 65536);
shm_p = mmap(NULL, 65536, PROT_READ | PROT_WRITE,
  MAP_SHARED, shm_fd, 0);
```

## 使用 mmap 访问设备内存

正如我在第八章中提到的，*介绍设备驱动程序*，驱动程序可以允许其设备节点被 mmap，并与应用程序共享一些设备内存。确切的实现取决于驱动程序。

一个例子是 Linux 帧缓冲区，`/dev/fb0`。该接口在`/usr/include/linux/fb.h`中定义，包括一个`ioctl`函数来获取显示的大小和每像素位数。然后，您可以使用 mmap 来请求视频驱动程序与应用程序共享帧缓冲区并读写像素：

```
int f;
int fb_size;
unsigned char *fb_mem;

f = open("/dev/fb0", O_RDWR);
/* Use ioctl FBIOGET_VSCREENINFO to find the display dimensions
  and calculate fb_size */
fb_mem = mmap(0, fb_size, PROT_READ | PROT_WRITE, MAP_SHARED, f, 0);
/* read and write pixels through pointer fb_mem */
```

第二个例子是流媒体视频接口，Video 4 Linux，版本 2，或者 V4L2，它在`/usr/include/linux/videodev2.h`中定义。每个视频设备都有一个名为`/dev/videoN`的节点，从`/dev/video0`开始。有一个`ioctl`函数来请求驱动程序分配一些视频缓冲区，你可以将其映射到用户空间。然后，只需要循环缓冲区并根据播放或捕获视频流的情况填充或清空它们。

# 我的应用程序使用了多少内存？

与内核空间一样，分配、映射和共享用户空间内存的不同方式使得回答这个看似简单的问题变得相当困难。

首先，您可以询问内核认为有多少可用内存，可以使用`free`命令来执行此操作。以下是输出的典型示例：

```
             total     used     free   shared  buffers   cached
Mem:        509016   504312     4704        0    26456   363860
-/+ buffers/cache:   113996   395020
Swap:            0        0        0
```

### 提示

乍一看，这看起来像是一个几乎没有内存的系统，只有 4704 KiB 的空闲内存，占用了 509,016 KiB 的不到 1%。然而，请注意，26,456 KiB 在缓冲区中，而 363,860 KiB 在缓存中。Linux 认为空闲内存是浪费的内存，因此内核使用空闲内存用于缓冲区和缓存，因为它们在需要时可以被收缩。从测量中去除缓冲区和缓存可以得到真正的空闲内存，即 395,020 KiB；占总量的 77%。在使用 free 时，标有`-/+ buffers/cache`的第二行上的数字是重要的。

您可以通过向`/proc/sys/vm/drop_caches`写入 1 到 3 之间的数字来强制内核释放缓存：

```
# echo 3 > /proc/sys/vm/drop_caches
```

实际上，该数字是一个位掩码，用于确定您要释放的两种广义缓存中的哪一种：1 表示页面缓存，2 表示 dentry 和 inode 缓存的组合。这些缓存的确切作用在这里并不特别重要，只是内核正在使用的内存可以在短时间内被回收。

# 每个进程的内存使用

有几种度量方法可以衡量进程使用的内存量。我将从最容易获得的两种开始——**虚拟集大小**（**vss**）和**驻留内存大小**（**rss**），这两种在大多数`ps`和`top`命令的实现中都可以获得：

+   **Vss**：在`ps`命令中称为 VSZ，在`top`中称为 VIRT，是进程映射的内存总量。它是`/proc/<PID>/map`中显示的所有区域的总和。这个数字的兴趣有限，因为只有部分虚拟内存在任何时候都被分配到物理内存。

+   **Rss**：在`ps`中称为 RSS，在`top`中称为 RES，是映射到物理内存页面的内存总和。这更接近进程的实际内存预算，但是有一个问题，如果将所有进程的 Rss 相加，您将高估内存的使用，因为一些页面将是共享的。

## 使用 top 和 ps

BusyBox 的`top`和`ps`版本提供的信息非常有限。以下示例使用了`procps`包中的完整版本。

`ps`命令显示了 Vss（VSZ）和 Rss（RSS）以及包括`vsz`和`rss`的自定义格式，如下所示：

```
# ps -eo pid,tid,class,rtprio,stat,vsz,rss,comm

  PID   TID CLS RTPRIO STAT    VSZ   RSS COMMAND
    1     1 TS       - Ss     4496  2652 systemd
  ...
  205   205 TS       - Ss     4076  1296 systemd-journal
  228   228 TS       - Ss     2524  1396 udevd
  581   581 TS       - Ss     2880  1508 avahi-daemon
  584   584 TS       - Ss     2848  1512 dbus-daemon
  590   590 TS       - Ss     1332   680 acpid
  594   594 TS       - Ss     4600  1564 wpa_supplicant
```

同样，`top`显示了每个进程的空闲内存和内存使用的摘要：

```
top - 21:17:52 up 10:04,  1 user,  load average: 0.00, 0.01, 0.05
Tasks:  96 total,   1 running,  95 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.7 us,  2.2 sy,  0.0 ni, 95.9 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
KiB Mem:    509016 total,   278524 used,   230492 free,    25572 buffers
KiB Swap:        0 total,        0 used,        0 free,   170920 cached

PID USER      PR  NI  VIRT  RES  SHR S  %CPU %MEM    TIME+  COMMAND
1098 debian    20   0 29076  16m 8312 S   0.0  3.2   0:01.29 wicd-client
  595 root      20   0 64920 9.8m 4048 S   0.0  2.0   0:01.09 node
  866 root      20   0 28892 9152 3660 S   0.2  1.8   0:36.38 Xorg
```

这些简单的命令让您感受到内存的使用情况，并在看到进程的 Rss 不断增加时第一次表明您有内存泄漏的迹象。然而，它们在绝对内存使用的测量上并不是非常准确。

## 使用 smem

2009 年，Matt Mackall 开始研究进程内存测量中共享页面的计算问题，并添加了两个名为**唯一集大小**或**Uss**和**比例集大小**或**Pss**的新指标：

+   **Uss**：这是分配给物理内存并且对进程唯一的内存量；它不与任何其他内存共享。这是如果进程终止将被释放的内存量。

+   **Pss**：这将共享页面的计算分配给所有映射了它们的进程。例如，如果一个库代码区域有 12 页长，并且被六个进程共享，每个进程将累积两页的 Pss。因此，如果将所有进程的 Pss 数字相加，就可以得到这些进程实际使用的内存量。换句话说，Pss 就是我们一直在寻找的数字。

这些信息可以在`/proc/<PID>/smaps`中找到，其中包含了`/proc/<PID>/maps`中显示的每个映射的附加信息。以下是这样一个文件中的一个部分，它提供了有关`libc`代码段的映射的信息：

```
b6e6d000-b6f45000 r-xp 00000000 b3:02 2444 /lib/libc-2.13.so
Size:                864 kB
Rss:                 264 kB
Pss:                   6 kB
Shared_Clean:        264 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:          264 kB
Anonymous:             0 kB
AnonHugePages:         0 kB
Swap:                  0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Locked:                0 kB
VmFlags: rd ex mr mw me
```

### 注意

请注意，Rss 为 264 KiB，但由于它在许多其他进程之间共享，因此 Pss 只有 6 KiB。

有一个名为 smem 的工具，它汇总了`smaps`文件中的信息，并以各种方式呈现，包括饼图或条形图。smem 的项目页面是[`www.selenic.com/smem`](https://www.selenic.com/smem)。它在大多数桌面发行版中都作为一个软件包提供。但是，由于它是用 Python 编写的，在嵌入式目标上安装它需要一个 Python 环境，这可能会给一个工具带来太多麻烦。为了解决这个问题，有一个名为`smemcap`的小程序，它可以在目标上捕获`/proc`的状态，并将其保存到一个 TAR 文件中，以便稍后在主机计算机上进行分析。它是 BusyBox 的一部分，但也可以从`smem`源代码编译而成。

以`root`身份本地运行`smem`，你会看到这些结果：

```
# smem -t
 PID User  Command                   Swap      USS     PSS     RSS
 610 0     /sbin/agetty -s ttyO0 11     0      128     149     720
1236 0     /sbin/agetty -s ttyGS0 1     0      128     149     720
 609 0     /sbin/agetty tty1 38400      0      144     163     724
 578 0     /usr/sbin/acpid              0      140     173     680
 819 0     /usr/sbin/cron               0      188     201     704
 634 103   avahi-daemon: chroot hel     0      112     205     500
 980 0     /usr/sbin/udhcpd -S /etc     0      196     205     568
  ...
 836 0     /usr/bin/X :0 -auth /var     0     7172    7746    9212
 583 0     /usr/bin/node autorun.js     0     8772    9043   10076
1089 1000  /usr/bin/python -O /usr/     0     9600   11264   16388
------------------------------------------------------------------
  53 6                                  0    65820   78251  146544
```

从输出的最后一行可以看出，在这种情况下，总的 Pss 大约是 Rss 的一半。

如果你没有或不想在目标上安装 Python，你可以再次以`root`身份使用`smemcap`来捕获状态：

```
# smemcap > smem-bbb-cap.tar
```

然后，将 TAR 文件复制到主机并使用`smem -S`读取，尽管这次不需要以`root`身份运行：

```
$ smem -t -S smem-bbb-cap.tar
```

输出与本地运行时的输出相同。

## 其他需要考虑的工具

另一种显示 Pss 的方法是通过`ps_mem`([`github.com/pixelb/ps_mem`](https://github.com/pixelb/ps_mem))，它以更简单的格式打印几乎相同的信息。它也是用 Python 编写的。

Android 也有一个名为`procrank`的工具，可以通过少量更改在嵌入式 Linux 上进行交叉编译。你可以从[`github.com/csimmonds/procrank_linux`](https://github.com/csimmonds/procrank_linux)获取代码。

# 识别内存泄漏

内存泄漏发生在分配内存后不释放它，当它不再需要时。内存泄漏并不是嵌入式系统特有的问题，但它成为一个问题的部分原因是目标本来就没有太多内存，另一部分原因是它们经常长时间运行而不重启，导致泄漏变成了一个大问题。

当你运行`free`或`top`并看到可用内存不断减少时，即使你清除缓存，你会意识到有一个泄漏，如前面的部分所示。你可以通过查看每个进程的 Uss 和 Rss 来确定罪魁祸首（或罪魁祸首）。

有几种工具可以识别程序中的内存泄漏。我将看两种：`mtrace`和`Valgrind`。

## mtrace

`mtrace`是 glibc 的一个组件，它跟踪对`malloc(3)`、`free(3)`和相关函数的调用，并在程序退出时识别未释放的内存区域。你需要在程序内部调用`mtrace()`函数开始跟踪，然后在运行时，将路径名写入`MALLOC_TRACE`环境变量，以便写入跟踪信息的文件。如果`MALLOC_TRACE`不存在或文件无法打开，`mtrace`钩子将不会安装。虽然跟踪信息是以 ASCII 形式写入的，但通常使用`mtrace`命令来查看它。

这是一个例子：

```
#include <mcheck.h>
#include <stdlib.h>
#include <stdio.h>

int main(int argc, char *argv[])
{
  int j;
  mtrace();
  for (j = 0; j < 2; j++)
    malloc(100);  /* Never freed:a memory leak */
  calloc(16, 16);  /* Never freed:a memory leak */
  exit(EXIT_SUCCESS);
}
```

当运行程序并查看跟踪时，你可能会看到以下内容：

```
$ export MALLOC_TRACE=mtrace.log
$ ./mtrace-example
$ mtrace mtrace-example mtrace.log

Memory not freed:
-----------------
           Address     Size     Caller
0x0000000001479460     0x64  at /home/chris/mtrace-example.c:11
0x00000000014794d0     0x64  at /home/chris/mtrace-example.c:11
0x0000000001479540    0x100  at /home/chris/mtrace-example.c:15
```

不幸的是，`mtrace`在程序运行时不能告诉你有关泄漏内存的信息。它必须先终止。

## Valgrind

Valgrind 是一个非常强大的工具，可以发现内存问题，包括泄漏和其他问题。一个优点是你不必重新编译要检查的程序和库，尽管如果它们已经使用`-g`选项编译，以便包含调试符号表，它的工作效果会更好。它通过在模拟环境中运行程序并在各个点捕获执行来工作。这导致 Valgrind 的一个很大的缺点，即程序以正常速度的一小部分运行，这使得它对测试任何具有实时约束的东西不太有用。

### 注意

顺便说一句，这个名字经常被错误发音：Valgrind 的 FAQ 中说*grind*的发音是短的*i*--就像*grinned*（押韵*tinned*）而不是*grined*（押韵*find*）。FAQ、文档和下载都可以在[`valgrind.org`](http://valgrind.org)找到。

Valgrind 包含几个诊断工具：

+   **memcheck**：这是默认工具，用于检测内存泄漏和内存的一般误用

+   **cachegrind**：这个工具计算处理器缓存命中率

+   **callgrind**：这个工具计算每个函数调用的成本

+   **helgrind**：这个工具用于突出显示 Pthread API 的误用、潜在死锁和竞争条件

+   **DRD**：这是另一个 Pthread 分析工具

+   **massif**：这个工具用于分析堆和栈的使用情况

您可以使用`-tool`选项选择您想要的工具。Valgrind 可以在主要的嵌入式平台上运行：ARM（Cortex A）、PPC、MIPS 和 32 位和 64 位的 x86。它在 Yocto Project 和 Buildroot 中都作为一个软件包提供。

要找到我们的内存泄漏，我们需要使用默认的`memcheck`工具，并使用选项`--leakcheck=full`来打印出发现泄漏的行：

```
$ valgrind --leak-check=full ./mtrace-example
==17235== Memcheck, a memory error detector
==17235== Copyright (C) 2002-2013, and GNU GPL'd, by Julian Seward et al.
==17235== Using Valgrind-3.10.0.SVN and LibVEX; rerun with -h for copyright info
==17235== Command: ./mtrace-example
==17235==
==17235==
==17235== HEAP SUMMARY:
==17235==  in use at exit: 456 bytes in 3 blocks
==17235==  total heap usage: 3 allocs, 0 frees, 456 bytes allocated
==17235==
==17235== 200 bytes in 2 blocks are definitely lost in loss record 1 of 2
==17235==    at 0x4C2AB80: malloc (in /usr/lib/valgrind/vgpreload_memcheck-linux.so)
==17235==    by 0x4005FA: main (mtrace-example.c:12)
==17235==
==17235== 256 bytes in 1 blocks are definitely lost in loss record 2 of 2
==17235==    at 0x4C2CC70: calloc (in /usr/lib/valgrind/vgpreload_memcheck-linux.so)
==17235==    by 0x400613: main (mtrace-example.c:14)
==17235==
==17235== LEAK SUMMARY:
==17235==    definitely lost: 456 bytes in 3 blocks
==17235==    indirectly lost: 0 bytes in 0 blocks
==17235==      possibly lost: 0 bytes in 0 blocks
==17235==    still reachable: 0 bytes in 0 blocks
==17235==         suppressed: 0 bytes in 0 blocks
==17235==
==17235== For counts of detected and suppressed errors, rerun with: -v
==17235== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```

# 内存不足

标准的内存分配策略是**过度承诺**，这意味着内核将允许应用程序分配比物理内存更多的内存。大多数情况下，这很好用，因为应用程序通常会请求比实际需要的更多的内存。它还有助于`fork(2)`的实现：可以安全地复制一个大型程序，因为内存页面是带有`copy-on-write`标志的共享的。在大多数情况下，`fork`后会调用`exec`函数，这样就会取消内存共享，然后加载一个新程序。

然而，总有可能某个特定的工作负载会导致一组进程同时尝试兑现它们被承诺的分配，因此需求超过了实际存在的内存。这是一种**内存不足**的情况，或者**OOM**。在这一点上，除了杀死进程直到问题消失之外别无选择。这是内存不足杀手的工作。

在我们讨论这些之前，有一个内核分配的调整参数在`/proc/sys/vm/overcommit_memory`中，你可以将其设置为：

+   `0`：启发式过度承诺（这是默认设置）

+   `1`：始终过度承诺，永不检查

+   `2`：始终检查，永不过度承诺

选项 1 只有在运行与大型稀疏数组一起工作并分配大量内存但只写入其中一小部分的程序时才真正有用。在嵌入式系统的环境中，这样的程序很少见。

选项 2，永不过度承诺，似乎是一个不错的选择，如果您担心内存不足，也许是在任务或安全关键的应用中。它将失败于大于承诺限制的分配，这个限制是交换空间的大小加上总内存乘以过度承诺比率。过度承诺比率由`/proc/sys/vm/overcommit_ratio`控制，默认值为 50%。

例如，假设您有一台设备，配备了 512MB 的系统 RAM，并设置了一个非常保守的比率为 25%：

```
# echo 25 > /proc/sys/vm/overcommit_ratio
# grep -e MemTotal -e CommitLimit /proc/meminfo
MemTotal:         509016 kB
CommitLimit:      127252 kB
```

没有交换空间，因此承诺限制是`MemTotal`的 25%，这是预期的。

`/proc/meminfo`中还有另一个重要的变量：`Committed_AS`。这是迄今为止需要满足所有分配的总内存量。我在一个系统上找到了以下内容：

```
# grep -e MemTotal -e Committed_AS /proc/meminfo
MemTotal:         509016 kB
Committed_AS:     741364 kB
```

换句话说，内核已经承诺了比可用内存更多的内存。因此，将`overcommit_memory`设置为`2`意味着所有分配都会失败，而不管`overcommit_ratio`如何。要使系统正常工作，我要么必须安装双倍的 RAM，要么严重减少正在运行的进程数量，大约有 40 个。

在所有情况下，最终的防御是 OOM killer。它使用一种启发式方法为每个进程计算 0 到 1000 之间的糟糕分数，然后终止具有最高分数的进程，直到有足够的空闲内存。您应该在内核日志中看到类似于这样的内容：

```
[44510.490320] eatmem invoked oom-killer: gfp_mask=0x200da, order=0, oom_score_adj=0
...
```

您可以使用`echo f > /proc/sysrq-trigger`来强制发生 OOM 事件。

您可以通过将调整值写入`/proc/<PID>/oom_score_adj`来影响进程的糟糕分数。值为`-1000`意味着糟糕分数永远不会大于零，因此永远不会被杀死；值为`+1000`意味着它将始终大于 1000，因此将始终被杀死。

# 进一步阅读

以下资源提供了有关本章介绍的主题的进一步信息：

+   *Linux 内核开发，第 3 版*，作者*Robert Love*，*Addison Wesley*，*O'Reilly Media*; (2010 年 6 月) ISBN-10: 0672329468

+   *Linux 系统编程，第 2 版*，作者*Robert Love*，*O'Reilly Media*; (2013 年 6 月 8 日) ISBN-10: 1449339530

+   *了解 Linux VM 管理器*，作者*Mel Gorman*：[`www.kernel.org/doc/gorman/pdf/understand.pdf`](https://www.kernel.org/doc/gorman/pdf/understand.pdf)

+   *Valgrind 3.3 - Gnu/Linux 应用程序的高级调试和性能分析*，作者*J Seward*，*N. Nethercote*和*J. Weidendorfer*，*Network Theory Ltd*; (2008 年 3 月 1 日) ISBN 978-0954612054

# 摘要

在虚拟内存系统中考虑每个内存使用的字节是不可能的。但是，您可以使用`free`命令找到一个相当准确的总空闲内存量，不包括缓冲区和缓存所占用的内存。通过在一段时间内监视它，并使用不同的工作负载，您应该对它将保持在给定限制内感到自信。

当您想要调整内存使用情况或识别意外分配的来源时，有一些资源可以提供更详细的信息。对于内核空间，最有用的信息在于`/proc: meminfo`，`slabinfo`和`vmallocinfo`。

在获取用户空间的准确测量方面，最佳指标是 Pss，如`smem`和其他工具所示。对于内存调试，您可以从诸如`mtrace`之类的简单跟踪器获得帮助，或者您可以选择使用 Valgrind memcheck 工具这样的重量级选项。

如果您担心内存不足的后果，您可以通过`/proc/sys/vm/overcommit_memory`微调分配机制，并且可以通过`oom_score_adj`参数控制特定进程被杀死的可能性。

下一章将全面介绍如何使用 GNU 调试器调试用户空间和内核代码，以及您可以从观察代码运行中获得的见解，包括我在这里描述的内存管理函数。
