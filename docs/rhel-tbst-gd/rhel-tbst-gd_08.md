# 第八章 硬件故障排除

在上一章中，我们确定了我们的 NFS 上的文件系统被挂载为**只读**。为了确定原因，我们围绕 NFS 和文件系统进行了大量的故障排除。我们使用了诸如`showmount`查看可用的 NFS 共享和`mount`命令显示已挂载的文件系统等命令。

一旦我们确定了问题，我们就能够使用`fsck`命令执行文件系统检查和恢复文件系统。

在本章中，我们将继续从第七章*文件系统错误和恢复*的路径，并调查硬件设备故障。本章将涵盖许多日志文件和工具，这些工具不仅可以确定硬件故障是否发生，还可以确定为什么发生。

# 从日志条目开始

在第七章*文件系统错误和恢复*中，当查看`/var/log/messages`日志文件以识别 NFS 服务器文件系统的问题时，我们注意到了以下消息：

```
Apr 26 10:25:44 nfs kernel: md/raid1:md127: Disk failure on sdb1, disabling device.
md/raid1:md127: Operation continuing on 1 devices.
Apr 26 10:25:55 nfs kernel: md: unbind<sdb1>
Apr 26 10:25:55 nfs kernel: md: export_rdev(sdb1)
Apr 26 10:27:20 nfs kernel: md: bind<sdb1>
Apr 26 10:27:20 nfs kernel: md: recovery of RAID array md127
Apr 26 10:27:20 nfs kernel: md: minimum _guaranteed_  speed: 1000 KB/sec/disk.
Apr 26 10:27:20 nfs kernel: md: using maximum available idle IO bandwidth (but not more than 200000 KB/sec) for recovery.
Apr 26 10:27:20 nfs kernel: md: using 128k window, over a total of 511936k.
Apr 26 10:27:20 nfs kernel: md: md127: recovery done.

```

前面的消息表明 RAID 设备`/dev/md127`发生了故障。由于上一章主要关注文件系统本身的问题，我们没有进一步调查 RAID 设备的故障。在本章中，我们将进行调查以确定原因和解决方法。

为了开始调查，我们应该首先查看原始日志消息，因为这些消息可以告诉我们关于 RAID 设备状态的很多信息。

首先，让我们将消息分解成以下几个小节：

```
Apr 26 10:25:44 nfs kernel: md/raid1:md127: Disk failure on sdb1, disabling device.
md/raid1:md127: Operation continuing on 1 devices.

```

第一条日志消息实际上非常有意义。显示的第一个关键信息是消息所涉及的 RAID 设备`(md/raid1:md127)`。

通过这个设备的名称，我们已经知道了很多。我们知道的第一件事是，这个 RAID 设备是由 Linux 的软件 RAID 系统**多设备驱动程序**（**md**）创建的。该系统允许 Linux 将两个独立的磁盘应用 RAID。

由于本章主要涉及 RAID，我们应该首先了解 RAID 是什么以及它是如何工作的。

# 什么是 RAID？

**独立磁盘冗余阵列**（**RAID**）通常是一个软件或硬件系统，允许用户将多个磁盘作为一个设备使用。RAID 可以以多种方式配置，从而实现更大的数据冗余或性能。

这种配置通常被称为 RAID 级别。不同类型的 RAID 级别提供不同的功能，以更好地了解 RAID 级别。让我们探索一些常用的 RAID 级别。

## RAID 0 – 分区

RAID 0 是最简单的 RAID 级别之一。RAID 0 的工作原理是将多个磁盘组合起来作为一个磁盘。当数据写入 RAID 设备时，数据被分割，部分数据被写入每个磁盘。为了更好地理解这一点，让我们举一个简单的例子。

+   如果我们有一个由五个 500GB 驱动器组成的简单 RAID 0 设备，我们的 RAID 设备将是所有五个驱动器的大小——2500GB 或 2.5TB。如果我们要向 RAID 设备写入一个 50MB 的文件，文件的 10MB 数据将同时写入每个磁盘。

这个过程通常被称为**分区**。在同样的情况下，当从 RAID 设备中读取那个 50MB 文件时，读取请求也将同时由每个磁盘处理。

将文件分割并同时处理每个磁盘的部分可以提供更好的写入或读取请求性能。事实上，因为我们有五个磁盘，请求速度会提高 5 倍。

一个简单的类比是，如果你有五个人以相同的速度建造一堵墙，他们将比一个人建造同样的墙快五倍。

虽然 RAID 0 提供了性能，但它并不提供任何数据保护。如果这种 RAID 中的单个驱动器失败，该驱动器的数据将不可用，这种故障可能导致完全的数据丢失。

## RAID 1 - 镜像

RAID 1 是另一种简单的 RAID 级别。与 RAID 0 不同，RAID 1 中的驱动器是镜像的。RAID 1 通常由两个或更多个驱动器组成。当数据被写入 RAID 设备时，数据会完整地写入每个设备。

这个过程被称为**镜像**，因为数据基本上在所有驱动器上都是镜像的：

+   使用与之前相同的场景，如果我们在 RAID 1 配置中有五个 500GB 磁盘驱动器，总磁盘大小将为 500GB。当我们将相同的 50MB 文件写入 RAID 设备时，每个驱动器将获得该 50MB 文件的副本。

+   这也意味着写请求的速度将只有 RAID 中最慢的驱动器那么快。对于 RAID 1，每个驱动器必须在被视为完成之前完成写请求。

+   然而，读请求可以由 RAID 1 中的任何一个驱动器提供。因此，RAID 1 有时可以更快地提供读请求，因为每个请求可以由 RAID 中的不同驱动器执行。

RAID 1 提供了最高级别的数据弹性，因为在故障期间只需要一个磁盘驱动器保持活动。使用我们的五盘场景，我们可以丢失五个磁盘中的四个并且仍然重建和使用 RAID。这就是为什么在数据保护比磁盘性能更重要时应该使用 RAID 1 的原因。

## RAID 5 - 条带化与分布式奇偶校验

**RAID 5**是一个难以理解的 RAID 级别的例子。RAID 5 通过将数据条带化到多个磁盘上来工作，就像 RAID 0 一样，但它还包括奇偶校验。奇偶校验数据是通过对写入 RAID 设备的数据执行异或运算而生成的特殊数据。生成的数据可以用于从另一个驱动器重建丢失的数据。

+   使用与之前相同的例子，我们在 RAID 5 配置中有五个 500GB 硬盘驱动器，如果我们再次写入一个 50MB 的文件，每个磁盘将接收 10MB 的数据；这与 RAID 0 完全相同。然而，与 RAID 0 不同的是，每个磁盘还会写入奇偶校验数据。由于额外的奇偶校验数据，RAID 可用的总数据大小是四个驱动器的总和，其中一个驱动器的数据分配给奇偶校验。在我们的情况下，这意味着可用的磁盘空间为 2TB，其中 500GB 用于奇偶校验。

通常有一个误解，即奇偶校验数据是写入专用驱动器的 RAID 5。事实并非如此。只是奇偶校验数据的大小是一个完整磁盘的空间。然而，这些数据是分布在所有磁盘上的。

使用 RAID 5 而不是 RAID 0 的原因是，如果单个驱动器失败，数据可以被重建。RAID 5 的唯一问题是，如果两个驱动器失败，RAID 无法重建，可能导致数据丢失。

## RAID 6 - 双分布式奇偶校验条带化

**RAID 6**本质上与 RAID 5 相同类型的 RAID；但是，奇偶校验数据是双倍的。通过加倍奇偶校验数据，RAID 可以在最多两个磁盘故障时存活。由于奇偶校验是双倍的，如果我们将五个 500GB 硬盘驱动器放入 RAID 6 配置中，可用的磁盘空间将是 1.5TB，即 3 个驱动器的总和；另外 1TB 的数据空间将被两组奇偶校验数据占用。

## RAID 10 - 镜像和条带化

**RAID 10**（通常称为 RAID 1 + 0）是另一种非常常见的 RAID 级别。RAID 10 本质上是 RAID 1 和 RAID 0 的组合。使用 RAID 10，每个磁盘都有一个镜像，并且数据被条带化到所有镜像驱动器上。为了解释这一点，我们将使用与上面类似的例子；但是，我们将使用六个 500GB 驱动器。

+   如果我们要写入一个 30MB 的文件，它将被分成 10MB 的块，并分别写入三个 RAID 设备。这些 RAID 设备是 RAID 1 的镜像。基本上，RAID 10 是许多 RAID 1 设备在 RAID 0 配置中一起条带化。

RAID 10 配置在性能和数据保护之间取得了良好的平衡。为了发生完全故障，镜像的两侧都必须失败；这意味着 RAID 1 的两侧都会失败。

考虑到 RAID 中的磁盘数量，这种情况发生的可能性比 RAID 5 的可能性要小。从性能的角度来看，RAID 10 仍然受益于条带化方法，并且能够将单个文件的不同块写入到每个磁盘，从而提高写入速度。

RAID 10 也受益于具有相同数据的两个磁盘；与 RAID 1 一样，当发出读取请求时，任何一个磁盘都可以处理该请求，从而允许每个磁盘独立处理并发的读取请求。

RAID 10 的缺点是，虽然它通常可以满足或超过 RAID 5 的性能，但通常需要更多的硬件来实现这一点，因为每个磁盘都是镜像的，你会失去一半的总磁盘空间。

以我们之前的例子，我们在 RAID 10 配置中使用六个 500GB 驱动器的可用空间将是 1.5TB。简单来说，它是我们磁盘容量的 50%。这个相同的容量在 RAID 5 中使用 4 个驱动器也是可用的。

# 回到排除故障我们的 RAID

现在我们对 RAID 和不同的配置有了更好的理解，让我们回到调查我们的错误。

```
Apr 26 10:25:44 nfs kernel: md/raid1:md127: Disk failure on sdb1, disabling device.
md/raid1:md127: Operation continuing on 1 devices.

```

从前面的错误中，我们可以看到我们的 RAID 设备是**md127**。我们还可以看到这个设备是一个 RAID 1 设备（`md/raid1`）。表明*操作在 1 个设备上继续*的消息意味着镜像的第二部分仍然可用。

好消息是，如果镜像的两侧都不可用，RAID 将完全失败并导致更严重的问题。

既然我们现在知道受影响的 RAID 设备、使用的 RAID 类型，甚至是失败的硬盘，我们对这次故障有了相当多的信息。如果我们继续查看`/var/log/messages`中的日志条目，我们甚至可以找到更多信息：

```
Apr 26 10:25:55 nfs kernel: md: unbind<sdb1>
Apr 26 10:25:55 nfs kernel: md: export_rdev(sdb1)
Apr 26 10:27:20 nfs kernel: md: bind<sdb1>
Apr 26 10:27:20 nfs kernel: md: recovery of RAID array md127
Apr 26 10:27:20 nfs kernel: md: minimum _guaranteed_  speed: 1000 KB/sec/disk.

```

前面的消息很有趣，因为它们表明 Linux 软件 RAID 服务 MD 尝试恢复 RAID：

```
Apr 26 10:25:55 nfs kernel: md: unbind<sdb1>

```

在日志的这一部分的第一行中，似乎设备`sdb1`已从 RAID 中移除：

```
Apr 26 10:27:20 nfs kernel: md: bind<sdb1>

```

然而，第三行表明设备`sdb1`已重新添加到 RAID 或“**绑定**”到 RAID。

第四和第五行显示 RAID 开始了恢复步骤：

```
Apr 26 10:27:20 nfs kernel: md: recovery of RAID array md127
Apr 26 10:27:20 nfs kernel: md: minimum _guaranteed_  speed: 1000 KB/sec/disk.

```

## RAID 恢复的工作原理

早些时候，我们讨论了各种 RAID 级别如何能够通过奇偶校验数据或镜像数据重建和恢复丢失的设备数据。

当 RAID 设备失去其中一个驱动器，并且该驱动器被替换或重新添加到 RAID 时，无论是软件还是硬件 RAID 管理器都将开始重建数据。重建的目标是重新创建应该在丢失的驱动器上的数据。

如果 RAID 是镜像 RAID，将从可用的镜像磁盘读取数据并写入替换的磁盘。

对于基于奇偶校验的 RAID，重建将基于 RAID 中已经条带化的存活数据和奇偶校验数据。

在奇偶校验 RAID 的重建过程中，任何额外的故障都可能导致重建失败。对于基于镜像的 RAID，只要有一份完整的数据副本用于重建，故障可以发生在任何磁盘上。

在我们捕获的日志消息的末尾，我们可以看到重建成功了：

```
Apr 26 10:27:20 nfs kernel: md: md127: recovery done.

```

根据前一章中日志消息的结尾，RAID`设备/dev/md127`是健康的。

## 检查当前的 RAID 状态

虽然`/var/log/messages`是查看服务器上发生了什么的好方法，但这并不一定意味着这些日志消息与 RAID 的当前状态准确无误。

为了查看 RAID 设备的当前状态，我们可以运行一些命令。

我们将使用的第一个命令是`mdadm`命令：

```
[nfs]# mdadm --detail /dev/md127
/dev/md127:
 Version : 1.0
 Creation Time : Wed Apr 15 09:39:22 2015
 Raid Level : raid1
 Array Size : 511936 (500.02 MiB 524.22 MB)
 Used Dev Size : 511936 (500.02 MiB 524.22 MB)
 Raid Devices : 2
 Total Devices : 1
 Persistence : Superblock is persistent

 Intent Bitmap : Internal

 Update Time : Sun May 10 06:16:10 2015
 State : clean, degraded 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
 Spare Devices : 0

 Name : localhost:boot
 UUID : 7adf0323:b0962394:387e6cd0:b2914469
 Events : 52

 Number   Major   Minor   RaidDevice State
 0       8        1        0      active sync   /dev/sda1
 2       0        0        2      removed

```

`mdadm`命令用于管理基于 Linux MD 的 RAID。在上述命令中，我们指定了标志`--detail`，后跟一个 RAID 设备。这告诉`mdadm`打印指定 RAID 设备的详细信息。

`mdadm`命令不仅可以打印状态，还可以用于执行 RAID 活动，如创建、销毁或修改 RAID 设备。

为了理解`--detail`标志的输出，让我们将上面的输出分解如下：

```
/dev/md127:
 Version : 1.0
 Creation Time : Wed Apr 15 09:39:22 2015
 Raid Level : raid1
 Array Size : 511936 (500.02 MiB 524.22 MB)
 Used Dev Size : 511936 (500.02 MiB 524.22 MB)
 Raid Devices : 2
 Total Devices : 1
 Persistence : Superblock is persistent

```

第一部分告诉我们关于 RAID 本身的很多信息。需要注意的重要项目是`Creation Time`，在这种情况下是`Wed April 15th`上午 9:39。这告诉我们 RAID 首次创建的时间。

`Raid Level`也被记录下来，就像我们在`/var/log/messages`中看到的那样是 RAID 1。我们还可以看到`Array Size`，告诉我们 RAID 设备将提供的总可用磁盘空间（524 MB）和在这个 RAID 数组中使用的`Raid Devices`的数量，这种情况下是两个设备。

组成这个 RAID 的设备数量很重要，因为它可以帮助我们了解这个 RAID 的状态。

由于我们的 RAID 由总共两个设备组成，如果任何一个设备失败，我们知道我们的 RAID 将面临完全失败的风险，如果剩下的磁盘丢失。然而，如果我们的 RAID 由三个设备组成，我们将知道即使丢失两个磁盘也不会导致完全的 RAID 失败。

仅从`mdadm`命令的前半部分，我们就可以看到关于这个 RAID 的相当多的信息。从后半部分，我们将找到更多关键信息，如下所示：

```
 Intent Bitmap : Internal

 Update Time : Sun May 10 06:16:10 2015
 State : clean, degraded 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
 Spare Devices : 0

 Name : localhost:boot
 UUID : 7adf0323:b0962394:387e6cd0:b2914469
 Events : 52

 Number   Major   Minor   RaidDevice State
 0       8        1        0      active sync   /dev/sda1
 2       0        0        2      removed

```

`Update Time`很有用，因为它显示了此 RAID 更改状态的最后时间，无论该状态更改是添加磁盘还是重建。

这个时间戳可能很有用，特别是如果我们试图将其与`/var/log/messages`或其他系统事件中的日志条目相关联。

另一个重要的信息是`RAID Device State`，对于我们的例子来说，是 clean, degraded。降级状态意味着 RAID 有一个失败的设备，但 RAID 本身仍然是功能性的。Degraded 只是意味着功能性但不是最佳状态。

如果我们的 RAID 设备正在进行重建或恢复，我们也会看到这些状态列出。

在当前状态输出下，我们可以看到四个设备类别，告诉我们关于用于此 RAID 的硬盘的信息。第一个是`Active Devices`；这告诉我们当前在 RAID 中活动的驱动器数量。

第二个是`Working Devices`；这告诉我们工作驱动器的数量。通常，`Working Devices`和`Active Devices`的数量将是相同的。

列表中的第四项是`Failed Devices`；这是当前标记为失败的设备数量。尽管我们的 RAID 目前有一个失败的设备，但这个数字是`0`。有一个有效的原因，但我们稍后会解释这个原因。

列表中的最后一项是`Spare Devices`的数量。在一些 RAID 系统中，您可以创建备用设备，用于在发生诸如驱动器故障之类的事件中重建 RAID。

这些备用设备可能会派上用场，因为 RAID 系统通常会自动重建 RAID，从而降低 RAID 完全失败的可能性。

通过`mdadm`的输出的最后两行，我们可以看到组成 RAID 的驱动器的信息。

```
 Number   Major   Minor   RaidDevice State
 0       8        1        0      active sync   /dev/sda1
 2       0        0        2      removed

```

从输出中，我们可以看到我们有一个磁盘设备`/dev/sda1`，目前处于活动同步状态。我们还可以看到另一个设备已从 RAID 中移除。

### 总结关键信息

从`mdadm --detail`的输出中，我们可以看到`/dev/md127`是一个 RAID 设备，其 RAID 级别为 1，目前处于降级状态。我们可以从详细信息中看到，降级状态是由于组成 RAID 的驱动器之一目前被移除。

## 使用/proc/mdstat 查看 md 状态

另一个查看 MD 当前状态的有用位置是`/proc/mdstat`；这个文件和`/proc`中的许多文件一样，是由内核不断更新的。如果我们使用`cat`命令来读取这个文件，我们可以快速查看服务器当前的 RAID 状态：

```
[nfs]# cat /proc/mdstat 
Personalities : [raid1] 
md126 : active raid1 sda2[0]
 7871488 blocks super 1.2 [2/1] [U_]
 bitmap: 1/1 pages [4KB], 65536KB chunk

md127 : active raid1 sda1[0]
 511936 blocks super 1.0 [2/1] [U_]
 bitmap: 1/1 pages [4KB], 65536KB chunk

unused devices: <none>

```

`/proc/mdstat`的内容有些晦涩，但如果我们分解它们，它包含了相当多的信息。

```
Personalities : [raid1]

```

第一行的`Personalities`告诉我们这个系统上内核当前支持的 RAID 级别。对于我们的例子，它是 RAID 1：

```
md126 : active raid1 sda2[0]
 7871488 blocks super 1.2 [2/1] [U_]
 bitmap: 1/1 pages [4KB], 65536KB chunk

```

接下来的几行是`/dev/md126`的当前状态，这是系统上另一个我们还没有看过的 RAID 设备。这三行实际上可以给我们提供关于`md126`的相当多的信息；事实上，它们给了我们和`mdadm --detail`几乎相同的信息。

```
md126 : active raid1 sda2[0]

```

在第一行中，我们可以看到设备名称`md126`。我们可以看到 RAID 的当前状态是活动的。我们还可以看到这个 RAID 设备的 RAID 级别是 RAID 1。最后，我们还可以看到组成这个 RAID 的磁盘设备；在我们的例子中，只有`sda2`。

第二行还包含以下关键信息：

```
 7871488 blocks super 1.2 [2/1] [U_]

```

具体来说，最后两个值对我们当前的任务最有用，`[2/1]`显示了分配给这个 RAID 的磁盘设备数量以及可用的数量。从例子中的值我们可以看到，期望有 2 个驱动器，但只有 1 个可用。

最后一个值`[U_]`显示了组成这个 RAID 的驱动器的当前状态。状态 U 表示正常，"_"表示不正常。

在我们的例子中，我们可以看到一个磁盘设备是正常的，另一个是不正常的。

根据以上信息，我们能够确定 RAID 设备`/dev/md126`目前处于活动状态；它正在使用 RAID 级别 1，目前有两个磁盘中的一个不可用。

如果我们继续查看`/proc/mdstat`文件，我们可以看到`md127`的状态也类似。

### 使用/proc/mdstat 和 mdadm

在查看`/proc/mdstat`和`mdadm --detail`之后，我们可以看到两者提供了类似的信息。根据我的经验，我发现同时使用`mdstat`和`mdadm`是有用的。`/proc/mdstat`文件通常是我快速查看系统上所有 RAID 设备的快照的地方，而`mdadm`命令通常是我用来获取更深入的 RAID 设备详细信息的地方（例如备用驱动器的数量、创建时间和最后更新时间等细节）。

# 识别更大的问题

之前在使用`mdadm`查看`md127`的当前状态时，我们发现 RAID 设备`md127`有一个磁盘被移出服务。在查看`/proc/mdstat`时，我们发现还有另一个 RAID 设备`/dev/md126`，也有一个磁盘被移出服务。

我们还可以看到的另一个有趣的事实是，RAID 设备`/dev/md126`是一个存活的磁盘：`/dev/sda1`。这很有趣，因为`/dev/md127`的存活磁盘是`/dev/sda2`。如果我们记得之前的章节，`/dev/sda1`和`/dev/sda2`只是来自同一物理磁盘的两个分区。考虑到两个 RAID 设备都有一个丢失的驱动器，而我们的日志表明`/dev/md127`曾经将`/dev/sdb1`移除并重新添加。很可能`/dev/md127`和`/dev/md126`都在使用`/dev/sdb`的分区。

由于`/proc/mdstat`对于 RAID 设备只有两种状态，正常或不正常，我们可以使用`--detail`标志来确认第二个磁盘是否真的已经从`/dev/md126`中移除：

```
[nfs]# mdadm --detail /dev/md126
/dev/md126:
 Version : 1.2
 Creation Time : Wed Apr 15 09:39:19 2015
 Raid Level : raid1
 Array Size : 7871488 (7.51 GiB 8.06 GB)
 Used Dev Size : 7871488 (7.51 GiB 8.06 GB)
 Raid Devices : 2
 Total Devices : 1
 Persistence : Superblock is persistent

 Intent Bitmap : Internal

 Update Time : Mon May 11 04:03:09 2015
 State : clean, degraded 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
 Spare Devices : 0

 Name : localhost:pv00
 UUID : bec13d99:42674929:76663813:f748e7cb
 Events : 5481

 Number   Major   Minor   RaidDevice State
 0       8        2        0      active sync   /dev/sda2
 2       0        0        2      removed

```

从输出中，我们可以看到`/dev/md126`的当前状态和配置与`/dev/md127`完全相同。根据这个信息，我们可以假设`/dev/md126`曾经将`/dev/sdb2`作为其 RAID 的一部分。

由于我们怀疑问题可能只是一个硬盘出了问题，我们需要验证这是否真的是这种情况。第一步是确定是否真的存在`/dev/sdb`设备；这样做的最快方法是使用`ls`命令在`/dev`中执行目录列表：

```
[nfs]# ls -la /dev/ | grep sd
brw-rw----.  1 root disk      8,   0 May 10 06:16 sda
brw-rw----.  1 root disk      8,   1 May 10 06:16 sda1
brw-rw----.  1 root disk      8,   2 May 10 06:16 sda2
brw-rw----.  1 root disk      8,  16 May 10 06:16 sdb
brw-rw----.  1 root disk      8,  17 May 10 06:16 sdb1
brw-rw----.  1 root disk      8,  18 May 10 06:16 sdb2

```

从这个`ls`命令的结果中，我们可以看到实际上有一个`sdb`、`sdb1`和`sdb2`设备。在进一步之前，让我们更清楚地了解一下`/dev`。

# 理解/dev

`/dev`目录是一个特殊的目录，其中的内容是内核在安装时创建的。该目录包含特殊文件，允许用户或应用程序与物理设备，有时是逻辑设备进行交互。

如果我们看一下之前`ls`命令的结果，我们可以看到在`/dev`目录中有几个以`sd`开头的文件。

在上一章中，我们学到以`sd`开头的文件实际上被视为 SCSI 或 SATA 驱动器。在我们的情况下，我们有`/dev/sda`和`/dev/sdb`；这意味着在这个系统上有两个物理 SCSI 或 SATA 驱动器。

额外的设备`/dev/sda1`、`/dev/sda2`、`/dev/sdb1`和`/dev/sdb2`只是这些磁盘的分区。实际上，对于磁盘驱动器，以数字结尾的设备名称通常是另一个设备的分区，就像`/dev/sdb1`是`/dev/sdb`的分区一样。虽然当然也有一些例外，但通常在排除磁盘驱动器故障时，做出这种假设是安全的。

## 不仅仅是磁盘驱动器

`/dev/`目录中包含的不仅仅是磁盘驱动器。如果我们查看`/dev/`，我们实际上可以看到一些常见的设备。

```
[nfs]# ls -F /dev
autofs           hugepages/       network_throughput  snd/     tty21  tty4   tty58    vcs1
block/           initctl|         null                sr0      tty22  tty40  tty59    vcs2
bsg/             input/           nvram               stderr@  tty23  tty41  tty6     vcs3
btrfs-control    kmsg             oldmem              stdin@   tty24  tty42  tty60    vcs4
bus/             log=             port                stdout@  tty25  tty43  tty61    vcs5
cdrom@           loop-control     ppp                 tty      tty26  tty44  tty62    vcs6
char/            lp0              ptmx                tty0     tty27  tty45  tty63    vcsa
console          lp1              pts/                tty1     tty28  tty46  tty7     vcsa1
core@            lp2              random              tty10    tty29  tty47  tty8     vcsa2
cpu/             lp3              raw/                tty11    tty3   tty48  tty9     vcsa3
cpu_dma_latency  mapper/          rtc@                tty12    tty30  tty49  ttyS0    vcsa4
crash            mcelog           rtc0                tty13    tty31  tty5   ttyS1    vcsa5
disk/            md/              sda                 tty14    tty32  tty50  ttyS2    vcsa6
dm-0             md0/             sda1                tty15    tty33  tty51  ttyS3    vfio/
dm-1             md126            sda2                tty16    tty34  tty52  uhid     vga_arbiter
dm-2             md127            sdb                 tty17    tty35  tty53  uinput   vhost-net
fd@              mem              sdb1                tty18    tty36  tty54  urandom  zero
full             mqueue/          sdb2                tty19    tty37  tty55  usbmon0
fuse             net/             shm/                tty2     tty38  tty56  usbmon1
hpet             network_latency  snapshot            tty20    tty39  tty57  vcs

```

从这个`ls`的结果中，我们可以看到`/dev`目录中有许多文件、目录和符号链接。

以下是一些常见的有用的设备或目录列表：

+   **/dev/cdrom**：这通常是一个指向`cdrom`设备的符号链接。CD-ROM 的实际设备遵循类似硬盘的命名约定，它以`sr`开头，后面跟着设备的编号。我们可以用`ls`命令看到`/dev/cdrom`符号链接指向哪里：

```
[nfs]# ls -la /dev/cdrom
lrwxrwxrwx. 1 root root 3 May 10 06:16 /dev/cdrom -> sr0

```

+   **/dev/console**：这个设备不一定与特定的硬件设备（如`/dev/sda`或`/dev/sr0`）相关联。控制台设备用于与系统控制台进行交互，这可能是实际的监视器，也可能不是。

+   **/dev/cpu**：实际上，这是一个目录，其中包含系统上每个 CPU 的附加目录。在这些目录中有一个`cpuid`文件，用于查询有关 CPU 的信息：

```
[nfs]# ls -la /dev/cpu/0/cpuid 
crw-------. 1 root root 203, 0 May 10 06:16 /dev/cpu/0/cpuid

```

+   **/dev/md**：这是另一个目录，其中包含指向实际 RAID 设备的用户友好名称的符号链接。如果我们使用`ls`，我们可以看到该系统上可用的 RAID 设备：

```
[nfs]# ls -la /dev/md/
total 0
drwxr-xr-x.  2 root root   80 May 10 06:16 .
drwxr-xr-x. 20 root root 3180 May 10 06:16 ..
lrwxrwxrwx.  1 root root    8 May 10 06:16 boot -> ../md127
lrwxrwxrwx.  1 root root    8 May 10 06:16 pv00 -> ../md126

```

+   **/dev/random**和**/dev/urandom**：这两个设备用于生成随机数据。`/dev/random`和`/dev/urandom`设备都会从内核的熵池中提取随机数据。这两者之间的一个区别是，当系统的熵计数较低时，`/dev/random`设备将等待直到重新添加足够的熵。

正如我们之前学到的，`/dev/`目录中有一些非常有用的文件和目录。然而，回到我们最初的问题，我们已经确定`/dev/sdb`存在，并且有两个分区`/dev/sdb1`和`/dev/sdb2`。

然而，我们还没有确定`/dev/sdb`最初是否是两个当前处于降级状态的 RAID 设备的一部分。为了做到这一点，我们可以利用`dmesg`设施。

# 使用 dmesg 查看设备消息

`dmesg`命令是一个用于排除硬件问题的好命令。当系统初始启动时，内核将识别该系统可用的各种硬件设备。

当内核识别这些设备时，信息被写入内核的环形缓冲区。这个环形缓冲区本质上是内核的内部日志。`dmesg`命令可以用来打印这个环形缓冲区。

以下是`dmesg`命令的一个示例输出；在这个示例中，我们将使用`head`命令将输出缩短到只有前 15 行：

```
[nfs]# dmesg | head -15
[    0.000000] Initializing cgroup subsys cpuset
[    0.000000] Initializing cgroup subsys cpu
[    0.000000] Initializing cgroup subsys cpuacct
[    0.000000] Linux version 3.10.0-229.1.2.el7.x86_64 (builder@kbuilder.dev.centos.org) (gcc version 4.8.2 20140120 (Red Hat 4.8.2-16) (GCC) ) #1 SMP Fri Mar 27 03:04:26 UTC 2015
[    0.000000] Command line: BOOT_IMAGE=/vmlinuz-3.10.0-229.1.2.el7.x86_64 root=/dev/mapper/md0-root ro rd.lvm.lv=md0/swap crashkernel=auto rd.md.uuid=bec13d99:42674929:76663813:f748e7cb rd.lvm.lv=md0/root rd.md.uuid=7adf0323:b0962394:387e6cd0:b2914469 rhgb quiet LANG=en_US.UTF-8 systemd.debug
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000001ffeffff] usable
[    0.000000] BIOS-e820: [mem 0x000000001fff0000-0x000000001fffffff] ACPI data
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
[    0.000000] NX (Execute Disable) protection: active
[    0.000000] SMBIOS 2.5 present.
[    0.000000] DMI: innotek GmbH VirtualBox/VirtualBox, BIOS VirtualBox 12/01/2006

```

我们将输出限制为只有 15 行，是因为`dmesg`命令会输出大量的数据。换个角度来看，我们可以再次运行命令，但这次将输出发送到`wc -l`，它将计算打印的行数：

```
[nfs]# dmesg | wc -l
597

```

正如我们所看到的，`dmesg`命令返回了`597`行。阅读内核环形缓冲区的所有 597 行并不是一个快速的过程。

由于我们的目标是找出关于`/dev/sdb`的信息，我们可以再次运行`dmesg`命令，这次使用`grep`命令来过滤输出到`/dev/sdb`相关的信息：

```
[nfs]# dmesg | grep -C 5 sdb
[    2.176800] scsi 3:0:0:0: CD-ROM            VBOX     CD-ROM           1.0  PQ: 0 ANSI: 5
[    2.194908] sd 0:0:0:0: [sda] 16777216 512-byte logical blocks: (8.58 GB/8.00 GiB)
[    2.194951] sd 0:0:0:0: [sda] Write Protect is off
[    2.194953] sd 0:0:0:0: [sda] Mode Sense: 00 3a 00 00
[    2.194965] sd 0:0:0:0: [sda] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
[    2.196250] sd 1:0:0:0: [sdb] 16777216 512-byte logical blocks: (8.58 GB/8.00 GiB)
[    2.196279] sd 1:0:0:0: [sdb] Write Protect is off
[    2.196281] sd 1:0:0:0: [sdb] Mode Sense: 00 3a 00 00
[    2.196294] sd 1:0:0:0: [sdb] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
[    2.197471]  sda: sda1 sda2
[    2.197700] sd 0:0:0:0: [sda] Attached SCSI disk
[    2.198139]  sdb: sdb1 sdb2
[    2.198319] sd 1:0:0:0: [sdb] Attached SCSI disk
[    2.200851] sr 3:0:0:0: [sr0] scsi3-mmc drive: 32x/32x xa/form2 tray
[    2.200856] cdrom: Uniform CD-ROM driver Revision: 3.20
[    2.200980] sr 3:0:0:0: Attached scsi CD-ROM sr0
[    2.366634] md: bind<sda1>
[    2.370652] md: raid1 personality registered for level 1
[    2.370820] md/raid1:md127: active with 1 out of 2 mirrors
[    2.371797] created bitmap (1 pages) for device md127
[    2.372181] md127: bitmap initialized from disk: read 1 pages, set 0 of 8 bits
[    2.373915] md127: detected capacity change from 0 to 524222464
[    2.374767]  md127: unknown partition table
[    2.376065] md: bind<sdb2>
[    2.382976] md: bind<sda2>
[    2.385094] md: kicking non-fresh sdb2 from array!
[    2.385102] md: unbind<sdb2>
[    2.385105] md: export_rdev(sdb2)
[    2.387559] md/raid1:md126: active with 1 out of 2 mirrors
[    2.387874] created bitmap (1 pages) for device md126
[    2.388339] md126: bitmap initialized from disk: read 1 pages, set 19 of 121 bits
[    2.390324] md126: detected capacity change from 0 to 8060403712
[    2.391344]  md126: unknown partition table

```

在执行前面的示例时，使用了`-C`（上下文）标志来告诉`grep`将五行上下文包含在输出中。通常情况下，当`grep`不带标志运行时，只会打印包含搜索字符串（本例中为"`sdb`"）的行。将上下文标志设置为五，`grep`命令将打印包含搜索字符串的每一行之前和之后的 5 行。

这种使用`grep`的方法使我们不仅能看到包含字符串`sdb`的行，还能看到之前和之后的行，这些行可能包含额外的信息。

现在我们有了这些额外的信息，让我们来分解一下，以更好地理解它告诉我们的内容：

```
[    2.176800] scsi 3:0:0:0: CD-ROM            VBOX     CD-ROM           1.0  PQ: 0 ANSI: 5
[    2.194908] sd 0:0:0:0: [sda] 16777216 512-byte logical blocks: (8.58 GB/8.00 GiB)
[    2.194951] sd 0:0:0:0: [sda] Write Protect is off
[    2.194953] sd 0:0:0:0: [sda] Mode Sense: 00 3a 00 00
[    2.194965] sd 0:0:0:0: [sda] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
[    2.196250] sd 1:0:0:0: [sdb] 16777216 512-byte logical blocks: (8.58 GB/8.00 GiB)
[    2.196279] sd 1:0:0:0: [sdb] Write Protect is off
[    2.196281] sd 1:0:0:0: [sdb] Mode Sense: 00 3a 00 00
[    2.196294] sd 1:0:0:0: [sdb] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
[    2.197471]  sda: sda1 sda2
[    2.197700] sd 0:0:0:0: [sda] Attached SCSI disk
[    2.198139]  sdb: sdb1 sdb2
[    2.198319] sd 1:0:0:0: [sdb] Attached SCSI disk

```

前面的信息似乎是关于`/dev/sdb`的标准信息。我们可以从这些消息中看到关于`/dev/sda`和`/dev/sdb`的一些基本信息。

从前面的信息中我们可以看到一个有用的东西是这些驱动器的大小：

```
[    2.194908] sd 0:0:0:0: [sda] 16777216 512-byte logical blocks: (8.58 GB/8.00 GiB)
[    2.196250] sd 1:0:0:0: [sdb] 16777216 512-byte logical blocks: (8.58 GB/8.00 GiB)

```

我们可以看到每个驱动器的大小为`8.58`GB。虽然这些信息一般来说是有用的，但对于我们目前的情况并不适用。然而，有用的是前面代码片段的最后四行：

```
[    2.197471]  sda: sda1 sda2
[    2.197700] sd 0:0:0:0: [sda] Attached SCSI disk
[    2.198139]  sdb: sdb1 sdb2
[    2.198319] sd 1:0:0:0: [sdb] Attached SCSI disk

```

这最后的四行显示了`/dev/sda`和`/dev/sdb`上的可用分区，以及一条消息说明每个磁盘都已经`Attached`。

这些信息非常有用，因为它在最基本的层面上告诉我们这两个驱动器正在工作。这对于`/dev/sdb`来说是一个问题，因为我们怀疑 RAID 系统已经将其移出了服务。

到目前为止，`dmesg`命令已经给了我们一些有用的信息；让我们继续查看数据，以更好地理解这些磁盘。

```
[    2.200851] sr 3:0:0:0: [sr0] scsi3-mmc drive: 32x/32x xa/form2 tray
[    2.200856] cdrom: Uniform CD-ROM driver Revision: 3.20
[    2.200980] sr 3:0:0:0: Attached scsi CD-ROM sr0

```

前面的三行在我们排除 CD-ROM 设备问题时可能有用。然而，对于我们的磁盘问题，它们并不有用，只是因为`grep`的上下文设置为 5 而包含在内。

然而，以下的行将会告诉我们关于我们的磁盘驱动器的很多信息：

```
[    2.366634] md: bind<sda1>
[    2.370652] md: raid1 personality registered for level 1
[    2.370820] md/raid1:md127: active with 1 out of 2 mirrors
[    2.371797] created bitmap (1 pages) for device md127
[    2.372181] md127: bitmap initialized from disk: read 1 pages, set 0 of 8 bits
[    2.373915] md127: detected capacity change from 0 to 524222464
[    2.374767]  md127: unknown partition table
[    2.376065] md: bind<sdb2>
[    2.382976] md: bind<sda2>
[    2.385094] md: kicking non-fresh sdb2 from array!
[    2.385102] md: unbind<sdb2>
[    2.385105] md: export_rdev(sdb2)
[    2.387559] md/raid1:md126: active with 1 out of 2 mirrors
[    2.387874] created bitmap (1 pages) for device md126
[    2.388339] md126: bitmap initialized from disk: read 1 pages, set 19 of 121 bits
[    2.390324] md126: detected capacity change from 0 to 8060403712
[    2.391344]  md126: unknown partition table

```

dmesg 输出的最后一部分告诉了我们关于 RAID 设备和`/dev/sdb`的很多信息。由于数据量很大，我们需要将其分解以真正理解其中的内容：

```
The first few lines show use information about /dev/md127.
[    2.366634] md: bind<sda1>
[    2.370652] md: raid1 personality registered for level 1
[    2.370820] md/raid1:md127: active with 1 out of 2 mirrors
[    2.371797] created bitmap (1 pages) for device md127
[    2.372181] md127: bitmap initialized from disk: read 1 pages, set 0 of 8 bits
[    2.373915] md127: detected capacity change from 0 to 524222464
[    2.374767]  md127: unknown partition table

```

```
/dev/md126; however, there is a bit more information included with those messages:
```

```
[    2.376065] md: bind<sdb2>
[    2.382976] md: bind<sda2>
[    2.385094] md: kicking non-fresh sdb2 from array!
[    2.385102] md: unbind<sdb2>
[    2.385105] md: export_rdev(sdb2)
[    2.387559] md/raid1:md126: active with 1 out of 2 mirrors
[    2.387874] created bitmap (1 pages) for device md126
[    2.388339] md126: bitmap initialized from disk: read 1 pages, set 19 of 121 bits
[    2.390324] md126: detected capacity change from 0 to 8060403712
[    2.391344]  md126: unknown partition table

```

前面的消息看起来与`/dev/md127`的消息非常相似；然而，有几行消息在`/dev/md127`的消息中没有出现：

```
[    2.376065] md: bind<sdb2>
[    2.382976] md: bind<sda2>
[    2.385094] md: kicking non-fresh sdb2 from array!
[    2.385102] md: unbind<sdb2>

```

如果我们看这些消息，我们可以看到`/dev/md126`尝试在 RAID 阵列中使用`/dev/sdb2`；然而，它发现该驱动器不是新的。非新的消息很有趣，因为它可能解释了为什么`/dev/sdb`没有被包含到 RAID 设备中。

## 总结 dmesg 提供的信息

在 RAID 集中，每个磁盘都维护每个写请求的事件计数。RAID 使用这个事件计数来确保每个磁盘接收了适当数量的写请求。这使得 RAID 能够验证整个 RAID 的一致性。

当 RAID 重新启动时，RAID 管理器将检查每个磁盘的事件计数，并确保它们是一致的。

从前面的消息中，看起来`/dev/sda2`的事件计数可能比`/dev/sdb2`高。这表明`/dev/sda1`上发生了一些写操作，而`/dev/sdb2`上从未发生过。这对于镜像阵列来说是异常的，也表明了`/dev/sdb2`存在问题。

当设备名称发生变化时，我们如何检查事件计数是否不同？使用`mdadm`命令，我们可以显示每个磁盘设备的事件计数。

# 使用 mdadm 来检查超级块

为了查看事件计数，我们将使用`mdadm`命令和`--examine`标志来检查磁盘设备：

```
[nfs]# mdadm --examine /dev/sda1
/dev/sda1:
 Magic : a92b4efc
 Version : 1.0
 Feature Map : 0x1
 Array UUID : 7adf0323:b0962394:387e6cd0:b2914469
 Name : localhost:boot
 Creation Time : Wed Apr 15 09:39:22 2015
 Raid Level : raid1
 Raid Devices : 2

 Avail Dev Size : 1023968 (500.07 MiB 524.27 MB)
 Array Size : 511936 (500.02 MiB 524.22 MB)
 Used Dev Size : 1023872 (500.02 MiB 524.22 MB)
 Super Offset : 1023984 sectors
 Unused Space : before=0 sectors, after=96 sectors
 State : clean
 Device UUID : 92d97c32:1f53f59a:14a7deea:34ec8c7c

Internal Bitmap : -16 sectors from superblock
 Update Time : Mon May 11 04:08:10 2015
 Bad Block Log : 512 entries available at offset -8 sectors
 Checksum : bd8c1d5b - correct
 Events : 60

 Device Role : Active device 0
 Array State : A. ('A' == active, '.' == missing, 'R' == replacing)

```

`--examine`标志与`--detail`非常相似，不同之处在于`--detail`用于打印 RAID 设备的详细信息。`--examine`用于打印组成 RAID 的单个磁盘的 RAID 详细信息。`--examine`打印的详细信息实际上来自磁盘上的超级块详细信息。

当 Linux RAID 利用磁盘作为 RAID 设备的一部分时，RAID 系统会在磁盘上保留一些空间用于**超级块**。这个超级块简单地用于存储关于磁盘和 RAID 的元数据。

在前面的命令中，我们只是打印了`/dev/sda1`的 RAID 超级块信息。为了更好地理解 RAID 超级块，让我们看一下`--examine`标志提供的详细信息：

```
/dev/sda1:
 Magic : a92b4efc
 Version : 1.0
 Feature Map : 0x1
 Array UUID : 7adf0323:b0962394:387e6cd0:b2914469
 Name : localhost:boot
 Creation Time : Wed Apr 15 09:39:22 2015
 Raid Level : raid1
 Raid Devices : 2

```

这个输出的第一部分提供了相当多有用的信息。例如，魔术数字被用作超级块头。这是一个用来指示超级块开始的值。

另一个有用的信息是`Array UUID`。这是这个磁盘所属的 RAID 的唯一标识符。如果我们打印`md127`的 RAID 的详细信息，我们可以看到`/dev/sda1`的 Array UUID 和`md127`的 UUID 是匹配的：

```
[nfs]# mdadm --detail /dev/md127 | grep UUID
 UUID : 7adf0323:b0962394:387e6cd0:b2914469

```

当 Linux RAID 利用磁盘作为 RAID 设备的一部分时，RAID 系统会在磁盘上保留一些空间用于**超级块**。这个超级块简单地用于存储关于磁盘和 RAID 的元数据。

底部的三行`Creation Time`、`RAID Level`和`RAID Devices`在与`--detail`输出一起使用时也非常有用。

这第二段信息对于确定磁盘设备的信息非常有用：

```
Avail Dev Size : 1023968 (500.07 MiB 524.27 MB)
 Array Size : 511936 (500.02 MiB 524.22 MB)
 Used Dev Size : 1023872 (500.02 MiB 524.22 MB)
 Super Offset : 1023984 sectors
 Unused Space : before=0 sectors, after=96 sectors
 State : clean
 Device UUID : 92d97c32:1f53f59a:14a7deea:34ec8c7c

```

```
State of the RAID. This state matches the state we see from the --detail output of /dev/md127.
```

```
[nfs]# mdadm --detail /dev/md127 | grep State
 State : clean, degraded

```

`--examine`输出的下一部分信息对我们的问题非常有用：

```
Internal Bitmap : -16 sectors from superblock
 Update Time : Mon May 11 04:08:10 2015
 Bad Block Log : 512 entries available at offset -8 sectors
 Checksum : bd8c1d5b - correct
 Events : 60

 Device Role : Active device 0
 Array State : A. ('A' == active, '.' == missing, 'R' == replacing)

```

在这一部分中，我们可以看到`Events`信息，显示了这个磁盘上的当前事件计数值。我们还可以看到`/dev/sda1`的`Array State`值。`A`的值表示从`/dev/sda1`的角度来看，它的镜像伙伴丢失了。

当我们检查`/dev/sdb1`下超级块的详细信息时，我们会看到`Array State`和`Events`值的一个有趣的差异：

```
[nfs]# mdadm --examine /dev/sdb1
/dev/sdb1:
 Magic : a92b4efc
 Version : 1.0
 Feature Map : 0x1
 Array UUID : 7adf0323:b0962394:387e6cd0:b2914469
 Name : localhost:boot
 Creation Time : Wed Apr 15 09:39:22 2015
 Raid Level : raid1
 Raid Devices : 2

 Avail Dev Size : 1023968 (500.07 MiB 524.27 MB)
 Array Size : 511936 (500.02 MiB 524.22 MB)
 Used Dev Size : 1023872 (500.02 MiB 524.22 MB)
 Super Offset : 1023984 sectors
 Unused Space : before=0 sectors, after=96 sectors
 State : clean
 Device UUID : 5a9bb172:13102af9:81d761fb:56d83bdd

Internal Bitmap : -16 sectors from superblock
 Update Time : Mon May  4 21:09:30 2015
 Bad Block Log : 512 entries available at offset -8 sectors
 Checksum : cd226d7b - correct
 Events : 48

 Device Role : Active device 1
 Array State : AA ('A' == active, '.' == missing, 'R' == replacing)

```

从结果来看，我们已经回答了关于`/dev/sdb1`的很多问题。

我们最初的问题是`/dev/sdb1`是否是 RAID 的一部分。从这个设备具有 RAID 超级块并且可以通过`mdadm`打印信息的事实来看，我们可以肯定是。

```
 Array UUID : 7adf0323:b0962394:387e6cd0:b2914469

```

通过查看`Array UUID`，我们还可以确定这个设备是否是`/dev/md127`的一部分，正如我们所怀疑的那样：

```
[nfs]# mdadm --detail /dev/md127 | grep UUID
 UUID : 7adf0323:b0962394:387e6cd0:b2914469

```

看起来`/dev/sdb1`在某个时候是`/dev/md127`的一部分。

我们需要回答的最后一个问题是`/dev/sda1`和`/dev/sdb1`之间的`Events`值是否不同。从`/dev/sda1`的`--examine`信息中，我们可以看到事件计数设置为 60。在前面的代码中，从`/dev/sdb1`的`--examine`结果中，我们可以看到事件计数要低得多——48：

```
 Events : 48

```

鉴于这种差异，我们可以确定`/dev/sdb1`比`/dev/sda1`落后 12 个事件。这是一个非常重要的差异，也是 MD 拒绝将`/dev/sdb1`添加到 RAID 数组的一个合理原因。

有趣的是，如果我们查看`/dev/sdb1`的`Array State`，我们可以看到它仍然认为自己是`/dev/md127`数组中的一个活动磁盘：

```
 Array State : AA ('A' == active, '.' == missing, 'R' == replacing)

```

这是因为由于设备不再是 RAID 的一部分，它不会被更新为当前状态。我们也可以从更新时间中看到这一点：

```
 Update Time : Mon May  4 21:09:30 2015

```

`/dev/sda1`的“更新时间”要新得多；因此，应该比磁盘`/dev/sdb1`更可信。

## 检查`/dev/sdb2`

现在我们知道了`/dev/sdb1`未被添加到`/dev/md127`的原因，我们应该确定是否对`/dev/sdb2`和`/dev/md126`也是如此。

由于我们已经知道`/dev/sda2`是健康的并且是`/dev/md126`数组的一部分，我们将专注于捕获其“事件”值：

```
[nfs]# mdadm --examine /dev/sda2 | grep Events
 Events : 7517

```

与`/dev/sda1`相比，`/dev/sda2`的事件计数相当高。从中我们可以确定`/dev/md126`可能是一个非常活跃的 RAID 设备。

现在我们知道了事件计数，让我们来看看`/dev/sdb2`的详细信息：

```
[nfs]# mdadm --examine /dev/sdb2
/dev/sdb2:
 Magic : a92b4efc
 Version : 1.2
 Feature Map : 0x1
 Array UUID : bec13d99:42674929:76663813:f748e7cb
 Name : localhost:pv00
 Creation Time : Wed Apr 15 09:39:19 2015
 Raid Level : raid1
 Raid Devices : 2

 Avail Dev Size : 15742976 (7.51 GiB 8.06 GB)
 Array Size : 7871488 (7.51 GiB 8.06 GB)
 Data Offset : 8192 sectors
 Super Offset : 8 sectors
 Unused Space : before=8104 sectors, after=0 sectors
 State : clean
 Device UUID : 01db1f5f:e8176cad:8ce68d51:deff57f8

Internal Bitmap : 8 sectors from superblock
 Update Time : Mon May  4 21:10:31 2015
 Bad Block Log : 512 entries available at offset 72 sectors
 Checksum : 98a8ace8 - correct
 Events : 541

 Device Role : Active device 1
 Array State : AA ('A' == active, '.' == missing, 'R' == replacing)

```

同样，从我们能够从`/dev/sdb2`打印超级块信息的事实中，我们已经确定这个设备实际上是 RAID 的一部分：

```
 Array UUID : bec13d99:42674929:76663813:f748e7cb

```

如果我们将`/dev/sdb2`的“数组 UUID”与`/dev/md126`的`UUID`进行比较，我们还将看到它实际上是该 RAID 数组的一部分：

```
[nfs]# mdadm --detail /dev/md126 | grep UUID
 UUID : bec13d99:42674929:76663813:f748e7cb

```

这回答了我们关于`/dev/sdb2`是否是`md126` RAID 的一部分的问题。如果我们查看`/dev/sdb2`的事件计数，我们也可以回答为什么它目前不是该 RAID 的一部分的问题：

```
Events : 541

```

看起来这个设备错过了发送到`md126` RAID 的写事件，因为`/dev/sda2`的“事件”计数为 7517，而`/dev/sdb2`的“事件”计数为 541。

# 到目前为止我们学到的内容

到目前为止，我们已经采取了一些故障排除步骤，收集了一些关键数据。让我们走一遍我们学到的东西，以及我们可以从这些发现中推断出什么：

+   在我们的系统上，我们有两个 RAID 设备。

使用`mdadm`命令和`/proc/mdstat`的内容，我们能够确定该系统有两个 RAID 设备—`/dev/md126`和`/dev/md127`。

+   两个 RAID 设备都是 RAID 1，缺少一个镜像设备。

通过`mdadm`命令和`dmesg`输出，我们能够确定两个 RAID 设备都设置为 RAID 1 设备。此外，我们还能够看到两个 RAID 设备都缺少一个磁盘；缺少的设备都是来自`/dev/sdb`硬盘的分区。

+   `/dev/sdb1`和`/dev/sdb2`的事件计数不匹配。

通过`mdadm`命令，我们能够检查`/dev/sdb1`和`/dev/sdb2`设备的`superblock`详细信息。在此期间，我们能够看到这些设备的事件计数与`/dev/sda`上的活动分区不匹配。

因此，RAID 不会将`/dev/sdb`设备重新添加到它们各自的 RAID 数组中。

+   磁盘`/dev/sdb`似乎是正常的。

虽然 RAID 没有将`/dev/sdb1`或`/dev/sdb2`添加到各自的 RAID 数组中，但这并不意味着设备`/dev/sdb`有故障。

从`dmesg`中的消息中，我们没有看到`/dev/sdb`设备本身的任何错误。我们还能够使用`mdadm`来检查这些驱动器上的分区。从到目前为止我们所做的一切来看，这些驱动器似乎是正常的。

# 重新将驱动器添加到数组

`/dev/sdb`磁盘似乎是正常的，除了事件计数的差异外，我们看不到 RAID 拒绝设备的任何原因。我们的下一步将是尝试将已移除的设备重新添加到它们的 RAID 数组中。

我们将首先尝试这样做的第一个 RAID 是`/dev/md127`：

```
[nfs]# mdadm --detail /dev/md127
/dev/md127:
 Version : 1.0
 Creation Time : Wed Apr 15 09:39:22 2015
 Raid Level : raid1
 Array Size : 511936 (500.02 MiB 524.22 MB)
 Used Dev Size : 511936 (500.02 MiB 524.22 MB)
 Raid Devices : 2
 Total Devices : 1
 Persistence : Superblock is persistent

 Intent Bitmap : Internal

 Update Time : Mon May 11 04:08:10 2015
 State : clean, degraded 
 Active Devices : 1
Working Devices : 1
 Failed Devices : 0
 Spare Devices : 0

 Name : localhost:boot
 UUID : 7adf0323:b0962394:387e6cd0:b2914469
 Events : 60

 Number   Major   Minor   RaidDevice State
 0       8        1        0      active sync   /dev/sda1
 2       0        0        2      removed

```

重新添加驱动器的最简单方法就是简单地使用`mdadm`的`-a`（添加）标志。

```
[nfs]# mdadm /dev/md127 -a /dev/sdb1
mdadm: re-added /dev/sdb1

```

上述命令将告诉`mdadm`将设备`/dev/sdb1`添加到 RAID 设备`/dev/md127`中。由于`/dev/sdb1`已经是 RAID 数组的一部分，MD 服务只是重新添加磁盘并同步来自`/dev/sda1`的丢失事件。

如果我们使用`--detail`标志查看 RAID 的详细信息，我们就可以看到这一点：

```
[nfs]# mdadm --detail /dev/md127
/dev/md127:
 Version : 1.0
 Creation Time : Wed Apr 15 09:39:22 2015
 Raid Level : raid1
 Array Size : 511936 (500.02 MiB 524.22 MB)
 Used Dev Size : 511936 (500.02 MiB 524.22 MB)
 Raid Devices : 2
 Total Devices : 2
 Persistence : Superblock is persistent

 Intent Bitmap : Internal

 Update Time : Mon May 11 16:47:32 2015
 State : clean, degraded, recovering 
 Active Devices : 1
Working Devices : 2
 Failed Devices : 0
 Spare Devices : 1

 Rebuild Status : 50% complete

 Name : localhost:boot
 UUID : 7adf0323:b0962394:387e6cd0:b2914469
 Events : 66

 Number   Major   Minor   RaidDevice State
 0       8        1        0      active sync   /dev/sda1
 1       8       17        1      spare rebuilding   /dev/sdb1

```

从前面的输出中，我们可以看到与之前示例的一些不同之处。一个非常重要的区别是`重建状态`：

```
Rebuild Status : 50% complete

```

通过`mdadm --detail`，我们可以看到驱动器重新同步的完成状态。如果在此过程中有任何错误，我们也将能够看到。如果我们看底部的三行，我们还可以看到哪些设备是活动的，哪些正在重建。

```
 Number   Major   Minor   RaidDevice State
 0       8        1        0      active sync   /dev/sda1
 1       8       17        1      spare rebuilding   /dev/sdb1

```

几秒钟后，如果我们再次运行`mdadm --detail`，我们应该看到 RAID 设备已经重新同步：

```
[nfs]# mdadm --detail /dev/md127
/dev/md127:
 Version : 1.0
 Creation Time : Wed Apr 15 09:39:22 2015
 Raid Level : raid1
 Array Size : 511936 (500.02 MiB 524.22 MB)
 Used Dev Size : 511936 (500.02 MiB 524.22 MB)
 Raid Devices : 2
 Total Devices : 2
 Persistence : Superblock is persistent

 Intent Bitmap : Internal

 Update Time : Mon May 11 16:47:32 2015
 State : clean 
 Active Devices : 2
Working Devices : 2
 Failed Devices : 0
 Spare Devices : 0

 Name : localhost:boot
 UUID : 7adf0323:b0962394:387e6cd0:b2914469
 Events : 69

 Number   Major   Minor   RaidDevice State
 0       8        1        0      active sync   /dev/sda1
 1       8       17        1      active sync   /dev/sdb1

```

现在我们可以看到两个驱动器都列为`active sync`状态，而 RAID 的`State`只是`clean`。

上述输出是一个正常的 RAID 1 设备应该看起来的样子。在这一点上，我们可以认为`/dev/md127`的问题已经解决。

## 添加新的磁盘设备

有时你会发现自己处于一个情况，你的磁盘驱动实际上是有故障的，实际的物理硬件必须被替换。在这种情况下，一旦重新创建分区`/dev/sdb1`和`/dev/sdb2`，设备可以简单地使用与之前相同的步骤添加到 RAID 中。

当执行命令`mdadm <raid device> -a <disk device>`时，`mdadm`首先检查磁盘设备是否曾经是 RAID 的一部分。

它通过读取磁盘设备上的超级块信息来执行此操作。如果设备以前曾是 RAID 的一部分，它会简单地重新添加并开始重建以重新同步驱动器。

如果磁盘设备以前从未参与过 RAID，它将被添加为备用设备，如果 RAID 处于降级状态，备用设备将被用来使 RAID 恢复到干净状态。

## 当磁盘没有被清洁添加时

在以前的工作环境中，当我们更换硬盘时，硬盘总是在用于替换生产环境中故障硬盘之前进行质量测试。通常，这种质量测试涉及创建分区并将这些分区添加到现有的 RAID 中。

因为这些设备已经在它们上面有一个 RAID 超级块，`mdadm`会拒绝将这些设备添加到 RAID 中。可以使用`mdadm`命令清除现有的 RAID`超级块`：

```
[nfs]# mdadm --zero-superblock /dev/sdb2

```

上述命令将告诉`mdadm`从指定的磁盘中删除 RAID`超级块`信息—在本例中是`/dev/sdb2`：

```
[nfs]# mdadm --examine /dev/sdb2
mdadm: No md superblock detected on /dev/sdb2.

```

使用`--examine`，我们可以看到现在设备上没有超级块。

`--zero-superblock`标志应谨慎使用，只有当设备数据不再需要时才使用。一旦删除了这些超级块信息，RAID 将把这个磁盘视为空白磁盘，在任何重新同步过程中，现有数据将被覆盖。

一旦超级块被移除，同样的步骤可以执行以将其添加到 RAID 阵列中：

```
[nfs]# mdadm /dev/md126 -a /dev/sdb2
mdadm: added /dev/sdb2

```

## 观察重建状态的另一种方法

之前我们使用`mdadm --detail`来显示`md127`的重建状态。另一种查看这些信息的方法是通过`/proc/mdstat`：

```
[nfs]# cat /proc/mdstat
Personalities : [raid1] 
md126 : active raid1 sdb2[2] sda2[0]
 7871488 blocks super 1.2 [2/1] [U_]
 [>....................]  recovery =  0.0% (1984/7871488) finish=65.5min speed=1984K/sec
 bitmap: 1/1 pages [4KB], 65536KB chunk

md127 : active raid1 sdb1[1] sda1[0]
 511936 blocks super 1.0 [2/2] [UU]
 bitmap: 0/1 pages [0KB], 65536KB chunk

unused devices: <none>

```

过一会儿，RAID 将完成重新同步；现在，两个 RAID 阵列都处于健康状态：

```
[nfs]# cat /proc/mdstat 
Personalities : [raid1] 
md126 : active raid1 sdb2[2] sda2[0]
 7871488 blocks super 1.2 [2/2] [UU]
 bitmap: 0/1 pages [0KB], 65536KB chunk

md127 : active raid1 sdb1[1] sda1[0]
 511936 blocks super 1.0 [2/2] [UU]
 bitmap: 0/1 pages [0KB], 65536KB chunk

unused devices: <none>

```

# 总结

在前一章中，第七章，*文件系统错误和恢复*，我们注意到在`/var/log/messages`日志文件中出现了一个简单的 RAID 故障消息。在本章中，我们使用了`数据收集器`方法来调查故障消息的原因。

在使用 RAID 管理命令`mdadm`进行调查后，我们发现了几个处于降级状态的 RAID 设备。使用`dmesg`，我们能够确定哪些硬盘设备受到影响，以及这些硬盘在某个时候被移出了服务。我们还发现硬盘的**事件计数**不匹配，阻止了硬盘的自动重新添加。

我们通过`dmesg`验证了设备没有物理故障，并选择将它们重新添加到 RAID 阵列中。

本章重点介绍了 RAID 和磁盘故障，但`/var/log/messages`和`dmesg`都可以用于排除其他设备故障。然而，对于除硬盘以外的设备，解决方案通常是简单的更换。当然，像大多数事情一样，这取决于所经历的故障类型。

在下一章中，我们将展示如何排除自定义用户应用程序的故障，并使用系统工具进行一些高级故障排除。
