# 第十三章：看门狗设备驱动程序

看门狗是一种旨在确保给定系统可用性的硬件（有时由软件模拟）设备。它有助于确保系统在关键挂起时始终重新启动，从而允许监视系统的“正常”行为。

无论是基于硬件还是由软件模拟，看门狗大多数情况下只是一个使用合理超时初始化的定时器，应该由受监视系统上运行的软件定期刷新。如果由于任何原因软件在超时之前停止/失败刷新定时器（并且没有明确关闭它），这将触发整个系统（在我们的情况下是计算机）的（硬件）复位。这种机制甚至可以帮助从内核恐慌中恢复。在本章结束时，您将能够做到以下事情：

+   阅读/理解现有的看门狗内核驱动程序，并在用户空间使用其提供的功能。

+   编写新的看门狗设备驱动程序。

+   掌握一些不太为人知的概念，如*看门狗管理器*和*预超时*。

在本章中，我们还将讨论 Linux 内核看门狗子系统背后的概念，包括以下主题：

+   看门狗数据结构和 API

+   看门狗用户空间接口

# 技术要求

在我们开始阅读本章之前，需要以下元素：

+   C 编程技能

+   基本电子知识

+   Linux 内核 v4.19.X 源代码，可在[`git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags)获取

# 看门狗数据结构和 API

在本节中，我们将深入研究看门狗框架，并了解其在底层的工作原理。看门狗子系统有一些数据结构。主要的是`struct watchdog_device`，它是 Linux 内核对看门狗设备的表示，包含有关它的所有信息。它在`include/linux/watchdog.h`中定义如下：

```
struct watchdog_device {
    int id;
    struct device *parent;
    const struct watchdog_info *info;
    const struct watchdog_ops *ops;
    const struct watchdog_governor *gov;
    unsigned int bootstatus;
    unsigned int timeout;
    unsigned int pretimeout;
    unsigned int min_timeout;
    struct watchdog_core_data *wd_data;
    unsigned long status;
    [...]
};
```

以下是此数据结构中字段的描述：

+   `id`：内核在设备注册期间分配的看门狗 ID。

+   `parent`：表示此设备的父级。

+   `info`：此`struct watchdog_info`结构指针提供有关看门狗定时器本身的一些附加信息。这是在看门狗字符设备上调用`WDIOC_GETSUPPORT` ioctl 以检索其功能时返回给用户的结构。我们稍后将详细介绍此结构。

+   `ops`：指向看门狗操作列表的指针。我们稍后将介绍这个数据结构。

+   `gov`：指向看门狗预超时管理器的指针。管理器只是根据某些事件或系统参数做出反应的策略管理器。

+   `bootstatus`：引导时看门狗设备的状态。这是触发系统复位的原因的位掩码。在描述`struct watchdog_info`结构时将枚举可能的值。

+   `timeout`：这是看门狗设备的超时值（以秒为单位）。

+   `pretimeout`：*预超时*的概念可以解释为在真正的超时发生之前的某个时间发生的事件，因此，如果系统处于不健康状态，它会在真正的超时复位之前触发中断。这些中断通常是不可屏蔽的（`pretimeout`字段实际上是触发真正超时中断之前的时间间隔（以秒为单位）。这不是直到预超时的秒数。例如，如果将超时设置为`60`秒，预超时设置为`10`，则预超时事件将在`50`秒触发。将预超时设置为`0`会禁用它。

+   `min_timeout`和`max_timeout`分别是看门狗设备的最小和最大超时值（以秒为单位）。这实际上是一个有效超时范围的下限和上限。如果值为 0，则框架将留下一个检查看门狗驱动程序本身。

+   `wd_data`：指向看门狗核心内部数据的指针。这个字段必须通过`watchdog_set_drvdata()`和`watchdog_get_drvdata()`助手来访问。

+   `status` 是一个包含设备内部状态位的字段。可能的值在这里列出：

--`WDOG_ACTIVE`：告诉看门狗是否正在运行/活动。

--`WDOG_NO_WAY_OUT`：通知是否设置了`nowayout`特性。您可以使用`watchdog_set_nowayout()`来设置`nowayout`特性；它的签名是`void watchdog_set_nowayout(struct watchdog_device *wdd, bool nowayout)`。

--`WDOG_STOP_ON_REBOOT`：应该在重启时停止。

--`WDOG_HW_RUNNING`：通知硬件看门狗正在运行。您可以使用`watchdog_hw_running()`助手来检查这个标志是否设置。但是，您应该在看门狗启动函数的成功路径上设置这个标志（或者在探测函数中，如果由于任何原因您在那里启动它或者发现看门狗已经启动）。您可以使用`set_bit()`助手来实现这一点。

--`WDOG_STOP_ON_UNREGISTER`：指定看门狗在注销时应该停止。您可以使用`watchdog_stop_on_unregister()`助手来设置这个标志。

正如我们之前介绍的，让我们详细了解`struct watchdog_info`结构，在`include/uapi/linux/watchdog.h`中定义，实际上，因为它是用户空间 API 的一部分：

```
struct watchdog_info {
    u32 options;
    u32 firmware_version;
    u8 identity[32];
};
```

这个结构也是在`WDIOC_GETSUPPORT`ioctl 的成功路径上返回给用户空间的。在这个结构中，字段的含义如下：

+   `options` 代表了卡片/驱动程序支持的能力。它是由看门狗设备/驱动程序支持的能力的位掩码，因为一些看门狗卡片提供的不仅仅是一个倒计时。这些标志中的一些也可以在`watchdog_device.bootstatus`字段中设置，以响应`GET_BOOT_STATUS`ioctl。这些标志如下列出，必要时给出双重解释：

--`WDIOF_SETTIMEOUT`表示看门狗设备可以设置超时。如果设置了这个标志，那么必须定义一个`set_timeout`回调。

--`WDIOF_MAGICCLOSE`表示驱动程序支持魔术关闭字符功能。由于关闭看门狗字符设备文件不会停止看门狗，这个功能意味着在这个看门狗文件中写入一个*V*字符（也称为魔术字符或魔术*V*）序列将允许下一个关闭关闭看门狗（如果没有设置`nowayout`）。

--`WDIOF_POWERUNDER`表示设备可以监视/检测不良电源或电源故障。当在`watchdog_device.bootstatus`中设置时，这个标志意味着机器显示了欠压触发了重置。

--另一方面，`WDIOF_POWEROVER`表示设备可以监视操作电压。当在`watchdog_device.bootstatus`中设置时，这意味着系统重置可能是由于过压状态。请注意，如果一个级别低于另一个级别，两个位都将被设置。

--`WDIOF_OVERHEAT`表示看门狗设备可以监视芯片/SoC 温度。当在`watchdog_device.bootstatus`中设置时，这意味着通过看门狗导致的上次机器重启的原因是超过了热限制。

--`WDIOF_FANFAULT`告诉我们这个看门狗设备可以监视风扇。当设置时，意味着看门狗卡监视的系统风扇已经失败。

一些设备甚至有单独的事件输入。如果定义了，这些输入上存在电信号，这也会导致重置。这就是`WDIOF_EXTERN1`和`WDIOF_EXTERN2`的目的。当在`watchdog_device.bootstatus`中设置时，这意味着机器上次重启是因为外部继电器/源 1 或 2。

--`WDIOF_PRETIMEOUT`表示这个看门狗设备支持预超时功能。

--`WDIOF_KEEPALIVEPING`表示此驱动程序支持`WDIOC_KEEPALIVE` ioctl（可以通过 ioctl 进行 ping）；否则，ioctl 将返回`-EOPNOTSUPP`。当在`watchdog_device.bootstatus`中设置时，此标志表示自上次查询以来看门狗看到了一个保持活动的 ping。

--`WDIOF_CARDRESET`：这是一个特殊标志，只能出现在`watchdog_device.bootstatus`中。它表示最后一次重启是由看门狗本身引起的（实际上是由它的超时引起的）。

+   `firmware_version`是卡的固件版本。

+   `identity`应该是描述设备的字符串。

另一个没有这个结构就无法实现任何操作的是`struct watchdog_ops`，定义如下：

```
struct watchdog_ops { struct module *owner;
    /* mandatory operations */
    int (*start)(struct watchdog_device *);
    int (*stop)(struct watchdog_device *);
    /* optional operations */
    int (*ping)(struct watchdog_device *);
    unsigned int (*status)(struct watchdog_device *);
    int (*set_timeout)(struct watchdog_device *, unsigned int);
    int (*set_pretimeout)(struct watchdog_device *,                           unsigned int);
    unsigned int (*get_timeleft)(struct watchdog_device *);
    int (*restart)(struct watchdog_device *,                    unsigned long, void *);
    long (*ioctl)(struct watchdog_device *, unsigned int,
                  unsigned long);
};
```

前面的结构包含了在看门狗设备上允许的操作列表。每个操作的含义在以下描述中呈现：

+   `start`和`stop`：这些是强制操作，分别启动和停止看门狗。

+   `ping`回调用于向看门狗发送保持活动的 ping。这个方法是可选的。如果没有定义，那么看门狗将通过`.start`操作重新启动，因为这意味着看门狗没有自己的 ping 方法。

+   `status`是一个可选的例程，返回看门狗设备的状态。如果定义了，其返回值将作为响应`WDIOC_GETBOOTSTATUS` ioctl 发送。

+   `set_timeout`是设置看门狗超时值（以秒为单位）的回调。如果定义了，还应该设置`X`选项标志；否则，任何尝试设置超时都将导致`-EOPNOTSUPP`错误。

+   `set_pretimeout`是设置预超时的回调。如果定义了，还应该设置`WDIOF_PRETIMEOUT`选项标志；否则，任何尝试设置预超时都将导致`-EOPNOTSUPP`错误。

+   `get_timeleft`是一个可选操作，返回重置前剩余的秒数。

+   `restart`：实际上是重新启动机器的例程（而不是看门狗设备）。如果设置了，您可能希望在注册看门狗设备之前调用`watchdog_set_restart_priority()`来设置此重启处理程序的优先级。

+   `ioctl`：除非必须，否则不应该实现此回调函数，例如，如果您需要处理额外/非标准的 ioctl 命令。如果定义了此方法，它将覆盖看门狗核心默认的 ioctl，除非它返回`-ENOIOCTLCMD`。

这个结构包含了设备支持的回调函数，根据其能力。

现在我们熟悉了数据结构，我们可以转向看门狗 API，特别是看如何在系统中注册和注销这样一个设备。

## 注册/注销看门狗设备

看门狗框架提供了两个基本函数来在系统中注册/注销看门狗设备。这些函数分别是`watchdog_register_device()`和`watchdog_unregister_device()`，它们的原型如下：

```
int watchdog_register_device(struct watchdog_device *wdd)
void watchdog_unregister_device(struct watchdog_device *wdd)
```

前面的注册方法在成功路径上返回零，或者在失败时返回一个负的*errno*代码。另一方面，`watchdog_unregister_device()`执行相反的操作。为了不再烦恼注销，您可以使用此函数的托管版本`devm_watchdog_register_device`，其原型如下：

```
int devm_watchdog_register_device(struct device *dev,                                   struct watchdog_device *wdd)
```

前面的托管版本将在驱动程序分离时自动处理注销。

注册方法（无论是托管还是非托管）都将检查是否提供了`wdd->ops->restart`函数，并将此方法注册为重启处理程序。因此，在向系统注册看门狗设备之前，驱动程序应该使用`watchdog_set_restart_priority()`助手来设置重启优先级，知道重启处理程序的优先级值应遵循以下准则：

+   `0`：这是最低优先级，意味着作为最后手段使用看门狗的重启功能；也就是说，在系统中没有提供其他重启处理程序时。

+   `128`：这是默认优先级，意味着如果没有其他处理程序可用，或者如果重新启动足以重新启动整个系统，则默认使用此重新启动处理程序。

+   `255`：这是最高优先级，可以抢占所有其他处理程序。

设备注册应该在您处理了我们讨论的所有元素之后才能完成；也就是说，在为看门狗设备提供有效的`.info`、`.ops`和与超时相关的字段之后。在所有这些之前，应为`watchdog_device`结构分配内存空间。将此结构包装在更大的、每个驱动程序数据结构中是一个好的做法，如下面的示例所示，这是从`drivers/watchdog/imx2_wdt.c`中摘录的：

```
[...]
struct imx2_wdt_device {
    struct clk *clk;
    struct regmap *regmap;
    struct watchdog_device wdog;
    bool ext_reset;
};
```

您可以看到看门狗设备数据结构嵌入在一个更大的结构`struct imx2_wdt_device`中。现在是`probe`方法，它初始化所有内容并在更大的结构中设置看门狗设备：

```
static int init imx2_wdt_probe(struct platform_device *pdev)
{
    struct imx2_wdt_device *wdev;
    struct watchdog_device *wdog; int ret;
    [...]
    wdev = devm_kzalloc(&pdev->dev, sizeof(*wdev), GFP_KERNEL);
    if (!wdev)
        return -ENOMEM;
    [...]
    Wdog = &wdev->wdog;
    if (imx2_wdt_is_running(wdev)) {
        imx2_wdt_set_timeout(wdog, wdog->timeout); 
        set_bit(WDOG_HW_RUNNING, &wdog->status);
    }
    ret = watchdog_register_device(wdog);
    if (ret) {
        dev_err(&pdev->dev, "cannot register watchdog device\n");
        [...]
    }
    return 0;
}
static int exit imx2_wdt_remove(struct platform_device *pdev)
{
    struct watchdog_device *wdog = platform_get_drvdata(pdev);
    struct imx2_wdt_device *wdev = watchdog_get_drvdata(wdog);
    watchdog_unregister_device(wdog);
    if (imx2_wdt_is_running(wdev)) {
      imx2_wdt_ping(wdog);
      dev_crit(&pdev->dev, "Device removed: Expect reboot!\n");
    }
    return 0;
}
[...]
```

此外，更大的结构可以在`move`方法中用于跟踪设备状态，特别是内嵌在其中的看门狗数据结构。这就是前面的代码摘录所突出的内容。

到目前为止，我们已经处理了看门狗的基础知识，走过了基本数据结构，并描述了主要的 API。现在，我们可以了解一些高级功能，比如预超时和管理者，以定义系统在看门狗事件发生时的行为。

## 处理预超时和管理者

在 Linux 内核中，*管理者*的概念出现在几个子系统中（热管理管理者、CPU 频率管理者，现在是看门狗管理者）。它只是一个实现策略管理（有时以算法的形式）的驱动程序，对系统的某些状态/事件做出反应。

每个子系统实现其管理者驱动程序的方式可能与其他子系统不同，但主要思想仍然是相同的。此外，管理者由唯一名称和正在使用的管理者（策略管理器）标识。它们通常可以在 sysfs 接口内部进行动态更改。

现在，回到看门狗预超时和管理者。可以通过启用`CONFIG_WATCHDOG_PRETIMEOUT_GOV`内核配置选项向 Linux 内核添加对它们的支持。实际上，内核中有两个看门狗管理者驱动程序：`drivers/watchdog/pretimeout_noop.c`和`drivers/watchdog/pretimeout_panic.c`。它们的唯一名称分别是`noop`和`panic`。可以通过启用`CONFIG_WATCHDOG_PRETIMEOUT_DEFAULT_GOV_NOOP`或`CONFIG_WATCHDOG_PRETIMEOUT_DEFAULT_GOV_PANIC`来默认使用其中任何一个。

本节的主要目标是将预超时事件传递给当前活动的看门狗管理者。这可以通过`watchdog_notify_pretimeout()`接口来实现，其原型如下：

```
void watchdog_notify_pretimeout(struct watchdog_device *wdd)
```

正如我们所讨论的，一些看门狗设备在预超时事件发生时会生成一个中断请求。主要思想是在此中断处理程序中调用`watchdog_notify_pretimeout()`。在底层，此接口将在全局注册的看门狗管理者列表中查找其名称，并调用其`.pretimeout`回调。

只是为了您的信息，以下是看门狗管理者结构的样子（您可以通过查看`drivers/watchdog/pretimeout_noop.c`或`drivers/watchdog/pretimeout_panic.c`中的源代码来了解更多关于看门狗管理者驱动程序的信息）：

```
struct watchdog_governor {
    const char name[WATCHDOG_GOV_NAME_MAXLEN];
    void (*pretimeout)(struct watchdog_device *wdd);
};
```

显然，其字段必须由底层看门狗管理器驱动程序填充。对于预超时通知的实际使用，您可以参考`drivers/watchdog/imx2_wdt.c`中定义的 i.MX6 看门狗驱动程序的 IRQ 处理程序。在前一节中已经显示了部分内容。在那里，您会注意到`watchdog_notify_pretimeout()`是从看门狗（实际上是预超时）IRQ 处理程序中调用的。此外，您会注意到，驱动程序根据看门狗是否有有效的 IRQ 使用不同的`watchdog_info`结构。如果有有效的 IRQ，将使用`.options`中设置了`WDIOF_PRETIMEOUT`标志的结构，这意味着设备具有预超时功能。否则，它将使用未设置`WDIOF_PRETIMEOUT`标志的结构。

现在我们已经熟悉了管理器和预超时的概念，我们可以考虑学习实现看门狗的另一种方法，例如基于 GPIO 的方法。

## 基于 GPIO 的看门狗

有时，使用外部看门狗设备可能比使用 SoC 本身提供的看门狗更好，例如出于功耗效率的原因，因为有些 SoC 的内部看门狗需要比外部看门狗更多的功率。大多数情况下，这种外部看门狗设备是通过 GPIO 线控制的，并且具有重置系统的可能性。它通过切换连接的 GPIO 线来进行 ping 操作。这种配置在 UDOO QUAD 中使用（未在其他 UDOO 变体上进行检查）。

Linux 内核能够通过启用`CONFIG_GPIO_WATCHDOG config`选项来处理此设备，这将拉取底层驱动程序`drivers/watchdog/gpio_wdt.c`。如果启用，它将定期通过切换连接到 GPIO 线的硬件来*ping*。如果该硬件未定期接收到 ping，它将重置系统。您应该使用这个而不是直接使用 sysfs 与 GPIO 进行通信；它提供了比 GPIO 更好的 sysfs 用户空间接口，并且与内核框架集成得比您的用户空间代码更好。

这种支持仅来自设备树，有关其绑定的更好文档可以在`Documentation/devicetree/bindings/watchdog/gpio-wdt.txt`中找到，显然是在内核源代码中。

以下是一个绑定示例：

```
watchdog: watchdog {
    compatible = "linux,wdt-gpio";
    gpios = <&gpio3 9 GPIO_ACTIVE_LOW>;
    hw_algo = "toggle";
    hw_margin_ms = <1600>;
};
```

`compatible`属性必须始终为`linux,wdt-gpio`。`gpios`是控制看门狗设备的 GPIO 指定器。`hw_algo`应为`toggle`或`level`。前者意味着可以使用低至高或高至低的转换来 ping 外部看门狗设备，并且当 GPIO 线浮空或连接到三态缓冲器时，看门狗被禁用。为了实现这一点，将 GPIO 配置为输入就足够了。第二个`algo`意味着应用信号电平（高或低）就足以 ping 看门狗。

其工作方式如下：当用户空间代码通过`/dev/watchdog`设备文件 ping 看门狗时，底层驱动程序（实际上是`gpio_wdt.c`）将切换 GPIO 线（如果`hw_algo`是`toggle`，则为`1-0-1`）或在该 GPIO 线上分配特定电平（如果`hw_algo`是`level`，则为高或低）。例如，UDOO QUAD 使用`APX823-31W5`，一个由 GPIO 控制的看门狗，其事件输出连接到 i.MX6 PORB 线（实际上是复位线）。其原理图在这里可用：[`udoo.org/download/files/schematics/UDOO_REV_D_schematics.pdf`](http://udoo.org/download/files/schematics/UDOO_REV_D_schematics.pdf)。

现在，我们在内核端完成了看门狗。我们已经了解了底层数据结构，处理了其 API，介绍了预超时的概念，甚至处理了基于 GPIO 的看门狗替代方案。在接下来的部分中，我们将研究用户空间实现，这是看门狗服务的一种消费者。

# 看门狗用户空间接口

在基于 Linux 的系统上，看门狗的标准用户空间接口是`/dev/watchdog`文件，通过它，守护进程将通知内核看门狗驱动程序用户空间仍然活动。文件打开后，看门狗立即启动，并通过定期写入此文件进行 ping。

当通知发生时，底层驱动程序将通知看门狗设备，这将导致重置其超时；然后看门狗将等待另一个`timeout`持续时间之后才重置系统。但是，如果由于任何原因用户空间在超时之前未执行通知，看门狗将重置系统（导致重新启动）。这种机制提供了一种强制系统可用性的方法。让我们从基础知识开始，学习如何启动和停止看门狗。

## 启动和停止看门狗

一旦您打开`/dev/watchdog`设备文件，看门狗就会自动启动，如下例所示：

```
int fd;
fd = open("/dev/watchdog", O_WRONLY);
if (fd == -1) {
    if (errno == ENOENT)
        printf("Watchdog device not enabled.\n");
    else if (errno == EACCES)
        printf("Run watchdog as root.\n");
    else
        printf("Watchdog device open failed %s\n", strerror(errno));
    exit(-1);
}
```

只是关闭看门狗设备文件并不能停止它。关闭文件后，您将惊讶地发现系统重置。要正确停止看门狗，您首先需要向看门狗设备文件写入魔术字符*V*。这会指示内核在下次关闭设备文件时关闭看门狗，如下所示：

```
const char v = 'V';
printf("Send magic character: V\n"); ret = write(fd, &v, 1);
if (ret < 0)
    printf("Stopping watchdog ticks failed (%d)...\n", errno);
```

然后，您需要关闭看门狗设备文件以停止它：

```
printf("Close for stopping..\n");
close(fd);
```

重要说明

关闭文件设备以停止看门狗时有一个例外：即内核的`CONFIG_WATCHDOG_NOWAYOUT`配置选项启用时。启用此选项后，看门狗将无法停止。因此，您需要一直对其进行服务，否则它将重置系统。此外，看门狗驱动程序应该在其选项中设置`WDIOF_MAGICCLOSE`标志；否则，魔术关闭功能将无法工作。

现在我们已经了解了如何启动和停止看门狗，现在是时候学习如何刷新设备以防止系统突然重新启动了。

### ping/kick 看门狗-发送保持活动的 ping

有两种方法可以踢或喂狗：

1.  向`/dev/watchdog`写入任何字符：向看门狗设备文件写入被定义为保持活动的 ping。建议根本不要写入`V`字符（因为它具有特定含义），即使它在字符串中也是如此。

1.  使用`WDIOC_KEEPALIVE` ioctl，`ioctl(fd, WDIOC_KEEPALIVE,` `0);`：忽略 ioctl 的参数。看门狗驱动程序应该在此 ioctl 之前在其选项中设置`WDIOF_KEEPALIVEPING`标志，以便其工作。

喂狗是一个好的做法，每半个超时值喂一次狗。这意味着如果超时是`30s`，您应该每`15s`喂一次。现在，让我们了解一些关于看门狗如何管理我们的系统的信息。

## 获取看门狗能力和身份

获取看门狗能力和/或身份包括抓取与看门狗关联的底层`struct watchdog_info`结构。如果您记得的话，这个信息结构是强制性的，并由看门狗驱动程序提供。

为了实现这一点，您需要使用`WDIOC_GETSUPPORT` ioctl。以下是一个例子：

```
struct watchdog_info ident;
ioctl(fd, WDIOC_GETSUPPORT, &ident);
printf("WDIOC_GETSUPPORT:\n");
/* Printing the watchdog's identity, its unique name actually */
printf("\tident.identity = %s\n",ident.identity);
/* Printing the firmware version */
printf("\tident.firmware_version = %d\n",        ident.firmware_version);
/* Printing supported options (capabilities) in hex format */
printf("WDIOC_GETSUPPORT: ident.options = 0x%x\n",       ident.options);
```

我们可以通过测试一些功能来进一步了解其能力，如下所示：

```
if (ident.options & WDIOF_KEEPALIVEPING)
    printf("\tKeep alive ping reply.\n");
if (ident.options & WDIOF_SETTIMEOUT)
    printf("\tCan set/get the timeout.\n");
```

您可以（或者我应该说"必须"）使用这个来在执行某些操作之前检查看门狗的功能。现在，我们可以进一步学习如何获取和设置更多花哨的看门狗属性。

## 设置和获取超时和预超时

在设置/获取超时之前，看门狗信息应该设置`WDIOF_SETTIMEOUT`标志。有一些驱动程序可以使用`WDIOC_SETTIMEOUT` ioctl 动态修改看门狗超时时间。这些驱动程序必须在其看门狗信息结构中设置`WDIOF_SETTIMEOUT`标志，并提供`.set_timeout`回调。

虽然这里的参数是以秒为单位表示的超时值的整数，但返回值是应用于硬件设备的实际超时，因为由于硬件限制，它可能与 ioctl 中请求的超时不同：

```
int timeout = 45;
ioctl(fd, WDIOC_SETTIMEOUT, &timeout);
printf("The timeout was set to %d seconds\n", timeout);
```

当涉及到查询当前超时时，您应该使用`WDIOC_GETTIMEOUT`ioctl，如下例所示：

```
int timeout;
ioctl(fd, WDIOC_GETTIMEOUT, &timeout);
printf("The timeout is %d seconds\n", timeout);
```

最后，当涉及到预超时时，看门狗驱动程序应该在选项中设置`WDIOF_PRETIMEOUT`，并在其操作中提供`.set_pretimeout`回调。然后，您应该使用`WDIOC_SETPRETIMEOUT`，并将预超时值作为参数：

```
pretimeout = 10;
ioctl(fd, WDIOC_SETPRETIMEOUT, &pretimeout);
```

如果所需的预超时值为`0`或大于当前超时，则会收到`-EINVAL`错误。

现在我们已经看到了如何获取和设置看门狗设备的超时/预超时，我们可以学习如何获取看门狗触发之前剩余的时间。

### 获取剩余时间

`WDIOC_GETTIMELEFT`ioctl 允许检查看门狗计数器在发生重置之前剩余多少时间。此外，看门狗驱动程序应通过提供`.get_timeleft()`回调来支持此功能；否则，您将收到`EOPNOTSUPP`错误。以下是一个示例，显示如何使用此 ioctl：

```
int timeleft;
ioctl(fd, WDIOC_GETTIMELEFT, &timeleft);
printf("The remaining timeout is %d seconds\n", timeleft);
```

`timeleft`变量在 ioctl 的返回路径上填充。

一旦看门狗触发，它会在配置为这样做时触发重启。在下一节中，我们将学习如何获取上次重启的原因，以查看重启是否是由看门狗引起的。

## 获取（引导/重新引导）状态

在本节中有两个 ioctl 命令可供使用。这些是`WDIOC_GETSTATUS`和`WDIOC_GETBOOTSTATUS`。这些的处理方式取决于驱动程序的实现，并且有两种类型的驱动程序实现：

+   通过杂项设备提供看门狗功能的旧驱动程序。这些驱动程序不使用通用看门狗框架接口，并提供自己的`file_ops`以及自己的`.ioctl`操作。此外，这些驱动程序仅支持`WDIOC_GETSTATUS`，而其他驱动程序可能同时支持`WDIOC_GETSTATUS`和`WDIOC_GETBOOTSTATUS`。两者之间的区别在于前者将返回设备状态寄存器的原始内容，而后者应该更智能，因为它解析原始内容并仅返回引导状态标志。这些驱动程序需要迁移到新的通用看门狗框架。请注意，一些支持两个命令的驱动程序可能会为两个 ioctl 返回相同的值（相同的`case`语句），而其他驱动程序可能会返回不同的值（每个命令都有自己的`case`语句）。

+   新驱动程序使用通用看门狗框架。这些驱动程序依赖于框架，不再关心`file_ops`。一切都是从`drivers/watchdog/watchdog_dev.c`文件中完成的（您可以查看，特别是如何实现 ioctl 命令）。对于这类驱动程序，`WDIOC_GETSTATUS`和`WDIOC_GETBOOTSTATUS`由看门狗核心分别处理。本节将处理这些驱动程序。

现在，让我们专注于通用实现。对于这些驱动程序，`WDIOC_GETBOOTSTATUS`将返回底层`watchdog_device.bootstatus`字段的值。对于`WDIOC_GETSTATUS`，如果提供了看门狗`.status`操作，将调用它，并将其返回值复制到用户;否则，将使用`AND`操作调整`watchdog_device.bootstatus`的内容，以清除（或标记）无意义的位。以下代码片段显示了在内核空间中如何完成这项工作：

```
static unsigned int watchdog_get_status(struct                                         watchdog_device *wdd)
{
    struct watchdog_core_data *wd_data = wdd->wd_data;
    unsigned int status;
    if (wdd->ops->status)
        status = wdd->ops->status(wdd);
    else
        status = wdd->bootstatus &
                      (WDIOF_CARDRESET | WDIOF_OVERHEAT |
                       WDIOF_FANFAULT | WDIOF_EXTERN1 |
                       WDIOF_EXTERN2 | WDIOF_POWERUNDER |
                       WDIOF_POWEROVER);
    if (test_bit(_WDOG_ALLOW_RELEASE, &wd_data->status))
        status |= WDIOF_MAGICCLOSE;
    if (test_and_clear_bit(_WDOG_KEEPALIVE, &wd_data->status))
        status |= WDIOF_KEEPALIVEPING;
    return status;
}
```

上述代码是一个用于获取看门狗状态的通用看门狗核心函数。实际上，它是一个负责调用底层`ops.status`回调的包装器。现在，回到我们的用户空间使用。我们可以这样做：

```
int flags = 0;
int flags;
ioctl(fd, WDIOC_GETSTATUS, &flags);
/* or ioctl(fd, WDIOC_GETBOOTSTATUS, &flags); */
```

显然，我们可以继续像在*获取看门狗功能和身份*部分中那样进行单独的标志检查。

到目前为止，我们已经编写了与看门狗设备交互的代码。下一节将向我们展示如何在用户空间处理看门狗而无需编写代码，基本上使用 sysfs 接口。

## 看门狗 sysfs 接口

看门狗框架提供了通过 sysfs 接口从用户空间管理看门狗设备的可能性。如果内核中启用了`CONFIG_WATCHDOG_SYSFS`配置选项，并且根目录是`/sys/class/watchdogX/`，则这是可能的。`X`是系统中看门狗设备的索引。sysfs 中的每个看门狗目录都具有以下内容：

+   `nowayout`：如果设备支持`nowayout`功能，则返回`1`，否则返回`0`。

+   `status`：这是`WDIOC_GETSTATUS`ioctl 的 sysfs 等效项。此 sysfs 文件报告了看门狗的内部状态位。

+   `timeleft`：这是`WDIOC_GETTIMELEFT`ioctl 的 sysfs 等效项。此 sysfs 条目返回在看门狗重置系统之前剩余的时间（实际上是秒数）。

+   `timeout`：给出已编程超时的当前值。

+   `identity`：包含看门狗设备的标识字符串。

+   `bootstatus`：这是`WDIOC_GETBOOTSTATUS`ioctl 的 sysfs 等效项。此条目通知系统重置是否是由看门狗设备引起的。

+   `state`：给出看门狗设备的活动/非活动状态。

现在，前面的看门狗属性已经描述了，我们可以专注于来自用户空间的预超时管理。

### 处理预超时事件

通过 sysfs 设置管理器。管理器只是根据一些外部（但输入）参数采取某些操作的策略管理器。有热管理器、CPUFreq 管理器，现在还有看门狗管理器。每个管理器都在自己的驱动程序中实现。

您可以使用以下命令检查看门狗（比如`watchdog0`）的可用管理器：

```
# cat /sys/class/watchdog/watchdog0/pretimeout_available_governors
noop panic
```

现在，我们可以检查是否可以选择预超时管理器：

```
# cat /sys/class/watchdog/watchdog0/pretimeout_governor
panic
# echo -n noop > /sys/class/watchdog/watchdog0/pretimeout_governor
# cat /sys/class/watchdog/watchdog0/pretimeout_governor
noop
```

要检查预超时值，您可以简单地执行以下操作：

```
# cat /sys/class/watchdog/watchdog0/pretimeout
10
```

现在我们熟悉了如何从用户空间使用看门狗 sysfs 接口。虽然我们不在内核中，但我们可以利用整个框架，特别是与看门狗参数交互。

# 总结

在本章中，我们讨论了看门狗设备的所有方面：它们的 API、GPIO 替代方案以及它们如何帮助保持系统可靠。我们看到了如何启动，如何（在可能的情况下）停止，以及如何维护看门狗设备。此外，我们介绍了预超时和专用看门狗管理器的概念。

在下一章中，我们将讨论一些 Linux 内核开发和调试技巧，比如分析内核恐慌消息和内核跟踪。
