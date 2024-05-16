# 第二章：*第二章*：利用 Regmap API 并简化代码

本章介绍了 Linux 内核寄存器映射抽象层，并展示了如何简化和委托 I/O 操作给 regmap 子系统。处理设备，无论是在 SoC 中内置的（也称为 MMIO）还是位于 I2C/SPI 总线上，都包括访问（读取/修改/更新）寄存器。Regmap 是必需的，因为许多设备驱动程序在其寄存器访问例程中使用了开放编码。**Regmap**代表**寄存器映射**。它最初是为了**ALSA SoC**（**ASoC**）而开发的，以消除编解码器驱动程序中多余的开放编码 SPI/I2C 寄存器访问例程。最初，regmap 提供了一组用于读取/写入非内存映射 I/O（例如，I2C 和 SPI 读/写）的 API。从那时起，MMIO regmap 已经升级，以便我们可以使用 regmap 来访问 MMIO。

如今，该框架抽象了 I2C，SPI 和 MMIO 寄存器访问，不仅在必要时处理锁定，还管理寄存器缓存，以及寄存器的可读性和可写性。它还处理 IRQ 芯片和 IRQ。本章将讨论 regmap，并解释如何使用它来抽象 I2C，SPI 和 MMIO 设备的寄存器访问。我们还将描述如何使用 regmap 来管理 IRQ 和 IRQ 控制器。

本章将涵盖以下主题：

+   Regmap 和其数据结构的介绍：I2C，SPI 和 MMIO

+   Regmap 和 IRQ 管理

+   Regmap IRQ API 和数据结构

# 技术要求

为了在阅读本章时感到舒适，您需要以下内容：

+   良好的 C 编程技能

+   熟悉设备树的概念

+   Linux 内核 v4.19.X 源代码，可在[`git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags)上获得

# Regmap 和其数据结构的介绍- I2C，SPI 和 MMIO

Regmap 是 Linux 内核提供的抽象寄存器访问机制，主要针对 SPI，I2C 和内存映射寄存器。

此框架中的 API 是总线不可知的，并在幕后处理底层配置。也就是说，该框架中的主要数据结构是`struct regmap_config`，在内核源代码树中的`include/linux/regmap.h`中定义如下：

```
struct regmap_config {
   const char *name;
   int reg_bits;
   int reg_stride;
   int pad_bits;
   int val_bits;
   bool (*writeable_reg)(struct device *dev, unsigned int reg);
   bool (*readable_reg)(struct device *dev, unsigned int reg);
   bool (*volatile_reg)(struct device *dev, unsigned int reg);
   bool (*precious_reg)(struct device *dev, unsigned int reg);
   int (*reg_read)(void *context, unsigned int reg,                   unsigned int *val);
   int (*reg_write)(void *context, unsigned int reg,                    unsigned int val);
   bool disable_locking;
   regmap_lock lock;
   regmap_unlock unlock;
   void *lock_arg;
   bool fast_io;
   unsigned int max_register;
   const struct regmap_access_table *wr_table;
   const struct regmap_access_table *rd_table;
   const struct regmap_access_table *volatile_table;
   const struct regmap_access_table *precious_table;
   const struct reg_default *reg_defaults;
   unsigned int num_reg_defaults;
   unsigned long read_flag_mask;
   unsigned long write_flag_mask;
   enum regcache_type cache_type;
   bool use_single_rw;
   bool can_multi_write;
};
```

为简单起见，本结构中的一些字段已被删除，在本章中不讨论。只要`struct regmap_config`正确完成，用户可以忽略底层总线机制。让我们介绍这个数据结构中的字段：

+   `reg_bits`表示寄存器的位数。换句话说，它是寄存器地址的位数。

+   `reg_stride`是寄存器地址的步幅。如果寄存器地址是该值的倍数，则为有效。如果设置为`0`，则将使用`1`的值，这意味着任何地址都是有效的。对不是该值的倍数的地址进行读/写将返回`-EINVAL`。

+   `pad_bits`是寄存器和值之间填充位的数量。这是在格式化时将寄存器的值左移的位数。

+   `val_bits`：表示用于存储寄存器值的位数。这是一个强制性字段。

+   `writeable_reg`：如果提供，将在每次 regmap 写操作时调用此可选回调函数，以检查给定地址是否可写。如果此函数在给定给 regmap 写事务的地址上返回`false`，则事务将返回`-EIO`。以下摘录显示了如何实现此回调：

```
static bool foo_writeable_register(struct device *dev,                                    unsigned int reg)
{
    switch (reg) {
    case 0x30 ... 0x38:
    case 0x40 ... 0x45:
    case 0x50 ... 0x57:
    case 0x60 ... 0x6e:
    case 0xb0 ... 0xb2:
        return true;
    default:
        return false;
    }
}
```

+   `readable_reg`：这与`writeable_reg`相同，但用于寄存器读取操作。

+   `volatile_reg`: 这是一个可选的回调，如果提供，将在每次需要通过 regmap 缓存读取或写入寄存器时调用。如果寄存器是易失性的（寄存器值无法被缓存），则该函数应返回`true`。然后在寄存器上执行直接读/写操作。如果返回`false`，表示寄存器是可缓存的。在这种情况下，将使用缓存进行读取操作，并在写入操作的情况下将写入缓存。以下是一个示例，其中随机选择了虚假寄存器地址：

```
static bool volatile_reg(struct device *dev,                          unsigned int reg)
{
    switch (reg) {
    case 0x30:
    case 0x31:
    [...]
    case 0xb3:
        return false;
    case 0xb4:
        return true;
    default:
        if ((reg >= 0xb5) && (reg <= 0xcc))
            return false;
    [...]
        break;
    }
    return true;
}
```

+   `reg_read`: 如果您的设备需要*特殊的黑客*来进行读取操作，您可以提供自定义的读取回调，并使该字段指向它，以便使用回调而不是标准的 regmap 读取函数。也就是说，大多数设备不需要这样做。

+   `reg_write`: 这与`reg_read`相同，但用于写操作。

+   `disable_locking`: 这显示了是否应该使用`lock`/`unlock`回调。如果为`false`，将不使用任何锁定机制。这意味着此 regmap 要么受到外部手段的保护，要么保证不会从多个线程访问。

+   `lock`/`unlock`: 这些是可选的锁定/解锁回调，它们会覆盖 regmap 的默认锁定/解锁函数。这些基于自旋锁或互斥锁，具体取决于访问底层设备是否可能休眠。

+   `lock_arg`: 这是`lock`/`unlock`函数的唯一参数（如果未覆盖常规的锁定/解锁函数，则将被忽略）。

+   `fast_io`: 这表示寄存器的 I/O 速度很快。如果设置了，regmap 将使用自旋锁而不是互斥锁来执行锁定。如果使用自定义的锁定/解锁（这里没有讨论）函数（请参阅内核源代码中`struct regmap_config`的`lock`/`unlock`字段），则此字段将被忽略。它应该仅用于“无总线”情况（MMIO 设备），而不是用于可能休眠的慢总线，如 I2C、SPI 或类似总线。

+   `wr_table`: 这是`writeable_reg()`回调的替代，类型为`regmap_access_table`，它是一个包含`yes_range`和`no_range`字段的结构，两者都是指向`struct regmap_range`的指针。属于`yes_range`条目的任何寄存器都被视为可写，如果属于`no_range`或未在`yes_range`中指定，则被视为不可写。

+   `rd_table`: 这与`wr_table`相同，但用于任何读取操作。

+   `volatile_table`: 您可以提供`volatile_table`而不是`volatile_reg`。其原理与`wr_table`和`rd_table`相同，但用于缓存机制。

+   `max_register`: 这是可选的；它指定了不允许任何操作的最大有效寄存器地址。

+   `reg_defaults`是`reg_default`类型的元素数组，其中每个元素都是表示给定寄存器的上电复位值的`{reg, value}`对。这与缓存一起使用，以便读取存在于此数组中且自上电复位以来尚未写入的地址时，将返回此数组中的默认寄存器值，而无需对设备执行任何读取事务。这的一个示例是 IIO 设备驱动程序，您可以在[`elixir.bootlin.com/linux/v4.19/source/drivers/iio/light/apds9960.c`](https://elixir.bootlin.com/linux/v4.19/source/drivers/iio/light/apds9960.c)上了解更多信息。

+   `use_single_rw`: 这是一个布尔值，如果设置，将指示 regmap 将设备上的任何批量写或读操作转换为一系列单个写或读操作。这对于不支持批量读取和/或写入操作的设备非常有用。

+   `can_multi_write`: 这仅针对写操作。如果设置，表示此设备支持批量写操作的多写模式。如果为空，多写请求将被拆分为单独的写操作。

+   `num_reg_defaults`: 这是`reg_defaults`中元素的数量。

+   `read_flag_mask`：这是在进行读取时要设置在寄存器的最高字节中的掩码。 通常，在 SPI 或 I2C 中，写入或读取将在顶部字节中设置最高位，以区分写入和读取操作。

+   `write_flag_mask`：这是在进行写入时要设置在寄存器的最高字节中的掩码。

+   `cache_type`：这是实际的缓存类型，可以是`REGCACHE_NONE`，`REGCACHE_RBTREE`，`REGCACHE_COMPRESSED`或`REGCACHE_FLAT`。

初始化 regmap 就像调用以下函数之一一样简单，具体取决于我们的设备所在的总线：

```
struct regmap * devm_regmap_init_i2c(
                    struct i2c_client *client,
                    struct regmap_config *config)
struct regmap * devm_regmap_init_spi(
                    struct spi_device *spi,
                    const struct regmap_config);
struct regmap * devm_regmap_init_mmio(
                    struct device *dev,
                    void __iomem *regs,
                    const struct regmap_config *config)
#define devm_regmap_init_spmi_base(dev, config) \
    __regmap_lockdep_wrapper(__devm_regmap_init_spmi_base, \
                             #config, dev, config)
#define devm_regmap_init_w1(w1_dev, config) \
    __regmap_lockdep_wrapper(__devm_regmap_init_w1, #config, \
                             w1_dev, config)
```

在前面的原型中，返回值将是一个有效的指向`struct regmap`的指针，如果出现错误，则返回`ERR_PTR()`。 regmap 将由设备管理代码自动释放。 `regs`是指向内存映射 IO 区域的指针（由`devm_ioremap_resource()`或任何`ioremap*`系列函数返回）。 `dev`是将要交互的设备（类型为`struct device`）。 以下示例是内核源代码中`drivers/mfd/sun4i-gpadc.c`的摘录：

```
struct sun4i_gpadc_dev {
    struct device *dev;
    struct regmap *regmap;
    struct regmap_irq_chip_data *regmap_irqc;
    void __iomem *base;
};
static const struct regmap_config sun4i_gpadc_regmap_config = {
    .reg_bits = 32,
    .val_bits = 32,
    .reg_stride = 4,
    .fast_io = true,
};
static int sun4i_gpadc_probe(struct platform_device *pdev)
{
    struct sun4i_gpadc_dev *dev;
    struct resource *mem;
    [...]
    mem = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    dev->base = devm_ioremap_resource(&pdev->dev, mem);
    if (IS_ERR(dev->base))
        return PTR_ERR(dev->base);
    dev->dev = &pdev->dev;
    dev_set_drvdata(dev->dev, dev);
    dev->regmap = devm_regmap_init_mmio(dev->dev, dev->base,
                                   &sun4i_gpadc_regmap_config);
    if (IS_ERR(dev->regmap)) {
       ret = PTR_ERR(dev->regmap);
       dev_err(&pdev->dev, "failed to init regmap: %d\n", ret);
       return ret;
    }
    [...]
```

这段摘录显示了如何创建 regmap。 尽管这段摘录是面向 MMIO 的，但对于其他类型，概念仍然相同。 而不是使用`devm_regmap_init_MMIO()`，我们将分别使用`devm_regmap_init_spi()`或`devm_regmap_init_i2c()`来创建基于 SPI 或 I2C 的 regmap。

## 访问设备寄存器

有两个主要函数用于访问设备寄存器。 这些是`regmap_write()`和`regmap_read()`，它们负责锁定和抽象底层总线：

```
int regmap_write(struct regmap *map,
                 unsigned int reg,
                 unsigned int val);
int regmap_read(struct regmap *map,
                unsigned int reg,
                unsigned int *val);
```

在前面的两个函数中，第一个参数`map`是初始化期间返回的 regmap 结构。 `reg`是要写入/读取数据的寄存器地址。 `val`是写操作中要写入的数据，或读操作中的读取值。以下是这些 API 的详细描述：

+   `regmap_write`用于向设备写入数据。 此函数执行以下步骤：

1) 首先，它检查`reg`是否与`regmap_config.reg_stride`对齐。 如果不是，则返回`-EINVAL`，函数失败。

2) 然后，根据`fast_io`，`lock`和`unlock`字段获取锁。 如果提供了`lock`回调，则将使用它来获取锁。 否则，regmap 核心将使用其内部默认的锁定函数，使用自旋锁或互斥锁，具体取决于是否设置了`fast_io`。 接下来，regmap 核心对传递的寄存器地址执行一些合理性检查，如下所示：

-如果设置了`max_register`，它将检查此寄存器的地址是否小于`max_register`。 如果地址不小于`max_register`，则`regmap_write()`失败，返回`-EIO`（无效的 I/O）错误代码

-然后，如果设置了`writeable_reg`回调，则将使用寄存器作为参数调用此回调。 如果此回调返回`false`，则`regmap_write()`失败，返回`-EIO`。 如果未设置`writeable_reg`但设置了`wr_table`，则 regmap 核心将检查寄存器地址是否位于`no_range`内。 如果是，则`regmap_write()`失败并返回`-EIO`。 如果不是，则 regmap 核心将检查寄存器地址是否位于`yes_range`内。 如果不在那里，则`regmap_write()`失败并返回`-EIO`。

3) 如果设置了`cache_type`字段，则将使用缓存。 要写入的值将被缓存以供将来参考，而不是写入硬件。

4) 如果未设置`cache_type`，则立即调用写入例程将值写入硬件寄存器。 在将值写入此寄存器之前，此例程将首先将`write_flag_mask`应用于寄存器地址的第一个字节。

5) 最后，使用适当的解锁函数释放锁。

+   `regmap_read`用于从设备中读取数据。此函数执行与`regmap_write()`相同的安全性和健全性检查，但用`readable_reg`和`rd_table`替换了`writable_reg`和`wr_table`。在缓存方面，如果启用了缓存，则从缓存中读取寄存器值。如果未启用缓存，则调用读取例程从硬件寄存器中读取值。该例程将在读取操作之前将`read_flag_mask`应用于寄存器地址的最高字节，并使用新读取的值更新`*val`。之后，使用适当的解锁函数释放锁。

虽然前面的访问器一次只针对一个寄存器，但其他访问器可以执行批量访问，我们将在下一节中看到。

### 一次性读/写多个寄存器

有时您可能希望同时对寄存器范围中的数据执行批量读/写操作。即使您在循环中使用`regmap_read()`或`regmap_write()`，最好的解决方案也是使用为此类情况提供的 regmap API。这些函数是`regmap_bulk_read()`和`regmap_bulk_write()`：

```
int regmap_bulk_read(struct regmap *map, unsigned int reg,
                     void *val, size_tval_count);
int regmap_bulk_write(struct regmap *map, unsigned int reg,
                      const void *val, size_t val_count)
```

这些函数从/向设备读/写多个寄存器。`map`是用于执行操作的 regmap。对于读操作，`reg`是应该开始读取的第一个寄存器，`val`是指向应该以*设备的本机寄存器大小*存储读取值的缓冲区的指针（这意味着如果设备寄存器大小为 4 字节，则读取值将以 4 字节单位存储），`val_count`是要读取的寄存器数量。对于写操作，`reg`是应该从中开始写入的第一个寄存器，`val`是指向应该以*设备的本机寄存器大小*写入的数据块的指针，`val_count`是要写入的寄存器数量。对于这两个函数，成功时将返回`0`的值，如果出现错误，则将返回负的`errno`。

提示

此框架提供了其他有趣的读/写函数。查看内核头文件以获取更多信息。一个有趣的函数是`regmap_multi_reg_write()`，它以任何顺序提供的{寄存器，值}对集合写入多个寄存器，可能不都在单个范围内，给定为参数的设备。

现在我们已经熟悉了寄存器访问，我们可以通过在位级别管理寄存器内容来进一步深入。

### 更新寄存器中的位

要更新给定寄存器中的位，我们有`regmap_update_bits()`，一个三合一的函数。其原型如下：

```
int regmap_update_bits(struct regmap *map, unsigned int reg,
                       unsigned int mask, unsigned int val)
```

它在寄存器映射上执行读/修改/写循环。它是`_regmap_update_bits()`的包装器，如下所示：

```
static int _regmap_update_bits(
                struct regmap *map, unsigned int reg,
                unsigned int mask, unsigned int val,
                bool *change, bool force_write)
{
    int ret;
    unsigned int tmp, orig;
    if (change)
        *change = false;
    if (regmap_volatile(map, reg) && map->reg_update_bits) {
        ret = map->reg_update_bits(map->bus_context,
                                    reg, mask, val);
        if (ret == 0 && change)
            *change = true;
    } else {
        ret = _regmap_read(map, reg, &orig);
        if (ret != 0)
            return ret;
        tmp = orig & ~mask;
        tmp |= val & mask;
        if (force_write || (tmp != orig)) {
            ret = _regmap_write(map, reg, tmp);
            if (ret == 0 && change)
                *change = true;
        }
    }
    return ret;
}
```

需要更新的位应该在`mask`中设置为`1`，相应的位将获得`val`中相同位置的位的值。例如，要将第一个（`BIT(0)`）和第三个（`BIT(2)`）位设置为`1`，`mask`应该是`0b00000101`，值应该是`0bxxxxx1x1`。要清除第七位（`BIT(6)`），`mask`必须是`0b01000000`，值应该是`0bx0xxxxxx`，依此类推。

提示

为了调试目的，您可以使用`debugfs`文件系统来转储 regmap 管理的寄存器内容，如下摘录所示：

# mount -t debugfs none /sys/kernel/debug

# cat /sys/kernel/debug/regmap/1-0008/registers

这将以`<addr:value>`格式转储寄存器地址及其值。

在本节中，我们已经看到了访问硬件寄存器是多么容易。此外，我们已经学会了一些在位级别上玩耍寄存器的花哨技巧，这在状态和配置寄存器中经常使用。接下来，我们将看一下 IRQ 管理。

Regmap 和 IRQ 管理

Regmap 不仅仅是对寄存器的访问进行了抽象。在这里，我们将看到这个框架如何在更低的级别抽象 IRQ 管理，比如 IRQ 芯片处理，从而隐藏样板操作。

## Linux 内核 IRQ 管理的快速回顾

通过特殊设备称为中断控制器向设备公开 IRQ。从软件角度来看，中断控制器设备驱动程序使用 Linux 内核中的 IRQ 域概念来管理和公开这些线。中断管理建立在以下结构之上：

+   `struct irq_chip`：这个结构体是 Linux 对 IRQ 控制器的表示，并实现了一组方法来驱动直接由核心 IRQ 代码调用的中断控制器。必要时，该结构应该由驱动程序填充，提供一组回调函数，允许我们在 IRQ 芯片上管理 IRQ，例如`irq_startup`、`irq_shutdown`、`irq_enable`、`irq_disable`、`irq_ack`、`irq_mask`、`irq_unmask`、`irq_eoi`和`irq_set_affinity`。愚蠢的 IRQ 芯片设备（例如不允许 IRQ 管理的芯片）应该使用内核提供的`dummy_irq_chip`。

+   `struct irq_domain`：每个中断控制器都有一个域，对于控制器来说，它就像地址空间对于进程一样。`struct irq_domain`结构存储了硬件 IRQ 和 Linux IRQ（即虚拟 IRQ 或 virq）之间的映射。它是硬件中断号转换对象。这个结构提供以下内容：

- 给定中断控制器的固件节点（`fwnode`）的指针。

- 将 IRQ 的固件（设备树）描述转换为中断控制器本地的 ID（硬件 IRQ 号，称为 hwirq）的方法。对于也充当 IRQ 控制器的 gpio 芯片，给定 gpio 线的硬件 IRQ 号（hwirq）大多数情况下对应于该线在芯片中的本地索引。

- 从 hwirq 中检索 IRQ 的 Linux 视图的方法。

+   `struct irq_desc`：这个结构是 Linux 内核对中断的视图，包含所有核心内容，并且与 Linux 中断号一一对应。

+   `struct irq_action`：这是 Linux 用来描述 IRQ 处理程序的结构。

+   `struct irq_data`：这个结构嵌入在`struct irq_desc`结构中，并包含以下内容：

- 与管理此中断的`irq_chip`相关的数据

- Linux IRQ 号和 hwirq 都是

- 指向`irq_chip`的指针

- 指向中断转换域（`irq_domain`）的指针

始终牢记**irq_domain 对于中断控制器就像地址空间对于进程一样，因为它存储了 virq 和 hwirq 之间的映射**。

中断控制器驱动程序通过调用`irq_domain_add_<mapping_method>()`函数之一来创建和注册`irq_domain`。这些函数实际上是`irq_domain_add_linear()`、`irq_domain_add_tree()`和`irq_domain_add_nomap()`。事实上，`<mapping_method>`是`hwirqs`应该映射到`virqs`的方法。

`irq_domain_add_linear()`创建一个空的固定大小的表，由 hwirq 号索引。为每个被映射的 hwirq 分配`struct irq_desc`。然后将分配的 IRQ 描述符存储在表中，索引等于它被分配的 hwirq。这种线性映射适用于固定和较小数量的 hwirq（小于 256）。

虽然这种映射的主要优势是 IRQ 号查找时间是固定的，并且`irq_desc`仅为正在使用的 IRQ 分配，但主要缺点来自表的大小，它可能与最大可能的`hwirq`号一样大。大多数驱动程序应该使用线性映射。这个函数有以下原型：

```
struct irq_domain *irq_domain_add_linear(
                             struct device_node *of_node,
                             unsigned int size,
                             const struct irq_domain_ops *ops,
                             void *host_data)
```

`irq_domain_add_tree()`创建一个空的`irq_domain`，在基数树中维护 Linux IRQ 和`hwirq`号之间的映射。当映射 hwirq 时，会分配一个`struct irq_desc`，并且 hwirq 被用作基数树的查找键。如果 hwirq 号非常大，则树映射是一个很好的选择，因为它不需要分配一个与最大 hwirq 号一样大的表。缺点是`hwirq-to-IRQ`号查找取决于表中有多少条目。很少有驱动程序需要这种映射。它有以下原型：

```
struct irq_domain *irq_domain_add_tree(
                       struct device_node *of_node,
                       const struct irq_domain_ops *ops,
                       void *host_data)
```

`irq_domain_add_nomap()`是您可能永远不会使用的东西；但是，其完整描述可以在内核源树中的`Documentation/IRQ-domain.txt`中找到。其原型如下：

```
struct irq_domain *irq_domain_add_nomap(
                              struct device_node *of_node,
                              unsigned int max_irq,
                              const struct irq_domain_ops *ops,
                              void *host_data)
```

在所有这些原型中，`of_node`是指向中断控制器的 DT 节点的指针。`size`表示线性映射情况下域中中断的数量。`ops`表示 map/unmap 域回调，`host_data`是控制器的私有数据指针。由于这三个函数都创建了空的`irq`域，因此应该使用`irq_create_mapping()`函数，将 hwirq 和传递给它的`irq`域的指针一起使用，以创建映射，并将此映射插入到域中：

```
unsigned int irq_create_mapping(struct irq_domain *domain,
                                irq_hw_number_t hwirq)
```

在上述原型中，`domain`是此硬件中断所属的域。`NULL`值表示默认域。`hwirq`是您需要为其创建映射的硬件 IRQ 号。此函数将硬件中断映射到 Linux IRQ 空间，并返回 Linux IRQ 号。还要记住，每个硬件中断只允许一个映射。以下是创建映射的示例：

```
unsigned int virq = 0;
virq = irq_create_mapping(irq_domain, hwirq);
if (!virq) {
    ret = -EINVAL;
    goto err_irq;
}
```

在上述代码中，`virq`是 Linux 内核 IRQ（**虚拟 IRQ 号**，**virq**）对应的映射。

重要提示

当为也是中断控制器的 GPIO 控制器编写驱动程序时，从`gpio_chip.to_irq()`回调中调用`irq_create_mapping()`，并将 virq 返回为`return irq_create_mapping(gpiochip->irq_domain, hwirq)`，其中`hwirq`是从 GPIO 芯片的 GPIO 偏移量。

一些驱动程序更喜欢在`probe()`函数内提前创建映射并填充每个 hwirq 的域，如下所示：

```
for (j = 0; j < gpiochip->chip.ngpio; j++) {
    irq = irq_create_mapping(gpiochip ->irq_domain, j);
}
```

之后，这样的驱动程序只需在`to_irq()`回调函数中调用`irq_find_mapping()`（给定 hwirq）。如果给定的`hwirq`尚不存在映射，则`irq_create_mapping()`将分配一个新的`struct irq_desc`结构，将其与 hwirq 关联，并调用`irq_domain_ops.map()`回调（使用`irq_domain_associate()`函数）以便驱动程序可以执行任何所需的硬件设置。

### struct irq_domain_ops

此结构公开了一些特定于 irq 域的回调。由于在给定的 irq 域中创建了映射，因此应为每个映射（实际上是每个`irq_desc`）提供一个 irq 配置、一些私有数据和一个转换函数（给定设备树节点和中断说明符，转换函数解码硬件 irq 号和 Linux irq 类型值）。这就是此结构中回调的作用：

```
struct irq_domain_ops {
    int (*map)(struct irq_domain *d, unsigned int virq,
               irq_hw_number_t hw);
    void (*unmap)(struct irq_domain *d, unsigned int virq);
   int (*xlate)(struct irq_domain *d, struct device_node *node,
                const u32 *intspec, unsigned int intsize,
                unsigned long *out_hwirq,                 unsigned int *out_type);
};
```

上述数据结构中 Linux 内核 IRQ 管理的元素都值得单独的部分来描述。

#### irq_domain_ops.map()

以下是此回调的原型：

```
int (*map)(struct irq_domain *d, unsigned int virq,
            irq_hw_number_t hw);
```

在描述此函数的功能之前，让我们描述一下它的参数：

+   `d`：此 IRQ 芯片使用的 IRQ 域

+   `virq`：此基于 GPIO 的 IRQ 芯片使用的全局 IRQ 号

+   `hw`：此 GPIO 芯片上的本地 IRQ/GPIO 线偏移量

`.map()`创建或更新 virq 和 hwirq 之间的映射。此回调设置 IRQ 配置。对于给定的映射，它只会被（由 irq 核心内部）调用一次。这是我们为给定的 irq 设置`irq`芯片数据的地方，可以使用`irq_set_chip_data()`来完成，其原型如下：

```
int irq_set_chip_data(unsigned int irq, void *data); 
```

根据 IRQ 芯片的类型（嵌套或链式），可以执行其他操作。

#### irq_domain_ops.xlate()

给定一个 DT 节点和一个中断指定器，这个回调函数解码硬件 IRQ 号以及它的 Linux IRQ 类型值。根据你的 DT 控制器节点中指定的`#interrupt-cells`属性，内核提供了一个通用的翻译函数：

+   `irq_domain_xlate_twocell()`: 这是一个用于直接双细胞绑定的通用翻译函数。DT IRQ 指定器与双细胞绑定一起工作，其中细胞值直接映射到`hwirq`号和 Linux IRQ 标志。

+   `irq_domain_xlate_onecell()`: 这是一个用于直接单细胞绑定的通用`xlate`函数。

+   `irq_domain_xlate_onetwocell()`: 这是一个用于单细胞或双细胞绑定的通用`xlate`函数。

域操作的一个示例如下：

```
static struct irq_domain_ops mcp23016_irq_domain_ops = {
    .map = my_irq_domain_map,
    .xlate = irq_domain_xlate_twocell,
};
```

前面数据结构的显著特点是分配给`.xlate`元素的值，即`irq_domain_xlate_twocell`。这意味着我们期望在设备树中有一个双细胞`irq`指定器，其中第一个细胞指定`irq`，第二个指定其标志。

### 链接 IRQ

当发生中断时，可以使用`irq_find_mapping()`辅助函数从`hwirq`号中找到 Linux IRQ 号。例如，这个`hwirq`号可能是 GPIO 控制器组中的 GPIO 偏移量。一旦找到并返回了有效的 virq，你应该在这个`virq`上调用`handle_nested_irq()`或`generic_handle_irq()`。魔法来自于前两个函数，它们管理了`irq`流处理程序。这意味着有两种处理中断处理程序的方法。硬中断处理程序，或者**链式中断**，是原子的，运行时中断被禁用，并且可能调度线程处理程序；还有简单的线程中断处理程序，称为**嵌套中断**，可能会被其他中断打断。

#### 链式中断

这种方法用于可能不休眠的控制器，比如 SoC 的内部 GPIO 控制器，它是内存映射的，其访问不休眠。*链式*意味着这些中断只是一系列函数调用（例如，SoC 的 GPIO 控制器中断处理程序是从 GIC 中断处理程序中调用的，就像函数调用一样）。采用这种方法，子 IRQ 处理程序在父 hwirq 处理程序内被调用。在这里必须使用`generic_handle_irq()`将子 IRQ 处理程序链接到父 hwirq 处理程序。即使在子中断处理程序内部，我们仍然处于原子上下文（硬件中断）。你不能调用可能会休眠的函数。

对于链式（仅链式）IRQ 芯片，`irq_domain_ops.map()`也是将高级`irq-type`流处理程序分配给给定的 irq 的正确位置，使用`irq_set_chip_and_handler()`，这样高级代码，根据它的内容，将在调用相应的 irq 处理程序之前执行一些操作。这里的魔法操作得益于`irq_set_chip_and_handler()`函数：

```
void irq_set_chip_and_handler(unsigned int irq,
                              struct irq_chip *chip,
                              irq_flow_handler_t handle)
```

在前面的原型中，`irq`代表 Linux IRQ（`virq`），作为参数传递给`irq_domain_ops.map()`函数；`chip`是你的`irq_chip`结构；`handle`是你的高级中断流处理程序。

重要提示

有些控制器非常简单，几乎不需要在它们的`irq_chip`结构中做任何事情。在这种情况下，你应该将`dummy_irq_chip`传递给`irq_set_chip_and_handler()`。`dummy_irq_chip`在`kernel/irq/dummychip.c`中定义。

以下是`irq_set_chip_and_handler()`的代码流程总结：

```
void irq_set_chip_and_handler(unsigned int irq,
                              struct irq_chip *chip,
                              irq_flow_handler_t handle)
{
    struct irq_desc *desc = irq_get_desc(irq);
    desc->irq_data.chip = chip;
    desc->handle_irq = handle;
}
```

这些是通用层提供的一些可能的高级 IRQ 流处理程序：

```
/*
 * Built-in IRQ handlers for various IRQ types,
 * callable via desc->handle_irq()
 */
void handle_level_irq(struct irq_desc *desc);
void handle_fasteoi_irq(struct irq_desc *desc);
void handle_edge_irq(struct irq_desc *desc);
void handle_edge_eoi_irq(struct irq_desc *desc);
void handle_simple_irq(struct irq_desc *desc);
void handle_untracked_irq(struct irq_desc *desc);
void handle_percpu_irq(struct irq_desc *desc);
void handle_percpu_devid_irq(struct irq_desc *desc);
void handle_bad_irq(struct irq_desc *desc);
```

每个函数名都很好地描述了它处理的 IRQ 类型。对于链式 IRQ 芯片，`irq_domain_ops.map()`可能如下所示：

```
static int my_chained_irq_domain_map(struct irq_domain *d,
                                     unsigned int virq,
                                     irq_hw_number_t hw)
{
    irq_set_chip_data(virq, d->host_data);
    irq_set_chip_and_handler(virq, &dummy_irq_chip,                              handle_ edge_irq);
    return 0;
}
```

在为链式 IRQ 芯片编写父 irq 处理程序时，代码应该在每个子 irq 上调用`generic_handle_irq()`。这个函数简单地调用`irq_desc->handle_irq()`，它指向使用`irq_set_chip_and_handler()`分配给给定子 IRQ 的高级中断处理程序。底层的高级`irq`事件处理程序（比如`handle_level_irq()`）首先会做一些小技巧，然后会运行硬`irq-handler`（`irq_desc->action->handler`），根据返回值，如果提供的话，会运行线程处理程序（`irq_desc->action->thread_fn`）。

以下是链式 IRQ 芯片的父 IRQ 处理程序的示例，其原始代码位于内核源码中的`drivers/pinctrl/pinctrl-at91.c`中：

```
static void parent_hwirq_handler(struct irq_desc *desc)
{
    struct irq_chip *chip = irq_desc_get_chip(desc);
    struct gpio_chip *gpio_chip =     irq_desc_get_handler_ data(desc);
    struct at91_gpio_chip *at91_gpio = gpiochip_get_data                                       (gpio_ chip);
    void __iomem *pio = at91_gpio->regbase;
    unsigned long isr;
    int n;
    chained_irq_enter(chip, desc);
    for (;;) {
        /* Reading ISR acks pending (edge triggered) GPIO
         * interrupts. When there are none pending, we’re
         * finished unless we need to process multiple banks
         * (like ID_PIOCDE on sam9263).
         */
        isr = readl_relaxed(pio + PIO_ISR) &
                           readl_relaxed(pio + PIO_IMR);
        if (!isr) {
            if (!at91_gpio->next)
                break;
            at91_gpio = at91_gpio->next;
            pio = at91_gpio->regbase;
            gpio_chip = &at91_gpio->chip;
            continue;
        }
        for_each_set_bit(n, &isr, BITS_PER_LONG) {
            generic_handle_irq(
                   irq_find_mapping(gpio_chip->irq.domain, n));
        }
    }
    chained_irq_exit(chip, desc);
    /* now it may re-trigger */
    [...]
}
```

链式 IRQ 芯片驱动程序不需要使用`devm_request_threaded_irq()`或`devm_request_irq()`注册父`irq`处理程序。当驱动程序在父 irq 上调用`irq_set_chained_handler_and_data()`时，此处理程序会自动注册，并提供相关的处理程序和一些私有数据：

```
void irq_set_chained_handler_and_data(unsigned int irq,
                                      irq_flow_handler_t                                       handle,
                                      void *data)
```

这个函数的参数非常容易理解。您应该在`probe`函数中调用这个函数，如下所示：

```
static int my_probe(struct platform_device *pdev)
{
    int parent_irq, i;
    struct irq_domain *my_domain;
    parent_irq = platform_get_irq(pdev, 0);
    if (!parent_irq) {
     pr_err("failed to map parent interrupt %d\n", parent_irq);
        return -EINVAL;
    }
    my_domain =
        irq_domain_add_linear(np, nr_irq, &my_irq_domain_ops,
                              my_private_data);
    if (WARN_ON(!my_domain)) {
        pr_warn("%s: irq domain init failed\n", __func__);
        return;
    }
    /* This may be done elsewhere */
    for(i = 0; i < nr_irq; i++) {
        int virqno = irq_create_mapping(my_domain, i);
         /*
          * May need to mask and clear all IRQs before           * registering a handler
          */
           [...]
          irq_set_chained_handler_and_data(parent_irq,
                                          parent_hwirq_handler,
                                          my_private_data);
          /* 
           * May need to call irq_set_chip_data() on            * the virqno too            */
        [...]
    }
    [...]
}
```

在前面虚假的`probe`方法中，使用`irq_domain_add_linear()`创建了一个线性域，并在该域中使用`irq_create_mapping()`创建了一个 irq 映射（虚拟 irq）。最后，我们为主（或父）IRQ 设置了一个高级链式流处理程序及其数据。

重要提示

请注意，`irq_set_chained_handler_and_data()`会自动启用中断（指定为第一个参数），分配其处理程序（也作为参数给出），并将此中断标记为`IRQ_NOREQUEST`、`IRQ_NOPROBE`或`IRQ_NOTHREAD`，这意味着此中断不能再通过`request_irq()`请求，不能通过自动探测进行探测，也不能线程化（它是链式的）。

#### 嵌套中断

嵌套流方法是由可能休眠的 IRQ 芯片使用的，例如那些位于慢总线上的 IRQ 芯片，比如 I2C（例如，I2C GPIO 扩展器）。"嵌套"指的是那些不在硬件上下文中运行的中断处理程序（它们实际上不是 hwirq，并且不在原子上下文中），而是线程化的，可以被抢占。在这里，处理程序函数是在调用线程的上下文中调用的。对于嵌套（仅对嵌套）IRQ 芯片，`irq_domain_ops.map()`回调也是设置`irq`配置标志的正确位置。最重要的配置标志如下：

+   `IRQ_NESTED_THREAD`：这是一个标志，表示在`devm_request_threaded_irq()`上，不应为 irq 处理程序创建专用的中断线程，因为它在解复用中断处理程序线程的上下文中被嵌套调用（在内核源码中的`kernel/irq/manage.c`中实现了`__setup_irq()`函数中有更多关于此的信息）。您可以使用`void irq_set_nested_thread(unsigned int irq, int nest)`来操作此标志，其中`irq`对应于全局中断号，`nest`应为`0`以清除或`1`以设置`IRQ_NESTED_THREAD`标志。

+   `IRQ_NOTHREAD`：可以使用`void irq_set_nothread(unsigned int irq)`设置此标志。它用于将给定的 IRQ 标记为不可线程化。

这是嵌套 IRQ 芯片的`irq_domain_ops.map()`可能看起来像这样：

```
static int my_nested_irq_domain_map(struct irq_domain *d,
                                    unsigned int virq,
                                    irq_hw_number_t hw)
{
    irq_set_chip_data(virq, d->host_data);
    irq_set_nested_thread(virq, 1);
    irq_set_noprobe(virq);
    return 0;
}
```

在为嵌套 IRQ 芯片编写父 irq 处理程序时，代码应该调用`handle_nested_irq()`来处理子 irq 处理程序，以便它们从父 irq 线程中运行。`handle_nested_irq()`不关心`irq_desc->action->handler`，即硬 irq 处理程序。它只运行`irq_desc->action->thread_fn`：

```
static irqreturn_t mcp23016_irq(int irq, void *data)
{
    struct mcp23016 *mcp = data;
    unsigned int child_irq, i;
    /* Do some stuff */
    [...]
    for (i = 0; i < mcp->chip.ngpio; i++) {
        if (gpio_value_changed_and_raised_irq(i)) {
            child_irq = irq_find_mapping(mcp->chip.irqdomain,                                         i);
            handle_nested_irq(child_irq);
        }
    }
    [...]
}
```

嵌套 IRQ 芯片驱动程序使用`devm_request_threaded_irq()`，因为对于这种类型的 IRQ 芯片没有像`irq_set_chained_handler_and_data()`这样的函数。对于嵌套 IRQ 芯片使用这个 API 是没有意义的。嵌套 IRQ 芯片大多数情况下是基于 GPIO 芯片的。因此，最好使用基于 GPIO 芯片的 IRQ 芯片 API，或者使用基于 regmap 的 IRQ 芯片 API，如下一节所示。然而，让我们看看这样一个例子是什么样子的：

```
static int my_probe(struct i2c_client *client,
                    const struct i2c_device_id *id)
{
    int parent_irq, i;
    struct irq_domain *my_domain;
    [...]
    int irq_nr = get_number_of_needed_irqs();
    /* Do we have an interrupt line ? Enable the IRQ chip */
    if (client->irq) {
        domain = irq_domain_add_linear(
                        client->dev.of_node, irq_nr,
                        &my_irq_domain_ops, my_private_data);
        if (!domain) {
            dev_err(&client->dev,
                    "could not create irq domain\n");
            return -ENODEV;
        }
        /*
         * May be creating irq mapping in this domain using
         * irq_create_mapping() or let the mfd core doing
         * this if it is an MFD chip device
         */
        [...]
        ret =
            devm_request_threaded_irq(
                &client->dev, client->irq,
                NULL, my_parent_irq_thread,
                IRQF_TRIGGER_FALLING | IRQF_ONESHOT,
                "my-parent-irq", my_private_data);
        [...]
    }
[...]
}
```

在上述的`probe`方法中，与链接流有两个主要区别：

+   首先，主 IRQ 的注册方式：在链接的 IRQ 芯片中使用了`irq_set_chained_handler_and_data()`，它自动注册了处理程序，而嵌套流方法必须使用`request_threaded_irq()`系列方法显式注册其处理程序。

+   其次，主 IRQ 处理程序调用底层 irq 处理程序的方式：在链接流中，主 IRQ 处理程序中调用`handle_nested_irq()`，它调用每个底层 irq 的处理程序作为一系列函数调用，这些函数调用在与主处理程序相同的上下文中执行，即原子地（原子性也称为`hard-irq`）。然而，嵌套流处理程序必须调用`handle_nested_irq()`，它在父级的线程上下文中执行底层 irq 的处理程序（`thread_fn`）。

这些是链接和嵌套流之间的主要区别。

### irqchip 和 gpiolib API - 新一代

由于每个`irq-gpiochip`驱动程序都在其自己的`irqdomain`处理中进行了开放编码，这导致了大量的冗余代码。内核开发人员决定将该代码移动到 gpiolib 框架中，从而提供了`GPIOLIB_IRQCHIP` Kconfig 符号，使我们能够为 GPIO 芯片使用统一的 irq 域管理 API。该代码部分有助于处理 GPIO IRQ 芯片和相关`irq_domain`和资源分配回调的管理，以及它们的设置，使用了一组减少的辅助函数。这些函数是`gpiochip_irqchip_add()`或`gpiochip_irqchip_add_nested()`，以及`gpiochip_set_chained_irqchip()`或`gpiochip_set_nested_irqchip()`。`gpiochip_irqchip_add()`或`gpiochip_irqchip_add_nested()`都向 GPIO 芯片添加一个 IRQ 芯片。以下是它们各自的原型：

```
static inline int gpiochip_irqchip_add(                                    struct gpio_chip *gpiochip,
                                    struct irq_chip *irqchip,
                                    unsigned int first_irq,
                                    irq_flow_handler_t handler,
                                    unsigned int type)
static inline int gpiochip_irqchip_add_nested(
                          struct gpio_chip *gpiochip,
                          struct irq_chip *irqchip,
                          unsigned int first_irq,
                          irq_flow_handler_t handler,
                          unsigned int type)
```

在上述原型中，`gpiochip`参数是要将`irqchip`添加到的 GPIO 芯片。`irqchip`是要添加到 GPIO 芯片以扩展其功能，使其也可以充当 IRQ 控制器的 IRQ 芯片。这个 IRQ 芯片必须被正确配置，要么由驱动程序，要么由 IRQ 核心代码（如果给定`dummy_irq_chip`作为参数）。如果没有动态分配，`first_irq`将是从中分配 GPIO 芯片 IRQ 的基础（第一个）IRQ。`handler`是要使用的主要 IRQ 处理程序（通常是预定义的高级 IRQ 核心函数之一）。`type`是此`IRQ 芯片`上 IRQ 的默认类型；在这里传递`IRQ_TYPE_NONE`，并让驱动程序在请求时配置这个。

每个函数操作的摘要如下：

+   第一个函数使用`irq_domain_add_simple()`函数为 GPIO 芯片分配了一个`struct irq_domain`。这个 IRQ 域的 ops 是使用内核 IRQ 核心域 ops 变量`gpiochip_domain_ops`设置的。这个域 ops 在`drivers/gpio/gpiolib.c`中定义，`irq_domain_ops.xlate`字段设置为`irq_domain_xlate_twocell`，这意味着这个 gpio 芯片将处理双细胞的 IRQ。

+   将`gpiochip.to_irq`字段设置为`gpiochip_to_irq`，这是一个回调函数，返回`irq_create_mapping(chip->irq.domain, offset)`，创建一个与 GPIO 偏移对应的 IRQ 映射。当我们在该 GPIO 上调用`gpiod_to_irq()`时执行此操作。这个函数假设`gpiochip`上的每个引脚都可以生成唯一的 IRQ。以下是`gpiochip_domain_ops` IRQ 域的定义：

```
static const struct irq_domain_ops gpiochip_domain_ops = {
  .map = gpiochip_irq_map,
  .unmap = gpiochip_irq_unmap,
  /* Virtually all GPIO-based IRQ chips are two-celled */
  .xlate = irq_domain_xlate_twocell,
};
```

`gpiochip_irqchip_add_nested()`和`gpiochip_irqchip_add()`之间唯一的区别是前者向 GPIO 芯片添加了一个嵌套的 IRQ 芯片（它将`gpio_chip->irq.threaded`字段设置为`true`），而后者向 GPIO 芯片添加了一个链式的 IRQ 芯片，并将此字段设置为`false`。另一方面，`gpiochip_set_chained_irqchip()`和`gpiochip_set_nested_irqchip()`分别将链式或嵌套的 IRQ 芯片分配/连接到 GPIO 芯片。以下是这两个函数的原型：

```
void gpiochip_set_chained_irqchip(                             struct gpio_chip *gpiochip,
                             struct irq_chip *irqchip,
                             unsigned int parent_irq,
                             irq_flow_handler_t parent_handler)
void gpiochip_set_nested_irqchip(struct gpio_chip *gpiochip,
                                 struct irq_chip *irqchip,
                                 unsigned int parent_irq)
```

在上述原型中，`gpiochip`是要设置`irqchip`链的 GPIO 芯片。`irqchip`代表要连接到 GPIO 芯片的 IRQ 芯片。`parent_irq`是与此链式 IRQ 芯片对应的父 IRQ 的 irq 编号。换句话说，它是连接到此芯片的 IRQ 编号。`parent_handler`是 GPIO 芯片累积的 IRQ 的父中断处理程序。实际上，这是 hwirq 处理程序。对于嵌套的 IRQ 芯片，这不会被使用，因为父处理程序是线程化的。链式变体将在`parent_handler`上内部调用`irq_set_chained_handler_and_data()`。

#### 链式 gpiochip 基于的 IRQ 芯片

`gpiochip_irqchip_add()`和`gpiochip_set_chained_irqchip()`用于链式 GPIO 芯片基于的 IRQ 芯片，而`gpiochip_irqchip_add_nested()`和`gpiochip_set_nested_irqchip()`仅用于嵌套 GPIO 芯片基于的 IRQ 芯片。对于链式 GPIO 芯片基于的 IRQ 芯片，`gpiochip_set_chained_irqchip()`将配置父 hwirq 的处理程序。不需要调用任何`devm_request_*` `irq`家族函数。但是，父 hwirq 的处理程序必须在引发的子`irqs`上调用`generic_handle_irq()`，如下面的示例（来自内核源代码中的`drivers/pinctrl/pinctrl-at91.c`），与标准的链式 IRQ 芯片有些相似：

```
static void gpio_irq_handler(struct irq_desc *desc)
{
    unsigned long isr;
    int n;
    struct irq_chip *chip = irq_desc_get_chip(desc);
    struct gpio_chip *gpio_chip =     irq_desc_get_handler_data(desc);
    struct at91_gpio_chip *at91_gpio =
                      gpiochip_get_data(gpio_chip);
    void __iomem *pio = at91_gpio->regbase;
    chained_irq_enter(chip, desc);
    for (;;) {
        isr = readl_relaxed(pio + PIO_ISR) &
                  readl_relaxed(pio + PIO_IMR);
        [...]
        for_each_set_bit(n, &isr, BITS_PER_LONG) {
            generic_handle_irq(irq_find_mapping(
                          gpio_chip->irq.domain, n));
        }
    }
    chained_irq_exit(chip, desc);
    [...]
}
```

在前面的代码中，首先介绍了中断处理程序。当 GPIO 芯片发出中断时，将读取整个 gpio 状态银行，以便检测其中设置的每个位，这意味着由相应的 gpio 线后面的设备触发的潜在 IRQ。

然后在域中索引与 gpio 状态银行中设置的位的索引相对应的每个 irq 描述符上调用`generic_handle_irq()`。这种方法将在原子上下文（`hard-irq`上下文）中调用在前一步中找到的每个描述符的每个处理程序，除非用于将 gpio 用作 irq 线的设备的底层驱动程序请求处理程序为线程化。

现在我们可以介绍`probe`方法，一个示例如下：

```
static int at91_gpio_probe(struct platform_device *pdev)
{
    [...]
    ret = gpiochip_irqchip_add(&at91_gpio->chip,
                                &gpio_irqchip,
                                0,
                                handle_edge_irq,
                                IRQ_TYPE_NONE);
    if (ret) {
       dev_err(
           &pdev->dev,
           "at91_gpio.%d: Couldn’t add irqchip to gpiochip.\n",
           at91_gpio->pioc_idx);
        return ret;
    }
    [...]
    /* Then register the chain on the parent IRQ */
    gpiochip_set_chained_irqchip(&at91_gpio->chip,
                                &gpio_irqchip,
                                at91_gpio->pioc_virq,
                                gpio_irq_handler);
    return 0;
}
```

这里没有什么特别的。这里的机制在某种程度上遵循了我们在通用 IRQ 芯片中看到的内容。父 IRQ 在这里不是使用任何`request_irq()`家族方法请求的，因为`gpiochip_set_chained_irqchip()`将在底层调用`irq_set_chained_handler_and_data()`。

#### 基于嵌套的 gpiochip irqchips

以下摘录显示了驱动程序如何注册其嵌套 GPIO 芯片基于的 IRQ 芯片。这在某种程度上类似于独立的嵌套 IRQ 芯片：

```
static irqreturn_t pcf857x_irq(int irq, void *data)
{
    struct pcf857x *gpio = data;
    unsigned long change, i, status;
    status = gpio->read(gpio->client);
    /*
     * call the interrupt handler if gpio is used as
     * interrupt source, just to avoid bad irqs
     */
    mutex_lock(&gpio->lock);
    change = (gpio->status ^ status) & gpio->irq_enabled;
    gpio->status = status;
    mutex_unlock(&gpio->lock);
    for_each_set_bit(i, &change, gpio->chip.ngpio)
        handle_nested_irq(
            irq_find_mapping(gpio->chip.irq.domain, i));
    return IRQ_HANDLED;
}
```

前面的代码是 IRQ 处理程序。正如我们所看到的，它使用`handle_nested_irq()`，这对我们来说并不新鲜。现在让我们检查`probe`方法：

```
static int pcf857x_probe(struct i2c_client *client,
                         const struct i2c_device_id *id)
{
    struct pcf857x *gpio;
    [...]
    /* Enable irqchip only if we have an interrupt line */
    if (client->irq) {
        status = gpiochip_irqchip_add_nested(&gpio->chip,
                                             &gpio->irqchip,
                                             0,                                              handle_level_irq,
                                             IRQ_TYPE_NONE);
        if (status) {
            dev_err(&client->dev, "cannot add irqchip\n");
            goto fail;
        }
        status = devm_request_threaded_irq(
                 &client->dev, client->irq,
                 NULL, pcf857x_irq,
                 IRQF_ONESHOT |IRQF_TRIGGER_FALLING |                  IRQF_SHARED,
              dev_name(&client->dev), gpio);
        if (status)
            goto fail;
        gpiochip_set_nested_irqchip(&gpio->chip,                                     &gpio->irqchip,
                                    client->irq);
    }
[...]
}
```

在这里，父 irq 处理程序是线程化的，并且必须使用`devm_request_threaded_irq()`进行注册。这解释了为什么它的 IRQ 处理程序必须在子 irq 上调用`handle_nested_irq()`以调用它们的处理程序。再次，这看起来像通用的嵌套`irqchips`，除了 gpiolib 已经包装了一些底层的嵌套`irqchip`API。要确认这一点，您可以查看`gpiochip_set_nested_irqchip()`和`gpiochip_irqchip_add_nested()`方法的主体。

## Regmap IRQ API 和数据结构

Regmap IRQ API 实现在 `drivers/base/regmap/regmap-irq.c` 中。它主要建立在两个基本函数 `devm_regmap_add_irq_chip()` 和 `regmap_irq_get_virq()` 以及三个数据结构 `struct regmap_irq_chip`、`struct regmap_irq_chip_data` 和 `struct regmap_irq` 的基础上。

重要说明

Regmap 的 `irqchip` API 完全使用了线程中断。因此，只有我们在 *嵌套中断* 部分看到的内容才适用于这里。

### Regmap IRQ 数据结构

如前所述，我们需要介绍 `regmap irq api` 的三个数据结构，以便了解它如何抽象中断管理。

#### `struct regmap_irq_chip` 和 `struct regmap_irq`

`struct regmap_irq_chip` 结构描述了一个通用的 `regmap irq_chip`。在讨论这个结构之前，让我们先介绍 `struct regmap_irq`，它存储了 `regmap irq_chip` 的中断的寄存器和掩码描述：

```
struct regmap_irq {
    unsigned int reg_offset;
    unsigned int mask;
    unsigned int type_reg_offset;
    unsigned int type_rising_mask;
    unsigned int type_falling_mask;
};
```

以下是前述结构中字段的描述：

+   `reg_offset` 是在银行内状态/掩码寄存器的偏移。该银行实际上可能是 IRQ `chip` 的 `{status/mask/unmask/ack/wake}_base` 寄存器。

+   `mask` 是用于标记/控制此中断状态寄存器的掩码。在禁用中断时，掩码值将与来自 regmap 的 `irq_chip.status_base` 寄存器的 `reg_offset` 的实际内容进行 *OR* 运算。对于中断使能，将进行 `~mask` 的 AND 运算。

+   `type_reg_offset` 是 IRQ 类型设置的偏移寄存器（从 `irqchip` 状态基地址寄存器）。

+   `type_rising_mask` 是用于配置 *上升* 类型中断的掩码位。当将中断类型设置为 `IRQ_TYPE_EDGE_RISING` 时，此值将与 `type_reg_offset` 的实际内容进行 OR 运算。

+   `type_falling_mask` 是用于配置 *下降* 类型中断的掩码位。当将中断类型设置为 `IRQ_TYPE_EDGE_FALLING` 时，此值将与 `type_reg_offset` 的实际内容进行 OR 运算。对于 `IRQ_TYPE_EDGE_BOTH` 类型，将使用 `(type_falling_mask | irq_data->type_rising_mask)` 作为掩码。

现在我们熟悉了 `struct regmap_irq`，让我们描述 `struct regmap_irq_chip`，其结构如下：

```
struct regmap_irq_chip {
    const char *name;
    unsigned int status_base;
    unsigned int mask_base;
    unsigned int unmask_base;
    unsigned int ack_base;
    unsigned int wake_base;
    unsigned int type_base;
    unsigned int irq_reg_stride;
    bool mask_writeonly:1;
    bool init_ack_masked:1;
    bool mask_invert:1;
    bool use_ack:1;
    bool ack_invert:1;
    bool wake_invert:1;
    bool type_invert:1;
    int num_regs;
    const struct regmap_irq *irqs;
    int num_irqs;
    int num_type_reg;
    unsigned int type_reg_stride;
    int (*handle_pre_irq)(void *irq_drv_data);
    int (*handle_post_irq)(void *irq_drv_data);
    void *irq_drv_data;
};
```

该结构描述了一个通用的 `regmap_irq_chip`，它可以处理大多数中断控制器（并非所有，我们稍后会看到）。以下列表描述了此数据结构中的字段：

+   `name` 是中断控制器的描述性名称。

+   `status_base` 是基本状态寄存器地址，regmap IRQ 核心在获取给定 `regmap_irq` 的最终状态寄存器之前会添加 `regmap_irq.reg_offset`。

+   `mask_writeonly` 表示基本掩码寄存器是否是只写的。如果是，将使用 `regmap_write_bits()` 写入寄存器，否则使用 `regmap_update_bits()`。

+   `unmask_base` 是基本取消屏蔽寄存器地址，对于具有单独的屏蔽和取消屏蔽寄存器的芯片必须指定。

+   `ack_base` 是确认基地址寄存器。使用 `use_ack` 位时，可以使用值 `0`。

+   `wake_base` 是 `wake enable` 的基地址，用于控制中断电源管理唤醒。如果值为 `0`，表示不支持。

+   `type_base` 是 IRQ 类型的基地址，regmap IRQ 核心在获取给定 `regmap_irq` 的最终类型寄存器之前会添加 `regmap_irq.type_reg_offset`。如果为 `0`，表示不支持。

+   `irq_reg_stride` 是在寄存器不连续的芯片上使用的步幅。

+   `init_ack_masked` 表示 regmap IRQ 核心在初始化期间是否应确认所有屏蔽中断。

+   `mask_invert`，如果为 `true`，表示掩码寄存器是反转的。这意味着清除的位索引对应于被屏蔽的中断。

+   `use_ack`，如果为 `true`，表示即使为 `0`，也应该使用确认寄存器。

+   `ack_invert`，如果为 `true`，表示确认寄存器是反转的：对于确认，相应的位将被清除。

+   `wake_invert`，如果为`true`，表示唤醒寄存器被反转：清除的位对应于唤醒使能。

+   `type_invert`，如果为`true`，表示使用反转类型标志。

+   `num_regs`是每个控制块中寄存器的数量。使用`regmap_bulk_read()`时要读取的寄存器数量将被给出。查看`regmap_irq_thread()`的定义以获取更多信息。

+   `irqs`是单个 IRQ 的描述符数组，`num_irqs`是数组中描述符的总数。中断号是基于该数组中的索引分配的。

+   `num_type_reg`是类型寄存器的数量，而`type_reg_stride`是用于非连续类型寄存器芯片的步幅。Regmap IRQ 实现了通用中断服务例程，对大多数设备都是通用的。

+   一些设备，比如`MAX77620`或`MAX20024`，在服务中断之前和之后需要特殊处理。这就是`handle_pre_irq`和`handle_post_irq`发挥作用的地方。这些是特定于驱动程序的回调函数，用于在`regmap_irq_handler`处理中断之前处理设备的中断。然后`irq_drv_data`是作为参数传递给这些前/后中断处理程序的数据。例如，`MAX77620`的中断服务编程指南如下所示：

--当来自 PMIC 的中断发生时，通过设置 GLBLM 来屏蔽 PMIC 中断。

--读取 IRQTOP 并相应地服务中断。

--一旦所有中断都经过检查和服务，中断服务例程通过清除 GLBLM 来取消屏蔽硬件中断线。

回到`regmap_irq_chip.irqs`字段，这个字段是之前介绍的`regmap_irq`类型。

#### 结构体`regmap_irq_chip_data`

该结构是 regmap IRQ 控制器的运行时数据结构，在`devm_regmap_add_irq_chip()`成功返回路径上分配。它必须存储在一个大型和私有的数据结构中以供以后使用。其定义如下：

```
struct regmap_irq_chip_data {
    struct mutex lock;
    struct irq_chip irq_chip;
    struct regmap *map;
    const struct regmap_irq_chip *chip;
    int irq_base;
    struct irq_domain *domain;
    int irq;
    [...]
};
```

为简单起见，结构中的一些字段已被删除。以下是该结构中字段的描述：

+   `lock`是用于保护对`regmap_irq_chip_data`所属的`irq_chip`的访问的锁。由于 regmap IRQ 是完全线程化的，因此可以安全地使用互斥锁。

+   `irq_chip`是此 regmap 启用的`irqchip`的底层中断芯片描述符结构（提供 IRQ 相关操作），使用`regmap_irq_chip`设置，如`drivers/base/regmap/regmap-irq.c`中所定义：

```
static const struct irq_chip regmap_irq_chip = {
    .irq_bus_lock = regmap_irq_lock,
    .irq_bus_sync_unlock = regmap_irq_sync_unlock,
    .irq_disable = regmap_irq_disable,
    .irq_enable = regmap_irq_enable,
    .irq_set_type = regmap_irq_set_type,
    .irq_set_wake = regmap_irq_set_wake,
};
```

+   `map`是前述`irq_chip`的 regmap 结构。

+   `chip`是指向通用 regmap `irq_chip`的指针，它应该已经在驱动程序中设置好。它作为参数传递给`devm_regmap_add_irq_chip()`。

+   `base`，如果大于零，是从其分配特定 IRQ 编号的基数。换句话说，IRQ 的编号从`base`开始。

+   `domain`是底层 IRQ 芯片的 IRQ 域，`ops`设置为`regmap_domain_ops`，定义如下：

```
static const struct irq_domain_ops regmap_domain_ops = {
    .map = regmap_irq_map,
    .xlate = irq_domain_xlate_onetwocell,
};
```

+   `irq`是`irq_chip`的父（基本）IRQ。它对应于给定给`devm_regmap_add_irq_chip()`的`irq`参数。

### Regmap IRQ API

在本章的前面，我们介绍了`devm_regmap_add_irq_chip()`和`regmap_irq_get_virq()`作为 regmap IRQ API 的两个基本函数。这些实际上是 regmap IRQ 管理中最重要的函数，以下是它们各自的原型：

```
int devm_regmap_add_irq_chip(struct device *dev,                          struct regmap *map,
                         int irq, int irq_flags,                          int irq_base,
                         const struct regmap_irq_chip *chip,
                         struct regmap_irq_chip_data **data)
int regmap_irq_get_virq(struct regmap_irq_chip_data *data,                         int irq)
```

在前面的代码中，`dev`是`irq_chip`所属的设备指针。`map`是设备的有效和初始化的 regmap。`irq_base`，如果大于零，将是第一个分配的 IRQ 的编号。`chip`是中断控制器的配置。在`regmap_irq_get_virq()`的原型中，`*data`是一个初始化的输入参数，必须通过`devm_regmap_add_irq_chip()`通过`**data`返回。

`devm_regmap_add_irq_chip()`是您应该在代码中使用以添加基于 regmap 的 irqchip 支持的函数。它的`data`参数是一个输出参数，表示在此函数调用成功时分配的控制器的运行时数据结构。它的`irq`参数是 irqchip 的父和主要 IRQ。这是设备用于发出中断信号的 IRQ，而`irq_flags`是用于此主要中断的`IRQF_`标志的掩码。如果此函数成功（即返回`0`），则输出数据将设置为类型为`regmap_irq_chip_data`的新分配和配置良好的结构。此函数在失败时返回`errno`。`devm_regmap_add_irq_chip()`是以下内容的组合：

+   分配和初始化`struct regmap_irq_chip_data`。

+   `irq_domain_add_linear()`（如果`irq_base == 0`），它根据域中所需的 IRQ 数量分配一个 IRQ 域。成功后，IRQ 域将分配给先前分配的 IRQ 芯片数据的`.domain`字段。此域的`ops.map`函数将配置每个 IRQ 子作为嵌套到父线程中，`ops.xlate`将设置为`irq_domain_xlate_onetwocell`。如果`irq_base > 0`，则使用`irq_domain_add_legacy()`而不是`irq_domain_add_linear()`。

+   `request_threaded_irq()`，以注册父 IRQ 线程处理程序。Regmap 使用其自定义的线程处理程序`regmap_irq_thread()`，在调用子`irqs`上的`handle_nested_irq()`之前进行一些修改。

以下是总结前述操作的摘录：

```
static int regmap_irq_map(struct irq_domain *h,                           unsigned int virq,
                          irq_hw_number_t hw)
{
    struct regmap_irq_chip_data *data = h->host_data;
    irq_set_chip_data(virq, data);
    irq_set_chip(virq, &data->irq_chip);
    irq_set_nested_thread(virq, 1);
    irq_set_parent(virq, data->irq);
    irq_set_noprobe(virq);
    return 0;
}
static const struct irq_domain_ops regmap_domain_ops = {
    .map = regmap_irq_map,
    .xlate = irq_domain_xlate_onetwocell,
};
static irqreturn_t regmap_irq_thread(int irq, void *d)
{
    [...]
    for (i = 0; i < chip->num_irqs; i++) {
        if (data->status_buf[chip->irqs[i].reg_offset /
            map->reg_stride] & chip->irqs[i].mask) {
            handle_nested_irq(irq_find_mapping(data->domain,             i));
          handled = true;
        }
    }
    [...]
    if (handled)
        return IRQ_HANDLED;
    else
        return IRQ_NONE;
}
int regmap_add_irq_chip(struct regmap *map, int irq,                         int irq_ flags,
                        int irq_base,                         const struct regmap_irq_chip *chip,
                        struct regmap_irq_chip_data **data)
{
    struct regmap_irq_chip_data *d;
    [...]
    d = kzalloc(sizeof(*d), GFP_KERNEL);
    if (!d)
        return -ENOMEM;
    /* The below is just for simplicity */
    initialize_irq_chip_data(d);
    if (irq_base)
        d->domain = irq_domain_add_legacy(map->dev->of_node,
                                          chip->num_irqs,
                                          irq_base, 0,
                                          &regmap_domain_ops,                                          d);
    else
        d->domain = irq_domain_add_linear(map->dev->of_node,
                                          chip->num_irqs,
                                          &regmap_domain_ops,                                           d);
    ret = request_threaded_irq(irq, NULL, regmap_irq_thread,
                               irq_flags | IRQF_ONESHOT,
                               chip->name, d);
    [...]
    *data = d;
    return 0;
}
```

`regmap_irq_get_virq()`将芯片上的中断映射到虚拟 IRQ。它只是在给定的`irq`和域上返回`irq_create_mapping(data->domain, irq)`，就像我们之前看到的那样。它的`irq`参数是在芯片 IRQs 中请求的中断的索引。

### Regmap IRQ API 示例

让我们使用`max7760` GPIO 控制器的驱动程序来看看 regmap IRQ API 背后的概念是如何应用的。此驱动程序位于内核源中的`drivers/gpio/gpio-max77620.c`，以下是此驱动程序使用 regmap 处理 IRQ 管理的简化摘录。

让我们首先定义将在编写代码期间使用的数据结构：

```
struct max77620_gpio {
    struct gpio_chip gpio_chip;
    struct regmap *rmap;
    struct device *dev;
};
struct max77620_chip {
    struct device *dev;
    struct regmap *rmap;
    int chip_irq;
    int irq_base;
    [...]
    struct regmap_irq_chip_data *top_irq_data;
    struct regmap_irq_chip_data *gpio_irq_data;
};
```

当您阅读代码时，上述数据结构的含义将变得清晰。接下来，让我们定义我们的 regmap IRQ 数组，如下所示：

```
static const struct regmap_irq max77620_gpio_irqs[] = {
    [0] = {
        .mask = MAX77620_IRQ_LVL2_GPIO_EDGE0,
        .type_rising_mask = MAX77620_CNFG_GPIO_INT_RISING,
        .type_falling_mask = MAX77620_CNFG_GPIO_INT_FALLING,
        .reg_offset = 0,
        .type_reg_offset = 0,
    },
    [1] = {
        .mask = MAX77620_IRQ_LVL2_GPIO_EDGE1,
        .type_rising_mask = MAX77620_CNFG_GPIO_INT_RISING,
        .type_falling_mask = MAX77620_CNFG_GPIO_INT_FALLING,
        .reg_offset = 0,
        .type_reg_offset = 1,
    },
    [2] = {
        .mask = MAX77620_IRQ_LVL2_GPIO_EDGE2,
        .type_rising_mask = MAX77620_CNFG_GPIO_INT_RISING,
        .type_falling_mask = MAX77620_CNFG_GPIO_INT_FALLING,
        .reg_offset = 0,
        .type_reg_offset = 2,
    },
    [...]
    [7] = {
        .mask = MAX77620_IRQ_LVL2_GPIO_EDGE7,
        .type_rising_mask = MAX77620_CNFG_GPIO_INT_RISING,
        .type_falling_mask = MAX77620_CNFG_GPIO_INT_FALLING,
        .reg_offset = 0,
        .type_reg_offset = 7,
    },
};
```

您可能已经注意到为了可读性而对数组进行了截断。然后，可以将此数组分配给`regmap_irq_chip`数据结构，如下所示：

```
static const struct regmap_irq_chip max77620_gpio_irq_chip = {
    .name = "max77620-gpio",
    .irqs = max77620_gpio_irqs,
    .num_irqs = ARRAY_SIZE(max77620_gpio_irqs),
    .num_regs = 1,
    .num_type_reg = 8,
    .irq_reg_stride = 1,
    .type_reg_stride = 1,
    .status_base = MAX77620_REG_IRQ_LVL2_GPIO,
    .type_base = MAX77620_REG_GPIO0,
};
```

总结前述摘录，驱动程序填充了一个`regmap_irq`数组（`max77620_gpio_irqs[]`），并使用它来构建一个`regmap_irq_chip`结构（`max77620_gpio_irq_chip`）。一旦`regmap_irq_chip`数据结构准备就绪，我们开始编写一个`irqchip`回调，这是内核`gpiochip`核心所需的：

```
static int max77620_gpio_to_irq(struct gpio_chip *gc,
                                unsigned int offset)
{
    struct max77620_gpio *mgpio = gpiochip_get_data(gc);
    struct max77620_chip *chip =                          dev_get_drvdata(mgpio->dev- >parent);
    return regmap_irq_get_virq(chip->gpio_irq_data, offset);
}
```

在上述片段中，我们只定义了将分配给 GPIO 芯片的`.to_irq`字段的回调。其他回调可以在原始驱动程序中找到。再次强调，此处的代码已被截断。在这个阶段，我们可以谈论`probe`方法，它将使用之前定义的所有函数：

```
static int max77620_gpio_probe(struct platform_device *pdev)
{
     struct max77620_chip *chip =      dev_get_drvdata(pdev->dev.parent);
     struct max77620_gpio *mgpio;
     int gpio_irq;
     int ret;
     gpio_irq = platform_get_irq(pdev, 0);
     [...]
     mgpio = devm_kzalloc(&pdev->dev, sizeof(*mgpio),                           GFP_KERNEL);
     if (!mgpio)
         return -ENOMEM;
     mgpio->rmap = chip->rmap;
     mgpio->dev = &pdev->dev;
     /* setting gpiochip stuffs*/
     mgpio->gpio_chip.direction_input =                                 max77620_gpio_dir_input;
     mgpio->gpio_chip.get = max77620_gpio_get;
     mgpio->gpio_chip.direction_output =                                 max77620_gpio_dir_output;
     mgpio->gpio_chip.set = max77620_gpio_set;
     mgpio->gpio_chip.set_config = max77620_gpio_set_config;
     mgpio->gpio_chip.to_irq = max77620_gpio_to_irq;
     mgpio->gpio_chip.ngpio = MAX77620_GPIO_NR;
     mgpio->gpio_chip.can_sleep = 1;
     mgpio->gpio_chip.base = -1;
     #ifdef CONFIG_OF_GPIO
     mgpio->gpio_chip.of_node = pdev->dev.parent->of_node;
     #endif
     ret = devm_gpiochip_add_data(&pdev->dev,
                                  &mgpio->gpio_chip, mgpio);
     [...]
     ret = devm_regmap_add_irq_chip(&pdev->dev,
                                    chip->rmap, gpio_irq,
                                    IRQF_ONESHOT, -1,
                                    &max77620_gpio_irq_chip,
                                    &chip->gpio_irq_data);
     [...]
     return 0;
}
```

在这个`probe`方法摘录中（没有错误检查），`max77620_gpio_irq_chip`最终被提供给`devm_regmap_add_irq_chip`，以便用 IRQ 填充 irqchip，然后将 IRQ 芯片添加到 regmap 核心。这个函数还使用一个有效的`regmap_irq_chip_data`结构设置了`chip->gpio_irq_data`，而`chip`是私有数据结构，允许我们存储这个 IRQ 芯片数据以供以后使用。由于这个 IRQ 控制器是建立在 GPIO 控制器（`gpiochip`）之上的，所以必须设置`gpio_chip.to_irq`字段，这里就是`max77620_gpio_to_irq`回调函数。这个回调函数简单地返回`regmap_irq_get_virq()`返回的值，后者根据给定的偏移量在`regmap_irq_chip_data.domain`中创建并返回一个有效的`irq`映射。其他函数已经被介绍过，对我们来说并不新鲜。

在本节中，我们介绍了使用 regmap 进行完整的 IRQ 管理。你已经准备好将基于 MMIO 的 IRQ 管理迁移到 regmap。

# 总结

这一章主要涉及 regmap 核心。我们介绍了这个框架，走过了它的 API，并描述了一些用例。除了寄存器访问，我们还学会了如何使用 regmap 进行基于 MMIO 的 IRQ 管理。下一章将涉及 MFD 设备和 syscon 框架，将大量使用本章学到的概念。在本章结束时，你应该能够开发支持 regmap 的 IRQ 控制器，并且不会发现自己在为寄存器访问重新发明轮子并利用这个框架。
