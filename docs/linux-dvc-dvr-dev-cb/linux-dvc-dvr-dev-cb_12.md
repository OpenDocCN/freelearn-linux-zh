# 第十二章：附加信息：高级字符驱动程序操作

# 技术要求

当我们必须管理外围设备时，通常需要修改其内部配置设置，或者将其从用户空间映射为内存缓冲区可能很有用，就好像我们可以通过引用指针来修改内部数据一样。

例如，帧缓冲区或帧抓取器是作为用户空间的大块内存映射的良好候选者。

在这种情况下，具有`lseek()`、`ioctl()`和`mmap()`系统调用的支持是至关重要的。如果从用户空间使用这些系统调用并不复杂，在内核中，它们需要驱动程序开发人员的一些注意，特别是`mmap()`系统调用，它涉及内核**内存管理单元**（**MMU**）。

不仅驱动程序开发人员必须注意的主要任务之一是与用户空间的数据交换机制；事实上，实现这种机制的良好实现可能会简化许多外围设备的管理。例如，使用读取和写入内存缓冲区可能会提高系统性能，当一个或多个进程访问外围设备时，为用户空间开发人员提供了一系列良好的设置和管理机制，使他们能够充分利用我们的硬件。

# 使用 lseek()在文件中上下移动

在这里，我们应该记住`read()`和`write()`系统调用的原型如下：

```
ssize_t (*read) (struct file *filp,
                 char __user *buf, size_t len, loff_t *ppos);
ssize_t (*write) (struct file *filp,
                 const char __user *buff, size_t len, loff_t *ppos);
```

当我们使用`chapter_03/chrdev_test.c`文件中的程序测试我们的字符驱动程序时，我们注意到除非我们对文件进行了如下修补，否则我们无法重新读取写入的数据：

```
--- a/chapter_03/chrdev_test.c
+++ b/chapter_03/chrdev_test.c
@@ -55,6 +55,16 @@ int main(int argc, char *argv[])
       dump("data written are: ", buf, n);
   }

+  close(fd);
+
+  ret = open(argv[1], O_RDWR);
+  if (ret < 0) {
+      perror("open");
+      exit(EXIT_FAILURE);
+  }
+  printf("file %s reopened\n", argv[1]);
+  fd = ret;
+
   for (c = 0; c < sizeof(buf); c += n) {
       ret = read(fd, buf, sizeof(buf));
       if (ret == 0) {
```

这是在不关闭然后重新打开与我们的驱动程序连接的文件的情况下（这样，内核会自动将`ppos`指向的值重置为`0`）。

然而，这并不是修改`ppos`指向的值的唯一方法；事实上，我们也可以使用`lseek()`系统调用来做到这一点。系统调用的原型，如其手册页（`man 2 lseek`）所述，如下所示：

```
off_t lseek(int fd, off_t offset, int whence);
```

在这里，`whence`参数可以假定以下值（由以下代码中的定义表示）：

```
  SEEK_SET
      The file offset is set to offset bytes.

  SEEK_CUR
      The file offset is set to its current location plus offset
      bytes.

  SEEK_END
      The file offset is set to the size of the file plus offset
      bytes.
```

因此，例如，如果我们希望像在第三章中所做的那样将`ppos`指向我们设备的数据缓冲区的开头，但是不关闭和重新打开设备文件，我们可以这样做：

```
--- a/chapter_03/chrdev_test.c
+++ b/chapter_03/chrdev_test.c
@@ -55,6 +55,13 @@ int main(int argc, char *argv[])
        dump("data written are: ", buf + c, n);
    }

+  ret = lseek(fd, SEEK_SET, 0);
+  if (ret < 0) {
+       perror("lseek");
+       exit(EXIT_FAILURE);
+  }
+  printf("*ppos moved to 0\n");
+
   for (c = 0; c < sizeof(buf); c += n) {
       ret = read(fd, buf, sizeof(buf));
       if (ret == 0) {
```

请注意，所有这些修改都存储在 GitHub 存储库中的`modify_lseek_to_chrdev_test.patch`文件中，可以通过在`chapter_03`目录中使用以下命令应用，该目录中包含`chrdev_test.c`文件：

**`$ patch -p2 < ../../chapter_07/modify_lseek_to_chrdev_test.patch`**

如果我们看一下`linux/include/uapi/linux/fs.h`头文件，我们可以看到这些定义是如何声明的：

```

#define SEEK_SET    0 /* seek relative to beginning of file */
#define SEEK_CUR    1 /* seek relative to current file position */
#define SEEK_END    2 /* seek relative to end of file */
```

`lseek()`的实现是如此简单，以至于在`linux/fs/read_write.c`文件中我们可以找到一个名为`default_llseek()`的此方法的默认实现。其原型如下所示：

```
loff_t default_llseek(struct file *file,
                      loff_t offset, int whence);
```

这是因为如果我们不指定自己的实现，那么内核将自动使用前面代码块中的实现。然而，如果我们快速查看`default_llseek()`函数，我们会注意到它对我们的设备不太适用，因为它太*面向文件*（也就是说，当`lseek()`操作的文件是真实文件而不是外围设备时，它可以很好地工作），因此我们可以使用`noop_llseek()`函数来代替`lseek()`的两种替代实现之一来执行无操作：

```
/**
 * noop_llseek - No Operation Performed llseek implementation
 * @file: file structure to seek on
 * @offset: file offset to seek to
 * @whence: type of seek
 *
 * This is an implementation of ->llseek useable for the rare special case when
 * userspace expects the seek to succeed but the (device) file is actually not
 * able to perform the seek. In this case you use noop_llseek() instead of
 * falling back to the default implementation of ->llseek.
 */
loff_t noop_llseek(struct file *file, loff_t offset, int whence)
{
    return file->f_pos;
}
```

或者我们可以返回一个错误，然后使用`no_llseek()`函数向用户空间发出信号，表明我们的设备不适合使用寻址：

```
loff_t no_llseek(struct file *file, loff_t offset, int whence)
{
    return -ESPIPE;
}
```

这两个前面的函数位于内核源码的`linux/fs/read_write.c`文件中。

这两个功能的不同用法在上面关于`noop_llseek()`的评论中有很好的描述；虽然`default_llseek()`通常不适用于字符设备，但我们可以简单地使用`no_llseek()`，或者在那些罕见的特殊情况下，用户空间期望寻址成功，但（设备）文件实际上无法执行寻址时，我们可以使用`no_llseek()`如下：

```
static const struct file_operations chrdev_fops = {
    .owner   = THIS_MODULE,
    .llseek  = no_llseek,
    .read    = chrdev_read,
    .write   = chrdev_write,
    .open    = chrdev_open,
    .release = chrdev_release
};
```

这段代码是在 GitHub 的`chapter_04/chrdev/chrdev.c`文件中讨论的 chrdev 字符驱动程序中提到的，如第四章中所述，*使用设备树*。

# 使用 ioctl()进行自定义命令

在[第三章](https://cdp.packtpub.com/linux_device_driver_development_cookbook/wp-admin/post.php?post=30&action=edit#post_26)中，*使用字符驱动程序*，我们讨论了文件抽象，并提到字符驱动程序在用户空间的角度上与通常的文件非常相似。但是，它根本不是一个文件；它被用作文件，但它属于一个外围设备，通常需要配置外围设备才能正常工作，因为它们可能支持不同的操作方法。

例如，让我们考虑一个串行端口；它看起来像一个文件，我们可以使用`read()`和`write()`系统调用进行读取或写入，但在大多数情况下，我们还必须设置一些通信参数，如波特率、奇偶校验位等。当然，这些参数不能通过`read()`或`write()`来设置，也不能通过使用`open()`系统调用来设置（即使它可以设置一些访问模式，如只读或只写），因此内核为我们提供了一个专用的系统调用，我们可以用来设置这些串行通信参数。这个系统调用就是`ioctl()`。

从用户空间的角度来看，它看起来像是它的 man 页面（通过使用`man 2 ioctl`命令可用）：

```
SYNOPSIS
   #include <sys/ioctl.h>

   int ioctl(int fd, unsigned long request, ...);

DESCRIPTION
   The ioctl() system call manipulates the underlying device parameters of special files. In particular, many operating characteristics of character special files (e.g., terminals) may be controlled with ioctl() requests.
```

如前面的段落所述，`ioctl()`系统调用通过获取文件描述符（通过打开我们的设备获得）作为第一个参数，以及设备相关的请求代码作为第二个参数，来操作特殊文件的底层设备参数（就像我们的字符设备一样，但实际上不仅仅是这样，它也可以用于网络或块设备），最后，作为第三个可选参数，是一个无类型指针，用户空间程序员可以用来与驱动程序交换数据。

因此，借助这个通用定义，驱动程序开发人员可以实现他们的自定义命令来管理底层设备。即使不是严格要求，`ioctl()`命令中编码了参数是输入参数还是输出参数，以及第三个参数的字节数。用于指定`ioctl()`请求的宏和定义位于`linux/include/uapi/asm-generic/ioctl.h`文件中，如下所述：

```
/*
 * Used to create numbers.
 *
 * NOTE: _IOW means userland is writing and kernel is reading. _IOR
 * means userland is reading and kernel is writing.
 */
#define _IO(type,nr)            _IOC(_IOC_NONE,(type),(nr),0)
#define _IOR(type,nr,size)      _IOC(_IOC_READ,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOW(type,nr,size)      _IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOWR(type,nr,size)     _IOC(_IOC_READ|_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
```

正如我们在前面的评论中也可以看到的，`read()`和`write()`操作是从用户空间的角度来看的，因此当我们将一个命令标记为*写入*时，我们的意思是用户空间在写入，内核在读取，而当我们将一个命令标记为*读取*时，我们的意思是完全相反。

关于如何使用这些宏的一个非常简单的例子，我们可以看一下关于看门狗的实现，位于文件`linux/include/uapi/linux/watchdog.h`中：

```
#include <linux/ioctl.h>
#include <linux/types.h>

#define WATCHDOG_IOCTL_BASE 'W'

struct watchdog_info {
    __u32 options;          /* Options the card/driver supports */
    __u32 firmware_version; /* Firmware version of the card */
    __u8 identity[32];      /* Identity of the board */
};

#define WDIOC_GETSUPPORT    _IOR(WATCHDOG_IOCTL_BASE, 0, struct watchdog_info)
#define WDIOC_GETSTATUS     _IOR(WATCHDOG_IOCTL_BASE, 1, int)
#define WDIOC_GETBOOTSTATUS _IOR(WATCHDOG_IOCTL_BASE, 2, int)
#define WDIOC_GETTEMP       _IOR(WATCHDOG_IOCTL_BASE, 3, int)
#define WDIOC_SETOPTIONS    _IOR(WATCHDOG_IOCTL_BASE, 4, int)
#define WDIOC_KEEPALIVE     _IOR(WATCHDOG_IOCTL_BASE, 5, int)
#define WDIOC_SETTIMEOUT    _IOWR(WATCHDOG_IOCTL_BASE, 6, int)
#define WDIOC_GETTIMEOUT    _IOR(WATCHDOG_IOCTL_BASE, 7, int)
#define WDIOC_SETPRETIMEOUT _IOWR(WATCHDOG_IOCTL_BASE, 8, int)
#define WDIOC_GETPRETIMEOUT _IOR(WATCHDOG_IOCTL_BASE, 9, int)
#define WDIOC_GETTIMELEFT   _IOR(WATCHDOG_IOCTL_BASE, 10, int)
```

看门狗（或看门狗定时器）通常用于自动化系统。它是一个电子定时器，用于检测和从计算机故障中恢复。事实上，在正常操作期间，系统中的一个进程应定期重置看门狗定时器，以防止它超时，因此，如果由于硬件故障或程序错误，系统未能重置看门狗，定时器将过期，并且系统将自动重新启动。

这里我们定义了一些命令来管理看门狗外围设备，每个命令都使用`_IOR()`宏（用于指定读取命令）或`_IOWR`宏（用于指定读/写命令）进行定义。每个命令都有一个渐进的数字，后面跟着第三个参数指向的数据类型，它可以是一个简单类型（如前面的`int`类型）或一个更复杂的类型（如前面的`struct watchdog_info`）。最后，`WATCHDOG_IOCTL_BASE`通用参数只是用来添加一个随机值，以避免命令重复。

在后面我们将解释我们的示例时，这些宏中`type`参数（在前面的示例中为`WATCHDOG_IOCTL_BASE`）的使用将更加清晰。

当然，这只是一个纯粹的约定，我们可以简单地使用渐进的整数来定义我们的`ioctl()`命令，它仍然可以完美地工作；然而，通过这种方式行事，我们将嵌入到命令代码中很多有用的信息。

一旦所有命令都被定义，我们需要添加我们自定义的`ioctl()`实现，并且通过查看`linux/include/linux/fs.h`文件中的`struct file_operations`，我们可以看到其中存在两个：

```
struct file_operations {
...
    long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
    long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
```

在 2.6.36 之前的内核中，只有一个`ioctl()`方法可以获取**Big Kernel Lock**（**BKL**），因此其他任何东西都无法同时执行。这导致多处理器机器上的性能非常糟糕，因此大力去除它，这就是为什么引入了`unlocked_ioctl()`。通过使用它，每个驱动程序开发人员都可以选择使用哪个锁。

另一方面，`compat_ioctl()`，尽管是同时添加的，但实际上与`unlocked_ioctl()`无关。它的目的是允许 32 位用户空间程序在 64 位内核上进行`ioctl()`调用。

最后，我们应该首先注意到命令和结构定义必须在用户空间和内核空间中使用，因此当我们定义交换的数据类型时，必须使用这两个空间都可用的数据类型（这就是为什么使用`__u32`类型而不是`u32`，后者实际上只存在于内核中）。

此外，当我们希望使用自定义的`ioctl()`命令时，我们必须将它们定义到一个单独的头文件中，并且必须与用户空间共享；通过这种方式，我们可以将内核代码与用户空间分开。然而，如果难以将所有用户空间代码与内核空间分开，我们可以使用`__KERNEL__`定义，如下面的片段所示，指示预处理器根据我们编译的空间来排除一些代码：

```
#ifdef __KERNEL__
  /* This is code for kernel space */
  ...
#else
  /* This is code for user space */
  ...
#endif
```

这就是为什么通常，包含`ioctl()`命令的头文件通常位于`linux/include/uapi`目录下，该目录包含用户空间程序编译所需的所有头文件。

# 使用 mmap()访问 I/O 内存

在[第六章](https://cdp.packtpub.com/linux_device_driver_development_cookbook/wp-admin/post.php?post=30&action=edit#post_29)的*杂项内核内部*中的*获取 I/O 内存访问*中，我们看到了 MMU 的工作原理以及如何访问内存映射的外围设备。在内核空间中，我们必须指示 MMU 以便正确地将虚拟地址转换为一个正确的地址，这个地址必须指向我们外围设备所属的一个明确定义的物理地址，否则我们无法控制它！

另一方面，在该部分，我们还使用了一个名为`devmem2`的用户空间工具，它可以使用`mmap()`系统调用从用户空间访问物理地址。这个系统调用非常有趣，因为它允许我们做很多有用的事情，所以让我们先来看一下它的 man 页面（`man 2 mmap`）：

```
NAME
   mmap, munmap - map or unmap files or devices into memory

SYNOPSIS
   #include <sys/mman.h>

   void *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset);
   int munmap(void *addr, size_t length);

DESCRIPTION
   mmap() creates a new mapping in the virtual address space of the call‐
   ing process. The starting address for the new mapping is specified in
   addr. The length argument specifies the length of the mapping (which
   must be greater than 0).
```

正如我们从前面的片段中看到的，通过使用`mmap()`，我们可以在调用进程的虚拟地址空间中创建一个新的映射，这个映射可以与作为参数传递的文件描述符`fd`相关联。

通常，此系统调用用于以这样的方式将普通文件映射到系统内存中，以便可以使用普通指针而不是通常的`read()`和`write()`系统调用来寻址。

举个简单的例子，让我们考虑一个通常的文件如下：

```
$ cat textfile.txt 
This is a test file

This is line 3.

End of the file
```

这是一个包含三行文本的普通文本文件。我们可以在终端上使用`cat`命令读取和写入它，就像之前所述的那样；当然，我们现在知道`cat`命令在文件上运行`open()`，然后是一个或多个`read()`操作，然后是一个或多个`write()`操作，最后是标准输出（反过来是连接到我们终端的文件抽象）。但是，这个文件也可以被读取为一个 char 的内存缓冲区，使用`mmap()`系统调用，可以通过以下步骤完成：

```
    ret = open(argv[1], O_RDWR);
    if (ret < 0) {
        perror("open");
        exit(EXIT_FAILURE);
    }
    printf("file %s opened\n", argv[1]);
    fd = ret;

    /* Try to remap file into memory */
    addr = mmap(NULL, len, PROT_READ | PROT_WRITE,
                MAP_FILE | MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED) {
        perror("mmap");
        exit(EXIT_FAILURE);
    }

    ptr = (char *) addr;
    for (i = 0; i < len; i++)
        printf("%c", ptr[i]);
```

前面示例的完整代码实现将在以下片段中呈现。这是`chrdev_mmap.c`文件的片段。

因此，正如我们所看到的，我们首先像往常一样打开文件，但是，我们没有使用`read()`系统调用，而是使用了`mmap()`，最后，我们使用返回的内存地址作为 char 指针来打印内存缓冲区。请注意，在`mmap()`之后，我们将在内存中得到文件的图像。

如果我们尝试在`textfile.txt`文件上执行前面的代码，我们会得到我们期望的结果：

```
# ls -l textfile.txt 
-rw-r--r-- 1 root root 54 May 11 16:41 textfile.txt
# ./chrdev_mmap textfile.txt 54 
file textfile.txt opened
got address=0xffff8357b000 and len=54
---
This is a test file

This is line 3.

End of the file
```

请注意，我使用`ls`命令获取了`chrdev_mmap`程序所需的文件长度。

现在我们应该问自己是否有办法像上面的文本文件一样映射字符设备（从用户空间的角度看起来非常类似文件）；显然，答案是肯定的！我们必须使用`struct file_operations`中定义的`mmap()`方法：

```
struct file_operations {
...
        int (*mmap) (struct file *, struct vm_area_struct *);
```

除了我们已经完全了解的通常的`struct file`指针之外，此函数还需要`vma`参数（指向`struct vm_area_struct`的指针），用于指示应该由驱动程序映射内存的虚拟地址空间。

`struct vm_area_struct`包含有关连续虚拟内存区域的信息，其特征是起始地址、停止地址、长度和权限。

每个进程拥有更多的虚拟内存区域，可以通过查看名为`/proc/<PID>/maps`的相对 procfs 文件来检查（其中`<PID>`是进程的 PID 号）。

虚拟内存区域是 Linux 内存管理器的一个非常复杂的部分，本书未涉及。好奇的读者可以查看[`www.kernel.org/doc/html/latest/admin-guide/mm/index.html`](https://www.kernel.org/doc/html/latest/admin-guide/mm/index.html)以获取更多信息。

将物理地址映射到用户地址空间，如`vma`参数所示，可以使用辅助函数轻松完成，例如在头文件`linux/include/linux/mm.h`中定义的`remap_pfn_range()`：

```
int remap_pfn_range(structure vm_area_struct *vma,
                    unsigned long addr,
                    unsigned long pfn, unsigned long size,
                    pgprot_t prot);
```

它将由`pfn`寻址的连续物理地址空间映射到由`vma`指针表示的虚拟空间。具体来说，参数是：

+   `vma` - 进行映射的虚拟内存空间

+   `addr` - 重新映射开始的虚拟地址空间

+   `pfn` - 虚拟地址应映射到的物理地址（以页面帧号表示）

+   `size` - 要映射的内存大小（以字节为单位）

+   `prot` - 此映射的保护标志

因此，一个真正简单的`mmap()`实现，考虑到外围设备在物理地址`base_addr`处具有内存区域，大小为`area_len`，可以如下所示：

```
static int my_mmap(struct file *filp, struct vm_area_struct *vma)
{
    struct my_device *my_ptr = filp->private_data;
    size_t size = vma->vm_end - vma->vm_start;
    phys_addr_t offset = (phys_addr_t) vma->vm_pgoff << PAGE_SHIFT;
    unsigned long pfn;

    /* Does it even fit in phys_addr_t? */
    if (offset >> PAGE_SHIFT != vma->vm_pgoff)
        return -EINVAL;

    /* We cannot mmap too big areas */
    if ((offset > my_ptr->area_len) ||
        (size > my_ptr->area_len - offset))
        return -EINVAL;

    /* Remap-pfn-range will mark the range VM_IO */
    if (remap_pfn_range(vma, vma->vm_start,
                        my_ptr->base_addr, size,
                        vma->vm_page_prot))
        return -EAGAIN;

    return 0;
}
```

最后需要注意的是，`remap_pfn_range()`使用物理地址，而使用`kmalloc()`或`vmalloc()`函数和相关函数（参见[第六章](https://cdp.packtpub.com/linux_device_driver_development_cookbook/wp-admin/post.php?post=30&action=edit#post_29)，*杂项内核内部*）分配的内存必须使用不同的方法进行管理。对于`kmalloc()`，我们可以使用以下方法来获取`pfn`参数：

```
unsigned long pfn = virt_to_phys(kvirt) >> PAGE_SHIFT;
```

其中 kvirt 是由`kmalloc()`返回的内核虚拟地址要重新映射，对于`vmalloc()`，我们可以这样做：

```
unsigned long pfn = vmalloc_to_pfn(vvirt);
```

在这里，`vvirt`是由`vmalloc()`返回的内核虚拟地址要重新映射。

请注意，使用`vmalloc()`分配的内存不是物理上连续的，因此，如果我们想要映射使用它分配的范围，我们必须逐个映射每个页面，并计算每个页面的物理地址。这是一个更复杂的操作，本书没有解释，因为它与设备驱动程序无关（真正的外围设备只使用物理地址）。

# 使用进程上下文进行锁定

了解如何避免竞争条件是很重要的，因为可能会有多个进程尝试访问我们的驱动程序，或者如何使读取进程进入睡眠状态（我们在这里讨论读取，但对于写入也是一样的）如果我们的驱动程序没有数据供应。前一种情况将在这里介绍，而后一种情况将在下一节介绍。

如果我们看一下我们的 chrdev 驱动程序中如何实现`read()`和`write()`系统调用，我们很容易注意到，如果多个进程尝试进行`read()`调用，甚至如果一个进程尝试进行`read()`调用，另一个尝试进行`write()`调用，就会发生竞争条件。这是因为 ESPRESSObin 的 CPU 是由两个核心组成的多处理器，因此它可以有效地同时执行两个进程。

然而，即使我们的系统只有一个核心，由于例如函数`copy_to_user()`和`copy_from_user()`可能使调用进程进入睡眠状态，因此调度程序可能会撤销 CPU 以便将其分配给另一个进程，这样，即使我们的系统只有一个核心，仍然可能发生`read()`或`write()`方法内部的代码以交错（即非原子）方式执行。

为了避免这些情况可能发生的竞争条件，一个真正可靠的解决方案是使用互斥锁，正如第五章中所介绍的那样，*管理中断和并发*。

我们只需要为每个 chrdev 设备使用一个互斥锁来保护对驱动程序方法的多次访问。

# 使用 poll()和 select()等待 I/O 操作

在现代计算机这样的复杂系统中，通常会有几个有用的外围设备来获取有关外部环境和/或系统状态的信息。有时，我们可能使用不同的进程来管理它们，但可能需要同时管理多个外围设备，但只有一个进程。

在这种情况下，我们可以想象对每个外围设备进行多次`read()`系统调用来获取其数据，但是如果一个外围设备非常慢，需要很长时间才能返回其数据会发生什么？如果我们这样做，可能会减慢所有数据采集的速度（甚至如果一个外围设备没有接收到新数据，可能会锁定数据采集）：

```
fd1 = open("/dev/device1", ...);
fd2 = open("/dev/device2", ...);
fd3 = open("/dev/device3", ...);

while (1) {
    read(fd1, buf1, size1);
    read(fd2, buf2, size2);
    read(fd3, buf3, size3);

    /* Now use data from peripherals */
    ...
}
```

实际上，如果一个外围设备很慢，或者需要很长时间才能返回其数据，我们的循环将停止等待它，我们的程序可能无法正常工作。

一个可能的解决方案是在有问题的外围设备上使用`O_NONBLOCK`标志，甚至在所有外围设备上使用，但这样做可能会使 CPU 过载，产生不必要的系统调用。向内核询问哪个文件描述符属于持有准备好被读取的数据的外围设备（或者可以用于写入）可能更加优雅（和有效）。

为此，我们可以使用`poll()`或`select()`系统调用。`poll()`手册页中指出：

```
NAME
   poll, ppoll - wait for some event on a file descriptor

SYNOPSIS
   #include <poll.h>

   int poll(struct pollfd *fds, nfds_t nfds, int timeout);

   #define _GNU_SOURCE /* See feature_test_macros(7) */
   #include <signal.h>
   #include <poll.h>

   int ppoll(struct pollfd *fds, nfds_t nfds,
           const struct timespec *tmo_p, const sigset_t *sigmask);
```

另一方面，`select()`手册页如下所示：

```
NAME
  select, pselect, FD_CLR, FD_ISSET, FD_SET, FD_ZERO - synchronous I/O
   multiplexing

SYNOPSIS
   /* According to POSIX.1-2001, POSIX.1-2008 */
   #include <sys/select.h>

   /* According to earlier standards */
   #include <sys/time.h>
   #include <sys/types.h>
   #include <unistd.h>

   int select(int nfds, fd_set *readfds, fd_set *writefds,
              fd_set *exceptfds, struct timeval *timeout);

   void FD_CLR(int fd, fd_set *set);
   int FD_ISSET(int fd, fd_set *set);
   void FD_SET(int fd, fd_set *set);
   void FD_ZERO(fd_set *set);
```

即使它们看起来非常不同，它们几乎做相同的事情；实际上，在内核内部，它们是通过使用相同的`poll()`方法来实现的，该方法在著名的`struct file_operations`中定义如下（请参阅`linux/include/linux/fs.h`文件）：

```
struct file_operations {
...
    __poll_t (*poll) (struct file *, struct poll_table_struct *);
```

从内核的角度来看，`poll()`方法的实现非常简单；我们只需要上面使用的等待队列，然后我们必须验证我们的设备是否有一些数据要返回。简而言之，通用的`poll()`方法如下所示：

```
static __poll_t simple_poll(struct file *filp, poll_table *wait)
{
    struct simple_device *chrdev = filp->private_data;
    __poll_t mask = 0;

    poll_wait(filp, &simple_device->queue, wait);

    if (has_data_to_read(simple_device))
        mask |= EPOLLIN | EPOLLRDNORM;

    if (has_space_to_write(simple_device))
        mask |= EPOLLOUT | EPOLLWRNORM;

    return mask;
}
```

我们只需使用`poll_wait()`函数告诉内核驱动程序使用哪个等待队列来使读取或写入进程进入睡眠状态，然后我们将变量`mask`返回为 0；如果没有准备好要读取的数据，或者我们无法接受新的要写入的数据，我们将返回`EPOLLIN | EPOLLRDNORM`值，如果有一些数据可以按位读取，并且我们也愿意接受这些数据。

所有可用的`poll()`事件都在头文件`linux/include/uapi/linux/eventpoll.h`中定义。

一旦`poll()`方法被实现，我们可以使用它，例如，如下所示使用`select()`：

```
fd_set read_fds;

fd1 = open("/dev/device1", ...);
fd2 = open("/dev/device2", ...);
fd3 = open("/dev/device3", ...);

while (1) {
    FD_ZERO(&read_fds);
    FD_SET(fd1, &read_fds);
    FD_SET(fd2, &read_fds);
    FD_SET(fd2, &read_fds);

    select(FD_SETSIZE, &read_fds, NULL, NULL, NULL);

    if (FD_ISSET(fd1, &read_fds))
        read(fd1, buf1, size1);
    if (FD_ISSET(fd2, &read_fds))
        read(fd2, buf2, size2);
    if (FD_ISSET(fd3, &read_fds))
        read(fd3, buf3, size3);

    /* Now use data from peripherals */
    ...
}
```

打开所有需要的文件描述符后，我们必须使用`FD_ZERO()`宏清除`read_fds`变量，然后使用`FD_SET()`宏将每个文件描述符添加到由`read_fds`表示的读取进程集合中。完成后，我们可以将`read_fds`传递给`select()`，以指示内核要观察哪些文件描述符。

请注意，通常情况下，我们应该将观察集合中文件描述符的最高编号加 1 作为`select()`系统调用的第一个参数；然而，我们也可以传递`FD_SETSIZE`值，这是系统允许的最大允许值。这可能是一个非常大的值，因此以这种方式编程会导致扫描整个文件描述符位图的低效性；好的程序员应该使用最大值加 1。

另外，请注意，我们的示例适用于读取，但完全相同的方法也适用于写入！

# 使用`fasync()`管理异步通知

在前一节中，我们考虑了一个特殊情况，即我们可能需要管理多个外围设备的情况。在这种情况下，我们可以询问内核，即准备好的文件描述符，从哪里获取数据或使用`poll()`或`select()`系统调用将数据写入。然而，这不是唯一的解决方案。另一种可能性是使用`fasync()`方法。

通过使用这种方法，我们可以要求内核在文件描述符上发生新事件时发送信号（通常是`SIGIO`）；当然，事件是准备好读取或准备好写入的事件，文件描述符是与我们的外围设备连接的文件描述符。

由于本书中已经介绍的方法，`fasync()`方法没有用户空间对应项；根本没有`fasync()`系统调用。我们可以通过利用`fcntl()`系统调用间接使用它。如果我们查看它的手册页，我们会看到以下内容：

```
NAME
   fcntl - manipulate file descriptor

SYNOPSIS
   #include <unistd.h>
   #include <fcntl.h>

   int fcntl(int fd, int cmd, ... /* arg */ );

...

   F_SETOWN (int)
          Set the process ID or process group ID that will receive SIGIO
          and SIGURG signals for events on the file descriptor fd. The
          target process or process group ID is specified in arg. A
          process ID is specified as a positive value; a process group ID
          is specified as a negative value. Most commonly, the calling
          process specifies itself as the owner (that is, arg is specified
          as getpid(2)).
```

现在，让我们一步一步来。从内核的角度来看，我们必须实现`fasync()`方法，如下所示（请参阅`linux/include/linux/fs.h`文件中的`struct file_operations`）：

```
struct file_operations {
...
    int (*fsync) (struct file *, loff_t, loff_t, int datasync);
```

它的实现非常简单，因为通过使用`fasync_helper()`辅助函数，我们只需要在以下通用驱动程序中报告的步骤：

```
static int simple_fasync(int fd, struct file *filp, int on)
{
    struct simple_device *simple = filp->private_data;

    return fasync_helper(fd, filp, on, &simple->fasync_queue);
}
```

其中，`fasync_queue`是一个指向`struct fasync_struct`的指针，内核使用它来排队所有对接收`SIGIO`信号感兴趣的进程，每当驱动程序准备好进行读取或写入操作时。这些事件使用`kill_fasync()`函数通知，通常在中断处理程序中或者每当我们知道新数据已经到达或者我们准备写入时。

```
kill_fasync(&simple->fasync_queue, SIGIO, POLL_IN);
```

请注意，当数据可供读取时，我们必须使用`POLL_IN`，而当我们的外围设备准备好接受新数据时，我们应该使用`POLL_OUT`。

请参阅`linux/include/uapi/asm-generic/siginfo.h`文件，查看所有可用的`POLL_*`定义。

从用户空间的角度来看，我们需要采取一些步骤来实现`SIGIO`信号：

1.  首先，我们必须安装一个合适的信号处理程序。

1.  然后，我们必须使用`F_SETOWN`命令调用`fcntl()`来设置将接收与我们的设备相关的`SIGIO`的进程 ID（通常称为 PID）（由文件描述符`fd`表示）。

1.  然后，我们必须通过设置`FASYNC`位来更改描述文件访问模式的`flags`。

一个可能的实现如下：

```
long flags;

fd = open("/dev/device", ...);

signal(SIGIO, sigio_handler);

fcntl(fd, F_SETOWN, getpid());

flags = fcntl(fd, F_GETFL);

fcntl(fd, F_SETFL, flags | FASYNC);
```
