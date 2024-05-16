# 第十章：附加信息：管理中断和并发

回顾我们在[第三章](https://cdp.packtpub.com/linux_device_driver_development_cookbook/wp-admin/post.php?post=28&action=edit#post_26)中所做的工作，即*使用 Char 驱动程序*，当我们讨论`read()`系统调用以及如何为我们的 char 驱动程序实现它时（请参阅 GitHub 上的`chapter_4/chrdev/chrdev.c`文件），我们注意到我们的实现很棘手，因为数据总是可用的：

```
static ssize_t chrdev_read(struct file *filp,
               char __user *buf, size_t count, loff_t *ppos)
{
    struct chrdev_device *chrdev = filp->private_data;
    int ret;

    dev_info(chrdev->dev, "should read %ld bytes (*ppos=%lld)\n",
                count, *ppos);

    /* Check for end-of-buffer */
    if (*ppos + count >= BUF_LEN)
        count = BUF_LEN - *ppos;

    /* Return data to the user space */
    ret = copy_to_user(buf, chrdev->buf + *ppos, count);
    if (ret < 0)
        return ret;

    *ppos += count;
    dev_info(chrdev->dev, "return %ld bytes (*ppos=%lld)\n", count, *ppos);

    return count;
}
```

在前面的示例中，`chrdev->buf`中的数据总是存在的，但在真实的外围设备中，这往往并不是真的；我们通常必须等待新数据，然后当前进程应该被挂起（即*休眠*）。这就是为什么我们的`chrdev_read()`应该是这样的：

```
static ssize_t chrdev_read(struct file *filp,
               char __user *buf, size_t count, loff_t *ppos)
{
    struct chrdev_device *chrdev = filp->private_data;
    int ret;

    /* Wait for available data */
    wait_for_event(chrdev->available > 0);

    /* Check for end-of-buffer */
    if (count > chrdev->available)
        count = chrdev->available;

    /* Return data to the user space */
    ret = copy_to_user(buf, ..., count);
    if (ret < 0)
        return ret;

    *ppos += count;

    return count;
}
```

请注意，由于一个真实（完整的）`read()`系统调用实现将在第七章中呈现，所以本示例故意不完整。在本章中，我们只是介绍机制，而不是如何在设备驱动程序中使用它们。

通过使用`wait_for_event()`函数，我们要求内核测试是否有一些可用数据，如果有的话，允许进程执行，否则，当前进程将被挂起，一旦条件`chrdev->available > 0`为真，就会再次唤醒。

外围设备通常使用中断来通知 CPU 有新数据可用（或者必须对它们进行一些重要的活动），因此很明显，我们作为设备驱动程序开发人员，必须在中断处理程序中通知内核，等待数据的睡眠进程应该被唤醒。在接下来的章节中，我们将通过使用非常简单的示例来看看内核中有哪些机制可用，并且它们如何被用来挂起一个进程，我们还将看到什么时候可以安全地这样做！事实上，如果我们要求调度程序在中断处理程序中将 CPU 撤销给当前进程以便将其分配给另一个进程，那么我们只是在进行一个无意义的操作。当我们处于中断上下文时，我们并不执行进程代码，那么我们可以撤销 CPU 给哪个进程呢？简而言之，当 CPU 处于进程上下文时，执行进程可以*进入睡眠*，而当我们处于中断上下文时，我们不能这样做，因为当前没有进程正式持有 CPU！

这个概念非常重要，设备驱动程序开发人员必须充分理解；事实上，如果我们尝试在 CPU 处于中断上下文时进入睡眠状态，那么将会生成一个严重的异常，并且很可能整个系统都会挂起。

另一个需要真正清楚的重要概念是**原子操作**。设备驱动程序不是一个有常规开始和结束的正常程序；相反，设备驱动程序是一组可以同时运行的方法和异步中断处理程序。这就是为什么我们很可能必须保护我们的数据，以防可能损坏它们的竞争条件。

例如，如果我们使用缓冲区仅保存来自外围设备的接收数据，我们必须确保数据被正确排队，以便读取进程可以读取有效数据，而且不会丢失任何信息。因此，在这些情况下，我们应该使用一些 Linux 提供给我们的互斥机制来完成我们的工作。然而，我们必须注意我们所做的事情，因为其中一些机制可以在进程或中断上下文中安全使用，而另一些则不行；其中一些只能在进程上下文中使用，如果我们在中断上下文中使用它们，可能会损坏我们的系统。

此外，我们应该考虑到现代 CPU 有多个核心，因此使用禁用 CPU 中断的技巧来获得原子代码根本行不通，必须使用特定的互斥机制。在 Linux 中，这种机制称为**自旋锁**，它可以在中断或进程上下文中使用，但是只能用于非常短的时间，因为它们是使用忙等待方法实现的。这意味着为了执行原子操作，当一个核心在属于这种原子操作的关键代码部分中操作时，CPU 中的所有其他核心都被排除在同一关键部分之外，通过在紧密循环中积极旋转来等待，这反过来意味着你实际上在浪费 CPU 的周期，这些周期没有做任何有用的事情。

在接下来的章节中，我们将详细讨论所有这些方面，并尝试用非常简单的例子解释它们的用法；在[第七章](https://cdp.packtpub.com/linux_device_driver_development_cookbook/wp-admin/post.php?post=28&action=edit#post_30)，*高级字符驱动程序操作*中，我们将看到如何在设备驱动程序中使用这些机制。

# 推迟工作

很久以前，有**底半部**，也就是说，硬件事件被分成两半：顶半部（硬件中断处理程序）和底半部（软件中断处理程序）。这是因为中断处理程序必须尽快执行，以准备为下一个传入的中断提供服务，因此，例如，CPU 不能在中断处理程序的主体中等待慢速外围设备发送或接收数据的时间太长。这就是为什么我们使用底半部；中断被分成两部分：顶部是真正的硬件中断处理程序，它快速执行并禁用中断，只是确认外围设备，然后启动一个底半部，启用中断，可以安全地完成发送/接收工作。

然而，底半部非常有限，因此内核开发人员在 Linux 2.4 系列中引入了**tasklets**。Tasklets 允许以非常简单的方式动态创建可延迟的函数；它们在软件中断上下文中执行，适合快速执行，因为它们不能休眠。但是，如果我们需要休眠，我们必须使用另一种机制。在 Linux 2.6 系列中，**workqueues**被引入作为 Linux 2.4 系列中已经存在的类似构造称为 taskqueue 的替代品；它们允许内核函数像 tasklets 一样被激活（或延迟）以供以后执行，但是与 tasklets（在软件中断中执行）相比，它们在称为**worker threads**的特殊内核线程中执行。这意味着两者都可以用于推迟工作，但是 workqueue 处理程序可以休眠。当然，这个处理程序的延迟更高，但是相比之下，workqueues 包括更丰富的工作推迟 API。

在结束本食谱之前，还有两个重要的概念要谈论：共享工作队列和`container_of()`宏。

# 共享工作队列

在食谱中的前面的例子可以通过使用**共享工作队列**来简化。这是内核本身定义的一个特殊工作队列，如果设备驱动程序（和其他内核实体）*承诺*不会长时间垄断队列（也就是说不会长时间休眠和不会长时间运行的任务），如果它们接受它们的处理程序可能需要更长时间来获得公平的 CPU 份额。如果两个条件都满足，我们可以避免使用`create_singlethread_workqueue()`创建自定义工作队列，并且可以通过简单地使用`schedule_work()`和`schedule_delayed_work()`来安排工作。以下是处理程序：

```
--- a/drivers/misc/irqtest.c
+++ b/drivers/misc/irqtest.c
...
+static void irqtest_work_handler(struct work_struct *ptr)
+{
+     struct irqtest_data *info = container_of(ptr, struct irqtest_data,
+                                                      work);
+     struct device *dev = info->dev;
+
+     dev_info(dev, "work executed after IRQ %d", info->irq);
+
+     /* Schedule the delayed work after 2 seconds */
+     schedule_delayed_work(&info->dwork, 2*HZ);
+}
+
 static irqreturn_t irqtest_interrupt(int irq, void *dev_id)
 {
      struct irqtest_data *info = dev_id;
@@ -36,6 +60,8 @@ static irqreturn_t irqtest_interrupt(int irq, void *dev_id)

      dev_info(dev, "interrupt occurred on IRQ %d\n", irq);

+     schedule_work(&info->work);
+
      return IRQ_HANDLED;
 }
```

然后，初始化和移除的修改：

```
@@ -80,6 +106,10 @@ static int irqtest_probe(struct platform_device *pdev)
      dev_info(dev, "GPIO %u correspond to IRQ %d\n",
                                irqinfo.pin, irqinfo.irq);

+     /* Init works */
+     INIT_WORK(&irqinfo.work, irqtest_work_handler);
+     INIT_DELAYED_WORK(&irqinfo.dwork, irqtest_dwork_handler);
+
      /* Request IRQ line and setup corresponding handler */
      irqinfo.dev = dev;
      ret = request_irq(irqinfo.irq, irqtest_interrupt, 0,
@@ -98,6 +128,8 @@ static int irqtest_remove(struct platform_device *pdev)
 {
        struct device *dev = &pdev->dev;

+     cancel_work_sync(&irqinfo.work);
+     cancel_delayed_work_sync(&irqinfo.dwork);
      free_irq(irqinfo.irq, &irqinfo);
      dev_info(dev, "IRQ %d is now unmanaged!\n", irqinfo.irq);
```

前面的补丁可以在 GitHub 存储库的`add_workqueue_2_to_irqtest_module.patch`文件中找到，并且可以使用以下命令像往常一样应用：

**`$ patch -p1 < add_workqueue_2_to_irqtest_module.patch`**

# `container_of()`宏

最后，我们应该利用一些词来解释一下`container_of()`宏。该宏在`linux/include/linux/kernel.h`中定义如下：

```
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr: the pointer to the member.
 * @type: the type of the container struct this is embedded in.
 * @member: the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({ \
    void *__mptr = (void *)(ptr); \
    BUILD_BUG_ON_MSG(!__same_type(*(ptr), ((type *)0)->member) && \
                     !__same_type(*(ptr), void), \
                     "pointer type mismatch in container_of()"); \
    ((type *)(__mptr - offsetof(type, member))); })
```

`container_of()`函数接受三个参数：一个指针`ptr`，容器的`type`，以及指针在容器内引用的`member`的名称。通过使用这些信息，宏可以扩展为指向包含结构的新地址，该结构容纳了相应的成员。

因此，在我们的示例中，在`irqtest_work_handler()`中，我们可以获得一个指向`struct irqtest_data`的指针，以告诉`container_of()`其成员`work`的地址。

有关`container_of()`函数的更多信息，可以在互联网上找到；但是，一个很好的起点是内核源代码中的`linux/Documentation/driver-model/design-patterns.txt`文件，该文件描述了在使用此宏的设备驱动程序中发现的一些常见设计模式。

可能有兴趣看一下**通知器链**，简称**通知器**，它是内核提供的一种通用机制，旨在为内核元素提供一种表达对一般**异步**事件发生感兴趣的方式。

# 通知器

通知器机制的基本构建块是在`linux/include/linux/notifier.h`头文件中定义的`struct notifier_block`，如下所示：

```
typedef int (*notifier_fn_t)(struct notifier_block *nb,
                        unsigned long action, void *data);

struct notifier_block {
    notifier_fn_t notifier_call;
    struct notifier_block __rcu *next;
    int priority;
};
```

该结构包含指向发生事件时要调用的函数的指针`notifier_call`。当调用通知器函数时传递的参数包括指向通知器块本身的`nb`指针，一个依赖于特定使用链的事件`action`代码，以及指向未指定私有数据类型的`data`指针，该类型可以以与 tasklets 或 waitqueues 类似的方式使用。

`next`字段由通知器内部管理，而`priority`字段定义了在通知器链中由`notifier_call`指向的函数的优先级。首先执行具有更高优先级的函数。实际上，几乎所有注册都将优先级留给通知器块定义之外，这意味着它以 0 作为默认值，并且执行顺序最终取决于注册顺序（这是一种半随机顺序）。

设备驱动程序开发人员不应该需要创建自己的通知器，而且很多时候他们需要使用现有的通知器。Linux 定义了几个通知器，如下所示：

+   网络设备通知器（参见`linux/include/linux/netdevice.h`）-报告网络设备的事件

+   背光通知器（参见`linux/include/linux/backlight.h`）-报告 LCD 背光事件

+   挂起通知器（参见`linux/include/linux/suspend.h`）-报告挂起和恢复相关事件的电源

+   重启通知器（参见`linux/include/linux/reboot.h`）-报告重启请求

+   电源供应通知器（参见`linux/include/linux/power_supply.h`）-报告电源供应活动

每个通知器都有一个注册函数，可以用来要求系统在特定事件发生时通知。例如，以下代码被报告为请求网络设备和重启事件的有用示例：

```
static int __init notifier_init(void)
{
    int ret;

    ninfo.netdevice_nb.notifier_call = netdevice_notifier;
    ninfo.netdevice_nb.priority = 10; 

    ret = register_netdevice_notifier(&ninfo.netdevice_nb);
    if (ret) {
        pr_err("unable to register netdevice notifier\n");
        return ret;
    }

    ninfo.reboot_nb.notifier_call = reboot_notifier;
    ninfo.reboot_nb.priority = 10; 

    ret = register_reboot_notifier(&ninfo.reboot_nb);
    if (ret) {
        pr_err("unable to register reboot notifier\n");
        goto unregister_netdevice;
    }

    pr_info("notifier module loaded\n");

    return 0;

unregister_netdevice:
    unregister_netdevice_notifier(&ninfo.netdevice_nb);
    return ret;
}

static void __exit notifier_exit(void)
{
    unregister_netdevice_notifier(&ninfo.netdevice_nb);
    unregister_reboot_notifier(&ninfo.reboot_nb);

    pr_info("notifier module unloaded\n");
}
```

这里呈现的所有代码都在 GitHub 存储库中的`notifier.c`文件中。

`register_netdevice_notifier()`和`register_reboot_notifier()`函数都使用以下定义的两个 struct notifier_block：

```
static struct notifier_data {
    struct notifier_block netdevice_nb;
    struct notifier_block reboot_nb;
    unsigned int data;
} ninfo;
```

通知器函数的定义如下：

```
static int netdevice_notifier(struct notifier_block *nb,
                              unsigned long code, void *unused)
{
    struct notifier_data *ninfo = container_of(nb, struct notifier_data,
                                               netdevice_nb);

    pr_info("netdevice: event #%d with code 0x%lx caught!\n",
                    ninfo->data++, code);

    return NOTIFY_DONE;
}

static int reboot_notifier(struct notifier_block *nb,
                           unsigned long code, void *unused)
{ 
    struct notifier_data *ninfo = container_of(nb, struct notifier_data,
                                               reboot_nb);

    pr_info("reboot: event #%d with code 0x%lx caught!\n",
                    ninfo->data++, code);

    return NOTIFY_DONE;
}
```

通过使用`container_of()`，像往常一样，我们可以获得指向我们的数据结构`struct notifier_data`的指针；然后，一旦我们的工作完成，我们必须返回在`linux/include/linux/notifier.h`头文件中定义的一个固定值：

```
#define NOTIFY_DONE       0x0000                     /* Don't care */
#define NOTIFY_OK         0x0001                     /* Suits me */
#define NOTIFY_STOP_MASK  0x8000                     /* Don't call further */
#define NOTIFY_BAD        (NOTIFY_STOP_MASK|0x0002)  /* Bad/Veto action */
```

它们的含义如下：

+   `NOTIFY_DONE`：对此通知不感兴趣。

+   `NOTIFY_OK`：通知已正确处理。

+   `NOTIFY_BAD`：此通知出现问题，因此停止调用此事件的回调函数！

`NOTIFY_STOP_MASK`可以用于封装（负）`errno`值，如下所示：

```
/* Encapsulate (negative) errno value (in particular, NOTIFY_BAD <=> EPERM). */
static inline int notifier_from_errno(int err)
{
    if (err)
        return NOTIFY_STOP_MASK | (NOTIFY_OK - err);

    return NOTIFY_OK;
}
```

然后可以使用`notifier_to_errno()`检索`errno`值，如下所示：

```
/* Restore (negative) errno value from notify return value. */
static inline int notifier_to_errno(int ret)
{
    ret &= ~NOTIFY_STOP_MASK;
    return ret > NOTIFY_OK ? NOTIFY_OK - ret : 0;
}
```

要测试我们的简单示例，我们必须编译`notifier.c`内核模块，然后将`notifier.ko`模块移动到 ESPRESSObin，然后可以将其插入内核，如下所示：

```
# insmod notifier.ko 
notifier:netdevice_notifier: netdevice: event #0 with code 0x5 caught!
notifier:netdevice_notifier: netdevice: event #1 with code 0x1 caught!
notifier:netdevice_notifier: netdevice: event #2 with code 0x5 caught!
notifier:netdevice_notifier: netdevice: event #3 with code 0x5 caught!
notifier:netdevice_notifier: netdevice: event #4 with code 0x5 caught!
notifier:netdevice_notifier: netdevice: event #5 with code 0x5 caught!
notifier:notifier_init: notifier module loaded
```

插入后，已经通知了一些事件；但是，为了生成新事件，我们可以尝试使用以下`ip`命令禁用或启用网络设备：

```
# ip link set lan0 up
notifier:netdevice_notifier: netdevice: event #6 with code 0xd caught!
RTNETLINK answers: Network is down
```

代码`0xd`对应于`linux/include/linux/netdevice.h`中定义的`NETDEV_PRE_UP`事件：

```
/* netdevice notifier chain. Please remember to update netdev_cmd_to_name()
 * and the rtnetlink notification exclusion list in rtnetlink_event() when
 * adding new types.
 */
enum netdev_cmd {
    NETDEV_UP = 1, /* For now you can't veto a device up/down */
    NETDEV_DOWN,
    NETDEV_REBOOT, /* Tell a protocol stack a network interface
                      detected a hardware crash and restarted
                      - we can use this eg to kick tcp sessions
                      once done */
    NETDEV_CHANGE, /* Notify device state change */
    NETDEV_REGISTER,
    NETDEV_UNREGISTER,
    NETDEV_CHANGEMTU, /* notify after mtu change happened */
    NETDEV_CHANGEADDR,
    NETDEV_GOING_DOWN,
    NETDEV_CHANGENAME,
    NETDEV_FEAT_CHANGE,
    NETDEV_BONDING_FAILOVER,
    NETDEV_PRE_UP,
...
```

如果我们重新启动系统，我们应该在内核消息中看到以下消息：

```
# reboot
...
[ 2804.502671] notifier:reboot_notifier: reboot: event #7 with code 1 caught!
```

# 内核定时器

**内核定时器**是请求内核在经过明确定义的时间后执行特定函数的简单方法。 Linux 实现了两种不同类型的内核定时器：在`linux/include/linux/timer.h`头文件中定义的旧但仍然有效的内核定时器和在`linux/include/linux/hrtimer.h`头文件中定义的新的**高分辨率**内核定时器。即使它们实现方式不同，但两种机制的工作方式非常相似：我们必须声明一个保存定时器数据的结构，可以通过适当的函数进行初始化，然后可以使用适当的函数启动定时器。一旦到期，定时器调用处理程序执行所需的操作，最终，我们有可能停止或重新启动定时器。

传统内核定时器仅支持 1 个 jiffy 的分辨率。 jiffy 的长度取决于 Linux 内核中定义的`HZ`的值（请参阅`linux/include/asm-generic/param.h`文件）；通常在 PC 和其他一些平台上为 1 毫秒，在大多数嵌入式平台上设置为 10 毫秒。过去，1 毫秒的分辨率解决了大多数设备驱动程序开发人员的问题，但是现在，大多数外围设备需要更高的分辨率才能得到正确管理。这就是为什么需要更高分辨率的定时器，允许系统在更准确的时间间隔内快速唤醒和处理数据。目前，内核定时器已被高分辨率定时器所取代（即使它们仍然在内核源代码周围使用），其目标是在 Linux 中实现 POSIX 1003.1b 第十四部分（时钟和定时器）API，即精度优于 1 个 jiffy 的定时器。

请注意，我们刚刚看到，为了延迟作业，我们还可以使用延迟工作队列。
