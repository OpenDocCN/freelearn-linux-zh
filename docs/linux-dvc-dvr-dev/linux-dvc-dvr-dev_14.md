# 第十四章：引脚控制和 GPIO 子系统

大多数嵌入式 Linux 驱动程序和内核工程师都使用 GPIO 或玩转引脚复用。在这里，引脚指的是组件的输出线。SoC 会复用引脚，这意味着一个引脚可能有多个功能，例如，在`arch/arm/boot/dts/imx6dl-pinfunc.h`中的`MX6QDL_PAD_SD3_DAT1`可以是 SD3 数据线 1、UART1 的 cts/rts、Flexcan2 的 Rx 或普通的 GPIO。

选择引脚应该工作的模式的机制称为引脚复用。负责此功能的系统称为引脚控制器。在本章的第二部分中，我们将讨论通用输入输出（GPIO），这是引脚可以操作的特殊功能（模式）。

在本章中，我们将：

+   了解引脚控制子系统，并看看如何在 DT 中声明它们的节点

+   探索传统的基于整数的 GPIO 接口，以及新的基于描述符的接口 API

+   处理映射到 IRQ 的 GPIO

+   处理专用于 GPIO 的 sysfs 接口

# 引脚控制子系统

引脚控制（pinctrl）子系统允许管理引脚复用。在 DT 中，需要以某种方式复用引脚的设备必须声明它们需要的引脚控制配置。

引脚控制子系统提供：

+   引脚复用，允许重用同一引脚用于不同的目的，比如一个引脚可以是 UART TX 引脚、GPIO 线或 HSI 数据线。复用可以影响引脚组或单个引脚。

+   引脚配置，应用引脚的电子属性，如上拉、下拉、驱动器强度、去抖时间等。

本书的目的仅限于使用引脚控制器驱动程序导出的函数，并不涉及如何编写引脚控制器驱动程序。

# 引脚控制和设备树

引脚控制只是一种收集引脚（不仅仅是 GPIO）并将它们传递给驱动程序的方法。引脚控制器驱动程序负责解析 DT 中的引脚描述并在芯片中应用它们的配置。驱动程序通常需要一组两个嵌套节点来描述引脚配置的组。第一个节点描述组的功能（组将用于什么目的），第二个节点保存引脚配置。

引脚组在设备树中的分配严重依赖于平台，因此也依赖于引脚控制器驱动程序。每个引脚控制状态都被赋予一个从 0 开始的连续整数 ID。可以使用一个名称属性，它将映射到 ID 上，以便相同的名称始终指向相同的 ID。

每个客户设备自己的绑定确定了必须在其 DT 节点中定义的状态集，以及是否定义必须提供的状态 ID 集，或者是否定义必须提供的状态名称集。在任何情况下，可以通过两个属性将引脚配置节点分配给设备：

+   `pinctrl-<ID>`：这允许为设备的某个状态提供所需的 pinctrl 配置列表。这是一个 phandle 列表，每个 phandle 指向一个引脚配置节点。这些引用的引脚配置节点必须是它们配置的引脚控制器的子节点。此列表中可能存在多个条目，以便可以配置多个引脚控制器，或者可以从单个引脚控制器的多个节点构建状态，每个节点都为整体配置的一部分做出贡献。

+   `pinctrl-name`：这允许为列表中的每个状态提供一个名称。列表条目 0 定义整数状态 ID 0 的名称，列表条目 1 定义状态 ID 1 的名称，依此类推。状态 ID 0 通常被赋予名称*default*。标准化状态列表可以在`include/linux/pinctrl/pinctrl-state.h`中找到。

+   以下是 DT 的摘录，显示了一些设备节点以及它们的引脚控制节点：

```
usdhc@0219c000 { /* uSDHC4 */ 
   non-removable; 
   vmmc-supply = <&reg_3p3v>; 
   status = "okay"; 
   pinctrl-names = "default"; 
   pinctrl-0 = <&pinctrl_usdhc4_1>; 
}; 

gpio-keys { 
    compatible = "gpio-keys"; 
    pinctrl-names = "default"; 
    pinctrl-0 = <&pinctrl_io_foo &pinctrl_io_bar>; 
}; 

iomuxc@020e0000 { 
    compatible = "fsl,imx6q-iomuxc"; 
    reg = <0x020e0000 0x4000>; 

    /* shared pinctrl settings */ 
    usdhc4 { /* first node describing the function */ 
        pinctrl_usdhc4_1: usdhc4grp-1 { /* second node */ 
            fsl,pins = < 
                MX6QDL_PAD_SD4_CMD__SD4_CMD    0x17059 
                MX6QDL_PAD_SD4_CLK__SD4_CLK    0x10059 
                MX6QDL_PAD_SD4_DAT0__SD4_DATA0 0x17059 
                MX6QDL_PAD_SD4_DAT1__SD4_DATA1 0x17059 
                MX6QDL_PAD_SD4_DAT2__SD4_DATA2 0x17059 
                MX6QDL_PAD_SD4_DAT3__SD4_DATA3 0x17059 
                MX6QDL_PAD_SD4_DAT4__SD4_DATA4 0x17059 
                MX6QDL_PAD_SD4_DAT5__SD4_DATA5 0x17059 
                MX6QDL_PAD_SD4_DAT6__SD4_DATA6 0x17059 
                MX6QDL_PAD_SD4_DAT7__SD4_DATA7 0x17059 
            >; 
        }; 
    }; 
    [...] 
    uart3 { 
        pinctrl_uart3_1: uart3grp-1 { 
            fsl,pins = < 
                MX6QDL_PAD_EIM_D24__UART3_TX_DATA 0x1b0b1 
                MX6QDL_PAD_EIM_D25__UART3_RX_DATA 0x1b0b1 
            >; 
        }; 
    }; 
    // GPIOs (Inputs) 
   gpios { 
        pinctrl_io_foo: pinctrl_io_foo { 
            fsl,pins = < 
                MX6QDL_PAD_DISP0_DAT15__GPIO5_IO09  0x1f059 
                MX6QDL_PAD_DISP0_DAT13__GPIO5_IO07  0x1f059 
            >; 
        }; 
        pinctrl_io_bar: pinctrl_io_bar { 
            fsl,pins = < 
                MX6QDL_PAD_DISP0_DAT11__GPIO5_IO05  0x1f059 
                MX6QDL_PAD_DISP0_DAT9__GPIO4_IO30   0x1f059 
                MX6QDL_PAD_DISP0_DAT7__GPIO4_IO28   0x1f059 
                MX6QDL_PAD_DISP0_DAT5__GPIO4_IO26   0x1f059 
            >; 
        }; 
    }; 
}; 
```

在上面的示例中，引脚配置以`<PIN_FUNCTION> <PIN_SETTING>`的形式给出。例如：

```
MX6QDL_PAD_DISP0_DAT15__GPIO5_IO09  0x80000000 
```

`MX6QDL_PAD_DISP0_DAT15__GPIO5_IO09`表示引脚功能，在这种情况下是 GPIO，`0x80000000`表示引脚设置。

对于这一行，

```
MX6QDL_PAD_EIM_D25__UART3_RX_DATA 0x1b0b1 
```

`MX6QDL_PAD_EIM_D25__UART3_RX_DATA`表示引脚功能，即 UART3 的 RX 线，`0x1b0b1`表示设置。

引脚功能是一个宏，其值仅对引脚控制器驱动程序有意义。这些通常在位于`arch/<arch>/boot/dts/`中的头文件中定义。例如，如果使用的是 UDOO quad，它具有 i.MX6 四核（ARM），则引脚功能头文件将是`arch/arm/boot/dts/imx6q-pinfunc.h`。以下是与 GPIO5 控制器的第五行对应的宏：

```
#define MX6QDL_PAD_DISP0_DAT11__GPIO5_IO05  0x19c 0x4b0 0x000 0x5 0x0 
```

`<PIN_SETTING>`可用于设置上拉电阻、下拉电阻、保持器、驱动强度等。如何指定它取决于引脚控制器绑定，其值的含义取决于 SoC 数据表，通常在 IOMUX 部分。在 i.MX6 IOMUXC 上，仅使用低于 17 位来实现此目的。

这些前置节点是从相应的驱动程序特定节点调用的。此外，这些引脚在相应的驱动程序初始化期间进行配置。在选择引脚组状态之前，必须首先使用`pinctrl_get()`函数获取引脚控制，调用`pinctrl_lookup_state()`来检查请求的状态是否存在，最后使用`pinctrl_select_state()`来应用状态。

以下是一个示例，显示如何获取 pincontrol 并应用其默认配置：

```
struct pinctrl *p; 
struct pinctrl_state *s; 
int ret; 

p = pinctrl_get(dev); 
if (IS_ERR(p)) 
    return p; 

s = pinctrl_lookup_state(p, name); 
if (IS_ERR(s)) { 
    devm_pinctrl_put(p); 
    return ERR_PTR(PTR_ERR(s)); 
} 

ret = pinctrl_select_state(p, s); 
if (ret < 0) { 
    devm_pinctrl_put(p); 
    return ERR_PTR(ret); 
} 
```

通常在驱动程序初始化期间执行这些步骤。此代码的适当位置可以在`probe()`函数内。

`pinctrl_select_state()`在内部调用`pinmux_enable_setting()`，后者又在引脚控制节点中的每个引脚上调用`pin_request()`。

可以使用`pinctrl_put()`函数释放引脚控制。可以使用 API 的资源管理版本。也就是说，可以使用`pinctrl_get_select()`，给定要选择的状态的名称，以配置引脚控制。该函数在`include/linux/pinctrl/consumer.h`中定义如下：

```
static struct pinctrl *pinctrl_get_select(struct device *dev, 
                             const char *name) 
```

其中`*name`是`pinctrl-name`属性中写的状态名称。如果状态的名称是`default`，可以直接调用`pinctr_get_select_default()`函数，这是`pinctl_get_select()`的包装器：

```
static struct pinctrl * pinctrl_get_select_default( 
                                struct device *dev) 
{ 
   return pinctrl_get_select(dev, PINCTRL_STATE_DEFAULT); 
} 
```

让我们看一个真实的例子，位于特定于板的 dts 文件（`am335x-evm.dts`）中：

```
dcan1: d_can@481d0000 { 
    status = "okay"; 
    pinctrl-names = "default"; 
    pinctrl-0 = <&d_can1_pins>; 
}; 
```

以及相应的驱动程序：

```
pinctrl = devm_pinctrl_get_select_default(&pdev->dev); 
if (IS_ERR(pinctrl)) 
    dev_warn(&pdev->dev,"pins are not configured from the driver\n"); 
```

当设备被探测时，引脚控制核心将自动为我们声明`default` pinctrl 状态。如果定义了`init`状态，引脚控制核心将在`probe()`函数之前自动将 pinctrl 设置为此状态，然后在`probe()`之后切换到`default`状态（除非驱动程序已经显式更改了状态）。

# GPIO 子系统

从硬件角度来看，GPIO 是一种功能，是引脚可以操作的模式。从软件角度来看，GPIO 只是一个数字线，可以作为输入或输出，并且只能有两个值：（`1`表示高，`0`表示低）。内核 GPIO 子系统提供了您可以想象的每个功能，以便从驱动程序内部设置和处理 GPIO 线：

+   在驱动程序中使用 GPIO 之前，应该向内核声明它。这是一种获取 GPIO 所有权的方法，可以防止其他驱动程序访问相同的 GPIO。获取 GPIO 所有权后，可以进行以下操作：

+   设置方向

+   切换其输出状态（将驱动线设置为高电平或低电平）如果用作输出

+   如果用作输入，则设置去抖动间隔并读取状态。对于映射到中断请求的 GPIO 线，可以定义触发中断的边缘/电平，并注册一个处理程序，每当中断发生时就会运行。

实际上，内核中处理 GPIO 有两种不同的方式，如下所示：

+   使用整数表示 GPIO 的传统和已弃用的接口

+   新的和推荐的基于描述符的接口，其中 GPIO 由不透明结构表示和描述，具有专用 API

# 基于整数的 GPIO 接口：传统

基于整数的接口是最为人熟知的。GPIO 由一个整数标识，该标识用于对 GPIO 执行的每个操作。以下是包含传统 GPIO 访问函数的标头：

```
#include <linux/gpio.h> 
```

内核中有众所周知的函数来处理 GPIO。

# 声明和配置 GPIO

可以使用`gpio_request（）`函数分配和拥有 GPIO：

```
static int  gpio_request(unsigned gpio, const char *label) 
```

`gpio`表示我们感兴趣的 GPIO 编号，`label`是内核在 sysfs 中用于 GPIO 的标签，如我们在`/sys/kernel/debug/gpio`中所见。必须检查返回的值，其中`0`表示成功，错误时为负错误代码。完成 GPIO 后，应使用`gpio_free（）`函数释放它：

```
void gpio_free(unsigned int gpio) 
```

如果有疑问，可以使用`gpio_is_valid（）`函数在分配之前检查系统上的 GPIO 编号是否有效：

```
static bool gpio_is_valid(int number) 
```

一旦我们拥有了 GPIO，就可以根据需要改变它的方向，无论是输入还是输出，都可以使用`gpio_direction_input（）`或`gpio_direction_output（）`函数：

```
static int  gpio_direction_input(unsigned gpio) 
static int  gpio_direction_output(unsigned gpio, int value) 
```

`gpio`是我们需要设置方向的 GPIO 编号。在配置 GPIO 为输出时有第二个参数：`value`，这是一旦输出方向生效后 GPIO 应处于的状态。同样，返回值为零或负错误号。这些函数在内部映射到我们使用的 GPIO 控制器驱动程序公开的较低级别回调函数之上。在下一章[第十五章](http://gpio)，*GPIO 控制器驱动程序-gpio_chip*中，处理 GPIO 控制器驱动程序，我们将看到 GPIO 控制器必须通过其`struct gpio_chip`结构公开一组通用的回调函数来使用其 GPIO。

一些 GPIO 控制器提供更改 GPIO 去抖动间隔的可能性（仅当 GPIO 线配置为输入时才有用）。这个功能是平台相关的。可以使用`int gpio_set_debounce（）`来实现这一点：

```
static  int  gpio_set_debounce(unsigned gpio, unsigned debounce) 
```

其中`debounce`是以毫秒为单位的去抖时间。

所有前述函数应在可能休眠的上下文中调用。从驱动程序的`probe`函数中声明和配置 GPIO 是一个良好的实践。

# 访问 GPIO-获取/设置值

在访问 GPIO 时应注意。在原子上下文中，特别是在中断处理程序中，必须确保 GPIO 控制器回调函数不会休眠。设计良好的控制器驱动程序应该能够通知其他驱动程序（实际上是客户端）其方法是否可能休眠。可以使用`gpio_cansleep（）`函数进行检查。

用于访问 GPIO 的函数都不返回错误代码。这就是为什么在 GPIO 分配和配置期间应注意并检查返回值的原因。

# 在原子上下文中

有一些 GPIO 控制器可以通过简单的内存读/写操作进行访问和管理。这些通常嵌入在 SoC 中，不需要休眠。对于这些控制器，`gpio_cansleep（）`将始终返回`false`。对于这样的 GPIO，可以在 IRQ 处理程序中使用众所周知的`gpio_get_value（）`或`gpio_set_value（）`获取/设置它们的值，具体取决于 GPIO 线被配置为输入还是输出：

```
static int  gpio_get_value(unsigned gpio) 
void gpio_set_value(unsigned int gpio, int value); 
```

当 GPIO 配置为输入（使用`gpio_direction_input（）`）时，应使用`gpio_get_value（）`，并返回 GPIO 的实际值（状态）。另一方面，`gpio_set_value（）`将影响 GPIO 的值，应该已经使用`gpio_direction_output（）`配置为输出。对于这两个函数，`value`可以被视为`布尔值`，其中零表示低，非零值表示高。

# 在可能休眠的非原子上下文中

另一方面，还有 GPIO 控制器连接在 SPI 和 I2C 等总线上。由于访问这些总线的函数可能导致休眠，因此`gpio_cansleep()`函数应始终返回`true`（由 GPIO 控制器负责返回 true）。在这种情况下，您不应该在 IRQ 处理中访问这些 GPIO，至少不是在顶半部分（硬 IRQ）。此外，您必须使用作为通用访问的访问器应该以`_cansleep`结尾。

```
static int gpio_get_value_cansleep(unsigned gpio); 
void gpio_set_value_cansleep(unsigned gpio, int value); 
```

它们的行为与没有`_cansleep()`名称后缀的访问器完全相同，唯一的区别是它们在访问 GPIO 时阻止内核打印警告。

# 映射到 IRQ 的 GPIO

输入 GPIO 通常可以用作 IRQ 信号。这些 IRQ 可以是边沿触发或电平触发的。配置取决于您的需求。GPIO 控制器负责提供 GPIO 和其 IRQ 之间的映射。可以使用`goio_to_irq()`将给定的 GPIO 号码映射到其 IRQ 号码：

```
int gpio_to_irq(unsigned gpio);
```

返回值是 IRQ 号码，可以调用`request_irq()`（或线程化版本`request_threaded_irq()`）来为此 IRQ 注册处理程序：

```
static irqreturn_t my_interrupt_handler(int irq, void *dev_id) 
{ 
    [...] 
    return IRQ_HANDLED; 
} 

[...] 
int gpio_int = of_get_gpio(np, 0); 
int irq_num = gpio_to_irq(gpio_int); 
int error = devm_request_threaded_irq(&client->dev, irq_num, 
                               NULL, my_interrupt_handler, 
                               IRQF_TRIGGER_RISING | IRQF_ONESHOT, 
                               input_dev->name, my_data_struct); 
if (error) { 
    dev_err(&client->dev, "irq %d requested failed, %d\n", 
        client->irq, error); 
    return error; 
} 
```

# 将所有内容放在一起

以下代码是将所有讨论的关于基于整数的接口的概念付诸实践的摘要。该驱动程序管理四个 GPIO：两个按钮（btn1 和 btn2）和两个 LED（绿色和红色）。Btn1 映射到 IRQ，每当其状态变为 LOW 时，btn2 的状态将应用于 LED。例如，如果 btn1 的状态变为 LOW，而 btn2 的状态为高，则`GREEN`和`RED` led 将被驱动到 HIGH：

```
#include <linux/init.h> 
#include <linux/module.h> 
#include <linux/kernel.h> 
#include <linux/gpio.h>        /* For Legacy integer based GPIO */ 
#include <linux/interrupt.h>   /* For IRQ */ 

static unsigned int GPIO_LED_RED = 49; 
static unsigned int GPIO_BTN1 = 115; 
static unsigned int GPIO_BTN2 = 116; 
static unsigned int GPIO_LED_GREEN = 120; 
static unsigned int irq; 

static irq_handler_t btn1_pushed_irq_handler(unsigned int irq, 
                             void *dev_id, struct pt_regs *regs) 
{ 
    int state; 

    /* read BTN2 value and change the led state */ 
    state = gpio_get_value(GPIO_BTN2); 
    gpio_set_value(GPIO_LED_RED, state); 
    gpio_set_value(GPIO_LED_GREEN, state); 

    pr_info("GPIO_BTN1 interrupt: Interrupt! GPIO_BTN2 state is %d)\n", state); 
    return IRQ_HANDLED; 
} 

static int __init helloworld_init(void) 
{ 
    int retval; 

    /* 
     * One could have checked whether the GPIO is valid on the controller or not, 
     * using gpio_is_valid() function. 
     * Ex: 
     *  if (!gpio_is_valid(GPIO_LED_RED)) { 
     *       pr_infor("Invalid Red LED\n"); 
     *       return -ENODEV; 
     *   } 
     */ 
    gpio_request(GPIO_LED_GREEN, "green-led"); 
    gpio_request(GPIO_LED_RED, "red-led"); 
    gpio_request(GPIO_BTN1, "button-1"); 
    gpio_request(GPIO_BTN2, "button-2"); 

    /* 
     * Configure Button GPIOs as input 
     * 
     * After this, one can call gpio_set_debounce() 
     * only if the controller has the feature 
     * 
     * For example, to debounce a button with a delay of 200ms 
     *  gpio_set_debounce(GPIO_BTN1, 200); 
     */ 
    gpio_direction_input(GPIO_BTN1); 
    gpio_direction_input(GPIO_BTN2); 

    /* 
     * Set LED GPIOs as output, with their initial values set to 0 
     */ 
    gpio_direction_output(GPIO_LED_RED, 0); 
    gpio_direction_output(GPIO_LED_GREEN, 0); 

    irq = gpio_to_irq(GPIO_BTN1); 
    retval = request_threaded_irq(irq, NULL,\ 
                            btn1_pushed_irq_handler, \ 
                            IRQF_TRIGGER_LOW | IRQF_ONESHOT, \ 
                            "device-name", NULL); 

    pr_info("Hello world!\n"); 
    return 0; 
} 

static void __exit hellowolrd_exit(void) 
{ 
    free_irq(irq, NULL); 
    gpio_free(GPIO_LED_RED); 
    gpio_free(GPIO_LED_GREEN); 
    gpio_free(GPIO_BTN1); 
    gpio_free(GPIO_BTN2); 

    pr_info("End of the world\n"); 
} 

module_init(hellowolrd_init); 
module_exit(hellowolrd_exit); 

MODULE_AUTHOR("John Madieu <john.madieu@gmail.com>"); 
MODULE_LICENSE("GPL"); 
```

# 基于描述符的 GPIO 接口：新的推荐方式

使用新的基于描述符的 GPIO 接口，GPIO 由一个连贯的`struct gpio_desc`结构来描述：

```
struct gpio_desc { 
   struct gpio_chip  *chip; 
   unsigned long flags; 
   const char *label; 
}; 
```

应该使用以下标头才能使用新接口：

```
#include <linux/gpio/consumer.h> 
```

使用基于描述符的接口，在分配和拥有 GPIO 之前，这些 GPIO 必须已经映射到某个地方。通过映射，我的意思是它们应该分配给您的设备，而使用传统的基于整数的接口，您只需在任何地方获取一个数字并将其请求为 GPIO。实际上，内核中有三种映射：

+   **平台数据映射**：映射在板文件中完成。

+   **设备树**：映射以 DT 样式完成，与前面的部分讨论的相同。这是我们将在本书中讨论的映射。

+   **高级配置和电源接口映射**（**ACPI**）：映射以 ACPI 样式完成。通常用于基于 x86 的系统。

# GPIO 描述符映射-设备树

GPIO 描述符映射在使用者设备的节点中定义。包含 GPIO 描述符映射的属性必须命名为`<name>-gpios`或`<name>-gpio`，其中`<name>`足够有意义，以描述这些 GPIO 将用于的功能。

应该始终在属性名称后缀加上`-gpio`或`-gpios`，因为每个基于描述符的接口函数都依赖于`gpio_suffixes[]`变量，在`drivers/gpio/gpiolib.h`中定义如下：

```
/* gpio suffixes used for ACPI and device tree lookup */ 
static const char * const gpio_suffixes[] = { "gpios", "gpio" }; 
```

让我们看看在 DT 中查找设备中 GPIO 描述符映射的函数：

```
static struct gpio_desc *of_find_gpio(struct device *dev, 
                                    const char *con_id, 
                                   unsigned int idx, 
                                   enum gpio_lookup_flags *flags) 
{ 
   char prop_name[32]; /* 32 is max size of property name */ 
   enum of_gpio_flags of_flags; 
   struct gpio_desc *desc; 
   unsigned int i; 

   for (i = 0; i < ARRAY_SIZE(gpio_suffixes); i++) { 
         if (con_id) 
               snprintf(prop_name, sizeof(prop_name), "%s-%s", 
                       con_id, 
                      gpio_suffixes[i]); 
         else 
               snprintf(prop_name, sizeof(prop_name), "%s", 
                      gpio_suffixes[i]); 

         desc = of_get_named_gpiod_flags(dev->of_node, 
                                          prop_name, idx, 
                                 &of_flags); 
         if (!IS_ERR(desc) || (PTR_ERR(desc) == -EPROBE_DEFER)) 
               break; 
   } 

   if (IS_ERR(desc)) 
         return desc; 

   if (of_flags & OF_GPIO_ACTIVE_LOW) 
         *flags |= GPIO_ACTIVE_LOW; 

   return desc; 
} 
```

现在，让我们考虑以下节点，这是`Documentation/gpio/board.txt`的摘录：

```
foo_device { 
   compatible = "acme,foo"; 
   [...] 
   led-gpios = <&gpio 15 GPIO_ACTIVE_HIGH>, /* red */ 
               <&gpio 16 GPIO_ACTIVE_HIGH>, /* green */ 
               <&gpio 17 GPIO_ACTIVE_HIGH>; /* blue */ 

   power-gpios = <&gpio 1 GPIO_ACTIVE_LOW>; 
   reset-gpios = <&gpio 1 GPIO_ACTIVE_LOW>; 
}; 
```

这就是映射应该看起来像的，具有有意义的名称。

# 分配和使用 GPIO

可以使用`gpiog_get()`或`gpiod_get_index()`来分配 GPIO 描述符：

```
struct gpio_desc *gpiod_get_index(struct device *dev, 
                                 const char *con_id, 
                                 unsigned int idx, 
                                 enum gpiod_flags flags) 
struct gpio_desc *gpiod_get(struct device *dev, 
                            const char *con_id, 
                            enum gpiod_flags flags) 
```

在错误的情况下，如果没有分配具有给定功能的 GPIO，则这些函数将返回`-ENOENT`，或者可以使用`IS_ERR()`宏的其他错误。第一个函数返回与给定索引处的 GPIO 对应的 GPIO 描述符结构，而第二个函数返回索引为 0 的 GPIO（对于单个 GPIO 映射很有用）。`dev`是 GPIO 描述符将属于的设备。这是你的设备。`con_id`是 GPIO 使用者内的功能。它对应于 DT 中属性名称的`<name>`前缀。`idx`是需要描述符的 GPIO 的索引（从 0 开始）。`flags`是一个可选参数，用于确定 GPIO 初始化标志，以配置方向和/或输出值。它是`include/linux/gpio/consumer.h`中定义的`enum gpiod_flags`的一个实例：

```
enum gpiod_flags { 
    GPIOD_ASIS = 0, 
    GPIOD_IN = GPIOD_FLAGS_BIT_DIR_SET, 
    GPIOD_OUT_LOW = GPIOD_FLAGS_BIT_DIR_SET | 
                    GPIOD_FLAGS_BIT_DIR_OUT, 
    GPIOD_OUT_HIGH = GPIOD_FLAGS_BIT_DIR_SET | 
                     GPIOD_FLAGS_BIT_DIR_OUT | 
                     GPIOD_FLAGS_BIT_DIR_VAL, 
}; 
```

现在让我们为在前面的 DT 中定义的映射分配 GPIO 描述符：

```
struct gpio_desc *red, *green, *blue, *power; 

red = gpiod_get_index(dev, "led", 0, GPIOD_OUT_HIGH); 
green = gpiod_get_index(dev, "led", 1, GPIOD_OUT_HIGH); 
blue = gpiod_get_index(dev, "led", 2, GPIOD_OUT_HIGH); 

power = gpiod_get(dev, "power", GPIOD_OUT_HIGH); 
```

LED GPIO 将是主动高电平，而电源 GPIO 将是主动低电平（即`gpiod_is_active_low(power)`将为 true）。分配的反向操作使用`gpiod_put()`函数完成：

```
gpiod_put(struct gpio_desc *desc); 
```

让我们看看如何释放`red`和`blue` GPIO LED：

```
gpiod_put(blue); 
gpiod_put(red); 
```

在我们继续之前，请记住，除了`gpiod_get()`/`gpiod_get_index()`和`gpio_put()`函数与`gpio_request()`和`gpio_free()`完全不同之外，可以通过将`gpio_`前缀更改为`gpiod_`来执行从基于整数的接口到基于描述符的接口的 API 转换。

也就是说，要更改方向，应该使用`gpiod_direction_input()`和`gpiod_direction_output()`函数：

```
int gpiod_direction_input(struct gpio_desc *desc); 
int gpiod_direction_output(struct gpio_desc *desc, int value); 
```

`value`是在将方向设置为输出后应用于 GPIO 的状态。如果 GPIO 控制器具有此功能，则可以使用其描述符设置给定 GPIO 的去抖动超时：

```
int gpiod_set_debounce(struct gpio_desc *desc, unsigned debounce); 
```

为了访问给定描述符的 GPIO，必须像基于整数的接口一样注意。换句话说，应该注意自己是处于原子（无法休眠）还是非原子上下文中，然后使用适当的函数：

```
int gpiod_cansleep(const struct gpio_desc *desc); 

/* Value get/set from sleeping context */ 
int gpiod_get_value_cansleep(const struct gpio_desc *desc); 
void gpiod_set_value_cansleep(struct gpio_desc *desc, int value); 

/* Value get/set from non-sleeping context */ 
int gpiod_get_value(const struct gpio_desc *desc); 
void gpiod_set_value(struct gpio_desc *desc, int value); 
```

对于映射到 IRQ 的 GPIO 描述符，可以使用`gpiod_to_irq()`来获取与给定 GPIO 描述符对应的 IRQ 编号，然后可以与`request_irq()`函数一起使用：

```
int gpiod_to_irq(const struct gpio_desc *desc); 
```

在代码中的任何时候，可以使用`desc_to_gpio()`或`gpio_to_desc()`函数从基于描述符的接口切换到传统的基于整数的接口，反之亦然：

```
/* Convert between the old gpio_ and new gpiod_ interfaces */ 
struct gpio_desc *gpio_to_desc(unsigned gpio); 
int desc_to_gpio(const struct gpio_desc *desc); 
```

# 把所有东西放在一起

以下是驱动程序总结了描述符接口中介绍的概念。原则是相同的，GPIO 也是一样的：

```
#include <linux/init.h> 
#include <linux/module.h> 
#include <linux/kernel.h> 
#include <linux/platform_device.h>      /* For platform devices */ 
#include <linux/gpio/consumer.h>        /* For GPIO Descriptor */ 
#include <linux/interrupt.h>            /* For IRQ */ 
#include <linux/of.h>                   /* For DT*/ 

/* 
 * Let us consider the below mapping in device tree: 
 * 
 *    foo_device { 
 *       compatible = "packt,gpio-descriptor-sample"; 
 *       led-gpios = <&gpio2 15 GPIO_ACTIVE_HIGH>, // red  
 *                   <&gpio2 16 GPIO_ACTIVE_HIGH>, // green  
 * 
 *       btn1-gpios = <&gpio2 1 GPIO_ACTIVE_LOW>; 
 *       btn2-gpios = <&gpio2 31 GPIO_ACTIVE_LOW>; 
 *   }; 
 */ 

static struct gpio_desc *red, *green, *btn1, *btn2; 
static unsigned int irq; 

static irq_handler_t btn1_pushed_irq_handler(unsigned int irq, 
                              void *dev_id, struct pt_regs *regs) 
{ 
    int state; 

    /* read the button value and change the led state */ 
    state = gpiod_get_value(btn2); 
    gpiod_set_value(red, state); 
    gpiod_set_value(green, state); 

    pr_info("btn1 interrupt: Interrupt! btn2 state is %d)\n", 
              state); 
    return IRQ_HANDLED; 
} 

static const struct of_device_id gpiod_dt_ids[] = { 
    { .compatible = "packt,gpio-descriptor-sample", }, 
    { /* sentinel */ } 
}; 

static int my_pdrv_probe (struct platform_device *pdev) 
{ 
    int retval; 
    struct device *dev = &pdev->dev; 

    /* 
     * We use gpiod_get/gpiod_get_index() along with the flags 
     * in order to configure the GPIO direction and an initial 
     * value in a single function call. 
     * 
     * One could have used: 
     *  red = gpiod_get_index(dev, "led", 0); 
     *  gpiod_direction_output(red, 0); 
     */ 
    red = gpiod_get_index(dev, "led", 0, GPIOD_OUT_LOW); 
    green = gpiod_get_index(dev, "led", 1, GPIOD_OUT_LOW); 

    /* 
     * Configure GPIO Buttons as input 
     * 
     * After this, one can call gpiod_set_debounce() 
     * only if the controller has the feature 
     * For example, to debounce  a button with a delay of 200ms 
     *  gpiod_set_debounce(btn1, 200); 
     */ 
    btn1 = gpiod_get(dev, "led", 0, GPIOD_IN); 
    btn2 = gpiod_get(dev, "led", 1, GPIOD_IN); 

    irq = gpiod_to_irq(btn1); 
    retval = request_threaded_irq(irq, NULL,\ 
                            btn1_pushed_irq_handler, \ 
                            IRQF_TRIGGER_LOW | IRQF_ONESHOT, \ 
                            "gpio-descriptor-sample", NULL); 
    pr_info("Hello! device probed!\n"); 
    return 0; 
} 

static void my_pdrv_remove(struct platform_device *pdev) 
{ 
    free_irq(irq, NULL); 
    gpiod_put(red); 
    gpiod_put(green); 
    gpiod_put(btn1); 
    gpiod_put(btn2); 
    pr_info("good bye reader!\n"); 
} 

static struct platform_driver mypdrv = { 
    .probe      = my_pdrv_probe, 
    .remove     = my_pdrv_remove, 
    .driver     = { 
        .name     = "gpio_descriptor_sample", 
        .of_match_table = of_match_ptr(gpiod_dt_ids),   
        .owner    = THIS_MODULE, 
    }, 
}; 
module_platform_driver(mypdrv); 
MODULE_AUTHOR("John Madieu <john.madieu@gmail.com>"); 
MODULE_LICENSE("GPL"); 
```

# GPIO 接口和设备树

无论需要为什么接口使用 GPIO，如何指定 GPIO 取决于提供它们的控制器，特别是关于其`#gpio-cells`属性，该属性确定用于 GPIO 指定器的单元格数量。 GPIO 指定器至少包含控制器 phandle 和一个或多个参数，其中参数的数量取决于提供 GPIO 的控制器的`#gpio-cells`属性。第一个单元通常是控制器上的 GPIO 偏移量编号，第二个表示 GPIO 标志。

GPIO 属性应命名为`[<name>-]gpios]`，其中`<name>`是设备的 GPIO 用途。请记住，对于基于描述符的接口，这个规则是必须的，并且变成了`<name>-gpios`（注意没有方括号，这意味着`<name>`前缀是必需的）：

```
gpio1: gpio1 { 
    gpio-controller; 
    #gpio-cells = <2>; 
}; 
gpio2: gpio2 { 
    gpio-controller; 
    #gpio-cells = <1>; 
}; 
[...] 

cs-gpios = <&gpio1 17 0>, 
           <&gpio2 2>; 
           <0>, /* holes are permitted, means no GPIO 2 */ 
           <&gpio1 17 0>; 

reset-gpios = <&gpio1 30 0>; 
cd-gpios = <&gpio2 10>; 
```

在前面的示例中，CS GPIO 包含控制器 1 和控制器 2 的 GPIO。如果不需要在列表中指定给定索引处的 GPIO，则可以使用`<0>`。复位 GPIO 有两个单元格（控制器 phandle 之后的两个参数），而 CD GPIO 只有一个单元格。您可以看到我给我的 GPIO 指定器起的名字是多么有意义。

# 传统的基于整数的接口和设备树

该接口依赖于以下标头：

```
#include <linux/of_gpio.h> 
```

当您需要使用传统的基于整数的接口支持 DT 时，您应该记住两个函数：`of_get_named_gpio()`和`of_get_named_gpio_count()`：

```
int of_get_named_gpio(struct device_node *np, 
                      const char *propname, int index) 
int of_get_named_gpio_count(struct device_node *np, 
                      const char* propname) 
```

给定设备节点，前者返回`*propname`属性在`index`位置的 GPIO 编号。第二个只返回属性中指定的 GPIO 数量：

```
int n_gpios = of_get_named_gpio_count(dev.of_node, 
                                    "cs-gpios"); /* return 4 */ 
int second_gpio = of_get_named_gpio(dev.of_node, "cs-gpio", 1); 
int rst_gpio = of_get_named_gpio("reset-gpio", 0); 
gpio_request(second_gpio, "my-gpio); 
```

仍然支持旧的说明符的驱动程序，其中 GPIO 属性命名为`[<name>-gpio`]或`gpios`。在这种情况下，应使用未命名的 API 版本，通过`of_get_gpio()`和`of_gpio_count()`：

```
int of_gpio_count(struct device_node *np) 
int of_get_gpio(struct device_node *np, int index) 
```

DT 节点如下所示：

```
my_node@addr { 
    compatible = "[...]"; 

    gpios = <&gpio1 2 0>, /* INT */ 
            <&gpio1 5 0>; /* RST */ 
    [...] 
}; 
```

驱动程序中的代码如下所示：

```
struct device_node *np = dev->of_node; 

if (!np) 
    return ERR_PTR(-ENOENT); 

int n_gpios = of_gpio_count(); /* Will return 2 */ 
int gpio_int = of_get_gpio(np, 0); 
if (!gpio_is_valid(gpio_int)) { 
    dev_err(dev, "failed to get interrupt gpio\n"); 
    return ERR_PTR(-EINVAL); 
} 

gpio_rst = of_get_gpio(np, 1); 
if (!gpio_is_valid(pdata->gpio_rst)) { 
    dev_err(dev, "failed to get reset gpio\n"); 
    return ERR_PTR(-EINVAL); 
} 
```

可以通过重写第一个驱动程序（基于整数接口的驱动程序）来总结这一点，以符合平台驱动程序结构，并使用 DT API：

```
#include <linux/init.h> 
#include <linux/module.h> 
#include <linux/kernel.h> 
#include <linux/platform_device.h>      /* For platform devices */ 
#include <linux/interrupt.h>            /* For IRQ */ 
#include <linux/gpio.h>        /* For Legacy integer based GPIO */ 
#include <linux/of_gpio.h>     /* For of_gpio* functions */ 
#include <linux/of.h>          /* For DT*/ 

/* 
 * Let us consider the following node 
 * 
 *    foo_device { 
 *       compatible = "packt,gpio-legacy-sample"; 
 *       led-gpios = <&gpio2 15 GPIO_ACTIVE_HIGH>, // red  
 *                   <&gpio2 16 GPIO_ACTIVE_HIGH>, // green  
 * 
 *       btn1-gpios = <&gpio2 1 GPIO_ACTIVE_LOW>; 
 *       btn2-gpios = <&gpio2 1 GPIO_ACTIVE_LOW>; 
 *   }; 
 */ 

static unsigned int gpio_red, gpio_green, gpio_btn1, gpio_btn2; 
static unsigned int irq; 

static irq_handler_t btn1_pushed_irq_handler(unsigned int irq, void *dev_id, 
                            struct pt_regs *regs) 
{ 
    /* The content of this function remains unchanged */ 
    [...] 
} 

static const struct of_device_id gpio_dt_ids[] = { 
    { .compatible = "packt,gpio-legacy-sample", }, 
    { /* sentinel */ } 
}; 

static int my_pdrv_probe (struct platform_device *pdev) 
{ 
    int retval; 
    struct device_node *np = &pdev->dev.of_node; 

    if (!np) 
        return ERR_PTR(-ENOENT); 

    gpio_red = of_get_named_gpio(np, "led", 0); 
    gpio_green = of_get_named_gpio(np, "led", 1); 
    gpio_btn1 = of_get_named_gpio(np, "btn1", 0); 
    gpio_btn2 = of_get_named_gpio(np, "btn2", 0); 

    gpio_request(gpio_green, "green-led"); 
    gpio_request(gpio_red, "red-led"); 
    gpio_request(gpio_btn1, "button-1"); 
    gpio_request(gpio_btn2, "button-2"); 

    /* Code to configure GPIO and request IRQ remains unchanged */ 
    [...] 
    return 0; 
} 

static void my_pdrv_remove(struct platform_device *pdev) 
{ 
    /* The content of this function remains unchanged */ 
    [...] 
} 

static struct platform_driver mypdrv = { 
    .probe  = my_pdrv_probe, 
    .remove = my_pdrv_remove, 
    .driver = { 
    .name   = "gpio_legacy_sample", 
            .of_match_table = of_match_ptr(gpio_dt_ids),   
            .owner    = THIS_MODULE, 
    }, 
}; 
module_platform_driver(mypdrv); 

MODULE_AUTHOR("John Madieu <john.madieu@gmail.com>"); 
MODULE_LICENSE("GPL"); 
```

# 设备树中的 GPIO 映射到 IRQ

可以轻松地在设备树中将 GPIO 映射到 IRQ。使用两个属性来指定中断：

+   `interrupt-parent`：这是 GPIO 的 GPIO 控制器

+   `interrupts`：这是中断说明符列表

这适用于传统和基于描述符的接口。 IRQ 说明符取决于提供此 GPIO 的 GPIO 控制器的`#interrupt-cell`属性。 `#interrupt-cell`确定在指定中断时使用的单元数。通常，第一个单元表示要映射到 IRQ 的 GPIO 编号，第二个单元表示应触发中断的电平/边缘。无论如何，中断说明符始终取决于其父级（具有设置中断控制器的父级），因此请参考内核源中的绑定文档：

```
gpio4: gpio4 { 
    gpio-controller; 
    #gpio-cells = <2>; 
    interrupt-controller; 
    #interrupt-cells = <2>; 
}; 

my_label: node@0 { 
    reg = <0>; 
    spi-max-frequency = <1000000>; 
    interrupt-parent = <&gpio4>; 
    interrupts = <29 IRQ_TYPE_LEVEL_LOW>; 
}; 
```

获取相应的 IRQ 有两种解决方案：

1.  **您的设备位于已知总线（I2C 或 SPI）上**：IRQ 映射将由您完成，并通过`struct i2c_client`或`struct spi_device`结构通过`probe()`函数（通过`i2c_client.irq`或`spi_device.irq`）提供。

1.  **您的设备位于伪平台总线上**：`probe()`函数将获得一个`struct platform_device`，您可以在其中调用`platform_get_irq()`：

```
int platform_get_irq(struct platform_device *dev, unsigned int num); 
```

随意查看第六章，*设备树的概念*。

# GPIO 和 sysfs

sysfs GPIO 接口允许人们通过集或文件管理和控制 GPIO。它位于`/sys/class/gpio`下。设备模型在这里被广泛使用，有三种类型的条目可用：

+   `/sys/class/gpio/：`这是一切的开始。此目录包含两个特殊文件，`export`和`unexport`：

+   `export`：这允许我们要求内核将给定 GPIO 的控制权导出到用户空间，方法是将其编号写入此文件。例如：`echo 21 > export`将为 GPIO＃21 创建一个 GPIO21 节点，如果内核代码没有请求。

+   `unexport`：这将撤消向用户空间的导出效果。例如：`echo 21 > unexport`将删除使用导出文件导出的任何 GPIO21 节点。

+   `/sys/class/gpio/gpioN/`：此目录对应于 GPIO 编号 N（其中 N 是系统全局的，而不是相对于芯片），可以使用`export`文件导出，也可以在内核内部导出。例如：`/sys/class/gpio/gpio42/`（对于 GPIO＃42）具有以下读/写属性：

+   `direction`文件用于获取/设置 GPIO 方向。允许的值是`in`或`out`字符串。通常可以写入此值。写入`out`会将值初始化为低。为了确保无故障操作，可以写入低值和高值以将 GPIO 配置为具有该初始值的输出。如果内核代码已导出此 GPIO，则不会存在此属性，从而禁用方向（请参阅`gpiod_export()`或`gpio_export()`函数）。

+   `value`属性允许我们根据方向（输入或输出）获取/设置 GPIO 线的状态。如果 GPIO 配置为输出，写入任何非零值将被视为高电平状态。如果配置为输出，写入`0`将使输出低电平，而`1`将使输出高电平。如果引脚可以配置为产生中断的线，并且已配置为生成中断，则可以在该文件上调用`poll(2)`系统调用，`poll(2)`将在中断触发时返回。使用`poll(2)`将需要设置事件`POLLPRI`和`POLLERR`。如果使用`select(2)`，则应在`exceptfds`中设置文件描述符。`poll(2)`返回后，要么`lseek(2)`到 sysfs 文件的开头并读取新值，要么关闭文件并重新打开以读取值。这与我们讨论的可轮询 sysfs 属性的原理相同。

+   `edge`确定了将让`poll()`或`select()`函数返回的信号边缘。允许的值为`none`，`rising`，`falling`或`both`。此文件可读/可写，仅在引脚可以配置为产生中断的输入引脚时存在。

+   `active_low`读取为 0（假）或 1（真）。写入任何非零值将反转*value*属性的读取和写入。现有和随后的`poll(2)`支持通过边缘属性进行配置，用于上升和下降边缘，将遵循此设置。内核中设置此值的相关函数是`gpio_sysf_set_active_low()`。

# 从内核代码中导出 GPIO

除了使用`/sys/class/gpio/export`文件将 GPIO 导出到用户空间外，还可以使用内核代码中的`gpio_export`（用于传统接口）或`gpioD_export`（新接口）等函数来显式管理已经使用`gpio_request()`或`gpiod_get()`请求的 GPIO 的导出：

```
int gpio_export(unsigned gpio, bool direction_may_change); 

int gpiod_export(struct gpio_desc *desc, bool direction_may_change); 
```

`direction_may_change`参数决定是否可以从输入更改信号方向为输出，反之亦然。内核的反向操作是`gpio_unexport()`或`gpiod_unexport()`：

```
void gpio_unexport(unsigned gpio); /* Integer-based interface */ 
void gpiod_unexport(struct gpio_desc *desc) /* Descriptor-based */ 
```

一旦导出，可以使用`gpio_export_link()`（或`gpiod_export_link()`用于基于描述符的接口）来创建符号链接，从 sysfs 的其他位置指向 GPIO sysfs 节点。驱动程序可以使用此功能在 sysfs 中的自己设备下提供接口，并提供描述性名称：

```
int gpio_export_link(struct device *dev, const char *name, 
                      unsigned gpio) 
int gpiod_export_link(struct device *dev, const char *name, 
                      struct gpio_desc *desc) 
```

可以在基于描述符的接口的`probe()`函数中使用如下：

```
static struct gpio_desc *red, *green, *btn1, *btn2; 

static int my_pdrv_probe (struct platform_device *pdev) 
{ 
    [...] 
    red = gpiod_get_index(dev, "led", 0, GPIOD_OUT_LOW); 
    green = gpiod_get_index(dev, "led", 1, GPIOD_OUT_LOW); 

    gpiod_export(&pdev->dev, "Green_LED", green); 
    gpiod_export(&pdev->dev, "Red_LED", red); 

       [...] 
    return 0; 
} 
```

对于基于整数的接口，代码如下：

```
static int my_pdrv_probe (struct platform_device *pdev) 
{ 
    [...] 

    gpio_red = of_get_named_gpio(np, "led", 0); 
    gpio_green = of_get_named_gpio(np, "led", 1); 
    [...] 

    int gpio_export_link(&pdev->dev, "Green_LED", gpio_green) 
    int gpio_export_link(&pdev->dev, "Red_LED", gpio_red) 
    return 0; 
} 
```

# 摘要

在本章中，我们展示了在内核中处理 GPIO 是一项简单的任务。讨论了传统接口和新接口，为您提供了选择适合您需求的接口的可能性，以编写增强的 GPIO 驱动程序。您将能够处理映射到 GPIO 的中断请求。下一章将处理提供和公开 GPIO 线的芯片，称为 GPIO 控制器。
