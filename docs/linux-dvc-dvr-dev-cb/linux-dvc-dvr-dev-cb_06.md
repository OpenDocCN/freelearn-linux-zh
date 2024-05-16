# 第六章：杂项内核内部

在内核开发中，我们可能需要执行一些杂项活动来实现我们的设备驱动程序，例如动态分配内存并使用特定的数据类型来存储寄存器数据，或者简单地等待一段时间，以确保外围设备已完成其复位过程。

为了执行所有这些任务，Linux 为内核开发人员提供了一套丰富的有用函数、宏和数据类型，我们将尝试通过非常简单的示例代码在本章中介绍它们，因为我们希望向读者指出如何使用它们来简化设备驱动程序开发。因此，在本章中，我们将涵盖以下内容：

+   使用内核数据类型

+   管理辅助函数

+   动态内存分配

+   管理内核链表

+   使用内核哈希表

+   访问 I/O 内存

+   在内核中花费时间

# 技术要求

有关本章的更多信息，您可以访问*附录*。

本章中使用的代码和其他文件可以从 GitHub 下载[`github.com/giometti/linux_device_driver_development_cookbook/tree/master/chapter_06`](https://github.com/giometti/linux_device_driver_development_cookbook/tree/master/chapter_06)。

# 使用内核数据类型

通常，内核代码需要特定大小的数据项来匹配预定义的二进制结构，保存外围设备的寄存器数据，与用户空间通信，或者仅仅通过插入填充字段在结构内对齐数据。

有时，内核代码需要特定大小的数据项，也许是为了匹配预定义的二进制结构，与用户空间通信，保存外围设备的寄存器数据，或者仅仅通过插入填充字段在结构内对齐数据。

在本节中，我们将看到一些特殊的数据类型，内核开发人员可以使用这些类型来简化他们的日常工作。接下来，我们将看到一个**固定大小数据类型**的示例，这些类型非常有用，可以定义与设备或通信协议期望的数据结构完全匹配的数据类型；细心的读者会认识到，确实无法使用标准 C 类型来定义这种固定大小的数据实体，因为 C 标准并没有明确保证在所有架构中都有固定大小的表示，当我们使用类似标准 C 类型如`int`、`short`或`long`时。

内核提供以下数据类型，以便在需要知道数据大小时使用（它们的实际定义取决于当前使用的架构，但它们在不同架构中都被命名为相同）：

+   `u8`: 无符号字节（8 位）

+   `u16`: 无符号字（16 位）

+   `u32`: 无符号 32 位（32 位）

+   `u64`: 无符号 64 位（64 位）

+   `s8`: 有符号字节（8 位）

+   `s16`: 有符号字（16 位）

+   `s32`: 有符号 32 位（32 位）

+   `s64`: 有符号 64 位（64 位）

有时，固定大小的数据类型必须用于与用户空间交换数据；然而，在这种情况下，我们不能使用前面的类型，而必须选择以下替代数据类型，这些类型等同于前面的类型，但可以在内核和用户空间中任意使用（这个概念将在第七章*,*高级字符驱动程序操作*中的*使用 ioctl()方法*中变得更加清晰）：

+   `__u8`: 无符号字节（8 位）

+   `__u16`: 无符号字（16 位）

+   `__u32`: 无符号 32 位（32 位）

+   `__u64`: 无符号 64 位（64 位）

+   `__s8`: 有符号字节（8 位）

+   `__s16`: 有符号字（16 位）

+   `__s32`: 有符号 32 位（32 位）

+   `__s64`: 有符号 64 位（64 位）

所有这些固定大小的类型都在头文件`linux/include/linux/types.h`中定义。

# 做好准备

为了展示如何使用前面的数据类型，我们可以再次使用一个内核模块来执行一些内核代码，其中使用它们来定义结构中的寄存器映射。

# 如何做...

让我们看看如何通过以下步骤来做到这一点：

1.  让我们看看`data_type.c`文件，我们将所有代码放入模块的`init()`函数中，如下所示：

```
static int __init data_types_init(void)
{
    struct dtypes_s *ptr = (struct dtypes_s *) base_addr;

    pr_info("\tu8\tu16\tu32\tu64\n");
    pr_info("size\t%ld\t%ld\t%ld\t%ld\n",
        sizeof(u8), sizeof(u16), sizeof(u32), sizeof(u64));

    pr_info("name\tptr\n");
    pr_info("reg0\t%px\n", &ptr->reg0);
    pr_info("reg1\t%px\n", &ptr->reg1);
    pr_info("reg2\t%px\n", &ptr->reg2);
    pr_info("reg3\t%px\n", &ptr->reg3);
    pr_info("reg4\t%px\n", &ptr->reg4);
    pr_info("reg5\t%px\n", &ptr->reg5);

    return -EINVAL;
}
```

# 工作原理...

在执行*步骤 1*之后，指针`ptr`将根据`base_addr`的值进行初始化，以便通过简单地引用`struct dtypes_s`的字段（在以下代码中定义）来指向正确的内存地址：

```
struct dtypes_s {
    u32 reg0;
    u8 pad0[2];
    u16 reg1;
    u32 pad1[2];
    u8 reg2;
    u8 reg3;
    u16 reg4;
    u32 reg5;
} __attribute__ ((packed));
```

在结构定义期间，我们应该意识到编译器可能会在结构本身中悄悄地插入填充，以确保每个字段都正确对齐，以便在目标处理器上获得良好的性能；避免这种行为的一种解决方法是告诉编译器结构必须是紧凑的，不添加填充。当然，这可以通过使用`__attribute__ ((packed))`来实现，就像以前一样。

# 还有更多...

如果我们希望验证这一步，我们可以通过测试代码来做到这一点。我们只需要像往常一样编译模块，然后将其移动到 ESPRESSObin，最后按照以下步骤插入内核：

```
# insmod data_types.ko 
```

您还应该收到以下错误消息：

`insmod: ERROR: could not insert module data_types.ko: Invalid parameters`

然而，这是由于`data_types_init()`函数中的最后一个`return -EINVAL`；我们在这里和接下来使用这个技巧，强制内核在模块的`init()`函数执行后移除模块。

我们得到的内核消息中的第一行是关于`u8`、`u16`、`u32`和`u64`类型的维度如下：

```
data_types:data_types_init:      u8 u16 u32 u64
data_types:data_types_init: size 1  2   4   8
```

然后，以下行（仍然在内核消息中）向我们展示了通过使用带有`u8`、`u16`、`u32`和`u64`的结构定义以及`__attribute__ ((packed))`语句可以实现的完美填充：

```
data_types:data_types_init: name ptr
data_types:data_types_init: reg0 0000000080000000
data_types:data_types_init: reg1 0000000080000006
data_types:data_types_init: reg2 0000000080000010
data_types:data_types_init: reg3 0000000080000011
data_types:data_types_init: reg4 0000000080000012
data_types:data_types_init: reg5 0000000080000014
```

# 另请参阅

+   关于内核数据类型的良好参考资料可以在[`kernelnewbies.org/InternalKernelDataTypes`](https://kernelnewbies.org/InternalKernelDataTypes)找到。

# 管理辅助函数

在设备驱动程序开发过程中，我们可能需要连接字符串或计算其长度，或者只是复制或移动内存区域（或字符串）。为了在用户空间执行这些常见操作，我们可以使用几个函数，比如`strcat()`、`strlen()`、`memcpy()`（或`strcpy()`）等等，Linux 也为我们提供了类似命名的函数，当然，这些函数在内核中是安全可用的。（请注意，内核代码不能链接到用户空间的 glibc 库。）

在本教程中，我们将看到如何使用一些内核辅助程序来管理内核中的字符串。

# 准备工作

如果我们在内核源代码中查看`linux/include/linux/string.h`包含文件，我们可以看到一长串通常的用户空间类似实用函数，如下所示：

```
#ifndef __HAVE_ARCH_STRCPY
extern char * strcpy(char *,const char *);
#endif
#ifndef __HAVE_ARCH_STRNCPY
extern char * strncpy(char *,const char *, __kernel_size_t);
#endif
#ifndef __HAVE_ARCH_STRLCPY
size_t strlcpy(char *, const char *, size_t);
#endif
#ifndef __HAVE_ARCH_STRSCPY
ssize_t strscpy(char *, const char *, size_t);
#endif
#ifndef __HAVE_ARCH_STRCAT
extern char * strcat(char *, const char *);
#endif
#ifndef __HAVE_ARCH_STRNCAT
extern char * strncat(char *, const char *, __kernel_size_t);
#endif
...
```

请注意，每个函数都被包含在`#ifndef`/`#endif`预处理器条件子句中，因为这些函数中的一些可以使用某种形式的优化来实现；因此，它们的实现可能在不同平台上有所不同。

为了展示如何使用前面的辅助函数，我们可以再次使用一个内核模块来执行使用其中一些函数的内核代码。

# 如何做...

让我们看看如何通过以下步骤来做到这一点：

1.  在`helper_funcs.c`文件中，我们可以看到一些非常愚蠢的代码，它演示了我们如何使用这些辅助函数。

鼓励您修改此代码以使用不同的内核辅助函数。

1.  所有的工作都是在模块的`init()`函数中完成的，就像在前面的部分一样。在这里，我们可以使用内核函数`strlen()`和`strncpy()`，就像它们的用户空间对应函数一样：

```
static int __init helper_funcs_init(void)
{
    char str2[STR2_LEN];

    pr_info("str=\"%s\"\n", str);
    pr_info("str size=%ld\n", strlen(str));

    strncpy(str2, str, STR2_LEN);

    pr_info("str2=\"%s\"\n", str2);
    pr_info("str2 size=%ld\n", strlen(str2));

    return -EINVAL;
}
```

这些函数是特殊的内核实现，它们不是我们通常在正常编程中使用的用户空间函数。我们不能将内核模块与 glibc 链接！

1.  `str`字符串定义为模块参数如下，并且可以用于尝试不同的字符串：

```
static char *str = "default string";
module_param(str, charp, S_IRUSR | S_IWUSR);
MODULE_PARM_DESC(str, "a string value");
```

# 还有更多...

如果您希望测试该示例中的代码，可以通过编译它然后将其移动到 ESPRESSObin 来进行测试。

首先，我们必须将模块插入内核：

```
# insmod helper_funcs.ko
```

您可以安全地忽略以下错误消息，如之前讨论的那样：

`insmod: ERROR: could not insert module helper_funcs.ko: Invalid parameters`

内核消息现在应该如下所示：

```
helper_funcs:helper_funcs_init: str="default string"
helper_funcs:helper_funcs_init: str size=14
helper_funcs:helper_funcs_init: str2="default string"
helper_funcs:helper_funcs_init: str2 size=14
```

在前面的输出中，我们可以看到字符串`str2`只是`str`的副本。

但是，如果我们使用以下`insmod`命令，输出将会发生变化：

```
# insmod helper_funcs.ko str=\"very very very loooooooooong string\"
helper_funcs:helper_funcs_init: str="very very very loooooooooong string"
helper_funcs:helper_funcs_init: str size=35
helper_funcs:helper_funcs_init: str2="very very very loooooooooong str"
helper_funcs:helper_funcs_init: str2 size=32
```

再次，字符串`str2`是`str`的副本，但其最大大小`STR2_LEN`定义如下：

```
#define STR2_LEN    32
```

# 另请参见

+   有关更完整的字符串操作函数列表，一个很好的起点是[`www.kernel.org/doc/htmldocs/kernel-api/ch02s02.html`](https://www.kernel.org/doc/htmldocs/kernel-api/ch02s02.html)。

+   关于字符串转换，您可以查看[`www.kernel.org/doc/htmldocs/kernel-api/libc.html#id-1.4.3`](https://www.kernel.org/doc/htmldocs/kernel-api/libc.html#id-1.4.3)。

# 动态内存分配

一个好的设备驱动程序不应该支持多个外围设备（可能）也不应该是固定数量的！但是，即使我们决定将驱动程序的使用限制为一个外围设备，也可能需要管理可变数量的数据块，因此无论如何，我们都需要能够管理**动态内存分配**。

在这个示例中，我们将看到如何在内核空间动态（并安全地）分配内存块。

# 如何做...

为了展示我们如何通过使用`kmalloc()`、`vmalloc()`和`kvmalloc()`从内核中分配内存，我们可以再次使用一个内核模块。

在`mem_alloc.c`文件中，我们可以看到一些非常简单的代码，显示了内存分配如何与相关的内存释放函数一起工作：

1.  所有的工作都是在模块的`init()`函数中完成的，就像以前一样。第一步是使用两个不同标志的`kmalloc()`，即`GFP_KERNEL`（可以休眠）和`GFP_ATOMIC`（不休眠，然后可以安全地在中断上下文中使用）：

```
static int __init mem_alloc_init(void)
{
    void *ptr;

    pr_info("size=%ldkbytes\n", size);

    ptr = kmalloc(size << 10, GFP_KERNEL);
    pr_info("kmalloc(..., GFP_KERNEL) =%px\n", ptr);
    kfree(ptr);

    ptr = kmalloc(size << 10, GFP_ATOMIC);
    pr_info("kmalloc(..., GFP_ATOMIC) =%px\n", ptr);
    kfree(ptr);
```

1.  然后，我们尝试使用`vmalloc()`来分配内存：

```
    ptr = vmalloc(size << 10);
    pr_info("vmalloc(...) =%px\n", ptr);
    vfree(ptr);
```

1.  最后，我们尝试使用`kvmalloc()`和两个不同的标志进行两种不同的分配，即`GFP_KERNEL`（可以休眠）和`GFP_ATOMIC`（不休眠，然后可以安全地在中断上下文中使用）：

```
    ptr = kvmalloc(size << 10, GFP_KERNEL);
    pr_info("kvmalloc(..., GFP_KERNEL)=%px\n", ptr);
    kvfree(ptr);

    ptr = kvmalloc(size << 10, GFP_ATOMIC);
    pr_info("kvmalloc(..., GFP_ATOMIC)=%px\n", ptr);
    kvfree(ptr);

    return -EINVAL;
}
```

请注意，对于每个分配函数，我们必须使用相关的`free()`函数！

要分配的内存块的大小作为内核参数传递如下：

```
static long size = 4;
module_param(size, long, S_IRUSR | S_IWUSR);
MODULE_PARM_DESC(size, "memory size in Kbytes");
```

# 还有更多...

好的，就像以前一样，只需编译模块，然后将其移动到 ESPRESSObin。

如果我们尝试使用默认内存大小（即 4 KB）插入模块，我们应该会得到以下内核消息：

```
# insmod mem_alloc.ko
mem_alloc:mem_alloc_init: size=4kbytes
mem_alloc:mem_alloc_init: kmalloc(..., GFP_KERNEL) =ffff800079831000
mem_alloc:mem_alloc_init: kmalloc(..., GFP_ATOMIC) =ffff800079831000
mem_alloc:mem_alloc_init: vmalloc(...) =ffff000009655000
mem_alloc:mem_alloc_init: kvmalloc(..., GFP_KERNEL)=ffff800079831000
mem_alloc:mem_alloc_init: kvmalloc(..., GFP_ATOMIC)=ffff800079831000
```

您可以安全地忽略以下错误消息，如前面讨论的那样：

`insmod: ERROR: could not insert module mem_alloc.ko: Invalid parameters`

这向我们表明所有分配函数都成功地完成了它们的工作。

但是，如果我们尝试增加内存块大小如下，会发生一些变化：

```
root@espressobin:~# insmod mem_alloc.ko size=5000
mem_alloc:mem_alloc_init: size=5000kbytes
mem_alloc:mem_alloc_init: kmalloc(..., GFP_KERNEL) =0000000000000000
mem_alloc:mem_alloc_init: kmalloc(..., GFP_ATOMIC) =0000000000000000
mem_alloc:mem_alloc_init: vmalloc(...) =ffff00000b9fb000
mem_alloc:mem_alloc_init: kvmalloc(..., GFP_KERNEL)=ffff00000c135000
mem_alloc:mem_alloc_init: kvmalloc(..., GFP_ATOMIC)=0000000000000000
```

现在`kmalloc()`函数失败，而`vmalloc()`由于它在非连续物理地址上分配虚拟内存空间而仍然成功。另一方面，当使用标志`GFP_KERNEL`调用`kvmalloc()`时成功，而使用标志`GFP_ATOMIC`时失败。（这是因为在这种特殊情况下它不能使用`vmalloc()`作为后备。）

# 另请参见

+   有关内存分配的更多信息，一个很好的起点是[`www.kernel.org/doc/html/latest/core-api/memory-allocation.html`](https://www.kernel.org/doc/html/latest/core-api/memory-allocation.html)。

# 管理内核链接列表

在内核内编程时，有能力管理数据列表可能非常有用，为了减少重复的代码量，内核开发人员创建了循环双向链表的标准实现。

在这个示例中，我们将看到如何使用 Linux API 在我们的代码中使用列表。

# 准备工作

为了演示列表 API 的工作原理，我们可以再次使用内核模块，在模块的`init()`函数中执行一些操作，就像以前一样。

# 如何做...

在`list.c`文件中，有我们的示例代码，所有游戏都在`list_init()`函数中进行：

1.  首先，让我们看一下实现列表元素和列表头的结构的声明：

```
static LIST_HEAD(data_list);

struct l_struct {
    int data;
    struct list_head list;
};
```

1.  现在，在`list_init()`中，我们定义了我们的元素：

```
static int __init list_init(void)
{
    struct l_struct e1 = {
        .data = 5
    };
    struct l_struct e2 = {
        .data = 1
    }; 
    struct l_struct e3 = {
        .data = 7
    };
```

1.  然后，我们向列表中添加第一个元素并打印它：

```
    pr_info("add e1...\n");
    add_ordered_entry(&e1);
    print_entries();
```

1.  接下来，我们继续添加元素并打印列表：

```
    pr_info("add e2, e3...\n");
    add_ordered_entry(&e2);
    add_ordered_entry(&e3);
    print_entries();
```

1.  最后，我们删除一个元素：

```
    pr_info("del data=5...\n");
    del_entry(5);
    print_entries();

    return -EINVAL;
}
```

1.  现在，让我们看看本地函数定义；要以有序模式添加元素，我们可以这样做：

```
static void add_ordered_entry(struct l_struct *new)
{
    struct list_head *ptr;
    struct l_struct *entry;

    list_for_each(ptr, &data_list) {
        entry = list_entry(ptr, struct l_struct, list);
        if (entry->data < new->data) {
            list_add_tail(&new->list, ptr);
            return;
        }
    }
    list_add_tail(&new->list, &data_list);
}
```

1.  与此同时，可以按照以下步骤进行条目删除：

```
static void del_entry(int data)
{
    struct list_head *ptr;
    struct l_struct *entry;

    list_for_each(ptr, &data_list) {
        entry = list_entry(ptr, struct l_struct, list);
        if (entry->data == data) {
            list_del(ptr);
            return;
        }
    }
}
```

1.  最后，可以通过以下方式打印列表中的所有元素：

```
static void print_entries(void)
{
    struct l_struct *entry;

    list_for_each_entry(entry, &data_list, list)
        pr_info("data=%d\n", entry->data);
}
```

在最后的函数中，我们使用宏`list_for_each_entry()`而不是`list_for_each()`和`list_entry()`的组合，以获得更紧凑和可读的代码，它本质上执行相同的步骤。

该宏在`linux/include/linux/list.h`文件中定义如下：

```
/**
 * list_for_each_entry - iterate over list of given type
 * @pos: the type * to use as a loop cursor.
 * @head: the head for your list.
 * @member: the name of the list_head within the struct.
 */
#define list_for_each_entry(pos, head, member) \
        for (pos = list_first_entry(head, typeof(*pos), member); \
             &pos->member != (head); \
             pos = list_next_entry(pos, member))
```

# 还有更多...

我们可以在编译并插入到 ESPRESSObin 的内核后测试代码。要插入内核，我们使用通常的`insmod`命令：

```
# insmod list.ko 
```

您可以安全地忽略以下错误消息，如前所述：

`insmod: ERROR: could not insert module list.ko: Invalid parameters`

然后，在第一次插入后，我们得到了以下内核消息：

```
list:list_init: add e1...
list:print_entries: data=5
```

在*步骤 1*和*步骤 2*中，我们定义了列表的元素，而在*步骤 3*中，我们进行了第一次插入到列表中，之前的消息是插入后得到的结果。

在*步骤 4*中进行第二次插入后，我们得到了以下结果：

```
list:list_init: add e2, e3...
list:print_entries: data=7
list:print_entries: data=5
list:print_entries: data=1
```

最后，在*步骤 5*删除后，列表变为如下：

```
list:list_init: del data=5...
list:print_entries: data=7
list:print_entries: data=1
```

请注意，在*步骤 6*中，我们提出了有序模式下元素插入的可能实现，但是开发人员可以根据实际情况选择最佳解决方案。对于*步骤 7*，我们实现了元素移除，而在*步骤 8*中，我们有打印函数。

# 另请参阅

+   关于 Linux 列表 API 的更完整的函数列表，可以在[`www.kernel.org/doc/htmldocs/kernel-api/adt.html#id-1.3.2`](https://www.kernel.org/doc/htmldocs/kernel-api/adt.html#id-1.3.2)找到一个很好的参考。

# 使用内核哈希表

与内核列表一样，Linux 为内核开发人员提供了一个通用接口来管理哈希表。它们的实现是基于前一节中看到的内核列表的特殊版本，并命名为`hlist`（仍然是双向链表，但是只有一个指针列表头）。该 API 在头文件`linux/include/linux/hashtable.h`中定义。

在这个示例中，我们将展示如何使用哈希表在内核代码中使用 Linux API。

# 准备工作

即使在这个示例中，我们也可以使用内核模块来查看测试代码的工作原理。

# 如何做...

在`hashtable.c`文件中，实现了一个与内核列表中提出的非常相似的示例：

1.  作为第一步，我们声明哈希表、数据结构和哈希函数如下：

```
static DEFINE_HASHTABLE(data_hash, 1);

struct h_struct {
    int data;
    struct hlist_node node;
};

static int hash_func(int data)
{
    return data % 2;
}
```

我们的哈希表只有两个桶，以便能够轻松地发生碰撞，因此哈希函数的实现非常简单；它只能返回值`0`或`1`。

1.  然后，在模块的`init()`函数中，我们定义我们的节点：

```
static int __init hashtable_init(void)
{
    struct h_struct e1 = {
        .data = 5
    };
    struct h_struct e2 = {
        .data = 2
    };
    struct h_struct e3 = {
        .data = 7
    };
```

1.  然后，我们进行第一次插入，然后打印数据：

```
    pr_info("add e1...\n");
    add_node(&e1);
    print_nodes();
```

1.  接下来，我们继续节点插入：

```
    pr_info("add e2, e3...\n");
    add_node(&e2);
    add_node(&e3);
    print_nodes();
```

1.  最后，我们尝试进行节点删除：

```
    pr_info("del data=5\n");
    del_node(5);
    print_nodes();

    return -EINVAL;
}
```

1.  作为最后一步，我们可以看一下节点的插入和删除函数：

```
static void add_node(struct h_struct *new)
{
    int key = hash_func(new->data);

    hash_add(data_hash, &new->node, key);
}

static void del_node(int data)
{
    int key = hash_func(data);
    struct h_struct *entry; 

    hash_for_each_possible(data_hash, entry, node, key) {
        if (entry->data == data) {
            hash_del(&entry->node);
            return;
        }
    }
}
```

这两个函数需要密钥生成，以确保将节点添加到正确的存储桶中或从中移除。

1.  可以通过使用`hash_for_each()`宏来进行哈希表的打印，如下所示：

```
static void print_nodes(void)
{
    int key;
    struct h_struct *entry;

    hash_for_each(data_hash, key, entry, node)
        pr_info("data=%d\n", entry->data);
}
```

# 还有更多...

同样，要测试代码，只需编译然后将内核模块插入 ESPRESSObin。

模块插入后，在内核消息中，我们应该看到第一行输出：

```
# insmod ./hashtable.ko 
hashtable:hashtable_init: add e1...
hashtable:print_nodes: data=5
```

您可以安全地忽略前面讨论过的以下错误消息：

`insmod: ERROR: could not insert module hashtable.ko: Invalid parameters`

在*步骤 1*和*步骤 2*中，我们已经定义了哈希表的节点，而在*步骤 3*中，我们已经对表进行了第一次插入，插入后的代码如上所示。

然后，在*步骤 4*中进行了第二次插入，我们添加了两个数据字段分别设置为`7`和`2`的节点：

```
hashtable:hashtable_init: add e2, e3...
hashtable:print_nodes: data=7
hashtable:print_nodes: data=2
hashtable:print_nodes: data=5
```

最后，在*步骤 5*中，我们移除了`data`字段设置为 5 的节点：

```
hashtable:hashtable_init: del data=5
hashtable:print_nodes: data=7
hashtable:print_nodes: data=2
```

请注意，在*步骤 6*中，我们展示了哈希表中节点插入的可能实现。在*步骤 7*中，我们有打印函数。

# 另请参阅

+   有关内核哈希表的更多信息，一个很好的起点（即使有点过时）是[`lwn.net/Articles/510202/`](https://lwn.net/Articles/510202/)。

# 获取 I/O 内存的访问

在这个示例中，我们将看到如何访问 CPU 的内部外围设备或连接到 CPU 的任何其他内存映射设备。

# 准备工作

这次，我们将使用内核源代码中已经存在的代码片段来展示一个示例，因此现在没有什么需要编译，但我们可以直接转到 ESPRESSObin 的内核源代码的根目录。

# 如何做...

1.  关于如何进行内存重映射的一个很好而且非常简单的例子在`linux/drivers/reset/reset-sunxi.c`文件的`sunxi_reset_init()`函数中报告如下：

```
static int sunxi_reset_init(struct device_node *np)
{
    struct reset_simple_data *data;
    struct resource res;
    resource_size_t size;
    int ret;

    data = kzalloc(sizeof(*data), GFP_KERNEL);
    if (!data)
        return -ENOMEM;

    ret = of_address_to_resource(np, 0, &res);
    if (ret)
        goto err_alloc;
```

通过使用`of_address_to_resource()`函数，我们询问设备树我们设备的内存映射，并将结果存储在`res`结构中。

1.  然后，我们使用`resource_size()`函数请求内存映射大小，然后调用`request_mem_region()`函数，以便向内核请求独占访问`res.start`和`res.start+size-1`之间的内存地址：

```
        size = resource_size(&res);
        if (!request_mem_region(res.start, size, np->name)) {
                ret = -EBUSY;
                goto err_alloc;
        }
```

如果没有人已经发出了相同的请求，该区域将被标记为我们使用的，并且标签名称存储在`np->name`中。

现在，名称和内存区域已经为我们保留，并且所有这些信息都可以从`/proc/iomem`文件中检索，如下一节所示。

1.  在进行了所有前期操作之后，我们最终可以调用`ioremap()`函数来实际进行重映射：

```
    data->membase = ioremap(res.start, size);
    if (!data->membase) {
        ret = -ENOMEM;
        goto err_alloc;
    }
```

在`data->membase`中存储了我们可以使用的虚拟地址，以便访问我们设备的寄存器。

`ioremap()`的原型及其对应的`iounmap()`在头文件`linux/include/asm-generic/io.h`中定义如下，当我们使用完这个映射时必须使用它：

```
void __iomem *ioremap(phys_addr_t phys_addr, size_t size);

void iounmap(void __iomem *addr);
```

请注意，在`linux/include/asm-generic/io.h`中，仅报告了没有 MMU 的系统的实现，因为每个平台都在`linux/arch`目录下有自己的实现。

# 工作原理...

要了解如何使用`ioremap()`，我们可以比较前面的代码和我们 ESPRESSObin 中的**通用异步收发器**（**UART**）驱动程序在`linux/drivers/tty/serial/mvebu-uart.c`文件中的示例。

```
...
    port->membase = devm_ioremap_resource(&pdev->dev, reg);
    if (IS_ERR(port->membase))
        return -PTR_ERR(port->membase);
...
    /* UART Soft Reset*/
    writel(CTRL_SOFT_RST, port->membase + UART_CTRL(port));
    udelay(1);
    writel(0, port->membase + UART_CTRL(port));
...
```

上述代码是`mvebu_uart_probe()`函数的一部分，该函数在某个时候调用`devm_ioremap_resource()`函数，该函数执行与*步骤 1*、*步骤 2*和*步骤 3*中呈现的函数的组合执行类似的步骤，即`of_address_to_resource()`、`request_mem_region()`和`ioremap()`函数同时进行：它从设备树中获取信息并进行内存重映射，仅保留这些寄存器供其独占使用。

这个注册（在*步骤 2*中之前完成）可以通过 procfs 文件`/proc/iomem`进行检查，我们可以看到内存区域`d0012000-d00121ff`分配给`serial@12000`：

```
root@espressobin:~# cat /proc/iomem 
00000000-7fffffff : System RAM
00080000-00faffff : Kernel code
010f0000-012a9fff : Kernel data
d0010600-d0010fff : spi@10600
d0012000-d00121ff : serial@12000
d0013000-d00130ff : nb-periph-clk@13000
d0013200-d00132ff : tbg@13200
d0013c00-d0013c1f : pinctrl@13800
d0018000-d00180ff : sb-periph-clk@18000
d0018c00-d0018c1f : pinctrl@18800
d001e808-d001e80b : sdhci@d0000
d0030000-d0033fff : ethernet@30000
d0058000-d005bfff : usb@58000
d005e000-d005ffff : usb@5e000
d0070000-d008ffff : pcie@d0070000
d00d0000-d00d02ff : sdhci@d0000
d00e0000-d00e1fff : sata@e0000
e8000000-e8ffffff : pcie@d0070000
```

正如本书中已经多次声明的那样，当我们在内核中时，没有人真的能阻止我们做某事；因此，当我谈到对内存区域的*独占使用*时，读者应该想象这是真实的，如果所有程序员自愿在之前的 I/O 内存区域的访问请求（比如之前发出的请求）失败后，他们都不会在该区域上发出内存访问。

# 另请参阅

+   有关内存映射的更多信息，一个很好的起点是[`linux-kernel-labs.github.io/master/labs/memory_mapping.html`](https://linux-kernel-labs.github.io/master/labs/memory_mapping.html)。

# 在内核中花费时间

在这个示例中，我们将看看如何通过使用繁忙循环或可能涉及挂起的更复杂的函数来延迟将来的执行。

# 做好准备

即使在这个示例中，我们也可以使用内核模块来查看测试代码的工作原理。

# 如何做...

在`time.c`文件中，我们可以找到一个简单的示例，说明了前面的函数是如何工作的：

1.  作为第一步，我们声明一个实用函数来获取代码行的执行时间（以纳秒为单位）：

```
#define print_time(str, code)     \
    do {                          \
        u64 t0, t1;               \
        t0 = ktime_get_real_ns(); \
        code;                     \
        t1 = ktime_get_real_ns(); \
        pr_info(str " -> %lluns\n", t1 - t0); \
    } while (0)
```

这是一个简单的技巧，定义一个宏，通过使用`ktime_get_real_ns（）`函数执行一行代码，同时返回其执行时间，该函数返回当前系统时间（以纳秒为单位）。

有关`ktime_get_real_ns（）`和相关函数的更多信息，您可以查看[`www.kernel.org/doc/html/latest/core-api/timekeeping.html`](https://www.kernel.org/doc/html/latest/core-api/timekeeping.html)。

1.  现在，对于模块的`init（）`函数，我们可以使用我们的宏，然后调用所有前面的延迟函数如下：

```
static int __init time_init(void)
{
    pr_info("*delay() functions:\n");
    print_time("10ns", ndelay(10));
    print_time("10000ns", udelay(10));
    print_time("10000000ns", mdelay(10));

    pr_info("*sleep() functions:\n");
    print_time("10000ns", usleep_range(10, 10));
    print_time("10000000ns", msleep(10));
    print_time("10000000ns", msleep_interruptible(10));
    print_time("10000000000ns", ssleep(10));

    return -EINVAL;
}
```

# 还有更多...

我们可以通过编译代码并将其插入到 ESPRESSObin 内核中来测试我们的代码：

```
# insmod time.ko 
```

以下内核消息应通过在*步骤 1*中定义的宏来打印出来。这个宏只是通过使用`ktime_get_real_ns（）`函数来获取传递给`code`参数的延迟函数的执行时间，这对于获取当前内核时间（以纳秒为单位）非常有用：

```
time:time_init: *delay() functions:
time:time_init: 10ns -> 480ns
time:time_init: 10000us -> 10560ns
time:time_init: 10000000ms -> 10387920ns
time:time_init: *sleep() functions:
time:time_init: 10000us -> 580720ns
time:time_init: 10000000ms -> 17979680ns
time:time_init: 10000000ms -> 17739280ns
time:time_init: 10000000000ms -> 10073738800ns
```

您可以安全地忽略以下错误消息，如前所述：

`insmod: ERROR: could not insert module time.ko: Invalid parameters`

请注意，由于`ssleep（10）`函数的最后调用，提示将在返回之前等待 10 秒，这是不可中断的；因此，即使我们按下*Ctrl* + *C*，我们也无法停止执行。

检查前面的输出（来自*步骤 2*），我们注意到`ndelay（）`对于少量时间来说并不像预期的那样可靠，而`udelay（）`和`mdelay（）`效果更好。至于`*sleep（）`函数，我们必须说它们受到机器负载的严重影响，因为它们可以休眠。

# 另请参阅

+   有关延迟函数的更多信息，一个很好的起点是内核文档中的`linux/Documentation/timers/timers-howto.txt`文件。
