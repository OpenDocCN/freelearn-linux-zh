# 第八章。介绍设备驱动程序

内核设备驱动程序是将底层硬件暴露给系统其余部分的机制。作为嵌入式系统的开发人员，您需要了解设备驱动程序如何适应整体架构以及如何从用户空间程序中访问它们。您的系统可能会有一些新颖的硬件部件，您将不得不找出一种访问它们的方法。在许多情况下，您会发现已经为您提供了设备驱动程序，您可以在不编写任何内核代码的情况下实现您想要的一切。例如，您可以使用`sysfs`中的文件来操作 GPIO 引脚和 LED，并且有库可以访问串行总线，包括 SPI 和 I2C。

有很多地方可以找到如何编写设备驱动程序的信息，但很少有地方告诉你为什么要这样做以及在这样做时的选择。这就是我想在这里介绍的内容。但是，请记住，这不是一本专门写内核设备驱动程序的书，这里提供的信息是为了帮助您在这个领域中导航，而不一定是为了在那里设置家。有很多好书和文章可以帮助您编写设备驱动程序，其中一些列在本章末尾。

# 设备驱动程序的作用

如第四章中所述，*移植和配置内核*，内核的功能之一是封装计算机系统的许多硬件接口，并以一致的方式呈现给用户空间程序。有设计的框架使得在内核中编写设备的接口逻辑变得容易，并且可以将其集成到内核中：这就是设备驱动程序，它是介于其上方的内核和其下方的硬件之间的代码片段。设备驱动程序是控制物理设备（如 UART 或 MMC 控制器）或虚拟设备（如空设备(`/dev/null`)或 ramdisk）的软件。一个驱动程序可以控制多个相同类型的设备。

内核设备驱动程序代码以高特权级别运行，就像内核的其余部分一样。它可以完全访问处理器地址空间和硬件寄存器。它可以处理中断和 DMA 传输。它可以利用复杂的内核基础设施进行同步和内存管理。这也有一个缺点，即如果有错误的驱动程序出现问题，它可能会导致系统崩溃。因此，有一个原则是设备驱动程序应尽可能简单，只提供信息给应用程序，真正的决策是在应用程序中做出的。你经常听到这被表达为*内核中没有策略*。

在 Linux 中，有三种主要类型的设备驱动程序：

+   **字符**：这是用于具有丰富功能范围和应用程序代码与驱动程序之间薄层的无缓冲 I/O。在实现自定义设备驱动程序时，这是首选。

+   **块**：这具有专门针对从大容量存储设备进行块 I/O 的接口。有一个厚的缓冲层，旨在使磁盘读取和写入尽可能快，这使其不适用于其他用途。

+   **网络**：这类似于块设备，但用于传输和接收网络数据包，而不是磁盘块。

还有第四种类型，它表现为伪文件系统中的一组文件。例如，您可以通过`/sys/class/gpio`中的一组文件访问 GPIO 驱动程序，我将在本章后面描述。让我们首先更详细地看一下三种基本设备类型。

# 字符设备

这些设备在用户空间通过文件名进行标识：如果你想从 UART 读取数据，你需要打开设备节点，例如，在 ARM Versatile Express 上的第一个串行端口将是`/dev/ttyAMA0`。驱动程序在内核中以不同的方式进行标识，使用的是主设备号，在给定的示例中是`204`。由于 UART 驱动程序可以处理多个 UART，还有第二个号码，称为次设备号，用于标识特定的接口，例如在这种情况下是 64。

```
# ls -l /dev/ttyAMA*

crw-rw----    1 root     root      204,  64 Jan  1  1970 /dev/ttyAMA0
crw-rw----    1 root     root      204,  65 Jan  1  1970 /dev/ttyAMA1
crw-rw----    1 root     root      204,  66 Jan  1  1970 /dev/ttyAMA2
crw-rw----    1 root     root      204,  67 Jan  1  1970 /dev/ttyAMA3
```

标准主设备号和次设备号的列表可以在内核文档中找到，位于`Documentation/devices.txt`中。该列表不经常更新，也不包括前面段落中描述的`ttyAMA`设备。然而，如果你查看`drivers/tty/serial/amba-pl011.c`中的源代码，你会看到主设备号和次设备号是如何声明的。

```
#define SERIAL_AMBA_MAJOR       204
#define SERIAL_AMBA_MINOR       64
```

当一个设备有多个实例时，设备节点的命名约定为`<基本名称><接口号>`，例如，`ttyAMA0`，`ttyAMA1`等。

正如我在第五章中提到的，*构建根文件系统*，设备节点可以通过多种方式创建：

+   `devtmpfs`：当设备驱动程序使用驱动程序提供的基本名称（`ttyAMA`）和实例号注册新的设备接口时创建的节点。

+   `udev`或`mdev`（没有`devtmpfs`）：与`devtmpfs`基本相同，只是需要一个用户空间守护程序从`sysfs`中提取设备名称并创建节点。我稍后会谈到`sysfs`。

+   `mknod`：如果你使用静态设备节点，可以使用`mknod`手动创建它们。

你可能会从上面我使用的数字中得到这样的印象，即主设备号和次设备号都是 8 位数字，范围在 0 到 255 之间。实际上，从 Linux 2.6 开始，主设备号有 12 位长，有效数字范围为 1 到 4095，次设备号有 20 位，范围为 0 到 1048575。

当你打开一个设备节点时，内核会检查主设备号和次设备号是否落在该类型设备驱动程序注册的范围内（字符或块）。如果是，它会将调用传递给驱动程序，否则打开调用失败。设备驱动程序可以提取次设备号以找出要使用的硬件接口。如果次设备号超出范围，它会返回错误。

要编写一个访问设备驱动程序的程序，你必须对其工作原理有一定了解。换句话说，设备驱动程序与文件不同：你对它所做的事情会改变设备的状态。一个简单的例子是伪随机数生成器`urandom`，每次读取它都会返回随机数据的字节。下面是一个执行此操作的程序：

```
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int main(void)
{
  int f;
  unsigned int rnd;
  int n;
  f = open("/dev/urandom", O_RDONLY);
  if (f < 0) {
    perror("Failed to open urandom");
    return 1;
  }
  n = read(f, &rnd, sizeof(rnd));
  if (n != sizeof(rnd)) {
    perror("Problem reading urandom");
    return 1;
  }
  printf("Random number = 0x%x\n", rnd);
  close(f);
  return 0;
}
```

Unix 驱动程序模型的好处在于，一旦我们知道有一个名为`urandom`的设备，并且每次从中读取数据时，它都会返回一组新的伪随机数据，我们就不需要再了解其他任何信息。我们可以直接使用诸如`open(2)`、`read(2)`和`close(2)`等普通函数。

我们可以使用流 I/O 函数`fopen(3)`、`fread(3)`和`fclose(3)`，但是这些函数隐含的缓冲区通常会导致意外的行为。例如，`fwrite(3)`通常只写入用户空间缓冲区，而不是设备。我们需要调用`fflush(3)`来强制刷新缓冲区。

### 提示

不要在调用设备驱动程序时使用流 I/O 函数，比如`fread(3)`和`fwrite(3)`。

# 块设备

块设备也与设备节点相关联，同样具有主设备号和次设备号。

### 提示

尽管字符设备和块设备使用主设备号和次设备号进行标识，但它们位于不同的命名空间。主设备号为 4 的字符驱动程序与主设备号为 4 的块驱动程序没有任何关联。

对于块设备，主编号用于标识设备驱动程序，次编号用于标识分区。让我们以 MMC 驱动程序为例：

```
# ls -l /dev/mmcblk*

brw-------    1 root root  179,   0 Jan  1  1970 /dev/mmcblk0
brw-------    1 root root  179,   1 Jan  1  1970 /dev/mmcblk0p1
brw-------    1 root root  179,   2 Jan  1  1970 /dev/mmcblk0p2
brw-------    1 root root  179,   8 Jan  1  1970 /dev/mmcblk1
brw-------    1 root root  179,   9 Jan  1  1970 /dev/mmcblk1p1
brw-------    1 root root  179,  10 Jan  1  1970 /dev/mmcblk1p2
```

主编号为 179（在`devices.txt`中查找！）。次编号用于标识不同的`mmc`设备和该设备上存储介质的分区。对于 mmcblk 驱动程序，每个设备有八个次编号范围：从 0 到 7 的次编号用于第一个设备，从 8 到 15 的次编号用于第二个设备，依此类推。在每个范围内，第一个次编号代表整个设备的原始扇区，其他次编号代表最多七个分区。

您可能已经了解到 SCSI 磁盘驱动程序，称为 sd，用于控制使用 SCSI 命令集的一系列磁盘，其中包括 SCSI、SATA、USB 大容量存储和 UFS（通用闪存存储）。它的主编号为 8，每个接口（或磁盘）有 16 个次编号。从 0 到 15 的次编号用于第一个接口，设备节点的名称为`sda`到`sda15`，从 16 到 31 的编号用于第二个磁盘，设备节点为`sdb`到`sdb15`，依此类推。这一直持续到第 16 个磁盘，从 240 到 255，节点名称为`sdp`。由于 SCSI 磁盘非常受欢迎，还有其他为它们保留的主编号，但我们不需要在这里担心这些。

分区是使用诸如`fdisk`、`sfidsk`或`parted`之类的实用程序创建的。一个例外是原始闪存：MTD 驱动程序的分区信息是内核命令行或设备树中的一部分，或者是第七章中描述的其他方法之一，*创建存储策略*。

用户空间程序可以通过设备节点直接打开和与块设备交互。这不是常见的操作，通常用于执行分区、格式化文件系统和挂载等管理操作。一旦文件系统被挂载，您将通过该文件系统中的文件间接与块设备交互。

# 网络设备

网络设备不是通过设备节点访问的，也没有主次编号。相反，内核会根据字符串和实例号为网络设备分配一个名称。以下是网络驱动程序注册接口的示例方式：

```
my_netdev = alloc_netdev(0, "net%d", NET_NAME_UNKNOWN, netdev_setup);
ret = register_netdev(my_netdev);
```

这将创建一个名为`net0`的网络设备，第一次调用时为`net1`，依此类推。更常见的名称是`lo`、`eth0`和`wlan0`。

请注意，这是它起始的名称；设备管理器（如`udev`）可能会在以后更改为其他名称。

通常，网络接口名称仅在使用诸如`ip`和`ifconfig`之类的实用程序配置网络以建立网络地址和路由时使用。此后，您通过打开套接字间接与网络驱动程序交互，并让网络层决定如何将它们路由到正确的接口。

但是，可以通过创建套接字并使用`include/linux/sockios.h`中列出的`ioctl`命令直接从用户空间访问网络设备。例如，此程序使用`SIOCGIFHWADDR`查询驱动程序的硬件（MAC）地址：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/sockios.h>
#include <net/if.h>
int main (int argc, char *argv[])
{
  int s;
  int ret;
  struct ifreq ifr;
  int i;
  if (argc != 2) {
    printf("Usage %s [network interface]\n", argv[0]);
    return 1;
  }
  s = socket(PF_INET, SOCK_DGRAM, 0);
  if (s < 0) {
    perror("socket");
    return 1;
  }
  strcpy(ifr.ifr_name, argv[1]);
  ret = ioctl(s, SIOCGIFHWADDR, &ifr);
  if (ret < 0) {
    perror("ioctl");
    return 1;
  }
  for (i = 0; i < 6; i++)
    printf("%02x:", (unsigned char)ifr.ifr_hwaddr.sa_data[i]);
  printf("\n");
  close(s);
  return 0;
}
```

这是一个标准设备`ioctl`，由网络层代表驱动程序处理，但是可以定义自己的`ioctl`编号并在自定义网络驱动程序中处理它们。

# 在运行时了解驱动程序

一旦您运行了 Linux 系统，了解加载的设备驱动程序及其状态是很有用的。您可以通过阅读`/proc`和`/sys`中的文件来了解很多信息。

首先，您可以通过读取`/proc/devices`来列出当前加载和活动的字符和块设备驱动程序：

```
# cat /proc/devices

Character devices:

  1 mem
  2 pty
  3 ttyp
  4 /dev/vc/0
  4 tty
  4 ttyS
  5 /dev/tty
  5 /dev/console
  5 /dev/ptmx
  7 vcs
 10 misc
 13 input
 29 fb
 81 video4linux
 89 i2c
 90 mtd
116 alsa
128 ptm
136 pts
153 spi
180 usb
189 usb_device
204 ttySC
204 ttyAMA
207 ttymxc
226 drm
239 ttyLP
240 ttyTHS
241 ttySiRF
242 ttyPS
243 ttyWMT
244 ttyAS
245 ttyO
246 ttyMSM
247 ttyAML
248 bsg
249 iio
250 watchdog
251 ptp
252 pps
253 media
254 rtc

Block devices:

259 blkext
  7 loop
  8 sd
 11 sr
 31 mtdblock
 65 sd
 66 sd
 67 sd
 68 sd
 69 sd
 70 sd
 71 sd
128 sd
129 sd
130 sd
131 sd
132 sd
133 sd
134 sd
135 sd
179 mmc
```

对于每个驱动程序，您可以看到主要编号和基本名称。但是，这并不能告诉您每个驱动程序连接到了多少设备。它只显示了`ttyAMA`，但并没有提示它连接了四个真实的 UART。我稍后会回到这一点，当我查看`sysfs`时。如果您正在使用诸如`mdev`、`udev`或`devtmpfs`之类的设备管理器，您可以通过查看`/dev`中的字符和块设备接口来列出它们。

您还可以使用`ifconfig`或`ip`列出网络接口：

```
# ip link show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00

2: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc pfifo_fast state DOWN mode DEFAULT qlen 1000
    link/ether 54:4a:16:bb:b7:03 brd ff:ff:ff:ff:ff:ff

3: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether aa:fb:7f:5e:a8:d5 brd ff:ff:ff:ff:ff:ff
```

您还可以使用众所周知的命令`lsusb`和`lspci`来查找连接到 USB 或 PCI 总线的设备。关于它们的信息在各自的手册和大量的在线指南中都有，所以我在这里不再详细描述它们。

真正有趣的信息在`sysfs`中，这是下一个主题。

## 从 sysfs 获取信息

您可以以一种迂腐的方式定义`sysfs`，即内核对象、属性和关系的表示。内核对象是一个目录，属性是一个文件，关系是从一个对象到另一个对象的符号链接。

从更实际的角度来看，自 Linux 设备驱动程序模型在 2.6 版本中引入以来，它将所有设备和驱动程序表示为内核对象。您可以通过查看`/sys`来看到系统的内核视图，如下所示：

```
# ls /sys

block  bus  class  dev  devices  firmware  fs  kernel  module  power
```

在发现有关设备和驱动程序的信息方面，我将查看三个目录：`devices`、`class`和`block`。

## 设备：/sys/devices

这是内核对自启动以来发现的设备及其相互连接的视图。它是按系统总线在顶层组织的，因此您看到的内容因系统而异。这是 Versatile Express 的 QEMU 仿真：

```
# ls
 /sys/devices
armv7_cortex_a9  platform      system
breakpoint       software      virtual
```

所有系统上都存在三个目录：

+   `系统`：这包含了系统核心的设备，包括 CPU 和时钟。

+   `虚拟`：这包含基于内存的设备。您将在`virtual/mem`中找到出现为`/dev/null`、`/dev/random`和`/dev/zero`的内存设备。您将在`virtual/net`中找到环回设备`lo`。

+   `平台`：这是一个通用术语，用于指代通过传统硬件总线连接的设备。这几乎可以是嵌入式设备上的任何东西。

其他设备出现在与实际系统总线对应的目录中。例如，PCI 根总线（如果有）显示为`pci0000:00`。

浏览这个层次结构相当困难，因为它需要对系统的拓扑结构有一定的了解，而且路径名变得相当长，很难记住。为了让生活变得更容易，`/sys/class`和`/sys/block`提供了设备的两种不同视图。

## 驱动程序：/sys/class

这是设备驱动程序的视图，按其类型呈现，换句话说，这是一种软件视图而不是硬件视图。每个子目录代表一个驱动程序类，并由驱动程序框架的一个组件实现。例如，UART 设备由`tty`层管理，您将在`/sys/class/tty`中找到它们。同样，您将在`/sys/class/net`中找到网络设备，在`/sys/class/input`中找到输入设备，如键盘、触摸屏和鼠标，依此类推。

每个子目录中都有一个符号链接，指向该类型设备的每个实例在`/sys/device`中的表示。 

举个具体的例子，让我们看一下`/sys/class/tty/ttyAMA0`：

```
# cd  /sys/class/tty/ttyAMA0/
# ls
close_delay      flags            line             uartclk
closing_wait     io_type          port             uevent
custom_divisor   iomem_base       power            xmit_fifo_size
dev              iomem_reg_shift  subsystem
device           irq              type
```

链接`设备`引用了设备的硬件节点，`子系统`指向`/sys/class/tty`。其他属性是设备的属性。有些属性是特定于 UART 的，比如`xmit_fifo_size`，而其他属性适用于许多类型的设备，比如中断号`irq`和设备号`dev`。一些属性文件是可写的，允许您在运行时调整驱动程序的参数。

`dev`属性特别有趣。如果您查看它的值，您会发现以下内容：

```
# cat /sys/class/tty/ttyAMA0/dev
204:64
```

这是设备的主要和次要编号。当驱动程序注册了这个接口时，就会创建这个属性，如果没有`devtmpfs`的帮助，`udev`和`mdev`就会从这个文件中读取这些信息。

## 块驱动程序：/sys/block

设备模型的另一个重要视图是块驱动程序视图，你可以在`/sys/block`中找到。每个块设备都有一个子目录。这个例子来自 BeagleBone Black：

```
# ls /sys/block/

loop0  loop4  mmcblk0       ram0   ram12  ram2  ram6
loop1  loop5  mmcblk1       ram1   ram13  ram3  ram7
loop2  loop6  mmcblk1boot0  ram10  ram14  ram4  ram8
loop3  loop7  mmcblk1boot1  ram11  ram15  ram5  ram9
```

如果你查看这块板上的 eMMC 芯片`mmcblk1`，你可以看到接口的属性和其中的分区：

```
# cd /sys/block/mmcblk1
# ls

alignment_offset   ext_range     mmcblk1p1  ro
bdi                force_ro      mmcblk1p2  size
capability         holders       power      slaves
dev                inflight      queue      stat
device             mmcblk1boot0  range      subsystem
discard_alignment  mmcblk1boot1  removable  uevent
```

因此，通过阅读`sysfs`，你可以了解系统上存在的设备（硬件）和驱动程序（软件）。

# 寻找合适的设备驱动程序

典型的嵌入式板是基于制造商的参考设计，经过更改以适合特定应用。它可能通过 I2C 连接温度传感器，通过 GPIO 引脚连接灯和按钮，通过外部以太网 MAC 连接，通过 MIPI 接口连接显示面板，或者其他许多东西。你的工作是创建一个自定义内核来控制所有这些，那么你从哪里开始呢？

有些东西非常简单，你可以编写用户空间代码来处理它们。通过 I2C 或 SPI 连接的 GPIO 和简单外围设备很容易从用户空间控制，我稍后会解释。

其他东西需要内核驱动程序，因此你需要知道如何找到一个并将其整合到你的构建中。没有简单的答案，但这里有一些地方可以找到。

最明显的地方是制造商网站上的驱动程序支持页面，或者你可以直接问他们。根据我的经验，这很少能得到你想要的结果；硬件制造商通常不太懂 Linux，他们经常给出误导性的信息。他们可能有二进制的专有驱动程序，也可能有源代码，但是适用于与你拥有的内核版本不同的版本。所以，尽管可以尝试这种途径。我总是会尽力寻找适合手头任务的开源驱动程序。

你的内核可能已经支持：主线 Linux 中有成千上万的驱动程序，供应商内核中也有许多特定于供应商的驱动程序。首先运行`make menuconfig`（或`xconfig`），搜索产品名称或编号。如果找不到完全匹配的，尝试更通用的搜索，考虑到大多数驱动程序处理同一系列产品。接下来，尝试在驱动程序目录中搜索代码（这里用`grep`）。始终确保你正在运行适合你的板的最新内核：较新的内核通常有更多的设备驱动程序。

如果你还没有驱动程序，可以尝试在线搜索并在相关论坛上询问，看看是否有适用于不同 Linux 版本的驱动程序。如果找到了，你就需要将其移植到你的内核中。如果内核版本相似，可能会很容易，但如果相隔 12 到 18 个月以上，接口很可能已经发生了变化，你将不得不重写驱动程序的一部分，以使其与你的内核集成。你可能需要外包这项工作。如果所有上述方法都失败了，你就得自己找解决方案。

# 用户空间的设备驱动程序

在你开始编写设备驱动程序之前，暂停一下，考虑一下是否真的有必要。对于许多常见类型的设备，有通用的设备驱动程序，允许你直接从用户空间与硬件交互，而不必编写一行内核代码。用户空间代码肯定更容易编写和调试。它也不受 GPL 的限制，尽管我不认为这本身是一个好理由。

它们可以分为两大类：通过`sysfs`中的文件进行控制的设备，包括 GPIO 和 LED，以及通过设备节点公开通用接口的串行总线，比如 I2C。

## GPIO

**通用输入/输出**（**GPIO**）是数字接口的最简单形式，因为它可以直接访问单个硬件引脚，每个引脚可以配置为输入或输出。 GPIO 甚至可以用于通过在软件中操作每个位来创建更高级的接口，例如 I2C 或 SPI，这种技术称为位操作。主要限制是软件循环的速度和准确性以及您想要为它们分配的 CPU 周期数。一般来说，使用`CONFIG_PREEMPT`编译的内核很难实现比毫秒更好的定时器精度，使用`RT_PREEMPT`编译的内核很难实现比 100 微秒更好的定时器精度，我们将在第十四章中看到，*实时编程*。 GPIO 的更常见用途是读取按钮和数字传感器以及控制 LED、电机和继电器。

大多数 SoC 有很多 GPIO 位，这些位被分组在 GPIO 寄存器中，通常每个寄存器有 32 位。芯片上的 GPIO 位通过多路复用器（称为引脚复用器）路由到芯片封装上的 GPIO 引脚，我稍后会描述。在电源管理芯片和专用 GPIO 扩展器中可能有额外的 GPIO 位，通过 I2C 或 SPI 总线连接。所有这些多样性都由一个名为`gpiolib`的内核子系统处理，它实际上不是一个库，而是 GPIO 驱动程序用来以一致的方式公开 IO 的基础设施。

有关`gpiolib`实现的详细信息在内核源中的`Documentation/gpio`中，驱动程序本身在`drivers/gpio`中。

应用程序可以通过`/sys/class/gpio`目录中的文件与`gpiolib`进行交互。以下是在典型嵌入式板（BeagleBone Black）上看到的内容的示例：

```
# ls  /sys/class/gpio
export  gpiochip0   gpiochip32  gpiochip64  gpiochip96  unexport
```

`gpiochip0`到`gpiochip96`目录代表了四个 GPIO 寄存器，每个寄存器有 32 个 GPIO 位。如果你查看其中一个`gpiochip`目录，你会看到以下内容：

```
# ls /sys/class/gpio/gpiochip96/
base  label   ngpio  power  subsystem  uevent
```

文件`base`包含寄存器中第一个 GPIO 引脚的编号，`ngpio`包含寄存器中位的数量。在这种情况下，`gpiochip96/base`是 96，`gpiochip96/ngpio`是 32，这告诉您它包含 GPIO 位 96 到 127。寄存器中最后一个 GPIO 和下一个寄存器中第一个 GPIO 之间可能存在间隙。

要从用户空间控制 GPIO 位，您首先必须从内核空间导出它，方法是将 GPIO 编号写入`/sys/class/gpio/export`。此示例显示了 GPIO 48 的过程：

```
# echo 48 > /sys/class/gpio/export
# ls /sys/class/gpio
export      gpio48    gpiochip0   gpiochip32  gpiochip64  gpiochip96  unexport
```

现在有一个新目录`gpio48`，其中包含了控制引脚所需的文件。请注意，如果 GPIO 位已被内核占用，您将无法以这种方式导出它。

目录`gpio48`包含这些文件：

```
# ls /sys/class/gpio/gpio48
active_low  direction  edge  power  subsystem   uevent  value
```

引脚最初是输入的。要将其更改为输出，请将`out`写入`direction`文件。文件`value`包含引脚的当前状态，低电平为 0，高电平为 1。如果它是输出，您可以通过向`value`写入 0 或 1 来更改状态。有时，在硬件中低电平和高电平的含义是相反的（硬件工程师喜欢做这种事情），因此将 1 写入`active_low`会反转含义，以便在`value`中将低电压报告为 1，高电压为 0。

您可以通过将 GPIO 编号写入`/sys/class/gpio/unexport`来从用户空间控制中删除 GPIO。

### 从 GPIO 处理中断

在许多情况下，可以将 GPIO 输入配置为在状态更改时生成中断，这允许您等待中断而不是在低效的软件循环中轮询。如果 GPIO 位可以生成中断，则文件`edge`存在。最初，它的值为`none`，表示它不会生成中断。要启用中断，您可以将其设置为以下值之一：

+   **rising**：上升沿中断

+   **falling**：下降沿中断

+   **both**：上升沿和下降沿中断

+   **none**：无中断（默认）

您可以使用`poll（）`函数等待中断，事件为`POLLPRI`。如果要等待 GPIO 48 上的上升沿，首先要启用中断：

```
# echo 48 > /sys/class/gpio/export
# echo rising > /sys/class/gpio/gpio48/edge
```

然后，您可以使用`poll（）`等待更改，如此代码示例所示：

```
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <poll.h>

int main (int argc, char *argv[])
{
  int f;
  struct pollfd poll_fds [1];
  int ret;
  char value[4];
  int n;
  f = open("/sys/class/gpio/gpio48", O_RDONLY);
  if (f == -1) {
    perror("Can't open gpio48");
    return 1;
  }
  poll_fds[0].fd = f;
  poll_fds[0].events = POLLPRI | POLLERR;
  while (1) {
    printf("Waiting\n");
    ret = poll(poll_fds, 1, -1);
    if (ret > 0) {
        n = read(f, &value, sizeof(value));
        printf("Button pressed: read %d bytes, value=%c\n",
        n, value[0]);
    }
  }
  return 0;
}
```

## LED

LED 通常是通过 GPIO 引脚控制的，但是还有另一个内核子系统，提供了更专门的控制，用于特定目的。 `leds`内核子系统增加了设置亮度的功能，如果 LED 具有该功能，并且可以处理连接方式不同于简单 GPIO 引脚的 LED。它可以配置为在事件上触发 LED，例如块设备访问或只是心跳以显示设备正在工作。在`Documentation/leds/`中有更多信息，驱动程序位于`drivers/leds/`中。

与 GPIO 一样，LED 通过`sysfs`中的接口进行控制，在`/sys/class/leds`中。LED 的名称采用`devicename:colour:function`的形式，如下所示：

```
# ls /sys/class/leds
beaglebone:green:heartbeat  beaglebone:green:usr2
beaglebone:green:mmc0       beaglebone:green:usr3
```

这显示了一个单独的 LED：

```
# ls /sys/class/leds/beaglebone:green:usr2
brightness    max_brightness  subsystem     uevent
device        power           trigger
```

`brightness`文件控制 LED 的亮度，可以是 0（关闭）到`max_brightness`（完全打开）之间的数字。如果 LED 不支持中间亮度，则任何非零值都会打开它，零会关闭它。文件`trigger`列出了触发 LED 打开的事件。触发器列表因实现而异。这是一个例子：

```
# cat /sys/class/leds/beaglebone:green:heartbeat/trigger
none mmc0 mmc1 timer oneshot [heartbeat] backlight gpio cpu0 default-on
```

当前选择的触发器显示在方括号中。您可以通过将其他触发器之一写入文件来更改它。如果您想完全通过“亮度”控制 LED，请选择`none`。如果将触发器设置为`timer`，则会出现两个额外的文件，允许您以毫秒为单位设置开启和关闭时间：

```
# echo timer > /sys/class/leds/beaglebone:green:heartbeat/trigger
# ls /sys/class/leds/beaglebone:green:heartbeat
brightness  delay_on    max_brightness  subsystem   uevent
delay_off   device      power           trigger
# cat /sys/class/leds/beaglebone:green:heartbeat/delay_on
500
# cat /sys/class/leds/beaglebone:green:heartbeat/delay_off
500
#
```

如果 LED 具有片上定时器硬件，则闪烁会在不中断 CPU 的情况下进行。

## I2C

I2C 是一种简单的低速 2 线总线，通常用于访问 SoC 板上没有的外围设备，例如显示控制器、摄像头传感器、GPIO 扩展器等。还有一个相关的标准称为 SMBus（系统管理总线），它在 PC 上发现，用于访问温度和电压传感器。SMBus 是 I2C 的子集。

I2C 是一种主从协议，主要是 SoC 上的一个或多个主控制器。从设备由制造商分配的 7 位地址 - 请阅读数据表 - 允许每个总线上最多 128 个节点，但保留了 16 个，因此实际上只允许 112 个节点。总线速度为标准模式下的 100 KHz，或者快速模式下的最高 400 KHz。该协议允许主设备和从设备之间的读写事务最多达 32 个字节。通常，第一个字节用于指定外围设备上的寄存器，其余字节是从该寄存器读取或写入的数据。

每个主控制器都有一个设备节点，例如，这个 SoC 有四个：

```
# ls -l /dev/i2c*
crw-rw---- 1 root i2c 89, 0 Jan  1 00:18 /dev/i2c-0
crw-rw---- 1 root i2c 89, 1 Jan  1 00:18 /dev/i2c-1
crw-rw---- 1 root i2c 89, 2 Jan  1 00:18 /dev/i2c-2
crw-rw---- 1 root i2c 89, 3 Jan  1 00:18 /dev/i2c-3
```

设备接口提供了一系列`ioctl`命令，用于查询主控制器并向 I2C 从设备发送`read`和`write`命令。有一个名为`i2c-tools`的软件包，它使用此接口提供基本的命令行工具来与 I2C 设备交互。工具如下：

+   `i2cdetect`：这会列出 I2C 适配器并探测总线

+   `i2cdump`：这会从 I2C 外设的所有寄存器中转储数据

+   `i2cget`：这会从 I2C 从设备读取数据

+   `i2cset`：这将数据写入 I2C 从设备

`i2c-tools`软件包在 Buildroot 和 Yocto Project 中可用，以及大多数主流发行版。只要您知道从设备的地址和协议，编写一个用户空间程序来与设备通信就很简单： 

```
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <i2c-dev.h>
#include <sys/ioctl.h>
#define I2C_ADDRESS 0x5d
#define CHIP_REVISION_REG 0x10

void main (void)
{
  int f_i2c;
  int val;

  /* Open the adapter and set the address of the I2C device */
  f_i2c = open ("/dev/i2c-1", O_RDWR);
  ioctl (f_i2c, I2C_SLAVE, I2C_ADDRESS);

  /* Read 16-bits of data from a register */
  val = i2c_smbus_read_word_data(f, CHIP_REVISION_REG);
  printf ("Sensor chip revision %d\n", val);
  close (f_i2c);
}
```

请注意，标头`i2c-dev.h`是来自`i2c-tools`软件包的标头，而不是来自 Linux 内核标头的标头。 `i2c_smbus_read_word_data（）`函数是在`i2c-dev.h`中内联编写的。

有关 I2C 在`Documentation/i2c/dev-interface`中的 Linux 实现的更多信息。主控制器驱动程序位于`drivers/i2c/busses`中。

## SPI

串行外围接口总线类似于 I2C，但速度更快，高达低 MHz。该接口使用四根线，具有独立的发送和接收线，这使得它可以全双工操作。总线上的每个芯片都使用专用的芯片选择线进行选择。它通常用于连接触摸屏传感器、显示控制器和串行 NOR 闪存设备。

与 I2C 一样，它是一种主从协议，大多数 SoC 实现了一个或多个主机控制器。有一个通用的 SPI 设备驱动程序，您可以通过内核配置`CONFIG_SPI_SPIDEV`启用它。它为每个 SPI 控制器创建一个设备节点，允许您从用户空间访问 SPI 芯片。设备节点的名称为`spidev[bus].[chip select]`。

```
# ls -l /dev/spi*
crw-rw---- 1 root root 153, 0 Jan  1 00:29 /dev/spidev1.0
```

有关使用`spidev`接口的示例，请参考`Documentation/spi`中的示例代码。

# 编写内核设备驱动程序

最终，当您耗尽了上述所有用户空间选项时，您会发现自己不得不编写一个设备驱动程序来访问连接到您的设备的硬件。虽然现在不是深入细节的时候，但值得考虑一下选择。字符驱动程序是最灵活的，应该可以满足 90%的需求；如果您正在使用网络接口，网络设备也适用；块设备用于大容量存储。编写内核驱动程序的任务是复杂的，超出了本书的范围。在本节末尾有一些参考资料，可以帮助您一路前行。在本节中，我想概述与驱动程序交互的可用选项——这通常不是涵盖的主题——并向您展示驱动程序的基本结构。

## 设计字符设备接口

主要的字符设备接口基于字节流，就像串口一样。然而，许多设备并不符合这个描述：例如，机器人手臂的控制器需要移动和旋转每个关节的功能。幸运的是，与设备驱动程序进行通信的其他方法不仅仅是`read(2)`和`write(2)`。

+   `ioctl`：`ioctl`函数允许您向驱动程序传递两个参数，这两个参数可以有任何您喜欢的含义。按照惯例，第一个参数是一个命令，用于选择驱动程序中的几个函数中的一个，第二个参数是一个指向结构体的指针，该结构体用作输入和输出参数的容器。这是一个空白画布，允许您设计任何您喜欢的程序接口，当驱动程序和应用程序紧密链接并由同一团队编写时，这是非常常见的。然而，在内核中，`ioctl`已经被弃用，您会发现很难让任何具有新`ioctl`用法的驱动程序被上游接受。内核维护人员不喜欢`ioctl`，因为它使内核代码和应用程序代码过于相互依赖，并且很难在内核版本和架构之间保持两者同步。

+   `sysfs`：这是现在的首选方式，一个很好的例子是之前描述的 GPIO 接口。其优点是它是自我记录的，只要您为文件选择描述性名称。它也是可脚本化的，因为文件内容是 ASCII 字符串。另一方面，每个文件要求包含一个单一值，这使得如果您需要同时更改多个值，就很难实现原子性。例如，如果您想设置两个值然后启动一个操作，您需要写入三个文件：两个用于输入，第三个用于触发操作。即使这样，也不能保证其他两个文件没有被其他人更改。相反，`ioctl`通过单个函数调用中的结构传递所有参数。

+   `mmap`：您可以通过将内核内存映射到用户空间来直接访问内核缓冲区和硬件寄存器，绕过内核。您可能仍然需要一些内核代码来处理中断和 DMA。有一个封装这个想法的子系统，称为`uio`，即用户 I/O。在`Documentation/DocBook/uio-howto`中有更多文档，`drivers/uio`中有示例驱动程序。

+   `sigio`：您可以使用内核函数`kill_fasync()`从驱动程序发送信号，以通知应用程序事件，例如输入准备就绪或接收到中断。按照惯例，使用信号 SIGIO，但它可以是任何人。您可以在 UIO 驱动程序`drivers/uio/uio.c`和 RTC 驱动程序`drivers/char/rtc.c`中看到一些示例。主要问题是编写可靠的信号处理程序很困难，因此它仍然是一个很少使用的设施。

+   `debugfs`：这是另一个伪文件系统，它将内核数据表示为文件和目录，类似于`proc`和`sysfs`。主要区别在于`debugfs`不得包含系统正常操作所需的信息；它仅用于调试和跟踪信息。它被挂载为`mount -t debugfs debug /sys/kernel/debug`。

内核文档中有关`debugfs`的良好描述，`Documentation/filesystems/debugfs.txt`。

+   `proc`：`proc`文件系统已被弃用，除非它与进程有关，这是文件系统的最初预期目的。但是，您可以使用`proc`发布您选择的任何信息。并且，与`sysfs`和`debugfs`不同，它可用于非 GPL 模块。

+   `netlink`：这是一个套接字协议族。`AF_NETLINK`创建一个将内核空间链接到用户空间的套接字。最初创建它是为了使网络工具能够与 Linux 网络代码通信，以访问路由表和其他详细信息。udev 也使用它将事件从内核传递给 udev 守护程序。一般设备驱动程序中很少使用它。

内核源代码中有许多先前文件系统的示例，您可以为驱动程序代码设计非常有趣的接口。唯一的普遍规则是*最少惊讶原则*。换句话说，使用您的驱动程序的应用程序编写人员应该发现一切都以逻辑方式工作，没有怪癖或奇怪之处。

## 设备驱动程序的解剖

现在是时候通过查看简单设备驱动程序的代码来汇总一些线索了。

提供了名为`dummy`的设备驱动程序的源代码，该驱动程序创建了四个通过`/dev/dummy0`到`/dev/dummy3`访问的设备。这是驱动程序的完整代码：

```
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/device.h>
#define DEVICE_NAME "dummy"
#define MAJOR_NUM 42
#define NUM_DEVICES 4

static struct class *dummy_class;
static int dummy_open(struct inode *inode, struct file *file)
{
  pr_info("%s\n", __func__);
  return 0;
}

static int dummy_release(struct inode *inode, struct file *file)
{
  pr_info("%s\n", __func__);
  return 0;
}

static ssize_t dummy_read(struct file *file,
  char *buffer, size_t length, loff_t * offset)
{
  pr_info("%s %u\n", __func__, length);
  return 0;
}

static ssize_t dummy_write(struct file *file,
  const char *buffer, size_t length, loff_t * offset)
{
  pr_info("%s %u\n", __func__, length);
  return length;
}

struct file_operations dummy_fops = {
  .owner = THIS_MODULE,
  .open = dummy_open,
  .release = dummy_release,
  .read = dummy_read,
  .write = dummy_write,
};

int __init dummy_init(void)
{
  int ret;
  int i;
  printk("Dummy loaded\n");
  ret = register_chrdev(MAJOR_NUM, DEVICE_NAME, &dummy_fops);
  if (ret != 0)
    return ret;
  dummy_class = class_create(THIS_MODULE, DEVICE_NAME);
  for (i = 0; i < NUM_DEVICES; i++) {
    device_create(dummy_class, NULL,
    MKDEV(MAJOR_NUM, i), NULL, "dummy%d", i);
  }
  return 0;
}

void __exit dummy_exit(void)
{
  int i;
  for (i = 0; i < NUM_DEVICES; i++) {
    device_destroy(dummy_class, MKDEV(MAJOR_NUM, i));
  }
  class_destroy(dummy_class);
  unregister_chrdev(MAJOR_NUM, DEVICE_NAME);
  printk("Dummy unloaded\n");
}

module_init(dummy_init);
module_exit(dummy_exit);
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Chris Simmonds");
MODULE_DESCRIPTION("A dummy driver");
```

代码末尾的宏`module_init`和`module_exit`指定了在加载和卸载模块时要调用的函数。其他三个添加了有关模块的一些基本信息，可以使用`modinfo`命令从编译的内核模块中检索。

模块加载时，将调用`dummy_init()`函数。

调用`register_chrdev`可以看到它何时成为一个字符设备，传递一个指向包含驱动程序实现的四个函数指针的`struct file_operations`指针。虽然`register_chrdev`告诉内核有一个主编号为 42 的驱动程序，但它并没有说明驱动程序的类型，因此它不会在`/sys/class`中创建条目。没有在`/sys/class`中的条目，设备管理器无法创建设备节点。因此，代码的下几行创建了一个设备类`dummy`，以及该类的四个名为`dummy0`到`dummy3`的设备。结果是`/sys/class/dummy`目录，其中包含`dummy0`到`dummy3`子目录，每个子目录中都包含一个名为`dev`的文件，其中包含设备的主要和次要编号。这就是设备管理器创建设备节点`/dev/dummy0`到`/dev/dummy3`所需的全部内容。

`exit`函数必须释放`init`函数声明的资源，这里指的是释放设备类和主要编号。

该驱动程序的文件操作由`dummy_open()`，`dummy_read()`，`dummy_write()`和`dummy_release()`实现，并在用户空间程序调用`open(2)`，`read(2)`，`write(2)`和`close(2)`时调用。 它们只是打印内核消息，以便您可以看到它们被调用。 您可以使用`echo`命令从命令行演示这一点：

```
# echo hello > /dev/dummy0

[ 6479.741192] dummy_open
[ 6479.742505] dummy_write 6
[ 6479.743008] dummy_release
```

在这种情况下，消息出现是因为我已登录到控制台，默认情况下内核消息会打印到控制台。

该驱动程序的完整源代码不到 100 行，但足以说明设备节点和驱动程序代码之间的链接方式，说明设备类是如何创建的，允许设备管理器在加载驱动程序时自动创建设备节点，以及数据如何在用户空间和内核空间之间移动。 接下来，您需要构建它。

### 编译和加载

此时，您有一些驱动程序代码，希望在目标系统上进行编译和测试。 您可以将其复制到内核源树中并修改 makefile 以构建它，或者您可以将其编译为树外模块。 让我们首先从树外构建开始。

您需要一个简单的 makefile，该 makefile 使用内核构建系统来完成艰苦的工作：

```
LINUXDIR := $(HOME)/MELP/build/linux

obj-m := dummy.o
all:
        make ARCH=arm CROSS_COMPILE=arm-cortex_a8-linux-gnueabihf- \
          -C $(LINUXDIR) M=$(shell pwd)
clean:
        make -C $(LINUXDIR) M=$(shell pwd) clean
```

将`LINUXDIR`设置为您将在目标设备上运行模块的内核目录。 代码`obj-m：= dummy.o`将调用内核构建规则，以获取源文件`dummy.c`并创建内核模块`dummy.ko`。 请注意，内核模块在内核发布和配置之间不具有二进制兼容性，该模块只能在其编译的内核上加载。

构建的最终结果是内核`dummy.ko`，您可以将其复制到目标并按照下一节中所示加载。

如果要在内核源树中构建驱动程序，该过程非常简单。 选择适合您的驱动程序类型的目录。 该驱动程序是基本字符设备，因此我将`dummy.c`放在`drivers/char`中。 然后，编辑该目录中的 makefile，并添加一行以无条件地构建驱动程序作为模块，如下所示：

```
obj-m  += dummy.o
```

或者将以下行添加到无条件构建为内置：

```
obj-y   += dummy.o
```

如果要使驱动程序可选，可以在`Kconfig`文件中添加菜单选项，并根据配置选项进行条件编译，就像我在第四章中描述的那样，*移植和配置内核*，描述内核配置时。

# 加载内核模块

您可以使用简单的`insmod`，`lsmod`和`rmmod`命令加载，卸载和列出模块。 这里显示了加载虚拟驱动程序：

```
# insmod /lib/modules/4.1.10/kernel/drivers/dummy.ko
# lsmod
dummy 1248 0 - Live 0xbf009000 (O)
# rmmod dummy
```

如果模块放置在`/lib/modules/<kernel release>`中的子目录中，例如示例中，可以使用`depmod`命令创建模块依赖数据库：

```
# depmod -a
# ls /lib/modules/4.1.10/
kernel               modules.builtin.bin  modules.order
modules.alias        modules.dep          modules.softdep
modules.alias.bin    modules.dep.bin      modules.symbols
modules.builtin      modules.devname      modules.symbols.bin
```

`module.*`文件中的信息由`modprobe`命令使用，以按名称而不是完整路径定位模块。 `modprobe`还具有许多其他功能，这些功能在手册中有描述。

模块依赖信息也被设备管理器使用，特别是`udev`。 例如，当检测到新硬件时，例如新的 USB 设备，`udevd`守护程序会被警报，并从硬件中读取供应商和产品 ID。 `udevd`扫描模块依赖文件，寻找已注册这些 ID 的模块。 如果找到一个，它将使用`modprobe`加载。

# 发现硬件配置

虚拟驱动程序演示了设备驱动程序的结构，但它缺乏与真实硬件的交互，因为它只操作内存结构。 设备驱动程序通常用于与硬件交互，其中的一部分是能够首先发现硬件，要记住的是在不同配置中它可能位于不同的地址。

在某些情况下，硬件本身提供信息。可发现总线上的设备（如 PCI 或 USB）具有查询模式，该模式返回资源需求和唯一标识符。内核将标识符和可能的其他特征与设备驱动程序进行匹配，并将它们配对。

然而，大多数 SoC 上的硬件块都没有这样的标识符。您必须以设备树或称为平台数据的 C 结构的形式提供信息。

在 Linux 的标准驱动程序模型中，设备驱动程序会向适当的子系统注册自己：PCI、USB、开放固件（设备树）、平台设备等。注册包括标识符和称为探测函数的回调函数，如果硬件的 ID 与驱动程序的 ID 匹配，则会调用该函数。对于 PCI 和 USB，ID 基于设备的供应商和产品 ID，对于设备树和平台设备，它是一个名称（ASCII 字符串）。

## 设备树

我在第三章中向您介绍了设备树，*关于引导程序的一切*。在这里，我想向您展示 Linux 设备驱动程序如何与这些信息连接。

作为示例，我将使用 ARM Versatile 板，`arch/arm/boot/dts/versatile-ab.dts`，其中以太网适配器在此处定义：

```
net@10010000 {
  compatible = "smsc,lan91c111";
  reg = <0x10010000 0x10000>;
  interrupts = <25>;
};
```

## 平台数据

在没有设备树支持的情况下，还有一种使用 C 结构描述硬件的备用方法，称为平台数据。

每个硬件都由`struct platform_device`描述，其中包含名称和资源数组的指针。资源的类型由标志确定，其中包括以下内容：

+   `IORESOURCE_MEM`：内存区域的物理地址

+   `IORESOURCE_IO`：IO 寄存器的物理地址或端口号

+   `IORESOURCE_IRQ`：中断号

以下是从`arch/arm/mach-versatile/core.c`中获取的以太网控制器的平台数据示例，已经编辑以提高清晰度：

```
#define VERSATILE_ETH_BASE     0x10010000
#define IRQ_ETH                25
static struct resource smc91x_resources[] = {
  [0] = {
    .start          = VERSATILE_ETH_BASE,
    .end            = VERSATILE_ETH_BASE + SZ_64K - 1,
    .flags          = IORESOURCE_MEM,
  },
  [1] = {
    .start          = IRQ_ETH,
    .end            = IRQ_ETH,
    .flags          = IORESOURCE_IRQ,
  },
};
static struct platform_device smc91x_device = {
  .name           = "smc91x",
  .id             = 0,
  .num_resources  = ARRAY_SIZE(smc91x_resources),
  .resource       = smc91x_resources,
};
```

它有一个 64 KiB 的内存区域和一个中断。平台数据必须在初始化板时向内核注册：

```
void __init versatile_init(void)
{
  platform_device_register(&versatile_flash_device);
  platform_device_register(&versatile_i2c_device);
  platform_device_register(&smc91x_device);
  [ ...]
```

## 将硬件与设备驱动程序连接起来

在前面的部分中，您已经看到了以设备树和平台数据描述以太网适配器的方式。相应的驱动程序代码位于`drivers/net/ethernet/smsc/smc91x.c`中，它可以与设备树和平台数据一起使用。以下是初始化代码，再次编辑以提高清晰度：

```
static const struct of_device_id smc91x_match[] = {
  { .compatible = "smsc,lan91c94", },
  { .compatible = "smsc,lan91c111", },
  {},
};
MODULE_DEVICE_TABLE(of, smc91x_match);
static struct platform_driver smc_driver = {
  .probe          = smc_drv_probe,
  .remove         = smc_drv_remove,
  .driver         = {
    .name   = "smc91x",
    .of_match_table = of_match_ptr(smc91x_match),
  },
};
static int __init smc_driver_init(void)
{
  return platform_driver_register(&smc_driver);
}
static void __exit smc_driver_exit(void) \
{
  platform_driver_unregister(&smc_driver);
}
module_init(smc_driver_init);
module_exit(smc_driver_exit);
```

当驱动程序初始化时，它调用`platform_driver_register()`，指向`struct platform_driver`，其中包含对探测函数的回调，驱动程序名称`smc91x`，以及对`struct of_device_id`的指针。

如果此驱动程序已由设备树配置，内核将在设备树节点中的`compatible`属性和兼容结构元素指向的字符串之间寻找匹配项。对于每个匹配项，它都会调用`probe`函数。

另一方面，如果通过平台数据配置，`probe`函数将针对`driver.name`指向的每个匹配项进行调用。

`probe`函数提取有关接口的信息：

```
static int smc_drv_probe(struct platform_device *pdev)
{
  struct smc91x_platdata *pd = dev_get_platdata(&pdev->dev);
  const struct of_device_id *match = NULL;
  struct resource *res, *ires;
  int irq;

  res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
  ires = platform_get_resource(pdev, IORESOURCE_IRQ, 0);
  [...]
  addr = ioremap(res->start, SMC_IO_EXTENT);
  irq = ires->start;
  [...]
}
```

调用`platform_get_resource()`从设备树或平台数据中提取内存和`irq`信息。驱动程序负责映射内存并安装中断处理程序。第三个参数在前面两种情况下都是零，如果有多个特定类型的资源，则会起作用。

设备树允许您配置的不仅仅是基本内存范围和中断。在`probe`函数中有一段代码，用于从设备树中提取可选参数。在这个片段中，它获取了`register-io-width`属性：

```
match = of_match_device(of_match_ptr(smc91x_match), &pdev->dev);
if (match) {
  struct device_node *np = pdev->dev.of_node;
  u32 val;
  [...]
  of_property_read_u32(np, "reg-io-width", &val);
  [...]
}
```

对于大多数驱动程序，特定的绑定都记录在`Documentation/devicetree/bindings`中。对于这个特定的驱动程序，信息在`Documentation/devicetree/bindings/net/smsc911x.txt`中。

这里要记住的主要事情是，驱动程序应该注册一个`probe`函数和足够的信息，以便内核在找到与其了解的硬件匹配时调用`probe`。设备树描述的硬件与设备驱动程序之间的链接是通过`compatible`属性实现的。平台数据与驱动程序之间的链接是通过名称实现的。

# 额外阅读

以下资源提供了关于本章介绍的主题的更多信息：

+   *Linux Device Drivers, 4th edition*，作者*Jessica McKellar*，*Alessandro Rubini*，*Jonathan Corbet*和*Greg Kroah-Hartman*。在撰写本文时尚未出版，但如果它像前作一样好，那将是一个不错的选择。但是，第三版已经过时，不建议阅读。

+   *Linux Kernel Development, 3rd edition*，作者*Robert Love*，*Addison-Wesley* Professional; (July 2, 2010) ISBN-10: 0672329468

+   *Linux Weekly News*，[www.lwn.net](https://www.lwn.net)。

# 摘要

设备驱动程序的工作是处理设备，通常是物理硬件，但有时也是虚拟接口，并以一种一致和有用的方式呈现给更高级别。Linux 设备驱动程序分为三大类：字符、块和网络。在这三种中，字符驱动程序接口是最灵活的，因此也是最常见的。Linux 驱动程序适用于一个称为驱动模型的框架，通过`sysfs`公开。几乎所有设备和驱动程序的状态都可以在`/sys`中看到。

每个嵌入式系统都有自己独特的硬件接口和要求。Linux 为大多数标准接口提供了驱动程序，通过选择正确的内核配置，您可以使设备非常快速地运行起来。这样，您就可以处理非标准组件，需要添加自己的设备支持。

在某些情况下，您可以通过使用通用的 GPIO、I2C 等驱动程序并编写用户空间代码来避开问题。我建议这作为一个起点，因为这样可以让您有机会熟悉硬件，而不必编写内核代码。编写内核驱动程序并不特别困难，但是如果您这样做，需要小心编码，以免影响系统的稳定性。

我已经谈到了编写内核驱动程序代码：如果您选择这条路线，您将不可避免地想知道如何检查它是否正常工作并检测任何错误。我将在第十二章中涵盖这个主题，*使用 GDB 进行调试*。

下一章将全面介绍用户空间初始化以及`init`程序的不同选项，从简单的 BusyBox 到复杂的 systemd。
