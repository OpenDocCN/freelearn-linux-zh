# 第十五章：GPIO 控制器驱动程序 - gpio_chip

在上一章中，我们处理了 GPIO 线路。这些线路通过一个名为 GPIO 控制器的特殊设备向系统公开。本章将逐步解释如何为这类设备编写驱动程序，从而涵盖以下主题：

+   GPIO 控制器驱动程序架构和数据结构

+   GPIO 控制器的 Sysfs 接口

+   DT 中 GPIO 控制器的表示

# 驱动程序架构和数据结构

这类设备的驱动程序应该提供：

+   建立 GPIO 方向（输入和输出）的方法。

+   用于访问 GPIO 值的方法（获取和设置）。

+   将给定的 GPIO 映射到 IRQ 并返回相关的编号的方法。

+   标志，表示其方法是否可以休眠，这非常重要。

+   可选的 `debugfs dump` 方法（显示额外状态，如上拉配置）。

+   可选的基数号码，从哪里开始对 GPIO 进行编号。如果省略，将自动分配。

在内核中，GPIO 控制器表示为 `struct gpio_chip` 的实例，定义在 `linux/gpio/driver.h` 中：

```
struct gpio_chip { 
  const char *label; 
  struct device *dev; 
  struct module *owner; 

  int (*request)(struct gpio_chip *chip, unsigned offset); 
  void (*free)(struct gpio_chip *chip, unsigned offset); 
  int (*get_direction)(struct gpio_chip *chip, unsigned offset); 
  int (*direction_input)(struct gpio_chip *chip, unsigned offset); 
  int (*direction_output)(struct gpio_chip *chip, unsigned offset, 

            int value); 
  int (*get)(struct gpio_chip *chip,unsigned offset); 
  void (*set)(struct gpio_chip *chip, unsigned offset, int value); 
  void (*set_multiple)(struct gpio_chip *chip, unsigned long *mask, 
            unsigned long *bits); 
  int (*set_debounce)(struct gpio_chip *chip, unsigned offset, 
            unsigned debounce); 

  int (*to_irq)(struct gpio_chip *chip, unsigned offset); 

  int base; 
  u16 ngpio; 
  const char *const *names; 
  bool can_sleep; 
  bool irq_not_threaded; 
  bool exported; 

#ifdef CONFIG_GPIOLIB_IRQCHIP 
  /* 
   * With CONFIG_GPIOLIB_IRQCHIP we get an irqchip 
    * inside the gpiolib to handle IRQs for most practical cases. 
   */ 
  struct irq_chip *irqchip; 
  struct irq_domain *irqdomain; 
  unsigned int irq_base; 
  irq_flow_handler_t  irq_handler; 
  unsigned int irq_default_type; 
#endif 

#if defined(CONFIG_OF_GPIO) 
  /* 
   * If CONFIG_OF is enabled, then all GPIO controllers described in the 
   * device tree automatically may have an OF translation 
   */ 
  struct device_node *of_node; 
  int of_gpio_n_cells; 
  int (*of_xlate)(struct gpio_chip *gc, 
      const struct of_phandle_args *gpiospec, u32 *flags); 
} 
```

以下是结构中每个元素的含义：

+   `request` 是一个可选的钩子，用于特定于芯片的激活。如果提供，它将在分配 GPIO 之前执行，每当调用 `gpio_request()` 或 `gpiod_get()` 时。

+   `free` 是一个可选的钩子，用于特定于芯片的停用。如果提供，它将在每次调用 `gpiod_put()` 或 `gpio_free()` 时，在释放 GPIO 之前执行。

+   `get_direction` 每当需要知道 GPIO `offset` 的方向时执行。返回值应为 0 表示输出，1 表示输入（与 `GPIOF_DIR_XXX` 相同），或者负错误。

+   `direction_input` 配置信号 `offset` 为输入，或返回错误。

+   `get` 返回 GPIO `offset` 的值；对于输出信号，这将返回实际感应到的值，或者零。

+   `set` 将输出值分配给 GPIO `offset`。

+   `set_multiple` 当需要为 `mask` 定义的多个信号分配输出值时调用。如果未提供，内核将安装一个通用的钩子，将遍历 `mask` 位并在每个设置的位上执行 `chip->set(i)` 。

请参阅以下内容，显示了如何实现此功能：

```
 static void gpio_chip_set_multiple(struct gpio_chip *chip, 
      unsigned long *mask, unsigned long *bits) 
{ 
  if (chip->set_multiple) { 
    chip->set_multiple(chip, mask, bits); 
  } else { 
    unsigned int i; 

    /* set outputs if the corresponding mask bit is set */ 
    for_each_set_bit(i, mask, chip->ngpio) 
      chip->set(chip, i, test_bit(i, bits)); 
  } 
} 
```

+   `set_debounce` 如果控制器支持，这个钩子是一个可选的回调，用于设置指定 GPIO 的去抖时间。

+   `to_irq` 是一个可选的钩子，用于提供 GPIO 到 IRQ 的映射。每当需要执行 `gpio_to_irq()` 或 `gpiod_to_irq()` 函数时，就会调用这个函数。这个实现可能不会休眠。

+   `base` 标识了该芯片处理的第一个 GPIO 编号；或者，在注册期间为负时，内核将自动（动态）分配一个。

+   `ngpio` 是该控制器提供的 GPIO 数量，从 `base` 开始，到 `(base + ngpio - 1)` 结束。

+   `names`，如果设置，必须是一个字符串数组，用作该芯片中 GPIO 的替代名称。数组的大小必须为 `ngpio`，任何不需要别名的 GPIO 可以在数组中的条目中设置为 `NULL`。

+   `can_sleep` 是一个布尔标志，如果 `get()`/`set()` 方法可能会休眠，则设置。对于 GPIO 控制器（也称为扩展器）位于总线上，如 I2C 或 SPI，其访问可能会导致休眠。这意味着如果芯片支持 IRQ，这些 IRQ 需要被线程化，因为芯片访问可能会休眠，例如，读取 IRQ 状态寄存器时。对于映射到内存（SoC 的一部分）的 GPIO 控制器，可以将其设置为 false。

+   `irq_not_threaded` 是一个布尔标志，如果设置了 `can_sleep`，则必须设置该标志，但 IRQs 不需要被线程化。

每个芯片公开了一些信号，通过方法调用中的偏移值（在范围 0（`ngpio - 1`）内）进行标识。当这些信号通过 `gpio_get_value(gpio)` 等调用引用时，偏移量是通过从 GPIO 编号中减去基数来计算的。

在定义了每个回调和其他字段之后，应在配置的 `struct gpio_chip` 结构上调用 `gpiochip_add()`，以便向内核注册控制器。在注销时，使用 `gpiochip_remove()`。就是这样。您可以看到编写自己的 GPIO 控制器驱动程序有多么容易。在本书源代码库中，您将找到一个可用的 MCP23016 I2C I/O 扩展器的 GPIO 控制器驱动程序，其数据表可在 [`ww1.microchip.com/downloads/en/DeviceDoc/20090C.pdf`](http://ww1.microchip.com/downloads/en/DeviceDoc/20090C.pdf) 上找到。

要编写这样的驱动程序，您应该包括：

```
#include <linux/gpio.h>  
```

以下是我们为控制器编写的驱动程序的摘录，只是为了向您展示编写 GPIO 控制器驱动程序有多么容易：

```
#define GPIO_NUM 16 
struct mcp23016 { 
  struct i2c_client *client; 
  struct gpio_chip chip; 
}; 

static int mcp23016_probe(struct i2c_client *client, 
          const struct i2c_device_id *id) 
{ 
  struct mcp23016 *mcp; 

  if (!i2c_check_functionality(client->adapter, 
      I2C_FUNC_SMBUS_BYTE_DATA)) 
    return -EIO; 

  mcp = devm_kzalloc(&client->dev, sizeof(*mcp), GFP_KERNEL); 
  if (!mcp) 
    return -ENOMEM; 

  mcp->chip.label = client->name; 
  mcp->chip.base = -1; 
  mcp->chip.dev = &client->dev; 
  mcp->chip.owner = THIS_MODULE; 
  mcp->chip.ngpio = GPIO_NUM; /* 16 */ 
  mcp->chip.can_sleep = 1; /* may not be accessed from actomic context */ 
  mcp->chip.get = mcp23016_get_value; 
  mcp->chip.set = mcp23016_set_value; 
  mcp->chip.direction_output = mcp23016_direction_output; 
  mcp->chip.direction_input = mcp23016_direction_input; 
  mcp->client = client; 
  i2c_set_clientdata(client, mcp); 

  return gpiochip_add(&mcp->chip); 
} 
```

要从控制器驱动程序内部请求自有 GPIO，不应使用 `gpio_request()`。GPIO 驱动程序可以使用以下函数来请求和释放描述符，而不会永远固定在内核中：

```
struct gpio_desc *gpiochip_request_own_desc(struct gpio_desc *desc, const char *label) 
void gpiochip_free_own_desc(struct gpio_desc *desc) 
```

使用 `gpiochip_request_own_desc()` 请求的描述符必须使用 `gpiochip_free_own_desc()` 释放。

# 引脚控制器指南

根据您为其编写驱动程序的控制器，您可能需要实现一些引脚控制操作，以处理引脚复用、配置等：

+   对于只能执行简单 GPIO 的引脚控制器，简单的 `struct gpio_chip` 就足以处理它。无需设置 `struct pinctrl_desc` 结构，只需编写 GPIO 控制器驱动程序即可。

+   如果控制器可以在 GPIO 功能之上生成中断，则必须设置并向 IRQ 子系统注册 `struct irq_chip`。

+   对于具有引脚复用、高级引脚驱动强度、复杂偏置的控制器，您应该设置以下三个接口：

+   `struct gpio_chip`，在本章前面讨论过

+   `struct irq_chip`，在下一章（[第十六章](http://advanced)，*高级中断管理*）中讨论。

+   `struct pinctrl_desc`，本书未讨论，但在内核文档 *Documentation/pinctrl.txt* 中有很好的解释

# GPIO 控制器的 Sysfs 接口

成功调用 `gpiochip_add()` 后，将创建一个目录条目，路径类似于 `/sys/class/gpio/gpiochipX/`，其中 `X` 是 GPIO 控制器基地址（提供从 `#X` 开始的 GPIO 的控制器），具有以下属性：

+   `base`，其值与 `X` 相同，对应于 `gpio_chip.base`（如果静态分配），并且是由此芯片管理的第一个 GPIO。

+   `label`，用于诊断（不一定是唯一的）。

+   `ngpio`，告诉这个控制器提供多少个 GPIO（`N` 到 `N + ngpio - 1`）。这与 `gpio_chip.ngpios` 中定义的相同。

所有前述属性都是只读的。

# GPIO 控制器和 DT

在 DT 中声明的每个 GPIO 控制器都必须设置布尔属性 `gpio-controller`。一些控制器提供与 GPIO 映射的中断。在这种情况下，还应该设置属性 `interrupt-cells`，通常使用 `2`，但这取决于需要。第一个单元格是引脚编号，第二个表示中断标志。

`gpio-cells` 应设置为标识用于描述 GPIO 指定器的单元格数量。通常使用 `<2>`，第一个单元格用于标识 GPIO 编号，第二个用于标志。实际上，大多数非内存映射 GPIO 控制器不使用标志：

```
expander_1: mcp23016@27 { 
    compatible = "microchip,mcp23016"; 
    interrupt-controller; 
    gpio-controller; 
    #gpio-cells = <2>; 
    interrupt-parent = <&gpio6>; 
    interrupts = <31 IRQ_TYPE_LEVEL_LOW>; 
    reg = <0x27>; 
    #interrupt-cells=<2>; 
}; 
```

前述示例是我们的 GPIO 控制器设备节点，完整的设备驱动程序随本书的源代码一起提供。

# 摘要

本章远不止是为您可能遇到的 GPIO 控制器编写驱动程序的基础。它解释了描述这些设备的主要结构。下一章将涉及高级中断管理，我们将看到如何管理中断控制器，并在微芯片的 MCP23016 扩展器驱动程序中添加此功能。
