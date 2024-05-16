# 第三章：*第三章*：深入研究 MFD 子系统和 Syscon API

设备的日益密集集成导致了一种由多个其他设备或 IP 组成的设备，可以实现专用功能。随着这种设备的出现，Linux 内核中出现了一个新的子系统。这些是**MFDs**，代表**多功能设备**。这些设备在物理上看起来是独立的设备，但从软件角度来看，它们在父子关系中表示，其中子设备是子设备。

一些基于 I2C 和 SPI 的设备/子设备可能需要一些黑客或配置才能被添加到系统中，还有一些基于 MMIO 的设备/子设备，它们不需要任何配置或黑客，只需要在子设备之间共享主设备的寄存器区域。简单的 mfd 助手被引入来处理零配置/黑客子设备的注册，syscon 被引入来与其他设备共享设备的内存区域。由于 regmap 处理 MMIO 寄存器并管理对内存的访问（也称为同步），因此将 syscon 构建在 regmap 之上是一个自然的选择。为了熟悉 MFD 子系统，在本章中，我们将首先介绍 MFD，您将了解其数据结构和 API，然后我们将研究设备树绑定，以便向内核描述这些设备。最后，我们将讨论 syscon 并介绍简单的 mfd 驱动程序，用于零配置/黑客子设备。

本章将涵盖以下主题：

+   介绍 MFD 和 syscon API 和数据结构

+   MFD 设备的设备树绑定

+   了解 syscon 和简单的 mfd

# 技术要求

为了利用本章，您需要以下内容：

+   C 编程技能

+   良好的 Linux 设备驱动程序模型知识

+   Linux 内核 v4.19.X 源代码，可在[`git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags)上找到

# 介绍 MFD 子系统和 Syscon API

在深入研究 syscon 框架及其 API 之前，我们将介绍 MFD。有些外围设备或硬件块通过它们嵌入的子设备来提供多个功能，这些子设备由内核中的单独子系统处理。也就是说，子设备是所谓的多功能设备中的专用实体，负责特定任务，并通过芯片的寄存器映射中的一组减少的寄存器进行管理。`ADP5520`是 MFD 设备的典型示例，因为它包含背光、键盘、LED 和 GPIO 控制器。每个子设备都被视为一个子设备，正如您所看到的，每个子设备都属于不同的子系统。MFD 子系统定义在`include/linux/mfd/core.h`中，并在`drivers/mfd/mfd-core.c`中实现，用于处理这些设备，允许以下功能：

+   在多个子系统中注册相同的设备

+   复用总线和寄存器访问，因为可能有一些寄存器在子设备之间共享

+   处理中断和时钟

在本节中，我们将研究来自对话半导体的`da9055`设备的驱动程序，位于内核源树中的`drivers/mfd/da9055-core.c`。该设备的数据手册可以在[`www.dialog-semiconductor.com/sites/default/files/da9055-00-ids3a_20120710.pdf`](https://www.dialog-semiconductor.com/sites/default/files/da9055-00-ids3a_20120710.pdf)找到。

在大多数情况下，MFD 设备驱动程序由两部分组成：

+   `drivers/mfd`，负责主要初始化并将每个子设备注册为系统上的平台设备（以及其平台数据）。该驱动程序应为子设备驱动程序提供通用服务。这些服务包括寄存器访问、控制和共享中断管理。当一个子系统的平台驱动程序被实例化时，核心初始化芯片（可以由平台数据指定）。单个内核映像中可以构建相同类型的多个块设备的支持。这要归功于平台数据的机制。内核中使用平台特定数据抽象机制来将配置传递给核心，并且子驱动程序使得支持多个相同类型的块设备成为可能。

+   **子设备驱动程序**，负责处理核心驱动程序早期注册的特定子设备。这些驱动程序位于各自的子系统目录中。每个外围（子系统设备）对设备有一个有限的视图，隐式地缩小到外围需要为了正确运行所需的特定资源集。

重要提示

本章中的子设备概念不应与*第七章*中的同名概念混淆，*解密 V4L2 和视频捕获设备驱动程序*，稍有不同，其中子设备还代表视频管道中的实体。

在 MFD 子系统中，子设备由`struct mfd_cell`结构的实例表示，您可以称之为`struct mfd_cell`结构，您可以指定更高级的东西，例如子设备使用的资源和挂起-恢复操作（从子设备的驱动程序调用）。该结构如下所示，为简化原因删除了一些字段：

```
/*
 * This struct describes the MFD part ("cell").
 * After registration the copy of this structure will
 * become the platform data of the resulting platform_device
 */
struct mfd_cell {
    const char *name;
    int id;
    [...]
    int (*suspend)(struct platform_device *dev);
    int (*resume)(struct platform_device *dev);
    /* platform data passed to the sub devices drivers */
    void *platform_data;
    size_t pdata_size;
    /* Device Tree compatible string */
    const char *of_compatible;
    /* Matches ACPI */
    const struct mfd_cell_acpi_match *acpi_match;
    /*
     * These resources can be specified relative to the
     * parent device. For accessing hardware, you should
     * use resources from the platform dev
     */
    int num_resources;
    const struct resource *resources;
    [...]
};
```

重要提示

创建的新平台设备将具有该单元结构作为其平台数据。然后可以通过`pdev->mfd_cell->platform_data`访问真实的平台数据。驱动程序还可以使用`mfd_get_cell()`来检索与平台设备对应的 MFD 单元：`const struct mfd_cell *cell = mfd_get_cell(pdev);`。

该结构的每个成员的功能是不言自明的。但以下内容会给您更多细节。

`.resources`元素是表示子设备特定资源的数组（也是平台设备），`.num_resources`是数组中条目的数量。这些是使用`platform_data`定义的，您可能希望为其命名以便轻松检索。以下是一个原始核心源文件为`drivers/mfd/da9055-core.c`的 MFD 驱动程序的示例：

```
static struct resource da9055_rtc_resource[] = {
    {
        .name = „ALM",
        .start = DA9055_IRQ_ALARM,
        .end = DA9055_IRQ_ALARM,
        .flags = IORESOURCE_IRQ,
    },
    {
        .name = "TICK",
        .start = DA9055_IRQ_TICK,
        .end = DA9055_IRQ_TICK,
        .flags = IORESOURCE_IRQ,
    },
};
static const struct mfd_cell da9055_devs[] = {
    ...
    {
        .of_compatible = "dlg,da9055-rtc",
        .name = "da9055-rtc",
        .resources = da9055_rtc_resource,
        .num_resources = ARRAY_SIZE(da9055_rtc_resource),
    },
    ...
};
```

以下示例显示了如何从子设备驱动程序中检索资源，本例中实现在`drivers/rtc/rtc-da9055.c`中：

```
static int da9055_rtc_probe(struct platform_device *pdev)
{
    [...]
    alm_irq = platform_get_irq_byname(pdev, "ALM");
    if (alm_irq < 0)
        return alm_irq;
    ret = devm_request_threaded_irq(&pdev->dev, alm_irq, NULL,
                                    da9055_rtc_alm_irq,
                                    IRQF_TRIGGER_HIGH |                                     IRQF_ONESHOT,
                                    "ALM", rtc);
    if (ret != 0)
        dev_err(rtc->da9055->dev,
                 "irq registration failed: %d\n", ret);
    [...]
}
```

实际上，您应该使用`platform_get_resource()`、`platform_get_resource_byname()`、`platform_get_irq()`和`platform_get_irq_byname()`来检索资源。

在使用`.of_compatible`时，该函数必须是 MFD 的子级（参见*MFD 设备的设备树绑定*部分）。您应该静态填充一个包含与设备上的子设备数量相同的条目的结构数组：

```
static struct resource da9055_rtc_resource[] = {
    {
        .name = „ALM",
        .start = DA9055_IRQ_ALARM,
        .end = DA9055_IRQ_ALARM,
        .flags = IORESOURCE_IRQ,
    },
    [...]
};
[...]
static const struct mfd_cell da9055_devs[] = {
    {
        .of_compatible = "dlg,da9055-gpio",
        .name = "da9055-gpio",
    },
    {
        .of_compatible = "dlg,da9055-regulator",
        .name = "da9055-regulator",
        .id = 1,
    },
    [...]
    {
        .of_compatible = "dlg,da9055-rtc",
        .name = "da9055-rtc",
        .resources = da9055_rtc_resource,
        .num_resources = ARRAY_SIZE(da9055_rtc_resource),
    },
    {
        .of_compatible = "dlg,da9055-watchdog",
        .name = "da9055-watchdog",
    },
};
```

填充`struct mfd_cell`数组后，必须将其传递给`devm_mfd_add_devices()`函数，如下所示：

```
int devm_mfd_add_devices(
                struct device *dev,
                int id,
                const struct mfd_cell *cells,
                int n_devs,
                struct resource *mem_base,
                int irq_base,
                struct irq_domain *domain)
```

该方法的参数解释如下：

+   `dev`是 MFD 芯片的通用结构设备。它将用于设置子设备的父级。

+   `id`：由于子设备被创建为平台设备，因此应该为其分配一个 ID。对于自动 ID 分配，应将此字段设置为`PLATFORM_DEVID_AUTO`，在这种情况下，相应单元的`mfd_cell.id`将被忽略。否则，您应该使用`PLATFORM_DEVID_NONE`。

+   `cells`是一个指向描述子设备的`struct mfd_cell`结构的列表（实际上是一个数组）的指针。

+   `n_dev`是要在数组中使用的`struct mfd_cell`条目的数量，以创建平台设备。要创建与数组中的单元数量相同的平台设备，应使用`ARRAY_SIZE()`宏。

+   `mem_base`：如果不是`NULL`，其`.start`字段将用作先前提到的数组中每个 MFD 单元的`IORESOURCE_MEM`类型资源的基址。以下是`mfd_add_device()`函数的摘录，显示了这一点：

```
for (r = 0; r < cell->num_resources; r++) {
    res[r].name = cell->resources[r].name;
    res[r].flags = cell->resources[r].flags;
    /* Find out base to use */
    if ((cell->resources[r].flags & IORESOURCE_MEM) &&          mem_base) {
         res[r].parent = mem_base;
         res[r].start =
             mem_base->start + cell->resources[r].start;
         res[r].end =
             mem_base->start + cell->resources[r].end;
    } else if (cell->resources[r].flags & IORESOURCE_IRQ) {
[...]
```

+   `irq_base`：如果设置了域，则此参数将被忽略。否则，它的行为类似于`mem_base`，但对于每个`IORESOURCE_IRQ`类型的资源。以下是`mfd_add_device()`函数的摘录，显示了这一点：

```
    } else if (cell->resources[r].flags & IORESOURCE_IRQ) {
        if (domain) {
          /* Unable to create mappings for IRQ ranges. */
            WARN_ON(cell->resources[r].start !=
                            cell->resources[r].end);
            res[r].start = res[r].end =
                irq_create_mapping(
                        domain,cell->resources[r].start);
        } else {
            res[r].start =
                irq_base + cell->resources[r].start;
            res[r].end =
                irq_base + cell->resources[r].end;
        }
    } else {
    [...]
```

+   `domain`：对于同时扮演子设备的 IRQ 控制器角色的 MFD 芯片，此参数将用作 IRQ 域，为这些子设备创建 IRQ 映射。它的工作方式是：对于每个单元中类型为`IORESOURCE_IRQ`的资源`r`，MFD 核心将创建一个相同类型的新资源`res`（实际上是一个 IRQ 资源，其`res.start`和`res.end`字段设置为与此域中对应于初始资源的`.start`字段的 IRQ 映射：`res[r].start = res[r].end = irq_create_mapping(domain, cell->resources[r].start);`）。然后，新的 IRQ 资源被分配给当前单元的平台设备，并对应于其 virqs。请查看上述摘录中的前一个参数描述。请注意，此参数可以是`NULL`。

现在让我们看看如何将这些内容与`da9055` MFD 驱动程序的摘录放在一起：

```
#define DA9055_IRQ_NONKEY_MASK 0x01
#define DA9055_IRQ_ALM_MASK 0x02
#define DA9055_IRQ_TICK_MASK 0x04
#define DA9055_IRQ_ADC_MASK 0x08
#define DA9055_IRQ_BUCK_ILIM_MASK 0x08
/*
 * PMIC IRQ
 */
#define DA9055_IRQ_ALARM 0x01
#define DA9055_IRQ_TICK 0x02
#define DA9055_IRQ_NONKEY 0x00
#define DA9055_IRQ_REGULATOR 0x0B
#define DA9055_IRQ_HWMON 0x03
struct da9055 {
    struct regmap *regmap;
    struct regmap_irq_chip_data *irq_data;
    struct device *dev;
    struct i2c_client *i2c_client;
    int irq_base;
    int chip_irq;
};
```

在上述摘录中，驱动程序定义了一些常量，以及一个私有数据结构，其含义将在阅读代码时清楚。之后，为寄存器映射核心定义了 IRQ，如下：

```
static const struct regmap_irq da9055_irqs[] = {
    [DA9055_IRQ_NONKEY] = {
        .reg_offset = 0,
        .mask = DA9055_IRQ_NONKEY_MASK,
    },
    [DA9055_IRQ_ALARM] = {
        .reg_offset = 0,
        .mask = DA9055_IRQ_ALM_MASK,
    },
    [DA9055_IRQ_TICK] = {
        .reg_offset = 0,
        .mask = DA9055_IRQ_TICK_MASK,
    },
    [DA9055_IRQ_HWMON] = {
        .reg_offset = 0,
        .mask = DA9055_IRQ_ADC_MASK,
    },
    [DA9055_IRQ_REGULATOR] = {
        .reg_offset = 1,
        .mask = DA9055_IRQ_BUCK_ILIM_MASK,
    },
};
static const struct regmap_irq_chip da9055_regmap_irq_chip = {
    .name = "da9055_irq",
    .status_base = DA9055_REG_EVENT_A,
    .mask_base = DA9055_REG_IRQ_MASK_A,
    .ack_base = DA9055_REG_EVENT_A,
    .num_regs = 3,
    .irqs = da9055_irqs,
    .num_irqs = ARRAY_SIZE(da9055_irqs),
};
```

在上述摘录中，`da9055_irqs`是`regmap_irq`类型的元素数组，描述了通用的 regmap IRQ。它被分配给`da9055_regmap_irq_chip`，它是`regmap_irq_chip`类型，代表了 regmap IRQ 芯片。两者都是 regmap IRQ 数据结构集的一部分。最后，`probe`方法被实现如下：

```
static int da9055_i2c_probe(struct i2c_client *client,
                            const struct i2c_device_id *id)
{
    int ret;
    struct da9055_pdata *pdata = dev_get_platdata(da9055->dev);
    uint8_t clear_events[3] = {0xFF, 0xFF, 0xFF};
    [...]
    ret =
        devm_regmap_add_irq_chip(
            &client->dev, da9055->regmap,
            da9055->chip_irq, IRQF_TRIGGER_LOW | IRQF_ONESHOT,
            da9055->irq_base, &da9055_regmap_irq_chip,
            &da9055->irq_data);
    if (ret < 0)
            return ret;
    da9055->irq_base = regmap_irq_chip_get_base(                       da9055->irq_data);
    ret = devm_mfd_add_devices(
                    da9055->dev, -1,
                    da9055_devs, ARRAY_SIZE(da9055_devs),
                    NULL, da9055->irq_base,
                    regmap_irq_get_domain(da9055->irq_data));
    if (ret)
        goto err;
    [...]
}
```

在上述的探测方法中，`da9055_regmap_irq_chip`（之前定义的）作为参数传递给`regmap_add_irq_chip()`，以便向 IRQ 核心添加一个有效的 regmap IRQ 控制器。该函数成功返回`0`。此外，它还通过最后一个参数返回一个完全配置的`regmap_irq_chip_data`结构，可以作为控制器的运行时数据结构后续使用。这个`regmap_irq_chip_data`结构将包含与先前添加的 IRQ 控制器相关联的 IRQ 域。最终，这个 IRQ 域作为参数传递给`devm_mfd_add_devices()`，以及 MFD 单元的数组和单元数量。

重要提示

请注意，`devm_mfd_add_devices()`实际上是`mfd_add_devices()`的资源管理版本，其具有以下函数调用序列：

```
mfd_add_devices()-> mfd_add_device()-> platform_device_alloc()                             -> platform_device_add_data()                             -> platform_device_add_resources()                             -> platform_device_add()
```

有些 I2C 芯片的芯片本身和内部子设备具有不同的 I2C 地址。这样的 I2C 子设备不能作为 I2C 客户端进行探测，因为 MFD 核心只实例化给定 MFD 单元的平台设备。这个问题通过以下方式解决：

+   创建一个虚拟 I2C 客户端，给定子设备的 I2C 地址和 MFD 芯片的适配器。这实际上对应于管理 MFD 设备的适配器（总线）。这可以使用`i2c_new_dummy()`来实现。返回的 I2C 客户端应该保存以备以后使用，例如，使用`i2c_unregister_device()`，在模块被卸载时应调用。

+   如果子设备需要自己的 regmap，则必须在其虚拟 I2C 客户端的基础上构建此 regmap。

+   仅存储 I2C 客户端（以备以后删除）或将其与 regmap 一起存储在可以分配给底层平台设备的私有数据结构中。

总结前面的步骤，让我们通过一个真实 MFD 设备 max8925 的驱动程序来走一遍（主要是电源管理 IC，但也由大量子设备组成）。我们的代码是原始代码的摘要（仅处理两个子设备），函数名称经过修改以便阅读。也就是说，原始驱动程序可以在内核源树中的`drivers/mfd/max8925-i2c.c`中找到。

让我们从上下文数据结构定义开始，如下所示：

```
struct priv_chip {
    struct device *dev;
    struct regmap *regmap;
    /* chip client for the parent chip, let's say the PMIC */
    struct i2c_client *client;
    /* chip client for subdevice 1, let's say an rtc */
    struct i2c_client *subdev1_client;
    /* chip client for subdevice 2 let's say a gpio controller      */
    struct i2c_client *subdev2_client;
    struct regmap *subdev1_regmap;
    struct regmap *subdev2_regmap;
    unsigned short subdev1_addr; /* subdevice 1 I2C address */
    unsigned short subdev2_addr; /* subdevice 2 I2C address */
};
const struct regmap_config chip_regmap_config = {
    [...]
};
const struct regmap_config subdev_rtc_regmap_config = {
    [...]
};
const struct regmap_config subdev_gpiochip_regmap_config = {
    [...]
};
```

在前面的摘录中，驱动程序定义了上下文数据结构`struct priv_chip`，其中包含子设备 regmaps，然后初始化了 MFD 设备 regmap 配置以及子设备自身的配置。然后，定义了`probe`方法，如下所示：

```
static int my_mfd_probe(struct i2c_client *client,
                        const struct i2c_device_id *id)
{
    struct priv_chip *chip;
    struct regmap *map;
    chip = devm_kzalloc(&client->dev,
                        sizeof(struct priv_chip), GFP_KERNEL);
    map = devm_regmap_init_i2c(client, &chip_regmap_config);
    chip->client = client;
    chip->regmap = map;
    chip->dev = &client->dev;
    dev_set_drvdata(chip->dev, chip);
    i2c_set_clientdata(chip->client, chip);
    chip->subdev1_addr = client->addr + 1;
    chip->subdev2_addr = client->addr + 2;
    /* subdevice 1, let's say an RTC */
    chip->subdev1_client = i2c_new_dummy(client->adapter,
                                         chip->subdev1_addr);
    chip->subdev1_regmap =
         devm_regmap_init_i2c(chip->subdev1_client,
                              &subdev_rtc_regmap_config);
    i2c_set_clientdata(chip->subdev1_client, chip);
    /* subdevice 2, let's say a gpio controller */
    chip->subdev2_client = i2c_new_dummy(client->adapter,
                                           chip->subdev2_addr);
    chip->subdev2_regmap =
        devm_regmap_init_i2c(chip->subdev2_client,
                             &subdev_gpiochip_regmap_config);
    i2c_set_clientdata(chip->subdev2_client, chip);
    /* mfd_add_devices() is called somewhere */
    [...]
}
```

为了便于阅读，前面的摘录省略了错误检查。此外，以下代码显示了如何删除虚拟 I2C 客户端：

```
static int my_mfd_remove(struct i2c_client *client)
{
    struct priv_chip *chip = i2c_get_clientdata(client);
    mfd_remove_devices(chip->dev);
    i2c_unregister_device(chip->subdev1_client);
    i2c_unregister_device(chip->subdev2_client);
    return 0;
}
```

最后，以下简化的代码显示了子设备驱动程序如何获取设置在 MFD 驱动程序中的 regmap 数据结构的指针：

```
static int subdev_rtc_probe(struct platform_device *pdev)
{
    struct priv_chip *chip = dev_get_drvdata(pdev->dev.parent);
    struct regmap *rtc_regmap = chip->subdev1_regmap;
    int ret;
    [...]
    if (!rtc_regmap) {
        dev_err(&pdev->dev, "no regmap!\n");
        ret = -EINVAL;
        goto out;
    }
    [...]
}
```

尽管我们已经掌握了开发 MFD 设备驱动程序所需的大部分知识，但是有必要将其与设备树集成，以便更好地（即非硬编码）描述我们的 MFD 设备。这是我们将在下一节中讨论的内容。

# MFD 设备的设备树绑定

尽管我们有必要的工具和输入来编写自己的 MFD 驱动程序，但是重要的是底层 MFD 设备在设备树中有其描述，因为这样可以让 MFD 核心知道我们的 MFD 设备由什么组成以及如何处理它。此外，设备树仍然是声明设备的正确位置，无论它们是否是 MFD。请记住，它的目的只是描述系统上的设备。由于子设备是构建它们的 MFD 设备的子级（存在从属关系），因此在父节点下声明这些子设备节点是一个良好的做法，就像以下示例中所示的那样。此外，子设备使用的资源有时是父设备的资源的一部分。因此，它强调了将子设备节点放在主设备节点下的想法。在每个子设备节点中，兼容属性应该与子设备的`cell.of_compatible`字段和子设备的`platform_driver.of_match_table`数组中的一个`.compatible`字符串条目之一匹配，或者与子设备的`cell.name`字段和子设备的`platform_driver.name`字段匹配：

重要说明

子设备的`cell.of_compatible`和`cell.name`字段是在 MFD 核心驱动程序中的子设备的`mfd_cell`结构中声明的。

```
&i2c3 {
    pinctrl-names = "default";
    pinctrl-0 = <&pinctrl_i2c3>;
    clock-frequency = <400000>;
    status = "okay";
    pmic0: da9062@58 {
        compatible = "dlg,da9062";
        reg = <0x58>;
        pinctrl-names = "default";
        pinctrl-0 = <&pinctrl_pmic>;
        interrupt-parent = <&gpio6>;
        interrupts = <11 IRQ_TYPE_LEVEL_LOW>;
        interrupt-controller;
        regulators {
            DA9062_BUCK1: buck1 {
                regulator-name = "BUCK1";
                regulator-min-microvolt = <300000>;
                regulator-max-microvolt = <1570000>;
                regulator-min-microamp = <500000>;
                regulator-max-microamp = <2000000>;
                regulator-boot-on;
            };
            DA9062_LDO1: ldo1 {
                regulator-name = "LDO_1";
                regulator-min-microvolt = <900000>;
                regulator-max-microvolt = <3600000>;
                regulator-boot-on;
            };
        };
        da9062_rtc: rtc {
            compatible = "dlg,da9062-rtc";
        };
        watchdog {
            compatible = "dlg,da9062-watchdog";
        };
        onkey {
            compatible = "dlg,da9062-onkey";
            dlg,disable-key-power;
        };
    };
};
```

在前面的设备树示例中，父节点（实际上是`da9092`的`da9062`节点）。让我们专注于子设备的`compatible`属性，并以`onkey`为例。此节点的 MFD 单元在 MFD 核心驱动程序中声明（源文件为`drivers/mfd/da9063-core.c`），如下所示：

```
static struct resource da9063_onkey_resources[] = {
    {
        .name = "ONKEY",
        .start = DA9063_IRQ_ONKEY,
        .end = DA9063_IRQ_ONKEY,
        .flags = IORESOURCE_IRQ,d
    },
};
static const struct mfd_cell da9062_devs[] = {
    [...]
    {
        .name = "da9062-onkey",
        .num_resources = ARRAY_SIZE(da9062_onkey_resources),
        .resources = da9062_onkey_resources,
        .of_compatible = "dlg,da9062-onkey",
    },
};
```

现在，这个`onekey`平台驱动程序结构被声明（以及其`.of_match_table`条目）在驱动程序中（源文件为`drivers/input/misc/da9063_onkey.c`），如下所示：

```
static const struct of_device_id da9063_compatible_reg_id_table[] = {
    { .compatible = "dlg,da9063-onkey", .data = &da9063_regs },
    { .compatible = "dlg,da9062-onkey", .data = &da9062_regs },
    { },
};
MODULE_DEVICE_TABLE(of, da9063_compatible_reg_id_table);
[...]
static struct platform_driver da9063_onkey_driver = {
    .probe = da9063_onkey_probe,
    .driver = {
        .name = DA9063_DRVNAME_ONKEY,
        .of_match_table = da9063_compatible_reg_id_table,
    },
};
```

您可以看到两个`compatible`字符串与设备节点中的`compatible`字符串匹配。另一方面，我们可以看到同一平台驱动程序可能用于两个或更多（子）设备。然后使用名称匹配会很令人困惑。这就是为什么您会使用设备树进行声明和`compatible`字符串进行匹配的原因。到目前为止，我们已经了解了 MFD 子系统如何处理设备以及反之。在下一节中，我们将把这些概念扩展到 syscon 和 simple-mfd，这是两个有助于 MFD 驱动程序开发的框架。

# 理解 Syscon 和 simple-mfd

**Syscon**代表**系统控制器**。SoC 有时会有一组专用于与特定 IP 无关的杂项功能的 MMIO 寄存器。显然，对于这种情况，不能有功能驱动程序，因为这些寄存器既不具有代表性，也不足以代表特定类型的设备。syscon 驱动程序处理这种情况。Syscon 允许其他节点通过 regmap 机制访问此寄存器空间。实际上，它只是 regmap 的一组包装 API。当您请求访问 syscon 时，如果尚不存在，则会创建 regmap。

使用 syscon API 所需的头文件是`<linux/mfd/syscon.h>`。由于此 API 基于 regmap，因此还必须包括`<linux/regmap.h>`。syscon API 在内核源树中的`drivers/mfd/syscon.c`中实现。其主要数据结构是`struct syscon`，尽管不应直接使用此结构：

```
struct syscon {
    struct device_node *np;
    struct regmap *regmap;
    struct list_head list;
};
```

在上述结构中，`np`是指向充当 syscon 的节点的指针。它还用于通过设备节点进行 syscon 查找。`regmap`是与此 syscon 关联的 regmap，`list`用于实现内核链表机制，用于将系统中的所有 syscon 连接到系统范围的列表`syscon_list`中，该列表在`drivers/mfd/syscon.c`中定义。此链表机制允许遍历整个 syscon 列表，无论是通过节点匹配还是通过 regmap 匹配。

Syscon 是通过在应充当 Syscon 的设备节点的兼容字符串列表中添加`"syscon"`来专门声明的。在早期引导期间，具有其兼容字符串列表中的`syscon`的每个节点将其`reg`内存区域进行 IO 映射，并根据默认的 regmap 配置`syscon_regmap_config`绑定到 MMIO regmap，如下所示：

```
static const struct regmap_config syscon_regmap_config = {
    .reg_bits = 32,
    .val_bits = 32,
    .reg_stride = 4,
};
```

然后，创建的 syscon 将添加到 syscon 框架范围的`syscon_list`中，并由`syscon_list_slock`自旋锁保护，如下所示：

```
static DEFINE_SPINLOCK(syscon_list_slock);
static LIST_HEAD(syscon_list);
static struct syscon *of_syscon_register(struct device_node                                          *np)
{
    struct syscon *syscon;
    struct regmap *regmap;
    void __iomem *base;
    [...]
    if (!of_device_is_compatible(np, "syscon"))
        return ERR_PTR(-EINVAL);
    [...]
    spin_lock(&syscon_list_slock);
    list_add_tail(&syscon->list, &syscon_list);
    spin_unlock(&syscon_list_slock);
    return syscon;
}
```

Syscon 绑定需要以下强制属性：

+   `compatible`: 此属性值应为`"syscon"`。

+   `reg`: 这是可以从 syscon 访问的寄存器区域。

以下是可选属性，用于篡改默认的`syscon_regmap_config` regmap 配置：

+   `reg-io-width`: 应在设备上执行的 IO 访问的大小（或宽度，以字节为单位）

+   `hwlocks`: 指向硬件自旋锁提供程序节点的 phandle 的引用

下面是一个示例，摘自内核文档，完整版本可在内核源代码中的`Documentation/devicetree/bindings/mfd/syscon.txt`中找到：

```
gpr: iomuxc-gpr@20e0000 {
    compatible = "fsl,imx6q-iomuxc-gpr", "syscon";
    reg = <0x020e0000 0x38>;
    hwlocks = <&hwlock1 1>;
};
hwlock1: hwspinlock@40500000 {
    ...
    reg = <0x40500000 0x1000>;
    #hwlock-cells = <1>;
};
```

在设备树中，可以通过三种不同的方式引用 syscon 节点：通过 phandle（在此驱动程序的设备节点中指定）、通过其路径，或者通过使用特定的兼容值进行搜索，之后驱动程序可以询问节点（或关联的此 regmap 的 OS 驱动程序）以确定寄存器的位置，最后直接访问寄存器。您可以使用以下 syscon API 之一来获取与给定 syscon 节点关联的 regmap 的指针：

```
struct regmap * syscon_node_to_regmap (struct device_node *np);
struct regmap * syscon_regmap_lookup_by_compatible(const char                                                    *s);
struct regmap * syscon_regmap_lookup_by_pdevname(const char                                                  *s);
struct regmap * syscon_regmap_lookup_by_phandle(
                            struct device_node *np,
                            const char *property);
```

上述 API 具有以下描述：

+   `syscon_regmap_lookup_by_compatible()`: 给定 syscon 设备节点的兼容字符串之一，此函数返回关联的 regmap，如果尚不存在，则创建一个，然后返回它。

+   `syscon_node_to_regmap()`: 给定一个 syscon 设备节点作为参数，此函数返回关联的 regmap，如果尚不存在，则创建一个，然后返回它。

+   `syscon_regmap_lookup_by_phandle()`: 给定一个包含 syscon 节点标识符的 phandle 属性，此函数返回与此 syscon 节点对应的 regmap。

在展示使用上述 API 的示例之前，让我们介绍以下平台设备节点，我们将编写`probe`函数。为了更好地理解`syscon_node_to_regmap()`，让我们将此节点声明为先前`gpr`节点的子节点：

```
gpr: iomuxc-gpr@20e0000 {
    compatible = "fsl,imx6q-iomuxc-gpr", "syscon";
    reg = <0x020e0000 0x38>;
    my_pdev: my_pdev {
        compatible = "company,regmap-sample";
        regmap-phandle = <&gpr>;
        [...]
    };
};
```

现在定义了设备树节点，我们可以专注于驱动程序的代码，如下所示，并使用前面列举的函数：

```
static struct regmap *by_node_regmap;
static struct regmap *by_compat_regmap;
static struct regmap *by_pdevname_regmap;
static struct regmap *by_phandle_regmap;
static int my_pdev_regmap_sample(struct platform_device *pdev)
{
    struct device_node *np = pdev->dev.of_node;
    struct device_node *syscon_node;
    [...]
    syscon_node = of_get_parent(np);
    if (!syscon_node)
        return -ENODEV;
    /* If we have a pointer to the syscon device node,    we use it */
    by_node_regmap = syscon_node_to_regmap(syscon_node);
    of_node_put(syscon_node);
    if (IS_ERR(by_node_regmap)) {
        pr_err("%s: could not find regmap by node\n",         __func__);
        return PTR_ERR(by_node_regmap);
    }
    /* or we have one of the compatible string of the syscon     node */
    by_compat_regmap =
        syscon_regmap_lookup_by_compatible("fsl,        imx6q-iomuxc-gpr");
    if (IS_ERR(by_compat_regmap)) {
        pr_err("%s: could not find regmap by compatible\n",         __func__);
        return PTR_ERR(by_compat_regmap);
    }
    /* Or a phandle property pointing to the syscon device node             
     */
    by_phandle_regmap =
        syscon_regmap_lookup_by_phandle(np, "fsl,tempmon");
    if (IS_ERR(map)) {
        pr_err("%s: could not find regmap by phandle\n",         __func__);
        return PTR_ERR(by_phandle_regmap);
    }
    /*
     * It is the extrem and rare case fallback
     * As of Linux kernel v4.18, there is only one driver
     * using this, drivers/tty/serial/clps711x.c
     */
    char pdev_syscon_name[9];
    int index = pdev->id;
    sprintf(syscon_name, "syscon.%i", index + 1);
    by_pdevname_regmap =
        syscon_regmap_lookup_by_pdevname(syscon_name);
    if (IS_ERR(by_pdevname_regmap)) {
        pr_err("%s: could not find regmap by pdevname\n",         __func__);
        return PTR_ERR(by_pdevname_regmap);
    }
    [...]
    return 0;
}
```

在前面的示例中，如果我们假设`syscon_name`包含`gpr`设备的平台设备名称，那么`by_node_regmap`、`by_compat_regmap`、`by_pdevname_regmap`和`by_phandle_regmap`变量将指向相同的 syscon regmap。然而，这里的目的只是解释概念。`my_pdev`可能是`gpr`的兄弟（或其他关系）节点。在这里使用它作为其子节点是为了理解概念和代码，并展示根据情况使用任一 API。现在我们熟悉了 syscon 框架，让我们看看它如何与 simple-mfd 一起使用。

## 介绍 simple-mfd

对于基于 MMIO 的 MFD 设备，在将它们添加到系统之前可能不需要配置子设备。因为这个配置是在 MFD 核心驱动程序内部完成的，所以这个 MFD 核心驱动程序的唯一目标将是向系统中添加平台子设备。由于存在许多基于 MMIO 的 MFD 设备，将会有大量冗余代码。简单的 MFD，即简单的 DT 绑定，解决了这个问题。

当将`simple-mfd`字符串添加到给定设备节点的兼容字符串列表中（在这里被视为 MFD 设备），它将使`for_each_child_of_node()`迭代器。simple-mfd 在`drivers/of/platform.c`中实现为 simple-bus 的别名，其文档位于内核源树中的`Documentation/devicetree/bindings/mfd/mfd.txt`中。

与 syscon 一起使用以创建 regmap，有助于避免编写 MFD 驱动程序，开发人员可以将精力放在编写子设备驱动程序上。以下是一个例子：

```
snvs: snvs@20cc000 {
    compatible = "fsl,sec-v4.0-mon", "syscon", "simple-mfd";
    reg = <0x020cc000 0x4000>;
    snvs_rtc: snvs-rtc-lp {
        compatible = "fsl,sec-v4.0-mon-rtc-lp";
        regmap = <&snvs>;
        offset = <0x34>;
        interrupts = <GIC_SPI 19 IRQ_TYPE_LEVEL_HIGH>,
                     <GIC_SPI 20 IRQ_TYPE_LEVEL_HIGH>;
    };
    snvs_poweroff: snvs-poweroff {
        compatible = "syscon-poweroff";
        regmap = <&snvs>;
        offset = <0x38>;
        value = <0x60>;
        mask = <0x60>;
        status = "disabled";
    };
    snvs_pwrkey: snvs-powerkey {
        compatible = "fsl,sec-v4.0-pwrkey";
        regmap = <&snvs>;
        interrupts = <GIC_SPI 4 IRQ_TYPE_LEVEL_HIGH>;
        linux,keycode = <KEY_POWER>;
        wakeup-source;
    };
    [...]
};
```

在前面的设备树摘录中，`snvs`是主设备。它由一个电源控制子设备（在主设备寄存器区域中表示为一个寄存器子区域）、一个`rtc`子设备以及一个电源键等组成。整个定义可以在`arch/arm/boot/dts/imx6qdl.dtsi`中找到，这是 i.MX6 芯片系列的 SoC 供应商`dtsi`。相应的驱动程序可以通过在内核源代码中搜索它们的`compatible`属性的内容来找到。总之，对于`snvs`节点中的每个子节点，MFD 核心将创建一个相应的设备以及其 regmap，该 regmap 将对应于它们在主设备的内存区域中的内存区域。

本节介绍了在处理 MMIO 设备时如何轻松进入 MFD 驱动程序开发。虽然 SPI/I2C 设备不属于这一类别，但它涵盖了几乎 95%的基于 MMIO 的 MFD 设备。

# 总结

本章讨论了 MFD 设备以及 syscon 和 regmap API。在这里，我们讨论了 MFD 设备的工作原理以及 regmap 是如何嵌入到 syscon 中的。到达本章末尾时，我们可以假设您能够开发支持 regmap 的 IRQ 控制器，并设计和使用 syscon 在设备之间共享寄存器区域。下一章将涉及通用时钟框架以及该框架的组织结构、实现方式、使用方法以及如何添加自己的时钟。
