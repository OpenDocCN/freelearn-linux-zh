# 第十三章：Linux 设备模型

直到 2.5 版本，内核没有描述和管理对象的方法，代码的可重用性也不像现在这样增强。换句话说，没有设备拓扑结构，也没有组织。没有关于子系统关系或系统如何组合的信息。然后**Linux 设备模型**（**LDM**）出现了，引入了：

+   类的概念，用于将相同类型的设备或公开相同功能的设备（例如，鼠标和键盘都是输入设备）分组。

+   通过名为`sysfs`的虚拟文件系统与用户空间通信，以便让用户空间管理和枚举设备及其公开的属性。

+   使用引用计数（在受管理资源中大量使用）管理对象生命周期。

+   电源管理，以处理设备应该关闭的顺序。

+   代码的可重用性。类和框架公开接口，行为类似于任何注册的驱动程序必须遵守的合同。

+   LDM 在内核中引入了类似于**面向对象**（**OO**）的编程风格。

在本章中，我们将利用 LDM 并通过`sysfs`文件系统向用户空间导出一些属性。

在本章中，我们将涵盖以下主题：

+   引入 LDM 数据结构（驱动程序，设备，总线）

+   按类型收集内核对象

+   处理内核`sysfs`接口

# LDM 数据结构

目标是构建一个完整的设备树，将系统上存在的每个物理设备映射到其中，并介绍它们的层次结构。已经创建了一个通用结构，用于表示可能是设备模型一部分的任何对象。LDM 的上一级依赖于内核中表示为`struct bus_type`实例的总线；设备驱动程序，表示为`struct device_driver`结构，以及设备，作为`struct device`结构的实例表示的最后一个元素。在本节中，我们将设计一个总线驱动程序包 bus，以深入了解 LDM 数据结构和机制。

# 总线

公共汽车是设备和处理器之间的通道链接。管理总线并向设备导出其协议的硬件实体称为总线控制器。例如，USB 控制器提供 USB 支持。I2C 控制器提供 I2C 总线支持。因此，总线控制器作为一个设备，必须像任何设备一样注册。它将是需要放在总线上的设备的父级。换句话说，每个放在总线上的设备必须将其父字段指向总线设备。总线在内核中由`struct bus_type`结构表示：

```
struct bus_type { 
   const char *name; 
   const char *dev_name; 
   struct device *dev_root; 
   struct device_attribute  *dev_attrs; /* use dev_groups instead */ 
   const struct attribute_group **bus_groups; 
   const struct attribute_group **dev_groups; 
   const struct attribute_group **drv_groups; 

   int (*match)(struct device *dev, struct device_driver *drv); 
   int (*probe)(struct device *dev); 
   int (*remove)(struct device *dev); 
   void (*shutdown)(struct device *dev); 

   int (*suspend)(struct device *dev, pm_message_t state); 
   int (*resume)(struct device *dev); 

   const struct dev_pm_ops *pm; 

   struct subsys_private *p; 
   struct lock_class_key lock_key; 
}; 
```

以下是结构中元素的含义：

+   `match`：这是一个回调，每当新设备或驱动程序添加到总线时都会调用。回调必须足够智能，并且在设备和驱动程序之间存在匹配时应返回非零值，这两者作为参数给出。`match`回调的主要目的是允许总线确定特定设备是否可以由给定驱动程序处理，或者其他逻辑，如果给定驱动程序支持给定设备。大多数情况下，验证是通过简单的字符串比较完成的（设备和驱动程序名称，或表和 DT 兼容属性）。对于枚举设备（PCI，USB），验证是通过比较驱动程序支持的设备 ID 与给定设备的设备 ID 进行的，而不会牺牲总线特定功能。

+   `probe`：这是在匹配发生后，当新设备或驱动程序添加到总线时调用的回调。此函数负责分配特定的总线设备结构，并调用给定驱动程序的`probe`函数，该函数应该管理之前分配的设备。

+   `remove`：当设备从总线中移除时调用此函数。

+   `suspend`：这是一种在总线上的设备需要进入睡眠模式时调用的方法。

+   `resume`：当总线上的设备需要被唤醒时调用此函数。

+   `pm`：这是总线的电源管理操作集，将调用特定设备驱动程序的`pm-ops`。

+   `drv_groups`：这是指向`struct attribute_group`元素列表（数组）的指针，每个元素都指向`struct attribute`元素列表（数组）。它代表总线上设备驱动程序的默认属性。传递给此字段的属性将赋予总线上注册的每个驱动程序。这些属性可以在`/sys/bus/<bus-name>/drivers/<driver-name>`中的驱动程序目录中找到。

+   `dev_groups`：这代表总线上设备的默认属性。通过传递给此字段的`struct attribute_group`元素的列表/数组，将赋予总线上注册的每个设备这些属性。这些属性可以在`/sys/bus/<bus-name>/devices/<device-name>`中的设备目录中找到。

+   `bus_group`：这保存总线注册到核心时自动添加的默认属性集（组）。

除了定义`bus_type`之外，总线控制器驱动程序还必须定义一个特定于总线的驱动程序结构，该结构扩展了通用的`struct device_driver`，以及一个特定于总线的设备结构，该结构扩展了通用的`struct device`结构，都是设备模型核心的一部分。总线驱动程序还必须为探测到的每个物理设备分配一个特定于总线的设备结构，并负责初始化设备的`bus`和`parent`字段，并将设备注册到 LDM 核心。这些字段必须指向总线设备和总线驱动程序中定义的`bus_type`结构。LDM 核心使用这些来构建设备层次结构并初始化其他字段。

在我们的示例中，以下是两个辅助宏，用于获取 packt 设备和 packt 驱动程序，给定通用的`struct device`和`struct driver`：

```
#define to_packt_driver(d) container_of(d, struct packt_driver, driver) 
#define to_packt_device(d) container_of(d, struct packt_device, dev) 
```

然后是用于识别 packt 设备的结构：

```
struct packt_device_id { 
    char name[PACKT_NAME_SIZE]; 
    kernel_ulong_t driver_data;   /* Data private to the driver */ 
}; 
```

以下是 packt 特定的设备和驱动程序结构：

```
/* 
 * Bus specific device structure 
 * This is what a packt device structure looks like 
 */ 
struct packt_device { 
   struct module *owner; 
   unsigned char name[30]; 
   unsigned long price; 
   struct device dev; 
}; 

/* 
 * Bus specific driver structure 
 * This is what a packt driver structure looks like 
 * You should provide your device's probe and remove function. 
 * may be release too 
 */ 
struct packt_driver { 
   int (*probe)(struct packt_device *packt); 
   int (*remove)(struct packt_device *packt); 
   void (*shutdown)(struct packt_device *packt); 
}; 
```

每个总线内部管理两个重要列表；添加到总线上的设备列表和注册到总线上的驱动程序列表。每当添加/注册或移除/注销设备/驱动程序到/从总线时，相应的列表都会更新为新条目。总线驱动程序必须提供辅助函数来注册/注销可以处理该总线上设备的设备驱动程序，以及注册/注销坐在总线上的设备的辅助函数。这些辅助函数始终包装 LDM 核心提供的通用函数，即`driver_register()`，`device_register()`，`driver_unregister`和`device_unregister()`。

```
/* 
 * Now let us write and export symbols that people writing 
 * drivers for packt devices must use. 
 */ 

int packt_register_driver(struct packt_driver *driver) 
{   
   driver->driver.bus = &packt_bus_type; 
   return driver_register(&driver->driver); 
} 
EXPORT_SYMBOL(packt_register_driver); 

void packt_unregister_driver(struct packt_driver *driver) 
{ 
   driver_unregister(&driver->driver); 
} 
EXPORT_SYMBOL(packt_unregister_driver); 

int packt_device_register(struct packt_device *packt) 
{ 
   return device_register(&packt->dev); 
} 
EXPORT_SYMBOL(packt_device_register); 

void packt_unregister_device(struct packt_device *packt) 
{ 
   device_unregister(&packt->dev); 
} 
EXPORT_SYMBOL(packt_device_unregister); 
```

用于分配 packt 设备的函数如下。必须使用此函数来创建总线上任何物理设备的实例：

```
/* 
 * This function allocate a bus specific device structure 
 * One must call packt_device_register to register 
 * the device with the bus 
 */ 
struct packt_device * packt_device_alloc(const char *name, int id) 
{ 
   struct packt_device *packt_dev; 
   int status; 

   packt_dev = kzalloc(sizeof *packt_dev, GFP_KERNEL); 
   if (!packt_dev) 
         return NULL; 

    /* new devices on the bus are son of the bus device */ 
    strcpy(packt_dev->name, name); 
    packt_dev->dev.id = id; 
    dev_dbg(&packt_dev->dev, 
      "device [%s] registered with packt bus\n", packt_dev->name); 

    return packt_dev; 

out_err: 
    dev_err(&adap->dev, "Failed to register packt client %s\n", packt_dev->name); 
    kfree(packt_dev); 
    return NULL; 
} 
EXPORT_SYMBOL_GPL(packt_device_alloc); 

int packt_device_register(struct packt_device *packt) 
{ 
    packt->dev.parent = &packt_bus; 
   packt->dev.bus = &packt_bus_type; 
   return device_register(&packt->dev); 
} 
EXPORT_SYMBOL(packt_device_register); 
```

# 总线注册

总线控制器本身也是一个设备，在 99%的情况下总线是平台设备（即使提供枚举的总线也是如此）。例如，PCI 控制器是一个平台设备，它的相应驱动程序也是如此。必须使用`bus_register(struct *bus_type)`函数来注册总线到内核。packt 总线结构如下：

```
/* 
 * This is our bus structure 
 */ 
struct bus_type packt_bus_type = { 
   .name      = "packt", 
   .match     = packt_device_match, 
   .probe     = packt_device_probe, 
   .remove    = packt_device_remove, 
   .shutdown  = packt_device_shutdown, 
}; 
```

总线控制器本身也是一个设备，它必须在内核中注册，并且将用作总线上设备的父设备。这是在总线控制器的`probe`或`init`函数中完成的。在 packt 总线的情况下，代码如下：

```
/* 
 * Bus device, the master. 
 *  
 */ 
struct device packt_bus = { 
    .release  = packt_bus_release, 
    .parent = NULL, /* Root device, no parent needed */ 
}; 

static int __init packt_init(void) 
{ 
    int status; 
    status = bus_register(&packt_bus_type); 
    if (status < 0) 
        goto err0; 

    status = class_register(&packt_master_class); 
    if (status < 0) 
        goto err1; 

    /* 
     * After this call, the new bus device will appear 
     * under /sys/devices in sysfs. Any devices added to this 
     * bus will shows up under /sys/devices/packt-0/. 
     */ 
    device_register(&packt_bus); 

   return 0; 

err1: 
   bus_unregister(&packt_bus_type); 
err0: 
   return status; 
} 
```

当总线控制器驱动程序注册设备时，设备的父成员必须指向总线控制器设备，其总线属性必须指向总线类型以构建物理 DT。要注册 packt 设备，必须调用`packt_device_register`，并将其分配为`packt_device_alloc`的参数：

```
int packt_device_register(struct packt_device *packt) 
{ 
    packt->dev.parent = &packt_bus; 
   packt->dev.bus = &packt_bus_type; 
   return device_register(&packt->dev); 
} 
EXPORT_SYMBOL(packt_device_register); 
```

# 设备驱动程序

全局设备层次结构允许以通用方式表示系统中的每个设备。这使得核心可以轻松地遍历 DT 以创建诸如适当排序的电源管理转换之类的东西：

```
struct device_driver { 
    const char *name; 
    struct bus_type *bus; 
    struct module *owner; 

    const struct of_device_id   *of_match_table; 
    const struct acpi_device_id  *acpi_match_table; 

    int (*probe) (struct device *dev); 
    int (*remove) (struct device *dev); 
    void (*shutdown) (struct device *dev); 
    int (*suspend) (struct device *dev, pm_message_t state); 
    int (*resume) (struct device *dev); 
    const struct attribute_group **groups; 

    const struct dev_pm_ops *pm; 
}; 
```

`struct device_driver` 定义了一组简单的操作，供核心对每个设备执行这些操作：

+   `* name` 表示驱动程序的名称。它可以通过与设备名称进行比较来进行匹配。

+   `* bus` 表示驱动程序所在的总线。总线驱动程序必须填写此字段。

+   `module` 表示拥有驱动程序的模块。在 99% 的情况下，应将此字段设置为 `THIS_MODULE`。

+   `of_match_table` 是指向 `struct of_device_id` 数组的指针。`struct of_device_id` 结构用于通过称为 DT 的特殊文件执行 OF 匹配，该文件在引导过程中传递给内核：

```
struct of_device_id { 
    char compatible[128]; 
    const void *data; 
}; 
```

+   `suspend` 和 `resume` 回调提供电源管理功能。当设备从系统中物理移除或其引用计数达到 `0` 时，将调用 `remove` 回调。在系统重新启动期间也会调用 `remove` 回调。

+   `probe` 是在尝试将驱动程序绑定到设备时运行的探测回调函数。总线驱动程序负责调用设备驱动程序的 `probe` 函数。

+   `group` 是指向 `struct attribute_group` 列表（数组）的指针，用作驱动程序的默认属性。使用此方法而不是单独创建属性。

# 设备驱动程序注册

`driver_register()` 是用于在总线上注册设备驱动程序的低级函数。它将驱动程序添加到总线的驱动程序列表中。当设备驱动程序与总线注册时，核心会遍历总线的设备列表，并对每个没有与之关联驱动程序的设备调用总线的匹配回调，以找出驱动程序可以处理的设备。

当发生匹配时，设备和设备驱动程序被绑定在一起。将设备与设备驱动程序关联的过程称为绑定。

现在回到使用我们的 packt 总线注册驱动程序，必须使用 `packt_register_driver(struct packt_driver *driver)`，这是对 `driver_register()` 的包装。在注册 packt 驱动程序之前，必须填写 `*driver` 参数。LDM 核心提供了用于遍历已注册到总线的驱动程序列表的辅助函数：

```
int bus_for_each_drv(struct bus_type * bus, 
                struct device_driver * start,  
                void * data, int (*fn)(struct device_driver *, 
                void *)); 
```

此助手遍历总线的驱动程序列表，并对列表中的每个驱动程序调用 `fn` 回调。

# 设备

结构体设备是用于描述和表征系统上每个设备的通用数据结构，无论其是否是物理设备。它包含有关设备的物理属性的详细信息，并提供适当的链接信息以构建合适的设备树和引用计数：

```
struct device { 
    struct device *parent; 
    struct kobject kobj; 
    const struct device_type *type; 
    struct bus_type      *bus; 
    struct device_driver *driver; 
    void    *platform_data; 
    void *driver_data; 
    struct device_node      *of_node; 
    struct class *class; 
    const struct attribute_group **groups; 
    void (*release)(struct device *dev); 
}; 
```

+   `* parent` 表示设备的父级，用于构建设备树层次结构。当与总线注册时，总线驱动程序负责使用总线设备设置此字段。

+   `* bus` 表示设备所在的总线。总线驱动程序必须填写此字段。

+   `* type` 标识设备的类型。

+   `kobj` 是处理引用计数和设备模型支持的 kobject。

+   `* of_node` 是指向与设备关联的 OF（DT）节点的指针。由总线驱动程序设置此字段。

+   `platform_data` 是指向特定于设备的平台数据的指针。通常在设备供应期间在特定于板的文件中声明。

+   `driver_data` 是驱动程序的私有数据的指针。

+   `class` 是指向设备所属类的指针。

+   `* group` 是指向 `struct attribute_group` 列表（数组）的指针，用作设备的默认属性。使用此方法而不是单独创建属性。

+   `release` 是在设备引用计数达到零时调用的回调。总线有责任设置此字段。packt 总线驱动程序向您展示了如何做到这一点。

# 设备注册

`device_register`是 LDM 核心提供的用于在总线上注册设备的函数。调用此函数后，将遍历驱动程序的总线列表以找到支持此设备的驱动程序，然后将此设备添加到总线的设备列表中。`device_register()`在内部调用`device_add()`：

```
int device_add(struct device *dev) 
{ 
    [...] 
    bus_probe_device(dev); 
       if (parent) 
             klist_add_tail(&dev->p->knode_parent, 
                          &parent->p->klist_children); 
    [...] 
} 
```

内核提供的用于遍历总线设备列表的辅助函数是`bus_for_each_dev`：

```
int bus_for_each_dev(struct bus_type * bus, 
                    struct device * start, void * data, 
                    int (*fn)(struct device *, void *)); 
```

每当添加设备时，核心都会调用总线驱动程序的匹配方法（`bus_type->match`）。如果匹配函数表示有驱动程序支持此设备，核心将调用总线驱动程序的`probe`函数（`bus_type->probe`），给定设备和驱动程序作为参数。然后由总线驱动程序调用设备的驱动程序的`probe`方法（`driver->probe`）。对于我们的 packt 总线驱动程序，用于注册设备的函数是`packt_device_register(struct packt_device *packt)`，它在内部调用`device_register`，参数是使用`packt_device_alloc`分配的 packt 设备。

# 深入 LDM

LDM 在内部依赖于三个重要的结构，即 kobject、kobj_type 和 kset。让我们看看这些结构中的每一个如何参与设备模型。

# kobject 结构

kobject 是设备模型的核心，运行在后台。它为内核带来了类似 OO 的编程风格，主要用于引用计数和公开设备层次结构和它们之间的关系。kobject 引入了封装常见对象属性的概念，如使用引用计数：

```
struct kobject { 
    const char *name; 
    struct list_head entry; 
    struct kobject *parent; 
    struct kset *kset; 
    struct kobj_type *ktype; 
    struct sysfs_dirent *sd; 
    struct kref kref; 
    /* Fields out of our interest have been removed */ 
}; 
```

+   `name`指向此 kobject 的名称。可以使用`kobject_set_name(struct kobject *kobj, const char *name)`函数来更改这个名称。

+   `parent`是指向此 kobject 父级的指针。它用于构建描述对象之间关系的层次结构。

+   `sd`指向一个`struct sysfs_dirent`结构，表示 sysfs 中此 kobject 的 inode 内部的结构。

+   `kref`提供了对 kobject 的引用计数。

+   `ktype`描述了对象，`kset`告诉我们这个对象属于哪个集合（组）。

每个嵌入 kobject 的结构都会嵌入并接收 kobject 提供的标准化函数。嵌入的 kobject 将使结构成为对象层次结构的一部分。

`container_of`宏用于获取 kobject 所属对象的指针。每个内核设备直接或间接地嵌入一个 kobject 属性。在添加到系统之前，必须使用`kobject_create()`函数分配 kobject，该函数将返回一个空的 kobject，必须使用`kobj_init()`进行初始化，给定分配和未初始化的 kobject 指针以及其`kobj_type`指针：

```
struct kobject *kobject_create(void) 
void kobject_init(struct kobject *kobj, struct kobj_type *ktype) 
```

`kobject_add()`函数用于将 kobject 添加和链接到系统，同时根据其层次结构创建其目录，以及其默认属性。反向函数是`kobject_del()`：

```
int kobject_add(struct kobject *kobj, struct kobject *parent, 
                const char *fmt, ...); 
```

`kobject_create`和`kobject_add`的反向函数是`kobject_put`。在书中提供的源代码中，将 kobject 绑定到系统的摘录是：

```
/* Somewhere */ 
static struct kobject *mykobj; 

mykobj = kobject_create(); 
    if (mykobj) { 
        kobject_init(mykobj, &mytype); 
        if (kobject_add(mykobj, NULL, "%s", "hello")) { 
             err = -1; 
             printk("ldm: kobject_add() failed\n"); 
             kobject_put(mykobj); 
             mykobj = NULL; 
        } 
        err = 0; 
    } 
```

可以使用`kobject_create_and_add`，它在内部调用`kobject_create`和`kobject_add`。`drivers/base/core.c`中的以下摘录显示了如何使用它：

```
static struct kobject * class_kobj   = NULL; 
static struct kobject * devices_kobj = NULL; 

/* Create /sys/class */ 
class_kobj = kobject_create_and_add("class", NULL); 

if (!class_kobj) { 
    return -ENOMEM; 
} 

/* Create /sys/devices */ 
devices_kobj = kobject_create_and_add("devices", NULL); 

if (!devices_kobj) { 
    return -ENOMEM; 
} 
```

如果 kobject 有一个`NULL`父级，那么`kobject_add`会将父级设置为 kset。如果两者都是`NULL`，对象将成为顶级 sys 目录的子成员

# kobj_type

`struct kobj_type`结构描述了 kobjects 的行为。`kobj_type`结构通过`ktype`字段描述了嵌入 kobject 的对象的类型。每个嵌入 kobject 的结构都需要一个相应的`kobj_type`，它将控制在创建和销毁 kobject 以及读取或写入属性时发生的情况。每个 kobject 都有一个`struct kobj_type`类型的字段，代表**内核对象类型**：

```
struct kobj_type { 
   void (*release)(struct kobject *); 
   const struct sysfs_ops sysfs_ops; 
   struct attribute **default_attrs; 
}; 
```

`struct kobj_type`结构允许内核对象共享公共操作（`sysfs_ops`），无论这些对象是否在功能上相关。该结构的字段是有意义的。`release`是由`kobject_put()`函数调用的回调，每当需要释放对象时。您必须在这里释放对象持有的内存。可以使用`container_of`宏来获取对象的指针。`sysfs_ops`字段指向 sysfs 操作，而`default_attrs`定义了与此 kobject 关联的默认属性。`sysfs_ops`是一组在访问 sysfs 属性时调用的回调（sysfs 操作）。`default_attrs`是指向`struct attribute`元素列表的指针，将用作此类型的每个对象的默认属性：

```
struct sysfs_ops { 
    ssize_t (*show)(struct kobject *kobj, 
                    struct attribute *attr, char *buf); 
    ssize_t (*store)(struct kobject *kobj, 
                     struct attribute *attr,const char *buf, 
                     size_t size); 
}; 
```

`show`是当读取具有此`kobj_type`的任何 kobject 的属性时调用的回调。缓冲区大小始终为`PAGE_SIZE`，即使要显示的值是一个简单的`char`。应该设置`buf`的值（使用`scnprintf`），并在成功时返回实际写入缓冲区的数据的大小（以字节为单位），或者在失败时返回负错误。`store`用于写入目的。它的`buf`参数最多为`PAGE_SIZE`，但可以更小。它在成功时返回实际从缓冲区读取的数据的大小（以字节为单位），或者在失败时返回负错误（或者如果它收到一个不需要的值）。可以使用`get_ktype`来获取给定 kobject 的`kobj_type`：

```
struct kobj_type *get_ktype(struct  kobject *kobj); 
```

在书中的示例中，我们的`k_type`变量表示我们 kobject 的类型：

```
static struct sysfs_ops s_ops = { 
    .show = show, 
    .store = store, 
}; 

static struct kobj_type k_type = { 
    .sysfs_ops = &s_ops, 
    .default_attrs = d_attrs, 
}; 
```

这里，`show`和`store`回调定义如下：

```
static ssize_t show(struct kobject *kobj, struct attribute *attr, char *buf) 
{ 
    struct d_attr *da = container_of(attr, struct d_attr, attr); 
    printk( "LDM show: called for (%s) attr\n", da->attr.name ); 
    return scnprintf(buf, PAGE_SIZE, 
                     "%s: %d\n", da->attr.name, da->value); 
} 

static ssize_t store(struct kobject *kobj, struct attribute *attr, const char *buf, size_t len) 
{ 
    struct d_attr *da = container_of(attr, struct d_attr, attr); 
    sscanf(buf, "%d", &da->value); 
    printk("LDM store: %s = %d\n", da->attr.name, da->value); 

    return sizeof(int); 
} 
```

# ksets

**内核对象集**（**ksets**）主要将相关的内核对象分组在一起。ksets 是 kobjects 的集合。换句话说，kset 将相关的 kobjects 聚集到一个地方，例如，所有块设备：

```
struct kset { 
   struct list_head list;  
   spinlock_t list_lock; 
   struct kobject kobj; 
 }; 
```

+   `list`是 kset 中所有 kobject 的链表

+   `list_lock`是用于保护链表访问的自旋锁

+   `kobj`表示集合的基类

每个注册（添加到系统中）的 kset 对应一个 sysfs 目录。可以使用`kset_create_and_add()`函数创建和添加 kset，并使用`kset_unregister()`函数删除：

```
struct kset * kset_create_and_add(const char *name, 
                                const struct kset_uevent_ops *u, 
                                struct kobject *parent_kobj); 
void kset_unregister (struct kset * k); 
```

将 kobject 添加到集合中就像将其 kset 字段指定为正确的 kset 一样简单：

```
static struct kobject foo_kobj, bar_kobj; 

example_kset = kset_create_and_add("kset_example", NULL, kernel_kobj); 
/* 
 * since we have a kset for this kobject, 
 * we need to set it before calling the kobject core. 
 */ 
foo_kobj.kset = example_kset; 
bar_kobj.kset = example_kset; 

retval = kobject_init_and_add(&foo_kobj, &foo_ktype, 
                              NULL, "foo_name"); 
retval = kobject_init_and_add(&bar_kobj, &bar_ktype, 
                              NULL, "bar_name"); 
```

现在在模块的`exit`函数中，kobject 及其属性已被删除：

```
kset_unregister(example_kset); 
```

# 属性

属性是由 kobjects 向用户空间导出的 sysfs 文件。属性表示可以从用户空间可读、可写或两者的对象属性。也就是说，每个嵌入`struct kobject`的数据结构可以公开由 kobject 本身提供的默认属性（如果有的话），也可以公开自定义属性。换句话说，属性将内核数据映射到 sysfs 中的文件。

属性定义如下：

```
struct attribute { 
        char * name; 
        struct module *owner; 
        umode_t mode; 
}; 
```

用于从文件系统中添加/删除属性的内核函数是：

```
int sysfs_create_file(struct kobject * kobj, 
                      const struct attribute * attr); 
void sysfs_remove_file(struct kobject * kobj, 
                        const struct attribute * attr); 
```

让我们尝试定义两个我们将导出的属性，每个属性由一个属性表示：

```
struct d_attr { 
    struct attribute attr; 
    int value; 
}; 

static struct d_attr foo = { 
    .attr.name="foo", 
    .attr.mode = 0644, 
    .value = 0, 
}; 

static struct d_attr bar = { 
    .attr.name="bar", 
    .attr.mode = 0644, 
    .value = 0, 
}; 
```

要单独创建每个枚举属性，我们必须调用以下内容：

```
sysfs_create_file(mykobj, &foo.attr); 
sysfs_create_file(mykobj, &bar.attr); 
```

属性的一个很好的起点是内核源码中的`samples/kobject/kobject-example.c`。

# 属性组

到目前为止，我们已经看到了如何单独添加属性，并在每个属性上调用（直接或间接通过包装函数，如`device_create_file()`，`class_create_file()`等）`sysfs_create_file()`。如果我们可以一次完成，为什么要自己处理多个调用呢？这就是属性组的作用。它依赖于`struct attribute_group`结构：

```
struct attribute_group { 
   struct attribute  **attrs; 
}; 
```

当然，我们已经删除了不感兴趣的字段。`attr`字段是指向属性列表/数组的指针。每个属性组必须给定一个指向`struct attribute`元素的列表/数组的指针。该组只是一个帮助包装器，使得更容易管理多个属性。

用于向文件系统添加/删除组属性的内核函数是：

```
int sysfs_create_group(struct kobject *kobj, 
                       const struct attribute_group *grp) 
void sysfs_remove_group(struct kobject * kobj, 
                        const struct attribute_group * grp) 
```

前面定义的两个属性可以嵌入到`struct attribute_group`中，只需一次调用即可将它们都添加到系统中：

```
static struct d_attr foo = { 
    .attr.name="foo", 
    .attr.mode = 0644, 
    .value = 0, 
}; 

static struct d_attr bar = { 
    .attr.name="bar", 
    .attr.mode = 0644, 
    .value = 0, 
}; 

/* attrs is a pointer to a list (array) of attributes */ 
static struct attribute * attrs [] = 
{ 
    &foo.attr, 
    &bar.attr, 
    NULL, 
}; 

static struct attribute_group my_attr_group = { 
    .attrs = attrs, 
}; 
```

在这里唯一需要调用的函数是：

```
sysfs_create_group(mykobj, &my_attr_group); 
```

这比为每个属性都调用一次要好得多。

# 设备模型和 sysfs

`Sysfs`是一个非持久的虚拟文件系统，它提供了系统的全局视图，并通过它们的 kobjects 公开了内核对象的层次结构（拓扑）。每个 kobjects 显示为一个目录，目录中的文件表示由相关 kobject 导出的内核变量。这些文件称为属性，可以被读取或写入。

如果任何注册的 kobject 在 sysfs 中创建一个目录，那么目录的创建取决于 kobject 的父对象（也是一个 kobject）。自然而然地，目录被创建为 kobject 的父目录的子目录。这将内部对象层次结构突显到用户空间。sysfs 中的顶级目录表示对象层次结构的共同祖先，也就是对象所属的子系统。

顶级 sysfs 目录可以在`/sys/`目录下找到：

```
    /sys$ tree -L 1

    ├── block

    ├── bus

    ├── class

    ├── dev

    ├── devices

    ├── firmware

    ├── fs

    ├── hypervisor

    ├── kernel

    ├── module

    └── power

```

`block`包含系统上每个块设备的目录，每个目录包含设备上分区的子目录。`bus`包含系统上注册的总线。`dev`以原始方式包含注册的设备节点（无层次结构），每个都是指向`/sys/devices`目录中真实设备的符号链接。`devices`显示系统中设备的拓扑视图。`firmware`显示系统特定的低级子系统树，例如：ACPI、EFI、OF（DT）。`fs`列出系统上实际使用的文件系统。`kernel`保存内核配置选项和状态信息。`Modules`是已加载模块的列表。

这些目录中的每一个都对应一个 kobject，其中一些作为内核符号导出。这些是：

+   `kernel_kobj`对应于`/sys/kernel`

+   `power_kobj`对应于`/sys/power`

+   `firmware_kobj`对应于`/sys/firmware`，在`drivers/base/firmware.c`源文件中导出

+   `hypervisor_kobj`对应于`/sys/hypervisor`，在`drivers/base/hypervisor.c`中导出

+   `fs_kobj`对应于`/sys/fs`，在`fs/namespace.c`文件中导出

然而，`class/`、`dev/`、`devices/`是在内核源代码中的`drivers/base/core.c`中由`devices_init`函数在启动时创建的，`block/`是在`block/genhd.c`中创建的，`bus/`是在`drivers/base/bus.c`中作为 kset 创建的。

当将 kobject 目录添加到 sysfs（使用`kobject_add`）时，它被添加的位置取决于 kobject 的父位置。如果其父指针已设置，则将其添加为父目录中的子目录。如果父指针为空，则将其添加为`kset->kobj`中的子目录。如果父字段和 kset 字段都未设置，则映射到 sysfs 中的根级目录（`/sys`）。

可以使用`sysfs_{create|remove}_link`函数在现有对象（目录）上创建/删除符号链接：

```
int sysfs_create_link(struct kobject * kobj, 
                      struct kobject * target, char * name);  
void sysfs_remove_link(struct kobject * kobj, char * name); 
```

这将允许一个对象存在于多个位置。创建函数将创建一个名为`name`的符号链接，指向`target` kobject sysfs 条目。一个众所周知的例子是设备同时出现在`/sys/bus`和`/sys/devices`中。创建的符号链接将在`target`被移除后仍然存在。您必须知道`target`何时被移除，然后删除相应的符号链接。

# Sysfs 文件和属性

现在我们知道，默认的文件集是通过 kobjects 和 ksets 中的 ktype 字段提供的，通过`kobj_type`的`default_attrs`字段。默认属性在大多数情况下都足够了。但有时，ktype 的一个实例可能需要自己的属性来提供不被更一般的 ktype 共享的数据或功能。

只是一个提醒，用于在默认集合之上添加/删除新属性（或属性组）的低级函数是：

```
int sysfs_create_file(struct kobject *kobj,  
                      const struct attribute *attr); 
void sysfs_remove_file(struct kobject *kobj, 
                       const struct attribute *attr); 
int sysfs_create_group(struct kobject *kobj, 
                       const struct attribute_group *grp); 
void sysfs_remove_group(struct kobject * kobj, 
                        const struct attribute_group * grp); 
```

# 当前接口

目前在 sysfs 中存在接口层。除了创建自己的 ktype 或 kobject 以添加属性外，还可以使用当前存在的属性：设备、驱动程序、总线和类属性。它们的描述如下：

# 设备属性

除了设备结构中嵌入的默认属性之外，您还可以创建自定义属性。用于此目的的结构是`struct device_attribute`，它只是标准`struct attribute`的包装，并且一组回调函数来显示/存储属性的值：

```
struct device_attribute { 
    struct attribute attr; 
    ssize_t (*show)(struct device *dev, 
                    struct device_attribute *attr, 
                   char *buf); 
    ssize_t (*store)(struct device *dev, 
                     struct device_attribute *attr, 
                     const char *buf, size_t count); 
}; 
```

它们的声明是通过`DEVICE_ATTR`宏完成的：

```
DEVICE_ATTR(_name, _mode, _show, _store); 
```

每当使用`DEVICE_ATTR`声明设备属性时，属性名称前缀`dev_attr_`将添加到属性名称中。例如，如果使用`_name`参数设置为 foo 来声明属性，则可以通过`dev_attr_foo`变量名称访问该属性。

要理解为什么，让我们看看`DEVICE_ATTR`宏在`include/linux/device.h`中是如何定义的：

```
#define DEVICE_ATTR(_name, _mode, _show, _store) \ 
   struct device_attribute dev_attr_##_name = __ATTR(_name, _mode, _show, _store) 
```

最后，您可以使用`device_create_file`和`device_remove_file`函数添加/删除这些：

```
int device_create_file(struct device *dev,  
                      const struct device_attribute * attr); 
void device_remove_file(struct device *dev, 
                       const struct device_attribute * attr); 
```

以下示例演示了如何将所有内容放在一起：

```
static ssize_t foo_show(struct device *child, 
    struct device_attribute *attr, char *buf) 
{ 
    return sprintf(buf, "%d\n", foo_value); 
} 

static ssize_t bar_show(struct device *child, 
         struct device_attribute *attr, char *buf) 
{ 
    return sprintf(buf, "%d\n", bar_value); 
}  
```

以下是属性的静态声明：

```
static DEVICE_ATTR(foo, 0644, foo_show, NULL); 
static DEVICE_ATTR(bar, 0644, bar_show, NULL); 
```

以下代码显示了如何在系统上实际创建文件：

```
if ( device_create_file(dev, &dev_attr_foo) != 0 ) 
    /* handle error */ 

if ( device_create_file(dev, &dev_attr_bar) != 0 ) 
    /* handle error*/ 
```

对于清理，属性的移除是在移除函数中完成的：

```
device_remove_file(wm->dev, &dev_attr_foo); 
device_remove_file(wm->dev, &dev_attr_bar); 
```

您可能会想知道我们是如何以前定义相同的存储/显示回调来处理相同 kobject/ktype 的所有属性，现在我们为每个属性使用自定义的回调。第一个原因是，设备子系统定义了自己的属性结构，它包装了标准属性结构，其次，它不是显示/存储属性的值，而是使用`container_of`宏来提取`struct device_attribute`，从而给出一个通用的`struct attribute`，然后根据用户操作执行 show/store 回调。以下是来自`drivers/base/core.c`的摘录，显示了设备 kobject 的`sysfs_ops`：

```
static ssize_t dev_attr_show(struct kobject *kobj, 
                            struct attribute *attr, 
                            char *buf) 
{ 
   struct device_attribute *dev_attr = to_dev_attr(attr); 
   struct device *dev = kobj_to_dev(kobj); 
   ssize_t ret = -EIO; 

   if (dev_attr->show) 
         ret = dev_attr->show(dev, dev_attr, buf); 
   if (ret >= (ssize_t)PAGE_SIZE) { 
         print_symbol("dev_attr_show: %s returned bad count\n", 
                     (unsigned long)dev_attr->show); 
   } 
   return ret; 
} 

static ssize_t dev_attr_store(struct kobject *kobj, struct attribute *attr, 
                     const char *buf, size_t count) 
{ 
   struct device_attribute *dev_attr = to_dev_attr(attr); 
   struct device *dev = kobj_to_dev(kobj); 
   ssize_t ret = -EIO; 

   if (dev_attr->store) 
         ret = dev_attr->store(dev, dev_attr, buf, count); 
   return ret; 
} 

static const struct sysfs_ops dev_sysfs_ops = { 
   .show = dev_attr_show, 
   .store      = dev_attr_store, 
}; 
```

原则对于总线（在`drivers/base/bus.c`中）、驱动程序（在`drivers/base/bus.c`中）和类（在`drivers/base/class.c`中）属性是相同的。它们使用`container_of`宏来提取其特定属性结构，然后调用其中嵌入的 show/store 回调。

# 总线属性

它依赖于`struct bus_attribute`结构：

```
struct bus_attribute { 
   struct attribute attr; 
   ssize_t (*show)(struct bus_type *, char * buf); 
   ssize_t (*store)(struct bus_type *, const char * buf, size_t count); 
}; 
```

使用`BUS_ATTR`宏声明总线属性：

```
BUS_ATTR(_name, _mode, _show, _store) 
```

使用`BUS_ATTR`声明的任何总线属性都将在属性变量名称中添加前缀`bus_attr_`：

```
#define BUS_ATTR(_name, _mode, _show, _store)      \ 
struct bus_attribute bus_attr_##_name = __ATTR(_name, _mode, _show, _store) 
```

它们是使用`bus_{create|remove}_file`函数创建/删除的：

```
int bus_create_file(struct bus_type *, struct bus_attribute *); 
void bus_remove_file(struct bus_type *, struct bus_attribute *); 
```

# 设备驱动程序属性

所使用的结构是`struct driver_attribute`：

```
struct driver_attribute { 
        struct attribute attr; 
        ssize_t (*show)(struct device_driver *, char * buf); 
        ssize_t (*store)(struct device_driver *, const char * buf, 
                         size_t count); 
}; 
```

声明依赖于`DRIVER_ATTR`宏，该宏将在属性变量名称中添加前缀`driver_attr_`：

```
DRIVER_ATTR(_name, _mode, _show, _store) 
```

宏定义如下：

```
#define DRIVER_ATTR(_name, _mode, _show, _store) \ 
struct driver_attribute driver_attr_##_name = __ATTR(_name, _mode, _show, _store) 
```

创建/删除依赖于`driver_{create|remove}_file`函数：

```
int driver_create_file(struct device_driver *, 
                       const struct driver_attribute *); 
void driver_remove_file(struct device_driver *, 
                       const struct driver_attribute *); 
```

# 类属性

`struct class_attribute`是基本结构：

```
struct class_attribute { 
        struct attribute        attr; 
        ssize_t (*show)(struct device_driver *, char * buf); 
        ssize_t (*store)(struct device_driver *, const char * buf, 
                         size_t count); 
}; 
```

类属性的声明依赖于`CLASS_ATTR`：

```
CLASS_ATTR(_name, _mode, _show, _store) 
```

正如宏的定义所示，使用`CLASS_ATTR`声明的任何类属性都将在属性变量名称中添加前缀`class_attr_`：

```
#define CLASS_ATTR(_name, _mode, _show, _store) \ 
struct class_attribute class_attr_##_name = __ATTR(_name, _mode, _show, _store) 
```

最后，文件的创建和删除是使用`class_{create|remove}_file`函数完成的：

```
int class_create_file(struct class *class, 
        const struct class_attribute *attr); 

void class_remove_file(struct class *class, 
        const struct class_attribute *attr); 
```

请注意，`device_create_file()`，`bus_create_file()`，`driver_create_file()`和`class_create_file()`都会内部调用`sysfs_create_file()`。由于它们都是内核对象，它们的结构中嵌入了`kobject`。然后将该`kobject`作为参数传递给`sysfs_create_file`，如下所示：

```
int device_create_file(struct device *dev, 
                    const struct device_attribute *attr) 
{ 
    [...] 
    error = sysfs_create_file(&dev->kobj, &attr->attr); 
    [...] 
} 

int class_create_file(struct class *cls, 
                    const struct class_attribute *attr) 
{ 
    [...] 
    error = 
        sysfs_create_file(&cls->p->class_subsys.kobj, 
                          &attr->attr); 
    return error; 
} 

int bus_create_file(struct bus_type *bus, 
                   struct bus_attribute *attr) 
{ 
    [...] 
    error = 
        sysfs_create_file(&bus->p->subsys.kobj, 
                           &attr->attr); 
    [...] 
} 
```

# 允许 sysfs 属性文件进行轮询

在这里，我们将看到如何避免进行 CPU 浪费的轮询以检测 sysfs 属性数据的可用性。想法是使用`poll`或`select`系统调用等待属性内容的更改。使 sysfs 属性可轮询的补丁是由**Neil Brown**和**Greg Kroah-Hartman**创建的。kobject 管理器（具有对 kobject 的访问权限的驱动程序）必须支持通知，以允许`poll`或`select`在内容更改时返回（被释放）。执行这一技巧的神奇函数来自内核侧，即`sysfs_notify()`：

```
void sysfs_notify(struct kobject *kobj, const char *dir, 
                  const char *attr) 
```

如果`dir`参数非空，则用于查找包含属性的子目录（可能是由`sysfs_create_group`创建的）。每个属性的成本为一个`int`，每个 kobject 的`wait_queuehead`，每个打开文件一个 int。

`poll`将返回`POLLERR|POLLPRI`，而`select`将返回 fd，无论它是等待读取、写入还是异常。阻塞的 poll 来自用户端。只有在调整内核属性值后才应调用`sysfs_notify()`。

将`poll()`（或`select()`）代码视为对感兴趣属性的更改通知的**订阅者**，并将`sysfs_notify()`视为**发布者**，通知订阅者任何更改。

以下是书中提供的代码摘录，这是属性的存储函数：

```
static ssize_t store(struct kobject *kobj, struct attribute *attr, 
                     const char *buf, size_t len) 
{ 
    struct d_attr *da = container_of(attr, struct d_attr, attr); 

    sscanf(buf, "%d", &da->value); 
    printk("sysfs_foo store %s = %d\n", a->attr.name, a->value); 

    if (strcmp(a->attr.name, "foo") == 0){ 
        foo.value = a->value; 
        sysfs_notify(mykobj, NULL, "foo"); 
    } 
    else if(strcmp(a->attr.name, "bar") == 0){ 
        bar.value = a->value; 
        sysfs_notify(mykobj, NULL, "bar"); 
    } 
    return sizeof(int); 
} 
```

用户空间的代码必须像这样才能感知数据的更改：

1.  打开文件属性。

1.  对所有内容进行虚拟读取。

1.  调用`poll`请求`POLLERR|POLLPRI`（select/exceptfds 也可以）。

1.  当`poll`（或`select`）返回（表示值已更改）时，读取数据已更改的文件内容。

1.  关闭文件并返回循环的顶部。

如果对 sysfs 属性是否可轮询存在疑问，请设置合适的超时值。书中提供了用户空间示例。

# 摘要

现在您已经熟悉了 LDM 概念及其数据结构（总线、类、设备驱动程序和设备），包括低级数据结构，即`kobject`、`kset`、`kobj_types`和属性（或这些属性的组合），内核中如何表示对象（因此 sysfs 和设备拓扑结构）不再是秘密。您将能够创建一个通过 sysfs 公开您的设备或驱动程序功能的属性（或组）。如果前面的话题对您来说很清楚，我们将转到下一个第十四章，*引脚控制和 GPIO 子系统*，该章节大量使用了`sysfs`的功能。
