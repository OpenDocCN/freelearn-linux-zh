# 第六章：设备树的概念

**设备树**（**DT**）是一个易于阅读的硬件描述文件，具有类似 JSON 的格式样式，是一个简单的树结构，其中设备由具有其属性的节点表示。属性可以是空的（只是键，用于描述布尔值），也可以是键值对，其中值可以包含任意字节流。本章是对 DT 的简单介绍。每个内核子系统或框架都有自己的 DT 绑定。当我们处理相关主题时，我们将讨论这些特定的绑定。DT 起源于 OF，这是一个由计算机公司认可的标准，其主要目的是为计算机固件系统定义接口。也就是说，可以在[`www.devicetree.org/`](http://www.devicetree.org/)找到更多关于 DT 规范的信息。因此，本章将涵盖 DT 的基础知识，例如：

+   命名约定，以及别名和标签

+   描述数据类型及其 API

+   管理寻址方案和访问设备资源

+   实现 OF 匹配样式并提供特定于应用程序的数据

# 设备树机制

通过将选项`CONFIG_OF`设置为`Y`，可以在内核中启用 DT。为了从驱动程序中调用 DT API，必须添加以下标头：

```
#include <linux/of.h> 
#include <linux/of_device.h> 
```

DT 支持几种数据类型。让我们通过一个示例节点描述来看看它们：

```
/* This is a comment */ 
// This is another comment 
node_label: nodename@reg{ 
   string-property = "a string"; 
   string-list = "red fish", "blue fish"; 
   one-int-property = <197>; /* One cell in this property */ 
   int-list-property = <0xbeef 123 0xabcd4>; /*each number (cell) is a                         

                                               *32 bit integer(uint32).

                                               *There are 3 cells in  

                                               */this property 

    mixed-list-property = "a string", <0xadbcd45>, <35>, [0x01 0x23 0x45] 
    byte-array-property = [0x01 0x23 0x45 0x67]; 
    boolean-property; 
}; 
```

以下是设备树中使用的一些数据类型的定义：

+   文本字符串用双引号表示。可以使用逗号创建字符串列表。

+   单元是由尖括号分隔的 32 位无符号整数。

+   布尔数据只是一个空属性。真或假的值取决于属性是否存在。

# 命名约定

每个节点必须具有形式为`<name>[@<address>]`的名称，其中`<name>`是一个长度最多为 31 个字符的字符串，`[@<address>]`是可选的，取决于节点是否表示可寻址设备。`<address>`应该是用于访问设备的主要地址。设备命名的示例如下：

```
expander@20 { 
    compatible = "microchip,mcp23017"; 
    reg = <20>; 
    [...]        
}; 
```

或

```
i2c@021a0000 { 
    compatible = "fsl,imx6q-i2c", "fsl,imx21-i2c"; 
    reg = <0x021a0000 0x4000>; 
    [...] 
}; 
```

另一方面，“标签”是可选的。只有当节点打算从另一个节点的属性引用时，标记节点才有用。可以将标签视为指向节点的指针，如下一节所述。

# 别名，标签和 phandle

了解这三个元素如何工作非常重要。它们在 DT 中经常被使用。让我们看看以下的 DT 来解释它们是如何工作的：

```
aliases { 
    ethernet0 = &fec; 
    gpio0 = &gpio1; 
    gpio1 = &gpio2; 
    mmc0 = &usdhc1; 
    [...] 
}; 
gpio1: gpio@0209c000 { 
    compatible = "fsl,imx6q-gpio", "fsl,imx35-gpio"; 
    [...] 
}; 
node_label: nodename@reg { 
    [...]; 
    gpios = <&gpio1 7 GPIO_ACTIVE_HIGH>; 
}; 
```

标签只是一种标记节点的方式，以便让节点通过唯一名称进行标识。在现实世界中，DT 编译器将该名称转换为唯一的 32 位值。在前面的示例中，`gpio1`和`node_label`都是标签。然后可以使用标签来引用节点，因为标签对节点是唯一的。

**指针句柄**（**phandle**）是与节点关联的 32 位值，用于唯一标识该节点，以便可以从另一个节点的属性中引用该节点。标签用于指向节点。通过使用`<&mylabel>`，您指向其标签为`mylabel`的节点。

使用`&`与 C 编程语言中的用法相同；用于获取元素的地址。

在前面的示例中，`&gpio1`被转换为 phandle，以便它引用`gpio1`节点。对于以下示例也是如此：

```
thename@address { 
    property = <&mylabel>; 
}; 

mylabel: thename@adresss { 
    [...] 
} 
```

为了不必遍历整个树来查找节点，引入了别名的概念。在 DT 中，`aliases`节点可以看作是一个快速查找表，另一个节点的索引。可以使用函数`find_node_by_alias()`来查找给定别名的节点。别名不直接在 DT 源中使用，而是由 Linux 内核进行解引用。

# DT 编译器

DT 有两种形式：文本形式，表示源也称为`DTS`，以及二进制 blob 形式，表示已编译的 DT，也称为`DTB`。源文件的扩展名为`.dts`。实际上，还有`.dtsi`文本文件，表示 SoC 级别定义，而`.dts`文件表示板级别定义。可以将`.dtsi`视为头文件，应包含在`.dts`中，这些是源文件，而不是反向的，有点像在源文件（`.c`）中包含头文件（`.h`）。另一方面，二进制文件使用`.dtb`扩展名。

实际上还有第三种形式，即在`/proc/device-tree`中的 DT 的运行时表示。

正如其名称所示，用于编译设备树的工具称为**设备树编译器**（**dtc**）。从根内核源中，可以编译特定体系结构的独立特定 DT 或所有 DT。

让我们为 arm SoC 编译所有 DT（`.dts`）文件：

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make dtbs 

```

对于独立的 DT：

```
ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- make imx6dl-sabrelite.dtb 

```

在前面的示例中，源文件的名称是`imx6dl-sabrelite.dts`。

给定一个已编译的设备树（.dtb）文件，您可以执行反向操作并提取源（.dts）文件：

```
dtc -I dtb -O dtsarch/arm/boot/dts imx6dl-sabrelite.dtb >path/to/my_devicetree.dts 
```

出于调试目的，将 DT 暴露给用户空间可能很有用。`CONFIG_PROC_DEVICETREE`配置变量将为您执行此操作。然后，您可以在`/proc/device-tree`中探索和浏览 DT。

# 表示和寻址设备

每个设备在 DT 中至少有一个节点。某些属性对许多设备类型都是共同的，特别是对于内核已知的总线上的设备（SPI、I2C、平台、MDIO 等）。这些属性是`reg`、`#address-cells`和`#size-cells`。这些属性的目的是在它们所在的总线上寻址设备。也就是说，主要的寻址属性是`reg`，这是一个通用属性，其含义取决于设备所在的总线。前缀`size-cell`和`address-cell`的`#`（sharp）可以翻译为`length`。

每个可寻址设备都有一个`reg`属性，其形式为`reg = <address0size0 [address1size1] [address2size2] ...>`的元组列表，其中每个元组表示设备使用的地址范围。`#size-cells`指示用于表示大小的 32 位单元的数量，如果大小不相关，则可能为 0。另一方面，`#address-cells`指示用于表示地址的 32 位单元的数量。换句话说，每个元组的地址元素根据`#address-cell`进行解释；大小元素也是如此，根据`#size-cell`进行解释。

实际上，可寻址设备继承自其父节点的`#size-cell`和`#address-cell`，即代表总线控制器的节点。给定设备中`#size-cell`和`#address-cell`的存在不会影响设备本身，而是其子级。换句话说，在解释给定节点的`reg`属性之前，必须了解父节点的`#address-cells`和`#size-cells`值。父节点可以自由定义适合设备子节点（子级）的任何寻址方案。

# SPI 和 I2C 寻址

SPI 和 I2C 设备都属于非内存映射设备，因为它们的地址对 CPU 不可访问。相反，父设备的驱动程序（即总线控制器驱动程序）将代表 CPU 执行间接访问。每个 I2C/SPI 设备始终表示为所在的 I2C/SPI 总线节点的子节点。对于非内存映射设备，“＃size-cells”属性为 0，寻址元组中的大小元素为空。这意味着这种类型的设备的`reg`属性始终在单元上：

```
&i2c3 { 
    [...] 
    status = "okay"; 

    temperature-sensor@49 { 
        compatible = "national,lm73"; 
        reg = <0x49>; 
    }; 

    pcf8523: rtc@68 { 
        compatible = "nxp,pcf8523"; 
        reg = <0x68>; 
    }; 
}; 

&ecspi1 { 
fsl,spi-num-chipselects = <3>; 
cs-gpios = <&gpio5 17 0>, <&gpio5 17 0>, <&gpio5 17 0>; 
status = "okay"; 
[...] 

ad7606r8_0: ad7606r8@1 { 
    compatible = "ad7606-8"; 
    reg = <1>; 
    spi-max-frequency = <1000000>; 
    interrupt-parent = <&gpio4>; 
    interrupts = <30 0x0>; 
    convst-gpio = <&gpio6 18 0>; 
}; 
}; 
```

如果有人查看`arch/arm/boot/dts/imx6qdl.dtsi`中的 SoC 级文件，就会注意到`#size-cells`和`#address-cells`分别设置为前者为`0`，后者为`1`，在`i2c`和`spi`节点中，它们分别是 I2C 和 SPI 设备在前面部分列举的父节点。这有助于我们理解它们的`reg`属性，地址值只有一个单元，大小值没有。

I2C 设备的`reg`属性用于指定总线上设备的地址。对于 SPI 设备，`reg`表示分配给设备的芯片选择线的索引，该索引位于控制器节点具有的芯片选择列表中。例如，对于 ad7606r8 ADC，芯片选择索引是`1`，对应于`cs-gpios`中的`<&gpio5 17 0>`，这是控制器节点的芯片选择列表。

你可能会问为什么我使用了 I2C/SPI 节点的 phandle：答案是因为 I2C/SPI 设备应该在板级文件（`.dts`）中声明，而 I2C/SPI 总线控制器应该在 SoC 级文件（`.dtsi`）中声明。

# 平台设备寻址

本节涉及的是内存对 CPU 可访问的简单内存映射设备。在这里，`reg`属性仍然定义了设备的地址，这是可以访问设备的内存区域列表。每个区域用单元组表示，其中第一个单元是内存区域的基地址，第二个单元是区域的大小。它的形式是`reg = <base0 length0 [base1 length1] [address2 length2] ...>`。每个元组表示设备使用的地址范围。

在现实世界中，不应该在不知道另外两个属性`#size-cells`和`#address-cells`的值的情况下解释`reg`属性。`#size-cells`告诉我们每个子`reg`元组中长度字段有多大。`#address-cell`也是一样，它告诉我们必须使用多少个单元来指定一个地址。

这种设备应该在一个具有特殊值`compatible = "simple-bus"`的节点中声明，表示一个没有特定处理或驱动程序的简单内存映射总线：

```
soc { 
    #address-cells = <1>; 
    #size-cells = <1>; 
    compatible = "simple-bus"; 
    aips-bus@02000000 { /* AIPS1 */ 
        compatible = "fsl,aips-bus", "simple-bus"; 
        #address-cells = <1>; 
        #size-cells = <1>; 
        reg = <0x02000000 0x100000>; 
        [...]; 

        spba-bus@02000000 { 
            compatible = "fsl,spba-bus", "simple-bus"; 
            #address-cells = <1>; 
            #size-cells = <1>; 
            reg = <0x02000000 0x40000>; 
            [...] 

            ecspi1: ecspi@02008000 { 
                #address-cells = <1>; 
                #size-cells = <0>; 
                compatible = "fsl,imx6q-ecspi", "fsl,imx51-ecspi"; 
                reg = <0x02008000 0x4000>; 
                [...] 
            }; 

            i2c1: i2c@021a0000 { 
                #address-cells = <1>; 
                #size-cells = <0>; 
                compatible = "fsl,imx6q-i2c", "fsl,imx21-i2c"; 
                reg = <0x021a0000 0x4000>; 
                [...] 
            }; 
        }; 
    }; 
```

在前面的例子中，具有`compatible`属性中`simple-bus`的子节点将被注册为平台设备。人们还可以看到 I2C 和 SPI 总线控制器如何通过设置`#size-cells = <0>;`来改变其子节点的寻址方案，因为这对它们来说并不重要。查找任何绑定信息的一个著名地方是内核设备树的文档：*Documentation/devicetree/bindings/*。

# 处理资源

驱动程序的主要目的是处理和管理设备，并且大部分时间将其功能暴露给用户空间。这里的目标是收集设备的配置参数，特别是资源（内存区域、中断线、DMA 通道、时钟等）。

以下是我们在本节中将使用的设备节点。它是在`arch/arm/boot/dts/imx6qdl.dtsi`中定义的 i.MX6 UART 设备节点：

```
uart1: serial@02020000 { 
        compatible = "fsl,imx6q-uart", "fsl,imx21-uart"; 
reg = <0x02020000 0x4000>; 
        interrupts = <0 26 IRQ_TYPE_LEVEL_HIGH>; 
        clocks = <&clks IMX6QDL_CLK_UART_IPG>, 
<&clks IMX6QDL_CLK_UART_SERIAL>; 
        clock-names = "ipg", "per"; 
dmas = <&sdma 25 4 0>, <&sdma 26 4 0>; 
dma-names = "rx", "tx"; 
        status = "disabled"; 
    }; 
```

# 命名资源的概念

当驱动程序期望某种类型的资源列表时，没有保证列表按照驱动程序期望的方式排序，因为编写板级设备树的人通常不是编写驱动程序的人。例如，驱动程序可能期望其设备节点具有 2 个 IRQ 线，一个用于索引 0 的 Tx 事件，另一个用于索引 1 的 Rx。如果顺序没有得到尊重会发生什么？驱动程序将产生不需要的行为。为了避免这种不匹配，引入了命名资源（`clock`，`irq`，`dma`，`reg`）的概念。它包括定义我们的资源列表，并对其进行命名，以便无论它们的索引如何，给定的名称始终与资源匹配。

用于命名资源的相应属性如下：

+   `reg-names`：这是在`reg`属性中的内存区域列表

+   `clock-names`：这是在`clocks`属性中命名时钟

+   `interrupt-names`：这为`interrupts`属性中的每个中断提供了一个名称

+   `dma-names`：这是`dma`属性

现在让我们创建一个虚假的设备节点条目来解释一下：

```
fake_device { 
    compatible = "packt,fake-device"; 
    reg = <0x4a064000 0x800>, <0x4a064800 0x200>, <0x4a064c00 0x200>; 
    reg-names = "config", "ohci", "ehci"; 
    interrupts = <0 66 IRQ_TYPE_LEVEL_HIGH>, <0 67 IRQ_TYPE_LEVEL_HIGH>; 
    interrupt-names = "ohci", "ehci"; 
    clocks = <&clks IMX6QDL_CLK_UART_IPG>, <&clks IMX6QDL_CLK_UART_SERIAL>; 
    clock-names = "ipg", "per"; 
    dmas = <&sdma 25 4 0>, <&sdma 26 4 0>; 
    dma-names = "rx", "tx"; 
}; 
```

驱动程序中提取每个命名资源的代码如下：

```
struct resource *res1, *res2; 
res1 = platform_get_resource_byname(pdev, IORESOURCE_MEM, "ohci"); 
res2 = platform_get_resource_byname(pdev, IORESOURCE_MEM, "config"); 

struct dma_chan  *dma_chan_rx, *dma_chan_tx; 
dma_chan_rx = dma_request_slave_channel(&pdev->dev, "rx"); 
dma_chan_tx = dma_request_slave_channel(&pdev->dev, "tx"); 

inttxirq, rxirq; 
txirq = platform_get_irq_byname(pdev, "ohci"); 
rxirq = platform_get_irq_byname(pdev, "ehci"); 

structclk *clck_per, *clk_ipg; 
clk_ipg = devm_clk_get(&pdev->dev, "ipg"); 
clk_ipg = devm_clk_get(&pdev->dev, "pre"); 
```

这样，您可以确保将正确的名称映射到正确的资源，而无需再使用索引。

# 访问寄存器

在这里，驱动程序将接管内存区域并将其映射到虚拟地址空间中。我们将在[第十一章](http://post)中更多地讨论这个问题，*内核内存管理*。

```
struct resource *res; 
void __iomem *base; 

res = platform_get_resource(pdev, IORESOURCE_MEM, 0); 
/* 
 * Here one can request and map the memory region 
 * using request_mem_region(res->start, resource_size(res), pdev->name) 
 * and ioremap(iores->start, resource_size(iores) 
 * 
 * These function are discussed in chapter 11, Kernel Memory Management. 
 */ 
base = devm_ioremap_resource(&pdev->dev, res); 
if (IS_ERR(base)) 
    return PTR_ERR(base); 
```

`platform_get_resource()`将根据 DT 节点中第一个（索引 0）`reg`分配中存在的内存区域设置`struct res`的开始和结束字段。请记住，`platform_get_resource()`的最后一个参数表示资源索引。在前面的示例中，`0`索引了该资源类型的第一个值，以防设备在 DT 节点中分配了多个内存区域。在我们的示例中，它是`reg = <0x02020000 0x4000>`，意味着分配的区域从物理地址`0x02020000`开始，大小为`0x4000`字节。然后，`platform_get_resource()`将设置`res.start = 0x02020000`和`res.end = 0x02023fff`。

# 处理中断

中断接口实际上分为两部分；消费者端和控制器端。在 DT 中用四个属性来描述中断连接：

控制器是向消费者公开 IRQ 线的设备。在控制器端，有以下属性：

+   `interrupt-controller`：一个空（布尔）属性，应该定义为标记设备为中断控制器

+   `#interrupt-cells`：这是中断控制器的属性。它说明用于为该中断控制器指定中断的单元格数

消费者是生成 IRQ 的设备。消费者绑定期望以下属性：

+   `interrupt-parent`：对于生成中断的设备节点，它是一个包含指向设备附加的中断控制器节点的指针`phandle`的属性。如果省略，设备将从其父节点继承该属性。

+   `interrupts`：这是中断指定器。

中断绑定和中断指定器与中断控制器设备绑定。用于定义中断输入的单元格数取决于中断控制器，这是唯一决定的，通过其`#interrupt-cells`属性。在 i.MX6 的情况下，中断控制器是**全局中断控制器**（**GIC**）。其绑定在*Documentation/devicetree/bindings/arm/gic.txt*中有很好的解释。

# 中断处理程序

这包括从 DT 中获取 IRQ 号，并将其映射到 Linux IRQ，从而为其注册一个函数回调。执行此操作的驱动程序代码非常简单：

```
int irq = platform_get_irq(pdev, 0); 
ret = request_irq(irq, imx_rxint, 0, dev_name(&pdev->dev), sport); 
```

`platform_get_irq()`调用将返回`irq`号；这个数字可以被`devm_request_irq()`使用（`irq`然后在`/proc/interrupts`中可见）。第二个参数`0`表示我们需要设备节点中指定的第一个中断。如果有多个中断，我们可以根据需要更改此索引，或者只使用命名资源。

在我们之前的例子中，设备节点包含一个中断指定器，看起来像这样：

```
interrupts = <0 66 IRQ_TYPE_LEVEL_HIGH>; 
```

+   根据 ARM GIC，第一个单元格告诉我们中断类型：

+   `0` **：共享外围中断**（**SPI**），用于在核心之间共享的中断信号，可以由 GIC 路由到任何核心

+   `1`：**私有外围中断**（**PPI**），用于单个核心的私有中断信号

文档可以在以下网址找到：[`infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0407e/CCHDBEBE.html`](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.ddi0407e/CCHDBEBE.html)。

+   第二个单元格保存中断号。这个数字取决于中断线是 PPI 还是 SPI。

+   第三个单元格，在我们的情况下是`IRQ_TYPE_LEVEL_HIGH`，表示感应电平。所有可用的感应电平都在`include/linux/irq.h`中定义。

# 中断控制器代码

`interrupt-controller`属性用于声明设备为中断控制器。`#interrupt-cells`属性定义了必须使用多少个单元格来定义单个中断线。我们将在[第十六章](http://advanced)中详细讨论这个问题，*高级中断管理*。

# 提取应用程序特定数据

应用程序特定数据是超出常见属性（既不是资源也不是 GPIO、调节器等）的数据。这些是可以分配给设备的任意属性和子节点。这些属性名称通常以制造商代码为前缀。这些可以是任何字符串、布尔值或整数值，以及它们在 Linux 源代码中`drivers/of/base.c`中定义的 API。我们讨论的以下示例并不详尽。现在让我们重用本章前面定义的节点：

```
node_label: nodename@reg{ 
  string-property = ""a string""; 
  string-list = ""red fish"", ""blue fish""; 
  one-int-property = <197>; /* One cell in this property */ 
  int-list-property = <0xbeef 123 0xabcd4>;/* each number (cell) is 32      a                                        * bit integer(uint32). There 
                                         * are 3 cells in this property 
                                         */ 
    mixed-list-property = "a string", <0xadbcd45>, <35>, [0x01 0x23 0x45] 
    byte-array-property = [0x01 0x23 0x45 0x67]; 
    one-cell-property = <197>; 
    boolean-property; 
}; 
```

# 文本字符串

以下是一个`string`属性：

```
string-property = "a string"; 
```

回到驱动程序中，应该使用`of_property_read_string()`来读取字符串值。其原型定义如下：

```
int of_property_read_string(const struct device_node *np, const 
                        char *propname, const char **out_string) 
```

以下代码显示了如何使用它：

```
const char *my_string = NULL; 
of_property_read_string(pdev->dev.of_node, "string-property", &my_string); 
```

# 单元格和无符号 32 位整数

以下是我们的`int`属性：

```
one-int-property = <197>; 
int-list-property = <1350000 0x54dae47 1250000 1200000>; 
```

应该使用`of_property_read_u32()`来读取单元格值。其原型定义如下：

```
int of_property_read_u32_index(const struct device_node *np, 
                     const char *propname, u32 index, u32 *out_value) 
```

回到驱动程序中，

```
unsigned int number; 
of_property_read_u32(pdev->dev.of_node, "one-cell-property", &number); 
```

可以使用`of_property_read_u32_array`来读取单元格列表。其原型如下：

```
int of_property_read_u32_array(const struct device_node *np, 
                      const char *propname, u32 *out_values, size_tsz); 
```

在这里，`sz`是要读取的数组元素的数量。查看`drivers/of/base.c`以查看如何解释其返回值：

```
unsigned int cells_array[4]; 
if (of_property_read_u32_array(pdev->dev.of_node, "int-list-property", 
cells_array, 4)) { 
    dev_err(&pdev->dev, "list of cells not specified\n"); 
    return -EINVAL; 
} 
```

# 布尔值

应该使用`of_property_read_bool()`来读取函数的第二个参数中给定的布尔属性的名称：

```
bool my_bool = of_property_read_bool(pdev->dev.of_node, "boolean-property"); 
If(my_bool){ 
    /* boolean is true */ 
} else 
    /* Bolean is false */ 
} 
```

# 提取和解析子节点

您可以在设备节点中添加任何子节点。给定表示闪存设备的节点，分区可以表示为子节点。对于处理一组输入和输出 GPIO 的设备，每个集合可以表示为一个子节点。示例节点如下：

```
eeprom: ee24lc512@55 { 
        compatible = "microchip,24xx512"; 
reg = <0x55>; 

        partition1 { 
            read-only; 
            part-name = "private"; 
            offset = <0>; 
            size = <1024>; 
        }; 

        partition2 { 
            part-name = "data"; 
            offset = <1024>; 
            size = <64512>; 
        }; 
    }; 
```

可以使用`for_each_child_of_node()`来遍历给定节点的子节点：

```
struct device_node *np = pdev->dev.of_node; 
struct device_node *sub_np; 
for_each_child_of_node(np, sub_np) { 
        /* sub_np will point successively to each sub-node */ 
        [...] 
int size; 
        of_property_read_u32(client->dev.of_node, 
"size", &size); 
        ... 
 } 
```

# 平台驱动程序和 DT

平台驱动程序也可以使用 DT。也就是说，这是处理平台设备的推荐方式，而且不再需要触及板文件，甚至在设备属性更改时重新编译内核。如果您还记得，在上一章中我们讨论了 OF 匹配样式，这是一种基于 DT 的匹配机制。让我们在下一节中看看它是如何工作的：

# OF 匹配样式

OF 匹配样式是平台核心执行的第一个匹配机制，用于将设备与其驱动程序匹配。它使用设备树的`compatible`属性来匹配`of_match_table`中的设备条目，这是`struct driver`子结构的一个字段。每个设备节点都有一个`compatible`属性，它是一个字符串或字符串列表。任何声明在`compatible`属性中列出的字符串之一的平台驱动程序都将触发匹配，并将看到其`probe`函数执行。

DT 匹配条目在内核中被描述为`struct of_device_id`结构的一个实例，该结构在`linux/mod_devicetable.h`中定义，如下所示：

```
// we are only interested in the two last elements of the structure 
struct of_device_id { 
    [...] 
    char  compatible[128]; 
    const void *data; 
}; 
```

以下是结构的每个元素的含义：

+   `char compatible[128]`：这是用于匹配设备节点的 DT 兼容属性的字符串。在匹配发生之前，它们必须相同。

+   `const void *data`：这可以指向任何结构，可以根据设备类型配置数据使用。

由于`of_match_table`是一个指针，您可以传递`struct of_device_id`的数组，使您的驱动程序与多个设备兼容：

```
static const struct of_device_id imx_uart_dt_ids[] = { 
    { .compatible = "fsl,imx6q-uart", }, 
    { .compatible = "fsl,imx1-uart", }, 
    { .compatible = "fsl,imx21-uart", }, 
    { /* sentinel */ } 
}; 
```

一旦填充了 id 数组，它必须传递给平台驱动程序的`of_match_table`字段，在驱动程序子结构中：

```
static struct platform_driver serial_imx_driver = { 
    [...] 
    .driver     = { 
        .name   = "imx-uart", 
        .of_match_table = imx_uart_dt_ids, 
        [...] 
    }, 
}; 
```

在这一步，只有您的驱动程序知道您的`of_device_id`数组。为了让内核也知道（以便它可以将您的 ID 存储在平台核心维护的设备列表中），您的数组必须在`MODULE_DEVICE_TABLE`中注册，如第五章中所述，*平台设备驱动程序*：

```
MODULE_DEVICE_TABLE(of, imx_uart_dt_ids); 
```

就是这样！我们的驱动程序是 DT 兼容的。回到我们的 DT，在那里声明一个与我们的驱动程序兼容的设备：

```
uart1: serial@02020000 { 
    compatible = "fsl,imx6q-uart", "fsl,imx21-uart"; 
    reg = <0x02020000 0x4000>; 
    interrupts = <0 26 IRQ_TYPE_LEVEL_HIGH>; 
    [...] 
}; 
```

这里提供了两个兼容的字符串。如果第一个与任何驱动程序都不匹配，核心将使用第二个进行匹配。

当发生匹配时，将调用您的驱动程序的`probe`函数，参数是一个`struct platform_device`结构，其中包含一个`struct device dev`字段，在其中有一个`struct device_node *of_node`字段，对应于我们的设备关联的节点，因此可以使用它来提取设备设置：

```
static int serial_imx_probe(struct platform_device *pdev) 
{ 
    [...] 
struct device_node *np; 
np = pdev->dev.of_node; 

    if (of_get_property(np, "fsl,dte-mode", NULL)) 
        sport->dte_mode = 1; 
        [...] 
 }   
```

一个可以检查 DT 节点是否设置来知道驱动程序是否已经在`of_match`的响应中加载，或者是在板子的`init`文件中实例化。然后应该使用`of_match_device`函数，以选择发起匹配的`struct *of_device_id`条目，其中可能包含您传递的特定数据：

```
static int my_probe(struct platform_device *pdev) 
{ 
struct device_node *np = pdev->dev.of_node; 
const struct of_device_id *match; 

    match = of_match_device(imx_uart_dt_ids, &pdev->dev); 
    if (match) { 
        /* Devicetree, extract the data */ 
        my_data = match->data 
    } else { 
        /* Board init file */ 
        my_data = dev_get_platdata(&pdev->dev); 
    } 
    [...] 
} 
```

# 处理非设备树平台

在内核中启用了`CONFIG_OF`选项的情况下启用了 DT 支持。当内核中未启用 DT 支持时，人们可能希望避免使用 DT API。可以通过检查`CONFIG_OF`是否设置来实现。人们过去通常会做如下操作：

```
#ifdef CONFIG_OF 
    static const struct of_device_id imx_uart_dt_ids[] = { 
        { .compatible = "fsl,imx6q-uart", }, 
        { .compatible = "fsl,imx1-uart", }, 
        { .compatible = "fsl,imx21-uart", }, 
        { /* sentinel */ } 
    }; 

    /* other devicetree dependent code */ 
    [...] 
#endif 
```

即使在缺少设备树支持时，`of_device_id`数据类型总是定义的，但在构建过程中，被包装在`#ifdef CONFIG_OF ... #endif`中的代码将被省略。这用于条件编译。这不是您唯一的选择；还有`of_match_ptr`宏，当`OF`被禁用时，它简单地返回`NULL`。在您需要将`of_match_table`作为参数传递的任何地方，它都应该被包装在`of_match_ptr`宏中，以便在`OF`被禁用时返回`NULL`。该宏在`include/linux/of.h`中定义：

```
#define of_match_ptr(_ptr) (_ptr) /* When CONFIG_OF is enabled */ 
#define of_match_ptr(_ptr) NULL   /* When it is not */ 
```

我们可以这样使用它：

```
static int my_probe(struct platform_device *pdev) 
{ 
    const struct of_device_id *match; 
    match = of_match_device(of_match_ptr(imx_uart_dt_ids), 
                     &pdev->dev); 
    [...] 
} 
static struct platform_driver serial_imx_driver = { 
    [...] 
    .driver         = { 
    .name   = "imx-uart", 
    .of_match_table = of_match_ptr(imx_uart_dt_ids), 
    }, 
}; 
```

这消除了使用`#ifdef`，在`OF`被禁用时返回`NULL`。

# 支持具有每个特定设备数据的多个硬件

有时，驱动程序可以支持不同的硬件，每个硬件都有其特定的配置数据。这些数据可能是专用的函数表、特定的寄存器值，或者是每个硬件独有的任何内容。下面的示例描述了一种通用的方法：

让我们首先回顾一下`include/linux/mod_devicetable.h`中`struct of_device_id`的外观。

```
/* 
 * Struct used for matching a device 
 */ 
struct of_device_id { 
        [...] 
        char    compatible[128]; 
const void *data; 
}; 
```

我们感兴趣的字段是`const void *data`，所以我们可以使用它来为每个特定设备传递任何数据。

假设我们拥有三种不同的设备，每个设备都有特定的私有数据。`of_device_id.data`将包含指向特定参数的指针。这个示例受到了`drivers/tty/serial/imx.c`的启发。

首先，我们声明私有结构：

```
/* i.MX21 type uart runs on all i.mx except i.MX1 and i.MX6q */ 
enum imx_uart_type { 
    IMX1_UART, 
    IMX21_UART, 
    IMX6Q_UART, 
}; 

/* device type dependent stuff */ 
struct imx_uart_data { 
    unsigned uts_reg; 
    enum imx_uart_type devtype; 
}; 
```

然后我们用每个特定设备的数据填充一个数组：

```
static struct imx_uart_data imx_uart_devdata[] = { 
        [IMX1_UART] = { 
                 .uts_reg = IMX1_UTS, 
                 .devtype = IMX1_UART, 
        }, 
        [IMX21_UART] = { 
                .uts_reg = IMX21_UTS, 
                .devtype = IMX21_UART, 
        }, 
        [IMX6Q_UART] = { 
                .uts_reg = IMX21_UTS, 
                .devtype = IMX6Q_UART, 
        }, 
}; 
```

每个兼容条目都与特定的数组索引相关联：

```
static const struct of_device_idimx_uart_dt_ids[] = { 
        { .compatible = "fsl,imx6q-uart", .data = &imx_uart_devdata[IMX6Q_UART], }, 
        { .compatible = "fsl,imx1-uart", .data = &imx_uart_devdata[IMX1_UART], }, 
        { .compatible = "fsl,imx21-uart", .data = &imx_uart_devdata[IMX21_UART], }, 
        { /* sentinel */ } 
}; 
MODULE_DEVICE_TABLE(of, imx_uart_dt_ids); 

static struct platform_driver serial_imx_driver = { 
    [...] 
    .driver         = { 
        .name   = "imx-uart", 
        .of_match_table = of_match_ptr(imx_uart_dt_ids), 
    }, 
}; 
```

现在在`probe`函数中，无论匹配条目是什么，它都将保存指向特定设备结构的指针：

```
static int imx_probe_dt(struct platform_device *pdev) 
{ 
    struct device_node *np = pdev->dev.of_node; 
    const struct of_device_id *of_id = 
    of_match_device(of_match_ptr(imx_uart_dt_ids), &pdev->dev); 

        if (!of_id) 
                /* no device tree device */ 
                return 1; 
        [...] 
        sport->devdata = of_id->data; /* Get private data back  */ 
} 
```

在前面的代码中，`devdata`是原始源代码中结构的一个元素，并且声明为`const struct imx_uart_data *devdata`；我们可以在数组中存储任何特定的参数。

# 匹配样式混合

OF 匹配样式可以与任何其他匹配机制结合使用。在下面的示例中，我们混合了 DT 和设备 ID 匹配样式：

我们为设备 ID 匹配样式填充一个数组，每个设备都有自己的数据：

```
static const struct platform_device_id sdma_devtypes[] = { 
    { 
        .name = "imx51-sdma", 
        .driver_data = (unsigned long)&sdma_imx51, 
    }, { 
        .name = "imx53-sdma", 
        .driver_data = (unsigned long)&sdma_imx53, 
    }, { 
        .name = "imx6q-sdma", 
        .driver_data = (unsigned long)&sdma_imx6q, 
    }, { 
        .name = "imx7d-sdma", 
        .driver_data = (unsigned long)&sdma_imx7d, 
    }, { 
        /* sentinel */ 
    } 
}; 
MODULE_DEVICE_TABLE(platform, sdma_devtypes); 
```

我们对 OF 匹配样式也是一样的：

```
static const struct of_device_idsdma_dt_ids[] = { 
    { .compatible = "fsl,imx6q-sdma", .data = &sdma_imx6q, }, 
    { .compatible = "fsl,imx53-sdma", .data = &sdma_imx53, }, 
       { .compatible = "fsl,imx51-sdma", .data = &sdma_imx51, }, 
    { .compatible = "fsl,imx7d-sdma", .data = &sdma_imx7d, }, 
    { /* sentinel */ } 
}; 
MODULE_DEVICE_TABLE(of, sdma_dt_ids); 
```

`probe`函数将如下所示：

```
static int sdma_probe(structplatform_device *pdev) 
{ 
conststructof_device_id *of_id = 
of_match_device(of_match_ptr(sdma_dt_ids), &pdev->dev); 
structdevice_node *np = pdev->dev.of_node; 

    /* If devicetree, */ 
    if (of_id) 
drvdata = of_id->data; 
    /* else, hard-coded */ 
    else if (pdev->id_entry) 
drvdata = (void *)pdev->id_entry->driver_data; 

    if (!drvdata) { 
dev_err(&pdev->dev, "unable to find driver data\n"); 
        return -EINVAL; 
    } 
    [...] 
} 
```

然后我们声明我们的平台驱动程序；将所有在前面的部分中定义的数组都传递进去：

```
static struct platform_driversdma_driver = { 
    .driver = { 
    .name   = "imx-sdma", 
    .of_match_table = of_match_ptr(sdma_dt_ids), 
    }, 
    .id_table  = sdma_devtypes, 
    .remove  = sdma_remove, 
    .probe   = sdma_probe, 
}; 
module_platform_driver(sdma_driver); 
```

# 平台资源和 DT

平台设备可以在启用设备树的系统中工作，无需任何额外修改。这就是我们在“处理资源”部分中所展示的。通过使用`platform_xxx`系列函数，核心还会遍历 DT（使用`of_xxx`系列函数）以找到所需的资源。反之则不成立，因为`of_xxx`系列函数仅保留给 DT 使用。所有资源数据将以通常的方式提供给驱动程序。现在驱动程序知道这个设备是否是在板文件中以硬编码参数初始化的。让我们以一个 uart 设备节点为例：

```
uart1: serial@02020000 { 
    compatible = "fsl,imx6q-uart", "fsl,imx21-uart"; 
reg = <0x02020000 0x4000>; 
    interrupts = <0 26 IRQ_TYPE_LEVEL_HIGH>; 
dmas = <&sdma 25 4 0>, <&sdma 26 4 0>; 
dma-names = "rx", "tx"; 
}; 
```

以下摘录描述了其驱动程序的`probe`函数。在`probe`中，函数“platform_get_resource（）”可用于提取任何资源（内存区域、DMA、中断），或特定功能，如“platform_get_irq（）”，它提取 DT 中`interrupts`属性提供的`irq`：

```
static int my_probe(struct platform_device *pdev) 
{ 
struct iio_dev *indio_dev; 
struct resource *mem, *dma_res; 
struct xadc *xadc; 
int irq, ret, dmareq; 

    /* irq */ 
irq = platform_get_irq(pdev, 0); 
    if (irq<= 0) 
        return -ENXIO; 
    [...] 

    /* memory region */ 
mem = platform_get_resource(pdev, IORESOURCE_MEM, 0); 
xadc->base = devm_ioremap_resource(&pdev->dev, mem); 
    /* 
     * We could have used 
     *      devm_ioremap(&pdev->dev, mem->start, resource_size(mem)); 
     * too. 
     */ 
    if (IS_ERR(xadc->base)) 
        return PTR_ERR(xadc->base); 
    [...] 

    /* second dma channel */ 
dma_res = platform_get_resource(pdev, IORESOURCE_DMA, 1); 
dmareq = dma_res->start; 

    [...] 
} 
```

总之，对于诸如`dma`、`irq`和`mem`之类的属性，您在平台驱动程序中无需做任何匹配`dtb`的工作。如果有人记得，这些数据与作为平台资源传递的数据类型相同。要理解原因，我们只需查看这些函数的内部处理方式；我们将看到它们如何内部处理 DT 函数。以下是`platform_get_irq`函数的示例：

```
int platform_get_irq(struct platform_device *dev, unsigned int num) 
{ 
    [...] 
    struct resource *r; 
    if (IS_ENABLED(CONFIG_OF_IRQ) &&dev->dev.of_node) { 
        int ret; 

        ret = of_irq_get(dev->dev.of_node, num); 
        if (ret > 0 || ret == -EPROBE_DEFER) 
            return ret; 
    } 

    r = platform_get_resource(dev, IORESOURCE_IRQ, num); 
    if (r && r->flags & IORESOURCE_BITS) { 
        struct irq_data *irqd; 
        irqd = irq_get_irq_data(r->start); 
        if (!irqd) 
            return -ENXIO; 
        irqd_set_trigger_type(irqd, r->flags & IORESOURCE_BITS); 
    } 
    return r ? r->start : -ENXIO; 
} 
```

也许有人会想知道`platform_xxx`函数如何从 DT 中提取资源。这应该是`of_xxx`函数族。你是对的，但在系统启动期间，内核会在每个设备节点上调用“of_platform_device_create_pdata（）”，这将导致创建一个带有相关资源的平台设备，您可以在其上调用`platform_xxx`系列函数。其原型如下：

```
static struct platform_device *of_platform_device_create_pdata( 
                 struct device_node *np, const char *bus_id, 
                 void *platform_data, struct device *parent) 
```

# 平台数据与 DT

如果您的驱动程序期望平台数据，您应该检查`dev.platform_data`指针。非空值意味着您的驱动程序已在板配置文件中以旧方式实例化，并且 DT 不涉及其中。对于从 DT 实例化的驱动程序，`dev.platform_data`将为`NULL`，并且您的平台设备将获得指向与`dev.of_node`指针中对应于您设备的 DT 条目（节点）的指针，从中可以提取资源并使用 OF API 来解析和提取应用程序数据。

还有一种混合方法可以用来将在 C 文件中声明的平台数据与 DT 节点关联起来，但这只适用于特殊情况：DMA、IRQ 和内存。这种方法仅在驱动程序仅期望资源而不是特定应用程序数据时使用。

可以将 I2C 控制器的传统声明转换为 DT 兼容节点，如下所示：

```
#define SIRFSOC_I2C0MOD_PA_BASE 0xcc0e0000 
#define SIRFSOC_I2C0MOD_SIZE 0x10000 
#define IRQ_I2C0 
static struct resource sirfsoc_i2c0_resource[] = { 
    { 
        .start = SIRFSOC_I2C0MOD_PA_BASE, 
        .end = SIRFSOC_I2C0MOD_PA_BASE + SIRFSOC_I2C0MOD_SIZE - 1, 
        .flags = IORESOURCE_MEM, 
    },{ 
        .start = IRQ_I2C0, 
        .end = IRQ_I2C0, 
        .flags = IORESOURCE_IRQ, 
    }, 
}; 
```

和 DT 节点：

```
i2c0: i2c@cc0e0000 { 
    compatible = "sirf,marco-i2c"; 
    reg = <0xcc0e0000 0x10000>; 
    interrupt-parent = <&phandle_to_interrupt_controller_node> 
    interrupts = <0 24 0>; 
    #address-cells = <1>; 
    #size-cells = <0>; 
    status = "disabled"; 
}; 
```

# 总结

现在是从硬编码设备配置切换到 DT 的时候了。本章为您提供了处理 DT 所需的一切。现在您已经具备了自定义或添加任何节点和属性到 DT 中，并从驱动程序中提取它们的必要技能。在下一章中，我们将讨论 I2C 驱动程序，并使用 DT API 来枚举和配置我们的 I2C 设备。
