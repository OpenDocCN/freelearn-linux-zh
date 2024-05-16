# 第四章：字符设备驱动程序

字符设备通过字符的方式（一个接一个）向用户应用程序传输数据，就像串行端口一样。字符设备驱动程序通过`/dev`目录中的特殊文件公开设备的属性和功能，可以用来在设备和用户应用程序之间交换数据，并且还允许你控制真实的物理设备。这是 Linux 的基本概念，即*一切都是文件*。字符设备驱动程序代表内核源代码中最基本的设备驱动程序。字符设备在内核中表示为`include/linux/cdev.h`中定义的`struct cdev`的实例：

```
struct cdev { 
    struct kobject kobj; 
    struct module *owner; 
    const struct file_operations *ops; 
    struct list_head list; 
    dev_t dev; 
    unsigned int count; 
}; 
```

本章将介绍字符设备驱动程序的具体特性，解释它们如何创建、识别和向系统注册设备，还将更好地概述设备文件方法，这些方法是内核向用户空间公开设备功能的方法，可通过使用与文件相关的系统调用（`read`，`write`，`select`，`open`，`close`等）访问，描述在`struct file_operations`结构中，这些你肯定以前听说过。

# 主要和次要背后的概念

字符设备位于`/dev`目录中。请注意，它们不是该目录中唯一的文件。字符设备文件可以通过其类型识别，我们可以通过`ls -l`命令显示。主要和次要标识并将设备与驱动程序绑定。让我们看看它是如何工作的，通过列出`*/dev*`目录的内容（`ls -l /dev`）：

```
[...]

drwxr-xr-x 2 root root 160 Mar 21 08:57 input

crw-r----- 1 root kmem 1, 2 Mar 21 08:57 kmem

lrwxrwxrwx 1 root root 28 Mar 21 08:57 log -> /run/systemd/journal/dev-log

crw-rw---- 1 root disk 10, 237 Mar 21 08:57 loop-control

brw-rw---- 1 root disk 7, 0 Mar 21 08:57 loop0

brw-rw---- 1 root disk 7, 1 Mar 21 08:57 loop1

brw-rw---- 1 root disk 7, 2 Mar 21 08:57 loop2

brw-rw---- 1 root disk 7, 3 Mar 21 08:57 loop3

```

给定上述摘录，第一列的第一个字符标识文件类型。可能的值有：

+   `c`：这是用于字符设备文件

+   `b`：这是用于块设备文件

+   `l`：这是用于符号链接

+   `d`：这是用于目录

+   `s`：这是用于套接字

+   `p`：这是用于命名管道

对于`b`和`c`文件类型，在日期之前的第五和第六列遵循<`X，Y`>模式。`X`代表主要号，`Y`是次要号。例如，第三行是<`1，2`>，最后一行是<`7，3`>。这是一种从用户空间识别字符设备文件及其主要和次要的经典方法之一。

内核在`dev_t`类型变量中保存标识设备的数字，它们只是`u32`（32 位无符号长整型）。主要号仅用 12 位表示，而次要号编码在剩余的 20 位上。

正如可以在`include/linux/kdev_t.h`中看到的，给定一个`dev_t`类型的变量，可能需要提取次要或主要。内核为这些目的提供了一个宏：

```
MAJOR(dev_t dev); 
MINOR(dev_t dev); 
```

另一方面，你可能有一个次要和一个主要，需要构建一个`dev_t`。你应该使用的宏是`MKDEV(int major, int minor);`：

```
#define MINORBITS    20 
#define MINORMASK    ((1U << MINORBITS) - 1) 
#define MAJOR(dev)   ((unsigned int) ((dev) >> MINORBITS)) 
#define MINOR(dev)   ((unsigned int) ((dev) & MINORMASK)) 
#define MKDEV(ma,mi) (((ma) << MINORBITS) | (mi)) 
```

设备注册时使用一个标识设备的主要号和一个次要号，可以将次要号用作本地设备列表的数组索引，因为同一驱动程序的一个实例可能处理多个设备，而不同的驱动程序可能处理相同类型的不同设备。

# 设备号分配和释放

设备号标识系统中的设备文件。这意味着，有两种分配这些设备号（实际上是主要和次要）的方法：

+   **静态**：使用`register_chrdev_region()`函数猜测尚未被其他驱动程序使用的主要号。应尽量避免使用这个。它的原型如下：

```
   int register_chrdev_region(dev_t first, unsigned int count, \ 
                             char *name); 
```

该方法在成功时返回`0`，在失败时返回负错误代码。`first`由我们需要的主要号和所需范围的第一个次要号组成。应该使用`MKDEV(ma,mi)`。`count`是所需的连续设备号的数量，`name`应该是相关设备或驱动程序的名称。

+   **动态地**：让内核为我们做这件事，使用`alloc_chrdev_region()`函数。这是获取有效设备号的推荐方法。它的原型如下：

```
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, \ 
                        unsigned int count, char *name); 
```

该方法在成功时返回`0`，在失败时返回负错误代码。`dev`是唯一的输出参数。它代表内核分配的第一个号码。`firstminor`是请求的次要号码范围的第一个，`count`是所需的次要号码数量，`name`应该是相关设备或驱动程序的名称。

两者之间的区别在于，对于前者，我们应该预先知道我们需要什么号码。这是注册：告诉内核我们想要什么设备号。这可能用于教学目的，并且只要驱动程序的唯一用户是您，它就可以工作。但是当要在另一台机器上加载驱动程序时，无法保证所选的号码在该机器上是空闲的，这将导致冲突和麻烦。第二种方法更干净、更安全，因为内核负责为我们猜测正确的号码。我们甚至不必关心在将模块加载到另一台机器上时的行为会是什么，因为内核会相应地进行调整。

无论如何，通常不直接从驱动程序中调用前面的函数，而是通过驱动程序依赖的框架（IIO 框架、输入框架、RTC 等）通过专用 API 进行屏蔽。这些框架在本书的后续章节中都有讨论。

# 设备文件操作简介

可以在文件上执行的操作取决于管理这些文件的驱动程序。这些操作在内核中被定义为`struct file_operations`的实例。`struct file_operations`公开了一组回调函数，这些函数将处理文件上的任何用户空间系统调用。例如，如果希望用户能够对表示我们设备的文件执行`write`操作，就必须实现与`write`函数对应的回调，并将其添加到与您的设备绑定的`struct file_operations`中。让我们填写一个文件操作结构：

```
struct file_operations { 
    struct module *owner; 
    loff_t (*llseek) (struct file *, loff_t, int); 
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *); 
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *); 
    unsigned int (*poll) (struct file *, struct poll_table_struct *); 
    int (*mmap) (struct file *, struct vm_area_struct *); 
    int (*open) (struct inode *, struct file *); 
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long); 
    int (*release) (struct inode *, struct file *); 
    int (*fsync) (struct file *, loff_t, loff_t, int datasync); 
    int (*fasync) (int, struct file *, int); 
    int (*lock) (struct file *, int, struct file_lock *); 
    int (*flock) (struct file *, int, struct file_lock *); 
   [...] 
}; 
```

前面的摘录只列出了结构的重要方法，特别是对本书需求相关的方法。可以在内核源码的`include/linux/fs.h`中找到完整的描述。这些回调函数中的每一个都与系统调用相关联，没有一个是强制性的。当用户代码对给定文件调用与文件相关的系统调用时，内核会寻找负责该文件的驱动程序（特别是创建文件的驱动程序），找到其`struct file_operations`结构，并检查与系统调用匹配的方法是否已定义。如果是，就简单地运行它。如果没有，就返回一个错误代码，这取决于系统调用。例如，未定义的`(*mmap)`方法将返回`-ENODEV`给用户，而未定义的`(*write)`方法将返回`-EINVAL`。

# 内核中的文件表示

内核将文件描述为`struct inode`的实例（而不是`struct file`），该结构在`include/linux/fs.h`中定义：

```
struct inode { 
    [...] 
   struct pipe_inode_info *i_pipe;     /* Set and used if this is a 
 *linux kernel pipe */ 
   struct block_device *i_bdev;  /* Set and used if this is a 
 * a block device */ 
   struct cdev       *i_cdev;    /* Set and used if this is a 
 * character device */ 
    [...] 
} 
```

`struct inode`是一个文件系统数据结构，保存着关于文件（无论其类型是字符、块、管道等）或目录（是的！从内核的角度来看，目录是一个文件，它指向其他文件）的与操作系统相关的信息。

`struct file`结构（也在`include/linux/fs.h`中定义）实际上是内核中表示打开文件的更高级别的文件描述，它依赖于较低级别的`struct inode`数据结构：

```
struct file { 
   [...] 
   struct path f_path;                /* Path to the file */ 
   struct inode *f_inode;             /* inode associated to this file */ 
   const struct file_operations *f_op;/* operations that can be 
          * performed on this file 
          */ 
   loff_t f_pos;                       /* Position of the cursor in 
 * this file */ 
   /* needed for tty driver, and maybe others */ 
   void *private_data;     /* private data that driver can set 
                            * in order to share some data between file 
                            * operations. This can point to any data 
                            * structure. 
 */ 
[...] 
} 
```

`struct inode`和`struct file`之间的区别在于 inode 不跟踪文件内的当前位置或当前模式。它只包含帮助操作系统找到底层文件结构（管道、目录、常规磁盘文件、块/字符设备文件等）内容的东西。另一方面，`struct file`被用作通用结构（实际上它持有一个指向`struct inode`结构的指针），代表并打开文件并提供一组与在底层文件结构上执行的方法相关的函数。这些方法包括：`open`，`write`，`seek`，`read`，`select`等。所有这些都强调了 UNIX 系统的哲学，即*一切皆为文件*。

换句话说，`struct inode`代表内核中的一个文件，`struct file`描述了它在实际打开时的情况。可能有不同的文件描述符代表同一个文件被多次打开，但这些将指向相同的 inode。

# 分配和注册字符设备

在内核中，字符设备被表示为`struct cdev`的实例。当编写字符设备驱动程序时，您的目标是最终创建并注册与`struct file_operations`相关联的该结构的实例，暴露一组用户空间可以对设备执行的操作（函数）。为了实现这个目标，我们必须经历一些步骤，如下所示：

1.  使用`alloc_chrdev_region()`保留一个主设备号和一系列次设备号。

1.  使用`class_create()`为您的设备创建一个类，在`/sys/class/`中可见。

1.  设置一个`struct file_operation`（要提供给`cdev_init`），并为每个需要创建的设备调用`cdev_init()`和`cdev_add()`来注册设备。

1.  然后为每个设备创建一个`device_create()`，并赋予一个适当的名称。这将导致您的设备在`/dev`目录中被创建：

```
#define EEP_NBANK 8 
#define EEP_DEVICE_NAME "eep-mem" 
#define EEP_CLASS "eep-class" 

struct class *eep_class; 
struct cdev eep_cdev[EEP_NBANK]; 
dev_t dev_num; 

static int __init my_init(void) 
{ 
    int i; 
    dev_t curr_dev; 

    /* Request the kernel for EEP_NBANK devices */ 
    alloc_chrdev_region(&dev_num, 0, EEP_NBANK, EEP_DEVICE_NAME); 

    /* Let's create our device's class, visible in /sys/class */ 
    eep_class = class_create(THIS_MODULE, EEP_CLASS); 

    /* Each eeprom bank represented as a char device (cdev)   */ 
    for (i = 0; i < EEP_NBANK; i++) { 

        /* Tie file_operations to the cdev */ 
        cdev_init(&my_cdev[i], &eep_fops); 
        eep_cdev[i].owner = THIS_MODULE; 

        /* Device number to use to add cdev to the core */ 
        curr_dev = MKDEV(MAJOR(dev_num), MINOR(dev_num) + i); 

        /* Now make the device live for the users to access */ 
        cdev_add(&eep_cdev[i], curr_dev, 1); 

        /* create a device node each device /dev/eep-mem0, /dev/eep-mem1, 
         * With our class used here, devices can also be viewed under 
         * /sys/class/eep-class. 
         */ 
        device_create(eep_class, 
                      NULL,     /* no parent device */ 
                      curr_dev, 
                      NULL,     /* no additional data */ 
                      EEP_DEVICE_NAME "%d", i); /* eep-mem[0-7] */ 
    } 
    return 0; 
} 
```

# 编写文件操作

在引入上述文件操作之后，是时候实现它们以增强驱动程序的功能并将设备的方法暴露给用户空间（通过系统调用）。这些方法各有其特点，我们将在本节中进行重点介绍。

# 在内核空间和用户空间之间交换数据

本节不描述任何驱动程序文件操作，而是介绍一些内核设施，可以用来编写这些驱动程序方法。驱动程序的`write()`方法包括从用户空间读取数据到内核空间，然后从内核处理该数据。这样的处理可能是像*推送*数据到设备一样。另一方面，驱动程序的`read()`方法包括将数据从内核复制到用户空间。这两种方法都引入了我们需要在跳转到各自步骤之前讨论的新元素。第一个是`__user`。`__user`是由稀疏（内核用于查找可能的编码错误的语义检查器）使用的一个标记，用于让开发人员知道他实际上将要不正确地使用一个不受信任的指针（或者在当前虚拟地址映射中可能无效的指针），并且他不应该解引用，而应该使用专用的内核函数来访问该指针指向的内存。

这使我们能够引入不同的内核函数，以便访问这样的内存，无论是读取还是写入。这些分别是`copy_from_user()`和`copy_from_user()`，用于将缓冲区从用户空间复制到内核空间，反之亦然，将缓冲区从内核复制到用户空间：

```
unsigned long copy_from_user(void *to, const void __user *from, 
                             unsigned long n) 
unsigned long copy_to_user(void __user *to, const void *from, 
                              unsigned long n) 
```

在这两种情况下，以`__user`为前缀的指针指向用户空间（不受信任）内存。`n`代表要复制的字节数。`from`代表源地址，`to`是目标地址。这些返回未能复制的字节数。成功时，返回值应为`0`。

请注意，使用`copy_to_user（）`，如果无法复制某些数据，函数将使用零字节填充已复制的数据以达到请求的大小。

# 单个值复制

在复制`char`和`int`等单个和简单变量时，但不是在复制结构或数组等较大的数据类型时，内核提供了专用宏以快速执行所需的操作。这些宏是`put_user(x, ptr)`和`get_used(x, ptr)`，解释如下：

+   `put_user(x, ptr);`：此宏将变量从内核空间复制到用户空间。`x`表示要复制到用户空间的值，`ptr`是用户空间中的目标地址。该宏在成功时返回`0`，在错误时返回`-EFAULT`。`x`必须可分配给解引用`ptr`的结果。换句话说，它们必须具有（或指向）相同的类型。

+   `get_user(x, ptr);`：此宏将变量从用户空间复制到内核空间，并在成功时返回`0`，在错误时返回`-EFAULT`。请注意，错误时`x`设置为`0`。`x`表示要存储结果的内核变量，`ptr`是用户空间中的源地址。解引用`ptr`的结果必须可分配给`x`而不需要转换。猜猜它是什么意思。

# 打开方法

`open`是每次有人打开设备文件时调用的方法。如果未定义此方法，则设备打开将始终成功。通常使用此方法来执行设备和数据结构初始化，并在出现问题时返回负错误代码，或`0`。`open`方法的原型定义如下：

```
int (*open)(struct inode *inode, struct file *filp); 
```

# 每个设备的数据

对于在您的字符设备上执行的每个`open`，回调函数将以`struct inode`作为参数，该参数是文件的内核底层表示。该`struct inode`结构具有一个名为`i_cdev`的字段，指向我们在`init`函数中分配的`cdev`。通过在以下示例中的`struct pcf2127`中将`struct cdev`嵌入到我们的设备特定数据中，我们将能够使用`container_of`宏获取指向该特定数据的指针。以下是一个`open`方法示例。

以下是我们的数据结构：

```
struct pcf2127 { 
    struct cdev cdev; 
    unsigned char *sram_data; 
    struct i2c_client *client; 
    int sram_size; 
    [...] 
}; 
```

根据这个数据结构，`open`方法将如下所示：

```
static unsigned int sram_major = 0; 
static struct class *sram_class = NULL; 

static int sram_open(struct inode *inode, struct file *filp) 
{ 
   unsigned int maj = imajor(inode); 
   unsigned int min = iminor(inode); 

   struct pcf2127 *pcf = NULL; 
   pcf = container_of(inode->i_cdev, struct pcf2127, cdev); 
   pcf->sram_size = SRAM_SIZE; 

   if (maj != sram_major || min < 0 ){ 
         pr_err ("device not found\n"); 
         return -ENODEV; /* No such device */ 
   } 

   /* prepare the buffer if the device is opened for the first time */ 
   if (pcf->sram_data == NULL) { 
         pcf->sram_data = kzalloc(pcf->sram_size, GFP_KERNEL); 
         if (pcf->sram_data == NULL) { 
               pr_err("Open: memory allocation failed\n"); 
               return -ENOMEM; 
         } 
   } 
   filp->private_data = pcf; 
   return 0; 
} 
```

# 释放方法

当设备关闭时，将调用`release`方法，这是`open`方法的反向操作。然后，您必须撤消在打开任务中所做的一切。您大致要做的是：

1.  释放在“open（）”步骤中分配的任何私有内存。

1.  关闭设备（如果支持），并在最后关闭时丢弃每个缓冲区（如果设备支持多次打开，或者驱动程序可以同时处理多个设备）。

以下是`release`函数的摘录：

```
static int sram_release(struct inode *inode, struct file *filp) 
{ 
   struct pcf2127 *pcf = NULL; 
   pcf = container_of(inode->i_cdev, struct pcf2127, cdev); 

   mutex_lock(&device_list_lock); 
   filp->private_data = NULL; 

   /* last close? */ 
   pcf2127->users--; 
   if (!pcf2127->users) { 
         kfree(tx_buffer); 
         kfree(rx_buffer); 
         tx_buffer = NULL; 
         rx_buffer = NULL; 

         [...] 

         if (any_global_struct) 
               kfree(any_global_struct); 
   } 
   mutex_unlock(&device_list_lock); 

   return 0; 
} 
```

# 写入方法

“write（）”方法用于向设备发送数据；每当用户应用程序在设备文件上调用`write`函数时，将调用内核实现。其原型如下：

```
ssize_t(*write)(struct file *filp, const char __user *buf, size_t count, loff_t *pos); 
```

+   返回值是写入的字节数（大小）

+   `*buf`表示来自用户空间的数据缓冲区

+   `count`是请求传输的大小

+   `*pos`表示应在文件中写入数据的起始位置

# 写入步骤

以下步骤不描述任何标准或通用的方法来实现驱动程序的“write（）”方法。它们只是概述了在此方法中可以执行的操作类型。

1.  检查来自用户空间的错误或无效请求。如果设备公开其内存（eeprom、I/O 内存等），可能存在大小限制，则此步骤才相关：

```
/* if trying to Write beyond the end of the file, return error. 
 * "filesize" here corresponds to the size of the device memory (if any) 
 */ 
if ( *pos >= filesize ) return -EINVAL; 
```

1.  调整`count`以便不超出文件大小的剩余字节。这一步骤不是强制性的，与步骤 1 的条件相同：

```
/* filesize coerresponds to the size of device memory */ 
if (*pos + count > filesize)  
    count = filesize - *pos; 
```

1.  找到要开始写入的位置。如果设备具有内存，供“write（）”方法写入给定数据，则此步骤才相关。与步骤 2 和 3 一样，此步骤不是强制性的：

```
/* convert pos into valid address */ 
void *from = pos_to_address( *pos );  
```

1.  从用户空间复制数据并将其写入适当的内核空间：

```
if (copy_from_user(dev->buffer, buf, count) != 0){ 
    retval = -EFAULT; 
    goto out; 
} 
/* now move data from dev->buffer to physical device */ 
```

1.  写入物理设备并在失败时返回错误：

```
write_error = device_write(dev->buffer, count); 
if ( write_error ) 
    return -EFAULT; 
```

1.  根据写入的字节数增加文件中光标的当前位置。最后，返回复制的字节数：

```
*pos += count; 
Return count; 
```

以下是`write`方法的一个示例。再次强调，这旨在给出一个概述：

```
ssize_t  
eeprom_write(struct file *filp, const char __user *buf, size_t count, 
   loff_t *f_pos) 
{ 
   struct eeprom_dev *eep = filp->private_data; 
   ssize_t retval = 0; 

    /* step (1) */ 
    if (*f_pos >= eep->part_size)  
        /* Writing beyond the end of a partition is not allowed. */ 
        return -EINVAL; 

    /* step (2) */ 
    if (*pos + count > eep->part_size) 
        count = eep->part_size - *pos; 

   /* step (3) */ 
   int part_origin = PART_SIZE * eep->part_index; 
   int register_address = part_origin + *pos; 

    /* step(4) */ 
    /* Copy data from user space to kernel space */ 
    if (copy_from_user(eep->data, buf, count) != 0) 
        return -EFAULT; 

       /* step (5) */ 
    /* perform the write to the device */ 
    if (write_to_device(register_address, buff, count) < 0){ 
        pr_err("ee24lc512: i2c_transfer failed\n");   
        return -EFAULT; 
     } 

    /* step (6) */ 
    *f_pos += count; 
    return count; 
} 
```

# 读取方法

`read()`方法的原型如下：

```
ssize_t (*read) (struct file *filp, char __user *buf, size_t count, loff_t *pos);
```

返回值是读取的大小。方法的其余元素在这里描述：

+   `*buf`是我们从用户空间接收的缓冲区

+   `count` 是请求传输的大小（用户缓冲区的大小）

+   `*pos`指示应从文件中读取数据的起始位置

# 读取步骤

1.  防止读取超出文件大小，并返回文件末尾：

```
if (*pos >= filesize) 
  return 0; /* 0 means EOF */ 
```

1.  读取的字节数不能超过文件大小。相应地调整`count`：

```
if (*pos + count > filesize) 
    count = filesize - (*pos); 
```

1.  找到将开始读取的位置：

```
void *from = pos_to_address (*pos); /* convert pos into valid address */ 
```

1.  将数据复制到用户空间缓冲区，并在失败时返回错误：

```
sent = copy_to_user(buf, from, count); 
if (sent) 
    return -EFAULT; 
```

1.  根据读取的字节数提前文件的当前位置，并返回复制的字节数：

```
*pos += count; 
Return count; 
```

以下是一个驱动程序`read()`文件操作的示例，旨在概述可以在那里完成的工作：

```
ssize_t  eep_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos) 
{ 
    struct eeprom_dev *eep = filp->private_data; 

    if (*f_pos >= EEP_SIZE) /* EOF */ 
        return 0; 

    if (*f_pos + count > EEP_SIZE) 
        count = EEP_SIZE - *f_pos; 

    /* Find location of next data bytes */ 
    int part_origin  =  PART_SIZE * eep->part_index; 
    int eep_reg_addr_start  =  part_origin + *pos; 

    /* perform the read from the device */ 
    if (read_from_device(eep_reg_addr_start, buff, count) < 0){ 
        pr_err("ee24lc512: i2c_transfer failed\n");   
        return -EFAULT; 
    }  

    /* copy from kernel to user space */ 
    if(copy_to_user(buf, dev->data, count) != 0) 
        return -EIO; 

    *f_pos += count; 
    return count; 
} 
```

# llseek 方法

当在文件内移动光标位置时，将调用`llseek`函数。该方法在用户空间的入口点是`lseek()`。可以参考 man 页面以打印用户空间中任一方法的完整描述：`man llseek`和`man lseek`。其原型如下：

```
loff_t(*llseek) (structfile *filp, loff_t offset, int whence); 
```

+   返回值是文件中的新位置

+   `loff_t`是相对于当前文件位置的偏移量，定义了它将被改变多少

+   `whence`定义了从哪里寻找。可能的值有：

+   `SEEK_SET`：这将光标放置在相对于文件开头的位置

+   `SEEK_CUR`：这将光标放置在相对于当前文件位置的位置

+   `SEEK_END`：这将光标调整到相对于文件末尾的位置

# llseek 步骤

1.  使用`switch`语句检查每种可能的`whence`情况，因为它们是有限的，并相应地调整`newpos`：

```
switch( whence ){ 
    case SEEK_SET:/* relative from the beginning of file */ 
        newpos = offset; /* offset become the new position */ 
        break; 
    case SEEK_CUR: /* relative to current file position */ 
        newpos = file->f_pos + offset; /* just add offset to the current position */ 
        break; 
    case SEEK_END: /* relative to end of file */ 
        newpos = filesize + offset; 
        break; 
    default: 
        return -EINVAL; 
} 
```

1.  检查`newpos`是否有效：

```
if ( newpos < 0 ) 
    return -EINVAL; 
```

1.  使用新位置更新`f_pos`：

```
filp->f_pos = newpos; 
```

1.  返回新的文件指针位置：

```
return newpos; 
```

以下是一个连续读取和搜索文件的用户程序示例。底层驱动程序将执行`llseek()`文件操作入口：

```
#include <unistd.h> 
#include <fcntl.h> 
#include <sys/types.h> 
#include <stdio.h> 

#define CHAR_DEVICE "toto.txt" 

int main(int argc, char **argv) 
{ 
    int fd= 0; 
    char buf[20]; 

    if ((fd = open(CHAR_DEVICE, O_RDONLY)) < -1) 
        return 1; 

    /* Read 20 bytes */ 
    if (read(fd, buf, 20) != 20) 
        return 1; 
    printf("%s\n", buf); 

    /* Move the cursor to 10 time, relative to its actual position */ 
    if (lseek(fd, 10, SEEK_CUR) < 0) 
        return 1; 
    if (read(fd, buf, 20) != 20)  
        return 1; 
    printf("%s\n",buf); 

    /* Move the cursor ten time, relative from the beginig of the file */ 
    if (lseek(fd, 7, SEEK_SET) < 0) 
        return 1; 
    if (read(fd, buf, 20) != 20) 
        return 1; 
    printf("%s\n",buf); 

    close(fd); 
    return 0; 
} 
```

代码产生以下输出：

```
jma@jma:~/work/tutos/sources$ cat toto.txt

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.

jma@jma:~/work/tutos/sources$ ./seek

Lorem ipsum dolor si

nsectetur adipiscing

psum dolor sit amet,

jma@jma:~/work/tutos/sources$

```

# 轮询方法

如果需要实现被动等待（在感知字符设备时不浪费 CPU 周期），必须实现`poll()`函数，每当用户空间程序对与设备关联的文件执行`select()`或`poll()`系统调用时都会调用该函数：

```
unsigned int (*poll) (struct file *, struct poll_table_struct *); 
```

这个方法的核心是`poll_wait()`内核函数，定义在`<linux/poll.h>`中，这是驱动程序代码中应该包含的头文件：

```
void poll_wait(struct file * filp, wait_queue_head_t * wait_address, 
poll_table *p) 
```

`poll_wait()`将与`struct file`结构（作为第一个参数给出）相关联的设备添加到可以唤醒进程的列表中（这些进程已经在`struct wait_queue_head_t`结构中休眠，该结构作为第二个参数给出），根据在`struct poll_table`结构中注册的事件（作为第三个参数给出）。用户进程可以运行`poll()`，`select()`或`epoll()`系统调用，将一组文件添加到等待的列表中，以便了解相关（如果有）设备的准备情况。然后内核将调用与每个设备文件相关联的驱动程序的`poll`入口。然后，每个驱动程序的`poll`方法应调用`poll_wait()`以注册进程需要被内核通知的事件，将该进程置于休眠状态，直到其中一个事件发生，并将驱动程序注册为可以唤醒该进程的驱动程序之一。通常的方法是根据`select()`（或`poll()`）系统调用支持的事件类型使用一个等待队列（一个用于可读性，另一个用于可写性，如果需要的话，最终还有一个用于异常）。

`(*poll)`文件操作的返回值必须设置为`POLLIN | POLLRDNORM`，如果有数据可读（在调用 select 或 poll 时），如果设备可写，则设置为`POLLOUT | POLLWRNORM`（在这里也是调用 select 或 poll），如果没有新数据且设备尚未可写，则设置为`0`。在下面的示例中，我们假设设备同时支持阻塞读和写。当然，可以只实现其中一个。如果驱动程序没有定义此方法，则设备将被视为始终可读和可写，因此`poll()`或`select()`系统调用会立即返回。

# 轮询步骤

当实现`poll`函数时，`read`或`write`方法中的任何一个都可能会发生变化：

1.  为需要实现被动等待的每种事件类型（读取、写入、异常）声明一个等待队列，当没有数据可读或设备尚不可写时，将任务放入其中：

```
static DECLARE_WAIT_QUEUE_HEAD(my_wq); 
static DECLARE_WAIT_QUEUE_HEAD(my_rq); 
```

1.  实现`poll`函数如下：

```
#include <linux/poll.h> 
static unsigned int eep_poll(struct file *file, poll_table *wait) 
{ 
    unsigned int reval_mask = 0; 
    poll_wait(file, &my_wq, wait); 
    poll_wait(file, &my_rq, wait); 

    if (new-data-is-ready) 
        reval_mask |= (POLLIN | POLLRDNORM); 
    if (ready_to_be_written) 
       reval_mask |= (POLLOUT | POLLWRNORM); 
    return reval_mask; 
} 
```

1.  当有新数据或设备可写时，通知等待队列：

```
wake_up_interruptible(&my_rq); /* Ready to read */ 
wake_up_interruptible(&my_wq); /* Ready to be written to */ 
```

可以从驱动程序的`write()`方法内部或者从 IRQ 处理程序内部通知可读事件，这意味着写入的数据可以被读取，或者从 IRQ 处理程序内部通知可写事件，这意味着设备已完成数据发送操作，并准备好再次接受数据。

在使用阻塞 I/O 时，`read`或`write`方法中的任何一个都可能会发生变化。在`poll`中使用的等待队列也必须在读取时使用。当用户需要读取时，如果有数据，该数据将立即发送到进程，并且必须更新等待队列条件（设置为`false`）；如果没有数据，进程将在等待队列中休眠。

如果`write`方法应该提供数据，那么在`write`回调中，您必须填充数据缓冲区并更新等待队列条件（设置为`true`），并唤醒读取者（参见*等待队列*部分）。如果是 IRQ，这些操作必须在其处理程序中执行。

以下是对在给定字符设备上进行`select()`以检测数据可用性的代码的摘录：

```
#include <unistd.h> 
#include <fcntl.h> 
#include <stdio.h> 
#include <stdlib.h> 
#include <sys/select.h> 

#define NUMBER_OF_BYTE 100 
#define CHAR_DEVICE "/dev/packt_char" 

char data[NUMBER_OF_BYTE]; 

int main(int argc, char **argv) 
{ 
    int fd, retval; 
    ssize_t read_count; 
    fd_set readfds; 

    fd = open(CHAR_DEVICE, O_RDONLY); 
    if(fd < 0) 
        /* Print a message and exit*/ 
        [...] 

    while(1){  
        FD_ZERO(&readfds); 
        FD_SET(fd, &readfds); 

        /* 
         * One needs to be notified of "read" events only, without timeout. 
         * This call will put the process to sleep until it is notified the 
         * event for which it registered itself 
         */ 
        ret = select(fd + 1, &readfds, NULL, NULL, NULL); 

        /* From this line, the process has been notified already */ 
        if (ret == -1) { 
            fprintf(stderr, "select call on %s: an error ocurred", CHAR_DEVICE); 
            break; 
        } 

        /* 
         * file descriptor is now ready. 
         * This step assume we are interested in one file only. 
         */ 
        if (FD_ISSET(fd, &readfds)) { 
            read_count = read(fd, data, NUMBER_OF_BYTE); 
            if (read_count < 0 ) 
                /* An error occured. Handle this */ 
                [...] 

            if (read_count != NUMBER_OF_BYTE) 
                /* We have read less than need bytes */ 
                [...] /* handle this */ 
            else 
            /* Now we can process data we have read */ 
            [...] 
        } 
    }     
    close(fd); 
    return EXIT_SUCCESS; 
} 
```

# ioctl 方法

典型的 Linux 系统包含大约 350 个系统调用（syscalls），但只有少数与文件操作相关。有时设备可能需要实现特定的命令，这些命令不是由系统调用提供的，特别是与文件相关的命令，因此是设备文件。在这种情况下，解决方案是使用**输入/输出控制**（**ioctl**），这是一种方法，通过它可以扩展与设备相关的系统调用（实际上是命令）的列表。可以使用它向设备发送特殊命令（`reset`，`shutdown`，`configure`等）。如果驱动程序没有定义此方法，内核将对任何`ioctl()`系统调用返回`-ENOTTY`错误。

为了有效和安全，一个`ioctl`命令需要由一个数字标识，这个数字应该对系统是唯一的。在整个系统中 ioctl 号的唯一性将防止它向错误的设备发送正确的命令，或者向正确的命令传递错误的参数（给定重复的 ioctl 号）。Linux 提供了四个辅助宏来创建`ioctl`标识符，具体取决于是否有数据传输，以及传输的方向。它们的原型分别是：

```
_IO(MAGIC, SEQ_NO) 
_IOW(MAGIC, SEQ_NO, TYPE) 
_IOR(MAGIC, SEQ_NO, TYPE) 
_IORW(MAGIC, SEQ_NO, TYPE) 
```

它们的描述如下：

+   `_IO`：`ioctl`不需要数据传输

+   `_IOW`：`ioctl`需要写参数（`copy_from_user`或`get_user`）

+   `_IOR`：`ioctl`需要读参数（`copy_to_user`或`put_user`）

+   `_IOWR`：`ioctl`需要写和读参数

它们的参数意义（按照它们传递的顺序）在这里描述：

1.  一个编码为 8 位（0 到 255）的数字，称为魔术数字。

1.  一个序列号或命令 ID，也是 8 位。

1.  一个数据类型（如果有的话），将通知内核要复制的大小。

在内核源中的*Documentation/ioctl/ioctl-decoding.txt*中有很好的文档，现有的`ioctl`在*Documentation/ioctl/ioctl-number.txt*中列出，这是需要创建`ioctl`命令时的好起点。

# 生成 ioctl 号（命令）

应该在专用的头文件中生成自己的 ioctl 号。这不是强制性的，但建议这样做，因为这个头文件也应该在用户空间中可用。换句话说，应该复制 ioctl 头文件，以便内核和用户空间各有一个，用户可以在用户应用程序中包含其中。现在让我们在一个真实的例子中生成 ioctl 号：

`eep_ioctl.h`：

```
#ifndef PACKT_IOCTL_H 
#define PACKT_IOCTL_H 
/* 
 * We need to choose a magic number for our driver, and sequential numbers 
 * for each command: 
 */ 
#define EEP_MAGIC 'E' 
#define ERASE_SEQ_NO 0x01 
#define RENAME_SEQ_NO 0x02 
#define ClEAR_BYTE_SEQ_NO 0x03 
#define GET_SIZE 0x04 

/* 
 * Partition name must be 32 byte max 
 */ 
#define MAX_PART_NAME 32 

/* 
 * Now let's define our ioctl numbers: 
 */ 
#define EEP_ERASE _IO(EEP_MAGIC, ERASE_SEQ_NO) 
#define EEP_RENAME_PART _IOW(EEP_MAGIC, RENAME_SEQ_NO, unsigned long) 
#define EEP_GET_SIZE _IOR(EEP_MAGIC, GET_SIZE, int *) 
#endif 
```

# `ioctl`的步骤

首先，让我们看一下它的原型。它看起来如下：

```
long ioctl(struct file *f, unsigned int cmd, unsigned long arg); 
```

只有一步：使用`switch ... case`语句，并在调用未定义的`ioctl`命令时返回`-ENOTTY`错误。可以在[`man7.org/linux/man-pages/man2/ioctl.2.html`](http://man7.org/linux/man-pages/man2/ioctl.2.html)找到更多信息：

```
/* 
 * User space code also need to include the header file in which ioctls 
 * defined are defined. This is eep_ioctl.h in our case. 
 */ 
#include "eep_ioctl.h" 
static long eep_ioctl(struct file *f, unsigned int cmd, unsigned long arg) 
{ 
    int part; 
    char *buf = NULL; 
    int size = 1300; 

    switch(cmd){ 
        case EEP_ERASE: 
            erase_eepreom(); 
            break; 
        case EEP_RENAME_PART: 
            buf = kmalloc(MAX_PART_NAME, GFP_KERNEL); 
            copy_from_user(buf, (char *)arg, MAX_PART_NAME); 
            rename_part(buf); 
            break; 
        case EEP_GET_SIZE: 
            copy_to_user((int*)arg, &size, sizeof(int)); 
            break; 
        default: 
            return -ENOTTY; 
    } 
    return 0; 
} 
```

如果您认为您的`ioctl`命令需要多个参数，您应该将这些参数收集在一个结构中，并只是将结构中的指针传递给`ioctl`。

现在，从用户空间，您必须使用与驱动程序代码中相同的`ioctl`头文件：

`my_main.c`

```
#include <stdio.h> 
#include <stdlib.h> 
#include <fcntl.h> 
#include <unistd.h> 
#include "eep_ioctl.h"  /* our ioctl header file */ 

int main() 
{ 
    int size = 0; 
    int fd; 
    char *new_name = "lorem_ipsum"; /* must not be longer than MAX_PART_NAME */ 

    fd = open("/dev/eep-mem1", O_RDWR); 
    if (fd == -1){ 
        printf("Error while opening the eeprom\n"); 
        return -1; 
    } 

    ioctl(fd, EEP_ERASE);  /* ioctl call to erase partition */ 
    ioctl(fd, EEP_GET_SIZE, &size); /* ioctl call to get partition size */ 
    ioctl(fd, EEP_RENAME_PART, new_name);  /* ioctl call to rename partition */ 

    close(fd); 
    return 0; 
} 
```

# 填充 file_operations 结构

在编写内核模块时，最好在静态初始化结构及其参数时使用指定的初始化器。它包括命名需要分配值的成员。形式是`.member-name`来指定应初始化的成员。这允许以未定义的顺序初始化成员，或者保持不想修改的字段不变，等等。

一旦我们定义了我们的函数，我们只需填充结构如下：

```
static const struct file_operations eep_fops = { 
   .owner =    THIS_MODULE, 
   .read =     eep_read, 
   .write =    eep_write, 
   .open =     eep_open, 
   .release =  eep_release, 
   .llseek =   eep_llseek, 
   .poll =     eep_poll, 
   .unlocked_ioctl = eep_ioctl, 
}; 
```

让我们记住，结构作为参数传递给`cdev_init`的`init`方法。

# 总结

在本章中，我们已经揭开了字符设备的神秘面纱，看到了如何通过设备文件让用户与我们的驱动程序进行交互。我们学会了如何将文件操作暴露给用户空间，并从内核内部控制它们的行为。我们甚至可以实现多设备支持。下一章有点偏向硬件，因为它涉及到将硬件设备的功能暴露给用户空间的平台驱动程序。字符驱动程序与平台驱动程序的结合力量简直令人惊叹。下一章见。
