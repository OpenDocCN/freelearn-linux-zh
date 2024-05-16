# 第七章：高级字符驱动程序操作

在之前的章节中，我们学到了一些在设备驱动程序开发中很有用的东西；然而，还需要一步。我们必须看看如何为我们的字符设备添加高级功能，并充分理解如何将用户空间进程与外围 I/O 活动同步。

在本章中，我们将看到如何为`lseek()`、`ioctl()`和`mmap()`函数实现系统调用，并且我们还将了解几种技术来使进程进入睡眠状态，以防我们的外围设备尚未有数据返回；因此，在本章中，我们将涵盖以下配方：

+   在文件中上下移动 lseek()

+   使用 ioctl()进行自定义命令

+   使用 mmap()访问 I/O 内存

+   与进程上下文锁定

+   锁定（和同步）中断上下文

+   使用 poll()和 select()等待 I/O 操作

+   使用 fasync()管理异步通知

# 技术要求

有关更多信息，请查看本章的附录部分。

本章中使用的代码和其他文件可以从 GitHub 上下载，网址为[`github.com/giometti/linux_device_driver_development_cookbook/tree/master/chapter_07`](https://github.com/giometti/linux_device_driver_development_cookbook/tree/master/chapter_07)。

# 在文件中上下移动 lseek()

在这个配方中，我们将更仔细地看一下如何操纵`ppos`指针（在第三章中的*与字符驱动程序交换数据*配方中描述），这与`read()`和`write()`系统调用的实现有关。

# 准备就绪

为了提供`lseek()`实现的一个简单示例，我们可以在第四章中的`chapter_04/chrdev`目录中重用我们的`chrdev`驱动程序（我们需要 GitHub 存储库的`chrdev.c`和`chrdev-req.c`文件），在那里我们可以根据我们的设备内存布局简单地添加我们的自定义`llseek()`方法。

为简单起见，我只是将这些文件复制到`chapter_07/chrdev/`目录中，并对其进行了重新处理。

我们还需要修改 ESPRESSObin 的 DTS 文件，就像我们在第四章中使用`chapter_04/chrdev/add_chrdev_devices.dts.patch`文件一样，以启用 chrdev 设备，然后最后，我们可以重用第三章中创建的`chrdev_test.c`程序，作为我们的`lseek()`实现测试的基本程序。

关于 ESPRESSObin 的 DTS 文件，我们可以通过进入内核源并执行`patch`命令来修补它，如下所示：

```
$ patch -p1 < ../github/chapter_04/chrdev/add_chrdev_devices.dts.patch 
patching file arch/arm64/boot/dts/marvell/armada-3720-espressobin.dts
```

然后，我们必须重新编译内核，并像我们在第一章中所做的那样，使用前述的 DTS 重新安装内核，最后，重新启动系统。

# 如何做...

让我们看看如何通过以下步骤来做到这一点：

1.  首先，我们可以通过添加我们的`chrdev_llseek`方法来简单地重新定义`struct file_operations`：

```
static const struct file_operations chrdev_fops = {
    .owner   = THIS_MODULE,
    .llseek  = chrdev_llseek,
    .read    = chrdev_read,
    .write   = chrdev_write,
    .open    = chrdev_open,
    .release = chrdev_release
};
```

1.  然后，我们通过使用一个大开关来定义方法的主体，根据驱动程序的内存布局来处理`SEEK_SET`、`SEEK_CUR`和`SEEK_END`可能的值：

```
static loff_t chrdev_llseek(struct file *filp, loff_t offset, int whence)
{
    struct chrdev_device *chrdev = filp->private_data;
    loff_t newppos;

    dev_info(chrdev->dev, "should move *ppos=%lld by whence %d off=%lld\n",
                filp->f_pos, whence, offset);

    switch (whence) {
    case SEEK_SET:
        newppos = offset; 
        break;

    case SEEK_CUR:
        newppos = filp->f_pos + offset; 
        break;

    case SEEK_END:
        newppos = BUF_LEN + offset; 
        break;

    default:
        return -EINVAL;
    }
```

1.  最后，我们必须验证`newppos`是否仍在 0 和`BUF_LEN`之间，并且在肯定的情况下，我们必须更新`filp->f_pos`为`newppos`值，如下所示：

```
    if ((newppos < 0) || (newppos >= BUF_LEN))
        return -EINVAL;

    filp->f_pos = newppos;
    dev_info(chrdev->dev, "return *ppos=%lld\n", filp->f_pos);

    return newppos;
}
```

请注意，可以从 GitHub 源中的`chapter_07/`目录中检索到`chrdev.c`驱动程序的新版本，该目录与本章相关。

# 它是如何工作的...

在*步骤 2*中，我们应该记住每个设备有一个`BUF_LEN`字节的内存缓冲区，因此我们可以通过执行一些简单的操作来计算设备内的新`newppos`位置。

因此，对于`SEEK_SET`，将`ppos`设置为`offset`，我们可以简单地执行赋值操作；对于`SEEK_CUR`，将`ppos`从其当前位置（即`filp->f_pos`）加上`offset`字节，我们执行求和操作；最后，对于`SEEK_END`，将`ppos`设置为文件末尾加上`offset`字节，我们仍然执行与`BUF_LEN`缓冲区大小的求和操作，因为我们期望从用户空间得到负值或零。

# 还有更多...

如果您现在希望测试`lseek()`系统调用，我们可以修改之前报告的`chrdev_test.c`程序，然后尝试在我们的新驱动程序版本上执行它。

因此，让我们使用`modify_lseek_to_chrdev_test.patch`文件修改`chrdev_test.c`，如下所示：

```
$ cd github/chapter_03/
$ patch -p2 < ../chapter_07/chrdev/modify_lseek_to_chrdev_test.patch 
```

然后，我们必须重新编译它，如下所示：

```
$ make CFLAGS="-Wall -O2" \
 CC=aarch64-linux-gnu-gcc \
 chrdev_test
aarch64-linux-gnu-gcc -Wall -O2 chrdev_test.c -o chrdev_test
```

请注意，可以通过简单地删除`CC=aarch64-linux-gnu-gcc`设置在 ESPRESSObin 上执行此命令。

然后，我们必须将新的`chrdev_test`可执行文件和具有`lseek()`支持的`chrdev.ko`（以及`chrdev-req.ko`内核模块）移动到 ESPRESSObin，然后将它们插入内核：

```
# insmod chrdev.ko 
chrdev:chrdev_init: got major 239
# insmod chrdev-req.ko
chrdev cdev-eeprom@2: chrdev cdev-eeprom with id 2 added
chrdev cdev-rom@4: chrdev cdev-rom with id 4 added

```

这个输出来自串行控制台，因此我们也会得到内核消息。如果您通过 SSH 连接执行这些命令，您将得不到输出，您将不得不使用`dmesg`命令来获取前面示例中的输出。

最后，我们可以在一个 chrdev 设备上执行`chrdev_test`程序，如下所示：

```
# ./chrdev_test /dev/cdev-eeprom\@2 
file /dev/cdev-eeprom@2 opened
wrote 11 bytes into file /dev/cdev-eeprom@2
data written are: 44 55 4d 4d 59 20 44 41 54 41 00 
*ppos moved to 0
read 11 bytes from file /dev/cdev-eeprom@2
data read are: 44 55 4d 4d 59 20 44 41 54 41 00 
```

正如预期的那样，`lseek()`系统调用调用了驱动程序的`chrdev_llseek()`方法，这正是我们所期望的。与前述命令相关的内核消息如下所示：

```
chrdev cdev-eeprom@2: chrdev (id=2) opened
chrdev cdev-eeprom@2: should write 11 bytes (*ppos=0)
chrdev cdev-eeprom@2: got 11 bytes (*ppos=11)
chrdev cdev-eeprom@2: should move *ppos=11 by whence 0 off=0
chrdev cdev-eeprom@2: return *ppos=0
chrdev cdev-eeprom@2: should read 11 bytes (*ppos=0)
chrdev cdev-eeprom@2: return 11 bytes (*ppos=11)
chrdev cdev-eeprom@2: chrdev (id=2) released
```

因此，当第一个`write()`系统调用执行时，`ppos`从字节 0 移动到字节 11，然后由于`lseek()`的作用又移回到 0，最后由于`read()`系统调用的执行又移动到 11。

请注意，我们还可以使用`dd`命令调用`lseek()`方法，如下所示：

```
# dd if=/dev/cdev-eeprom\@2 skip=11 bs=1 count=3 | od -tx1
3+0 records in
3+0 records out
3 bytes copied, 0.0530299 s, 0.1 kB/s
0000000 00 00 00
0000003

```

在这里，我们打开设备，然后将`ppos`从开头向前移动 11 个字节，然后对每个进行三次 1 字节长度的读取。

在以下内核消息中，我们可以验证`dd`程序的行为与预期完全一致：

```
chrdev cdev-eeprom@2: chrdev (id=2) opened
chrdev cdev-eeprom@2: should move *ppos=0 by whence 1 off=0
chrdev cdev-eeprom@2: return *ppos=0
chrdev cdev-eeprom@2: should move *ppos=0 by whence 1 off=11
chrdev cdev-eeprom@2: return *ppos=11
chrdev cdev-eeprom@2: should read 1 bytes (*ppos=11)
chrdev cdev-eeprom@2: return 1 bytes (*ppos=12)
chrdev cdev-eeprom@2: should read 1 bytes (*ppos=12)
chrdev cdev-eeprom@2: return 1 bytes (*ppos=13)
chrdev cdev-eeprom@2: should read 1 bytes (*ppos=13)
chrdev cdev-eeprom@2: return 1 bytes (*ppos=14)
chrdev cdev-eeprom@2: chrdev (id=2) released
```

# 另请参阅

+   有关`lseek()`系统调用的更多信息，一个很好的起点是它的 man 页面，可以使用`man 2 lseek`命令获取。

# 使用 ioctl()进行自定义命令

在本教程中，我们将看到如何以非常定制的方式添加自定义命令来配置或管理我们的外围设备。

# 准备工作完成

现在，为了展示如何在我们的驱动程序中实现`ioctl()`系统调用的简单示例，我们仍然可以使用之前介绍的 chrdev 驱动程序，在其中添加`unlocked_ioctl()`方法，如后面所述。

# 如何做...

让我们按照以下步骤来做：

1.  首先，我们必须在`chrdev_fops`结构中添加`unlocked_ioctl()`方法：

```
static const struct file_operations chrdev_fops = {
    .owner          = THIS_MODULE,
    .unlocked_ioctl = chrdev_ioctl,
    .llseek         = chrdev_llseek,
    .read           = chrdev_read,
    .write          = chrdev_write,
    .open           = chrdev_open,
    .release        = chrdev_release
};
```

1.  然后，我们添加方法的主体，在开始时，我们进行了一些赋值和检查，如下所示：

```
static long chrdev_ioctl(struct file *filp,
                unsigned int cmd, unsigned long arg)
{
    struct chrdev_device *chrdev = filp->private_data;
    struct chrdev_info info;
    void __user *uarg = (void __user *) arg;
    int __user *iuarg = (int __user *) arg;
    int ret;

    /* Get some command information */
    if (_IOC_TYPE(cmd) != CHRDEV_IOCTL_BASE) {
        dev_err(chrdev->dev, "command %x is not for us!\n", cmd);
        return -EINVAL;
    }
    dev_info(chrdev->dev, "cmd nr=%d size=%d dir=%x\n",
                _IOC_NR(cmd), _IOC_SIZE(cmd), _IOC_DIR(cmd));
```

1.  然后，我们可以实现一个大开关来执行请求的命令，如下所示：

```
    switch (cmd) {
    case CHRDEV_IOC_GETINFO:
        dev_info(chrdev->dev, "CHRDEV_IOC_GETINFO\n");

        strncpy(info.label, chrdev->label, NAME_LEN);
        info.read_only = chrdev->read_only;

        ret = copy_to_user(uarg, &info, sizeof(struct chrdev_info));
        if (ret)
            return -EFAULT;

        break;

    case WDIOC_SET_RDONLY:
        dev_info(chrdev->dev, "WDIOC_SET_RDONLY\n");

        ret = get_user(chrdev->read_only, iuarg); 
        if (ret)
            return -EFAULT;

        break;

    default:
        return -ENOIOCTLCMD;
    }

    return 0;
}
```

1.  在最后一步中，我们必须定义`chrdev_ioctl.h`包含文件，以便与用户空间共享，其中包含在前面的代码块中定义的`ioctl()`命令：

```
/*
 * Chrdev ioctl() include file
 */

#include <linux/ioctl.h>
#include <linux/types.h>

#define CHRDEV_IOCTL_BASE    'C'
#define CHRDEV_NAME_LEN      32

struct chrdev_info {
    char label[CHRDEV_NAME_LEN];
    int read_only;
};

/*
 * The ioctl() commands
 */

#define CHRDEV_IOC_GETINFO _IOR(CHRDEV_IOCTL_BASE, 0, struct chrdev_info)
#define WDIOC_SET_RDONLY _IOW(CHRDEV_IOCTL_BASE, 1, int)
```

# 工作原理...

在*步骤 2*中，将使用`info`、`uarg`和`iuarg`变量，而使用`_IOC_TYPE()`宏是为了通过检查命令的类型与`CHRDEV_IOCTL_BASE`定义相比较来验证`cmd`命令对我们的驱动程序是否有效。

细心的读者应该注意，由于命令类型只是一个随机数，因此此检查并不是绝对可靠的；但是，对于我们在这里的目的来说可能已经足够了。

此外，通过使用`_IOC_NR()`、`_IOC_SIZE()`和`_IOC_DIR()`，我们可以从命令中提取其他信息，这对进一步的检查可能有用。

在*步骤 3*中，我们可以看到对于每个命令，根据它是读取还是写入（或两者），我们必须通过使用适当的访问函数从用户空间获取或放置用户数据，如第三章中所解释的那样，*使用字符驱动程序*，以避免内存损坏！

现在应该清楚`info`、`uarg`和`iuarg`变量是如何使用的。第一个用于本地存储`struct chrdev_info`数据，而其他变量用于具有适当类型的数据，以便与`copy_to_user()`或`get_user()`函数一起使用。

# 还有更多...

为了测试代码并查看其行为，我们需要制作一个适当的工具来执行我们的新`ioctl()`命令。

`chrdev_ioctl.c`文件中提供了一个示例，并在下面的片段中使用了`ioctl()`调用：

```
    /* Try reading device info */
    ret = ioctl(fd, CHRDEV_IOC_GETINFO, &info);
        if (ret < 0) {
            perror("ioctl(CHRDEV_IOC_GETINFO)");
            exit(EXIT_FAILURE);
        }
    printf("got label=%s and read_only=%d\n", info.label, info.read_only);

    /* Try toggling the device reading mode */
    read_only = !info.read_only;
    ret = ioctl(fd, WDIOC_SET_RDONLY, &read_only);
        if (ret < 0) {
            perror("ioctl(WDIOC_SET_RDONLY)");
            exit(EXIT_FAILURE);
        }
    printf("device has now read_only=%d\n", read_only);
```

现在，让我们在主机 PC 上使用下一个命令行编译`chrdev_ioctl.c`程序：

```
$ make CFLAGS="-Wall -O2 -Ichrdev/" \
 CC=aarch64-linux-gnu-gcc \
 chrdev_ioctl aarch64-linux-gnu-gcc -Wall -O2 chrdev_ioctl.c -o chrdev_ioctl 
```

请注意，这个命令也可以在 ESPRESSObin 上执行，只需删除`CC=aarch64-linux-gnu-gcc`设置。

现在，如果我们尝试在 chrdev 设备上执行该命令，我们应该得到以下输出：

```
# ./chrdev_ioctl /dev/cdev-eeprom\@2
file /dev/cdev-eeprom@2 opened
got label=cdev-eeprom and read_only=0
device has now read_only=1
```

当然，为了使其工作，我们将已经加载了包含`ioctl()`方法的新 chrdev 驱动程序版本。

在内核消息中，我们应该得到以下内容：

```
chrdev cdev-eeprom@2: chrdev (id=2) opened
chrdev cdev-eeprom@2: cmd nr=0 size=36 dir=2
chrdev cdev-eeprom@2: CHRDEV_IOC_GETINFO
chrdev cdev-eeprom@2: cmd nr=1 size=4 dir=1
chrdev cdev-eeprom@2: WDIOC_SET_RDONLY
chrdev cdev-eeprom@2: chrdev (id=2) released
```

如我们所见，在设备打开后，两个`ioctl()`命令按预期执行。

# 另请参阅

+   有关`ioctl()`系统调用的更多信息，一个很好的起点是它的 man 页面，可以使用`man 2 ioctl`命令获得。

# 使用 mmap()访问 I/O 内存

在这个示例中，我们将看到如何映射一个 I/O 内存区域到进程内存空间，以便通过内存中的指针访问我们的外围设备内部。

# 准备就绪

现在，让我们看看如何为我们的 chrdev 驱动程序实现自定义的`mmap()`系统调用。

由于我们有一个完全映射到内存中的虚拟设备，我们可以假设`struct chrdev_device`中的`buf`缓冲区代表要映射的内存区域。此外，我们需要动态分配它以便重新映射；这是因为内核虚拟内存地址不能使用`remap_pfn_range()`函数重新映射。

这是`remap_pfn_range()`的唯一限制，它无法重新映射未动态分配的内核虚拟内存地址。这些地址也可以重新映射，但是使用本书未涵盖的另一种技术。

为了准备我们的驱动程序，我们必须对`struct chrdev_device`进行以下修改：

```
diff --git a/chapter_07/chrdev/chrdev.h b/chapter_07/chrdev/chrdev.h
index 6b925fe..40a244f 100644
--- a/chapter_07/chrdev/chrdev.h
+++ b/chapter_07/chrdev/chrdev.h
@@ -7,7 +7,7 @@

 #define MAX_DEVICES 8
 #define NAME_LEN    CHRDEV_NAME_LEN
-#define BUF_LEN     300
+#define BUF_LEN     PAGE_SIZE

 /*
  * Chrdev basic structs
@@ -17,7 +17,7 @@
 struct chrdev_device {
     char label[NAME_LEN];
     unsigned int busy : 1;
-    char buf[BUF_LEN];
+    char *buf;
     int read_only;

     unsigned int id;
```

请注意，我们还修改了缓冲区大小，至少为`PAGE_SIZE`长，因为我们不能重新映射小于`PAGE_SIZE`字节的内存区域。

然后，为了动态分配内存缓冲区，我们必须进行以下列出的修改：

```
diff --git a/chapter_07/chrdev/chrdev.c b/chapter_07/chrdev/chrdev.c
index 3717ad2..a8bffc3 100644
--- a/chapter_07/chrdev/chrdev.c
+++ b/chapter_07/chrdev/chrdev.c
@@ -7,6 +7,7 @@
 #include <linux/module.h>
 #include <linux/fs.h>
 #include <linux/uaccess.h>
+#include <linux/slab.h>
 #include <linux/mman.h>

@@ -246,6 +247,13 @@ int chrdev_device_register(const char *label, unsigned int 
id,
          return -EBUSY;
      }

+     /* First try to allocate memory for internal buffer */
+     chrdev->buf = kzalloc(BUF_LEN, GFP_KERNEL);
+     if (!chrdev->buf) {
+         dev_err(chrdev->dev, "cannot allocate memory buffer!\n");
+         return -ENOMEM;
+     }
+
      /* Create the device and initialize its data */
      cdev_init(&chrdev->cdev, &chrdev_fops);
      chrdev->cdev.owner = owner;
@@ -255,7 +263,7 @@ int chrdev_device_register(const char *label, unsigned int id,
      if (ret) {
          pr_err("failed to add char device %s at %d:%d\n",
                            label, MAJOR(chrdev_devt), id);
-         return ret;
+         goto kfree_buf;
      }
 chrdev->dev = device_create(chrdev_class, parent, devt, chrdev,
```

这是前面`diff`文件的延续：

```
@@ -272,7 +280,6 @@ int chrdev_device_register(const char *label, unsigned int id,
      chrdev->read_only = read_only;
      chrdev->busy = 1;
      strncpy(chrdev->label, label, NAME_LEN);
-     memset(chrdev->buf, 0, BUF_LEN);

      dev_info(chrdev->dev, "chrdev %s with id %d added\n", label, id);

@@ -280,6 +287,8 @@ int chrdev_device_register(const char *label, unsigned int id,

  del_cdev:
      cdev_del(&chrdev->cdev);
+ kfree_buf:
+     kfree(chrdev->buf);

      return ret;
 }
@@ -309,6 +318,9 @@ int chrdev_device_unregister(const char *label, unsigned int id)

      dev_info(chrdev->dev, "chrdev %s with id %d removed\n", label, id);

+     /* Free allocated memory */
+     kfree(chrdev->buf);
+
        /* Dealocate the device */
        device_destroy(chrdev_class, chrdev->dev->devt);
        cdev_del(&chrdev->cdev);
```

然而，除了这个小注释，我们可以像之前一样继续，即修改我们的 chrdev 驱动程序并添加新的方法。

# 如何做...

让我们按照以下步骤来做：

1.  与前几节一样，第一步是将我们的新`mmap()`方法添加到驱动程序的`struct file_operations`中：

```
static const struct file_operations chrdev_fops = {
    .owner          = THIS_MODULE,
    .mmap           = chrdev_mmap,
    .unlocked_ioctl = chrdev_ioctl,
    .llseek         = chrdev_llseek,
    .read           = chrdev_read,
    .write          = chrdev_write,
    .open           = chrdev_open,
    .release        = chrdev_release
};
```

1.  然后，我们添加`chrdev_mmap()`实现，如前一节中所解释的并在下面报告：

```
static int chrdev_mmap(struct file *filp, struct vm_area_struct *vma)
{
    struct chrdev_device *chrdev = filp->private_data;
    size_t size = vma->vm_end - vma->vm_start;
    phys_addr_t offset = (phys_addr_t) vma->vm_pgoff << PAGE_SHIFT;
    unsigned long pfn;

    /* Does it even fit in phys_addr_t? */
    if (offset >> PAGE_SHIFT != vma->vm_pgoff)
        return -EINVAL;

    /* We cannot mmap too big areas */
    if ((offset > BUF_LEN) || (size > BUF_LEN - offset))
        return -EINVAL;
```

1.  然后，我们必须获取`buf`缓冲区的物理地址：

```
    /* Get the physical address belong the virtual kernel address */
    pfn = virt_to_phys(chrdev->buf) >> PAGE_SHIFT;
```

请注意，如果我们只想重新映射外围设备映射的物理地址，则不需要这一步。

1.  最后，我们可以进行重新映射：

```
    /* Remap-pfn-range will mark the range VM_IO */
    if (remap_pfn_range(vma, vma->vm_start,
                pfn, size,
                vma->vm_page_prot))
        return -EAGAIN;

    return 0;
}
```

# 它是如何工作的...

在*步骤 2*中，函数从一些健全性检查开始，我们必须验证所请求的内存区域是否与系统和外围设备的要求兼容。在我们的示例中，我们必须验证内存区域的大小和偏移量，以及映射开始的位置是否在`buf`的大小（`BUF_LEN`字节）内。

# 还有更多...

为了测试我们的新的`mmap()`实现，我们可以使用之前介绍的`chrdev_mmap.c`程序。在这里我们谈到了`textfile.txt`。要编译它，我们可以在主机 PC 上使用以下命令：

```
$ make CFLAGS="-Wall -O2" \
 CC=aarch64-linux-gnu-gcc \
 chrdev_mmap
aarch64-linux-gnu-gcc -Wall -O2 chrdev_mmap.c -o chrdev_mmap
```

请注意，可以通过简单删除`CC=aarch64-linux-gnu-gcc`设置在 ESPRESSObin 中执行此命令。

现在，让我们开始在驱动程序中写点东西：

```
# cp textfile.txt /dev/cdev-eeprom\@2
```

内核消息如下：

```
chrdev cdev-eeprom@2: chrdev (id=2) opened
chrdev cdev-eeprom@2: chrdev (id=2) released
chrdev cdev-eeprom@2: chrdev (id=2) opened
chrdev cdev-eeprom@2: should write 54 bytes (*ppos=0)
chrdev cdev-eeprom@2: got 54 bytes (*ppos=54)
chrdev cdev-eeprom@2: chrdev (id=2) released
```

现在，如预期的那样，在我们的内存缓冲区中有`textfile.txt`的内容；实际上：

```
# cat /dev/cdev-eeprom\@2 
This is a test file

This is line 3.

End of the file
```

现在我们可以尝试在我们的设备上执行`chrdev_mmap`程序，以验证一切是否正常工作：

```
# ./chrdev_mmap /dev/cdev-eeprom\@2 54
file /dev/cdev-eeprom@2 opened
got address=0xffff9896c000 and len=54
---
This is a test file

This is line 3.

End of the file
```

请注意，我们必须确保不指定大于设备缓冲区大小的值，例如在我们的示例中为 4,096。实际上，如果我们这样做，会出现错误：

**`./chrdev_mmap /dev/cdev-eeprom\@2 4097`**

`file /dev/cdev-eeprom@2 opened`

`mmap: Invalid argument`

这意味着我们成功了！请注意，`chrdev_mmap`程序（如`cp`和`cat`）在通常文件和我们的字符设备上的工作完全相同。

与`mmap()`执行相关的内核消息如下：

```
chrdev cdev-eeprom@2: chrdev (id=2) opened
chrdev cdev-eeprom@2: mmap vma=ffff9896c000 pfn=79ead size=1000
chrdev cdev-eeprom@2: chrdev (id=2) released
```

请注意，在重新映射之后，程序不执行任何系统调用来访问数据。这导致在获取对设备数据的访问权限时，可能会比我们需要使用`read()`或`write()`系统调用的情况下性能更好。

我们还可以通过向`chrdev_mmap`程序添加可选参数`0`来修改缓冲区内容，如下所示：

```
./chrdev_mmap /dev/cdev-eeprom\@2 54 0
file /dev/cdev-eeprom@2 opened
got address=0xffff908ef000 and len=54
---
This is a test file

This is line 3.

End of the file
---
First character changed to '0'
```

然后，当我们使用`read()`系统调用和`cat`命令再次读取缓冲区时，我们可以看到文件中的第一个字符已经按预期更改为 0：

```
# cat /dev/cdev-eeprom\@2 
0his is a test file

This is line 3.

End of the file
```

# 另请参阅

+   有关`mmap()`的更多信息，一个很好的起点是它的 man 页面（`man 2 mmap`）；然后，查看[`linux-kernel-labs.github.io/master/labs/memory_mapping.html`](https://linux-kernel-labs.github.io/master/labs/memory_mapping.html)会更好。

# 使用进程上下文进行锁定

在这个示例中，我们将看到如何保护数据，以防止两个或更多进程并发访问，以避免竞争条件。

# 如何做...

为了简单地演示如何向 chrdev 驱动程序添加互斥体，我们可以对其进行一些修改，如下所示。

1.  首先，我们必须在`chrdev.h`头文件中的驱动程序主结构中添加`mux`互斥体，如下所示：

```
/* Main struct */
struct chrdev_device {
    char label[NAME_LEN];
    unsigned int busy : 1;
    char *buf;
    int read_only;

    unsigned int id;
    struct module *owner;
    struct cdev cdev;
    struct device *dev;

    struct mutex mux;
};
```

这里介绍的所有修改都可以应用于 chrdev 代码，使用`add_mutex_to_chrdev.patch`文件中的`patch`命令，如下所示：

**`$ patch -p3 < add_mutex_to_chrdev.patch`**

1.  然后，在`chrdev_device_register()`函数中，我们必须使用`mutex_init()`函数初始化互斥体：

```
    /* Init the chrdev data */
    chrdev->id = id;
    chrdev->read_only = read_only;
    chrdev->busy = 1;
    strncpy(chrdev->label, label, NAME_LEN);
    mutex_init(&chrdev->mux);

    dev_info(chrdev->dev, "chrdev %s with id %d added\n", label, id);

    return 0;
```

1.  接下来，我们可以修改`read()`和`write()`方法以保护它们。然后，`read()`方法应该如下所示：

```
static ssize_t chrdev_read(struct file *filp,
               char __user *buf, size_t count, loff_t *ppos)
{
    struct chrdev_device *chrdev = filp->private_data;
    int ret;

    dev_info(chrdev->dev, "should read %ld bytes (*ppos=%lld)\n",
                count, *ppos);
    mutex_lock(&chrdev->mux); // Grab the mutex

    /* Check for end-of-buffer */
    if (*ppos + count >= BUF_LEN)
        count = BUF_LEN - *ppos;

    /* Return data to the user space */
    ret = copy_to_user(buf, chrdev->buf + *ppos, count);
    if (ret < 0) {
        count = -EFAULT;
        goto unlock;
    }

    *ppos += count;
    dev_info(chrdev->dev, "return %ld bytes (*ppos=%lld)\n", count, *ppos);

unlock:
    mutex_unlock(&chrdev->mux); // Release the mutex

    return count;
}
```

`write()`方法报告如下：

```
static ssize_t chrdev_write(struct file *filp,
                const char __user *buf, size_t count, loff_t *ppos)
{
    struct chrdev_device *chrdev = filp->private_data;
    int ret;

    dev_info(chrdev->dev, "should write %ld bytes (*ppos=%lld)\n",
                count, *ppos);

    if (chrdev->read_only)
        return -EINVAL;

    mutex_lock(&chrdev->mux); // Grab the mutex

    /* Check for end-of-buffer */
    if (*ppos + count >= BUF_LEN)
        count = BUF_LEN - *ppos;

    /* Get data from the user space */
    ret = copy_from_user(chrdev->buf + *ppos, buf, count);
    if (ret < 0) {
        count = -EFAULT;
        goto unlock;
    }

    *ppos += count;
    dev_info(chrdev->dev, "got %ld bytes (*ppos=%lld)\n", count, *ppos);

unlock:
    mutex_unlock(&chrdev->mux); // Release the mutex

    return count;
}
```

1.  最后，我们还必须保护`ioctl()`方法，因为驱动程序的`read_only`属性可能会改变：

```
static long chrdev_ioctl(struct file *filp,
            unsigned int cmd, unsigned long arg)
{
    struct chrdev_device *chrdev = filp->private_data;
    struct chrdev_info info;
    void __user *uarg = (void __user *) arg;
    int __user *iuarg = (int __user *) arg;
    int ret;

...

    /* Grab the mutex */
    mutex_lock(&chrdev->mux);

    switch (cmd) {
    case CHRDEV_IOC_GETINFO:
        dev_info(chrdev->dev, "CHRDEV_IOC_GETINFO\n");

...

    default:
        ret = -ENOIOCTLCMD;
        goto unlock;
    }
    ret = 0;

unlock:
    /* Release the mutex */
    mutex_unlock(&chrdev->mux);

    return ret;
}
```

这确实是一个愚蠢的例子，但你应该考虑即使`ioctl()`方法也可能改变驱动程序的数据缓冲区或其他共享数据的情况。

这一次，我们删除了所有的`return`语句，改用`goto`。

# 工作原理...

展示代码的工作原理是非常困难的，因为在复制竞争条件时存在固有的困难，所以最好讨论一下我们可以从中期望什么。

但是，您仍然被鼓励测试代码，也许尝试编写一个更复杂的驱动程序，如果不正确地使用互斥体来管理并发，可能会成为一个真正的问题。

在*步骤 1*中，我们为系统中可能有的每个 chrdev 设备添加了一个互斥体。然后，在*步骤 2*中初始化后，我们可以有效地使用它，如*步骤 3*和*步骤 4*中所述。

通过使用`mutex_lock()`函数，实际上告诉内核没有其他进程可以在这一点之后并发地进行，以确保只有一个进程可以管理驱动程序的共享数据。如果其他进程确实尝试在第一个进程已经持有互斥锁的情况下获取互斥锁，新进程将在它尝试获取已锁定的互斥锁的确切时刻被放入等待队列中进入睡眠状态。

完成后，通过使用`mutex_unlock()`，我们通知内核`mux`互斥锁已被释放，因此，任何等待（即睡眠）的进程将被唤醒；然后，一旦最终重新调度运行，它可以继续并尝试，反过来，抓住锁。

请注意，在*步骤 3*中，在两个函数中，我们在真正有用的时候才抓住互斥锁，而不是在它们的开始；实际上，我们应该尽量保持锁定尽可能小，以保护共享数据（在我们的例子中，`ppos`指针和`buf`数据缓冲区）。通过这样做，我们将我们选择的互斥排除机制的使用限制在代码的最小可能部分（临界区），这个临界区访问我们想要保护免受在先前指定的条件下发生的竞争条件引入的可能破坏。

还要注意的是，我们必须小心，不要在释放锁之前返回，否则新的访问进程将挂起！这就是为什么我们删除了所有的`return`语句，除了最后一个，并且使用`goto`语句跳转到`unlock`标签。

# 另请参阅

+   有关互斥锁和锁定的更多信息，请参阅内核文档目录中的`linux/Documentation/locking/mutex-design.txt`。

# 使用中断上下文进行锁定（和同步）

现在，让我们看看如何避免进程上下文和中断上下文之间的竞争条件。然而，这一次我们必须比以前更加注意，因为这一次我们必须实现一个锁定机制来保护进程上下文和中断上下文之间的共享数据。但是，我们还必须为读取进程和驱动程序之间提供同步机制，以允许读取进程在驱动程序的队列中存在要读取的数据时继续执行其操作。

为了解释这个问题，最好做一个实际的例子。假设我们有一个生成数据供读取进程使用的外围设备。为了通知新数据已到达，外围设备向 CPU 发送中断，因此我们可以想象使用循环缓冲区来实现我们的驱动程序，其中中断处理程序将数据从外围设备保存到缓冲区中，并且任何读取进程可以从中获取数据。

循环缓冲区（也称为环形缓冲区）是固定大小的缓冲区，其工作方式就好像内存是连续的，所有内存位置都以循环方式处理。随着信息从缓冲区生成和消耗，不需要重新整理；我们只需调整头指针和尾指针。添加数据时，头指针前进，而消耗数据时，尾指针前进。如果到达缓冲区的末尾，那么每个指针都会简单地回到环的起始位置。

在这种情况下，我们必须保护循环缓冲区免受进程和中断上下文的竞争条件，因为两者都可以访问它，但我们还必须提供同步机制，以便在没有可供读取的数据时使任何读取进程进入睡眠状态！

在[第五章](https://cdp.packtpub.com/linux_device_driver_development_cookbook/wp-admin/post.php?post=30&action=edit#post_28)中，*管理中断和并发*，我们介绍了自旋锁，它可以用于在进程和中断上下文之间放置锁定机制；我们还介绍了等待队列，它可以用于将读取进程与中断处理程序同步。

# 准备工作

这一次，我们必须使用我们 chrdev 驱动程序的修改版本。在 GitHub 存储库的`chapter_07/chrdev/`目录中，我们可以找到实现我们修改后的驱动程序的`chrdev_irq.c`和`chrdev_irq.h`文件。

我们仍然可以使用`chrdev-req.ko`在系统中生成 chrdev 设备，但现在内核模块将使用`chrdev_irq.ko`而不是`chrdev.ko`。

此外，由于我们有一个真正的外围设备，我们可以使用内核定时器来模拟 IRQ（请参阅[第五章](https://cdp.packtpub.com/linux_device_driver_development_cookbook/wp-admin/post.php?post=30&action=edit#post_28)，*管理中断和并发性*），该定时器还使用以下`get_new_char()`函数触发数据生成：

```
/*
 * Dummy function to generate data
 */

static char get_new_char(void)
{
    static char d = 'A' - 1;

    if (++d == ('Z' + 1))
        d = 'A';

    return d;
}
```

该功能每次调用时都会简单地从 A 到 Z 生成一个新字符，在生成 Z 后重新从字符 A 开始。

为了集中精力关注驱动程序的锁定和同步机制，我们在这里介绍了一些有用的函数来管理循环缓冲区，这是不言自明的。以下是两个检查缓冲区是否为空或已满的函数：

```
/*
 * Circular buffer management functions
 */

static inline bool cbuf_is_empty(size_t head, size_t tail,
                                 size_t len)
{
    return head == tail;
}

static inline bool cbuf_is_full(size_t head, size_t tail,
                                 size_t len)
{
    head = (head + 1) % len;
    return head == tail;
}
```

然后，有两个函数来检查缓冲区的内存区域直到末尾有多少数据或多少空间可用。当我们必须使用`memmove()`等函数时，它们非常有用：

```
static inline size_t cbuf_count_to_end(size_t head, size_t tail,
                                  size_t len)
{
    if (head >= tail)
        return head - tail;
    else
        return len - tail + head;
}

static inline size_t cbuf_space_to_end(size_t head, size_t tail,
                                  size_t len)
{
    if (head >= tail)
        return len - head + tail - 1;
    else
        return tail - head - 1;
}
```

最后，我们可以使用函数正确地向前移动头部或尾部指针，以便在缓冲区末尾时重新开始：

```
static inline void cbuf_pointer_move(size_t *ptr, size_t n,
                                 size_t len)
{
    *ptr = (*ptr + n) % len;
}
```

# 如何做...

让我们按照以下步骤来做：

1.  第一步是通过添加`mux`互斥锁（与以前一样）、`lock`自旋锁、内核`timer`和等待队列`queue`来重写我们驱动程序的主要结构，如下所示：

```
 /* Main struct */
struct chrdev_device {
    char label[NAME_LEN];
    unsigned int busy : 1;
    char *buf;
    size_t head, tail;
    int read_only;

    unsigned int id;
    struct module *owner;
    struct cdev cdev;
    struct device *dev;

    struct mutex mux;
    struct spinlock lock;
    struct wait_queue_head queue;
    struct hrtimer timer;
};
```

1.  然后，在`chrdev_device_register()`函数中进行设备分配期间必须对其进行初始化，如下所示：

```
    /* Init the chrdev data */
    chrdev->id = id;
    chrdev->read_only = read_only;
    chrdev->busy = 1;
    strncpy(chrdev->label, label, NAME_LEN);
    mutex_init(&chrdev->mux);
    spin_lock_init(&chrdev->lock);
    init_waitqueue_head(&chrdev->queue);
    chrdev->head = chrdev->tail = 0;

    /* Setup and start the hires timer */
    hrtimer_init(&chrdev->timer, CLOCK_MONOTONIC,
                        HRTIMER_MODE_REL | HRTIMER_MODE_SOFT);
    chrdev->timer.function = chrdev_timer_handler;
    hrtimer_start(&chrdev->timer, ns_to_ktime(delay_ns),
                        HRTIMER_MODE_REL | HRTIMER_MODE_SOFT);
```

1.  现在，`read()`方法的可能实现如下代码片段所示。我们首先获取互斥锁，以对其他进程进行第一次锁定：

```
static ssize_t chrdev_read(struct file *filp,
                           char __user *buf, size_t count, loff_t *ppos)
{
    struct chrdev_device *chrdev = filp->private_data;
    unsigned long flags;
    char tmp[256];
    size_t n;
    int ret;

    dev_info(chrdev->dev, "should read %ld bytes\n", count);

    /* Grab the mutex */
    mutex_lock(&chrdev->mux);
```

现在，我们可以确信没有其他进程可以超越这一点，但是在中断上下文中运行的一些核心仍然可以这样做！

1.  这就是为什么我们需要以下步骤来确保它们与中断上下文同步：

```
    /* Check for some data into read buffer */
    if (filp->f_flags & O_NONBLOCK) {
        if (cbuf_is_empty(chrdev->head, chrdev->tail, BUF_LEN)) {
            ret = -EAGAIN;
            goto unlock;
        }
    } else if (wait_event_interruptible(chrdev->queue,
        !cbuf_is_empty(chrdev->head, chrdev->tail, BUF_LEN))) {
        count = -ERESTARTSYS;
        goto unlock; 
    }

    /* Grab the lock */
    spin_lock_irqsave(&chrdev->lock, flags);
```

1.  当我们获取了锁时，我们可以确信我们是唯一的读取进程，并且我们也受到中断上下文的保护；因此，我们可以安全地从循环缓冲区读取数据，然后释放锁，如下所示：

```
    /* Get data from the circular buffer */
    n = cbuf_count_to_end(chrdev->head, chrdev->tail, BUF_LEN);
    count = min(count, n); 
    memcpy(tmp, &chrdev->buf[chrdev->tail], count);

    /* Release the lock */
    spin_unlock_irqrestore(&chrdev->lock, flags);
```

请注意，我们必须将数据从循环缓冲区复制到本地缓冲区，而不是直接复制到用户空间缓冲区`buf`，使用`copy_to_user()`函数；这是因为此函数可能会进入睡眠状态，而在我们睡眠时持有自旋锁是不好的！

1.  自旋锁释放后，我们可以安全地调用`copy_to_user()`将数据发送到用户空间：

```
    /* Return data to the user space */
    ret = copy_to_user(buf, tmp, count);
    if (ret < 0) {
        ret = -EFAULT;
        goto unlock; 
    }
```

1.  最后，在释放互斥锁之前，我们必须更新循环缓冲区的`tail`指针，如下所示：

```
    /* Now we can safely move the tail pointer */
    cbuf_pointer_move(&chrdev->tail, count, BUF_LEN);
    dev_info(chrdev->dev, "return %ld bytes\n", count);

unlock:
    /* Release the mutex */
    mutex_unlock(&chrdev->mux);

    return count;
}
```

请注意，由于在进程上下文中只有读取器，它们是唯一移动`tail`指针的进程（或者中断处理程序这样做——请参见下面的代码片段），我们可以确信一切都会正常工作。

1.  最后，中断处理程序（在我们的情况下，它是由内核定时器处理程序模拟的）如下所示：

```
static enum hrtimer_restart chrdev_timer_handler(struct hrtimer *ptr)
{
    struct chrdev_device *chrdev = container_of(ptr,
                    struct chrdev_device, timer);

    spin_lock(&chrdev->lock);    /* grab the lock */ 

    /* Now we should check if we have some space to
     * save incoming data, otherwise they must be dropped...
     */
    if (!cbuf_is_full(chrdev->head, chrdev->tail, BUF_LEN)) {
        chrdev->buf[chrdev->head] = get_new_char();

        cbuf_pointer_move(&chrdev->head, 1, BUF_LEN);
    }
    spin_unlock(&chrdev->lock);  /* release the lock */

    /* Wake up any possible sleeping process */
    wake_up_interruptible(&chrdev->queue);

    /* Now forward the expiration time and ask to be rescheduled */
    hrtimer_forward_now(&chrdev->timer, ns_to_ktime(delay_ns));
    return HRTIMER_RESTART;
}
```

处理程序的主体很简单：它获取锁，然后将单个字符添加到循环缓冲区。

请注意，在这里，由于我们有一个真正的外围设备，我们只是丢弃数据；在实际情况下，驱动程序开发人员可能需要采取任何必要的措施来防止数据丢失，例如停止外围设备，然后以某种方式向用户空间发出此错误条件的信号！

此外，在退出之前，它使用`wake_up_interruptible()`函数唤醒等待队列上可能正在睡眠的进程。

# 工作原理...

这些步骤相当不言自明。但是，在*步骤 4*中，我们执行了两个重要步骤：第一个是如果循环缓冲区为空，则挂起进程，如果不是，则使用中断上下文抓取锁，因为我们将要访问循环缓冲区。

对`O_NONBLOCK`标志的检查只是为了遵守`read()`的行为，即如果使用了`O_NONBLOCK`标志，那么它应该继续进行，然后如果没有数据可用，则返回`EAGAIN`错误。

请注意，在检查缓冲区是否为空之前，可以安全地获取锁，因为如果我们决定缓冲区为空，但同时到达了一些新数据并且`O_NONBLOCK`处于活动状态，我们只需返回`EAGAIN`（向读取进程发出重新执行操作的信号）。如果不是，我们会在等待队列上睡眠，然后会被中断处理程序唤醒（请参阅以下信息）。在这两种情况下，我们的操作都是正确的。

# 还有更多...

如果您希望测试代码，请编译代码并将其插入 ESPRESSObin 中：

```
# insmod chrdev_irq.ko 
chrdev_irq:chrdev_init: got major 239
# insmod chrdev-req.ko 
chrdev cdev-eeprom@2: chrdev cdev-eeprom with id 2 added
chrdev cdev-rom@4: chrdev cdev-rom with id 4 added
```

现在我们的外围设备已启用（内核定时器已在`chrdev_device_register()`函数中的*步骤 2*中启用），并且应该已经有一些数据可供读取；实际上，如果我们通过使用`cat`命令在驱动程序上进行`read()`，我们会得到以下结果：

```
# cat /dev/cdev-eeprom\@2 
ACEGIKMOQSUWYACEGIKMOQSUWYACEGIKMOQSUWYACEGIKMOQSUWYACEGIKMOQSUWYACEGIKMOQSUWYACEGIKMOQSUW
```

在这里，我们应该注意，由于我们在系统中定义了两个设备（请参阅本章开头使用的`chapter_04/chrdev/add_chrdev_devices.dts.patch` DTS 文件），因此`get_new_char()`函数每秒执行两次，这就是为什么我们得到序列`ACE...`而不是`ABC...`。

在这里，一个很好的练习是修改驱动程序，当第一次打开驱动程序时启动内核定时器，然后在最后一次释放时停止它。此外，您可以尝试为每个系统中的设备提供一个每设备的`get_new_char()`函数来生成正确的序列（ABC...）。

相应的内核消息如下所示：

```
chrdev cdev-eeprom@2: chrdev (id=2) opened
chrdev cdev-eeprom@2: should read 131072 bytes
chrdev cdev-eeprom@2: return 92 bytes
```

在这里，由于*步骤 3*到*步骤 7*，`read()`系统调用使调用进程进入睡眠状态，然后一旦数据到达就立即返回新数据。

实际上，如果我们等一会儿，我们会看到以下内核消息每秒获得一个新字符：

```
...
[ 227.675229] chrdev cdev-eeprom@2: should read 131072 bytes
[ 228.292171] chrdev cdev-eeprom@2: return 1 bytes
[ 228.294129] chrdev cdev-eeprom@2: should read 131072 bytes
[ 229.292156] chrdev cdev-eeprom@2: return 1 bytes
...
```

我留下了时间，以便了解生成每条消息的时间。

这种行为是由*步骤 8*引起的，内核定时器生成新数据。

# 另请参阅

+   有关自旋锁和锁定的更多信息，请参阅内核文档目录中的`linux/Documentation/locking/spinlocks.txt`。

# 使用 poll()和 select()等待 I/O 操作

在本教程中，我们将了解如何要求内核在我们的驱动程序有新数据可供读取（或愿意接受新数据进行写入）时为我们检查，然后唤醒读取（或写入）进程，而不会在 I/O 操作上被阻塞。

# 做好准备

要测试我们的实现，我们仍然可以像以前一样使用`chrdev_irq.c`驱动程序；这是因为我们可以使用内核定时器模拟的*新数据*事件。

# 如何做...

让我们看看如何通过以下步骤来做到这一点：

1.  首先，我们必须在驱动程序的`struct file_operations`中添加我们的新`chrdev_poll()`方法：

```
static const struct file_operations chrdev_fops = {
    .owner   = THIS_MODULE,
    .poll    = chrdev_poll,
    .llseek  = no_llseek,
    .read    = chrdev_read,
    .open    = chrdev_open,
    .release = chrdev_release
};
```

1.  然后，实现如下。我们首先通过将当前设备`chrdev->queue`的等待队列传递给`poll_wait()`函数：

```
static __poll_t chrdev_poll(struct file *filp, poll_table *wait)
{
    struct chrdev_device *chrdev = filp->private_data;
    __poll_t mask = 0;

    poll_wait(filp, &chrdev->queue, wait);
```

1.  最后，在检查循环缓冲区不为空并且我们可以继续从中读取数据之前，我们抓住互斥锁：

```
    /* Grab the mutex */
    mutex_lock(&chrdev->mux);

    if (!cbuf_is_empty(chrdev->head, chrdev->tail, BUF_LEN))
        mask |= EPOLLIN | EPOLLRDNORM;

    /* Release the mutex */
    mutex_unlock(&chrdev->mux);

    return mask;
}
```

请注意，抓取自旋锁也是不必要的。这是因为如果缓冲区为空，当新数据通过中断（在我们的模拟中是内核定时器）处理程序到达时，我们将得到通知。这将反过来调用`wake_up_interruptible(&chrdev->queue)`，它作用于我们之前提供给`poll_wait()`函数的等待队列。另一方面，如果缓冲区不为空，它不可能在中断上下文中变为空，因此我们根本不可能有任何竞争条件。

# 还有更多...

与以前一样，如果我们希望测试代码，我们需要实现一个适当的工具来执行我们的新`poll()`方法。当我们将其添加到驱动程序中时，我们将获得`poll()`和`select()`系统调用支持；`select()`的使用示例在`chrdev_select.c`文件中报告，在下面，有一个片段中使用了`select()`调用：

```
    while (1) {
        /* Set up reading file descriptors */
        FD_ZERO(&read_fds);
        FD_SET(STDIN_FILENO, &read_fds);
        FD_SET(fd, &read_fds);

        /* Wait for any data from our device or stdin */
        ret = select(FD_SETSIZE, &read_fds, NULL, NULL, NULL);
        if (ret < 0) {
            perror("select");
            exit(EXIT_FAILURE);
        }

        if (FD_ISSET(STDIN_FILENO, &read_fds)) {
            ret = read(STDIN_FILENO, &c, 1);
            if (ret < 0) { 
                perror("read(STDIN, ...)");
                exit(EXIT_FAILURE);
            }
            printf("got '%c' from stdin!\n", c);
        }
 ...

    }
```

正如我们所看到的，这个程序将使用`select()`系统调用来监视我们进程的标准输入通道（名为`stdin`）和字符设备，`select()`系统调用又调用我们在*步骤 2*和*步骤 3*中实现的新`poll()`方法。

现在，让我们在我们的主机 PC 上使用下一个命令行编译`chrdev_select.c`程序：

```
$ make CFLAGS="-Wall -O2 -Ichrdev/" \
 CC=aarch64-linux-gnu-gcc \
 chrdev_select aarch64-linux-gnu-gcc -Wall -O2 chrdev_ioctl.c -o chrdev_select
```

请注意，这个命令可以通过简单地删除`CC=aarch64-linux-gnu-gcc`设置在 ESPRESSObin 上执行。

现在，如果我们尝试在 chrdev 设备上执行该命令，我们应该会得到这个输出：

```
# ./chrdev_select /dev/cdev-eeprom\@2
file /dev/cdev-eeprom@2 opened
got 'K' from device!
got 'M' from device!
got 'O' from device!
got 'Q' from device!
...
```

当然，我们已经加载了包含`poll()`方法的`chrdev_irq`驱动程序。

如果我们尝试从标准输入插入一些字符，如下所示，我们可以看到当设备有新数据时，进程可以安全地对其进行读取而不会阻塞，而当标准输入有新数据时，进程也可以做同样的事情，同样也不会阻塞：

```
...
got 'Y' from device!
got 'A' from device!
TEST
got 'T' from stdin!
got 'E' from stdin!
got 'S' from stdin!
got 'T' from stdin!
got '
' from stdin!
got 'C' from device!
got 'E' from device!
...
```

# 另请参阅

+   有关`poll()`或`select()`的更多信息，一个很好的起点是它们的 man 页面（`man 2 poll`和`man 2 select`）。

# 使用`fasync()`管理异步通知

在这个示例中，我们将看到如何在我们的驱动程序有新数据要读取时（或者愿意接受来自用户空间的新数据）生成异步的`SIGIO`信号。

# 准备工作

与以前一样，我们仍然可以使用`chrdev_irq.c`驱动程序来展示我们的实现。

# 如何做...

让我们看看如何通过以下步骤来做到：

1.  首先，我们必须在驱动程序的`struct file_operations`中添加我们的新`chrdev_fasync()`方法：

```
static const struct file_operations chrdev_fops = {
    .owner   = THIS_MODULE,
    .fasync  = chrdev_fasync,
    .poll    = chrdev_poll,
    .llseek  = no_llseek,
    .read    = chrdev_read,
    .open    = chrdev_open,
    .release = chrdev_release
};
```

1.  实现如下：

```
static int chrdev_fasync(int fd, struct file *filp, int on)
{
    struct chrdev_device *chrdev = filp->private_data;

    return fasync_helper(fd, filp, on, &chrdev->fasync_queue);
}
```

1.  最后，我们必须在我们的（模拟的）中断处理程序中添加`kill_fasync()`调用，以表示由于有新数据准备好被读取，可以发送`SIGIO`信号：

```
static enum hrtimer_restart chrdev_timer_handler(struct hrtimer *ptr)
{
    struct chrdev_device *chrdev = container_of(ptr,
                                    struct chrdev_device, timer);

...
    /* Wake up any possible sleeping process */
    wake_up_interruptible(&chrdev->queue);
    kill_fasync(&chrdev->fasync_queue, SIGIO, POLL_IN);

    /* Now forward the expiration time and ask to be rescheduled */
    hrtimer_forward_now(&chrdev->timer, ns_to_ktime(delay_ns));
    return HRTIMER_RESTART;
}
```

# 还有更多...

如果您希望测试代码，您需要实现一个适当的工具来执行所有步骤，以要求内核接收`SIGIO`信号。下面报告了`chrdev_fasync.c`程序的片段，其中执行了所需的操作：

```
    /* Try to install the signal handler and the fasync stuff */
    sigh = signal(SIGIO, sigio_handler);
    if (sigh == SIG_ERR) {
            perror("signal");
            exit(EXIT_FAILURE);
    }
    ret = fcntl(fd, F_SETOWN, getpid());
    if (ret < 0) {
            perror("fcntl(..., F_SETOWN, ...)");
            exit(EXIT_FAILURE);
    }
    flags = fcntl(fd, F_GETFL);
    if (flags < 0) {
            perror("fcntl(..., F_GETFL)");
            exit(EXIT_FAILURE);
    }
    ret = fcntl(fd, F_SETFL, flags | FASYNC);
    if (flags < 0) {
            perror("fcntl(..., F_SETFL, ...)");
            exit(EXIT_FAILURE);
    }
```

这段代码是要求内核调用我们在*步骤 2*中实现的`fasync()`方法。然后，每当有新数据到达时，由于*步骤 3*，`SIGIO`信号将发送到我们的进程，并且信号处理程序`sigio_handler()`将被执行，即使进程被挂起，例如，在读取另一个文件描述符时。

```
void sigio_handler(int unused) {
    char c;
    int ret;

    ret = read(fd, &c, 1);
    if (ret < 0) {
        perror("read");
        exit(EXIT_FAILURE);
    }
    ret = write(STDOUT_FILENO, &c, 1);
    if (ret < 0) {
        perror("write");
        exit(EXIT_FAILURE);
    }
}
```

现在，让我们在我们的主机 PC 上使用下一个命令行编译`chrdev_fasync.c`程序：

```
$ make CFLAGS="-Wall -O2 -Ichrdev/" \
 CC=aarch64-linux-gnu-gcc \
 chrdev_fasync aarch64-linux-gnu-gcc -Wall -O2 chrdev_ioctl.c -o chrdev_fasync
```

请注意，这个命令可以通过简单地删除`CC=aarch64-linux-gnu-gcc`设置在 ESPRESSObin 上执行。

现在，如果我们尝试在 chrdev 设备上执行该命令，我们应该会得到以下输出：

```
# ./chrdev_fasync /dev/cdev-eeprom\@2 
file /dev/cdev-eeprom@2 opened
QSUWYACEGI
```

当然，我们已经加载了包含`fasync()`方法的`chrdev_irq`驱动程序。

在这里，进程在标准输入上的`read()`上被挂起，每当信号到达时，信号处理程序被执行并且新数据被读取。然而，当我们尝试向标准输入发送一些字符时，进程会如预期地读取它们：

```
# ./chrdev_fasync /dev/cdev-eeprom\@2 
file /dev/cdev-eeprom@2 opened
QSUWYACEGIKMOQS....
got '.' from stdin!
got '.' from stdin!
got '.' from stdin!
got '.' from stdin!
got '
' from stdin!
UWYACE
```

# 另请参阅

+   有关`fasync()`方法或`fcntl()`系统调用的更多信息，一个很好的起点是`man 2 fcntl`手册页。
