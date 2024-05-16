# 第七章：解密 V4L2 和视频捕获设备驱动程序

视频一直是嵌入式系统中固有的。鉴于 Linux 是这些系统中常用的内核，可以毫不夸张地说它本身就原生支持视频。这就是所谓的**V4L2**，代表**Video 4 (for) Linux 2**。是的！*2*是因为有第一个版本，*V4L*。V4L2 通过内存管理功能和其他元素增强了 V4L，使得该框架尽可能通用。通过这个框架，Linux 内核能够处理摄像头设备和它们连接的桥接器，以及相关的 DMA 引擎。这些并不是 V4L2 支持的唯一元素。我们将从框架架构的介绍开始，了解它的组织方式，并浏览它包括的主要数据结构。然后，我们将学习如何设计和编写桥接设备驱动程序，负责 DMA 操作，最后，我们将深入研究子设备驱动程序。因此，在本章中，将涵盖以下主题：

+   框架架构和主要数据结构

+   视频桥设备驱动程序

+   子设备的概念

+   V4L2 控制基础设施

# 技术要求

本章的先决条件如下：

+   高级计算机体系结构知识和 C 编程技能

+   Linux 内核 v4.19.X 源代码，可在[`git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags`](https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git/refs/tags)获取。

# 框架架构和主要数据结构

视频设备变得越来越复杂。在这种设备中，硬件通常包括多个集成 IP，需要以受控的方式相互合作，这导致复杂的 V4L2 驱动程序。这要求在深入代码之前弄清楚架构，这正是本节要解决的要求。

众所周知，驱动程序通常在编程中反映硬件模型。在 V4L2 上下文中，各种 IP 组件被建模为称为子设备的软件块。V4L2 子设备通常是仅内核对象。此外，如果 V4L2 驱动程序实现了媒体设备 API（我们将在下一章[*第八章*]（B10985_08_ePub_AM.xhtml#_idTextAnchor342）中讨论，*与 V4L2 异步和媒体控制器框架集成*），这些子设备将自动继承自媒体实体，允许应用程序枚举子设备并使用媒体框架的实体、端口和链接相关的枚举 API 来发现硬件拓扑。

尽管使子设备可发现，驱动程序也可以决定以简单的方式使其可由应用程序配置。当子设备驱动程序和 V4L2 设备驱动程序都支持此功能时，子设备将在其上调用**ioctls**（输入/输出控制）的字符设备节点，以便查询、读取和写入子设备功能（包括控制），甚至在单个子设备端口上协商图像格式。

在驱动程序级别，V4L2 为驱动程序开发人员做了很多工作，因此他们只需实现与硬件相关的代码并注册相关设备。在继续之前，我们必须介绍构成 V4L2 核心的几个重要结构：

+   `struct v4l2_device`：硬件设备可能包含多个子设备，例如电视卡以及捕获设备，可能还有 VBI 设备或 FM 调谐器。`v4l2_device`是所有这些设备的根节点，负责管理所有子设备。

+   `struct video_device`：此结构的主要目的是提供众所周知的`/dev/videoX`或`/dev/v4l-subdevX`设备节点。此结构主要抽象了捕获接口，也称为`/dev/v4l-subdevX`节点及其文件操作。在子设备驱动程序中，只有核心访问底层子设备中的这个结构。

+   `struct vb2_queue`：对我来说，这是视频驱动程序中的主要数据结构，因为它在数据流逻辑和 DMA 操作的中心部分中使用，以及`struct vb2_v4l2_buffer`。

+   `struct v4l2_subdev`：这是负责实现特定功能并在 SoC 的视频系统中抽象特定功能的子设备。

`struct video_device`可以被视为所有设备和子设备的基类。当我们编写自己的驱动程序时，对这个数据结构的访问可能是直接的（如果我们正在处理桥接驱动程序）或间接的（如果我们正在处理子设备，因为子设备 API 抽象和隐藏了嵌入到每个子设备数据结构中的底层`struct video_device`）。

现在我们知道了这个框架由哪些数据结构组成。此外，我们介绍了它们的关系和各自的目的。现在是时候深入了解细节，介绍如何初始化和注册 V4L2 设备到系统中了。

## 初始化和注册 V4L2 设备

在被使用或成为系统的一部分之前，V4L2 设备必须被初始化和注册，这是本节的主要内容。一旦框架架构描述完成，我们就可以开始阅读代码了。在这个内核中，V4L2 设备是`struct v4l2_device`结构的一个实例。这是媒体框架中的最高数据结构，维护着媒体管道由哪些子设备组成，并充当桥接设备的父级。V4L2 驱动程序应该包括`<media/v4l2-device.h>`，这将引入`struct v4l2_device`的以下定义：

```
struct v4l2_device {
    struct device *dev;
    struct media_device *mdev;
    struct list_head subdevs;
    spinlock_t lock;
    char name[V4L2_DEVICE_NAME_SIZE];
    void (*notify)(struct v4l2_subdev *sd,
                   unsigned int notification, void *arg);
    struct v4l2_ctrl_handler *ctrl_handler;
    struct v4l2_prio_state prio;
    struct kref ref;
    void (*release)(struct v4l2_device *v4l2_dev);
};
```

与我们将在以下部分介绍的其他与视频相关的数据结构不同，此结构中只有少数字段。它们的含义如下：

+   `dev`是指向此 V4L2 设备的父`struct device`的指针。这将在注册时自动设置，`dev->driver_data`将指向这个`v4l2`结构。

+   `mdev`是指向此 V4L2 设备所属的`struct media_device`对象的指针。这个字段涉及媒体控制器框架，并将在相关部分介绍。如果不需要与媒体控制器框架集成，则可能为`NULL`。

+   `subdevs`是此 V4L2 设备的子设备列表。

+   `lock`是保护对此结构的访问的锁。

+   `name`是此 V4L2 设备的唯一名称。默认情况下，它是从驱动程序名称加上总线 ID 派生的。

+   `notify`是指向通知回调的指针，由子设备调用以通知此 V4L2 设备某些事件。

+   `ctrl_handler`是与此设备关联的控制处理程序。它跟踪此 V4L2 设备拥有的所有控件。如果没有控件，则可能为`NULL`。

+   `prio`是设备的优先级状态。

+   `ref`是核心用于引用计数的内部使用。

+   `release`是当此结构的最后一个用户退出时要调用的回调函数。

这个顶层结构通过相同的函数`v4l2_device_register()`初始化并注册到核心，其原型如下：

```
int v4l2_device_register(struct device *dev,
                         struct v4l2_device *v4l2_dev);
```

第一个`dev`参数通常是桥接总线相关设备数据结构的 struct device 指针。即`pci_dev`、`usb_device`或`platform_device`。

如果`dev->driver_data`字段为`NULL`，此函数将使其指向正在注册的实际`v4l2_dev`对象。此外，如果`v4l2_dev->name`为空，则将设置为从`dev driver name + dev device name`的连接结果。

但是，如果 `dev` 参数为 `NULL`，则在调用 `v4l2_device_register()` 之前必须设置 `v4l2_dev->name`。另一方面，可以使用 `v4l2_device_unregister()` 注销先前注册的 V4L2 设备，如下所示：

```
v4l2_device_unregister(struct v4l2_device *v4l2_dev);
```

调用此函数时，所有子设备也将被注销。这一切都与 V4L2 设备有关。但是，您应该记住，它是顶层结构，维护媒体设备的子设备列表，并充当桥接设备的父级。

现在我们已经完成了主要的 V4L2 设备（包含其他设备相关数据结构的设备）的初始化和注册，我们可以引入特定的设备驱动程序，从桥接驱动程序开始，这是特定于平台的。

# 引入视频设备驱动程序 - 桥接驱动程序

桥接驱动程序控制平台 `/USB/PCI/...` 硬件，负责 DMA 传输。这是处理从设备进行数据流的驱动程序。桥接驱动程序直接处理的主要数据结构之一是 `struct video_device`。此结构嵌入了执行视频流所需的整个元素，它与用户空间的第一个交互之一是在 `/dev/` 目录中创建设备文件。

`struct video_device` 结构在 `include/media/v4l2-dev.h` 中定义，这意味着驱动程序代码必须包含 `#include <media/v4l2-dev.h>`。以下是在定义它的头文件中看到的这个结构的样子：

```
struct video_device
{
#if defined(CONFIG_MEDIA_CONTROLLER)
    struct media_entity entity;
    struct media_intf_devnode *intf_devnode;
    struct media_pipeline pipe;
#endif
    const struct v4l2_file_operations *fops;
    u32 device_caps;
    struct device dev; struct cdev *cdev;
    struct v4l2_device *v4l2_dev;
    struct device *dev_parent;
    struct v4l2_ctrl_handler *ctrl_handler;
    struct vb2_queue *queue;
    struct v4l2_prio_state *prio;
    char name[32];
    enum vfl_devnode_type vfl_type;
    enum vfl_devnode_direction vfl_dir;
    int minor;
    u16 num;
    unsigned long flags; int index;
    spinlock_t fh_lock;
    struct list_head fh_list;
    void (*release)(struct video_device *vdev);
    const struct v4l2_ioctl_ops *ioctl_ops;
    DECLARE_BITMAP(valid_ioctls, BASE_VIDIOC_PRIVATE);
    struct mutex *lock;
};
```

不仅桥接驱动程序可以操作此结构 - 当涉及表示 V4L2 兼容设备（包括子设备）时，此结构是主要的 `v4l2` 结构。但是，根据驱动程序的性质（无论是桥接驱动程序还是子设备驱动程序），某些元素可能会有所不同或可能为 `NULL`。以下是结构中每个元素的描述：

+   `entity`、`intf_node` 和 `pipe` 是与媒体框架集成的一部分，我们将在同名部分中看到。前者从媒体框架内部抽象出视频设备（成为实体），而 `intf_node` 表示媒体接口设备节点，`pipe` 表示实体所属的流水线。

+   `fops` 表示视频设备文件节点的文件操作。V4L2 核心通过一些子系统所需的额外逻辑覆盖虚拟设备文件操作。

+   `cdev` 是字符设备结构，抽象出底层的 `/dev/videoX` 文件节点。`vdev->cdev->ops` 由 V4L2 核心设置为 `v4l2_fops`（在 `drivers/media/v4l2-core/v4l2-dev.c` 中定义）。`v4l2_fops` 实际上是一个通用的（在实现的操作方面）和面向 V4L2 的（在这些操作所做的方面）文件操作，分配给每个 `/dev/videoX` 字符设备，并包装在 `vdev->fops` 中定义的视频设备特定操作。在它们的返回路径上，`v4l2_fops` 中的每个回调将调用 `vdev->fops` 中的对应项。`v4l2_fops` 回调在调用 `vdev->fops` 中的真实操作之前执行一些合理性检查。例如，在用户空间对 `/dev/videoX` 文件发出的 `mmap()` 系统调用上，将首先调用 `v4l2_fops->mmap`，这将确保在调用之前设置了 `vdev->fops->mmap`，并在需要时打印调试消息。

+   `ctrl_handler`：默认值为 `vdev->v4l2_dev->ctrl_handler`。

+   `queue` 是与此设备节点关联的缓冲区管理队列。这是桥接驱动程序唯一可以操作的数据结构之一。这可能是 `NULL`，特别是当涉及非桥接视频驱动程序（例如子设备）时。

+   `prio` 是指向具有设备优先级状态的 `&struct v4l2_prio_state` 的指针。如果此状态为 `NULL`，则将使用 `v4l2_dev->prio`。

+   `name` 是视频设备的名称。

+   `vfl_type` 是 V4L 设备类型。可能的值由 `enum vfl_devnode_type` 定义，包括以下内容：

- `VFL_TYPE_GRABBER`：用于视频输入/输出设备

– `VFL_TYPE_VBI`：用于垂直空白数据（未解码）

– `VFL_TYPE_RADIO`：用于无线电卡

– `VFL_TYPE_SUBDEV`：用于 V4L2 子设备

– `VFL_TYPE_SDR`：软件定义无线电

– `VFL_TYPE_TOUCH`：用于触摸传感器

+   `vfl_dir` 是一个 V4L 接收器、发射器或内存到内存（表示为 m2m 或 mem2mem）设备。可能的值由 `enum vfl_devnode_direction` 定义，包括以下内容：

– `VFL_DIR_RX`：用于捕获设备

– `VFL_DIR_TX`：用于输出设备

– `VFL_DIR_M2M`：应该是 mem2mem 设备（读取内存到内存，也称为内存到内存设备）。mem2mem 设备是使用用户空间应用程序传递的内存缓冲区作为源和目的地的设备。这与当前和现有的仅使用其中一个的内存缓冲区的驱动程序不同。这样的设备在 V4L2 框架中不存在，但是存在对这种模型的需求，例如，用于 '调整器设备' 或 V4L2 回环驱动程序。

+   `v4l2_dev` 是此视频设备的 `v4l2_device` 父设备。

+   `dev_parent` 是此视频设备的设备父级。如果未设置，核心将使用 `vdev->v4l2_dev->dev` 进行设置。

+   `ioctl_ops` 是指向 `&struct v4l2_ioctl_ops` 的指针，它定义了一组 ioctl 回调。

+   `release` 是核心在视频设备的最后一个用户退出时调用的回调。这必须是非-`NULL`。

+   `lock` 是一个互斥锁，用于串行访问此设备。这是主要的串行化锁，通过它所有的 ioctls 都被串行化。桥接驱动程序通常会使用相同的互斥锁设置此字段，就像 *queue->lock* 一样，这是用于串行化访问队列的锁（串行化流）。但是，如果设置了 *queue->lock*，那么流 ioctls 将由单独的锁串行化。

+   `num` 是核心分配的实际设备节点索引。它对应于 `/dev/videoX` 中的 *X*。

+   `flags` 是视频设备的标志。您应该使用位操作来设置/清除/测试标志。它们包含一组 `&enum v4l2_video_device_flags` 标志。

+   `fh_list` 是一个 `struct v4l2_fh` 列表，描述了一个 V4L2 文件处理程序，可以跟踪为此视频设备打开的文件句柄的数量。`fh_lock` 是与此列表关联的锁。

+   `class` 对应于 sysfs 类。它由核心分配。此类条目对应于 `/sys/video4linux/` sysfs 目录。

## 初始化和注册视频设备

在注册之前，视频设备可以动态分配，使用 `video_device_alloc()`（简单调用 `kzalloc()`），或者静态嵌入到动态分配的结构中，这是大多数情况下的设备状态结构。

视频设备是使用 `video_device_alloc()` 动态分配的，就像以下示例中一样：

```
struct video_device * vdev;
vdev = video_device_alloc();
if (!vdev)
    return ERR_PTR(-ENOMEM);
vdev->release = video_device_release;
```

在前面的摘录中，最后一行提供了视频设备的 `release` 方法，因为 `.release` 字段必须是非-`NULL`。内核提供了 `video_device_release()` 回调。它只调用 `kfree()` 来释放分配的内存。

当它嵌入到设备状态结构中时，代码变为如下：

```
struct my_struct {
    [...]
    struct video_device vdev;
};
[...]
struct my_struct *my_dev;
struct video_device *vdev;
my_dev =	kzalloc(sizeof(struct my_struct), GFP_KERNEL);
if (!my_dev)
    return ERR_PTR(-ENOMEM);
vdev = &my_vdev->vdev;
/* Now work with vdev as our video_device struct */
vdev->release = video_device_release_empty;
[...]
```

在这里，视频设备不能单独释放，因为它是一个更大的整体的一部分。当视频设备嵌入到另一个结构中时，就像前面的示例中一样，它不需要任何东西被释放。在这一点上，由于释放回调必须是非-`NULL`，我们可以分配一个空函数，例如 `video_device_release_empty()`，也由内核提供。

我们已经完成了分配。在这一点上，我们可以使用 `video_register_device()` 来注册视频设备。以下是此函数的原型：

```
int video_register_device(struct video_device *vdev,
                           enum vfl_devnode_type type, int nr)
```

在上述原型中，`type` 指定了要注册的桥接设备的类型。它将被分配给 `vdev->vfl_type` 字段。在本章的其余部分，我们将考虑将其设置为 `VFL_TYPE_GRABBER`，因为我们正在处理视频捕获接口。`nr` 是所需的设备节点号（*0 == /dev/video0*，*1 == /dev/video1*，...）。但是，将其值设置为 `-1` 将指示内核选择第一个空闲索引并使用它。指定固定索引可能对构建复杂的 *udev* 规则很有用，因为设备节点名称是预先知道的。为了使注册成功，必须满足以下要求：

+   首先，*必须* 设置 `vdev->release` 函数，因为它不能是空的。如果不需要它，可以传递 V4L2 核心的空释放方法。

+   其次，*必须* 设置 `vdev->v4l2_dev` 指针；它应该指向视频设备的 V4L2 父设备。

+   最后，但不是强制的，您应该设置 `vdev->fops` 和 `vdev->ioctl_ops`。

`video_register_device()` 在成功时返回 `0`。但是，如果没有空闲的次要设备，找不到设备节点号，或者设备节点的注册失败，它可能会失败。在任何错误情况下，它都会返回一个负的错误号。每个注册的视频设备都会在 `/sys/class/video4linux` 中创建一个目录条目，并在其中包含一些属性。

重要提示

次要号是动态分配的，除非内核使用内核选项 `CONFIG_VIDEO_FIXED_MINOR_RANGES` 进行编译。在这种情况下，次要号根据设备节点类型（视频、收音机等）分配在不同的范围内，总限制为 `VIDEO_NUM_DEVICES`，设置为 `256`。

如果注册失败，`vdev->release()` 回调将永远不会被调用。在这种情况下，如果动态分配了 `video_device` 结构，您需要调用 `video_device_release()` 来释放它，或者如果 `video_device` 被嵌入其中，则释放您自己的结构。

在驱动程序卸载路径上，或者当不再需要视频节点时，您应该调用 `video_unregister_device()` 来注销视频设备，以便其节点可以被移除：

```
void video_unregister_device(struct video_device *vdev)
```

在上述调用之后，设备的 sysfs 条目将被移除，导致 *udev* 移除 `/dev/` 中的节点。

到目前为止，我们只讨论了注册过程中最简单的部分，但是视频设备中还有一些复杂的字段需要在注册之前初始化。这些字段通过提供视频设备文件操作、一致的一组 ioctl 回调以及最重要的是媒体队列和内存管理接口来扩展驱动程序的功能。我们将在接下来的章节中讨论这些内容。

## 视频设备文件操作

视频设备（通过其驱动程序）旨在作为 `/dev/` 目录中的特殊文件暴露给用户空间，用户空间可以使用它与底层设备进行交互：流式传输数据。为了使视频设备能够响应用户空间查询（通过系统调用），必须从驱动程序内部实现一组标准回调。这些回调形成了今天所知的 `struct v4l2_file_operations` 类型，定义在 `include/media/v4l2-dev.h` 中，如下所示：

```
struct v4l2_file_operations {
    struct module *owner;
    ssize_t (*read) (struct file *file, char user *buf,
                       size_t, loff_t *ppos);
    ssize_t (*write) (struct file *file, const char user *buf,
                       size_t, loff_t *ppos);
    poll_t (*poll) (struct file *file,
                      struct poll_table_struct *);
    long (*unlocked_ioctl) (struct file *file,
                          unsigned int cmd, unsigned long arg);
#ifdef CONFIG_COMPAT
     long (*compat_ioctl32) (struct file *file,
                          unsigned int cmd, unsigned long arg);
#endif
    unsigned long (*get_unmapped_area) (struct file *file,
                              unsigned long, unsigned long,
                              unsigned long, unsigned long);
    int (*mmap) (struct file *file,                  struct vm_area_struct *vma);
    int (*open) (struct file *file);
    int (*release) (struct file *file);
};
```

这些可以被视为顶层回调，因为它们实际上是由另一个低级设备文件操作调用的（当然，经过一些合理性检查），这次是与 `vdev->cdev` 字段相关联的低级设备文件操作，设置为 `vdev->cdev->ops = &v4l2_fops;` 在文件节点创建时。这允许内核实现额外的逻辑并强制执行合理性：

+   `owner` 是指向模块的指针。大多数情况下，它是 `THIS_MODULE`。

+   `open`应包含实现`open()`系统调用所需的操作。大多数情况下，这可以设置为`v4l2_fh_open`，这是一个 V4L2 助手，简单地分配和初始化一个`v4l2_fh`结构，并将其添加到`vdev->fh_list`列表中。但是，如果您的设备需要一些额外的初始化，请在内部执行初始化，然后调用`v4l2_fh_open(struct file * filp)`。无论如何，您*必须*处理`v4l2_fh_open`。

+   `release`应包含实现`close()`系统调用所需的操作。这个回调必须处理`v4l2_fh_release`。它可以设置为以下之一：

- `vb2_fop_release`，这是一个 videobuf2-V4L2 释放助手，将清理任何正在进行的流。这个助手将调用`v4l2_fh_release`。

- 撤销`.open`中所做的工作的自定义回调，并且必须直接或间接调用`v4l2_fh_release`（例如，使用`_vb2_fop_release()`助手），以便 V4L2 核心处理任何正在进行的流的清理。

+   `read`应包含实现`read()`系统调用所需的操作。大多数情况下，videobuf2-V4L2 助手`vb2_fop_read`就足够了。

+   `write`在我们的情况下不需要，因为它是用于输出类型设备。但是，在这里使用`vb2_fop_write`可以完成工作。

+   如果您使用`v4l2_ioctl_ops`，则必须将`unlocked_ioctl`设置为`video_ioctl2`。下一节将详细解释这一点。这个 V4L2 核心助手是`__video_do_ioctl()`的包装器，它处理真正的逻辑，并将每个 ioctl 路由到`vdev->ioctl_ops`中的适当回调，这是单独的 ioctl 处理程序定义的地方。

+   `mmap`应包含实现`mmap()`系统调用所需的操作。大多数情况下，videobuf2-V4L2 助手`vb2_fop_mmap`就足够了，除非在执行映射之前需要额外的元素。内核中的视频缓冲区（响应于`VIDIOC_REQBUFS`ioctl 而分配）在被访问用户空间之前必须单独映射。这就是这个`.mmap`回调的目的，它只需要将一个视频缓冲区映射到用户空间。查询将缓冲区映射到用户空间所需的信息是使用`VIDIOC_QUERYBUF`ioctl 向内核查询的。给定`vma`参数，您可以按如下方式获取指向相应视频缓冲区的指针：

```
struct vb2_queue *q = container_of_myqueue_wrapper();
unsigned long off = vma->vm_pgoff << PAGE_SHIFT;
struct vb2_buffer *vb;
unsigned int buffer = 0, plane = 0;
for (i = 0; i < q->num_buffers; i++) {
    struct vb2_buffer *buf = q->bufs[i];
    /* The below assume we are on a single-planar system,
     * else we would have loop over each plane
     */
    if (buf->planes[0].m.offset == off)
        break;
    return i;
}
videobuf_queue_unlock(myqueue);
```

+   `poll`应包含实现`poll()`系统调用所需的操作。大多数情况下，videobuf2-V4L2 助手`vb2_fop_call`就足够了。如果这个助手不知道如何锁定（`queue->lock`和`vdev->lock`都没有设置），那么您不应该使用它，而应该编写自己的助手，可以依赖于不处理锁定的`vb2_poll()`助手。

在这两个回调中，您可以使用`v4l2_fh_is_singular_file()`助手来检查给定的文件是否是关联`video_device`的唯一文件句柄。它的替代方法是`v4l2_fh_is_singular()`，这次依赖于`v4l2_fh`：

```
int v4l2_fh_is_singular_file(struct file *filp)
int v4l2_fh_is_singular(struct v4l2_fh *fh);
```

总之，捕获视频设备驱动程序的文件操作可能如下所示：

```
static int foo_vdev_open(struct file *file)
{
    struct mydev_state_struct *foo_dev = video_drvdata(file);
    int ret;
[...]
    if (!v4l2_fh_is_singular_file(file))
        goto fh_rel;
[...]
fh_rel:
    if (ret)
        v4l2_fh_release(file);
    return ret;
}
static int foo_vdev_release(struct file *file)
{
    struct mydev_state_struct *foo_dev = video_drvdata(file);
    bool fh_singular;
    int ret;
[...]
    fh_singular = v4l2_fh_is_singular_file(file);
    ret = _vb2_fop_release(file, NULL);
    if (fh_singular)
        /* do something */
        [...]
    return ret;
}
static const struct v4l2_file_operations foo_fops = {
    .owner = THIS_MODULE,
    .open = foo_vdev_open,
    .release = foo_vdev_release,
    .unlocked_ioctl = video_ioctl2,
    .poll = vb2_fop_poll,
    .mmap = vb2_fop_mmap,
    .read = vb2_fop_read,
};
```

您可以观察到，在前面的块中，我们在我们的文件操作中只使用了标准的核心助手。

重要提示

Mem2mem 设备可以使用它们相关的基于 v4l2-mem2mem 的助手。看看`drivers/media/v4l2-core/v4l2-mem2mem.c`。

## V4L2 ioctl 处理

让我们再谈谈`v4l2_file_operations.unlocked_ioctl`回调。正如我们在前一节中所看到的，它应该设置为`video_ioctl2`。`video_ioctl2`负责在内核和用户空间之间进行参数复制，并在将每个单独的`ioctl()`调用分派到驱动程序之前执行一些合理性检查（例如，ioctl 命令是否有效），这最终会进入`video_device->ioctl_ops`字段中的回调条目，该字段是`struct v4l2_ioctl_ops`类型。

`struct v4l2_ioctl_ops`结构包含了 V4L2 框架中每个可能的 ioctl 的回调。然而，你应该根据你的设备类型和驱动程序的能力来设置这些回调。结构中的每个回调都映射一个 ioctl，结构定义如下：

```
struct v4l2_ioctl_ops {
    /* VIDIOC_QUERYCAP handler */
    int (*vidioc_querycap)(struct file *file, void *fh,
                            struct v4l2_capability *cap);
    /* Buffer handlers */
    int (*vidioc_reqbufs)(struct file *file, void *fh,
                           struct v4l2_requestbuffers *b);
    int (*vidioc_querybuf)(struct file *file, void *fh,
                            struct v4l2_buffer *b);
    int (*vidioc_qbuf)(struct file *file, void *fh,
                        struct v4l2_buffer *b);
    int (*vidioc_expbuf)(struct file *file, void *fh,
                          struct v4l2_exportbuffer *e);
    int (*vidioc_dqbuf)(struct file *file, void *fh,
                          struct v4l2_buffer *b);
    int (*vidioc_create_bufs)(struct file *file, void *fh,
                               struct v4l2_create_buffers *b);
    int (*vidioc_prepare_buf)(struct file *file, void *fh,
                               struct v4l2_buffer *b);
    int (*vidioc_overlay)(struct file *file, void *fh,
                           unsigned int i);
[...]
};
```

这个结构有超过 120 个条目，描述了每一个可能的 V4L2 ioctl 的操作，无论设备类型是什么。在前面的摘录中，只列出了我们可能感兴趣的部分。我们不会在这个结构中引入回调。然而，当你到达[*第九章*]（B10985_09_ePub_AM.xhtml#_idTextAnchor396），*从用户空间利用 V4L2 API*时，我鼓励你回到这个结构，事情会更清楚。

也就是说，因为你提供了一个回调，它仍然是可访问的。有些情况下，你可能希望忽略在`v4l2_ioctl_ops`中指定的回调。如果基于外部因素（例如使用的卡），你希望在`v4l2_ioctl_ops`中关闭某些功能而不必创建新的结构，那么就需要这样做。为了让核心意识到这一点并忽略回调，你应该在调用`video_register_device()`之前对相关的 ioctl 命令调用`v4l2_disable_ioctl()`：

```
v4l2_disable_ioctl (vdev, cmd)
```

以下是一个例子：`v4l2_disable_ioctl(&tea->vd, VIDIOC_S_HW_FREQ_SEEK);`。前面的调用将标记`tea->vd`视频设备上的`VIDIOC_S_HW_FREQ_SEEK`ioctl 为被忽略。

## videobuf2 接口和 API

videobuf2 框架用于连接 V4L2 驱动程序层和用户空间层，提供了一个数据交换通道，可以分配和管理视频帧数据。videobuf2 内存管理后端是完全模块化的。这允许为具有非标准内存管理要求的设备和平台插入自定义内存管理例程，而无需更改高级缓冲区管理函数和 API。该框架提供以下功能：

+   实现流式 I/O V4L2 ioctls 和文件操作

+   高级视频缓冲区、视频队列和状态管理功能

+   视频缓冲区内存分配和管理

Videobuf2（或者只是 vb2）促进了驱动程序的开发，减少了驱动程序的代码大小，并有助于在驱动程序中正确和一致地实现 V4L2 API。然后 V4L2 驱动程序负责从传感器（通常通过某种 DMA 控制器）获取视频数据，并将其提供给由 vb2 框架管理的缓冲区。

这个框架实现了许多 ioctl 函数，包括缓冲区分配、入队、出队和数据流控制。然后废弃了任何特定于供应商的解决方案，大大减少了媒体框架代码的大小，并减轻了编写 V4L2 设备驱动程序所需的工作量。

重要提示

每个 videobuf2 助手、API 和数据结构都以`vb2_`为前缀，而版本 1（videobuf，定义在`drivers/media/v4l2-core/videobuf-core.c`中）的对应物使用了`videobuf_`前缀。

这个框架包括一些可能对你们一些人来说很熟悉的概念，但仍然需要详细讨论。

### 缓冲区的概念

缓冲区是在 vb2 和用户空间之间进行单次数据交换的单位。从用户空间代码的角度来看，V4L2 缓冲区代表与视频帧对应的数据（例如，在捕获设备的情况下）。流式传输涉及在内核和用户空间之间交换缓冲区。vb2 使用`struct vb2_buffer`数据结构来描述视频缓冲区。该结构在`include/media/videobuf2-core.h`中定义如下：

```
struct vb2_buffer {
    struct vb2_queue *vb2_queue;
    unsigned int index;
    unsigned int type;
    unsigned int memory;
    unsigned int num_planes;
    u64 timestamp;
    /* private: internal use only
     *
     * state: current buffer state; do not change
     * queued_entry: entry on the queued buffers list, which
     * holds all buffers queued from userspace
     * done_entry: entry on the list that stores all buffers
     * ready to be dequeued to userspace
     * vb2_plane: per-plane information; do not change
     */
    enum vb2_buffer_state state;
    struct vb2_plane planes[VB2_MAX_PLANES];
    struct list_head queued_entry;
    struct list_head done_entry;
[...]
};
```

在前面的数据结构中，我们已经删除了对我们没有兴趣的字段。剩下的字段定义如下：

+   `vb2_queue`是这个缓冲区所属的`vb2`队列。这将引导我们进入下一节，介绍根据 videobuf2 的队列概念。

+   `index`是这个缓冲区的 ID。

+   `type`是缓冲区的类型。它由`vb2`在分配时设置。它与其所属队列的类型匹配：`vb->type = q->type`。

+   `memory`是用于使缓冲区在用户空间可见的内存模型类型。此字段的值是`enum vb2_memory`类型，与其 V4L2 用户空间对应项`enum v4l2_memory`相匹配。此字段由`vb2`在缓冲区分配时设置，并报告了与`vIDIOC_REQBUFS`给定的`v4l2_requestbuffers`的`.memory`字段分配的用户空间值的 vb2 等价项。可能的值包括以下内容：

- `VB2_MEMORY_MMAP`：其在用户空间分配的等价物是`V4L2_MEMORY_MMAP`，表示缓冲区用于内存映射 I/O。

- `VB2_MEMORY_USERPTR`：其在用户空间分配的等价物是`V4L2_MEMORY_USERPTR`，表示用户在用户空间分配缓冲区，并通过`v4l2_buffer`的`buf.m.userptr`成员传递指针。V4L2 中`USERPTR`的目的是允许用户直接通过`malloc()`或静态方式传递在用户空间分配的缓冲区。

- `VB2_MEMORY_DMABUF`。其在用户空间分配的等价物是`V4L2_MEMORY_DMABUF`，表示内存由驱动程序分配并导出为 DMABUF 文件处理程序。这个 DMABUF 文件处理程序可以在另一个驱动程序中导入。

+   `state`是`enum vb2_buffer_state`类型，表示此视频缓冲区的当前状态。驱动程序可以使用`void vb2_buffer_done(struct vb2_buffer *vb, enum vb2_buffer_state state)` API 来更改此状态。可能的状态值包括以下内容：

- `VB2_BUF_STATE_DEQUEUED` 表示缓冲区在用户空间控制之下。这是由 videobuf2 核心在`VIDIOC_REQBUFS` ioctl 的执行路径中设置的。

- `VB2_BUF_STATE_PREPARING` 表示缓冲区正在 videobuf2 中准备。这个标志是由 videobuf2 核心在支持的驱动程序的`VIDIOC_PREPARE_BUF` ioctl 的执行路径中设置的。

- `VB2_BUF_STATE_QUEUED` 表示缓冲区在 videobuf 中排队，但尚未在驱动程序中。这是由 videobuf2 核心在`VIDIOC_QBUF` ioctl 的执行路径中设置的。然而，如果驱动程序无法启动流，则驱动程序必须将所有缓冲区的状态设置为`VB2_BUF_STATE_QUEUED`。这相当于将缓冲区返回给 videobuf2。

- `VB2_BUF_STATE_ACTIVE` 表示缓冲区实际上在驱动程序中排队，并可能在硬件操作（例如 DMA）中使用。驱动程序无需设置此标志，因为在调用缓冲区`.buf_queue`回调之前，核心会设置此标志。

- `VB2_BUF_STATE_DONE` 表示驱动程序应在此缓冲区的 DMA 操作成功路径上设置此标志，以将缓冲区传递给 vb2。这意味着 videobuf2 核心从驱动程序返回缓冲区，但尚未将其出队到用户空间。

- `VB2_BUF_STATE_ERROR` 与上述相同，但是对缓冲区的操作以错误结束，当它被出队时将向用户空间报告。

如果在阅读完后，缓冲区技能的概念对您来说显得复杂，那么我鼓励您先阅读*第九章*，*从用户空间利用 V4L2 API*，然后再回到这里。

#### 平面的概念

有些设备要求每个输入或输出视频帧的数据放在不连续的内存缓冲区中。在这种情况下，一个视频帧必须使用多个内存地址来寻址，换句话说，每个“平面”有一个指针。平面是当前帧的子缓冲区（或帧的一部分）。

因此，在单平面系统中，一个平面代表整个视频帧，而在多平面系统中，一个平面只代表视频帧的一部分。由于内存是不连续的，多平面设备使用 Scatter/Gather DMA。

### 队列的概念

队列是流处理的中心元素，是桥接驱动程序的 DMA 引擎相关部分。实际上，它是驱动程序向 videobuf2 介绍自己的元素。它帮助我们在驱动程序中实现数据流管理模块。队列通过以下结构表示：

```
struct vb2_queue {
    unsigned int type;
    unsigned int io_modes;
    struct device *dev;
    struct mutex *lock;
    const struct vb2_ops *ops;
    const struct vb2_mem_ops *mem_ops;
    const struct vb2_buf_ops *buf_ops;
    u32 min_buffers_needed;
    gfp_t gfp_flags;
    void *drv_priv;
    struct vb2_buffer *bufs[VB2_MAX_FRAME];
    unsigned int num_buffers;
    /* Lots of private and debug stuff omitted */
    [...]
};
```

结构应该被清零，并填写前面的字段。以下是结构中每个元素的含义：

+   `type` 是缓冲区类型。这应该使用`include/uapi/linux/videodev2.h`中定义的`enum v4l2_buf_type`中的一个值进行设置。在我们的情况下，这必须是`V4L2_BUF_TYPE_VIDEO_CAPTURE`。

+   `io_modes`是描述可以处理的缓冲区类型的位掩码。可能的值包括以下内容：

- `VB2_MMAP`：在内核中分配并通过`mmap()`访问的缓冲区；vmalloc'ed 和连续 DMA 缓冲区通常属于这种类型。

- `VB2_USERPTR`：这是为用户空间分配的缓冲区。通常，只有可以进行散射/聚集 I/O 的设备才能处理用户空间缓冲区。然而，不支持对巨大页面的连续 I/O。有趣的是，videobuf2 支持用户空间分配的连续缓冲区。不过，唯一的方法是使用某种特殊机制，比如非树 Android `pmem`驱动程序。

- `VB2_READ, VB2_WRITE`：这些是通过`read()`和`write()`系统调用提供的用户空间缓冲区。

+   `lock`是用于流 ioctls 的串行化锁的互斥体。通常将此锁与`video_device->lock`相同，这是主要的串行化锁。但是，如果一些非流 ioctls 需要很长时间才能执行，那么您可能希望在这里使用不同的锁，以防止`VIDIOC_DQBUF`在等待另一个操作完成时被阻塞。

+   `ops`代表特定于驱动程序的回调，用于设置此队列和控制流操作。它是`struct vb2_ops`类型。我们将在下一节详细讨论这个结构。

+   `mem_ops`字段是驱动程序告诉 videobuf2 它实际使用的缓冲区类型的地方；它应该设置为`vb2_vmalloc_memops`、`vb2_dma_contig_memops`或`vb2_dma_sg_memops`中的一个。这是 videobuf2 实现的三种基本类型的缓冲区分配：

- 第一种是`vmalloc()`，因此在内核空间中是虚拟连续的，不保证在物理上是连续的。

- 第二种是`vb2_mem_ops`，以满足这种需求。没有限制。

+   您可能不关心`buf_ops`，因为如果未设置，它由`vb2`核心提供。但是，它包含了在用户空间和内核空间之间传递缓冲区信息的回调。

+   `min_buffers_needed`是在开始流之前需要的最小缓冲区数量。如果这个值不为零，那么只有用户空间排队了至少这么多的缓冲区，`vb2_queue->ops->start_streaming`才会被调用。换句话说，它表示 DMA 引擎在启动之前需要有多少可用的缓冲区。

+   `bufs`是此队列中缓冲区的指针数组。它的最大值是`VB2_MAX_FRAME`，这对应于`vb2`核心允许每个队列的最大缓冲区数量。它被设置为`32`，这已经是一个相当可观的值。

+   `num_buffers`是队列中已分配/已使用的缓冲区数量。

#### 特定于驱动程序的流回调

桥接驱动程序需要公开一系列函数来管理缓冲区队列，包括队列和缓冲区初始化。这些函数将处理来自用户空间的缓冲区分配、排队和与流相关的请求。这可以通过设置`struct vb2_ops`的实例来完成，定义如下：

```
struct vb2_ops {
    int (*queue_setup)(struct vb2_queue *q,
                       unsigned int *num_buffers,                        unsigned int *num_planes,
                       unsigned int sizes[],                        struct device *alloc_devs[]);
    void (*wait_prepare)(struct vb2_queue *q);
    void (*wait_finish)(struct vb2_queue *q);
    int (*buf_init)(struct vb2_buffer *vb);
    int (*buf_prepare)(struct vb2_buffer *vb);
    void (*buf_finish)(struct vb2_buffer *vb);
    void (*buf_cleanup)(struct vb2_buffer *vb);
    int (*start_streaming)(struct vb2_queue *q,                            unsigned int count);
    void (*stop_streaming)(struct vb2_queue *q);
    void (*buf_queue)(struct vb2_buffer *vb);
};
```

以下是结构中每个回调的目的：

+   `queue_setup`：此回调函数由驱动程序的`v4l2_ioctl_ops.vidioc_reqbufs()`方法调用（响应`VIDIOC_REQBUFS`和`VIDIOC_CREATE_BUFS` ioctls），以调整缓冲区计数和大小。此回调的目标是通知 videobuf2-core 需要多少个缓冲区和每个缓冲区的平面，以及每个平面的大小和分配器上下文。换句话说，所选的 vb2 内存分配器调用此方法与驱动程序协商在流媒体期间使用的缓冲区和每个缓冲区的平面数量。`3`是一个很好的选择作为最小缓冲区数量，因为大多数 DMA 引擎至少需要队列中的`2`个缓冲区。此回调的参数定义如下：

- `q`是`vb2_queue`指针。

- `num_buffers`是应用程序请求的缓冲区数量的指针。然后，驱动程序应在此`*num_buffers`字段中设置分配的缓冲区数量。由于此回调在协商过程中可能会被调用两次，因此应检查`queue->num_buffers`以了解在设置此值之前已分配的缓冲区数量。

- `num_planes`包含保存帧所需的不同视频平面的数量。这应该由驱动程序设置。

- `sizes`包含每个平面的大小（以字节为单位）。对于单平面系统，只需设置`size[0]`。

- `alloc_devs`是一个可选的每平面分配器特定设备数组。将其视为分配上下文的指针。

以下是`queue_setup`回调的示例：

```
/* Setup vb_queue minimum buffer requirements */
static int rcar_drif_queue_setup(struct vb2_queue *vq,
                             unsigned int *num_buffers,                             unsigned int *num_planes,
                             unsigned int sizes[],                              struct device *alloc_devs[])
{
    struct rcar_drif_sdr *sdr = vb2_get_drv_priv(vq);
    /* Need at least 16 buffers */
    if (vq->num_buffers + *num_buffers < 16)
        *num_buffers = 16 - vq->num_buffers;
    *num_planes = 1;
    sizes[0] = PAGE_ALIGN(sdr->fmt->buffersize);
    rdrif_dbg(sdr, "num_bufs %d sizes[0] %d\n",
              *num_buffers, sizes[0]);
    return 0;
}
```

+   `buf_init`在为缓冲区分配内存后或在新的`USERPTR`缓冲区排队后会被调用一次。例如，可以用来固定页面，验证连续性，并设置 IOMMU 映射。

+   `buf_prepare`在`VIDIOC_QBUF` ioctl 的执行路径上被调用。它应该准备好缓冲区以排队到 DMA 引擎。缓冲区被准备好，并且用户空间虚拟地址或用户地址被转换为物理地址。

+   `buf_finish`在每个`DQBUF` ioctl 上被调用。例如，可以用于缓存同步和从反弹缓冲区复制回来。

+   `buf_cleanup`在释放内存之前调用。可以用于取消映射内存等。

+   `buf_queue`：videobuf2 核心在调用此回调之前在缓冲区中设置`VB2_BUF_STATE_ACTIVE`标志。但是，它是代表`VIDIOC_QBUF` ioctl 调用的。用户空间逐个排队缓冲区，一个接一个。此外，缓冲区可能会比桥接设备从捕获设备抓取数据到缓冲区的速度更快。与此同时，在发出`VIDIOC_DQBUF`之前可能会多次调用`VIDIOC_QBUF`。建议驱动程序维护一个排队用于 DMA 的缓冲区列表，以便在任何 DMA 完成时，填充的缓冲区被移出列表，同时通过填充其时间戳并将缓冲区添加到 videobuf2 的完成缓冲区列表中，如果需要，则更新 DMA 指针。粗略地说，此回调函数应将缓冲区添加到驱动程序的 DMA 队列中，并在该缓冲区上启动 DMA。与此同时，驱动程序通常会重新实现自己的缓冲区数据结构，建立在通用的`vb2_v4l2_buffer`结构之上，但添加一个列表以解决我们刚才描述的排队问题。以下是这样一个自定义缓冲区数据结构的示例：

```
struct dcmi_buf {
   struct vb2_v4l2_buffer vb;
   dma_addr_t paddr; /* the bus address of this buffer */
   size_t size;
   struct list_head list; /* list entry for tracking    buffers */
};
```

+   `start_streaming`启动了流式传输的 DMA 引擎。在开始流式传输之前，必须首先检查是否已排队了最少数量的缓冲区。如果没有，应返回`-ENOBUFS`，`vb2`框架将在下次缓冲区排队时再次调用此函数，直到有足够的缓冲区可用于实际启动 DMA 引擎。如果支持以下操作，还应在子设备上启用流式传输：`v4l2_subdev_call(subdev, video, s_stream, 1)`。应从缓冲区队列中获取下一帧并在其上启动 DMA。通常，在捕获新帧后会发生中断。处理程序的工作是从内部缓冲区中删除新帧（使用`list_del()`）并将其返回给`vb2`框架（通过`vb2_buffer_done()`），同时更新序列计数字段和时间戳。

+   `stop_streaming`停止所有待处理的 DMA 操作，停止 DMA 引擎，并释放 DMA 通道资源。如果支持以下操作，还应在子设备上禁用流式传输：`v4l2_subdev_call(subdev, video, s_stream, 0)`。如有必要，禁用中断。由于驱动程序维护了排队进行 DMA 的缓冲区列表，因此必须将该列表中排队的所有缓冲区以错误状态返回给 vb2。

#### 初始化和释放 vb2 队列

为了使驱动程序完成队列初始化，应调用`vb2_queue_init()`函数，给定队列作为参数。但是，`vb2_queue`结构应首先由驱动程序分配。此外，驱动程序必须清除其内容并为一些必需的条目设置初始值，然后才能调用此函数。这些必需的值是`q->ops`、`q->mem_ops`、`q->type`和`q->io_modes`。否则，队列初始化将失败，如下所示的`vb2_core_queue_init()`函数将会被调用，并且从`vb2_queue_init()`中检查其返回值：

```
int vb2_core_queue_init(struct vb2_queue *q)
{
    /*
     * Sanity check
     */
    if (WARN_ON(!q) || WARN_ON(!q->ops) ||          WARN_ON(!q->mem_ops) ||
         WARN_ON(!q->type) || WARN_ON(!q->io_modes) ||
         WARN_ON(!q->ops->queue_setup) ||          WARN_ON(!q->ops->buf_queue))
        return -EINVAL;
    INIT_LIST_HEAD(&q->queued_list);
    INIT_LIST_HEAD(&q->done_list);
    spin_lock_init(&q->done_lock);
    mutex_init(&q->mmap_lock);
    init_waitqueue_head(&q->done_wq);
    q->memory = VB2_MEMORY_UNKNOWN;
    if (q->buf_struct_size == 0)
        q->buf_struct_size = sizeof(struct vb2_buffer);
    if (q->bidirectional)
        q->dma_dir = DMA_BIDIRECTIONAL;
    else
        q->dma_dir = q->is_output ? DMA_TO_DEVICE :         DMA_FROM_DEVICE;
    return 0;
}
```

上述摘录显示了内核中`vb2_core_queue_init()`的主体。这个内部 API 是一个纯基本的初始化方法，它只是进行一些合理性检查并初始化基本数据结构（列表、互斥锁和自旋锁）。

# 子设备的概念

在 V4L2 子系统的早期，只有两个主要的数据结构：

+   `struct video_device`：这是`/dev/<type>X`出现的结构。

+   `struct vb2_queue`：这负责缓冲区管理。

在那个时代，这已经足够了，因为嵌入视频桥的 IP 块并不多。如今，SoC 中的图像块嵌入了许多 IP 块，每个 IP 块都通过卸载特定任务来发挥特定作用，例如图像调整、图像转换和视频去隔行功能。为了使用模块化方法来解决这种多样性，引入了子设备的概念。这为硬件的软件建模带来了模块化方法，允许将每个硬件组件抽象为软件块。

采用这种方法，处理管道中的每个 IP 块（除了桥接设备）都被视为一个子设备，甚至包括摄像头传感器本身。桥接视频设备节点采用`/dev/videoX`模式，而子设备则采用`/dev/v4l-subdevX`模式（假设它们在创建节点之前已设置了适当的标志）。

重要说明

为了更好地理解桥接设备和子设备之间的区别，可以将桥接设备视为处理管道中的最终元素，有时负责 DMA 事务。一个例子是 Atmel-`drivers/media/platform/atmel/atmel-isc.c`：`Sensor-->PFE-->WB-->CFA-->CC-->GAM-->CSC-->CBC-->SUB-->RLP-->DMA`。鼓励您查看此驱动程序以了解每个元素的含义。

从编码的角度来看，驱动程序应包括`<media/v4l-subdev.h>`，该文件定义了`struct v4l2_subdev`结构，该结构是用于在内核中实例化子设备的抽象数据结构。此结构定义如下：

```
struct v4l2_subdev {
#if defined(CONFIG_MEDIA_CONTROLLER)
    struct media_entity entity;
#endif
    struct list_head list; 
    struct module *owner;
    bool owner_v4l2_dev;
    u32 flags;
    struct v4l2_device *v4l2_dev;
    const struct v4l2_subdev_ops *ops;
[...]
    struct v4l2_ctrl_handler *ctrl_handler;
    char name[V4L2_SUBDEV_NAME_SIZE];
    u32 grp_id; void *dev_priv;
    void *host_priv;
    struct video_device *devnode;
    struct device *dev;
    struct fwnode_handle *fwnode;
    struct device_node *of_node;
    struct list_head async_list;
    struct v4l2_async_subdev *asd;
    struct v4l2_async_notifier *notifier;
    struct v4l2_async_notifier *subdev_notifier;
    struct v4l2_subdev_platform_data *pdata;
};
```

此结构的`entity`字段将在下一章*第八章**，与 V4L2 异步和媒体控制器框架集成*中讨论。与此同时，我们不感兴趣的字段已被删除。

但是，结构中的其他字段定义如下：

+   `list`是`list_head`类型，并由核心用于将当前子设备插入`v4l2_device`维护的子设备列表中。

+   `owner`由核心设置，表示拥有此结构的模块。

+   `flags`表示驱动程序可以设置的子设备标志，可以具有以下值：

- 如果此子设备实际上是 I2C 设备，则应设置`V4L2_SUBDEV_FL_IS_I2C`标志。

- 如果此子设备是 SPI 设备，则应设置`V4L2_SUBDEV_FL_IS_SPI`。

- 如果子设备需要设备节点（著名的`/dev/v4l-subdevX`条目），则应设置`V4L2_SUBDEV_FL_HAS_DEVNODE`。使用此标志的 API 是`v4l2_device_register_subdev_nodes()`，稍后将讨论并由桥接调用以创建子设备节点条目。

- `V4L2_SUBDEV_FL_HAS_EVENTS`表示此子设备生成事件。

+   `v4l2_dev`由核心在子设备注册时设置，并指向此子设备所属的`struct 4l2_device`的指针。

+   `ops`是可选的。这是指向`struct v4l2_subdev_ops`的指针，由驱动程序设置以提供核心可以依赖的此子设备的回调。

+   `ctrl_handler`是指向`struct v4l2_ctrl_handler`的指针。它表示此子设备提供的控件列表，我们将在*V4L2 控件基础设施*部分中看到。

+   `name`是子设备的唯一名称。在子设备初始化后，驱动程序应设置它。对于 I2C 变体的初始化，核心分配的默认名称是`("%s %d-%04x", driver->name, i2c_adapter_id(client->adapter), client->addr)`。在包括**媒体控制器**支持时，此名称用作媒体实体名称。

+   `grp_id`是驱动程序特定的，在异步模式下由核心提供，并用于对类似的子设备进行分组。

+   `dev_priv`是设备的私有数据指针（如果有的话）。

+   `host_priv`是指向设备的私有数据的指针，用于连接子设备的设备。

+   `devnode`是此子设备的设备节点，由核心在调用`v4l2_device_register_subdev_nodes()`时设置，不要与基于相同结构构建的桥接设备混淆。您应该记住，每个`v4l2`元素（无论是子设备还是桥接）都是视频设备。

+   `dev`是指向物理设备的指针（如果有的话）。驱动程序可以使用`void` `v4l2_set_subdevdata(struct v4l2_subdev *sd, void *p)`设置此值，或者可以使用`void *v4l2_get_subdevdata(const struct v4l2_subdev *sd)`获取它。

+   `fwnode`是此子设备的固件节点对象句柄。在较旧的内核版本中，此成员曾经是`struct device_node *of_node`，并指向`struct fwnode_handle`，因为它允许根据在平台上使用的设备树节点/ACPI 设备进行切换。换句话说，它是`dev->of_node->fwnode`或`dev->fwnode`，以非`NULL`的方式。

`async_list`、`asd`、`subdev_notifier`和`notifier`元素是 v4l2-async 框架的一部分，我们将在下一节中看到。但是，这里提供了这些元素的简要描述：

+   `async_list`：当与异步核心注册时，此成员由核心用于将此子设备链接到全局`subdev_list`（这是一个孤立子设备的列表，不属于任何通知程序，这意味着此子设备在其父级桥之前注册）或其父级桥的`notifier->done`列表。我们将在下一章中详细讨论这一点，*第八章**，与 V4L2 异步和媒体控制器框架集成*。

+   `asd`：此字段是`struct v4l2_async_subdev`类型，并在异步核心中抽象了这个子设备。

+   `subdev_notifier`：这是由此子设备隐式注册的通知程序，以防需要通知其他子设备的探测。它通常用于涉及多个子设备的流水线的系统，其中子设备 N 需要被通知子设备 N-1 的探测。

+   `notifier`：这是由异步核心设置的，并对应于其底层的`.asd`异步子设备匹配的通知程序。

+   `pdata`：这是子设备平台数据的常见部分。

## 子设备初始化

每个子设备驱动程序必须有一个`struct v4l2_subdev`结构，可以是独立的，也可以嵌入到更大和特定于设备的结构中。推荐第二种情况，因为它允许跟踪设备状态。以下是典型设备特定结构的示例：

```
struct mychip_struct {
    struct v4l2_subdev sd;
[...]
    /* device speific fields*/
[...]
};
```

在被访问之前，V4L2 子设备需要使用`v4l2_subdev_init()` API 进行初始化。然而，当涉及到具有基于 I2C 或 SPI 的控制接口（通常是摄像头传感器）的子设备时，内核提供了`v4l2_spi_subdev_init()`和`v4l2_i2c_subdev_init()`变体：

```
void v4l2_subdev_init(struct v4l2_subdev *sd,
                       const struct v4l2_subdev_ops *ops)
void v4l2_i2c_subdev_init(struct v4l2_subdev *sd,
                       struct i2c_client *client,
                       const struct v4l2_subdev_ops *ops)
void v4l2_spi_subdev_init(struct v4l2_subdev *sd,
                          struct spi_device *spi,
                          const struct v4l2_subdev_ops *ops)
```

所有这些 API 都将`struct v4l2_subdev`结构的指针作为第一个参数。使用我们的设备特定数据结构注册我们的子设备将如下所示：

```
v4l2_i2c_subdev_init(&mychip_struct->sd, client, subdev_ops);
/*or*/
v4l2_subdev_init(&mychip_struct->sd, subdev_ops);
```

`spi`/`i2c`变体包装了`v4l2_subdev_init()`函数。此外，它们需要作为第二个参数的底层低级、特定于总线的结构。此外，这些特定于总线的变体将存储子设备对象（作为第一个参数给出）作为低级、特定于总线的设备数据，反之亦然，通过将低级、特定于总线的结构存储为子设备的私有数据。这样，`i2c_client`（或`spi_device`）和`v4l2_subdev`相互指向，这意味着通过拥有指向 I2C 客户端的指针，例如，您可以调用`i2c_set_clientdata()`（例如`struct v4l2_subdev *sd = i2c_get_clientdata(client);`）来获取指向我们内部子设备对象的指针，并使用`container_of`宏（例如`struct mychip_struct *foo = container_of(sd, struct mychip_struct, sd);`）来获取指向芯片特定结构的指针。另一方面，拥有指向子设备对象的指针，您可以使用`v4l2_get_subdevdata()`来获取底层特定于总线的结构。

最后但并非最不重要的是，这些特定于总线的变体将损坏子设备名称，就像在介绍`struct v4l2_subdev`数据结构时所解释的那样。`v4l2_i2c_subdev_init()`的摘录可以更好地理解这一点：

```
void v4l2_i2c_subdev_init(struct v4l2_subdev *sd,
                          struct i2c_client *client,
                          const struct v4l2_subdev_ops *ops)
{
   v4l2_subdev_init(sd, ops);
   sd->flags |= V4L2_SUBDEV_FL_IS_I2C;
   /* the owner is the same as the i2c_client's driver owner */
   sd->owner = client->dev.driver->owner;
   sd->dev = &client->dev;
   /* i2c_client and v4l2_subdev point to one another */     
   v4l2_set_subdevdata(sd, client);
   i2c_set_clientdata(client, sd);
   /* initialize name */
   snprintf(sd->name, sizeof(sd->name),
            "%s %d-%04x", client->dev.driver->name,    
            i2c_adapter_id(client->adapter), client->addr);
}
```

在前面三个初始化 API 中，`ops`是最后一个参数，是指向表示子设备公开/支持的操作的`struct v4l2_subdev_ops`的指针。然而，让我们在下一节中讨论这个问题。

## 子设备操作

子设备是以某种方式连接到主桥设备的设备。在整个媒体设备中，每个 IP（子设备）都有其自己的功能集。这些功能必须通过内核开发人员为常用功能定义的回调来向核心公开。这就是`struct v4l2_subdev_ops`的目的。

然而，一些子设备可以执行如此多不同和不相关的事情，以至于甚至 `struct v4l2_subdev_ops` 已经被分成小的和分类的一致的子结构操作，每个子结构操作都收集相关的功能，以便 `struct v4l2_subdev_ops` 成为顶级操作结构，描述如下：

```
struct v4l2_subdev_ops {
    const struct v4l2_subdev_core_ops          *core;
    const struct v4l2_subdev_tuner_ops         *tuner;
    const struct v4l2_subdev_audio_ops         *audio;
    const struct v4l2_subdev_video_ops         *video;
    const struct v4l2_subdev_vbi_ops           *vbi;
    const struct v4l2_subdev_ir_ops            *ir;
    const struct v4l2_subdev_sensor_ops        *sensor;
    const struct v4l2_subdev_pad_ops           *pad;
};
```

重要提示

操作应该只为用户空间公开的子设备提供，通过底层字符设备文件节点。注册时，该设备文件节点将具有与前面讨论的相同的文件操作，即 `v4l2_fops`。然而，正如我们之前所看到的，这些低级操作只是包装（处理）`video_device->fops`。因此，为了达到 `v4l2_subdev_ops`，核心使用 `subdev->video_device->fops` 作为中间，并在初始化时分配另一个文件操作（`subdev->vdev->fops = &v4l2_subdev_fops;`），它将包装并调用真正的子设备操作。这里的调用链是 `v4l2_fops ==> v4l2_subdev_fops ==> our_custom_subdev_ops`。

您可以看到前面的顶级操作结构由指向类别操作结构的指针组成，如下所示：

+   `v4l2_subdev_core_ops` 类型的 `core`：这是核心操作类别，提供通用的回调，比如日志记录和调试。它还允许提供额外和自定义的 ioctls（特别是当 ioctl 不适用于任何类别时非常有用）。

+   `v4l2_subdev_video_ops` 类型的 `video`：`.s_stream` 在流媒体开始时被调用。它根据所选择的帧大小和格式向摄像头的寄存器写入不同的配置值。

+   `v4l2_subdev_pad_ops` 类型的 `pad`：对于支持多个帧大小和图像采样格式的摄像头，这些操作允许用户从可用选项中进行选择。

+   `tuner`、`audio`、`vbi` 和 `ir` 超出了本书的范围。

+   `v4l2_subdev_sensor_ops` 类型的 `sensor`：这涵盖了摄像头传感器操作，通常用于已知有错误的传感器，需要跳过一些帧或行，因为它们已损坏。

每个类别结构中的每个回调对应一个 ioctl。路由实际上是由 `subdev_do_ioctl()` 在低级别执行的，该函数在 `drivers/media/v4l2-core/v4l2-subdev.c` 中定义，并间接地由 `subdev_ioctl()` 调用，对应于 `v4l2_subdev_fops.unlocked_ioctl`。真正的调用链应该是 `v4l2_fops ==> v4l2_subdev_fops.unlocked_ioctl ==> our_custom_subdev_ops`。

这个顶级 `struct v4l2_subdev_ops` 结构的性质只是确认了 V4L2 可能支持的设备范围有多广。对于子设备驱动程序不感兴趣的操作类别可以保持 `NULL`。还要注意，`.core` 操作对所有子设备都是通用的。这并不意味着它是强制性的；它只是意味着任何类别的子设备驱动程序都可以实现 `.core` 操作，因为它的回调是与类别无关的。

### struct v4l2_subdev_core_ops

这个结构实现了通用的回调，并具有以下定义：

```
struct v4l2_subdev_core_ops {
    int (*log_status)(struct v4l2_subdev *sd);
    int (*load_fw)(struct v4l2_subdev *sd);
    long (*ioctl)(struct v4l2_subdev *sd, unsigned int cmd,
                   void *arg);
[...]
#ifdef CONFIG_COMPAT
    long (*compat_ioctl32)(struct v4l2_subdev *sd,                            unsigned int cmd, 
                           unsigned long arg);
#endif
#ifdef CONFIG_VIDEO_ADV_DEBUG
   int (*g_register)(struct v4l2_subdev *sd,
                     struct v4l2_dbg_register *reg);
   int (*s_register)(struct v4l2_subdev *sd,
                     const struct v4l2_dbg_register *reg);
#endif
   int (*s_power)(struct v4l2_subdev *sd, int on);
   int (*interrupt_service_routine)(struct v4l2_subdev *sd,
                                    u32 status,                                     bool *handled);
   int (*subscribe_event)(struct v4l2_subdev *sd,                           struct v4l2_fh *fh,
                          struct v4l2_event_subscription *sub);
   int (*unsubscribe_event)(struct v4l2_subdev *sd,
                          struct v4l2_fh *fh,                           struct v4l2_event_subscription *sub);
};
```

在前面的结构中，我们已经删除了对我们不感兴趣的字段。剩下的字段定义如下：

+   `.log_status` 用于记录目的。您应该使用 `v4l2_info()` 宏来实现这一点。

+   `.s_power` 将子设备（例如摄像头）置于省电模式（`on==0`）或正常操作模式（`on==1`）。

+   `.load_fw` 操作必须被调用以加载子设备的固件。

+   如果子设备提供额外的 ioctl 命令，应该定义 `.ioctl`。

+   `.g_register` 和 `.s_register` 仅用于高级调试，需要设置内核配置选项 `CONFIG_VIDEO_ADV_DEBUG`。这些操作允许读取和写入硬件寄存器，以响应 `VIDIOC_DBG_G_REGISTER` 和 `VIDIOC_DBG_S_REGISTER` ioctls。`reg` 参数（类型为 `v4l2_dbg_register`，在 `include/uapi/linux/videodev2.h` 中定义）由应用程序填充和提供。

+   `.interrupt_service_routine`由桥接器在其 IRQ 处理程序中调用（应使用`v4l2_subdev_call`），当由于此子设备而引发中断状态时，以便子设备处理详细信息。`handled`是桥接驱动程序提供的输出参数，但必须由子设备驱动程序填充，以便通知（作为*true 或 false*）其处理结果。我们处于 IRQ 上下文中，因此不能休眠。位于 I2C/SPI 总线后面的子设备可能应该在线程化的上下文中安排其工作。

+   `.subscribe_event`和`.unsubscribe_event`用于订阅或取消订阅控制更改事件。请查看其他实现此功能的 V4L2 驱动程序，以了解如何实现您的驱动程序。

### struct v4l2_subdev_video_ops 或 struct v4l2_subdev_pad_ops

人们经常需要决定是否实现`struct v4l2_subdev_video_ops`或`struct v4l2_subdev_pad_ops`，因为这两个结构中的一些回调是多余的。问题是，当 V4L2 设备以视频模式打开时，`struct v4l2_subdev_video_ops`结构的回调被使用，其中包括电视、摄像头传感器和帧缓冲区。到目前为止，一切顺利。`struct v4l2_subdev_pad_ops`的概念也不需要。然而，媒体控制器框架通过实体对象（稍后我们将看到）抽象了子设备，通过 PAD 连接到其他元素。在这种情况下，使用与 PAD 相关的功能而不是与子设备相关的功能是有意义的，因此，使用`struct v4l2_subdev_pad_ops`而不是`struct v4l2_subdev_video_ops`。

由于我们还没有介绍媒体框架，所以我们只对`struct v4l2_subdev_video_ops`结构感兴趣，其定义如下：

```
struct v4l2_subdev_video_ops {
    int (*querystd)(struct v4l2_subdev *sd, v4l2_std_id *std);
[...]
    int (*s_stream)(struct v4l2_subdev *sd, int enable);
    int (*g_frame_interval)(struct v4l2_subdev *sd,
                  struct v4l2_subdev_frame_interval *interval);
    int (*s_frame_interval)(struct v4l2_subdev *sd,
                  struct v4l2_subdev_frame_interval *interval);
[...]
};
```

在上述摘录中，为了便于阅读，我删除了与电视和视频输出相关的回调，以及与摄像头设备无关的回调，这对我们也没有什么用。对于常用的回调，它们的定义如下：

+   `querystd`：这是`VIDIOC_QUERYSTD()`ioctl 处理程序代码的回调。

+   `s_stream`：用于通知驱动程序视频流将开始或已停止，取决于`enable`参数的值。

+   `g_frame_interval`：这是`VIDIOC_SUBDEV_G_FRAME_INTERVAL()`ioctl 处理程序代码的回调。

+   `s_frame_interval`：这是`VIDIOC_SUBDEV_S_FRAME_INTERVAL()`ioctl 处理程序代码的回调。

### struct v4l2_subdev_sensor_ops

当传感器开始流式传输时，有些传感器会产生初始垃圾帧。这样的传感器可能需要一些时间来确保其某些属性的稳定性。该结构使得可以通知核心跳过多少帧以避免垃圾。此外，一些传感器可能始终在顶部产生一定数量的损坏行的图像，或者在这些行中嵌入它们的元数据。在这两种情况下，它们产生的帧始终是损坏的。该结构还允许我们指定在抓取每帧之前要跳过的行数。

以下是`v4l2_subdev_sensor_ops`结构的定义：

```
struct v4l2_subdev_sensor_ops {
    int (*g_skip_top_lines)(struct v4l2_subdev *sd,                             u32 *lines);
    int (*g_skip_frames)(struct v4l2_subdev *sd, u32 *frames);
};
```

`g_skip_top_lines`用于指定传感器每幅图像中要跳过的行数，而`g_skip_frames`允许我们指定要跳过的初始帧数，以避免垃圾，如以下示例所示：

```
#define OV5670_NUM_OF_SKIP_FRAMES	2
static int ov5670_get_skip_frames(struct v4l2_subdev *sd,                                   u32 *frames)
{
    *frames = OV5670_NUM_OF_SKIP_FRAMES;
    return 0;
}
```

`lines`和`frames`参数是输出参数。每个回调应返回`0`。

### 调用子设备操作

最后，如果提供了`subdev`回调，则打算调用它们。也就是说，调用 ops 回调就像直接调用它一样简单，如下所示：

```
err = subdev->ops->video->s_stream(subdev, 1);
```

然而，有一种更方便和更安全的方法可以实现这一点，即使用`v4l2_subdev_call()`宏：

```
err = v4l2_subdev_call(subdev, video, s_stream, 1);
```

在`include/media/v4l2-subdev.h`中定义的宏将执行以下操作：

+   它将首先检查子设备是否为`NULL`，否则返回`-ENODEV`。

+   如果类别(`subdev->video`在我们的示例中)或回调本身(`subdev->video->s_stream`在我们的示例中)为`NULL`，则它将返回`-ENOIOCTLCMD`，或者它将返回`subdev->ops->video->s_stream`操作的实际结果。

还可以调用所有或部分子设备：

```
v4l2_device_call_all(dev, 0, core, g_chip_ident, &chip);
```

不支持此回调的任何子设备都将被跳过，错误结果将被忽略。如果要检查错误，请使用以下命令：

```
err = v4l2_device_call_until_err(dev, 0, core,                                  g_chip_ident, &chip);
```

除了`-ENOIOCTLCMD`之外的任何错误都将以该错误退出循环。如果没有错误(除了`- ENOIOCTLCMD`)发生，则返回`0`。

## 传统子设备(取消)注册

有两种方式可以将子设备注册到桥接设备，取决于媒体设备的性质：

1.  **同步模式**：这是传统的方法。在这种模式下，桥接驱动程序负责注册子设备。子设备驱动程序要么是从桥接驱动程序中实现的，要么您必须找到一种方法让桥接驱动程序获取其负责的子设备的句柄。这通常是通过平台数据实现的，或者通过桥接驱动程序公开一组 API，这些 API 将被子设备驱动程序使用，从而允许桥接驱动程序了解这些子设备(例如通过在私有内部列表中跟踪它们)。使用这种方法，桥接驱动程序必须了解连接到它的子设备，并确切地知道何时注册它们。这通常适用于内部子设备，例如 SoC 内的视频数据处理单元或复杂的 PCI(e)板，或者 USB 摄像头中的摄像头传感器或连接到 SoC。

1.  异步模式：这是关于子设备信息独立于桥接设备向系统提供的情况，这通常是基于设备树的系统的情况。这将在下一章中讨论，*第八章*，*与 V4L2 异步和媒体控制器框架集成*。

但是，为了桥接驱动程序注册子设备，必须调用`v4l2_device_register_subdev()`，而必须调用`v4l2_device_unregister_subdev()`来注销此子设备。同时，在将子设备注册到核心后，可能需要为具有设置`V4L2_SUBDEV_FL_HAS_DEVNODE`标志的子设备创建它们各自的字符文件节点`/dev/v4l-subdevX`。您可以使用`v4l2_device_register_subdev_nodes()`来实现这一点：

```
int v4l2_device_register_subdev(struct v4l2_device *v4l2_dev,
                                struct v4l2_subdev *sd)
void v4l2_device_unregister_subdev(struct v4l2_subdev *sd)
int v4l2_device_register_subdev_nodes(struct                                       v4l2_device *v4l2_dev)
```

`v4l2_device_register_subdev()`将`sd`插入`v4l2_dev->subdevs`，这是由 V4L2 设备维护的子设备列表。如果`subdev`模块在注册之前消失，这可能会失败。成功调用此函数后，`subdev->v4l2_dev`字段指向`v4l2_device`。此函数在成功时返回`0`，或者`v4l2_device_unregister_subdev()`将从列表中取出`sd`。然后，`v4l2_device_register_subdev_nodes()`遍历`v4l2_dev->subdevs`，为每个具有设置`V4L2_SUBDEV_FL_HAS_DEVNODE`标志的子设备创建一个特殊的字符文件节点(`/dev/v4l-subdevX`)。

重要提示

`/dev/v4l-subdevX`设备节点允许直接控制子设备的高级和硬件特定功能。

现在我们已经了解了子设备的初始化、操作和注册，让我们在下一节中看看 V4L2 控件。

# V4L2 控件基础设施

一些设备具有可由用户设置的控件，以修改一些定义的属性。其中一些控件可能支持预定义值列表、默认值、调整等。问题是，不同的设备可能提供具有不同值的不同控件。此外，虽然其中一些控件是标准的，但其他可能是特定于供应商的。控件框架的主要目的是向用户呈现控件，而不假设其目的。在本节中，我们只讨论标准控件。

控件框架依赖于两个主要对象，都在`include/media/v4l2- ctrls.h`中定义，就像该框架提供的其他数据结构和 API 一样。第一个是`struct v4l2_ctrl`。这个结构描述了控件的属性，并跟踪控件的值。第二个和最后一个是`struct v4l2_ctrl_handler`，它跟踪所有的控件。它们的详细定义在这里呈现：

```
struct v4l2_ctrl_handler {
    [...]
    struct mutex *lock;
    struct list_head ctrls;
    v4l2_ctrl_notify_fnc notify;
    void *notify_priv;
    [...]
};
```

在`struct v4l2_ctrl_handler`的前述定义摘录中，`ctrls`表示此处理程序拥有的控件列表。`notify`是一个通知回调，每当控件更改值时都会被调用。这个回调在持有处理程序的`lock`时被调用。最后，`notify_priv`是作为参数给出的上下文数据。接下来是`struct v4l2_ctrl`，定义如下：

```
struct v4l2_ctrl {
    struct list_head node;
    struct v4l2_ctrl_handler *handler;
    unsigned int is_private:1;
    [...]
    const struct v4l2_ctrl_ops *ops;
    u32 id;
    const char *name;
    enum v4l2_ctrl_type type;
    s64 minimum, maximum, default_value;
    u64 step;
    unsigned long flags; [...]
}
```

这个结构代表了控件本身，具有重要的成员。这些定义如下：

+   `node`用于将控件插入处理程序的控件列表中。

+   `handler`是此控件所属的处理程序。

+   `ops`是`struct v4l2_ctrl_ops`类型，并表示此控件的获取/设置操作。

+   `id` 是此控件的 ID。

+   `name` 是控件的名称。

+   `minimum`和`maximum`分别是控件接受的最小值和最大值。

+   `default_value`是控件的默认值。

+   `step`是非菜单控件的递增/递减步长。

+   `flags`涵盖了控件的标志。虽然整个标志列表在`include/uapi/linux/videodev2.h`中定义，但一些常用的标志如下：

– `V4L2_CTRL_FLAG_DISABLED`，表示控件被禁用

– `V4L2_CTRL_FLAG_READ_ONLY`，用于只读控件

– `V4L2_CTRL_FLAG_WRITE_ONLY`，用于只写控件

– `V4L2_CTRL_FLAG_VOLATILE`，用于易失性控件

+   `is_private`，如果设置，将阻止此控件被添加到任何其他处理程序中。它使得此控件对最初添加它的处理程序私有。这可以用来防止将`subdev`控件可用于 V4L2 驱动程序控件。

重要提示

`enum`通常像一种菜单，因此称为*菜单控件*。

V4L2 控件由唯一的 ID 标识。它们以`V4L2_CID_`为前缀，并且都在`include/uapi/linux/v4l2-controls.h`中可用。视频捕获设备支持的常见标准控件如下（以下列表不是详尽无遗的）：

```
#define V4L2_CID_BRIGHTNESS        (V4L2_CID_BASE+0)
#define V4L2_CID_CONTRAST          (V4L2_CID_BASE+1)
#define V4L2_CID_SATURATION        (V4L2_CID_BASE+2)
#define V4L2_CID_HUE	(V4L2_CID_BASE+3)
#define V4L2_CID_AUTO_WHITE_BALANCE      (V4L2_CID_BASE+12)
#define V4L2_CID_DO_WHITE_BALANCE  (V4L2_CID_BASE+13)
#define V4L2_CID_RED_BALANCE (V4L2_CID_BASE+14)
#define V4L2_CID_BLUE_BALANCE      (V4L2_CID_BASE+15)
#define V4L2_CID_GAMMA       (V4L2_CID_BASE+16)
#define V4L2_CID_EXPOSURE    (V4L2_CID_BASE+17)
#define V4L2_CID_AUTOGAIN    (V4L2_CID_BASE+18)
#define V4L2_CID_GAIN  (V4L2_CID_BASE+19)
#define V4L2_CID_HFLIP (V4L2_CID_BASE+20)
#define V4L2_CID_VFLIP (V4L2_CID_BASE+21)
[...]
#define V4L2_CID_VBLANK  (V4L2_CID_IMAGE_SOURCE_CLASS_BASE + 1) #define V4L2_CID_HBLANK  (V4L2_CID_IMAGE_SOURCE_CLASS_BASE + 2) #define V4L2_CID_LINK_FREQ (V4L2_CID_IMAGE_PROC_CLASS_BASE + 1)
```

前面的列表只包括标准控件。要支持自定义控件，你应该根据控件的基类描述符添加其 ID，并确保 ID 不重复。要向驱动程序添加控件支持，控件处理程序应首先使用`v4l2_ctrl_handler_init()`宏进行初始化。这个宏接受要初始化的处理程序以及此处理程序可以引用的控件数量，如下原型所示：

```
v4l2_ctrl_handler_init(hdl, nr_of_controls_hint)
```

完成控件处理程序后，你可以调用`v4l2_ctrl_handler_free()`释放此控件处理程序的资源。一旦控件处理程序被初始化，就可以创建控件并将其添加到其中。对于标准的 V4L2 控件，你可以使用`v4l2_ctrl_new_std()`来分配和初始化新的控件：

```
struct v4l2_ctrl *v4l2_ctrl_new_std(                               struct v4l2_ctrl_handler *hdl,
                               const struct v4l2_ctrl_ops *ops,                               u32 id, s64 min, s64 max,                                u64 step, s64 def);
```

这个函数在大多数字段上都是基于控件 ID 的。然而对于自定义控件（这里不讨论），你应该使用`v4l2_ctrl_new_custom()`辅助函数。在前面的原型中，以下元素被定义如下：

+   `hdl`表示先前初始化的控件处理程序。

+   `ops`是`struct v4l2_ctrl_ops`类型，并表示控件操作。

+   `id`是控件 ID，定义为`V4L2_CID_*`。

+   `min`是此控件可以接受的最小值。根据控件 ID，这个值可能会被核心修改。

+   `max`是此控件可以接受的最大值。根据控件 ID，这个值可能会被核心修改。

+   `step` 是控件的步进值。

+   `def` 是控件的默认值。

控件的目的是设置/获取。这是前面的 ops 参数的目的。这意味着在初始化控件之前，您应该首先定义将在设置/获取此控件的值时调用的操作。也就是说，整个控件列表可以由相同的操作处理。在这种情况下，操作回调将必须使用 `switch ... case` 来处理不同的控件。

正如我们之前所看到的，控件操作是 `struct v4l2_ctrl_ops` 类型，并被定义如下：

```
struct v4l2_ctrl_ops {
    int (*g_volatile_ctrl)(struct v4l2_ctrl *ctrl);
    int (*try_ctrl)(struct v4l2_ctrl *ctrl);
    int (*s_ctrl)(struct v4l2_ctrl *ctrl);
};
```

前面的结构由三个回调组成，每个都有特定的目的：

+   `g_volatile_ctrl` 获取给定控件的新值。只有在对易失性控件（由硬件自身更改，并且大部分时间是只读的，例如信号强度或自动增益）提供此回调才有意义。

+   `try_ctrl`，如果设置，将被调用来测试要应用的控件值是否有效。只有在通常的最小/最大/步长检查不足以时，提供此回调才有意义。

+   `s_ctrl` 被调用来设置控件的值。

可选地，您可以在控件处理程序上调用 `v4l2_ctrl_handler_setup()` 来设置此处理程序的控件为它们的默认值。这有助于确保硬件和驱动程序的内部数据结构保持同步：

```
int v4l2_ctrl_handler_setup(struct v4l2_ctrl_handler *hdl);
```

此函数遍历给定处理程序中的所有控件，并使用每个控件的默认值调用 `s_ctrl` 回调。

总结一下我们在整个 V4L2 控件接口部分所看到的内容，现在让我们更详细地研究一下 `OV7740` 摄像头传感器的驱动程序（位于 `drivers/media/i2c/ov7740.c` 中），特别是处理 V4L2 控件的部分。

首先，我们有控件 `ops->sg_ctrl` 回调的实现：

```
static int ov7740_get_volatile_ctrl(struct v4l2_ctrl *ctrl)
{
    struct ov7740 *ov7740 = container_of(ctrl->handler,
    struct ov7740, ctrl_handler);
    int ret;
    switch (ctrl->id) {
    case V4L2_CID_AUTOGAIN:
        ret = ov7740_get_gain(ov7740, ctrl);
        break;
    default:
        ret = -EINVAL;
        break;
    }
    return ret;
}
```

前面的回调只涉及 `V4L2_CID_AUTOGAIN` 的控件 ID。这是有意义的，因为增益值可能在 *自动* 模式下由硬件更改。此驱动程序实现了 `ops->s_ctrl` 控件如下：

```
static int ov7740_set_ctrl(struct v4l2_ctrl *ctrl)
{
    struct ov7740 *ov7740 =
             container_of(ctrl->handler, struct ov7740,                           ctrl_handler);
    struct i2c_client *client =     v4l2_get_subdevdata(&ov7740->subdev); 
    struct regmap *regmap = ov7740->regmap;
    int ret;
    u8 val = 0;
[...]
    switch (ctrl->id) {
    case V4L2_CID_AUTO_WHITE_BALANCE:
        ret = ov7740_set_white_balance(ov7740, ctrl->val); break;
    case V4L2_CID_SATURATION:
        ret = ov7740_set_saturation(regmap, ctrl->val); break;
    case V4L2_CID_BRIGHTNESS:
        ret = ov7740_set_brightness(regmap, ctrl->val); break;
    case V4L2_CID_CONTRAST:
        ret = ov7740_set_contrast(regmap, ctrl->val); break;
    case V4L2_CID_VFLIP:
        ret = regmap_update_bits(regmap, REG_REG0C,
                                 REG0C_IMG_FLIP, val); break;
    case V4L2_CID_HFLIP:
        val = ctrl->val ? REG0C_IMG_MIRROR : 0x00;
        ret = regmap_update_bits(regmap, REG_REG0C,
                                 REG0C_IMG_MIRROR, val);
        break;
    case V4L2_CID_AUTOGAIN:
        if (!ctrl->val)
            return ov7740_set_gain(regmap, ov7740->gain->val);
        ret = ov7740_set_autogain(regmap, ctrl->val); break;
    case V4L2_CID_EXPOSURE_AUTO:
        if (ctrl->val == V4L2_EXPOSURE_MANUAL)
        return ov7740_set_exp(regmap, ov7740->exposure->val);
        ret = ov7740_set_autoexp(regmap, ctrl->val); break;
    default:
        ret = -EINVAL; break;
    }
[...]
    return ret;
}
```

前面的代码块还展示了使用 `V4L2_CID_EXPOSURE_AUTO` 控件作为示例来实现菜单控件有多么容易，其可能的值在 `enum v4l2_exposure_auto_type` 中被枚举。最后，将用于控件创建的控件操作结构被定义如下：

```
static const struct v4l2_ctrl_ops ov7740_ctrl_ops = {
    .g_volatile_ctrl = ov7740_get_volatile_ctrl,
    .s_ctrl = ov7740_set_ctrl,
};
```

一旦定义，这个控件操作可以用来初始化控件。以下是 `ov7740_init_controls()` 方法（在 `probe()` 函数中调用）的摘录，为了可读性的目的而被修改和缩小：

```
static int ov7740_init_controls(struct ov7740 *ov7740)
{
[...]
    struct v4l2_ctrl *auto_wb;
    struct v4l2_ctrl *gain;
    struct v4l2_ctrl *vflip;
    struct v4l2_ctrl *auto_exposure;
    struct v4l2_ctrl_handler *ctrl_hdlr
    v4l2_ctrl_handler_init(ctrl_hdlr, 12);
    auto_wb = v4l2_ctrl_new_std(ctrl_hdlr, &ov7740_ctrl_ops,
                                V4L2_CID_AUTO_WHITE_BALANCE,                                 0, 1, 1, 1);
    vflip = v4l2_ctrl_new_std(ctrl_hdlr, &ov7740_ctrl_ops,
                              V4L2_CID_VFLIP, 0, 1, 1, 0);
    gain = v4l2_ctrl_new_std(ctrl_hdlr, &ov7740_ctrl_ops,
                             V4L2_CID_GAIN, 0, 1023, 1, 500);
    /* let's mark this control as volatile*/
    gain->flags |= V4L2_CTRL_FLAG_VOLATILE;
    contrast = v4l2_ctrl_new_std(ctrl_hdlr, &ov7740_ctrl_ops,
                                 V4L2_CID_CONTRAST, 0, 127,                                  1, 0x20);
    ov7740->auto_exposure =
                   v4l2_ctrl_new_std_menu(ctrl_hdlr,                                        &ov7740_ctrl_ops,
                                       V4L2_CID_EXPOSURE_AUTO,                                        V4L2_EXPOSURE_MANUAL,
                                       0, V4L2_EXPOSURE_AUTO);
[...]
    ov7740->subdev.ctrl_handler = ctrl_hdlr;
    return 0;
}
```

您可以看到控件处理程序被分配给子设备在前面函数的返回路径上。最后，在代码的某个地方（ov7740 的驱动程序在子设备的 `v4l2_subdev_video_ops.s_stream` 回调中执行此操作），您应该将所有控件设置为它们的默认值：

```
ret = v4l2_ctrl_handler_setup(ctrl_hdlr);
if (ret) {
    dev_err(&client->dev, "%s control init failed (%d)\n",
             __func__, ret);
   goto error;
}
```

有关 V4L2 控件的更多信息，请访问[`www.kernel.org/doc/html/v4.19/media/kapi/v4l2-controls.html`](https://www.kernel.org/doc/html/v4.19/media/kapi/v4l2-controls.html)。

## 关于控件继承的说明

子设备驱动程序通常实现了桥接的 V4L2 驱动程序已经实现的控件。

当在 `v4l2_subdev` 和 `v4l2_device` 上调用 `v4l2_device_register_subdev()` 并设置两者的 `ctrl_handler` 字段时，那么子设备的控件将被添加（通过 `v4l2_ctrl_add_handler()` 辅助函数，该函数将给定处理程序的控件添加到另一个处理程序中）到 `v4l2_device` 的控件中。已经由 `v4l2_device` 实现的子设备控件将被跳过。这意味着 V4L2 驱动程序可以始终覆盖 `subdev` 控件。

也就是说，控制可能对给定的子设备执行低级别的硬件特定操作，而子设备驱动程序可能不希望此控制对 V4L2 驱动程序可用（因此不会添加到其控制处理程序）。在这种情况下，子设备驱动程序必须将控件的`is_private`成员设置为`1`（或`true`）。这将使控制对子设备私有。

重要说明

即使子设备控件被添加到 V4L2 设备，它们仍然可以通过控制设备节点访问。

# 总结

在本章中，我们讨论了 V4L2 桥接设备驱动程序的开发，以及子设备的概念。我们了解了 V4L2 的架构，并且现在熟悉了它的数据结构。我们学习了 videobuf2 API，并且现在能够编写平台桥接设备驱动程序。此外，我们应该能够实现子设备操作，并利用 videobuf2 核心。

本章可以被视为一个大局的第一部分，因为下一章仍然涉及 V4L2，但我们将处理异步核心和与媒体控制器框架的集成。
