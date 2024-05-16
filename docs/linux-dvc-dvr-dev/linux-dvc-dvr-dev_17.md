# 第十七章：输入设备驱动程序

输入设备是可以与系统交互的设备。这些设备是按钮、键盘、触摸屏、鼠标等。它们通过发送事件来工作，由输入核心捕获并广播到系统中。本章将解释输入核心用于处理输入设备的每个结构。也就是说，我们将看到如何从用户空间管理事件。

在本章中，我们将涵盖以下主题：

+   输入核心数据结构

+   分配和注册输入设备，以及轮询设备系列

+   生成并向输入核心报告事件

+   用户空间的输入设备

+   编写驱动程序示例

# 输入设备结构

首先，要与输入子系统进行接口的主文件是 `linux/input.h`：

```
#include <linux/input.h> 
```

无论输入设备的类型是什么，它发送的事件的类型是什么，输入设备在内核中都表示为 `struct input_dev` 的实例：

```
struct input_dev { 
  const char *name; 
  const char *phys; 

  unsigned long evbit[BITS_TO_LONGS(EV_CNT)]; 
  unsigned long keybit[BITS_TO_LONGS(KEY_CNT)]; 
  unsigned long relbit[BITS_TO_LONGS(REL_CNT)]; 
  unsigned long absbit[BITS_TO_LONGS(ABS_CNT)]; 
  unsigned long mscbit[BITS_TO_LONGS(MSC_CNT)]; 

  unsigned int repeat_key; 

  int rep[REP_CNT]; 
  struct input_absinfo *absinfo; 
  unsigned long key[BITS_TO_LONGS(KEY_CNT)]; 

  int (*open)(struct input_dev *dev); 
  void (*close)(struct input_dev *dev); 

  unsigned int users; 
  struct device dev; 

  unsigned int num_vals; 
  unsigned int max_vals; 
  struct input_value *vals; 

  bool devres_managed; 
}; 
```

字段的含义如下：

+   `name` 表示设备的名称。

+   `phys` 是设备在系统层次结构中的物理路径。

+   `evbit` 是设备支持的事件类型的位图。一些类型的区域如下：

+   `EV_KEY` 用于支持发送键事件（键盘、按钮等）的设备。

+   `EV_REL` 用于支持发送相对位置的设备（鼠标、数字化器等）

+   `EV_ABS` 用于支持发送绝对位置（游戏手柄）的设备

事件列表在内核源代码中的 `include/linux/input-event-codes.h` 文件中可用。我们使用 `set_bit()` 宏来根据我们的输入设备功能设置适当的位。当然，设备可以支持多种类型的事件。例如，鼠标将同时设置 `EV_KEY` 和 `EV_REL`。

```
set_bit(EV_KEY, my_input_dev->evbit); 
set_bit(EV_REL, my_input_dev->evbit); 
```

+   `keybit` 用于启用 `EV_KEY` 类型的设备，是该设备公开的键/按钮的位图。例如，`BTN_0`，`KEY_A`，`KEY_B`等。键/按钮的完整列表在 `include/linux/input-event-codes.h` 文件中。

+   `relbit` 用于启用 `EV_REL` 类型的设备，是设备的相对轴的位图。例如，`REL_X`，`REL_Y`，`REL_Z`，`REL_RX`等。请查看 `include/linux/input-event-codes.h` 获取完整列表。

+   `absbit` 用于启用 `EV_ABS` 类型的设备，是设备的绝对轴的位图。例如，`ABS_Y`，`ABS_X`等。请查看相同的先前文件以获取完整列表。

+   `mscbit` 用于启用 `EV_MSC` 类型的设备，是设备支持的各种杂项事件的位图。

+   `repeat_key` 存储最后按下的键的键码；用于实现软件自动重复。

+   `rep`，自动重复参数（延迟、速率）的当前值。

+   `absinfo` 是一个 `&struct input_absinfo` 元素的数组，其中包含有关绝对轴的信息（当前值、最小值、最大值、平坦值、模糊值、分辨率）。您应该使用 `input_set_abs_params()` 函数来设置这些值。

```
void input_set_abs_params(struct input_dev *dev, unsigned int axis, 

                             int min, int max, int fuzz, int flat) 
```

+   `min` 和 `max` 指定了较低和较高的边界值。`fuzz` 表示指定输入设备的指定通道上的预期噪音。以下是一个示例，我们仅设置每个通道的边界：

```
#define ABSMAX_ACC_VAL 0x01FF 
#define ABSMIN_ACC_VAL -(ABSMAX_ACC_VAL) 
[...] 
set_bit(EV_ABS, idev->evbit); 
input_set_abs_params(idev, ABS_X, ABSMIN_ACC_VAL, 
                     ABSMAX_ACC_VAL, 0, 0); 
input_set_abs_params(idev, ABS_Y, ABSMIN_ACC_VAL, 
                     ABSMAX_ACC_VAL, 0, 0); 
input_set_abs_params(idev, ABS_Z, ABSMIN_ACC_VAL, 
                     ABSMAX_ACC_VAL, 0, 0); 
```

+   `key` 反映了设备键/按钮的当前状态。

+   `open` 是在第一个用户调用 `input_open_device()` 时调用的方法。使用此方法来准备设备，例如中断请求、轮询线程启动等。

+   `close` 在最后一个用户调用 `input_close_device()` 时被调用。在这里，您可以停止轮询（这会消耗大量资源）。

+   `users` 存储了打开此设备的用户（输入处理程序）的数量。它被 `input_open_device()` 和 `input_close_device()` 使用，以确保只有在第一个用户打开设备时才调用 `dev->open()`，并且在最后一个用户关闭设备时调用 `dev->close()`。

+   `dev` 是与此设备关联的设备结构（用于设备模型）。

+   `num_vals` 是当前帧中排队的值的数量。

+   `max_vals` 是在一个帧中排队的值的最大数量。

+   `Vals` 是当前帧中排队的值的数组。

+   `devres_managed` 表示设备由 `devres` 框架管理，不需要显式取消注册或释放。

# 分配和注册输入设备

在注册并向输入设备发送事件之前，应使用 `input_allocate_device()` 函数为其分配内存。为了释放先前为未注册的输入设备分配的内存，应使用 `input_free_device()` 函数。如果设备已经注册，应改用 `input_unregister_device()`。像每个需要内存分配的函数一样，我们可以使用函数的资源管理版本：

```
struct input_dev *input_allocate_device(void) 
struct input_dev *devm_input_allocate_device(struct device *dev) 

void input_free_device(struct input_dev *dev) 
static void devm_input_device_unregister(struct device *dev, 
                                         void *res) 
int input_register_device(struct input_dev *dev) 
void input_unregister_device(struct input_dev *dev) 
```

设备分配可能会休眠，因此不能在原子上下文中调用，也不能在持有自旋锁时调用。

以下是一个位于 I2C 总线上的输入设备的 `probe` 函数的摘录：

```
struct input_dev *idev; 
int error; 

idev = input_allocate_device(); 
if (!idev) 
    return -ENOMEM; 

idev->name = BMA150_DRIVER; 
idev->phys = BMA150_DRIVER "/input0"; 
idev->id.bustype = BUS_I2C; 
idev->dev.parent = &client->dev; 

set_bit(EV_ABS, idev->evbit); 
input_set_abs_params(idev, ABS_X, ABSMIN_ACC_VAL, 
                     ABSMAX_ACC_VAL, 0, 0); 
input_set_abs_params(idev, ABS_Y, ABSMIN_ACC_VAL, 
                     ABSMAX_ACC_VAL, 0, 0); 
input_set_abs_params(idev, ABS_Z, ABSMIN_ACC_VAL, 
                     ABSMAX_ACC_VAL, 0, 0); 

error = input_register_device(idev); 
if (error) { 
    input_free_device(idev); 
    return error; 
} 

error = request_threaded_irq(client->irq, 
            NULL, my_irq_thread, 
            IRQF_TRIGGER_RISING | IRQF_ONESHOT, 
            BMA150_DRIVER, NULL); 
if (error) { 
    dev_err(&client->dev, "irq request failed %d, error %d\n", 
            client->irq, error); 
    input_unregister_device(bma150->input); 
    goto err_free_mem; 
} 
```

# 轮询输入设备子类

轮询输入设备是一种特殊类型的输入设备，它依赖轮询来感知设备状态的变化，而通用输入设备类型依赖于 IRQ 来感知变化并将事件发送到输入核心。

内核中描述了一个轮询输入设备，它是 `struct input_polled_dev` 结构的一个实例，它是通用 `struct input_dev` 结构的一个包装器：

```
struct input_polled_dev { 
    void *private; 

    void (*open)(struct input_polled_dev *dev); 
    void (*close)(struct input_polled_dev *dev); 
    void (*poll)(struct input_polled_dev *dev); 
    unsigned int poll_interval; /* msec */ 
    unsigned int poll_interval_max; /* msec */ 
    unsigned int poll_interval_min; /* msec */ 

    struct input_dev *input; 

    bool devres_managed; 
}; 
```

这个结构中元素的含义如下：

+   `private` 是驱动程序的私有数据。

+   `open` 是一个可选的方法，用于准备设备进行轮询（启用设备，可能刷新设备状态）。

+   `close` 是一个可选的方法，当设备不再被轮询时调用。它用于将设备置于低功耗模式。

+   `poll` 是一个强制性的方法，每当需要轮询设备时都会调用。它以 `poll_interval` 的频率调用。

+   `poll_interval` 是应调用 `poll()` 方法的频率。默认为 500 毫秒，除非在注册设备时被覆盖。

+   `poll_interval_max` 指定了轮询间隔的上限。默认为 `poll_interval` 的初始值。

+   `poll_interval_min` 指定了轮询间隔的下限。默认为 0。

+   `input` 是轮询设备构建的输入设备。它必须由驱动程序正确初始化（ID、名称、位）。轮询输入设备只提供了一个接口，用于使用轮询而不是 IRQ 来感知设备状态变化。

使用 `input_allocate_polled_device()` 和 `input_free_polled_device()` 来分配/释放 `struct input_polled_dev` 结构。您应该注意初始化其中嵌入的 `struct input_dev` 的强制性字段。轮询间隔也应该设置，否则默认为 500 毫秒。也可以使用资源管理版本。两个原型如下：

```
struct input_polled_dev *devm_input_allocate_polled_device(struct             device *dev) 
struct input_polled_dev *input_allocate_polled_device(void) 
void input_free_polled_device(struct input_polled_dev *dev) 
```

对于资源管理的设备，输入核心将设置字段 `input_dev->devres_managed` 为 true。

在分配和正确初始化字段之后，可以使用 `input_register_polled_device()` 注册轮询输入设备，成功时返回 0。反向操作（取消注册）使用 `input_unregister_polled_device()` 函数完成：

```
int input_register_polled_device(struct input_polled_dev *dev) 
void  input_unregister_polled_device(struct input_polled_dev *dev) 
```

这样的设备的 `probe()` 函数的典型示例如下：

```
static int button_probe(struct platform_device *pdev) 
{ 
    struct my_struct *ms; 
    struct input_dev *input_dev; 
    int retval; 

    ms = devm_kzalloc(&pdev->dev, sizeof(*ms), GFP_KERNEL); 
    if (!ms) 
        return -ENOMEM; 

    ms->poll_dev = input_allocate_polled_device(); 
    if (!ms->poll_dev){ 
        kfree(ms); 
        return -ENOMEM; 
    } 

    /* This gpio is not mapped to IRQ */ 
    ms->reset_btn_desc = gpiod_get(dev, "reset", GPIOD_IN); 

    ms->poll_dev->private = ms ; 
    ms->poll_dev->poll = my_btn_poll; 
    ms->poll_dev->poll_interval = 200; /* Poll every 200ms */ 
    ms->poll_dev->open = my_btn_open; /* consist */ 

    input_dev = ms->poll_dev->input; 
    input_dev->name = "System Reset Btn"; 

    /* The gpio belong to an expander sitting on I2C */ 
    input_dev->id.bustype = BUS_I2C;  
    input_dev->dev.parent = &pdev->dev; 

    /* Declare the events generated by this driver */ 
    set_bit(EV_KEY, input_dev->evbit); 
    set_bit(BTN_0, input_dev->keybit); /* buttons */ 

    retval = input_register_polled_device(mcp->poll_dev); 
    if (retval) { 
        dev_err(&pdev->dev, "Failed to register input device\n"); 
        input_free_polled_device(ms->poll_dev); 
        kfree(ms);   
    } 
    return retval; 
} 
```

以下是我们的 `struct my_struct` 结构的样子：

```
struct my_struct { 
    struct gpio_desc *reset_btn_desc; 
    struct input_polled_dev *poll_dev; 
} 
```

以下是 `open` 函数的样子：

```
static void my_btn_open(struct input_polled_dev *poll_dev) 
{ 
    struct my_strut *ms = poll_dev->private; 
    dev_dbg(&ms->poll_dev->input->dev, "reset open()\n"); 
} 
```

`open` 方法用于准备设备所需的资源。对于这个例子，我们实际上不需要这个方法。

# 生成和报告输入事件

设备分配和注册是必不可少的，但它们不是输入设备驱动程序的主要目标，输入设备驱动程序旨在向输入核心报告。根据设备支持的事件类型，内核提供了适当的 API 来将它们报告给核心。

给定一个支持 `EV_XXX` 的设备，相应的报告函数将是 `input_report_xxx()` 。以下表格显示了最重要的事件类型及其报告函数之间的映射关系：

| **事件类型** | **报告函数** | **代码示例** |
| --- | --- | --- |
| `EV_KEY` | `input_report_key()` | `input_report_key(poll_dev->input, BTN_0, gpiod_get_value(ms-> reset_btn_desc) & 1)` ; |
| `EV_REL` | `input_report_rel()` | `input_report_rel(nunchuk->input, REL_X, (nunchuk->report.joy_x - 128)/10)` ; |
| `EV_ABS` | `input_report_abs()` | `input_report_abs(bma150->input, ABS_X, x_value)` ;`input_report_abs(bma150->input, ABS_Y, y_value)` ;`input_report_abs(bma150->input, ABS_Z, z_value)` ; |

它们的原型如下：

```
void input_report_abs(struct input_dev *dev, 
                      unsigned int code, int value) 
void input_report_key(struct input_dev *dev, 
                      unsigned int code, int value) 
void input_report_rel(struct input_dev *dev, 
                      unsigned int code, int value) 
```

可用报告函数的列表可以在内核源文件 `include/linux/input.h` 中找到。它们都具有相同的框架：

+   `dev` 是负责事件的输入设备。

+   `code` 表示事件代码，例如 `REL_X` 或 `KEY_BACKSPACE` 。完整的列表在 `include/linux/input-event-codes.h` 中。

+   `value` 是事件携带的值。对于 `EV_REL` 事件类型，它携带相对变化。对于 `EV_ABS`（如摇杆等）事件类型，它包含绝对的新值。对于 `EV_KEY` 事件类型，应设置为 `0` 表示按键释放，`1` 表示按键按下，`2` 表示自动重复。

在报告了所有更改之后，驱动程序应调用 `input_sync()` 来指示输入设备此事件已完成。输入子系统将这些事件收集到一个数据包中，并通过 `/dev/input/event<X>` 发送，这是表示系统上的 `struct input_dev` 的字符设备，其中 `<X>` 是输入核心分配给驱动程序的接口号：

```
void input_sync(struct input_dev *dev) 
```

让我们看一个示例，这是 `drivers/input/misc/bma150.c` 中 `bma150` 数字加速传感器驱动程序的摘录：

```
static void threaded_report_xyz(struct bma150_data *bma150) 
{ 
  u8 data[BMA150_XYZ_DATA_SIZE]; 
  s16 x, y, z; 
  s32 ret; 

  ret = i2c_smbus_read_i2c_block_data(bma150->client, 
      BMA150_ACC_X_LSB_REG, BMA150_XYZ_DATA_SIZE, data); 
  if (ret != BMA150_XYZ_DATA_SIZE) 
    return; 

  x = ((0xc0 & data[0]) >> 6) | (data[1] << 2); 
  y = ((0xc0 & data[2]) >> 6) | (data[3] << 2); 
  z = ((0xc0 & data[4]) >> 6) | (data[5] << 2); 

  /* sign extension */ 
  x = (s16) (x << 6) >> 6; 
  y = (s16) (y << 6) >> 6; 
  z = (s16) (z << 6) >> 6; 

  input_report_abs(bma150->input, ABS_X, x); 
  input_report_abs(bma150->input, ABS_Y, y); 
  input_report_abs(bma150->input, ABS_Z, z); 
  /* Indicate this event is complete */ 
  input_sync(bma150->input); 
} 
```

在前面的示例中，`input_sync()` 告诉核心将这三个报告视为同一事件。这是有道理的，因为位置有三个轴（X、Y、Z），我们不希望 X、Y 或 Z 分别报告。

报告事件的最佳位置是在轮询设备的 `poll` 函数中，或者在启用了 IRQ 的设备的 IRQ 例程（线程部分或非线程部分）中。如果执行了可能休眠的操作，应在 IRQ 处理的线程部分内处理报告：

```
static void my_btn_poll(struct input_polled_dev *poll_dev) 
{ 
    struct my_struct *ms = poll_dev->private; 
    struct i2c_client *client = mcp->client; 

    input_report_key(poll_dev->input, BTN_0, 
                     gpiod_get_value(ms->reset_btn_desc) & 1); 
    input_sync(poll_dev->input); 
} 
```

# 用户空间接口

每个注册的输入设备都由 `/dev/input/event<X>` 字符设备表示，我们可以从用户空间读取该设备的事件。读取此文件的应用程序将以 `struct input_event` 格式接收事件数据包：

```
struct input_event { 
  struct timeval time; 
  __u16 type; 
  __u16 code; 
  __s32 value; 
} 
```

让我们看看结构中每个元素的含义：

+   `time` 是时间戳，它返回事件发生的时间。

+   `type` 是事件类型。例如，`EV_KEY` 表示按键按下或释放，`EV_REL` 表示相对移动，`EV_ABS` 表示绝对移动。更多类型在 `include/linux/input-event-codes.h` 中定义。

+   `code` 是事件代码，例如：`REL_X` 或 `KEY_BACKSPACE` ，完整的列表在 `include/linux/input-event-codes.h` 中。

+   `value` 是事件携带的值。对于 `EV_REL` 事件类型，它携带相对变化。对于 `EV_ABS`（如摇杆等）事件类型，它包含绝对的新值。对于 `EV_KEY` 事件类型，应设置为 `0` 表示按键释放，`1` 表示按键按下，`2` 表示自动重复。

用户空间应用程序可以使用阻塞和非阻塞读取，还可以使用 `poll()` 或 `select()` 系统调用来在打开此设备后接收事件通知。以下是一个使用 `select()` 系统调用的示例，完整的源代码在书籍源代码库中提供：

```
#include <unistd.h> 
#include <fcntl.h> 
#include <stdio.h> 
#include <stdlib.h> 
#include <linux/input.h> 
#include <sys/select.h> 

#define INPUT_DEVICE "/dev/input/event1" 

int main(int argc, char **argv) 
{    
    int fd; 
    struct input_event event; 
    ssize_t bytesRead; 

    int ret; 
    fd_set readfds; 

    fd = open(INPUT_DEVICE, O_RDONLY); 
    /* Let's open our input device */ 
    if(fd < 0){ 
        fprintf(stderr, "Error opening %s for reading", INPUT_DEVICE); 
        exit(EXIT_FAILURE); 
    } 

    while(1){  
        /* Wait on fd for input */ 
        FD_ZERO(&readfds); 
        FD_SET(fd, &readfds); 

        ret = select(fd + 1, &readfds, NULL, NULL, NULL); 
        if (ret == -1) { 
            fprintf(stderr, "select call on %s: an error ocurred", 
                    INPUT_DEVICE); 
            break; 
        } 
        else if (!ret) { /* If we have decided to use timeout */ 
            fprintf(stderr, "select on %s: TIMEOUT", INPUT_DEVICE); 
            break; 
        } 

        /* File descriptor is now ready */ 
        if (FD_ISSET(fd, &readfds)) { 
            bytesRead = read(fd, &event, 
                             sizeof(struct input_event)); 
            if(bytesRead == -1) 
                /* Process read input error*/ 
            if(bytesRead != sizeof(struct input_event)) 
                /* Read value is not an input even */ 

            /*  
             * We could have done a switch/case if we had 
             * many codes to look for 
             */ 
            if(event.code == BTN_0) { 
                /* it concerns our button */ 
                if(event.value == 0){ 
                    /* Process Release */ 
                    [...] 
                } 
                else if(event.value == 1){ 
                    /* Process KeyPress */ 
                    [...] 
                } 
            } 
        } 
    } 
    close(fd); 
    return EXIT_SUCCESS; 
} 
```

# 将所有内容整合在一起

到目前为止，我们已经描述了在编写输入设备驱动程序时使用的结构，以及它们如何可以从用户空间进行管理。

1.  根据其类型，轮询或非轮询，使用`input_allocate_polled_device()`或`input_allocate_device()`分配新的输入设备。

1.  填写强制字段或不填写（如果有必要）：

+   +   通过在`input_dev.evbit`字段上使用`set_bit()`辅助宏指定设备支持的事件类型

+   根据事件类型，`EV_REL`、`EV_ABS`、`EV_KEY`或其他，指定此设备可以报告的代码，使用`input_dev.relbit`、`input_dev.absbit`、`input_dev.keybit`或其他。

+   指定`input_dev.dev`以设置正确的设备树

+   如有必要，填写`abs_`信息

+   对于轮询设备，请指定应调用`poll()`函数的间隔：

1.  如果有必要，请编写您的`open()`函数，在其中应准备和设置设备使用的资源。此函数仅调用一次。在此函数中，设置 GPIO，如有需要请求中断，初始化设备。

1.  编写您的`close()`函数，在其中释放和释放`open()`函数中完成的内容。例如，释放 GPIO，IRQ，将设备置于省电模式。

1.  将您的`open()`或`close()`函数（或两者）传递给`input_dev.open`和`input_dev.close`字段。

1.  如果是轮询的，请使用`input_register_polled_device()`注册您的设备，如果不是，请使用`input_register_device()`。

1.  在您的 IRQ 函数（线程化或非线程化）或`poll()`函数中，根据事件类型收集和报告事件，使用`input_report_key()`、`input_report_rel()`、`input_report_abs()`或其他，并在输入设备上调用`input_sync()`以指示帧结束（报告完成）。

通常的方法是，如果没有提供 IRQ，则使用经典输入设备，否则回退到轮询设备：

```
if(client->irq > 0){ 
    /* Use generic input device */ 
} else { 
    /* Use polled device */ 
} 
```

查看如何从用户空间管理这些设备，请参考书籍源代码中提供的示例。

# 驱动程序示例

可以总结以下两个驱动程序。第一个是基于未映射到 IRQ 的 GPIO 的轮询输入设备。轮询输入核心将轮询 GPIO 以检测任何变化。此驱动程序配置为发送 0 键代码。每个 GPIO 状态对应于按键按下或释放：

```
#include <linux/kernel.h> 
#include <linux/module.h> 
#include <linux/slab.h> 
#include <linux/of.h>                   /* For DT*/ 
#include <linux/platform_device.h>      /* For platform devices */ 
#include <linux/gpio/consumer.h>        /* For GPIO Descriptor interface */ 
#include <linux/input.h> 
#include <linux/input-polldev.h> 

struct poll_btn_data { 
   struct gpio_desc *btn_gpiod; 
   struct input_polled_dev *poll_dev; 
}; 

static void polled_btn_open(struct input_polled_dev *poll_dev) 
{ 
    /* struct poll_btn_data *priv = poll_dev->private; */ 
    pr_info("polled device opened()\n"); 
} 

static void polled_btn_close(struct input_polled_dev *poll_dev) 
{ 
    /* struct poll_btn_data *priv = poll_dev->private; */ 
    pr_info("polled device closed()\n"); 
} 

static void polled_btn_poll(struct input_polled_dev *poll_dev) 
{ 
    struct poll_btn_data *priv = poll_dev->private; 

    input_report_key(poll_dev->input, BTN_0, gpiod_get_value(priv->btn_gpiod) & 1); 
    input_sync(poll_dev->input); 
} 

static const struct of_device_id btn_dt_ids[] = { 
    { .compatible = "packt,input-polled-button", }, 
    { /* sentinel */ } 
}; 

static int polled_btn_probe(struct platform_device *pdev) 
{ 
    struct poll_btn_data *priv; 
    struct input_polled_dev *poll_dev; 
    struct input_dev *input_dev; 
    int ret; 

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL); 
    if (!priv) 
        return -ENOMEM; 

    poll_dev = input_allocate_polled_device(); 
    if (!poll_dev){ 
        devm_kfree(&pdev->dev, priv); 
        return -ENOMEM; 
    } 

    /* We assume this GPIO is active high */ 
    priv->btn_gpiod = gpiod_get(&pdev->dev, "button", GPIOD_IN); 

    poll_dev->private = priv; 
    poll_dev->poll_interval = 200; /* Poll every 200ms */ 
    poll_dev->poll = polled_btn_poll; 
    poll_dev->open = polled_btn_open; 
    poll_dev->close = polled_btn_close; 
    priv->poll_dev = poll_dev; 

    input_dev = poll_dev->input; 
    input_dev->name = "Packt input polled Btn"; 
    input_dev->dev.parent = &pdev->dev; 

    /* Declare the events generated by this driver */ 
    set_bit(EV_KEY, input_dev->evbit); 
    set_bit(BTN_0, input_dev->keybit); /* buttons */ 

    ret = input_register_polled_device(priv->poll_dev); 
    if (ret) { 
        pr_err("Failed to register input polled device\n"); 
        input_free_polled_device(poll_dev); 
        devm_kfree(&pdev->dev, priv); 
        return ret; 
    } 

    platform_set_drvdata(pdev, priv); 
    return 0; 
} 

static int polled_btn_remove(struct platform_device *pdev) 
{ 
   struct poll_btn_data *priv = platform_get_drvdata(pdev); 
   input_unregister_polled_device(priv->poll_dev); 
    input_free_polled_device(priv->poll_dev); 
    gpiod_put(priv->btn_gpiod); 
   return 0; 
} 

static struct platform_driver mypdrv = { 
    .probe      = polled_btn_probe, 
    .remove     = polled_btn_remove, 
    .driver     = { 
        .name     = "input-polled-button", 
        .of_match_table = of_match_ptr(btn_dt_ids),   
        .owner    = THIS_MODULE, 
    }, 
}; 
module_platform_driver(mypdrv); 

MODULE_LICENSE("GPL"); 
MODULE_AUTHOR("John Madieu <john.madieu@gmail.com>"); 
MODULE_DESCRIPTION("Polled input device"); 
```

这第二个驱动程序根据按钮的 GPIO 映射到的 IRQ 向输入核心发送事件。当使用 IRQ 来检测按键按下或释放时，最好在边缘变化时触发中断：

```
#include <linux/kernel.h> 
#include <linux/module.h> 
#include <linux/slab.h> 
#include <linux/of.h>                   /* For DT*/ 
#include <linux/platform_device.h>      /* For platform devices */ 
#include <linux/gpio/consumer.h>        /* For GPIO Descriptor interface */ 
#include <linux/input.h> 
#include <linux/interrupt.h> 

struct btn_data { 
   struct gpio_desc *btn_gpiod; 
   struct input_dev *i_dev; 
   struct platform_device *pdev; 
   int irq; 
}; 

static int btn_open(struct input_dev *i_dev) 
{ 
    pr_info("input device opened()\n"); 
    return 0; 
} 

static void btn_close(struct input_dev *i_dev) 
{ 
    pr_info("input device closed()\n"); 
} 

static irqreturn_t packt_btn_interrupt(int irq, void *dev_id) 
{ 
    struct btn_data *priv = dev_id; 

   input_report_key(priv->i_dev, BTN_0, gpiod_get_value(priv->btn_gpiod) & 1); 
    input_sync(priv->i_dev); 
   return IRQ_HANDLED; 
} 

static const struct of_device_id btn_dt_ids[] = { 
    { .compatible = "packt,input-button", }, 
    { /* sentinel */ } 
}; 

static int btn_probe(struct platform_device *pdev) 
{ 
    struct btn_data *priv; 
    struct gpio_desc *gpiod; 
    struct input_dev *i_dev; 
    int ret; 

    priv = devm_kzalloc(&pdev->dev, sizeof(*priv), GFP_KERNEL); 
    if (!priv) 
        return -ENOMEM; 

    i_dev = input_allocate_device(); 
    if (!i_dev) 
        return -ENOMEM; 

    i_dev->open = btn_open; 
    i_dev->close = btn_close; 
    i_dev->name = "Packt Btn"; 
    i_dev->dev.parent = &pdev->dev; 
    priv->i_dev = i_dev; 
    priv->pdev = pdev; 

    /* Declare the events generated by this driver */ 
    set_bit(EV_KEY, i_dev->evbit); 
    set_bit(BTN_0, i_dev->keybit); /* buttons */ 

    /* We assume this GPIO is active high */ 
    gpiod = gpiod_get(&pdev->dev, "button", GPIOD_IN); 
    if (IS_ERR(gpiod)) 
        return -ENODEV; 

    priv->irq = gpiod_to_irq(priv->btn_gpiod); 
    priv->btn_gpiod = gpiod; 

    ret = input_register_device(priv->i_dev); 
    if (ret) { 
        pr_err("Failed to register input device\n"); 
        goto err_input; 
    } 

    ret = request_any_context_irq(priv->irq, 
                           packt_btn_interrupt, 
                           (IRQF_TRIGGER_FALLING | IRQF_TRIGGER_RISING), 
                           "packt-input-button", priv); 
    if (ret < 0) { 
        dev_err(&pdev->dev, 
            "Unable to acquire interrupt for GPIO line\n"); 
        goto err_btn; 
    } 

    platform_set_drvdata(pdev, priv); 
    return 0; 

err_btn: 
    gpiod_put(priv->btn_gpiod); 
err_input: 
    printk("will call input_free_device\n"); 
    input_free_device(i_dev); 
    printk("will call devm_kfree\n"); 
    return ret; 
} 

static int btn_remove(struct platform_device *pdev) 
{ 
    struct btn_data *priv; 
    priv = platform_get_drvdata(pdev); 
    input_unregister_device(priv->i_dev); 
    input_free_device(priv->i_dev); 
    free_irq(priv->irq, priv); 
    gpiod_put(priv->btn_gpiod); 
    return 0; 
} 

static struct platform_driver mypdrv = { 
    .probe      = btn_probe, 
    .remove     = btn_remove, 
    .driver     = { 
    .name     = "input-button", 
    .of_match_table = of_match_ptr(btn_dt_ids),   
    .owner    = THIS_MODULE, 
    }, 
}; 
module_platform_driver(mypdrv); 

MODULE_LICENSE("GPL"); 
MODULE_AUTHOR("John Madieu <john.madieu@gmail.com>"); 
MODULE_DESCRIPTION("Input device (IRQ based)"); 
```

对于这两个示例，当设备与模块匹配时，将在`/dev/input`目录中创建一个节点。该节点对应于我们示例中的`event0`。可以使用`udevadm`工具来显示有关设备的信息：

```
# udevadm info /dev/input/event0

P: /devices/platform/input-button.0/input/input0/event0

N: input/event0

S: input/by-path/platform-input-button.0-event

E: DEVLINKS=/dev/input/by-path/platform-input-button.0-event

E: DEVNAME=/dev/input/event0

E: DEVPATH=/devices/platform/input-button.0/input/input0/event0

E: ID_INPUT=1

E: ID_PATH=platform-input-button.0

E: ID_PATH_TAG=platform-input-button_0

E: MAJOR=13

E: MINOR=64

E: SUBSYSTEM=input

E: USEC_INITIALIZED=74842430

```

实际允许我们将事件键打印到屏幕的工具是`evtest`，给定输入设备的路径：

```
# evtest /dev/input/event0

input device opened()

Input driver version is 1.0.1

Input device ID: bus 0x0 vendor 0x0 product 0x0 version 0x0

Input device name: "Packt Btn"

Supported events:

Event type 0 (EV_SYN)

Event type 1 (EV_KEY)

Event code 256 (BTN_0)

```

由于第二个模块是基于 IRQ 的，可以轻松检查 IRQ 请求是否成功，并且它已被触发了多少次：

```
$ cat /proc/interrupts | grep packt

160: 0 0 0 0 gpio-mxc 0 packt-input-button

```

最后，可以连续按下/释放按钮，并检查 GPIO 的状态是否发生了变化：

```
$ cat /sys/kernel/debug/gpio | grep button

gpio-193 (button-gpio ) in hi

$ cat /sys/kernel/debug/gpio | grep button

gpio-193 (button-gpio ) in lo

```

# 总结

本章描述了整个输入框架，并突出了轮询和中断驱动输入设备之间的区别。在本章结束时，您将具备为任何输入驱动程序编写驱动程序的必要知识，无论其类型和支持的输入事件如何。还讨论了用户空间接口，并提供了示例。下一章将讨论另一个重要的框架，即 RTC，它是 PC 和嵌入式设备中时间管理的关键元素。
