# 第十一章：附加信息：杂项内核内部

以下是有关动态内存分配和 I/O 内存访问方法的一些一般信息。

在谈论动态内存分配时，我们应该记住我们是在内核中使用 C 语言进行编程，因此非常重要的一点是要记住每个分配的内存块在不再使用时必须被释放。这非常重要，因为在用户空间，当一个进程结束执行时，内核（实际上知道进程拥有的所有内存块）可以轻松地收回所有进程分配的内存；但对于内核来说，情况并非如此。实际上，要求内存块的驱动程序（或其他内核实体）必须确保释放它，否则没有人会要求它回来，内存块将丢失，直到机器重新启动。

关于对 I/O 内存的访问，这是由底层外围寄存器下的内存单元组成的区域，我们必须考虑到我们不能使用它们的物理内存地址来访问它们；相反，我们将不得不使用相应的虚拟地址。事实上，Linux 是一个使用**内存管理单元**（MMU）来虚拟化和保护内存访问的操作系统，因此我们必须将每个外围设备的物理内存区域重新映射到其相应的虚拟内存区域，以便能够从中读取和写入。

这个操作可以很容易地通过使用代码段中介绍的内核函数来完成，但是非常重要的一点是必须在尝试进行任何 I/O 内存访问之前完成，否则将触发段错误。这可能会终止用户空间中的进程，或者可能因为设备驱动程序中的错误而终止内核本身。

# 动态内存分配

分配内存的最直接方式是使用`kmalloc()`函数，并且为了安全起见，最好使用清除分配的内存为零的例程，例如`kzalloc()`函数。另一方面，如果我们需要为数组分配内存，有专门的函数`kmalloc_array()`和`kcalloc()`。

以下是包含内存分配内核函数（以及相关的内核内存释放函数）的一些片段，如内核源文件`linux/include/linux/slab.h`中所述。

```
/**
 * kmalloc - allocate memory
 * @size: how many bytes of memory are required.
 * @flags: the type of memory to allocate.
...
*/
static __always_inline void *kmalloc(size_t size, gfp_t flags);

/**
 * kzalloc - allocate memory. The memory is set to zero.
 * @size: how many bytes of memory are required.
 * @flags: the type of memory to allocate (see kmalloc).
 */
static inline void *kzalloc(size_t size, gfp_t flags)
{
    return kmalloc(size, flags | __GFP_ZERO);
}

/**
 * kmalloc_array - allocate memory for an array.
 * @n: number of elements.
 * @size: element size.
 * @flags: the type of memory to allocate (see kmalloc).
 */
static inline void *kmalloc_array(size_t n, size_t size, gfp_t flags);

/**
 * kcalloc - allocate memory for an array. The memory is set to zero.
 * @n: number of elements.
 * @size: element size.
 * @flags: the type of memory to allocate (see kmalloc).
 */
static inline void *kcalloc(size_t n, size_t size, gfp_t flags)
{
    return kmalloc_array(n, size, flags | __GFP_ZERO);
}

void kfree(const void *);
```

所有前述函数都暴露了用户空间对应的`malloc()`和其他内存分配函数之间的两个主要区别：

1.  使用`kmalloc()`和其他类似函数分配的块的最大大小是有限的。实际限制取决于硬件和内核配置，但是最好的做法是对小于页面大小的对象使用`kmalloc()`和其他内核辅助函数。

定义`PAGE_SIZE`信息内核源文件`linux/include/asm-generic/page.h`中指定了构成页面大小的字节数；通常情况下，32 位系统为 4096 字节，64 位系统为 8192 字节。用户可以通过通常的内核配置机制来明确选择它。

1.  用于动态内存分配的内核函数，如`kmalloc()`和类似函数，需要额外的参数；分配标志用于指定`kmalloc()`的行为方式，如下面从内核源文件`linux/include/linux/slab.h`中报告的片段所述。

```
/**
 * kmalloc - allocate memory
 * @size: how many bytes of memory are required.
 * @flags: the type of memory to allocate.
 *
 * kmalloc is the normal method of allocating memory
 * for objects smaller than page size in the kernel.
 *
 * The @flags argument may be one of:
 *
 * %GFP_USER - Allocate memory on behalf of user. May sleep.
 *
 * %GFP_KERNEL - Allocate normal kernel ram. May sleep.
 *
 * %GFP_ATOMIC - Allocation will not sleep. May use emergency pools.
 * For example, use this inside interrupt handlers.
 *
 * %GFP_HIGHUSER - Allocate pages from high memory.
 *
 * %GFP_NOIO - Do not do any I/O at all while trying to get memory.
 *
 * %GFP_NOFS - Do not make any fs calls while trying to get memory.
 *
 * %GFP_NOWAIT - Allocation will not sleep.
...
```

正如我们所看到的，存在许多标志；然而，设备驱动程序开发人员主要感兴趣的是`GFP_KERNEL`和`GFP_ATOMIC`。

很明显，这两个标志之间的主要区别在于前者可以分配正常的内核 RAM 并且可能会休眠，而后者在不允许调用者休眠的情况下执行相同的操作。这两个函数之间的这个巨大区别告诉我们，当我们处于中断上下文或进程上下文时，我们必须使用哪个标志。

如[第五章](https://cdp.packtpub.com/linux_device_driver_development_cookbook/wp-admin/post.php?post=6&action=edit#post_28)所示，*管理中断和并发*，当我们处于中断上下文时，我们不能休眠（如上面的代码所述），在这种情况下，我们必须通过指定`GFP_ATOMIC`标志来调用`kmalloc()`和相关函数，而`GFP_KERNEL`标志可以在其他地方使用，需要注意的是它可能导致调用者休眠，然后 CPU 可能会让我们执行其他操作；因此，我们应该避免以下操作：

```
spin_lock(...);
ptr = kmalloc(..., GFP_KERNEL);
spin_unlock(...);
```

实际上，即使我们在进程上下文中执行，持有自旋锁的休眠`kmalloc()`被认为是邪恶的！因此，在这种情况下，我们无论如何都必须使用`GFP_ATOMIC`标志。此外，需要注意的是，对于相同原因，成功的`GFP_ATOMIC`分配请求的最大大小往往比`GFP_KERNEL`请求要小，这与物理连续内存分配有关，内核保留了有限的内存池可供原子分配使用。

关于上面的第一点，对于可分配内存块的有限大小，对于大型分配，我们可以考虑使用另一类函数：`vmalloc()`和`vzalloc()`，即使我们必须强调`vmalloc()`和相关函数分配的内存不是物理上连续的，不能用于**直接内存访问**（**DMA**）活动（而`kmalloc()`和相关函数，如前面所述，分配了虚拟和物理寻址空间中的连续内存区域）。

目前，本书未涉及为 DMA 活动分配内存；但是，您可以在内核源代码中的`linux/Documentation/DMA-API.txt`和`linux/Documentation/DMA-API-HOWTO.txt`文件中获取有关此问题的更多信息。

以下是`vmalloc()`函数的原型和在`linux/include/linux/vmalloc.h`头文件中报告的函数定义：

```
extern void *vmalloc(unsigned long size);
extern void *vzalloc(unsigned long size);
```

如果我们不确定分配的大小是否对于`kmalloc()`来说太大，我们可以使用`kvmalloc()`及其衍生函数。这个函数将尝试使用`kmalloc()`来分配内存，如果分配失败，它将退而使用`vmalloc()`。

请注意，`kvmalloc()`可能返回的内存不是物理上连续的。

还有关于`kvmalloc()`可以与哪些`GFP_*`标志一起使用的限制，可以在[`www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kvmalloc_node`](https://www.kernel.org/doc/html/latest/core-api/mm-api.html#c.kvmalloc_node)中的`kvmalloc_node()`文档中找到。

以下是`linux/include/linux/mm.h`头文件中报告的`kvmalloc()`、`kvzalloc()`、`kvmalloc_array()`、`kvcalloc()`和`kvfree()`的代码片段：

```
static inline void *kvmalloc(size_t size, gfp_t flags)
{
    return kvmalloc_node(size, flags, NUMA_NO_NODE);
}

static inline void *kvzalloc(size_t size, gfp_t flags)
{
    return kvmalloc(size, flags | __GFP_ZERO);
}

static inline void *kvmalloc_array(size_t n, size_t size, gfp_t flags)
{
    size_t bytes;

    if (unlikely(check_mul_overflow(n, size, &bytes)))
        return NULL;

    return kvmalloc(bytes, flags);
}

static inline void *kvcalloc(size_t n, size_t size, gfp_t flags)
{
    return kvmalloc_array(n, size, flags | __GFP_ZERO);
}

extern void kvfree(const void *addr);
```

# 内核双向链表

在使用 Linux 的**双向链表**接口时，我们应该始终记住这些列表函数不执行锁定，因此我们的设备驱动程序（或其他内核实体）可能尝试对同一列表执行并发操作。这就是为什么我们必须确保实现一个良好的锁定方案来保护我们的数据免受竞争条件的影响。

要使用列表机制，我们的驱动程序必须包括头文件`linux/include/linux/list.h`；这个文件包括头文件`linux/include/linux/types.h`，在这里定义了`struct list_head`类型的简单结构如下：

```
struct list_head {
    struct list_head *next, *prev;
};
```

正如我们所看到的，这个结构包含两个指针（`prev`和`next`）指向`list_head`结构；这两个指针实现了双向链表的功能。然而，有趣的是`struct list_head`没有专用的数据字段，就像在经典的列表实现中那样。事实上，在 Linux 内核列表实现中，数据字段并没有嵌入在列表元素本身中；相反，列表结构是被认为被封装在相关数据结构中。这可能会让人困惑，但实际上并不是；事实上，要在我们的代码中使用 Linux 列表功能，我们只需要在使用列表的结构中嵌入一个`struct list_head`。

我们可以在设备驱动程序中声明对象结构的简单示例如下：

```
struct l_struct {
    int data;
    ... 
    /* other driver specific fields */
    ...
    struct list_head list;
};
```

通过这样做，我们创建了一个带有自定义数据的双向链表。然后，要有效地创建我们的列表，我们只需要声明并初始化列表头，使用以下代码：

```
struct list_head data_list;
INIT_LIST_HEAD(&data_list);
```

与其他内核结构一样，我们有编译时对应的宏`LIST_HEAD()`，它可以用于在非动态列表分配的情况下执行相同的操作。在我们的示例中，我们可以这样做：`LIST_HEAD(data_list)`；

一旦列表头部被声明并正确初始化，我们可以使用`linux/include/linux/list.h`文件中的几个函数来添加、删除或执行其他列表条目操作。

如果我们查看头文件，我们可以看到以下函数用于向列表中添加或删除元素：

```
/**
 * list_add - add a new entry
 * @new: new entry to be added
 * @head: list head to add it after
 *
 * Insert a new entry after the specified head.
 * This is good for implementing stacks.
 */
static inline void list_add(struct list_head *new, struct list_head *head);

 * list_del - deletes entry from list.
 * @entry: the element to delete from the list.
 * Note: list_empty() on entry does not return true after this, the entry is
 * in an undefined state.
 */
static inline void list_del(struct list_head *entry);
```

用于用新条目替换旧条目的以下函数也是可见的：

```
/**
 * list_replace - replace old entry by new one
 * @old : the element to be replaced
 * @new : the new element to insert
 *
 * If @old was empty, it will be overwritten.
 */
static inline void list_replace(struct list_head *old,
                                struct list_head *new);
...
```

这只是所有可用函数的一个子集。鼓励您查看`linux/include/linux/list.h`文件以发现更多。

除了前面的函数之外，用于向列表中添加或删除条目的宏更有趣。例如，如果我们希望以有序的方式添加新条目，我们可以这样做：

```
void add_ordered_entry(struct l_struct *new)
{
    struct list_head *ptr;
    struct my_struct *entry;

    list_for_each(ptr, &data_list) {
        entry = list_entry(ptr, struct l_struct, list);
        if (entry->data < new->data) {
            list_add_tail(&new->list, ptr);
            return;
        }
    }
    list_add_tail(&new->list, &data_list)
}
```

通过使用`list_for_each()`宏，我们可以迭代列表，并通过使用`list_entry()`，我们可以获得指向我们封闭数据的指针。请注意，我们必须将指向当前元素`ptr`、我们的结构类型以及我们结构中的列表条目的名称（在前面的示例中为`list`）传递给`list_entry()`。

最后，我们可以使用`list_add_tail()`函数将我们的新元素添加到正确的位置。

请注意，`list_entry()`只是使用`container_of()`宏来执行其工作。该宏在第五章*管理中断和并发性*的*container_of()宏*部分中有解释。

如果我们再次查看`linux/include/linux/list.h`文件，我们可以看到更多的函数，我们可以使用这些函数来从列表中获取条目或以不同的方式迭代所有列表元素：

```
/**
 * list_entry - get the struct for this entry
 * @ptr: the &struct list_head pointer.
 * @type: the type of the struct this is embedded in.
 * @member: the name of the list_head within the struct.
 */
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)

/**
 * list_first_entry - get the first element from a list
 * @ptr: the list head to take the element from.
 * @type: the type of the struct this is embedded in.
 * @member: the name of the list_head within the struct.
 *
 * Note, that list is expected to be not empty.
 */
#define list_first_entry(ptr, type, member) \
        list_entry((ptr)->next, type, member)

/**
 * list_last_entry - get the last element from a list
 * @ptr: the list head to take the element from.
 * @type: the type of the struct this is embedded in.
 * @member: the name of the list_head within the struct.
 *
 * Note, that list is expected to be not empty.
 */
#define list_last_entry(ptr, type, member) \
        list_entry((ptr)->prev, type, member)
...
```

一些宏也可用于迭代每个列表的元素：

```
/**
 * list_for_each - iterate over a list
 * @pos: the &struct list_head to use as a loop cursor.
 * @head: the head for your list.
 */
#define list_for_each(pos, head) \
        for (pos = (head)->next; pos != (head); pos = pos->next)

/**
 * list_for_each_prev - iterate over a list backwards
 * @pos: the &struct list_head to use as a loop cursor.
 * @head: the head for your list.
 */
#define list_for_each_prev(pos, head) \
        for (pos = (head)->prev; pos != (head); pos = pos->prev)
...
```

再次注意，这只是所有可用函数的一个子集，因此鼓励您查看`linux/include/linux/list.h`文件以发现更多。

# 内核哈希表

如前所述，对于链表，当使用 Linux 的**哈希表**接口时，我们应该始终记住这些哈希函数不执行锁定，因此我们的设备驱动程序（或其他内核实体）可能尝试对同一哈希表执行并发操作。这就是为什么我们必须确保还实现了一个良好的锁定方案来保护我们的数据免受竞争条件的影响。

与内核列表一样，我们可以声明然后初始化一个具有 2 的幂位大小的哈希表，使用以下代码：

```
DECLARE_HASHTABLE(data_hash, bits)
hash_init(data_hash);
```

与列表一样，我们有编译时对应的宏`DEFINE_HASHTABLE()`，它可以用于在非动态哈希表分配的情况下执行相同的操作。在我们的示例中，我们可以使用`DEFINE_HASHTABLE(data_hash, bits)`；

这将创建并初始化一个名为`data_hash`的表，其大小基于 2 的幂。正如刚才所说，该表是使用包含内核`struct hlist_head`类型的桶来实现的；这是因为内核哈希表是使用哈希链实现的，而哈希冲突只是添加到列表的头部。为了更好地看到这一点，我们可以参考`DECLARE_HASHTABLE()`宏的定义：

```
#define DECLARE_HASHTABLE(name, bits) \
    struct hlist_head name[1 << (bits)]
```

完成后，可以构建一个包含`struct hlist_node`指针的结构来保存要插入的数据，就像我们之前为列表所做的那样：

```
struct h_struct {
    int key;
    int data;
    ... 
    /* other driver specific fields */
    ...
    struct hlist_node node;
};
```

`struct hlist_node`及其头`struct hlist_head`在`linux/include/linux/types.h`头文件中定义如下：

```
struct hlist_head {
    struct hlist_node *first;
};

struct hlist_node {
    struct hlist_node *next, **pprev;
};
```

然后可以使用`hash_add()`函数将新节点添加到哈希表中，如下所示，其中`&entry.node`是数据结构中`struct hlist_node`的指针，`key`是哈希键：

```
hash_add(data_hash, &entry.node, key);
```

密钥可以是任何东西；但通常是通过使用特殊的哈希函数应用于要存储的数据来计算的。例如，有一个 256 个桶的哈希表，密钥可以用以下`hash_func()`计算：

```
u8 hash_func(u8 *buf, size_t len)
{
    u8 key = 0;

    for (i = 0; i < len; i++)
        key += data[i];

    return key;
}
```

相反的操作，即删除，可以通过使用`hash_del()`函数来完成，如下所示：

```
hash_del(&entry.node);
```

但是，与列表一样，最有趣的宏是用于迭代表的宏。存在两种机制；一种是遍历整个哈希表，返回每个桶中的条目：

```
hash_for_each(name, bkt, node, obj, member)
```

另一个仅返回与密钥的哈希桶对应的条目：

```
hash_for_each_possible(name, obj, member, key)
```

通过使用最后一个宏，从哈希表中删除节点的过程如下：

```
void del_node(int data)
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

请注意，此实现只删除第一个匹配的条目。

通过使用`hash_for_each_possible()`，我们可以迭代与密钥相关的桶中的列表。

以下是`linux/include/linux/hashtable.h`文件中报告的`hash_add()`、`hash_del()`和`hash_for_each_possible()`的定义：

```
/**
 * hash_add - add an object to a hashtable
 * @hashtable: hashtable to add to
 * @node: the &struct hlist_node of the object to be added
 * @key: the key of the object to be added
 */
#define hash_add(hashtable, node, key) \
        hlist_add_head(node, &hashtable[hash_min(key, HASH_BITS(hashtable))])

/**
 * hash_del - remove an object from a hashtable
 * @node: &struct hlist_node of the object to remove
 */
static inline void hash_del(struct hlist_node *node);

/**
 * hash_for_each_possible - iterate over all possible objects hashing to the
 * same bucket
 * @name: hashtable to iterate
 * @obj: the type * to use as a loop cursor for each entry
 * @member: the name of the hlist_node within the struct
 * @key: the key of the objects to iterate over
 */
#define hash_for_each_possible(name, obj, member, key) \
        hlist_for_each_entry(obj, &name[hash_min(key, HASH_BITS(name))], member)
```

这些只是管理哈希表的所有可用函数的子集。鼓励您查看`linux/include/linux/hashtable.h`文件以了解更多。

# 访问 I/O 内存

为了能够有效地与外围设备通信，我们需要一种方法来读写其寄存器，为此我们有两种方法：通过**I/O 端口**或通过**I/O 内存**。前一种机制在本书中没有涵盖，因为它在现代平台中（除了 x86 和 x86_64 之外）并不经常使用，而后者只是使用正常的内存区域来映射每个外围寄存器，这是现代 CPU 中常用的一种方法。事实上，I/O 内存映射在**片上系统**（**SoC**）系统中非常常见，其中 CPU 可以通过读写到众所周知的物理地址来与其内部外围设备通信；在这种情况下，每个外围设备都有其自己的保留地址，并且每个外围设备都连接到一个寄存器。

要看我所说的一个简单示例，您可以从[`ww1.microchip.com/downloads/en/DeviceDoc/Atmel-11121-32-bit-Cortex-A5-Microcontroller-SAMA5D3_Datasheet_B.pdf`](http://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-11121-32-bit-Cortex-A5-Microcontroller-SAMA5D3_Datasheet_B.pdf)获取 SAMA5D3 CPU 的数据表，查看第 30 页，其中报告了整个 CPU 的完整内存映射。

然后，这个 I/O 内存映射被报告在与平台相关的设备树文件中。举个例子，如果我们看一下内核源文件中`linux/arch/arm64/boot/dts/marvell/armada-37xx.dtsi`文件中我们 ESPRESSObin 的 CPU 的 UART 控制器的定义，我们可以看到以下设置：

```
soc {
    compatible = "simple-bus";
    #address-cells = <2>;
    #size-cells = <2>;
    ranges;

    internal-regs@d0000000 {
        #address-cells = <1>;
        #size-cells = <1>;
        compatible = "simple-bus";
        /* 32M internal register @ 0xd000_0000 */
        ranges = <0x0 0x0 0xd0000000 0x2000000>;

...

        uart0: serial@12000 {
            compatible = "marvell,armada-3700-uart";
            reg = <0x12000 0x200>;
            clocks = <&xtalclk>;
            interrupts =
            <GIC_SPI 11 IRQ_TYPE_LEVEL_HIGH>,
            <GIC_SPI 12 IRQ_TYPE_LEVEL_HIGH>,
            <GIC_SPI 13 IRQ_TYPE_LEVEL_HIGH>;
            interrupt-names = "uart-sum", "uart-tx", "uart-rx";
            status = "disabled";
        };
```

如第四章中所解释的，*使用设备树*，我们可以推断 UART0 控制器被映射到物理地址`0xd0012000`。这也被我们在启动时可以看到的以下内核消息所证实：

```
d0012000.serial: ttyMV0 at MMIO 0xd0012000 (irq = 0, base_baud = 
1562500) is a mvebu-uart
```

好的，现在我们必须记住`0xd0012000`是 UART 控制器的**物理地址**，但我们的 CPU 知道**虚拟地址**，因为它使用其 MMU 来访问 RAM！那么，我们如何在物理地址`0xd0012000`和其虚拟对应地址之间进行转换呢？答案是：通过内存重新映射。在每次读取或写入 UART 控制器的寄存器之前，必须在内核中执行此操作，否则将引发段错误。

只是为了了解物理地址和虚拟地址之间的差异以及重新映射操作的行为，我们可以看一下名为`devmem2`的实用程序，该实用程序可以通过 ESPRESSObin 上的`wget`程序从[`free-electrons.com/pub/mirror/devmem2.c`](http://free-electrons.com/pub/mirror/devmem2.c)下载：

```
# wget http://free-electrons.com/pub/mirror/devmem2.c
```

如果我们看一下代码，我们会看到以下操作：

```
    if((fd = open("/dev/mem", O_RDWR | O_SYNC)) == -1) FATAL;
    printf("/dev/mem opened.\n"); 
    fflush(stdout);

    /* Map one page */
    map_base = mmap(0, MAP_SIZE,
                    PROT_READ | PROT_WRITE,
                    MAP_SHARED, fd, target & ~MAP_MASK);
    if(map_base == (void *) -1) FATAL;
    printf("Memory mapped at address %p.\n", map_base); 
    fflush(stdout);
```

因此，`devmem2`程序只是打开`/dev/mem`设备，然后调用`mmap()`系统调用。这将导致在内核源文件`linux/ drivers/char/mem.c`中执行`mmap_mem（）`方法，其中实现了`/dev/mem`字符设备：

```
static int mmap_mem(struct file *file, struct vm_area_struct *vma)
{
    size_t size = vma->vm_end - vma->vm_start;
    phys_addr_t offset = (phys_addr_t)vma->vm_pgoff << PAGE_SHIFT;

...

    /* Remap-pfn-range will mark the range VM_IO */
    if (remap_pfn_range(vma,
                        vma->vm_start, vma->vm_pgoff,
                        size,
                        vma->vm_page_prot)) {
        return -EAGAIN;
    }
    return 0;
}
```

有关这些内存重新映射操作以及`remap_pfn_range（）`函数和类似函数的使用的更多信息将在第七章“高级字符驱动程序操作”中更清楚。

好吧，`mmap_mem（）`方法对物理地址`0xd0012000`进行内存重新映射操作，将其映射为适合 CPU 访问 UART 控制器寄存器的虚拟地址。

如果我们尝试在 ESPRESSObin 上使用以下命令编译代码，我们将得到一个可执行文件，从用户空间访问 UART 控制器的寄存器：

```
# make CFLAGS="-Wall -O" devmem2 cc -Wall -O devmem2.c -o devmem2
```

您可以安全地忽略下面显示的可能的警告消息：

`devmem2.c:104:33: 警告：格式“％X”需要类型为“unsigned int”的参数，`

`但参数 2 的类型为'off_t {aka long int}' [-Wformat=]`

`printf("地址 0x%X（%p）处的值：0x%X\n"，target，virt_addr，read_result`

`);`

`devmem2.c:104:44: 警告：格式“％X”需要类型为“unsigned int”的参数，`

`但参数 4 的类型为'long unsigned int' [-Wformat=]`

`printf("地址 0x%X（%p）处的值：0x%X\n"，target，virt_addr，read_result`

`);`

`devmem2.c:123:22: 警告：格式“％X”需要类型为“unsigned int”的参数，`

`但参数 2 的类型为'long unsigned int' [-Wformat=]`

`printf("写入 0x%X；读回 0x%X\n"，writeval，read_result);`

`devmem2.c:123:37: 警告：格式“％X”需要类型为“unsigned int”的参数，`

`但参数 3 的类型为'long unsigned int' [-Wformat=]`

`printf("写入 0x%X；读回 0x%X\n"，writeval，read_result);`

然后，如果我们执行程序，我们应该得到以下输出：

```
# ./devmem2 0xd0012000 
/dev/mem opened.
Memory mapped at address 0xffffbd41d000.
Value at address 0xD0012000 (0xffffbd41d000): 0xD
```

正如我们所看到的，`devmem2`程序按预期打印了重新映射结果，并且实际读取是使用虚拟地址完成的，而 MMU 又将其转换为所需的物理地址`0xd0012000`。

好了，现在清楚了，访问外围寄存器需要进行内存重新映射，我们可以假设一旦我们有了一个虚拟地址物理映射到一个寄存器，我们可以简单地引用它来实际读取或写入数据。这是错误的！实际上，尽管硬件寄存器在内存中映射和通常的 RAM 内存之间有很强的相似性，但当我们访问 I/O 寄存器时，我们必须小心避免被 CPU 或编译器优化所欺骗，这些优化可能会修改预期的 I/O 行为。

I/O 寄存器和 RAM 之间的主要区别在于 I/O 操作具有副作用，而内存操作则没有；实际上，当我们向 RAM 中写入一个值时，我们希望它不会被其他人改变，但对于 I/O 内存来说，这并不是真的，因为我们的外设可能会改变寄存器中的一些数据，即使我们向其中写入了特定的值。这是一个非常重要的事实，因为为了获得良好的性能，RAM 内容可以被缓存，并且 CPU 指令流水线可以重新排序读/写指令；此外，编译器可以自主决定将数据值放入 CPU 寄存器而不将其写入内存，即使最终将其存储到内存中，写入和读取操作都可以在缓存内存上进行，而不必到达物理 RAM。即使最终将其存储到内存中，这两种优化在 I/O 内存上是不可接受的。实际上，这些优化在应用于常规内存时是透明且良性的，但在 I/O 操作中可能是致命的，因为外设有明确定义的编程方式，对其寄存器的读写操作不能重新排序或缓存，否则会导致故障。

这些是我们不能简单地引用虚拟内存地址来从内存映射的外设中读取和写入数据的主要原因。因此，驱动程序必须确保在访问寄存器时不执行缓存操作，也不进行读取或写入重排序；解决方案是使用实际执行读写操作的特殊函数。在`linux/include/asm-generic/io.h`头文件中，我们可以找到这些函数，如以下示例所示：

```
static inline void writeb(u8 value, volatile void __iomem *addr)
{
    __io_bw();
    __raw_writeb(value, addr);
    __io_aw();
}

static inline void writew(u16 value, volatile void __iomem *addr)
{
    __io_bw();
    __raw_writew(cpu_to_le16(value), addr);
    __io_aw();
}

static inline void writel(u32 value, volatile void __iomem *addr)
{
    __io_bw();
    __raw_writel(__cpu_to_le32(value), addr);
    __io_aw();
}

#ifdef CONFIG_64BIT
static inline void writeq(u64 value, volatile void __iomem *addr)
{
    __io_bw();
    __raw_writeq(__cpu_to_le64(value), addr);
    __io_aw();
}
#endif /* CONFIG_64BIT */
```

前述函数仅用于写入数据；您可以查看头文件以查看读取函数的定义，例如`readb()`、`readw()`、`readl()`和`readq()`。

每个函数都定义为与要操作的寄存器的大小相对应的明确定义的数据类型一起使用；此外，它们每个都使用内存屏障来指示 CPU 按照明确定义的顺序执行读写操作。

我不打算在本书中解释内存屏障是什么；如果您感兴趣，您可以在`linux/Documentation/memory-barriers.txt`文件中的内核文档目录中阅读更多相关内容。

作为前述功能的一个简单示例，我们可以看一下 Linux 源文件中`linux/drivers/watchdog/sunxi_wdt.c`文件中的`sunxi_wdt_start()`函数：

```
static int sunxi_wdt_start(struct watchdog_device *wdt_dev)
{
...
    void __iomem *wdt_base = sunxi_wdt->wdt_base;
    const struct sunxi_wdt_reg *regs = sunxi_wdt->wdt_regs;

...

    /* Set system reset function */
    reg = readl(wdt_base + regs->wdt_cfg);
    reg &= ~(regs->wdt_reset_mask);
    reg |= regs->wdt_reset_val;
    writel(reg, wdt_base + regs->wdt_cfg);

    /* Enable watchdog */
    reg = readl(wdt_base + regs->wdt_mode);
    reg |= WDT_MODE_EN;
    writel(reg, wdt_base + regs->wdt_mode);

    return 0;
}
```

一旦寄存器的基地址`wdt_base`和寄存器的映射`regs`已经获得，我们可以简单地通过使用`readl()`和`writel()`来执行我们的读写操作，如前面的部分所示，并且我们可以放心地确保它们将被正确执行。

# 在内核中花费时间

在第五章中，*管理中断和并发*，我们看到了如何延迟在以后的时间执行操作；然而，可能会发生这样的情况，我们仍然需要在外设上的两个操作之间等待一段时间，如下所示：

```
writeb(0x12, ctrl_reg);
wait_us(100);
writeb(0x00, ctrl_reg);
```

也就是说，如果我们需要向寄存器中写入一个值，然后等待 100 微秒，然后再写入另一个值，这些操作可以通过简单地使用`linux/include/linux/delay.h`头文件（和其他文件）中定义的函数来完成，而不是使用之前介绍的技术（内核定时器和工作队列等）：

```
void ndelay(unsigned long nsecs);
void udelay(unsigned long usecs);
void mdelay(unsigned long msecs);

void usleep_range(unsigned long min, unsigned long max);
void msleep(unsigned int msecs);
unsigned long msleep_interruptible(unsigned int msecs);
void ssleep(unsigned int seconds);
```

所有这些函数都是用于延迟一定量的时间，以纳秒、微秒或毫秒（或仅以秒为单位，如`ssleep()`）表示。

第一组函数（即`*delay()`函数）可以在中断或进程上下文中的任何地方使用，而第二组函数必须仅在进程上下文中使用，因为它们可能会隐式进入睡眠状态。

此外，我们看到，例如，`usleep_range()`函数采用最小和最大睡眠时间，以通过允许高分辨率定时器利用已经安排的中断来减少功耗，而不是仅为此睡眠安排新的中断。以下是`linux/kernel/time/timer.c`文件中的函数描述：

```
/**
 * usleep_range - Sleep for an approximate time
 * @min: Minimum time in usecs to sleep
 * @max: Maximum time in usecs to sleep
 *
 * In non-atomic context where the exact wakeup time is flexible, use
 * usleep_range() instead of udelay(). The sleep improves responsiveness
 * by avoiding the CPU-hogging busy-wait of udelay(), and the range reduces
 * power usage by allowing hrtimers to take advantage of an already-
 * scheduled interrupt instead of scheduling a new one just for this sleep.
 */
void __sched usleep_range(unsigned long min, unsigned long max);
```

此外，在同一文件中，我们看到`msleep_interruptible()`是`msleep()`的变体，可以被信号中断（在*等待事件*配方中，在第五章中，*管理中断和并发性*，我们谈到了这种可能性），返回值只是由于中断而未睡眠的时间（以毫秒为单位）：

```
/**
 * msleep_interruptible - sleep waiting for signals
 * @msecs: Time in milliseconds to sleep for
 */
unsigned long msleep_interruptible(unsigned int msecs);
```

最后，我们还应该注意以下内容：

+   `*delay()`函数使用时钟速度的 jiffy 估计（`loops_per_jiffy`值），并将忙等待足够的循环周期以实现所需的延迟。

+   `*delay()`函数可能会在计算出的`loops_per_jiffy`太低（由于执行定时器中断所需的时间）或者缓存行为影响执行循环函数所需的时间，或者由于 CPU 时钟速率的变化而提前返回。

+   `udelay()`是通常首选的 API，`ndelay()`的级别精度实际上可能不存在于许多非 PC 设备上。

+   `mdelay()`是对`udelay()`的宏包装，以考虑将大参数传递给`udelay()`时可能发生的溢出。这就是为什么不建议使用`mdelay()`，代码应该重构以允许使用`msleep()`。
