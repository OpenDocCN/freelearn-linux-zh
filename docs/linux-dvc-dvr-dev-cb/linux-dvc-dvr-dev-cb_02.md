# 第二章：深入了解内核

简单的操作系统（如 MS-DOS）总是在单 CPU 模式下执行，但类 Unix 操作系统使用双模式来有效地实现时间共享和资源分配和保护。在 Linux 中，CPU 在任何时候都处于受信任的**内核模式**（我们可以做任何我们想做的事情）或受限的**用户模式**（某些操作不允许）。所有用户进程都在用户模式下执行，而核心内核本身和大多数设备驱动程序（除了在用户空间实现的驱动程序）都在内核模式下运行，因此它们可以无限制地访问整个处理器指令集以及完整的内存和 I/O 空间。

当用户模式进程需要访问外围设备时，它不能自己完成，而必须通过设备驱动程序或其他内核模式代码通过**系统调用**来传递请求，系统调用在控制进程活动和管理数据交换中起着重要作用。在本章中，我们不会看到系统调用（它们将在第三章中介绍），但我们将通过直接向内核源代码添加新代码或使用内核模块来开始在内核中编程，这是另一种更灵活的方式来向内核添加代码。

一旦我们开始编写内核代码，我们必须不要忘记，当处于用户模式时，每个资源分配（CPU、RAM 等）都由内核自动管理（当进程死亡时可以适当释放它们），在内核模式下，我们被允许独占处理器，直到我们自愿放弃 CPU 或发生中断或异常；此外，如果不适当释放，每个请求的资源（如 RAM）都会丢失。这就是为什么正确管理 CPU 使用和释放我们请求的任何资源非常重要！

现在，是时候第一次跳入内核了，因此在本章中，我们将涵盖以下示例：

+   向源代码添加自定义代码

+   使用内核消息

+   使用内核模块

+   使用模块参数

# 技术要求

在本章中，我们需要在第一章的*配置和构建内核*示例中已经下载的内核源代码，当然，我们还需要安装交叉编译器，就像在第一章的*设置主机机器*示例中所示。本章中使用的代码和其他文件可以从 GitHub 上下载：[`github.com/giometti/linux_device_driver_development_cookbook/tree/master/chapter_02`](https://github.com/giometti/linux_device_driver_development_cookbook/tree/master/chapter_02)。

# 向源代码添加自定义代码

作为第一步，让我们看看如何向我们的内核源代码中添加一些简单的代码。在这个示例中，我们将简单地添加一些愚蠢的代码，只是为了演示它有多容易，但在本书的后面，我们将添加更复杂的代码。

# 准备工作

由于我们需要将我们的代码添加到 Linux 源代码中，让我们进入存放所有源代码的目录。在我的系统中，我使用位于我的主目录中的`Projects/ldddc/linux/`路径。以下是内核源代码的样子：

```
$ cd Projects/ldddc/linux/
$ ls
arch        Documentation  Kbuild       mm               scripts   virt
block       drivers        Kconfig      modules.builtin  security  vmlinux
built-in.a  firmware       kernel       modules.order    sound     vmlinux.o
certs       fs             lib          Module.symvers   stNXtP40
COPYING     include        LICENSES     net System.map
CREDITS     init           MAINTAINERS  README tools
crypto      ipc            Makefile     samples usr
```

现在，我们需要设置环境变量`ARCH`和`CROSS_COMPILE`，如下所示，以便能够为 ESPRESSObin 进行交叉编译代码：

```
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-linux-gnu-
```

因此，如果我们尝试执行以下`make`命令，系统应该像往常一样开始编译内核：

```
$ make Image dtbs modules
  CALL scripts/checksyscalls.sh
...
```

请注意，您可以通过在以下命令行上指定它们来避免导出前面的变量：

`$ make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- \`

`Image dtbs modules`

此时，内核源代码和编译环境已经准备就绪。

# 如何做...

让我们按照以下步骤来做：

1.  由于本书涉及设备驱动程序，让我们从 Linux 源代码的`drivers`目录下开始添加我们的代码，具体来说是在`drivers/misc`中，杂项驱动程序所在的地方。我们应该在`drivers/misc`中放置一个名为`dummy-code.c`的文件，内容如下：

```
/*
 * Dummy code
 */

#include <linux/module.h>

static int __init dummy_code_init(void)
{
    printk(KERN_INFO "dummy-code loaded\n");
    return 0;
}

static void __exit dummy_code_exit(void)
{
    printk(KERN_INFO "dummy-code unloaded\n");
}

module_init(dummy_code_init);
module_exit(dummy_code_exit);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Rodolfo Giometti");
MODULE_DESCRIPTION("Dummy code");
```

1.  我们的新文件`drivers/misc/dummy-code.c`如果不正确地插入到内核配置和构建系统中，将不会产生任何效果。为了做到这一点，我们必须修改`drivers/misc/Kconfig`和`drivers/misc/Makefile`文件如下。前者文件必须更改如下：

```
--- a/drivers/misc/Kconfig
+++ b/drivers/misc/Kconfig
@@ -527,4 +527,10 @@ source "drivers/misc/echo/Kconfig"
 source "drivers/misc/cxl/Kconfig"
 source "drivers/misc/ocxl/Kconfig"
 source "drivers/misc/cardreader/Kconfig"
+
+config DUMMY_CODE
+       tristate "Dummy code"
+       default n
+       ---help---
+         This module is just for demonstration purposes.
 endmenu
```

后者的修改如下：

```
--- a/drivers/misc/Makefile
+++ b/drivers/misc/Makefile
@@ -58,3 +58,4 @@ obj-$(CONFIG_ASPEED_LPC_SNOOP) += aspeed-lpc-snoop.o
 obj-$(CONFIG_PCI_ENDPOINT_TEST) += pci_endpoint_test.o
 obj-$(CONFIG_OCXL) += ocxl/
 obj-$(CONFIG_MISC_RTSX) += cardreader/
+obj-$(CONFIG_DUMMY_CODE) += dummy-code.o
```

请注意，您可以通过在 Linux 源代码的主目录中使用`patch`命令轻松添加前面的代码以及编译所需的任何内容，如下所示：

**`$ patch -p1 < add_custom_code.patch`**

1.  好吧，如果我们现在使用`make menuconfig`命令，并且我们通过设备驱动程序导航到杂项设备菜单条目的底部，我们应该会得到以下截图所示的内容：

！[](img/01c65282-ef07-458f-bc60-be630fb3a9e1.png)

在前面的截图中，我已经选择了虚拟代码条目，以便我们可以看到最终的设置应该是什么样子的。

请注意，虚拟代码条目必须选择为内置（`*`字符），而不是模块（`M`字符）。

还要注意，如果我们不执行`make menuconfig`命令，而是直接执行`make Image`命令来编译内核，那么构建系统将询问我们如何处理`DUMMY_CODE`设置，如下所示。显然，我们必须使用`y`字符回答是：

**`$ make Image`**

`scripts/kconfig/conf --syncconfig Kconfig`

`*`

`* 重新启动配置...`

`*`

`*`

`* 杂项设备`

`*`

`模拟设备数字电位器（AD525X_DPOT）[N/m/y/?] n`

`...`

`虚拟代码（DUMMY_CODE）[N/m/y/?]（NEW）y`

1.  如果一切都摆放正确，那么我们执行`make Image`命令重新编译内核。我们应该看到我们的新文件被编译然后添加到内核`Image`文件中，如下所示：

```
$ make Image
scripts/kconfig/conf --syncconfig Kconfig
...
  CC drivers/misc/dummy-code.o
  AR drivers/misc/built-in.a
  AR drivers/built-in.a
...
  LD vmlinux
  SORTEX vmlinux
  SYSMAP System.map
  OBJCOPY arch/arm64/boot/Image
```

1.  好了，现在我们要做的就是用刚刚重新构建的`Image`文件替换 microSD 上的`Image`文件，然后重新启动系统（参见第一章中的*如何添加内核*配方，*安装开发系统*）。

# 它是如何工作的...

现在，是时候看看之前所有步骤是如何工作的了。在接下来的章节中，我们将更好地解释这段代码的真正作用。但是，目前，我们应该注意以下内容。

在*步骤 1*中，请注意对`module_init()`和`module_exit()`的调用，这是内核提供的 C 宏，用于告诉内核，在系统启动或关闭期间，必须调用我们提供的函数，名为`dummy_code_init()`和`dummy_code_exit()`，这些函数只是打印一些信息消息。

在本章的后面，我们将详细了解`printk()`的作用以及`KERN_INFO`宏的含义，但是目前，我们只需要考虑它们用于在引导（或关闭）期间打印消息。例如，前面的代码指示内核在引导阶段的某个时候打印出消息 dummy-code loaded。

在*步骤 2*中，在`Makefile`中，我们只是告诉内核，如果启用了`CONFIG_DUMMY_CODE`（即`CONFIG_DUMMY_CODE=y`），那么必须编译并插入内核二进制文件（链接）`dummy-code.c`，而使用`Kconfig`文件，我们只是将新模块添加到内核配置系统中。

在*步骤 3*中，我们使用`make menuconfig`命令启用我们的代码的编译。

最后，在*步骤 4*中，我们重新编译内核以将我们的代码添加到其中。

在*步骤 5*中，在引导过程中，我们应该看到以下内核消息：

```
...
loop: module loaded
dummy-code loaded
ahci-mvebu d00e0000.sata: AHCI 0001.0300 32 slots 1 ports 6 Gbps
...
```

# 另请参阅

+   有关内核配置及其构建系统工作原理的更多信息，我们可以查看内核源代码中的内核文档文件，路径为`linux/Documentation/kbuild/kconfig-macro-language.txt`。

# 使用内核消息

正如前面所述，串行控制台在我们需要从头开始设置系统时非常有用，但如果我们希望在生成时立即看到内核消息，它也非常有用。为了生成内核消息，我们可以使用多个函数，在本教程中，我们将看看它们以及如何在串行控制台或通过 SSH 连接显示消息。

# 准备工作

我们的 ESPRESSObin 是生成内核消息的系统，所以我们需要与它建立连接。通过串行控制台，这些消息一旦到达就会自动显示，但如果我们使用 SSH 连接，我们仍然可以通过读取特定文件来显示它们，就像以下命令一样：

```
# tail -f /var/log/kern.log
```

然而，串行控制台值得特别注意：实际上，在我们的示例中，只有当`/proc/sys/kernel/printk`文件中最左边的数字大于七时，内核消息才会自动显示在串行控制台上，如下所示：

```
# cat /proc/sys/kernel/printk
10      4       1       7
```

这些魔术数字有明确定义的含义；特别是第一个代表内核必须在串行控制台上显示的错误消息级别。这些级别在`linux/include/linux/kern_levels.h`文件中定义，如下所示：

```
#define KERN_EMERG KERN_SOH "0"    /* system is unusable */
#define KERN_ALERT KERN_SOH "1"    /* action must be taken immediately */
#define KERN_CRIT KERN_SOH "2"     /* critical conditions */
#define KERN_ERR KERN_SOH "3"      /* error conditions */
#define KERN_WARNING KERN_SOH "4"  /* warning conditions */
#define KERN_NOTICE KERN_SOH "5"   /* normal but significant condition */
#define KERN_INFO KERN_SOH "6"     /* informational */
#define KERN_DEBUG KERN_SOH "7"    /* debug-level messages */
```

例如，如果前面文件的内容是 4，如下所示，只有具有`KERN_EMERG`、`KERN_ALERT`、`KERN_CRIT`和`KERN_ERR`级别的消息才会自动显示在串行控制台上：

```
# cat /proc/sys/kernel/printk
4       4       1       7
```

为了允许显示所有消息、它们的子集或不显示任何消息，我们必须使用`echo`命令修改`/proc/sys/kernel/printk`文件的最左边的数字，就像在以下示例中那样，我们以这种方式完全禁用所有内核消息的打印。这是因为没有消息的优先级可以大于 0：

```
 # echo 0 > /proc/sys/kernel/printk
```

内核消息的优先级从 0（最高）开始，到 7（最低）结束！

现在我们知道如何显示内核消息，我们可以尝试对内核代码进行一些修改，以便对内核消息进行一些实验。

# 如何做到...

在前面的示例中，我们看到可以使用`printk()`函数生成内核消息，但是还有其他函数可以替代`printk()`，以便获得更高效的消息和更紧凑可读的代码：

1.  使用以下宏（在`include/linux/printk.h`文件中定义），如下所示：

```
#define pr_emerg(fmt, ...) \
        printk(KERN_EMERG pr_fmt(fmt), ##__VA_ARGS__)
#define pr_alert(fmt, ...) \
        printk(KERN_ALERT pr_fmt(fmt), ##__VA_ARGS__)
#define pr_crit(fmt, ...) \
        printk(KERN_CRIT pr_fmt(fmt), ##__VA_ARGS__)
#define pr_err(fmt, ...) \
        printk(KERN_ERR pr_fmt(fmt), ##__VA_ARGS__)
#define pr_warning(fmt, ...) \
        printk(KERN_WARNING pr_fmt(fmt), ##__VA_ARGS__)
#define pr_warn pr_warning
#define pr_notice(fmt, ...) \
        printk(KERN_NOTICE pr_fmt(fmt), ##__VA_ARGS__)
#define pr_info(fmt, ...) \
        printk(KERN_INFO pr_fmt(fmt), ##__VA_ARGS__)
```

1.  现在，要生成一个内核消息，我们可以这样做：查看这些定义，我们可以将前面示例中的`dummy_code_init()`和`dummy_code_exit()`函数重写到`dummy-code.c`文件中，如下所示：

```
static int __init dummy_code_init(void)
{
        pr_info("dummy-code loaded\n");
        return 0;
}

static void __exit dummy_code_exit(void)
{
        pr_info("dummy-code unloaded\n");
}
```

# 工作原理...

如果我们仔细观察前面的打印函数（`pr_info()`和类似的函数），我们会注意到它们还依赖于`pr_fmt(fmt)`参数，该参数可用于向我们的消息中添加其他有用的信息。例如，以下定义通过添加当前模块和调用函数名称来改变`pr_info()`生成的所有消息：

```
#define pr_fmt(fmt) "%s:%s: " fmt, KBUILD_MODNAME, __func__
```

请注意，`pr_fmt()`宏定义必须出现在文件的开头，甚至在包含之前，才能生效。

如果我们将这行添加到我们的`dummy-code.c`中，内核消息将会按照描述发生变化：

```
/*
 * Dummy code
 */

#define pr_fmt(fmt) "%s:%s: " fmt, KBUILD_MODNAME, __func__
#include <linux/module.h>
```

实际上，当执行`pr_info()`函数时，输出消息会告诉我们模块已被插入，变成以下形式，我们可以看到模块名称和调用函数名称，然后是加载消息：

```
dummy_code:dummy_code_init: dummy-code loaded
```

还有另一组打印函数，但在开始讨论它们之前，我们需要一些位于第三章中的信息，*使用设备树*，所以，暂时，我们只会继续使用这些函数。

# 还有更多...

有许多内核活动，其中许多确实很复杂，而且经常，内核开发人员必须处理几条消息，而不是所有消息都有趣；因此，我们需要找到一些方法来过滤出有趣的消息。

# 过滤内核消息

假设我们希望知道在引导期间检测到了哪些串行端口。我们知道可以使用`tail`命令，但是通过使用它，我们只能看到最新的消息；另一方面，我们可以使用`cat`命令来回忆自引导以来的所有内核消息，但那是大量的信息！或者，我们可以使用以下步骤来过滤内核消息：

1.  在这里，我们使用`grep`命令来过滤`uart`（或`UART`）字符串中的行：

```
# cat /var/log/kern.log | grep -i uart
Feb 7 19:33:14 espressobin kernel: [ 0.000000] earlycon: ar3700_uart0 at MMIO 0x00000000d0012000 (options '')
Feb 7 19:33:14 espressobin kernel: [ 0.000000] bootconsole [ar3700_uart0] enabled
Feb 7 19:33:14 espressobin kernel: [ 0.000000] Kernel command line: console=ttyMV0,115200 earlycon=ar3700_uart,0xd0012000 loglevel=0 debug root=/dev/mmcblk0p1 rw rootwait net.ifnames=0 biosdevname=0
Feb 7 19:33:14 espressobin kernel: [ 0.289914] Serial: AMBA PL011 UART driver
Feb 7 19:33:14 espressobin kernel: [ 0.296443] mvebu-uart d0012000.serial: could not find pctldev for node /soc/internal-regs@d0000000/pinctrl@13800/uart1-pins, deferring probe
...
```

前面的输出也可以通过使用`dmesg`命令来获得，这是一个专为此目的设计的工具：

```
# dmesg | grep -i uart
[ 0.000000] earlycon: ar3700_uart0 at MMIO 0x00000000d0012000 (options '')
[ 0.000000] bootconsole [ar3700_uart0] enabled
[ 0.000000] Kernel command line: console=ttyMV0,115200 earlycon=ar3700_uart,0
xd0012000 loglevel=0 debug root=/dev/mmcblk0p1 rw rootwait net.ifnames=0 biosdev
name=0
[ 0.289914] Serial: AMBA PL011 UART driver
[ 0.296443] mvebu-uart d0012000.serial: could not find pctldev for node /soc/
internal-regs@d0000000/pinctrl@13800/uart1-pins, deferring probe
...
```

请注意，虽然`cat`显示日志文件中的所有内容，甚至是来自先前操作系统执行的非常旧的消息，但`dmesg`仅显示当前操作系统执行的消息。这是因为`dmesg`直接从当前运行的系统通过其环形缓冲区（即存储所有消息的缓冲区）获取内核消息。

1.  另一方面，如果我们想收集有关早期引导活动的信息，我们仍然可以使用`dmesg`命令和`head`命令，以仅显示`dmesg`输出的前 10 行：

```
# dmesg | head -10 
[ 0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[ 0.000000] Linux version 4.18.0-dirty (giometti@giometti-VirtualBox) (gcc ve
rsion 7.3.0 (Ubuntu/Linaro 7.3.0-27ubuntu1~18.04)) #5 SMP PREEMPT Sun Jan 27 13:
33:24 CET 2019
[ 0.000000] Machine model: Globalscale Marvell ESPRESSOBin Board
[ 0.000000] earlycon: ar3700_uart0 at MMIO 0x00000000d0012000 (options '')
[ 0.000000] bootconsole [ar3700_uart0] enabled
[ 0.000000] efi: Getting EFI parameters from FDT:
[ 0.000000] efi: UEFI not found.
[ 0.000000] cma: Reserved 32 MiB at 0x000000007e000000
[ 0.000000] NUMA: No NUMA configuration found
[ 0.000000] NUMA: Faking a node at [mem 0x0000000000000000-0x000000007fffffff]
```

1.  另一方面，如果我们对最后 10 行感兴趣，我们可以使用`tail`命令。实际上，我们已经看到，为了监视内核活动，我们可以像下面这样使用它：

```
# tail -f /var/log/kern.log
```

因此，要查看最后 10 行，我们可以执行以下操作：

```
# dmesg | tail -10 
```

1.  同样，也可以使用`dmesg`，通过添加`-w`选项参数，如下例所示：

```
# dmesg -w
```

1.  `dmesg`命令也可以根据它们的级别过滤内核消息，方法是使用`-l`（或`--level`）选项参数，如下所示：

```
# dmesg -l 3 
[ 1.687783] advk-pcie d0070000.pcie: link never came up
[ 3.153849] advk-pcie d0070000.pcie: Posted PIO Response Status: CA, 0xe00 @ 0x0
[ 3.688578] Unable to create integrity sysfs dir: -19
```

前面的命令显示具有`KERN_ERR`级别的内核消息，而以下是显示具有`KERN_WARNING`级别的消息的命令：

```
# dmesg -l 4
[ 3.164121] EINJ: ACPI disabled.
[ 3.197263] cacheinfo: Unable to detect cache hierarchy for CPU 0
[ 4.572660] xenon-sdhci d00d0000.sdhci: Timing issue might occur in DDR mode
[ 5.316949] systemd-sysv-ge: 10 output lines suppressed due to ratelimiting
```

1.  我们还可以组合级别，以同时具有`KERN_ERR`和`KERN_WARNING`：

```
# dmesg -l 3,4
[ 1.687783] advk-pcie d0070000.pcie: link never came up
[ 3.153849] advk-pcie d0070000.pcie: Posted PIO Response Status: CA, 0xe00 @ 0x0
[ 3.164121] EINJ: ACPI disabled.
[ 3.197263] cacheinfo: Unable to detect cache hierarchy for CPU 0
[ 3.688578] Unable to create integrity sysfs dir: -19
[ 4.572660] xenon-sdhci d00d0000.sdhci: Timing issue might occur in DDR mode
[ 5.316949] systemd-sysv-ge: 10 output lines suppressed due to ratelimiting
```

1.  最后，在大量嘈杂的消息的情况下，我们可以要求系统通过使用以下命令来清除内核环形缓冲区（存储所有内核消息的地方）：

```
# dmesg -C
```

现在，如果我们再次使用`dmesg`，我们将只看到新生成的内核消息。

# 另请参阅

+   有关内核消息管理的更多信息，一个很好的起点是`dmesg`手册页，我们可以通过执行`man dmesg`命令来显示它。

# 使用内核模块

了解如何向内核添加自定义代码是有用的，但是，当我们必须编写新的驱动程序时，将我们的代码编写为**内核模块**可能更有用。实际上，通过使用模块，我们可以轻松修改内核代码，然后在不需要每次重新启动系统的情况下进行测试！我们只需删除然后重新插入模块（在必要的修改之后）以测试我们代码的新版本。

在这个示例中，我们将看看即使在内核树之外的目录中，内核模块也可以被编译。

# 准备工作

要将我们的`dummy-code.c`文件转换为内核模块，我们只需更改内核设置，允许编译我们示例模块（在内核配置菜单中用`*`字符替换为`M`）。但是，在某些情况下，将我们的驱动程序发布到与内核源代码完全分开的专用存档中可能更有用。即使在这种情况下，也不需要对现有代码进行任何更改，我们将能够在内核源树内部或者在外部编译`dummy-code.c`！

要构建我们的第一个内核模块作为外部代码，我们可以安全地使用前面的`dummy-code.c`文件，然后将其放入一个专用目录，并使用以下`Makefile`：

```
ifndef KERNEL_DIR
$(error KERNEL_DIR must be set in the command line)
endif
PWD := $(shell pwd)
ARCH ?= arm64
CROSS_COMPILE ?= aarch64-linux-gnu-

# This specifies the kernel module to be compiled
obj-m += dummy-code.o

# The default action
all: modules

# The main tasks
modules clean:
    make -C $(KERNEL_DIR) \
              ARCH=$(ARCH) \
              CROSS_COMPILE=$(CROSS_COMPILE) \
              SUBDIRS=$(PWD) $@
```

查看前面的代码，我们看到`KERNEL_DIR`变量必须在命令行上提供，指向 ESPRESSObin 之前编译的内核源代码的路径，而`ARCH`和`CROSS_COMPILE`变量不是强制性的，因为`Makefile`指定了它们（但是，在命令行上提供它们将优先）。

此外，我们应该验证`insmod`和`rmmod`命令是否在我们的 ESPRESSObin 中可用，如下所示：

```
# insmod -h
Usage:
        insmod [options] filename [args]
Options:
        -V, --version show version
        -h, --help show this help
```

如果不存在，那么可以通过使用通常的`apt install kmod`命令添加`kmod`软件包来安装它们。

# 如何做...

让我们看看如何通过以下步骤来做到这一点：

1.  在将`dummy-code.c`和`Makefile`文件放置在主机 PC 上的当前工作目录后，当使用`ls`命令时，它应该如下所示：

```
$ ls
dummy-code.c  Makefile
```

1.  然后，我们可以使用以下命令编译我们的模块：

```
$ make KERNEL_DIR=../../../linux/
make -C ../../../linux/ \
 ARCH=arm64 \
 CROSS_COMPILE=aarch64-linux-gnu- \
 SUBDIRS=/home/giometti/Projects/ldddc/github/chapter_2/module modules
make[1]: Entering directory '/home/giometti/Projects/ldddc/linux'
 CC [M] /home/giometti/Projects/ldddc/github/chapter_2/module/dummy-code.o
 Building modules, stage 2.
 MODPOST 1 modules
 CC /home/giometti/Projects/ldddc/github/chapter_2/module/dummy-code.mod.o
 LD [M] /home/giometti/Projects/ldddc/github/chapter_2/module/dummy-code.ko
make[1]: Leaving directory '/home/giometti/Projects/ldddc/linux'
```

如我们所见，现在我们在当前工作目录中有几个文件，其中一个名为`dummy-code.ko`；这是我们的内核模块，准备好传输到 ESPRESSObin！

1.  一旦模块已经移动到目标系统（例如，通过使用`scp`命令），我们可以使用`insmod`实用程序加载它，如下所示：

```
# insmod dummy-code.ko
```

1.  现在，通过使用`lsmod`命令，我们可以要求系统显示所有加载的模块。在我的 ESPRESSObin 上，我只有`dummy-code.ko`模块，所以我的输出如下所示：

```
# lsmod 
Module         Size  Used by
dummy_code    16384  0
```

请注意，由于内核模块名称中的`-`字符被替换为`_`，内核模块名称的`.ko`后缀已被删除。

1.  然后，我们可以使用`rmmod`命令从内核中删除我们的模块，如下所示：

```
# rmmod dummy_code
```

如果出现以下错误，请验证您是否运行了我们在第一章中获得的正确`Image`文件，*安装开发系统*

`rmmod: ERROR: ../libkmod/libkmod.c:514 lookup_builtin_file() could not open builtin file '/lib/modules/4.18.0-dirty/modules.builtin.bin'`

# 它是如何工作的...

`insmod`命令只是将我们的模块插入内核；之后，它执行`module_init()`函数。

在模块插入期间，如果我们在 SSH 连接上，终端上将看不到任何内容，我们必须使用`dmesg`来查看内核消息（或者在串行控制台上，在插入模块后，我们应该看到类似以下内容的内容：

```
dummy_code: loading out-of-tree module taints kernel.
dummy_code:dummy_code_init: dummy-code loaded
```

请注意，消息“加载非树模块会污染内核”只是一个警告，可以安全地忽略我们的目的。有关污染内核的更多信息，请参见[`www.kernel.org/doc/html/v4.15/admin-guide/tainted-kernels.html`](https://www.kernel.org/doc/html/v4.15/admin-guide/tainted-kernels.html)。

`rmmod`命令执行`module_exit()`函数，然后从内核中删除模块，执行`insmod`的逆步骤。

# 另请参阅

+   有关 modutils 的更多信息，它们的手册页是一个很好的起点（命令是：`man insmod`，`man rmmod`和`man modinfo`）；此外，我们可以通过阅读其手册页（`man modprobe`）来了解`modprobe`命令。

# 使用模块参数

在内核模块开发过程中，动态设置一些变量在模块插入时非常有用，而不仅仅是在编译时。在 Linux 中，可以通过使用内核模块的参数来实现，这允许通过在`insmod`命令的命令行上指定参数来传递参数给模块。

# 准备工作

为了举例说明，让我们考虑一个情况，我们有一个新的模块信息文件`module_par.c`（此文件也在我们的 GitHub 存储库中）。

# 如何做...

让我们看看如何通过以下步骤来做到这一点：

1.  首先，让我们定义我们的模块参数，如下所示：

```
static int var = 0x3f;
module_param(var, int, S_IRUSR | S_IWUSR);
MODULE_PARM_DESC(var, "an integer value");

static char *str = "default string";
module_param(str, charp, S_IRUSR | S_IWUSR);
MODULE_PARM_DESC(str, "a string value");

#define ARR_SIZE 8
static int arr[ARR_SIZE];
static int arr_count;
module_param_array(arr, int, &arr_count, S_IRUSR | S_IWUSR);
MODULE_PARM_DESC(arr, "an array of " __stringify(ARR_SIZE) " values");
```

1.  然后，我们可以使用以下的`init`和`exit`函数：

```
static int __init module_par_init(void)
{
    int i;

    pr_info("loaded\n");
    pr_info("var = 0x%02x\n", var);
    pr_info("str = \"%s\"\n", str);
    pr_info("arr = ");
    for (i = 0; i < ARR_SIZE; i++)
        pr_cont("%d ", arr[i]);
    pr_cont("\n");

    return 0;
}

static void __exit module_par_exit(void)
{
    pr_info("unloaded\n");
}

module_init(module_par_init);
module_exit(module_par_exit);
```

1.  最后，在最后，我们可以像往常一样添加模块描述宏：

```
MODULE_LICENSE("GPL");
MODULE_AUTHOR("Rodolfo Giometti");
MODULE_DESCRIPTION("Module with parameters");
MODULE_VERSION("0.1");
```

# 工作原理...

编译完成后，应该会生成一个名为`module_par.ko`的新文件，可以加载到我们的 ESPRESSObin 中。但在这之前，让我们使用`modinfo`实用程序对其进行如下操作：

```
# modinfo module_par.ko 
filename:    /root/module_par.ko
version:     0.1
description: Module with parameters
author:      Rodolfo Giometti
license:     GPL
srcversion:  21315B65C307ABE9769814F
depends: 
name:        module_par
vermagic:    4.18.0 SMP preempt mod_unload aarch64
parm:        var:an integer value (int)
parm:        str:a string value (charp)
parm:        arr:an array of 8 values (array of int)
```

`modinfo`命令也包含在`kmod`软件包中，名为`insmod`。

正如我们在最后三行中所看到的（都以`parm：`字符串为前缀），我们在代码中使用`module_param（）`和`module_param_array（）`宏定义了模块的参数列表，并使用`MODULE_PARM_DESC（）`进行描述。

现在，如果我们像以前一样插入模块，我们会得到默认值，如下面的代码块所示：

```
# insmod module_par.ko 
[ 6021.345064] module_par:module_par_init: loaded
[ 6021.347028] module_par:module_par_init: var = 0x3f
[ 6021.351810] module_par:module_par_init: str = "default string"
[ 6021.357904] module_par:module_par_init: arr = 0 0 0 0 0 0 0 0
```

但是，如果我们使用下一个命令行，我们可以强制使用新值：

```
# insmod module_par.ko var=0x01 str=\"new value\" arr='1,2,3' 
[ 6074.175964] module_par:module_par_init: loaded
[ 6074.177915] module_par:module_par_init: var = 0x01
[ 6074.184932] module_par:module_par_init: str = "new value"
[ 6074.189765] module_par:module_par_init: arr = 1 2 3 0 0 0 0 0 
```

在尝试使用新值重新加载之前，请不要忘记使用`rmmod module_par`命令删除`module_par`模块！

最后，让我建议仔细查看以下模块参数定义：

```
static int var = 0x3f;
module_param(var, int, S_IRUSR | S_IWUSR);
MODULE_PARM_DESC(var, "an integer value");
```

首先，我们有代表参数的变量声明，然后是真正的模块参数定义（在这里我们指定类型和文件访问权限），然后是描述。

`modinfo`命令能够显示所有前面的信息，除了文件访问权限，这些权限是指与`sysfs`文件系统中的参数相关的文件！实际上，如果我们看一下`/sys/module/module_par/parameters/`目录，我们会得到以下内容：

```
# ls -l /sys/module/module_par/parameters/
total 0
-rw------- 1 root root 4096 Feb 1 12:46 arr
-rw------- 1 root root 4096 Feb 1 12:46 str
-rw------- 1 root root 4096 Feb 1 12:46 var
```

现在，应该清楚参数`S_IRUSR`和`S_IWUSR`的含义；它们允许模块用户（即 root 用户）写入这些文件，然后从中读取相应的参数。

`S_IRUSR`和相关函数的定义在以下文件中：`linux/include/uapi/linux/stat.h`。

# 另请参阅

+   关于内核模块的一般信息以及如何导出内核符号，您可以查看在线提供的*Linux 内核模块编程指南*，网址为[`www.tldp.org/LDP/lkmpg/2.6/html/index.html`](https://www.tldp.org/LDP/lkmpg/2.6/html/index.html)[.](https://www.tldp.org/LDP/lkmpg/2.6/html/index.html)
