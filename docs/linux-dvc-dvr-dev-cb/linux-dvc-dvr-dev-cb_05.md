# 第五章：管理中断和并发

在实现设备驱动程序时，开发人员必须解决两个主要问题：

+   如何与外围设备交换数据

+   如何管理外围设备生成的中断到 CPU

第一个点（至少对于字符驱动程序）在以前的章节中已经涵盖了，而第二个点（及其相关内容）将是本章的主题。

在内核中，我们可以将 CPU（或执行某些代码的内部核心）视为运行在两个主要执行上下文中——**中断上下文**和**进程上下文**。中断上下文非常容易理解；事实上，每当 CPU 执行中断处理程序时（即内核每次发生中断时执行的特殊代码），CPU 就处于这种上下文中。除此之外，中断可以由硬件或甚至软件生成；这就是为什么我们谈论硬件中断和软件中断（我们将在接下来的章节中更详细地了解软件中断），从而定义了**硬件中断上下文**和**软件中断上下文**。

另一方面，**进程上下文**是指 CPU（或其内部核心之一）在内核空间中执行进程的某些代码时（进程也在用户空间中执行，但我们这里不涉及），也就是说，当 CPU 执行进程调用的系统调用代码时。在这种情况下，很常见的是让出 CPU，然后暂停当前进程，因为外围设备的一些数据尚未准备好读取；例如；这可以通过要求调度程序接管 CPU，然后将其分配给另一个进程来完成。当这种情况发生时，我们通常说当前**进程已进入睡眠状态**，当数据新可用时，我们说**进程已被唤醒**，并且它会在先前中断的地方重新执行。

在本章中，我们将看到如何执行所有这些操作，设备驱动程序开发人员如何要求内核暂停当前的读取过程，因为外围设备尚未准备好提供请求，并且还将看到如何唤醒睡眠进程。我们还将看到如何管理对驱动程序方法的并发访问，以避免由于竞争条件导致的数据损坏，以及如何管理时间流以便在经过明确定义的时间后执行特定操作，以尊重外围设备可能需要的时间约束。

我们还将看看如何在字符驱动程序和用户空间之间交换数据，以及如何处理驱动程序应该能够管理的内核事件。第一个（也可能是最重要的）示例是如何管理中断，其次是如何推迟工作“稍后”，以及如何等待事件。我们可以使用以下方法来执行所有这些操作：

+   实现中断处理程序

+   推迟工作

+   使用内核定时器管理时间

+   等待事件

+   执行原子操作

# 技术要求

有关本章的更多信息，您可以访问*附录*。

本章中使用的代码和其他文件可以从 GitHub 下载：[`github.com/giometti/linux_device_driver_development_cookbook/tree/master/chapter_05`](https://github.com/giometti/linux_device_driver_development_cookbook/tree/master/chapter_05)。

# 实现中断处理程序

在内核中，**中断处理程序**是与 CPU 中断线（或引脚）相关联的函数，当连接到该线的外围设备更改引脚状态时，Linux 会执行该函数；当这种情况发生时，会为 CPU 生成中断请求，并且被内核捕获，然后执行适当的处理程序。

在这个示例中，我们将看到如何安装一个中断处理程序，内核每次在一个明确定义的线上发生中断时都会执行该处理程序。

# 准备就绪

实现中断处理程序的最简单代码是`linux/drivers/misc/dummy-irq.c`中的代码。这是处理程序：

```
static int irq = -1;

static irqreturn_t dummy_interrupt(int irq, void *dev_id)
{
    static int count = 0;

    if (count == 0) {
        printk(KERN_INFO "dummy-irq: interrupt occurred on IRQ %d\n",
                irq);
        count++;
    }

    return IRQ_NONE;
}
```

以下是安装或删除它的代码：

```
static int __init dummy_irq_init(void)
{
    if (irq < 0) {
        printk(KERN_ERR "dummy-irq: no IRQ given. Use irq=N\n");
        return -EIO;
    }
    if (request_irq(irq, &dummy_interrupt, IRQF_SHARED, "dummy_irq", &irq)) {
        printk(KERN_ERR "dummy-irq: cannot register IRQ %d\n", irq);
        return -EIO;
    }
    printk(KERN_INFO "dummy-irq: registered for IRQ %d\n", irq);
    return 0;
}

static void __exit dummy_irq_exit(void)
{
    printk(KERN_INFO "dummy-irq unloaded\n");
    free_irq(irq, &irq);
}
```

这段代码非常简单，正如我们所看到的，它在`dummy_irq_init()`模块初始化函数中调用`request_irq()`函数，并在`dummy_irq_exit()`模块退出函数中调用`free_irq()`函数。然后，这两个函数分别要求内核将`dummy_interrupt()`中断处理程序连接到`irq`中断线，并在相反操作中将处理程序从中断线中分离。

这段代码简要地展示了如何安装中断处理程序；然而，它并没有展示设备驱动程序开发人员如何安装自己的处理程序；这就是为什么在下一节中，我们将使用一个真实的中断线的实际示例，使用通用输入输出线（GPIO）模拟。

为了实现对我们的第一个**中断请求**（**IRQ**）处理程序的管理，我们可以使用一个普通的 GPIO 作为中断线；然而，在这样做之前，我们必须验证我们的 GPIO 线是否正确检测到高低输入电平。

为了管理 GPIO，我们将使用其 sysfs 接口，因此，首先，我们必须验证它是否当前对我们的内核启用，方法是检查`/sys/class/gpio`目录是否存在。如果不存在，我们将不得不通过使用内核配置菜单（`make menuconfig`）启用`CONFIG_GPIO_SYSFS`内核配置条目；可以通过转到设备驱动程序，然后 GPIO 支持，启用/sys/class/gpio/...（sysfs 接口）菜单条目来完成。

通过使用以下命令行，我们可以快速检查条目是否已启用：

```
$ rgrep CONFIG_GPIO_SYSFS .config
CONFIG_GPIO_SYSFS=y
```

否则，如果它没有被启用，我们将得到以下输出，然后我们必须启用它：

```
$ rgrep CONFIG_GPIO_SYSFS .config
# CONFIG_GPIO_SYSFS is not set
```

如果一切就绪，我们应该得到类似以下的内容：

```
# ls /sys/class/gpio/
export  gpiochip446  gpiochip476  unexport
```

`gpiochip446`和`gpiochip476`目录代表了两个 ESPRESSObin 的 GPIO 控制器，正如我们在上一章中描述设备树时所看到的。（参见附录中的*The Armada 3720*部分第四章，*使用设备树*，*为特定外围设备配置 CPU 引脚*部分）。`export`和`unexport`文件用于访问 GPIO 线。

为了完成我们的工作，我们需要访问映射到 ESPRESSObin 扩展#2 的引脚 12 的 MPP2_20 CPU 线；也就是说，在 ESPRESSObin 原理图上的连接器 P8（或 J18）。 （参见第一章中的*技术要求*部分，*安装开发系统*）。在 CPU 数据表中，我们发现 MPP2_20 线连接到第二个 pinctrl 控制器（在设备树中命名为南桥并映射为`pinctrl_sb: pinctrl@18800`）。要知道使用哪个正确的 gpiochip 设备，我们仍然可以使用 sysfs 如下：

```
# ls -l /sys/class/gpio/gpiochip4*
lrwxrwxrwx 1 root root 0 Mar 7 20:20 /sys/class/gpio/gpiochip446 ->
  ../../devices/platform/soc/soc:internal-regs@d0000000/d0018800.pinctrl/gpio/gpiochip446
lrwxrwxrwx 1 root root 0 Mar 7 20:20 /sys/class/gpio/gpiochip476 ->
  ../../devices/platform/soc/soc:internal-regs@d0000000/d0013800.pinctrl/gpio/gpiochip476
```

现在很明显我们必须使用`gpiochip446`。在那个目录中，我们会找到`base`文件，告诉我们第一个 GPIO 线的对应编号，由于我们使用的是第 20 条线，我们应该将`base+20` GPIO 线导出如下：

```
# cat /sys/class/gpio/gpiochip446/base 
446
# echo 466 > /sys/class/gpio/export 
```

如果一切正常，现在在`/sys/class/gpio`目录中会出现一个新的`gpio466`条目，对应于我们刚刚导出的 GPIO 线：

```
# ls /sys/class/gpio/ 
export  gpio466  gpiochip446  gpiochip476  unexport
```

太好了！`gpio466`目录现在已经准备好使用了，通过查看其中的内容，我们得到以下文件：

```
# ls /sys/class/gpio/gpio466/
active_low device direction edge power subsystem uevent value
```

为了查看我们是否能够修改我们的 GPIO 线，我们可以简单地使用以下命令：

```
cat /sys/class/gpio/gpio466/value 
1
```

请注意，即使未连接，该线路也被设置为 1，因为该引脚通常配置为内部上拉，强制引脚状态为高电平。

这个输出告诉我们 GPIO 线 20 当前处于高电平，但是，如果我们将 P8 连接器的引脚 12 连接到同一连接器（P8/J8）的地线（引脚 1 或 2），GPIO 线应该转为低电平，前面的命令现在应该返回 0，如下所示：

```
# cat /sys/class/gpio/gpio466/value 
0
```

如果线路没有改变，您应该验证您是否在正确的引脚/连接器上工作。此外，您应该查看`/sys/class/gpio/gpio466/direction`文件，其中应该包含`in`字符串，如下所示：

**`# cat /sys/class/gpio/gpio466/direction`**

`in`

好了。现在我们准备生成我们的中断！

# 如何做...

通过以下步骤来看看如何做：

1.  现在，让我们假设我们有一个专用的平台驱动程序名为`irqtest`，在 ESPRESSObin 设备树中定义如下：

```
    irqtest {
        compatible = "ldddc,irqtest";

        gpios = <&gpiosb 20 GPIO_ACTIVE_LOW>;
    };
```

请记住，ESPRESSObin 设备树文件是`linux/arch/arm64/boot/dts/marvell/armada-3720-espressobin.dts`。

1.  然后，我们必须像在上一章中那样向内核添加一个平台驱动程序，使用以下代码：

```
static const struct of_device_id irqtest_dt_ids[] = {
    { .compatible = "ldddc,irqtest", },
    { /* sentinel */ }
};
MODULE_DEVICE_TABLE(of, irqtest_dt_ids);

static struct platform_driver irqtest_driver = {
    .probe = irqtest_probe,
    .remove = irqtest_remove,
    .driver = {
        .name = "irqtest",
        .of_match_table = irqtest_dt_ids,
    },
};

module_platform_driver(irqtest_driver);
```

请注意，这里呈现的所有代码都可以通过在内核源代码的根目录中执行`patch`命令应用`add_irqtest_module.patch`补丁来从 GitHub 存储库获取，如下所示：

`**$ patch -p1 < ../linux_device_driver_development_cookbook/chapter_5/add_irqtest_module.patch**`

1.  现在，我们知道一旦内核在设备树中检测到与`ldddc,irqtest`兼容的驱动程序，将执行以下`irqtest_probe()`探测函数。这个函数与前面的`linux/drivers/misc/dummy-irq.c`文件中的函数非常相似，即使有点更复杂。实际上，首先我们必须从设备树中读取中断信号来自哪个 GPIO 线，使用`of_get_gpio()`函数：

```
static int irqtest_probe(struct platform_device *pdev)
{
    struct device *dev = &pdev->dev;
    struct device_node *np = dev->of_node;
    int ret;

    /* Read gpios property (just the first entry) */
    ret = of_get_gpio(np, 0); 
    if (ret < 0) {
        dev_err(dev, "failed to get GPIO from device tree\n");
        return ret;
    }
    irqinfo.pin = ret;
    dev_info(dev, "got GPIO %u from DTS\n", irqinfo.pin);
```

1.  然后，我们必须使用`devm_gpio_request()`函数向内核请求 GPIO 线：

```
    /* Now request the GPIO and set the line as an input */
    ret = devm_gpio_request(dev, irqinfo.pin, "irqtest");
    if (ret) {
        dev_err(dev, "failed to request GPIO %u\n", irqinfo.pin);
        return ret;
    }
    ret = gpio_direction_input(irqinfo.pin);
    if (ret) {
        dev_err(dev, "failed to set pin input direction\n");
        return -EINVAL;
    }

    /* Now ask to the kernel to convert GPIO line into an IRQ line */
    ret = gpio_to_irq(irqinfo.pin);
    if (ret < 0) {
        dev_err(dev, "failed to map GPIO to IRQ!\n");
        return -EINVAL;
    }
    irqinfo.irq = ret;
    dev_info(dev, "GPIO %u correspond to IRQ %d\n",
                irqinfo.pin, irqinfo.irq);
```

1.  确定 GPIO 仅供我们使用后，我们必须将其设置为输入（中断是传入信号），使用`gpio_direction_input()`函数，然后我们必须使用`gpio_to_irq()`函数获取相应的中断线号（通常是不同的号码）：

```
    ret = gpio_direction_input(irqinfo.pin);
    if (ret) {
        dev_err(dev, "failed to set pin input direction\n");
        return -EINVAL;
    }

    /* Now ask to the kernel to convert GPIO line into an IRQ line */
    ret = gpio_to_irq(irqinfo.pin);
    if (ret < 0) {
        dev_err(dev, "failed to map GPIO to IRQ!\n");
        return -EINVAL;
    }
    irqinfo.irq = ret;
    dev_info(dev, "GPIO %u correspond to IRQ %d\n",
                irqinfo.pin, irqinfo.irq);
```

1.  之后，我们有了所有必要的信息，可以使用`linux/include/linux/interrupt.h`头文件中定义的`request_irq()`函数安装我们的中断处理程序，如下所示：

```
extern int __must_check
request_threaded_irq(unsigned int irq, irq_handler_t handler,
            irq_handler_t thread_fn,
            unsigned long flags, const char *name, void *dev);

static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
            const char *name, void *dev)
{
    return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}

extern int __must_check
request_any_context_irq(unsigned int irq, irq_handler_t handler,
            unsigned long flags, const char *name, void *dev_id);
```

1.  最后，`handler`参数指定要作为中断处理程序执行的函数，`dev`是一个指针，内核在执行时会原样传递给处理程序。在我们的示例中，中断处理程序定义如下：

```
static irqreturn_t irqtest_interrupt(int irq, void *dev_id)
{
    struct irqtest_data *info = dev_id;
    struct device *dev = info->dev;

    dev_info(dev, "interrupt occurred on IRQ %d\n", irq);

    return IRQ_HANDLED;
}
```

# 工作原理...

在*步骤 1*中，节点声明了一个与驱动程序名为`ldddc,irqtest`兼容的设备，该设备需要使用`gpiosb`节点的 GPIO 线 20，如在 Armada 3270 设备树`arch/arm64/boot/dts/marvell/armada-37xx.dtsi`文件中定义的那样：

```
    pinctrl_sb: pinctrl@18800 {
        compatible = "marvell,armada3710-sb-pinctrl",
                 "syscon", "simple-mfd";
        reg = <0x18800 0x100>, <0x18C00 0x20>;
        /* MPP2[23:0] */
        gpiosb: gpio {
            #gpio-cells = <2>;
            gpio-ranges = <&pinctrl_sb 0 0 30>;
            gpio-controller;
            interrupt-controller;
            #interrupt-cells = <2>;
            interrupts =
            <GIC_SPI 160 IRQ_TYPE_LEVEL_HIGH>,
            <GIC_SPI 159 IRQ_TYPE_LEVEL_HIGH>,
            <GIC_SPI 158 IRQ_TYPE_LEVEL_HIGH>,
            <GIC_SPI 157 IRQ_TYPE_LEVEL_HIGH>,
            <GIC_SPI 156 IRQ_TYPE_LEVEL_HIGH>;
        };
   ...
```

在这里，我们确认`gpiosb`节点与 MPP2 线相关。

在*步骤 2*中，我们只是在内核中声明驱动程序，而在*步骤 3*中，该函数从`gpio`属性获取 GPIO 信息，并且通过将第二个参数设置为`0`，我们只是请求第一个条目。返回值保存在模块的数据结构中，现在定义如下：

```
static struct irqtest_data {
    int irq;
    unsigned int pin;
    struct device *dev;
} irqinfo;
```

在步骤 4 中，实际上，`devm_gpio_request()`调用并不是严格需要的，因为我们在内核中，没有人可以阻止我们使用资源；但是，如果所有驱动程序都这样做，我们可以确保在有其他人持有资源时得到通知！

现在我们应该注意到`devm_gpio_request()`函数在模块的`exit()`函数`irqtest_remove()`中没有对应的函数。这是因为带有`devm`前缀的函数与能够在所有者设备从系统中移除时自动释放资源的托管设备相关。

在定义此函数的`linux/drivers/gpio/devres.c`文件中，我们看到以下注释，解释了此函数的工作原理：

`/**`

`* devm_gpio_request - 为托管设备请求 GPIO`

`* @dev: 请求 GPIO 的设备`

`* @gpio: 要分配的 GPIO`

`* @label: 请求的 GPIO 的名称`

`*`

`* 除了额外的@dev 参数外，此函数还需要`

`* 使用相同的参数并执行相同的功能`

`* gpio_request()。使用此功能请求的 GPIO 将被`

`* 在驱动程序分离时自动释放。`

`*`

`* 如果使用此功能分配的 GPIO 需要被释放`

`* 另外，必须使用 devm_gpio_free()。`

`*/`

这是高级资源管理，超出了本书的范围。但是，如果你感兴趣，互联网上有很多信息，以下是一个很好的文章起点：[`lwn.net/Articles/222860/`](https://lwn.net/Articles/222860/)。

无论如何，`devm_gpio_request()`函数的正常对应函数是`gpio_request()`和`gpio_free()`函数。

在第 5 步中，请注意，GPIO 线号几乎永远不对应中断线号；这就是为什么我们需要调用`gpio_to_irq()`函数以获取与我们的 GPIO 线相关的正确 IRQ 线的原因。

在第 6 步中，我们可以看到`request_irq()`函数是`request_threaded_irq()`函数的一个特例，它告诉我们中断处理程序可以在中断上下文中运行，或者在进程上下文中运行的内核线程中运行。

目前，我们仍然不知道什么是内核线程（它们将在第六章中解释，*杂项内核内部*），但应该很容易理解它们类似于在内核空间中执行的线程（或进程）。

还可以使用`request_any_context_irq()`函数来委托内核自动请求正常的中断处理程序或线程中的中断处理程序，具体取决于 IRQ 线的特性。

这是中断处理程序的一个非常高级的用法，当我们需要管理外围设备（如 I2C 或 SPI 设备）时，我们需要挂起中断处理程序才能从外围寄存器中读取或写入数据。

除了这些方面，所有的`request_irq*()`函数都需要几个参数。首先是`irq`线，然后是一个符号`name`，描述我们可以在`/proc/interrupts`文件中找到的中断线，然后我们可以使用`flags`参数来指定一些特殊设置，如下所示（请参阅`linux/include/linux/interrupt.h`文件以获取完整列表）：

```
/*
 * These correspond to the IORESOURCE_IRQ_* defines in
 * linux/ioport.h to select the interrupt line behaviour. When
 * requesting an interrupt without specifying a IRQF_TRIGGER, the
 * setting should be assumed to be "as already configured", which
 * may be as per machine or firmware initialisation.
 */
#define IRQF_TRIGGER_NONE 0x00000000
#define IRQF_TRIGGER_RISING 0x00000001
#define IRQF_TRIGGER_FALLING 0x00000002
#define IRQF_TRIGGER_HIGH 0x00000004
#define IRQF_TRIGGER_LOW 0x00000008
...
/*
 * IRQF_SHARED - allow sharing the irq among several devices
 * IRQF_ONESHOT - Interrupt is not reenabled after the hardirq handler finished.
 * Used by threaded interrupts which need to keep the
 * irq line disabled until the threaded handler has been run.
 * IRQF_NO_SUSPEND - Do not disable this IRQ during suspend. Does not guarantee
 * that this interrupt will wake the system from a suspended
 * state. See Documentation/power/suspend-and-interrupts.txt
 */
```

当 IRQ 线与多个外围设备共享时，应该使用`IRQF_SHARED`标志。（现在它几乎没有用，但在过去，它非常有用，特别是在 x86 机器上。）`IRQF_ONESHOT`标志被系统用来确保即使线程中断处理程序也可以在其自己的 IRQ 线被禁用时运行。`IRQF_NO_SUSPEND`标志可用于允许我们的外围设备从挂起状态唤醒系统，通过发送适当的中断请求。（有关更多详细信息，请参阅`linux/Documentation/power/suspend-and-interrupts.txt`文件。）

然后，`IRQF_TRIGGER_*`标志可用于指定我们外围设备的 IRQ 触发模式，即中断是否必须在高电平或低电平上产生，或在上升或下降转换期间产生。

这些最后的标志组应该仔细检查设备树 pinctrl 设置；否则，我们可能会看到一些意外的行为。

在第 7 步中，由于在`request_irq()`函数中我们将`dev`参数设置为`struct irqtest_data`模块的指针，当`irqtest_interrupt()`中断处理程序执行时，它将在`dev_id`参数中找到我们提供给`request_irq()`的相同指针。通过使用这个技巧，我们可以得到从探测函数中得到的`dev`值，并且可以安全地将其重新用作`dev_info()`函数的参数，就像之前一样。

在我们的示例中，中断处理程序几乎什么都没做，只是显示一条消息。但是，通常在中断处理程序中，我们必须确认外围设备，从中读取或写入数据，然后唤醒所有正在等待外围设备活动的睡眠进程。无论如何，在最后，处理程序应该返回`linux/include/linux/irqreturn.h`文件中列出的一个值：

```
/**
 * enum irqreturn
 * @IRQ_NONE interrupt was not from this device or was not handled
 * @IRQ_HANDLED interrupt was handled by this device
 * @IRQ_WAKE_THREAD handler requests to wake the handler thread
 */
```

`IRQ_NONE`值在我们正在处理共享中断的情况下非常有用，以通知系统当前的 IRQ 不是针对我们的，并且必须传递给下一个处理程序，而`IRQ_WAKE_THREAD`应该在使用线程化 IRQ 处理程序的情况下使用。当然，必须使用`IRQ_HANDLED`来向系统报告 IRQ 已被处理。

# 还有更多...

如果您想要检查这是如何工作的，我们可以通过测试我们的示例来做到这一点。我们必须编译它，然后将内核与我们编译为内置的代码一起重新安装，因此让我们使用通常的`make menuconfig`命令并启用我们的测试代码，或者只需使用`make oldconfig`，在系统要求选择时回答`y`，如下所示：

```
Simple IRQ test (IRQTEST_CODE) [N/m/y/?] (NEW)
```

之后，我们只需重新编译和重新安装内核，然后重新启动 ESPRESSObin。如果在引导序列期间一切正常，我们应该看到内核消息如下：

```
irqtest irqtest: got GPIO 466 from DTS
irqtest irqtest: GPIO 466 correspond to IRQ 40
irqtest irqtest: interrupt handler for IRQ 40 is now ready!
```

现在，MPP2_20 线已被内核占用，并转换为编号 40 的中断线。为了验证它，我们可以查看`/proc/interrupts`文件，其中包含内核中所有已注册的中断线。之前，在中断处理程序注册期间，我们在`request_irq()`函数中使用了`irqtest`标签，因此我们必须使用`grep`在文件中搜索它，如下所示：

```
# grep irqtest /proc/interrupts 
 40:     0     0     GPIO2   20   Edge   irqtest
```

好的。中断线 40 已分配给我们的模块，我们注意到这个 IRQ 线对应于 GPIO2 组的 GPIO 线 20（即 MPP2_20 线）。如果我们查看`/proc/interrupts`文件的开头，我们应该得到以下输出：

```
# head -4 /proc/interrupts 
           CPU0   CPU1 
  1:          0      0   GICv3   25   Level   vgic
  3:       5944  20941   GICv3   30   Level   arch_timer
  4:          0      0   GICv3   27   Level   kvm guest timer
...
```

第一个数字是中断线；第二个和第三个数字显示 CPU0 和 CPU1 分别服务了多少次中断，因此我们可以使用这些信息来验证哪个 CPU 服务了我们的中断。

好的。现在我们准备好了。只需将引脚 12 连接到 P8 扩展连接器的引脚 1；至少应该生成一个中断，并且内核消息中应该出现以下消息：

```
irqtest irqtest: interrupt occurred on IRQ 40
```

请注意，由于在短路操作期间，电信号可能会产生多次振荡，因此您可能会收到多条消息。

最后，让我们看看如果我们尝试使用 sysfs 接口导出编号 466 的 GPIO 线会发生什么，就像我们之前做的那样：

```
# echo 466 > /sys/class/gpio/export 
-bash: echo: write error: Device or resource busy
```

现在，由于内核在我们使用`devm_gpio_request()`函数时请求了这样一个 GPIO，我们正确地得到了一个忙碌错误。

# 另请参阅

+   有关中断处理程序的更多信息，一个很好的起点（即使它有点过时）是 Linux 内核模块编程指南，网址为[`www.tldp.org/LDP/lkmpg/2.4/html/x1210.html`](https://www.tldp.org/LDP/lkmpg/2.4/html/x1210.html)[.](https://www.tldp.org/LDP/lkmpg/2.4/html/x1210.html)

# 推迟工作

中断是由外围设备生成的事件，但正如前面所说的，它们并不是内核能够处理的唯一事件。事实上，还存在软件中断，类似于硬件中断，但是由软件生成。在本书中，我们将看到两个此类软件中断的示例；它们都可以用于安全地推迟将来的工作。我们还将看看设备驱动程序开发人员可以使用的一个有用机制，以捕获特殊的内核事件并根据情况执行操作（例如，当网络设备启用时，或系统正在重新启动等）。

在本教程中，我们将看到如何在内核中发生特定事件时推迟工作。

# 准备就绪

由于 tasklet 和 workqueue 是用来推迟工作的，它们的主要用途是在中断处理程序中，我们只需确认中断请求（通常命名为 IRQ），然后调用 tasklet/workqueue 完成工作。

但是，不要忘记这只是 tasklet 和工作队列的几种可能用法之一，当然，即使没有中断，也可以使用它们。

# 如何做...

在本节中，我们将使用针对先前的 `irqtest.c` 示例的补丁，展示关于 tasklet 和工作队列的简单示例。

在接下来的章节中，每当需要时，我们将展示这些机制的更复杂用法，但目前我们只关注理解它们的基本用法。

# Tasklets

让我们按照以下步骤来做：

1.  需要以下修改来将自定义 tasklet 调用添加到我们的 `irqtest_interrupt()` 中断处理程序中：

```
--- a/drivers/misc/irqtest.c
+++ b/drivers/misc/irqtest.c
@@ -26,9 +26,19 @@ static struct irqtest_data {
 } irqinfo;

 /*
- * The interrupt handler
+ * The interrupt handlers
  */

+static void irqtest_tasklet_handler(unsigned long flag)
+{
+     struct irqtest_data *info = (struct irqtest_data *) flag;
+     struct device *dev = info->dev;
+
+     dev_info(dev, "tasklet executed after IRQ %d", info->irq);
+}
+DECLARE_TASKLET(irqtest_tasklet, irqtest_tasklet_handler,
+                   (unsigned long) &irqinfo);
+
 static irqreturn_t irqtest_interrupt(int irq, void *dev_id)
 {
      struct irqtest_data *info = dev_id;
@@ -36,6 +46,8 @@ static irqreturn_t irqtest_interrupt(int irq, void *dev_id)

      dev_info(dev, "interrupt occurred on IRQ %d\n", irq);

+     tasklet_schedule(&irqtest_tasklet);
+
      return IRQ_HANDLED;
 }

@@ -98,6 +110,7 @@ static int irqtest_remove(struct platform_device *pdev)
 {
      struct device *dev = &pdev->dev;

+     tasklet_kill(&irqtest_tasklet);
      free_irq(irqinfo.irq, &irqinfo);
      dev_info(dev, "IRQ %d is now unmanaged!\n", irqinfo.irq);
```

前面的补丁可以在 GitHub 资源中的 `add_tasklet_to_irqtest_module.patch` 文件中找到，并且可以像往常一样应用。

**`patch -p1 < add_tasklet_to_irqtest_module.patch`** 命令。

1.  一旦 tasklet 被定义，就可以使用 `tasklet_schedule()` 函数来调用它，就像之前展示的那样。要停止它，我们可以使用 `tasklet_kill()` 函数，在我们的示例中用于 `irqtest_remove()` 函数来在从内核中卸载模块之前停止 tasklet。实际上，我们必须确保在卸载模块之前，我们的驱动程序之前分配和/或启用的每个资源都已被禁用和/或释放，否则可能会发生内存损坏。

请注意，`DECLARE_TASKLET()` 的编译时使用并不是声明 tasklet 的唯一方式。实际上，以下是另一种方式：

```
--- a/drivers/misc/irqtest.c
+++ b/drivers/misc/irqtest.c
@@ -23,12 +23,21 @@ static struct irqtest_data {
      int irq;
      unsigned int pin;
      struct device *dev;
+     struct tasklet_struct task;
 } irqinfo;

 /*
- * The interrupt handler
+ * The interrupt handlers
  */

+static void irqtest_tasklet_handler(unsigned long flag)
+{
+     struct irqtest_data *info = (struct irqtest_data *) flag;
+     struct device *dev = info->dev;
+
+     dev_info(dev, "tasklet executed after IRQ %d", info->irq);
+}
+
 static irqreturn_t irqtest_interrupt(int irq, void *dev_id)
 {
      struct irqtest_data *info = dev_id;
@@ -36,6 +45,8 @@ static irqreturn_t irqtest_interrupt(int irq, void *dev_id)

      dev_info(dev, "interrupt occurred on IRQ %d\n", irq);

+     tasklet_schedule(&info->task);
+
      return IRQ_HANDLED;
 }

@@ -80,6 +91,10 @@ static int irqtest_probe(struct platform_device *pdev)
      dev_info(dev, "GPIO %u correspond to IRQ %d\n",
                                irqinfo.pin, irqinfo.irq);
```

然后，我们创建我们的 tasklet 如下：

```
+     /* Create our tasklet */
+     tasklet_init(&irqinfo.task, irqtest_tasklet_handler,
+                               (unsigned long) &irqinfo);
+
      /* Request IRQ line and setup corresponding handler */
      irqinfo.dev = dev;
      ret = request_irq(irqinfo.irq, irqtest_interrupt, 0,
@@ -98,6 +113,7 @@ static int irqtest_remove(struct platform_device *pdev)
 {
      struct device *dev = &pdev->dev;

+     tasklet_kill(&irqinfo.task);
      free_irq(irqinfo.irq, &irqinfo);
      dev_info(dev, "IRQ %d is now unmanaged!\n", irqinfo.irq);
```

前面的补丁可以在 GitHub 资源中的 `add_tasklet_2_to_irqtest_module.patch` 文件中找到，并且可以像往常一样应用。

**`patch -p1 < add_tasklet_2_to_irqtest_module.patch`** 命令。

当我们必须在设备结构中嵌入 tasklet 并动态生成它时，这种第二种形式是有用的。

# 工作队列

现在让我们来看看工作队列。在下面的示例中，我们添加了一个自定义工作队列，由 `irqtest_wq` 指针引用，并命名为 `irqtest`，它执行两种不同的工作，由 `work` 和 `dwork` 结构描述：前者是正常工作，而后者代表延迟工作，即在经过一段时间延迟后执行的工作。

1.  首先，我们必须添加我们的数据结构：

```
a/drivers/misc/irqtest.c
+++ b/drivers/misc/irqtest.c
@@ -14,6 +14,7 @@
 #include <linux/gpio.h>
 #include <linux/irq.h>
 #include <linux/interrupt.h>
+#include <linux/workqueue.h>

 /*
  * Module data
@@ -23,12 +24,37 @@ static struct irqtest_data {
        int irq;
      unsigned int pin;
      struct device *dev;
+     struct work_struct work;
+     struct delayed_work dwork;
 } irqinfo;

+static struct workqueue_struct *irqtest_wq;
...
```

所有这些修改都可以在 GitHub 资源中的 `add_workqueue_to_irqtest_module.patch` 文件中找到，并且可以像往常一样应用。

**`patch -p1 < add_workqueue_to_irqtest_module.patch`** 命令。

1.  然后，我们必须创建工作队列并使其工作。对于工作队列的创建，我们可以使用 `create_singlethread_workqueue()` 函数，而两个工作可以通过使用 `INIT_WORK()` 和 `INIT_DELAYED_WORK()` 进行初始化，如下所示：

```
@@ -80,24 +108,40 @@ static int irqtest_probe(struct platform_device *pdev)
      dev_info(dev, "GPIO %u correspond to IRQ %d\n",
                                irqinfo.pin, irqinfo.irq);

+     /* Create our work queue and init works */
+     irqtest_wq = create_singlethread_workqueue("irqtest");
+     if (!irqtest_wq) {
+         dev_err(dev, "failed to create work queue!\n");
+         return -EINVAL;
+     }
+     INIT_WORK(&irqinfo.work, irqtest_work_handler);
+     INIT_DELAYED_WORK(&irqinfo.dwork, irqtest_dwork_handler);
+
      /* Request IRQ line and setup corresponding handler */
      irqinfo.dev = dev;
      ret = request_irq(irqinfo.irq, irqtest_interrupt, 0,
                                "irqtest", &irqinfo);
      if (ret) {
          dev_err(dev, "cannot register IRQ %d\n", irqinfo.irq);
-         return -EIO;
+         goto flush_wq;
      }
      dev_info(dev, "interrupt handler for IRQ %d is now ready!\n",
                                irqinfo.irq);

      return 0;
+
+flush_wq:
+     flush_workqueue(irqtest_wq);
+     return -EIO;
 }
```

要创建工作队列，我们也可以使用 `create_workqueue()` 函数；然而，这会创建一个在系统上每个处理器都有专用线程的工作队列。在许多情况下，所有这些线程都是多余的，使用 `create_singlethread_workqueue()` 获得的单个工作线程就足够了。

请注意，内核文档文件（`linux/Documentation/core-api/workqueue.rst`）中提供的并发管理工作队列 API 表明，`create_*workqueue()` 函数已被弃用并计划移除。然而，它们似乎仍然广泛用于内核源代码中。

1.  接下来是处理程序体，表示正常工作队列和延迟工作队列的有效工作负载，如下所示：

```
+static void irqtest_dwork_handler(struct work_struct *ptr)
+{
+     struct irqtest_data *info = container_of(ptr, struct irqtest_data,
+                                                   dwork.work);
+     struct device *dev = info->dev;
+
+     dev_info(dev, "delayed work executed after work");
+}
+
+static void irqtest_work_handler(struct work_struct *ptr)
+{
+     struct irqtest_data *info = container_of(ptr, struct irqtest_data,
+                                                   work);
+     struct device *dev = info->dev;
+
+     dev_info(dev, "work executed after IRQ %d", info->irq);
+
+     /* Schedule the delayed work after 2 seconds */
+     queue_delayed_work(irqtest_wq, &info->dwork, 2*HZ);
+}
```

请注意，为了指定两秒的延迟，我们使用了`2*HZ`代码，其中`HZ`是一个定义（有关`HZ`的更多信息，请参见下一节），表示需要多少个 jiffies 来组成一秒。因此，为了延迟两秒，我们必须将`HZ`乘以二。

1.  中断处理程序现在只使用以下`queue_work()`函数来在返回之前执行第一个工作队列：

```
@@ -36,6 +62,8 @@ static irqreturn_t irqtest_interrupt(int irq, void *dev_id)

      dev_info(dev, "interrupt occurred on IRQ %d\n", irq);

+     queue_work(irqtest_wq, &info->work);
+
      return IRQ_HANDLED;
 }
```

因此，当`irqtest_interrupt()`结束时，系统会调用`irqtest_work_handler()`，然后调用`irqtest_dwork_handler()`，使用`queue_delayed_work()`来延迟两秒。

1.  最后，对于任务队列，在退出模块之前，我们必须使用`cancel_work_sync()`取消所有工作和工作队列（如果已创建），对于延迟工作，使用`cancel_delayed_work_sync()`，以及（在我们的情况下）使用`flush_workqueue()`来停止`irqtest`工作队列：

```
 static int irqtest_remove(struct platform_device *pdev)
 {
      struct device *dev = &pdev->dev;

+     cancel_work_sync(&irqinfo.work);
+     cancel_delayed_work_sync(&irqinfo.dwork);
+     flush_workqueue(irqtest_wq);
      free_irq(irqinfo.irq, &irqinfo);
      dev_info(dev, "IRQ %d is now unmanaged!\n", irqinfo.irq);
```

# 还有更多...

我们可以通过测试示例来检查它的工作原理。因此，我们必须应用所需的补丁，然后重新编译内核，重新安装并重新启动 ESPRESSObin。

# 任务队列

要测试任务队列，我们可以像以前一样，将引脚 12 连接到扩展连接器 P8 的引脚 1。以下是我们应该收到的内核消息：

```
irqtest irqtest: interrupt occurred on IRQ 40
irqtest irqtest: tasklet executed after IRQ 40
```

如预期的那样，会生成一个中断，然后由硬件`irqtest_interrupt()`中断处理程序来管理，然后执行`irqtest_tasklet_handler()`任务处理程序。

# 工作队列

要测试工作队列，我们必须短接我们熟悉的引脚，然后应该有以下输出：

```
[ 33.113008] irqtest irqtest: interrupt occurred on IRQ 40
[ 33.115731] irqtest irqtest: work executed after IRQ 40
...
[ 33.514268] irqtest irqtest: interrupt occurred on IRQ 40
[ 33.516990] irqtest irqtest: work executed after IRQ 40
[ 33.533121] irqtest irqtest: interrupt occurred on IRQ 40
[ 33.535846] irqtest irqtest: work executed after IRQ 40
[ 35.138114] irqtest irqtest: delayed work executed after work
```

请注意，这次我没有删除内核消息的第一部分，以便查看时间，并更好地评估正常工作和延迟工作之间的延迟。

正如我们所看到的，一旦连接 ESPRESSObin 引脚，我们会有几个中断，然后是工作，但延迟的工作只执行一次。这是因为，即使安排了多次，只有第一次调用才会生效，因此我们可以看到延迟的工作最终在第一次`schedule_work()`调用后的 2.025106 秒后执行。这也意味着它实际上比所需和预期的两秒晚了 25.106 毫秒。这种明显的异常是由于当您要求内核安排一些工作在将来的某个时间点执行时，内核肯定会在未来的所需时间点安排您的工作，但它不会保证您会在那个时间点执行。它只会保证这样的工作不会在请求的截止日期之前执行。这种额外的随机延迟的长度取决于当时系统的工作负载水平。

# 另请参阅

+   关于任务队列，您可能希望查看[`www.kernel.org/doc/htmldocs/kernel-hacking/basics-softirqs.html.`](https://www.kernel.org/doc/htmldocs/kernel-hacking/basics-softirqs.html)

+   有关工作队列的更多信息，请访问[`www.kernel.org/doc/html/v4.15/core-api/workqueue.html`](https://www.kernel.org/doc/html/v4.15/core-api/workqueue.html)。

# 使用内核定时器管理时间

在设备驱动程序开发过程中，可能需要在特定时刻执行多次重复操作，或者需要延迟一些代码的执行。在这些情况下，内核定时器可以帮助设备驱动程序开发人员。

在本教程中，我们将看到如何使用内核定时器在明确定义的时间间隔内执行重复的工作，或者在明确定的时间间隔之后延迟工作。

# 准备工作

对于内核定时器的一个简单示例，我们仍然可以使用一个内核模块，在模块的初始化函数中定义一个内核定时器。

在 GitHub 资源的`chapter_05/timer`目录中，有两个关于**内核定时器**（**ktimer**）和**高分辨率定时器**（**hrtimer**）的简单示例，在接下来的章节中，我们将详细解释它们，首先从新的高分辨率实现开始，这应该是新驱动程序中首选的。还介绍了旧的 API 以完整图片。

# 如何做...

`hires_timer.c`文件的以下主要部分包含了有关高分辨率内核定时器的简单示例。

1.  让我们从文件的末尾开始，使用模块`init()`函数：

```
static int __init hires_timer_init(void)
{
    /* Set up hires timer delay */

    pr_info("delay is set to %dns\n", delay_ns);

    /* Setup and start the hires timer */
    hrtimer_init(&hires_tinfo.timer, CLOCK_MONOTONIC,
                HRTIMER_MODE_REL | HRTIMER_MODE_SOFT);
    hires_tinfo.timer.function = hires_timer_handler;
    hrtimer_start(&hires_tinfo.timer, ns_to_ktime(delay_ns),
                HRTIMER_MODE_REL | HRTIMER_MODE_SOFT);

    pr_info("hires timer module loaded\n");
    return 0;
}
```

让我们看看模块`exit()`函数的位置：

```
static void __exit hires_timer_exit(void)
{
    hrtimer_cancel(&hires_tinfo.timer);

    pr_info("hires timer module unloaded\n");
}

module_init(hires_timer_init);
module_exit(hires_timer_exit);
```

正如我们在模块`hires_timer_init()`初始化函数中所看到的，我们读取`delay_ns`参数，并且使用`hrtimer_init()`函数，首先通过指定一些特性来初始化定时器：

```
/* Initialize timers: */
extern void hrtimer_init(struct hrtimer *timer, clockid_t which_clock,
                         enum hrtimer_mode mode);
```

通过使用`which_clock`参数，我们要求内核使用特定的时钟。在我们的示例中，我们使用了`CLOCK_MONOTONIC`，这对于可靠的时间戳和准确测量短时间间隔非常有用（它从系统启动时间开始，但在挂起期间停止），但我们也可以使用其他值（请参阅`linux/include/uapi/linux/time.h`头文件以获取完整列表），例如：

+   +   `CLOCK_BOOTTIME`：这个时钟类似于`CLOCK_MONOTONIC`，但当系统进入挂起模式时不会停止。这对于需要与挂起操作同步的关键到期时间非常有用。

+   `CLOCK_REALTIME`：这个时钟使用相对于 1970 年开始的 UNIX 纪元时间，使用**协调世界时**（**UTC**），就像`gettimeofday()`在用户空间中一样。这用于所有需要在重启后持续存在的时间戳，因为它可能会由于闰秒更新，**网络时间协议（NTP）**调整以及来自用户空间的`settimeofday()`操作而向后跳跃。但是，这个时钟在设备驱动程序中很少使用。

+   `CLOCK_MONOTONIC_RAW`：类似于`CLOCK_MONOTONIC`，但以硬件时钟源的相同速率运行，不会对时钟漂移进行调整（例如 NTP）。这在设备驱动程序中也很少需要。

1.  在定时器初始化后，我们必须通过使用`function`指针来设置回调或处理函数，如下所示，我们已将`timer.function`设置为`hires_timer_handler`：

```
hires_tinfo.timer.function = hires_timer_handler;
```

这一次，`hires_tinfo`模块数据结构定义如下：

```
static struct hires_timer_data {
    struct hrtimer timer;
    unsigned int data;
} hires_tinfo;
```

1.  定时器初始化后，我们可以通过调用`hrtimer_start()`来启动它，在这里我们只需使用`ns_to_ktime()`这样的函数设置到期时间，以防我们有一个时间间隔，或者使用`ktime_set()`，以防我们有秒/纳秒值。

请参阅`linux/include/linux/ktime.h`头文件，了解更多`ktime*()`函数。

如果我们查看`linux/include/linux/hrtimer.h`文件，我们会发现启动高分辨率定时器的主要函数是`hrtimer_start_range_ns()`，而`hrtimer_start()`是该函数的一个特例，如下所示：

```
/* Basic timer operations: */
extern void hrtimer_start_range_ns(struct hrtimer *timer, ktime_t tim,
                           u64 range_ns, const enum hrtimer_mode mode);

/**
 * hrtimer_start - (re)start an hrtimer
 * @timer: the timer to be added
 * @tim: expiry time
 * @mode: timer mode: absolute (HRTIMER_MODE_ABS) or
 * relative (HRTIMER_MODE_REL), and pinned (HRTIMER_MODE_PINNED);
 * softirq based mode is considered for debug purpose only!
 */
static inline void hrtimer_start(struct hrtimer *timer, ktime_t tim,
                                 const enum hrtimer_mode mode)
{
    hrtimer_start_range_ns(timer, tim, 0, mode);
}
```

我们还发现`HRTIMER_MODE_SOFT`模式除了用于调试目的外，不应该使用。

通过使用`hrtimer_start_range_ns()`函数，我们允许`range_ns`时间差，这使得内核可以自由地安排实际的唤醒时间，以便既节能又性能友好。内核对到期时间加上时间差提供了正常的尽力而为的行为，但可能决定提前触发定时器，但不会早于`tim`到期时间。

1.  `hires_timer.c`文件中的`hires_timer_handler()`函数是回调函数的一个示例：

```
static enum hrtimer_restart hires_timer_handler(struct hrtimer *ptr)
{
    struct hires_timer_data *info = container_of(ptr,
                    struct hires_timer_data, timer);

    pr_info("kernel timer expired at %ld (data=%d)\n",
                jiffies, info->data++);

    /* Now forward the expiration time and ask to be rescheduled */
    hrtimer_forward_now(&info->timer, ns_to_ktime(delay_ns));
    return HRTIMER_RESTART;
}
```

通过使用`container_of()`操作符，我们可以获取指向我们的数据结构的指针（在示例中定义为`struct hires_timer_data`），然后，在完成工作后，我们调用`hrtimer_forward_now()`来设置新的到期时间，并通过返回`HRTIMER_RESTART`值，要求内核重新启动定时器。对于一次性定时器，我们可以返回`HRTIMER_NORESTART`。

1.  在模块退出时，在`hires_timer_exit()`函数中，我们必须使用`hrtimer_cancel()`函数等待定时器停止。等待定时器停止是非常重要的，因为定时器是异步事件，可能会发生我们在定时器回调执行时移除`struct hires_timer_data`模块释放结构，这可能导致严重的内存损坏！

请注意，同步是作为一个睡眠（或挂起）`进程`实现的，这意味着当我们处于中断上下文（硬或软）时，不能调用`hrtimer_cancel()`函数。然而，在这些情况下，我们可以使用`hrtimer_try_to_cancel()`，它只是在定时器正确停止（或根本不活动）时返回一个非负值。

# 它是如何工作的...

为了看看它是如何工作的，我们通过简单地编译代码然后将代码移动到我们的 ESPRESSObin 上来测试我们的代码。当一切就绪时，我们只需要将模块加载到内核中，如下所示：

```
# insmod hires_timer.ko
```

然后，在内核消息中，我们应该得到类似以下的内容：

```
[ 528.902156] hires_timer:hires_timer_init: delay is set to 1000000000ns
[ 528.911593] hires_timer:hires_timer_init: hires timer module loaded
```

在*步骤 1*、*2*和*3*中设置了定时器，我们知道它已经以一秒的延迟启动。

当定时器到期时，由于步骤 4，我们执行内核定时器的处理程序：

```

[ 529.911604] hires_timer:hires_timer_handler: kernel timer expired at 4295024749 (data=0)
[ 530.911602] hires_timer:hires_timer_handler: kernel timer expired at 4295024999 (data=1)
[ 531.911602] hires_timer:hires_timer_handler: kernel timer expired at 4295025249 (data=2)
[ 532.911602] hires_timer:hires_timer_handler: kernel timer expired at 4295025499 (data=3)
...
```

我留下了时间，这样你就能了解内核定时器的精度。

正如我们所看到的，到期时间非常准确（几微秒）。

现在，由于*步骤 5*，如果我们移除模块，定时器会停止，如下所示：

```
hires_timer:hires_timer_exit: hires timer module unloaded
```

# 还有更多...

为了完善你的理解，看一下传统内核定时器 API 可能会很有趣。

# 传统内核定时器

`ktimer.c`文件包含了传统内核定时器的一个简单示例。和往常一样，让我们从文件末尾开始，那里是模块`init()`和`exit()`函数所在的地方：

```
static int __init ktimer_init(void)
{
    /* Save kernel timer delay */
    ktinfo.delay_jiffies = msecs_to_jiffies(delay_ms);
    pr_info("delay is set to %dms (%ld jiffies)\n",
                delay_ms, ktinfo.delay_jiffies);

    /* Setup and start the kernel timer */
    timer_setup(&ktinfo.timer, ktimer_handler, 0); 
    mod_timer(&ktinfo.timer, jiffies + ktinfo.delay_jiffies);

    pr_info("kernel timer module loaded\n");
    return 0;
}

static void __exit ktimer_exit(void)
{
    del_timer_sync(&ktinfo.timer);

    pr_info("kernel timer module unloaded\n");
}
```

具有处理程序函数的模块数据结构如下：

```
static struct ktimer_data {
    struct timer_list timer;
    long delay_jiffies;
    unsigned int data;
} ktinfo;

...

static void ktimer_handler(struct timer_list *t)
{
    struct ktimer_data *info = from_timer(info, t, timer);

    pr_info("kernel timer expired at %ld (data=%d)\n",
                jiffies, info->data++);

    /* Reschedule kernel timer */
    mod_timer(&info->timer, jiffies + info->delay_jiffies);
}
```

正如我们所看到的，这个实现与高分辨率定时器非常相似。实际上，在`ktimer_init()`初始化函数中，我们读取模块的`delay_ms`参数，并通过使用`msecs_to_jiffies()`将其值转换为 jiffies，这是内核定时器的计量单位。（请记住，传统内核定时器的时间限制设置为一个 jiffy。）

然后，我们使用`timer_setup()`和`mod_timer()`函数分别设置内核定时器并启动它。`timer_setup()`函数接受三个参数：

```
/**
 * timer_setup - prepare a timer for first use
 * @timer: the timer in question
 * @callback: the function to call when timer expires
 * @flags: any TIMER_* flags
 *
 * Regular timer initialization should use either DEFINE_TIMER() above,
 * or timer_setup(). For timers on the stack, timer_setup_on_stack() must
 * be used and must be balanced with a call to destroy_timer_on_stack().
 */
#define timer_setup(timer, callback, flags) \
    __init_timer((timer), (callback), (flags))
```

`struct timer_list`类型的变量`timer`，一个`callback`（或处理程序）函数，以及一些标志（在`flags`变量中）可以用来指定我们内核定时器的一些特殊特性。为了让你了解可用标志及其含义，以下是`linux/include/linux/timer.h`文件中的一些标志定义：

```
/*
 * A deferrable timer will work normally when the system is busy, but
 * will not cause a CPU to come out of idle just to service it; instead,
 * the timer will be serviced when the CPU eventually wakes up with a
 * subsequent non-deferrable timer.
 *
 * An irqsafe timer is executed with IRQ disabled and it's safe to wait for
 * the completion of the running instance from IRQ handlers, for example,
 * by calling del_timer_sync().
 *
 * Note: The irq disabled callback execution is a special case for
 * workqueue locking issues. It's not meant for executing random crap
 * with interrupts disabled. Abuse is monitored!
 */
#define TIMER_CPUMASK     0x0003FFFF
#define TIMER_MIGRATING   0x00040000
#define TIMER_BASEMASK    (TIMER_CPUMASK | TIMER_MIGRATING)
#define TIMER_DEFERRABLE  0x00080000
#define TIMER_PINNED      0x00100000
#define TIMER_IRQSAFE     0x00200000
```

关于回调函数，让我们看一下我们示例中的`ktimer_handler()`：

```
static void ktimer_handler(struct timer_list *t)
{
    struct ktimer_data *info = from_timer(info, t, timer);

    pr_info("kernel timer expired at %ld (data=%d)\n",
                jiffies, info->data++);

    /* Reschedule kernel timer */
    mod_timer(&info->timer, jiffies + info->delay_jiffies);
}
```

通过使用`from_timer()`，我们可以获取到我们数据结构的指针（在示例中定义为`struct ktimer_data`），然后，在完成工作后，我们可以再次调用`mod_timer()`来重新安排新的定时器执行；否则，一切都会停止。

请注意，`from_timer()`函数仍然使用`container_of()`来完成其工作，如`linux/include/linux/timer.h`文件中的以下定义所示：

`#define from_timer(var, callback_timer, timer_fieldname) \`

`container_of(callback_timer, typeof(*var), timer_fieldname)`.

在模块退出时，在`ktimer_exit()`函数中，我们必须使用`del_timer_sync()`函数等待定时器停止。我们之前关于等待退出的陈述仍然有效，因此，要从中断上下文中停止内核定时器，我们可以使用`try_to_del_timer_sync()`，它只是在定时器正确停止时返回一个非负值。

为了测试我们的代码，我们只需要编译然后将其移动到我们的 ESPRESSObin，然后我们可以按照以下方式将模块加载到内核中：

```
# insmod ktimer.ko
```

然后，在内核消息中，我们应该得到类似这样的内容：

```
[ 122.174020] ktimer:ktimer_init: delay is set to 1000ms (250 jiffies)
[ 122.180519] ktimer:ktimer_init: kernel timer module loaded
[ 123.206222] ktimer:ktimer_handler: kernel timer expired at 4294923072 (data=0)
[ 124.230222] ktimer:ktimer_handler: kernel timer expired at 4294923328 (data=1)
[ 125.254218] ktimer:ktimer_handler: kernel timer expired at 4294923584 (data=2)
```

同样，我留下了时间，让你了解内核定时器的精度。

在这里，我们发现 1000 毫秒等于 250 个 jiffies；也就是说，1 个 jiffy 等于 4 毫秒，我们还可以看到定时器的处理程序大约每秒执行一次。（与 4 毫秒非常接近的抖动，即 1 个 jiffy。）

当我们移除模块时，定时器会停止，如下所示：

```
ktimer:ktimer_exit: kernel timer module unloaded
```

# 另请参阅

+   有关高分辨率内核定时器的有趣文档在内核源代码中的`linux/Documentation/timers/hrtimers.txt`。

# 等待事件

在前面的章节中，我们看到如何直接在处理程序中管理中断，或者通过使用任务队列、工作队列等来推迟中断活动。此外，我们还看到如何执行周期性操作或如何将操作延迟到未来；然而，设备驱动程序可能需要等待特定事件，例如等待某些数据、等待缓冲区变满，或者等待变量达到所需值。

请不要混淆之前看到的由通知程序管理的与特定驱动程序相关的内核相关事件，与通用事件。

当没有数据可以从外围设备中读取时，读取进程必须进入睡眠状态，然后在“数据准备就绪”事件到达时被唤醒。另一个例子是当我们启动一个复杂的作业并希望在完成时得到信号；在这种情况下，我们启动作业，然后进入睡眠状态，直到“作业完成”事件到达。所有这些任务都可以通过使用**等待队列**（waitqueues）或**完成**（仍然由等待队列实现）来完成。

等待队列（或完成）只是一个队列，其中一个或多个进程等待与队列相关的事件；当事件到达时，一个、多个或甚至所有睡眠进程都会被唤醒，以便让某人来管理它。在这个示例中，我们将学习如何使用等待队列。

# 准备工作

为了准备一个关于等待队列的简单示例，我们可以再次使用一个内核模块，在该模块的初始化函数中定义一个内核定时器，该定时器的任务是生成我们的事件，然后我们使用等待队列或完成来等待它。

在 GitHub 资源的`chapter_05/wait_event`目录中，有两个关于等待队列和完成的简单示例，然后在*工作原理...*部分，我们将详细解释它们。

# 如何做...

首先，让我们看一个关于等待队列用于等待“数据大于 5”事件的简单示例。

# 等待队列

以下是`waitqueue.c`文件的主要部分，其中包含有关等待队列的简单示例。

1.  再次从末尾开始，看一下模块的`init()`函数：

```
static int __init waitqueue_init(void)
{
    int ret;

    /* Save kernel timer delay */
    wqinfo.delay_jiffies = msecs_to_jiffies(delay_ms);
    pr_info("delay is set to %dms (%ld jiffies)\n",
                delay_ms, wqinfo.delay_jiffies);

    /* Init the wait queue */
    init_waitqueue_head(&wqinfo.waitq);

    /* Setup and start the kernel timer */
    timer_setup(&wqinfo.timer, ktimer_handler, 0);
    mod_timer(&wqinfo.timer, jiffies + wqinfo.delay_jiffies);
```

内核定时器启动后，我们可以使用`wait_event_interruptible()`函数在`wqinfo.waitq`等待队列上等待`wqinfo.data > 5`事件，如下所示：

```
    /* Wait for the wake up event... */
    ret = wait_event_interruptible(wqinfo.waitq, wqinfo.data > 5);
    if (ret < 0)
        goto exit;

    pr_info("got event data > 5\n");

    return 0;

exit:
    if (ret == -ERESTARTSYS)
        pr_info("interrupted by signal!\n");
    else
        pr_err("unable to wait for event\n");

    del_timer_sync(&wqinfo.timer);

    return ret;
}
```

1.  现在定义了数据结构，如下所示：

```
static struct ktimer_data {
    struct wait_queue_head waitq;
    struct timer_list timer;
    long delay_jiffies;
    unsigned int data;
} wqinfo;
```

1.  然而，在等待队列上发生任何操作之前，必须进行初始化，因此，在启动内核定时器之前，我们使用`init_waitqueue_head()`函数来正确设置存储在`struct ktimer_data`中的`struct wait_queue_head waitq`。

如果我们查看`linux/include/linux/wait.h`头文件，我们可以看到`wait_event_interruptible()`的工作原理：

```
/**
 * wait_event_interruptible - sleep until a condition gets true
 * @wq_head: the waitqueue to wait on
 * @condition: a C expression for the event to wait for
 *
 * The process is put to sleep (TASK_INTERRUPTIBLE) until the
 * @condition evaluates to true or a signal is received.
 * The @condition is checked each time the waitqueue @wq_head is woken up.
 *
 * wake_up() has to be called after changing any variable that could
 * change the result of the wait condition.
 *
 * The function will return -ERESTARTSYS if it was interrupted by a
 * signal and 0 if @condition evaluated to true.
 */
#define wait_event_interruptible(wq_head, condition) \
```

1.  要了解如何唤醒睡眠进程，我们应该考虑`waitqueue.c`文件中名为`ktimer_handler()`的内核定时器处理程序：

```
static void ktimer_handler(struct timer_list *t)
{
    struct ktimer_data *info = from_timer(info, t, timer);

    pr_info("kernel timer expired at %ld (data=%d)\n",
                jiffies, info->data++);

    /* Wake up all sleeping processes */
    wake_up_interruptible(&info->waitq);

    /* Reschedule kernel timer */
    mod_timer(&info->timer, jiffies + info->delay_jiffies);
}
```

# 完成

如果我们希望等待作业完成，我们仍然可以使用等待队列，但最好使用完成，因为它专门设计用于执行此类活动。以下是一个简单的示例，可以从 GitHub 关于竞赛的`completion.c`文件中检索到。

1.  首先，让我们看看模块`init()`和`exit()`函数：

```
static int __init completion_init(void)
{
    /* Save kernel timer delay */
    cinfo.delay_jiffies = msecs_to_jiffies(delay_ms);
    pr_info("delay is set to %dms (%ld jiffies)\n",
                delay_ms, cinfo.delay_jiffies);

    /* Init the wait queue */
    init_completion(&cinfo.done);

    /* Setup and start the kernel timer */
    timer_setup(&cinfo.timer, ktimer_handler, 0); 
    mod_timer(&cinfo.timer, jiffies + cinfo.delay_jiffies);

    /* Wait for completition... */
    wait_for_completion(&cinfo.done);

    pr_info("job done\n");

    return 0;
}

static void __exit completion_exit(void)
{
    del_timer_sync(&cinfo.timer);

    pr_info("module unloaded\n");
}
```

1.  现在模块的数据结构如下：

```
static struct ktimer_data {
    struct completion done;
    struct timer_list timer;
    long delay_jiffies;
    unsigned int data;
} cinfo;
```

1.  作业完成后，我们可以使用`complete()`函数向`ktimer_handler()`内核定时器处理程序发出完成信号：

```
static void ktimer_handler(struct timer_list *t)
{
    struct ktimer_data *info = from_timer(info, t, timer);

    pr_info("kernel timer expired at %ld (data=%d)\n",
                jiffies, info->data++);

    /* Signal that job is done */
    complete(&info->done);
}
```

当调用`complete()`时，等待完成的单个线程被通知：

```
/**
 * complete: - signals a single thread waiting on this completion
 * @x: holds the state of this particular completion
 *
 * This will wake up a single thread waiting on this completion. Threads will be
 * awakened in the same order in which they were queued.
 *
 * See also complete_all(), wait_for_completion() and related routines.
 *
 * It may be assumed that this function implies a write memory barrier before
 * changing the task state if and only if any tasks are woken up.
 */
void complete(struct completion *x)
```

而如果我们调用`complete_all()`，所有等待完成的线程都会被通知：

```
/**
 * complete_all: - signals all threads waiting on this completion
 * @x: holds the state of this particular completion
 *
 * This will wake up all threads waiting on this particular completion
 * event.
 * It may be assumed that this function implies a write memory barrier
 * before changing the task state if and only if any tasks are
 * woken up.
 * Since complete_all() sets the completion of @x permanently to done
 * to allow multiple waiters to finish, a call to reinit_completion()
 * must be used on @x if @x is to be used again. The code must make
 * sure that all waiters have woken and finished before reinitializing
 * @x. Also note that the function completion_done() can not be used
 * to know if there are still waiters after complete_all() has been
 * called.
 */
void complete_all(struct completion *x)
```

# 它是如何工作的...

让我们在接下来的几节中看看这是如何工作的：

# 等待队列

在步骤 3 中，如果条件为真，调用进程将继续执行；否则，它会进入睡眠状态，直到条件变为真或收到信号。（在这种情况下，函数返回`-ERESTARTSYS`值。）

为了完全理解，我们应该注意`linux/include/linux/wait.h`头文件中定义的另外两个等待事件函数的变体。第一个变体就是`wait_event()`函数，它的工作方式与`wait_event_interruptible()`完全相同，但它不能被任何信号中断：

```
/**
 * wait_event - sleep until a condition gets true
 * @wq_head: the waitqueue to wait on
 * @condition: a C expression for the event to wait for
 *
 * The process is put to sleep (TASK_UNINTERRUPTIBLE) until the
 * @condition evaluates to true. The @condition is checked each time
 * the waitqueue @wq_head is woken up.
 *
 * wake_up() has to be called after changing any variable that could
 * change the result of the wait condition.
 */
#define wait_event(wq_head, condition) \
```

而第二个是`wait_event_timeout()`或`wait_event_interruptible_timeout()`，它的工作方式与之前相同，直到超时为止：

```
/** * wait_event_interruptible_timeout - sleep until a condition
 *    gets true or a timeout elapses
 * @wq_head: the waitqueue to wait on
 * @condition: a C expression for the event to wait for
 * @timeout: timeout, in jiffies
 *
 * The process is put to sleep (TASK_INTERRUPTIBLE) until the
 * @condition evaluates to true or a signal is received.
 * The @condition is checked each time the waitqueue @wq_head
 * is woken up.
 * wake_up() has to be called after changing any variable that could
 * change the result of the wait condition.
 * Returns:
 * 0 if the @condition evaluated to %false after the @timeout elapsed,
 * 1 if the @condition evaluated to %true after the @timeout elapsed,
 * the remaining jiffies (at least 1) if the @condition evaluated
 * to %true before the @timeout elapsed, or -%ERESTARTSYS if it was
 * interrupted by a signal.
 */
#define wait_event_interruptible_timeout(wq_head, condition, timeout) \
```

在*步骤 4*中，在这个函数中，我们改变了存储在数据中的值，然后我们在等待队列上使用`wake_up_interruptible()`来通知一个正在睡眠的进程数据已经被改变，它应该醒来测试条件是否为真。

在`linux/include/linux/wait.h`头文件中，定义了几个函数，用于通过使用通用的`__wake_up()`函数唤醒一个、多个或所有等待进程（可中断或不可中断）：

```
#define wake_up(x)         __wake_up(x, TASK_NORMAL, 1, NULL)
#define wake_up_nr(x, nr)  __wake_up(x, TASK_NORMAL, nr, NULL)
#define wake_up_all(x)     __wake_up(x, TASK_NORMAL, 0, NULL)
...
#define wake_up_interruptible(x) __wake_up(x, TASK_INTERRUPTIBLE, 1, NULL)
#define wake_up_interruptible_nr(x, nr) __wake_up(x, TASK_INTERRUPTIBLE, nr, NULL)
#define wake_up_interruptible_all(x) __wake_up(x, TASK_INTERRUPTIBLE, 0, NULL)
...
```

在我们的示例中，我们要求数据大于五，所以前五次调用`wake_up_interruptible()`不应该唤醒我们的进程；让我们在下一节中验证一下！

请注意，将进入睡眠状态的进程只是`insmod`命令，它是调用模块初始化函数的命令。

# 完成

在*步骤 1*中，我们可以看到代码与之前的等待队列示例非常相似；我们只是使用`init_completion()`函数像往常一样初始化完成，然后在`struct ktimer_data`结构中的`struct completion done`上调用`wait_for_completion()`来等待作业结束。

至于等待队列，在`linux/include/linux/completion.h`头文件中，我们可以找到`wait_for_completion()`函数的几个变体：

```
extern void wait_for_completion(struct completion *);
extern int wait_for_completion_interruptible(struct completion *x);
extern unsigned long wait_for_completion_timeout(struct completion *x,
                                                   unsigned long timeout);
extern long wait_for_completion_interruptible_timeout(
        struct completion *x, unsigned long timeout);
```

# 还有更多...

现在，为了在两种情况下测试我们的代码，我们必须编译内核模块，然后将它们移动到 ESPRESSObin 上；此外，为了更好地理解示例的工作原理，我们应该使用 SSH 连接，然后从另一个终端窗口查看串行控制台上的内核消息。

# 等待队列

当我们使用`insmod`插入`waitqueue.ko`模块时，应该注意到该进程被挂起，直到数据变大于五为止：

```
# insmod waitqueue.ko
```

当`insmod`进程被挂起时，直到测试完成，你不应该得到提示。

在串行控制台上，我们应该收到以下消息：

```
waitqueue:waitqueue_init: delay is set to 1000ms (250 jiffies)
waitqueue:ktimer_handler: kernel timer expired at 4295371304 (data=0)
waitqueue:ktimer_handler: kernel timer expired at 4295371560 (data=1)
waitqueue:ktimer_handler: kernel timer expired at 4295371816 (data=2)
waitqueue:ktimer_handler: kernel timer expired at 4295372072 (data=3)
waitqueue:ktimer_handler: kernel timer expired at 4295372328 (data=4)
waitqueue:ktimer_handler: kernel timer expired at 4295372584 (data=5)
waitqueue:waitqueue_init: got event data > 5
waitqueue:ktimer_handler: kernel timer expired at 4295372840 (data=6)
...
```

一旦屏幕上显示了`got event data > 5`的消息，`insmod`进程应该返回，并且应该显示一个新的提示。

为了验证`wait_event_interruptible()`在信号到达时返回`-ERESTARTSYS`，我们可以卸载模块，然后重新加载它，然后在数据达到 5 之前按下*CTRL*+*C*键：

```
# rmmod waitqueue 
# insmod waitqueue.ko
^C
```

这次在内核消息中，我们应该得到类似以下的内容：

```
waitqueue:waitqueue_init: delay is set to 1000ms (250 jiffies)
waitqueue:ktimer_handler: kernel timer expired at 4295573632 (data=0)
waitqueue:ktimer_handler: kernel timer expired at 4295573888 (data=1)
waitqueue:waitqueue_init: interrupted by signal!
```

# 完成

要测试完成，我们必须将`completion.ko`模块插入内核。现在你应该注意到，如果我们按下*CTRL*+*C*，什么都不会发生，因为我们使用了`wait_for_completion()`而不是`wait_for_completion_interruptible()`：

```
# insmod completion.ko
^C^C^C^C
```

然后在五秒后提示返回，内核消息类似以下内容：

```
completion:completion_init: delay is set to 5000ms (1250 jiffies)
completion:ktimer_handler: kernel timer expired at 4296124608 (data=0)
completion:completion_init: job done
```

# 另请参阅

+   虽然有点过时，但在[`lwn.net/Articles/577370/`](https://lwn.net/Articles/577370/)的 URL 上有一些关于等待队列的好信息。

# 执行原子操作

原子操作在设备驱动程序开发中是至关重要的一步。事实上，驱动程序不像一个从头到尾执行的普通程序，因为它提供了多种方法（例如，读取或写入外围设备的数据，或设置一些通信参数），这些方法可以异步地相互调用。所有这些方法都同时在共同的数据结构上操作，这些数据结构必须以一致的方式进行修改。这就是为什么我们需要能够执行原子操作。

Linux 内核使用各种原子操作。每个操作用于不同的操作，取决于 CPU 是否在中断或进程上下文中运行。

当 CPU 处于进程上下文时，我们可以安全地使用**互斥锁**，如果互斥锁被锁定，可以使当前运行的进程进入睡眠状态；然而，在中断上下文中，“进入睡眠”是不允许的，因此我们需要另一种机制，Linux 给了我们**自旋锁**，它允许在任何地方进行锁定，但是时间很短。这是因为自旋锁通过在当前 CPU 上执行一个忙等待的紧密循环来完成工作，如果我们停留时间太长，就会损失性能。

在这个示例中，我们将看到如何以不可中断的方式对数据进行操作，以避免数据损坏。

# 准备就绪

同样，为了构建我们的示例，我们可以使用一个定义了内核定时器的内核模块，在模块`init()`函数中，它负责生成一个异步执行，我们可以在其中使用我们的互斥机制来保护我们的数据。

在 GitHub 资源的`chapter_05/atomic`目录中，有关互斥锁、自旋锁和原子数据的简单示例，在接下来的章节中，我们将详细解释它们。

# 如何做...

在本段中，我们将介绍两个如何使用互斥锁和自旋锁的示例。我们应该将它们视为如何使用 API 的演示，因为在真实的驱动程序中，它们的使用方式有些不同，并且将在第七章 *高级字符驱动程序操作*和接下来的章节中进行介绍。

# 互斥锁

以下是`mutex.c`文件的结尾，其中定义了互斥锁并为模块`init()`函数进行了初始化：

```
static int __init mut_init(void)
{
    /* Save kernel timer delay */
    minfo.delay_jiffies = msecs_to_jiffies(delay_ms);
    pr_info("delay is set to %dms (%ld jiffies)\n",
                delay_ms, minfo.delay_jiffies);

    /* Init the mutex */
    mutex_init(&minfo.lock);

    /* Setup and start the kernel timer */
    timer_setup(&minfo.timer, ktimer_handler, 0); 
    mod_timer(&minfo.timer, jiffies);

    mutex_lock(&minfo.lock);
    minfo.data++;
    mutex_unlock(&minfo.lock);

    pr_info("mutex module loaded\n");
    return 0;
}
```

以下是模块`exit()`函数的初始化：

```
static void __exit mut_exit(void)
{
    del_timer_sync(&minfo.timer);

    pr_info("mutex module unloaded\n");
}

module_init(mut_init);
module_exit(mut_exit);
```

1.  在模块初始化的`mut_init()`函数中，我们使用`mutex_init()`来初始化`lock`互斥锁；然后我们可以安全地启动定时器。

模块数据结构定义如下：

```
static struct ktimer_data {
    struct mutex lock;
    struct timer_list timer;
    long delay_jiffies;
    int data;
} minfo;
```

1.  我们使用`mutex_trylock()`来尝试安全地获取锁：

```
static void ktimer_handler(struct timer_list *t)
{
    struct ktimer_data *info = from_timer(info, t, timer);
    int ret;

    pr_info("kernel timer expired at %ld (data=%d)\n",
                jiffies, info->data);
    ret = mutex_trylock(&info->lock);
    if (ret) {
        info->data++;
        mutex_unlock(&info->lock);
    } else
        pr_err("cannot get the lock!\n");

    /* Reschedule kernel timer */
    mod_timer(&info->timer, jiffies + info->delay_jiffies);
}
```

# 自旋锁

1.  像往常一样，`spinlock.c`文件被用作自旋锁使用的示例。以下是模块`init()`函数：

```
static int __init spin_init(void)
{
    unsigned long flags;

    /* Save kernel timer delay */
    sinfo.delay_jiffies = msecs_to_jiffies(delay_ms);
    pr_info("delay is set to %dms (%ld jiffies)\n",
                delay_ms, sinfo.delay_jiffies);

    /* Init the spinlock */
    spin_lock_init(&sinfo.lock);

    /* Setup and start the kernel timer */
    timer_setup(&sinfo.timer, ktimer_handler, 0); 
    mod_timer(&sinfo.timer, jiffies);

    spin_lock_irqsave(&sinfo.lock, flags);
    sinfo.data++;
    spin_unlock_irqrestore(&sinfo.lock, flags);

    pr_info("spinlock module loaded\n");
    return 0;
}
```

以下是模块`exit()`函数：

```
static void __exit spin_exit(void)
{
    del_timer_sync(&sinfo.timer);

    pr_info("spinlock module unloaded\n");
}

module_init(spin_init);
module_exit(spin_exit);
```

模块数据结构如下：

```
static struct ktimer_data {
    struct spinlock lock;
    struct timer_list timer;
    long delay_jiffies;
    int data;
} sinfo;
```

1.  在这个示例中，我们使用`spin_lock_init()`来初始化自旋锁，然后我们使用两个不同的函数对来保护我们的数据：`spin_lock()`和`spin_unlock()`；这两者都使用自旋锁来避免竞争条件，而`spin_lock_irqsave()`和`spin_unlock_irqrestore()`在当前 CPU 中断被禁用时使用自旋锁：

```
static void ktimer_handler(struct timer_list *t)
{
    struct ktimer_data *info = from_timer(info, t, timer);

    pr_info("kernel timer expired at %ld (data=%d)\n",
                jiffies, info->data);
    spin_lock(&sinfo.lock);
    info->data++;
    spin_unlock(&info->lock);

    /* Reschedule kernel timer */
    mod_timer(&info->timer, jiffies + info->delay_jiffies);
}
```

通过使用`spin_lock_irqsave()`和`spin_unlock_irqrestore()`，我们可以确保没有人可以中断我们，因为 IRQ 被禁用，也没有其他 CPU 可以执行我们的代码（由于自旋锁）。

# 工作原理...

让我们看看互斥锁和自旋锁在接下来的两个部分中是如何工作的。

# 互斥锁

在*步骤 2*中，每次我们需要修改数据时，我们可以通过调用`mutex_lock()`和`mutex_unlock()`对其进行保护，将互斥锁的指针作为参数传递；当然，我们不能在中断上下文中执行此操作（如内核定时器处理程序），这就是为什么我们使用`mutex_trylock()`来尝试安全地获取锁。

# 自旋锁

在步骤 1 中，示例与之前的示例非常相似，但它展示了互斥锁和自旋锁之间一个非常重要的区别：前者保护代码免受进程的并发影响，而后者保护代码免受 CPU 的并发影响！实际上，如果内核没有对称多处理支持（在内核`.config`文件中`CONFIG_SMP=n`），那么自旋锁就会消失。

这是一个非常重要的概念，设备驱动程序开发人员应该非常了解；否则，驱动程序可能根本无法工作，或者导致严重的错误。

# 还有更多...

由于最后的示例只是为了展示互斥锁和自旋锁，API 测试是相当无用的。然而，如果我们仍然希望这样做，程序是一样的：编译模块，然后将它们移动到 ESPRESSObin。

# 互斥锁

当我们插入`mutex.ko`模块时，输出应该类似于以下内容：

```
# insmod mutex.ko 
mutex:mut_init: delay is set to 1000ms (250 jiffies)
mutex:mut_init: mutex module loaded
```

在步骤 1 中，我们执行模块`init()`函数，在其中增加了一个在互斥锁保护区域内的`minfo.data`。

```
mutex:ktimer_handler: kernel timer expired at 4294997916 (data=1)
mutex:ktimer_handler: kernel timer expired at 4294998168 (data=2)
mutex:ktimer_handler: kernel timer expired at 4294998424 (data=3)
...
```

当我们执行处理程序时，我们可以确保如果模块`init()`函数当前持有互斥锁，它就不能增加`minfo.data`。

# 自旋锁

当我们插入`spinlock.ko`模块时，输出应该类似于以下内容：

```
# insmod spinlock.ko 
spinlock:spin_init: delay is set to 1000ms (250 jiffies)
spinlock:spin_init: spinlock module loaded
```

与之前一样，在*步骤 1*中，我们执行模块`init()`函数，在其中增加了一个在自旋锁保护区域内的`minfo.data`。

```
spinlock:ktimer_handler: kernel timer expired at 4295019195 (data=1)
spinlock:ktimer_handler: kernel timer expired at 4295019448 (data=2)
spinlock:ktimer_handler: kernel timer expired at 4295019704 (data=3)
...
```

同样，在执行处理程序时，我们可以确保如果模块`init()`函数当前持有自旋锁，它就不能增加`minfo.data`。

请注意，在单核机器的情况下，自旋锁会消失，并且我们可以通过禁用中断来确保`minfo.data`的锁。

通过使用互斥锁和自旋锁，我们可以保护数据免受竞态条件的影响；然而，Linux 为我们提供了另一个 API，原子操作。

# 原子数据类型

在设备驱动程序开发过程中，我们可能需要以原子方式增加或减少一个变量，或者更简单地在一个变量中设置一个或多个位。为此，我们可以使用一组变量和操作，内核保证这些操作是原子的，而不是使用复杂的互斥机制。

在 GitHub 资源的`atomic.c`文件中，我们可以看到一个关于原子操作的简单示例，其中原子变量可以定义如下：

```
static atomic_t bitmap = ATOMIC_INIT(0xff);

static struct ktimer_data {
    struct timer_list timer;
    long delay_jiffies;
    atomic_t data;
} ainfo;
```

此外，以下是模块`init()`函数：

```
static int __init atom_init(void)
{
    /* Save kernel timer delay */
    ainfo.delay_jiffies = msecs_to_jiffies(delay_ms);
    pr_info("delay is set to %dms (%ld jiffies)\n",
                delay_ms, ainfo.delay_jiffies);

    /* Init the atomic data */
    atomic_set(&ainfo.data, 10);

    /* Setup and start the kernel timer after required delay */
    timer_setup(&ainfo.timer, ktimer_handler, 0); 
    mod_timer(&ainfo.timer, jiffies + ainfo.delay_jiffies);

    pr_info("data=%0x\n", atomic_read(&ainfo.data));
    pr_info("bitmap=%0x\n", atomic_fetch_and(0x0f, &bitmap));

    pr_info("atomic module loaded\n");
    return 0;
}
```

以下是模块`exit()`函数：

```
static void __exit atom_exit(void)
{
    del_timer_sync(&ainfo.timer);

    pr_info("atomic module unloaded\n");
}
```

在前面的代码中，我们使用`ATOMIC_INIT()`来静态定义和初始化原子变量，而`atomic_set()`函数可以用来动态地做同样的事情。随后，原子变量可以通过使用带有`atomic_*()`前缀的函数来进行操作，这些函数位于`linux/include/linux/atomic.h`和`linux/include/asm-generic/atomic.h`文件中。

最后，内核定时器处理程序可以实现如下：

```
static void ktimer_handler(struct timer_list *t)
{
    struct ktimer_data *info = from_timer(info, t, timer);

    pr_info("kernel timer expired at %ld (data=%d)\n",
                jiffies, atomic_dec_if_positive(&info->data));

    /* Compute an atomic bitmap operation */
    atomic_xor(0xff, &bitmap);
    pr_info("bitmap=%0x\n", atomic_read(&bitmap));

    /* Reschedule kernel timer */
    mod_timer(&info->timer, jiffies + info->delay_jiffies);
}
```

原子数据可以通过特定值进行加法或减法运算，增加、减少、或运算、与运算、异或运算等，所有这些操作都由内核保证是原子的，因此它们的使用非常简单。

同样，测试代码是相当无用的。然而，如果我们在 ESPRESSObin 中编译然后插入`atomic.ko`模块，输出如下：

```
# insmod atomic.ko 
atomic:atom_init: delay is set to 1000ms (250 jiffies)
atomic:atom_init: data=a
atomic:atom_init: bitmap=ff
atomic:atom_init: atomic module loaded
atomic:ktimer_handler: kernel timer expired at 4295049912 (data=9)
atomic:ktimer_handler: bitmap=f0
atomic:ktimer_handler: kernel timer expired at 4295050168 (data=8)
atomic:ktimer_handler: bitmap=f
...
atomic:ktimer_handler: kernel timer expired at 4295051960 (data=1)
atomic:ktimer_handler: bitmap=f0
atomic:ktimer_handler: kernel timer expired at 4295052216 (data=0)
atomic:ktimer_handler: bitmap=f
atomic:ktimer_handler: kernel timer expired at 4295052472 (data=-1)
```

此时，`data`保持在`-1`，不再减少。

# 另请参阅

+   有关内核锁定机制的几个示例，请参阅[`www.kernel.org/doc/htmldocs/kernel-locking/locks.html`](https://www.kernel.org/doc/htmldocs/kernel-locking/locks.html)。

+   有关原子操作的更多信息，请参阅[`www.kernel.org/doc/html/v4.12/core-api/atomic_ops.html`](https://www.kernel.org/doc/html/v4.12/core-api/atomic_ops.html)。
