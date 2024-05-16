# 第十二章：*第十二章*：利用 NVMEM 框架

`drivers/misc/`，在大多数情况下，每个驱动程序都必须实现自己的 API 来处理相同的功能，无论是为内核用户还是向用户空间公开其内容。结果表明，这些驱动程序严重缺乏抽象代码。此外，内核对这些设备的支持不断增加，导致了大量的代码重复。

在内核中引入此框架的目的是解决先前提到的这些问题。它还为消费者设备引入了 DT 表示，以从 NVMEM 获取它们需要的数据（MAC 地址、SoC/修订 ID、零件号等）。我们将从介绍 NVMEM 数据结构开始本章，这是必须了解该框架的内容，然后我们将看看 NVMEM 提供者驱动程序，学习如何将 NVMEM 内存区域暴露给消费者。最后，我们将学习 NVMEM 消费者驱动程序，以利用提供者公开的内容。

在本章中，我们将涵盖以下主题：

+   介绍 NVMEM 数据结构和 API

+   编写 NVMEM 提供者驱动程序

+   NVMEM 消费者驱动 API

# 技术要求

以下是本章的先决条件：

+   C 编程技能

+   内核编程和设备驱动程序开发技能

+   Linux 内核 v4.19.X 源码，可在[`git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags)获取

# 介绍 NVMEM 数据结构和 API

NVMEM 是一个具有减少 API 和数据结构集的小型框架。在本节中，我们将介绍这些 API 和数据结构，以及**cell**的概念，这是该框架的基础。

NVMEM 基于生产者/消费者模式，就像*第四章*中描述的时钟框架一样，*Storming the Common Clock Framework*。NVMEM 设备只有一个驱动程序，公开设备单元格，以便消费者驱动程序可以访问和操作它们。虽然 NVMEM 设备驱动程序必须包括`<linux/nvmem-provider.h>`，但消费者必须包括`<linux/nvmem-consumer.h>`。该框架只有少量数据结构，其中包括`struct nvmem_device`，其外观如下：

```
struct nvmem_device {
    const char  *name;
    struct module *owner;
    struct device dev;
    int stride;
    int word_size;
    int id;
    int users;
    size_t size;
    bool read_only;
    int flags;
    nvmem_reg_read_t reg_read;
    nvmem_reg_write_t reg_write; void *priv;
    [...]
};
```

这个结构实际上抽象了真实的 NVMEM 硬件。它是由框架在设备注册时创建和填充的。也就是说，它的字段实际上是使用`struct nvmem_config`中字段的完整副本设置的，该结构描述如下：

```
struct nvmem_config {
    struct device *dev;
    const char *name;
    int id;
    struct module *owner;
    const struct nvmem_cell_info *cells;
    int ncells;
    bool read_only;
    bool root_only;
    nvmem_reg_read_t reg_read;     nvmem_reg_write_t reg_write;
    int size;
    int word_size;
    int stride;
    void *priv;
    [...]
};
```

这个结构是 NVMEM 设备的运行时配置，提供有关它的信息或访问其数据单元的辅助函数。在设备注册时，大多数字段都用于填充新创建的`nvmem_device`结构。

结构中字段的含义如下（了解这些用于构建底层`struct nvmem_device`）：

+   `dev`是父设备。

+   `name`是此 NVMEM 设备的可选名称。它与填充的`id`一起用于构建完整的设备名称。最终的 NVMEM 设备名称将是`<name><id>`。最好在名称中添加`-`，以便完整名称可以具有此模式：`<name>-<id>`。这是 PCF85363 驱动程序中使用的方法。如果省略，将使用`nvmem<id>`作为默认名称。

+   `id`是此 NVMEM 设备的可选 ID。如果`name`为`NULL`，则会被忽略。如果设置为`-1`，内核将负责为设备提供唯一的 ID。

+   `owner`是拥有此 NVMEM 设备的模块。

+   `cells`是预定义的 NVMEM 单元格数组。这是可选的。

+   `ncells`是单元格中元素的数量。

+   `read_only`将此设备标记为只读。

+   `root_only`指示此设备是否仅对 root 可访问。

+   `reg_read`和`reg_write`是框架用于分别读取和写入数据的基础回调。它们的定义如下：

```
typedef int (*nvmem_reg_read_t)(void *priv,                                 unsigned int offset,
                                void *val, size_t bytes);
typedef int (*nvmem_reg_write_t)(void *priv,                                  unsigned int offset,
                                 void *val,                                  size_t bytes);
```

+   `size`表示设备的大小。

+   `word_size`是此设备的最小读/写访问粒度。`stride`是最小读/写访问跨距。其原理已在前几章中解释过。

+   `priv`是传递给读/写回调的上下文数据。例如，它可以是包装此 NVMEM 设备的更大结构。

以前，我们在提供者方面使用了`struct nvmem_cell_info`结构，而在消费者方面使用了`struct nvmem_cell`。在 NVMEM 核心代码中，内核使用`nvmem_cell_info_to_nvmem_cell()`从前一个结构切换到第二个结构。

这些结构如下引入：

```
struct nvmem_cell {
    const char *name;
    int offset;
    int bytes;
    int bit_offset;
    int nbits;
    struct nvmem_device *nvmem;
    struct list_head node;
};
```

另一个数据结构`struct nvmem_cell`如下所示：

```
struct nvmem_cell_info {
    const char *name;
    unsigned int offset;
    unsigned int bytes;
    unsigned int bit_offset;
    unsigned int nbits;
};
```

如您所见，前两个数据结构几乎具有相同的属性。让我们看看它们的含义，如下所示：

+   `name`是单元的名称。

+   `偏移量`是单元从整个硬件数据寄存器中的偏移量（开始位置）。

+   `bytes`是从`offset`开始的数据单元的大小（以字节为单位）。

+   单元可以具有位级粒度。对于这些单元，应设置`bit_offset`以指定单元内的位偏移，并且应根据感兴趣区域的大小（以位为单位）定义`nbits`。

+   `nvmem`是此单元所属的 NVMEM 设备。

+   `node`用于跟踪整个单元系统。此字段最终出现在`nvmem_cells`列表中，该列表保存系统上所有可用的单元，而不管它们属于哪个 NVMEM 设备。这个全局列表实际上由`drivers/nvmem/core.c`中静态定义的互斥体`nvmem_cells_mutex`保护。

为了澄清前面的解释，让我们以以下配置为例的单元：

```
static struct nvmem_cellinfo mycell = {
    .offset = 0xc,
    .bytes = 0x1,
    [...],
}
```

在前面的例子中，如果我们将`.nbits`和`.bit_offset`都等于`0`，这意味着我们对单元的整个数据区域感兴趣，在我们的情况下是 1 字节大小。但是如果我们只对位 2 到 4（实际上是 3 位）感兴趣呢？结构将如下所示：

```
staic struct nvmem_cellinfo mycell = {
    .offset = 0xc,
    .bytes = 0x1,
    .bit_offset = 2,
    .nbits = 2 [...]
}
```

重要说明

前面的例子仅用于教学目的。即使您可以在驱动程序代码中预定义单元，也建议您依赖设备树声明单元，正如我们稍后在章节中将看到的那样，*NVMEM 提供程序的设备树绑定*部分。

消费者驱动程序和提供者驱动程序都不应创建`struct nvmem_cell`的实例。NVMEM 核心在生产者提供单元信息数组时，或者在消费者请求单元时，内部处理这一点。

到目前为止，我们已经介绍了该框架提供的数据结构和 API。但是，NVMEM 设备可以从内核或用户空间访问。此外，在内核中，必须有一个暴露设备存储的驱动程序，以便其他驱动程序访问它。这是生产者/消费者设计，其中提供者驱动程序是生产者，而其他驱动程序是消费者。现在，让我们从该框架的提供者（又名生产者）部分开始。

# 编写 NVMEM 提供程序驱动程序

提供者是暴露设备内存以便其他驱动程序（消费者）可以访问的人。这些驱动程序的主要任务如下：

+   提供与设备数据表相关的适当的 NVMEM 配置，以及允许您访问内存的例程

+   向系统注册设备

+   提供设备树绑定文档

这就是提供者必须做的全部。大多数（其余）机制/逻辑由 NVMEM 框架的代码处理。

## NVMEM 设备（取消）注册

注册/注销 NVMEM 设备实际上是提供方驱动程序的一部分，它可以使用`nvmem_register()`/`nvmem_unregister()`函数或其托管版本`devm_nvmem_register()`/`devm_nvmem_unregister()`：

```
struct nvmem_device *nvmem_register(const                                    struct nvmem_config *config)
struct nvmem_device *devm_nvmem_register(struct device *dev,
                             const struct nvmem_config *config)
int nvmem_unregister(struct nvmem_device *nvmem)
int devm_nvmem_unregister(struct device *dev,
                          struct nvmem_device *nvmem)
```

注册后，将创建`/sys/bus/nvmem/devices/dev-name/nvmem`二进制条目。在这些接口中，`*config`参数是描述要创建的 NVMEM 设备的 NVMEM 配置。`*dev`参数仅适用于托管版本，并表示使用 NVMEM 设备的设备。在成功路径上，这些函数返回一个指向`nvmem_device`的指针，否则在出错时返回`ERR_PTR()`。

另一方面，注销函数接受在注册函数成功路径上创建的 NVMEM 设备的指针。在成功注销时返回`0`，否则返回负错误。

### RTC 设备中的 NVMEM 存储

在许多`include/linux/rtc.h`中，您会注意到以下与 NVMEM 相关的字段：

```
struct rtc_device {
    [...]
    struct nvmem_device *nvmem;
    /* Old ABI support */
    bool nvram_old_abi;
    struct bin_attribute *nvram;
    [...]
}
```

请注意前面结构摘录中的以下内容：

+   `nvmem` 抽象了底层硬件内存。

+   `nvram_old_abi` 是一个布尔值，告诉我们是否要使用旧的（现在已弃用）NVRAM ABI 来注册此 RTC 的 NVMEM，该 ABI 使用`/sys/class/rtc/rtcx/device/nvram`来公开内存。只有在您有现有应用程序（您不想破坏）使用这个旧的 ABI 接口时，才应将此字段设置为`true`。新驱动程序不应设置此字段。

+   `nvram` 实际上是底层内存的二进制属性，仅由 RTC 框架用于旧 ABI 支持；也就是说，如果`nvram_old_abi`为`true`。

RTC 相关的 NVMEM 框架 API 可以通过`RTC_NVMEM`内核配置选项启用。此 API 在`drivers/rtc/nvmem.c`中定义，并分别公开了`rtc_nvmem_register()`和`rtc_nvmem_unregister()`，用于 RTC-NVMEM 注册和注销。它们的描述如下：

```
int rtc_nvmem_register(struct rtc_device *rtc,
                        struct nvmem_config *nvmem_config)
void rtc_nvmem_unregister(struct rtc_device *rtc)
```

`rtc_nvmem_register()` 在成功时返回`0`。它接受一个有效的 RTC 设备作为其第一个参数。这对代码有影响。这意味着只有在实际的 RTC 设备成功注册后，RTC 的 NVMEM 才应该被注册。换句话说，只有在`rtc_register_device()`成功后才应该调用`rtc_nvmem_register()`。第二个参数应该是一个指向有效的`nvmem_config`对象的指针。此外，正如我们已经看到的，这个配置可以在堆栈中声明，因为它的所有字段都被完全复制以构建`nvmem_device`结构。相反的是`rtc_nvmem_unregister()`，它取消注册 NVMEM。

让我们通过 DS1307 RTC 驱动程序的`probe`函数的摘录来总结一下，`drivers/rtc/rtc-ds1307.c`：

```
static int ds1307_probe(struct i2c_client *client,
                        const struct i2c_device_id *id)
{
    struct ds1307 *ds1307;
    int err = -ENODEV;
    int tmp;
    const struct chip_desc *chip;
    [...]
    ds1307->rtc->ops = chip->rtc_ops ?: &ds13xx_rtc_ops;
    err = rtc_register_device(ds1307->rtc);
    if (err)
        return err;
    if (chip->nvram_size) {
        struct nvmem_config nvmem_cfg = {
            .name = "ds1307_nvram",
            .word_size = 1,
            .stride = 1,
            .size = chip->nvram_size,
            .reg_read = ds1307_nvram_read,
            .reg_write = ds1307_nvram_write,
            .priv = ds1307,
        };
        ds1307->rtc->nvram_old_abi = true;
        rtc_nvmem_register(ds1307->rtc, &nvmem_cfg);
    }
    [...]
}
```

前面的代码首先在注册 NVMEM 设备之前将 RTC 注册到内核，提供与 RTC 存储空间相对应的 NVMEM 配置。前面是与 RTC 相关的，而不是通用的。其他 NVMEM 设备必须让其驱动程序公开回调，NVMEM 框架将把任何来自用户空间或内核内部的读/写请求转发给这些回调。下一节将解释如何实现这一点。

## 实现 NVMEM 读/写回调

为了使内核和其他框架能够从 NVMEM 设备和其单元中读取/写入数据，每个 NVMEM 提供程序必须公开一对回调，允许进行这些读/写操作。这种机制允许硬件无关的消费者代码，因此来自消费者端的任何读/写请求都会被重定向到底层提供程序的读/写回调。以下是每个提供程序必须符合的读/写原型：

```
typedef int (*nvmem_reg_read_t)(void *priv,                                 unsigned int offset,
                                void *val, size_t bytes);
typedef int (*nvmem_reg_write_t)(void *priv,                                  unsigned int offset,
                                 void *val, size_t bytes);
```

这些与 NVMEM 设备所在的底层总线无关。`nvmem_reg_read_t` 用于从 NVMEM 设备读取数据。`priv` 是 NVMEM 配置中提供的用户上下文，`offset` 是读取应该开始的位置，`val` 是读取数据必须存储的输出缓冲区，`bytes` 是要读取的数据的大小（实际上是字节数）。该函数应在成功时返回成功读取的字节数，并在出错时返回负错误代码。

另一方面，`nvmem_reg_write_t` 用于写入目的。`priv` 的含义与读取相同，`offset` 是写入应该从哪里开始的地方，`val` 是包含要写入的数据的缓冲区，`bytes` 是 `val` 中数据的字节数，应该被写入。`bytes` 不一定是 `val` 的大小。此函数应在成功时返回成功写入的字节数，并在出错时返回负错误代码。

现在我们已经看到了如何实现提供者读/写回调，让我们看看如何通过设备树扩展提供者的功能。

## NVMEM 提供者的设备树绑定

NVMEM 数据提供者没有特定的绑定。它应该根据其父总线 DT 绑定进行描述。这意味着，例如，如果它是一个 I2C 设备，它应该（相对于 I2C 绑定）被描述为坐在代表其后面的 I2C 总线节点的子节点。但是，还有一个可选的 `read-only` 属性，使设备成为只读。此外，每个子节点都将被视为数据单元（NVMEM 设备中的内存区域）。

让我们考虑以下 MMIO NVMEM 设备以及其子节点以进行解释：

```
ocotp: ocotp@21bc000 {
    #address-cells = <1>;
    #size-cells = <1>;
    compatible = "fsl,imx6sx-ocotp", "syscon";
    reg = <0x021bc000 0x4000>;
    [...]
    tempmon_calib: calib@38 {
        reg = <0x38 4>;
    };
    tempmon_temp_grade: temp-grade@20 {
        reg = <0x20 4>;
    };
    foo: foo@6 {
        reg = <0x6 0x2> bits = <7 2>
    };
    [...]
};
```

根据子节点中定义的属性，NVMEM 框架构建适当的 `nvmem_cell` 结构，并将它们插入到系统范围的 `nvmem_cells` 列表中。以下是数据单元绑定的可能属性：

+   `reg`：此属性是强制性的。它是一个双单元属性，描述了 NVMEM 设备中数据区域的字节偏移量（属性的第一个单元）和字节大小（属性的第二个单元）。

+   `bits`：这是一个可选的双单元属性，指定位偏移量（可能的值为 `0`-`7`）和由 `reg` 属性指定的地址范围内的位数。

在提供者节点内定义数据单元后，可以使用 `nvmem-cells` 属性将其分配给消费者，该属性是指向 NVMEM 提供者的句柄列表。此外，还应该有一个 `nvmem-cell-names` 属性，其主要目的是为每个数据单元命名。因此，分配的名称可以用于使用消费者 API 查找适当的数据单元。以下是一个示例分配：

```
tempmon: tempmon {
    compatible = "fsl,imx6sx-tempmon", "fsl,imx6q-tempmon";
    interrupt-parent = <&gpc>;
    interrupts = <GIC_SPI 49 IRQ_TYPE_LEVEL_HIGH>;
    fsl,tempmon = <&anatop>;
    clocks = <&clks IMX6SX_CLK_PLL3_USB_OTG>;
    nvmem-cells = <&tempmon_calib>, <&tempmon_temp_grade>;
    nvmem-cell-names = "calib", "temp_grade";
};
```

完整的 NVMEM 设备树绑定可在 `Documentation/devicetree/bindings/nvmem/nvmem.txt` 中找到。

我们刚刚了解了实现驱动程序（所谓的生产者）的存储的情况。虽然这并不总是这样，但内核中可能有其他驱动程序需要访问生产者（也称为提供者）提供的存储。下一节将详细描述这些驱动程序。

# NVMEM 消费者驱动程序 API

NVMEM 消费者是访问生产者提供的存储的驱动程序。这些驱动程序可以通过包含 `<linux/nvmem-consumer.h>` 来调用 NVMEM 消费者 API，这将带来以下基于单元的 API：

```
struct nvmem_cell *nvmem_cell_get(struct device *dev,
                                  const char *name);
struct nvmem_cell *devm_nvmem_cell_get(struct device *dev,
                                       const char *name);
void nvmem_cell_put(struct nvmem_cell *cell);
void devm_nvmem_cell_put(struct device *dev,
                         struct nvmem_cell *cell);
void *nvmem_cell_read(struct nvmem_cell *cell, size_t *len);
int nvmem_cell_write(struct nvmem_cell *cell,                      void *buf, size_t len); 
int nvmem_cell_read_u32(struct device *dev,                         const char *cell_id,
                        u32 *val);
```

`devm_` 前缀的 API 是受资源管理的版本，应尽可能使用。

话虽如此，消费者接口完全取决于生产者公开（部分）单元的能力，以便其他人可以访问它们。如前所述，提供/公开单元的能力应通过设备树完成。`devm_nvmem_cell_get()` 用于根据通过 `nvmem-cell-names` 属性分配的名称获取给定单元。`nvmem_cell_read` API 总是读取整个单元大小（即 `nvmem_cell->bytes`），如果可能的话。它的第三个参数 `len` 是一个输出参数，保存实际读取的 `nvmem_config.word_size` 的数量（实际上，大部分时间它保存 `1`，这意味着一个字节）。

成功读取后，`len` 指向的内容将等于单元格中的字节数：`*len = nvmem_cell->bytes`。另一方面，`nvmem_cell_read_u32()` 以 `u32` 的形式读取单元格的值。

以下是在前一节中描述的`tempmon`节点分配的单元，并读取它们的内容的代码：

```
static int imx_init_from_nvmem_cells(struct                                      platform_device *pdev)
{
    int ret; u32 val;
    ret = nvmem_cell_read_u32(&pdev->dev, "calib", &val);
    if (ret)
        return ret;
    ret = imx_init_calib(pdev, val);
    if (ret)
        return ret;
    ret = nvmem_cell_read_u32(&pdev->dev, "temp_grade", &val);
    if (ret)
        return ret;
    imx_init_temp_grade(pdev, val);
    return 0;
}
```

在这里，我们已经介绍了这个框架的消费者和生产者方面。通常，驱动程序需要将它们的服务暴露给用户空间。NVMEM 框架（就像其他 Linux 内核框架一样）可以透明地处理将 NVMEM 服务暴露给用户空间。下一节将详细解释这一点。

## 用户空间中的 NVMEM

NVMEM 用户空间接口依赖于`sysfs`，就像大多数内核框架一样。系统中注册的每个 NVMEM 设备在`/sys/bus/nvmem/devices`中创建一个目录条目，以及在该目录中创建一个`nvmem`二进制文件（您可以使用`hexdump`或`echo`），代表设备的内存。完整路径遵循以下模式：`/sys/bus/nvmem/devices/<dev-name>X/nvmem`。在这个路径模式中，`<dev-name>`是生产驱动程序提供的`nvmem_config.name`名称。以下代码摘录显示了 NVMEM 核心如何构造`<dev-name>X`模式：

```
int rval;
rval	= ida_simple_get(&nvmem_ida, 0, 0, GFP_KERNEL);
nvmem->id = rval;
if (config->id == -1 && config->name) {
    dev_set_name(&nvmem->dev, "%s", config->name);
} else {
    dev_set_name(&nvmem->dev, "%s%d", config->name ? : "nvmem",
    config->name ? config->id : nvmem->id);
}
```

前面的代码表示，如果`nvmem_config->id == -1`，那么模式中的`X`将被省略，只使用`nvmem_config->name`来命名`sysfs`目录条目。如果`nvmem_config->id != -1`并且设置了`nvmem_config->name`，它将与驱动程序设置的`nvmem_config->id`字段一起使用（这是模式中的`X`）。但是，如果驱动程序没有设置`nvmem_config->name`，核心将使用`nvmem`字符串以及已生成的 ID（这是模式中的`X`）。

重要说明

无论定义了什么单元，NVMEM 框架都通过 NVMEM 二进制文件而不是单元来暴露完整的寄存器空间。从用户空间访问单元需要预先知道它们的偏移量和大小。

然后，NVMEM 内容可以在用户空间中使用`sysfs`接口进行读取，可以使用`hexdump`或简单的`cat`命令。例如，假设我们在系统上注册了一个 I2C EEPROM，它位于 I2C 编号 2 的地址 0x55 处，作为 NVMEM 设备注册，其`sysfs`路径将是`/sys/bus/nvmem/devices/2-00550/nvmem`。以下是如何写入/读取一些内容：

```
cat /sys/bus/nvmem/devices/2-00550/nvmem
echo "foo" > /sys/bus/nvmem/devices/2-00550/nvmem
cat /sys/bus/nvmem/devices/2-00550/nvmem
```

现在我们已经看到了 NVMEM 寄存器是如何暴露给用户空间的。虽然本节内容很短，但我们已经涵盖了足够的内容来从用户空间利用这个框架。

# 总结

在本章中，我们介绍了 Linux 内核中 NVMEM 框架的实现。我们从生产者和消费者方面介绍了其 API，并讨论了如何从用户空间使用它。我毫不怀疑这些设备在嵌入式世界中有它们的位置。

在下一章中，我们将通过看门狗设备来解决可靠性问题，讨论如何设置这些设备并编写它们的 Linux 内核驱动程序。
