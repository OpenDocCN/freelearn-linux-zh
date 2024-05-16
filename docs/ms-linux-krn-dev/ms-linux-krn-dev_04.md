# 第四章：内存管理和分配器

内存管理的效率广泛地决定了整个内核的效率。随意管理的内存系统可能严重影响其他子系统的性能，使内存成为内核的关键组成部分。这个子系统通过虚拟化物理内存和管理它们发起的所有动态分配请求来启动所有进程和内核服务。内存子系统还处理维持操作效率和优化资源的广泛操作。这些操作既是特定于架构的，也是独立的，这要求整体设计和实现是公正和可调整的。在本章中，我们将密切关注以下方面，以便努力理解这个庞大的子系统：

+   物理内存表示

+   节点和区域的概念

+   页分配器

+   伙伴系统

+   Kmalloc 分配

+   Slab 高速缓存

+   Vmalloc 分配

+   连续内存分配

# 初始化操作

在大多数架构中，在*复位*时，处理器以正常或物理地址模式（也称为 x86 中的**实模式**）初始化，并开始执行平台固件指令，这些指令位于**复位向量**处。这些固件指令（可以是单一二进制或多阶段二进制）被编程来执行各种操作，包括初始化内存控制器，校准物理 RAM，并将二进制内核映像加载到物理内存的特定区域，等等。

在实模式下，处理器不支持虚拟寻址，而 Linux 是为具有**保护模式**的系统设计和实现的，需要**虚拟寻址**来启用进程保护和隔离，这是内核提供的关键抽象（回顾第一章，*理解进程、地址空间和线程*）。这要求处理器在内核启动和子系统初始化之前切换到保护模式并打开虚拟地址支持。切换到保护模式需要初始化 MMU 芯片组，通过设置适当的核心数据结构，从而启用*分页*。这些操作是特定于架构的，并且在内核源代码树的*arch*分支中实现。在内核构建期间，这些源代码被编译并链接为保护模式内核映像的头文件；这个头文件被称为**内核引导程序**或**实模式内核**。

![](img/00019.jpeg)

以下是 x86 架构引导程序的`main()`例程；这个函数在实模式下执行，并负责在调用`go_to_protected_mode()`之前分配适当的资源，然后进入保护模式：

```
/* arch/x86/boot/main.c */
void main(void)
{
 /* First, copy the boot header into the "zeropage" */
 copy_boot_params();

 /* Initialize the early-boot console */
 console_init();
 if (cmdline_find_option_bool("debug"))
 puts("early console in setup coden");

 /* End of heap check */
 init_heap();

 /* Make sure we have all the proper CPU support */
 if (validate_cpu()) {
 puts("Unable to boot - please use a kernel appropriate "
 "for your CPU.n");
 die();
 }

 /* Tell the BIOS what CPU mode we intend to run in. */
 set_bios_mode();

 /* Detect memory layout */
 detect_memory();

 /* Set keyboard repeat rate (why?) and query the lock flags */
 keyboard_init();

 /* Query Intel SpeedStep (IST) information */
 query_ist();

 /* Query APM information */
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
 query_apm_bios();
#endif

 /* Query EDD information */
#if defined(CONFIG_EDD) || defined(CONFIG_EDD_MODULE)
 query_edd();
#endif

 /* Set the video mode */
 set_video();

 /* Do the last things and invoke protected mode */
 go_to_protected_mode();
}
```

实模式内核例程是为了设置 MMU 并处理转换到保护模式而调用的，这些例程是特定于架构的（我们不会在这里涉及这些例程）。不管所涉及的特定于架构的代码是什么，主要目标是通过打开分页来启用对虚拟寻址的支持。启用分页后，系统开始将物理内存（RAM）视为固定大小的块数组，称为页帧。页帧的大小通过适当地编程 MMU 的分页单元来配置；大多数 MMU 支持 4k、8k、16k、64k 直到 4MB 的选项来配置帧大小。然而，Linux 内核对大多数架构的默认构建配置选择 4k 作为其标准页帧大小。

# 页描述符

**页面帧**是内存的最小分配单元，内核需要利用它们来满足其所有的内存需求。一些页面帧将被用于将物理内存映射到用户模式进程的虚拟地址空间，一些用于内核代码和其数据结构，一些用于处理进程或内核服务提出的动态分配请求。为了有效地管理这些操作，内核需要区分当前*使用*的页面帧和空闲可用的页面帧。这个目的通过一个与架构无关的数据结构`struct page`来实现，该结构被定义为保存与页面帧相关的所有元数据，包括其当前状态。为每个找到的物理页面帧分配一个`struct page`的实例，并且内核必须始终在主内存中维护页面实例的列表。

**页面结构**是内核中使用最频繁的数据结构之一，并且在各种内核代码路径中被引用。该结构填充有各种元素，其相关性完全基于物理帧的状态。例如，页面结构的特定成员指定相应的物理页面是否映射到进程或一组进程的虚拟地址空间。当物理页面被保留用于动态分配时，这些字段被认为无效。为了确保内存中的页面实例只分配有关字段，联合体被广泛用于填充成员字段。这是一个明智的选择，因为它使得能够在不增加内存中的页面结构大小的情况下将更多的信息塞入页面结构中：

```
/*include/linux/mm-types.h */ 
/* The objects in struct page are organized in double word blocks in
 * order to allows us to use atomic double word operations on portions
 * of struct page. That is currently only used by slub but the arrangement
 * allows the use of atomic double word operations on the flags/mapping
 * and lru list pointers also.
 */
struct page {
        /* First double word block */
         unsigned long flags; /* Atomic flags, some possibly updated asynchronously */   union {
          struct address_space *mapping; 
          void *s_mem; /* slab first object */
          atomic_t compound_mapcount; /* first tail page */
          /* page_deferred_list().next -- second tail page */
   };
  ....
  ....

}
```

以下是页面结构的重要成员的简要描述。请注意，这里的许多细节假定您熟悉我们在本章的后续部分中讨论的内存子系统的其他方面，比如内存分配器、页表等等。我建议新读者跳过并在熟悉必要的先决条件后再回顾本节。

# 标志

这是一个`unsigned long`位字段，它保存描述物理页面状态的标志。标志常量是通过内核头文件`include/linux/page-flags.h`中的`enum`定义的。以下表列出了重要的标志常量：

| **标志** | **描述** |
| --- | --- |
| `PG_locked` | 用于指示页面是否被锁定；在对页面进行 I/O 操作时设置此位，并在完成时清除。 |
| `PG_error` | 用于指示错误页面。在页面发生 I/O 错误时设置。 |
| `PG_referenced` | 设置以指示页面缓存的页面回收。 |
| `PG_uptodate` | 设置以指示从磁盘读取操作后页面是否有效。 |
| `PG_dirty` | 当文件支持的页面被修改并且与磁盘镜像不同步时设置。 |
| `PG_lru` | 用于指示最近最少使用位被设置，有助于处理页面回收。 |
| `PG_active` | 用于指示页面是否在活动列表中。 |
| `PG_slab` | 用于指示页面由 slab 分配器管理。 |
| `PG_reserved` | 用于指示不可交换的保留页面。 |
| `PG_private` | 用于指示页面被文件系统用于保存其私有数据。 |
| `PG_writeback` | 在对文件支持的页面进行写回操作时设置 |
| `PG_head` | 用于指示复合页面的头页面。 |
| `PG_swapcache` | 用于指示页面是否在 swapcache 中。 |
| `PG_mappedtodisk` | 用于指示页面被映射到存储上的*块*。 |
| `PG_swapbacked` | 页面由交换支持。 |
| `PG_unevictable` | 用于指示页面在不可驱逐列表中；通常，此位用于 ramfs 拥有的页面和`SHM_LOCKed`共享内存页面。 |
| `PG_mlocked` | 用于指示页面上启用了 VMA 锁。 |

存在许多宏来`检查`，`设置`和`清除`单个页面位；这些操作被保证是`原子的`，并且在内核头文件`/include/linux/page-flags.h`中声明。它们被调用以从各种内核代码路径操纵页面标志：

```
/*Macros to create function definitions for page flags */
#define TESTPAGEFLAG(uname, lname, policy) \
static __always_inline int Page##uname(struct page *page) \
{ return test_bit(PG_##lname, &policy(page, 0)->flags); }

#define SETPAGEFLAG(uname, lname, policy) \
static __always_inline void SetPage##uname(struct page *page) \
{ set_bit(PG_##lname, &policy(page, 1)->flags); }

#define CLEARPAGEFLAG(uname, lname, policy) \
static __always_inline void ClearPage##uname(struct page *page) \
{ clear_bit(PG_##lname, &policy(page, 1)->flags); }

#define __SETPAGEFLAG(uname, lname, policy) \
static __always_inline void __SetPage##uname(struct page *page) \
{ __set_bit(PG_##lname, &policy(page, 1)->flags); }

#define __CLEARPAGEFLAG(uname, lname, policy) \
static __always_inline void __ClearPage##uname(struct page *page) \
{ __clear_bit(PG_##lname, &policy(page, 1)->flags); }

#define TESTSETFLAG(uname, lname, policy) \
static __always_inline int TestSetPage##uname(struct page *page) \
{ return test_and_set_bit(PG_##lname, &policy(page, 1)->flags); }

#define TESTCLEARFLAG(uname, lname, policy) \
static __always_inline int TestClearPage##uname(struct page *page) \
{ return test_and_clear_bit(PG_##lname, &policy(page, 1)->flags); }

*....
....* 
```

# 映射

页面描述符的另一个重要元素是类型为`struct address_space`的指针`*mapping`。然而，这是一个棘手的指针，可能是指向`struct address_space`的一个实例，也可能是指向`struct anon_vma`的一个实例。在我们深入了解如何实现这一点之前，让我们首先了解这些结构及它们所代表的资源的重要性。

文件系统利用空闲页面（来自页面缓存）来缓存最近访问的磁盘文件的数据。这种机制有助于最小化磁盘 I/O 操作：当缓存中的文件数据被修改时，适当的页面通过设置`PG_dirty`位被标记为脏；所有脏页面都会在策略性间隔时段通过调度磁盘 I/O 写入相应的磁盘块。`struct address_space`是一个表示为文件缓存而使用的页面集合的抽象。页面缓存的空闲页面也可以被**映射**到进程或进程组以进行动态分配，为这种分配映射的页面被称为**匿名**页面映射。`struct anon_vma`的一个实例表示使用匿名页面创建的内存块，这些页面被映射到进程或进程的虚拟地址空间（通过 VMA 实例）。

通过位操作实现指针动态初始化为指向这两种数据结构中的任意一种的地址是有技巧的。如果指针`*mapping`的低位清除，则表示页面映射到`inode`，指针指向`struct address_space`。如果低位设置，这表示匿名映射，这意味着指针指向`struct anon_vma`的一个实例。这是通过确保`address_space`实例的分配对齐到`sizeof(long)`来实现的，这使得指向`address_space`的指针的最低有效位被清除（即设置为 0）。

# 区域和节点

对于整个内存管理框架至关重要的主要数据结构是**区域**和***节点***。让我们熟悉一下这些数据结构背后的核心概念。

# 内存区域

为了有效管理内存分配，物理页面被组织成称为**区域**的组。每个*区域*中的页面用于特定需求，如 DMA、高内存和其他常规分配需求。内核头文件`mmzone.h`中的`enum`声明了*区域*常量：

```
/* include/linux/mmzone.h */
enum zone_type {
#ifdef CONFIG_ZONE_DMA
ZONE_DMA,
#endif
#ifdef CONFIG_ZONE_DMA32
 ZONE_DMA32,
#endif
#ifdef CONFIG_HIGHMEM
 ZONE_HIGHMEM,
#endif
 ZONE_MOVABLE,
#ifdef CONFIG_ZONE_DEVICE
 ZONE_DEVICE,
#endif
 __MAX_NR_ZONES
};
```

`ZONE_DMA`：

这个*区域*中的页面是为不能在所有可寻址内存上启动 DMA 的设备保留的。这个*区域*的大小是特定于架构的：

| 架构 | 限制 |
| --- | --- |
| parsic, ia64, sparc | <4G |
| s390 | <2G |
| ARM | 可变 |
| alpha | 无限制或<16MB |
| alpha, i386, x86-64 | <16MB |

`ZONE_DMA32`：这个*区域*用于支持可以在<4G 内存上执行 DMA 的 32 位设备。这个*区域*仅存在于 x86-64 平台上。

`ZONE_NORMAL`：所有可寻址内存被认为是正常的*区域*。只要 DMA 设备支持所有可寻址内存，就可以在这些页面上启动 DMA 操作。

`ZONE_HIGHMEM`：这个*区域*包含只能通过显式映射到内核地址空间中的内核访问的页面；换句话说，所有超出内核段的物理内存页面都属于这个*区域*。这个*区域*仅存在于 3:1 虚拟地址分割（3G 用于用户模式，1G 地址空间用于内核）的 32 位平台上；例如在 i386 上，允许内核访问超过 900MB 的内存将需要为内核需要访问的每个页面设置特殊映射（页表条目）。

`ZONE_MOVABLE`：内存碎片化是现代操作系统处理的挑战之一，Linux 也不例外。从内核启动的那一刻开始，直到运行时，页面被分配和释放用于一系列任务，导致具有物理连续页面的小内存区域。考虑到 Linux 对虚拟寻址的支持，碎片化可能不会成为各种进程顺利执行的障碍，因为物理上分散的内存总是可以通过页表映射到虚拟连续地址空间。然而，有一些场景，比如 DMA 分配和为内核数据结构设置缓存，对物理连续区域有严格的需求。

多年来，内核开发人员一直在演进各种抗碎片化技术来减轻**碎片化**。引入`ZONE_MOVABLE`就是其中之一。这里的核心思想是跟踪每个*区域*中的*可移动*页面，并将它们表示为这个伪*区域*，这有助于防止碎片化（我们将在下一节关于伙伴系统中更多讨论这个问题）。

这个*区域*的大小将在启动时通过内核参数`kernelcore`进行配置；请注意，分配的值指定了被视为*不可移动*的内存量，其余的是*可移动*的。一般规则是，内存管理器被配置为考虑从最高填充的*区域*迁移页面到`ZONE_MOVABLE`，对于 x86 32 位机器来说，这可能是`ZONE_HIGHMEM`，对于 x86_64 来说，可能是`ZONE_DMA32`。

`ZONE_DEVICE`：这个*区域*被划分出来支持热插拔内存，比如大容量的*持久内存数组*。**持久内存**在许多方面与 DRAM 非常相似；特别是，CPU 可以直接以字节级寻址它们。然而，特性如持久性、性能（写入速度较慢）和大小（通常以 TB 为单位）使它们与普通内存有所区别。为了让内核支持这样的具有 4KB 页面大小的内存，它需要枚举数十亿个页结构，这将消耗主内存的大部分或根本不适合。因此，内核开发人员选择将持久内存视为**设备**，而不是像**内存**一样；这意味着内核可以依靠适当的**驱动程序**来管理这样的内存。

```
void *devm_memremap_pages(struct device *dev, struct resource *res,
                        struct percpu_ref *ref, struct vmem_altmap *altmap); 
```

持久内存驱动程序的`devm_memremap_pages()`例程将持久内存区域映射到内核的地址空间，并在持久设备内存中设置相关的页结构。这些映射下的所有页面都被分组到`ZONE_DEVICE`下。为这样的页面设置一个独特的*区域*可以让内存管理器将它们与常规统一内存页面区分开来。

# 内存节点

Linux 内核长期以来一直实现了对多处理器机器架构的支持。内核实现了各种资源，比如每 CPU 数据缓存、互斥锁和原子操作宏，这些资源在各种 SMP 感知子系统中被使用，比如进程调度器和设备管理等。特别是，内存管理子系统的作用对于内核在这样的架构上运行至关重要，因为它需要将每个处理器所看到的内存虚拟化。多处理器机器架构基于每个处理器的感知和对系统内存的访问延迟，被广泛分类为两种类型。

**统一内存访问架构（UMA）**：这些是多处理器架构的机器，处理器通过互连连接并共享物理内存和 I/O 端口。它们被称为 UMA 系统，因为无论从哪个处理器发起，内存访问延迟都是统一和固定的。大多数对称多处理器系统都是 UMA。

**非均匀内存访问架构（NUMA）：**这些是多处理器机器，设计与 UMA 相反。这些系统为每个处理器设计了专用内存，并具有固定的访问延迟时间。但是，处理器可以通过适当的互连发起对其他处理器本地内存的访问操作，并且这样的操作会产生可变的访问延迟时间。

这种模型的机器由于每个处理器对系统内存的非均匀（非连续）视图而得名为**NUMA**：

**![](img/00020.jpeg)**

为了扩展对 NUMA 机器的支持，内核将每个非均匀内存分区（本地内存）视为一个`node`。每个节点由`type pg_data_t`的描述符标识，该描述符根据之前讨论的分区策略引用该节点下的页面。每个*区域*通过`struct zone`的实例表示。UMA 机器将包含一个节点描述符，该描述符下表示整个内存，而在 NUMA 机器上，将枚举一系列节点描述符，每个描述一个连续的内存节点。以下图表说明了这些数据结构之间的关系：

![](img/00021.jpeg)

我们将继续使用*节点*和*区域*描述符数据结构定义。请注意，我们不打算描述这些结构的每个元素，因为它们与内存管理的各个方面有关，而这超出了本章的范围。

# 节点描述符结构

节点描述符结构`pg_data_t`在内核头文件`mmzone.h`中声明：

```
/* include/linux/mmzone.h */typedef struct pglist_data {
  struct zone node_zones[MAX_NR_ZONES];
 struct zonelist node_zonelists[MAX_ZONELISTS];
 int nr_zones;

#ifdef CONFIG_FLAT_NODE_MEM_MAP /* means !SPARSEMEM */
  struct page *node_mem_map;
#ifdef CONFIG_PAGE_EXTENSION
  struct page_ext *node_page_ext;
#endif
#endif

#ifndef CONFIG_NO_BOOTMEM
  struct bootmem_data *bdata;
#endif
#ifdef CONFIG_MEMORY_HOTPLUG
 spinlock_t node_size_lock;
#endif
 unsigned long node_start_pfn;
 unsigned long node_present_pages; /* total number of physical pages */
 unsigned long node_spanned_pages; 
 int node_id;
 wait_queue_head_t kswapd_wait;
 wait_queue_head_t pfmemalloc_wait;
 struct task_struct *kswapd; 
 int kswapd_order;
 enum zone_type kswapd_classzone_idx;

#ifdef CONFIG_COMPACTION
 int kcompactd_max_order;
 enum zone_type kcompactd_classzone_idx;
 wait_queue_head_t kcompactd_wait;
 struct task_struct *kcompactd;
#endif
#ifdef CONFIG_NUMA_BALANCING
 spinlock_t numabalancing_migrate_lock;
 unsigned long numabalancing_migrate_next_window;
 unsigned long numabalancing_migrate_nr_pages;
#endif
 unsigned long totalreserve_pages;

#ifdef CONFIG_NUMA
 unsigned long min_unmapped_pages;
 unsigned long min_slab_pages;
#endif /* CONFIG_NUMA */

 ZONE_PADDING(_pad1_)
 spinlock_t lru_lock;

#ifdef CONFIG_DEFERRED_STRUCT_PAGE_INIT
 unsigned long first_deferred_pfn;
#endif /* CONFIG_DEFERRED_STRUCT_PAGE_INIT */

#ifdef CONFIG_TRANSPARENT_HUGEPAGE
 spinlock_t split_queue_lock;
 struct list_head split_queue;
 unsigned long split_queue_len;
#endif
 unsigned int inactive_ratio;
 unsigned long flags;

 ZONE_PADDING(_pad2_)
 struct per_cpu_nodestat __percpu *per_cpu_nodestats;
 atomic_long_t vm_stat[NR_VM_NODE_STAT_ITEMS];
} pg_data_t;
```

根据选择的机器类型和内核配置，各种元素被编译到这个结构中。我们将看一些重要的元素：

| 字段 | 描述 |
| --- | --- |
| `node_zones` | 一个包含此节点中页面的*区域*实例的数组。 |
| `node_zonelists` | 一个指定节点中区域的首选分配顺序的数组。 |
| `nr_zones` | 当前节点中区域的计数。 |
| `node_mem_map` | 指向当前节点中页面描述符列表的指针。 |
| `bdata` | 指向引导内存描述符的指针（在后面的部分中讨论） |
| `node_start_pfn` | 持有此节点中第一个物理页面的帧编号；对于 UMA 系统，此值将为*零*。 |
| `node_present_pages` | 节点中页面的总数 |
| `node_spanned_pages` | 物理页面范围的总大小，包括任何空洞。 |
| `node_id` | 持有唯一节点标识符（节点从零开始编号） |
| `kswapd_wait` | `kswapd`内核线程的等待队列 |
| `kswapd` | 指向`kswapd`内核线程的任务结构的指针 |
| `totalreserve_pages` | 未用于用户空间分配的保留页面的计数。 |

# 区域描述符结构

`mmzone.h`头文件还声明了`struct zone`，它充当*区域*描述符。以下是结构定义的代码片段，并且有很好的注释。我们将继续描述一些重要字段：

```
struct zone {
 /* Read-mostly fields */

 /* zone watermarks, access with *_wmark_pages(zone) macros */
 unsigned long watermark[NR_WMARK];

 unsigned long nr_reserved_highatomic;

 /*
 * We don't know if the memory that we're going to allocate will be
 * freeable or/and it will be released eventually, so to avoid totally
 * wasting several GB of ram we must reserve some of the lower zone
 * memory (otherwise we risk to run OOM on the lower zones despite
 * there being tons of freeable ram on the higher zones). This array is
 * recalculated at runtime if the sysctl_lowmem_reserve_ratio sysctl
 * changes.
 */
 long lowmem_reserve[MAX_NR_ZONES];

#ifdef CONFIG_NUMA
 int node;
#endif
 struct pglist_data *zone_pgdat;
 struct per_cpu_pageset __percpu *pageset;

#ifndef CONFIG_SPARSEMEM
 /*
 * Flags for a pageblock_nr_pages block. See pageblock-flags.h.
 * In SPARSEMEM, this map is stored in struct mem_section
 */
 unsigned long *pageblock_flags;
#endif /* CONFIG_SPARSEMEM */

 /* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
 unsigned long zone_start_pfn;

 /*
 * spanned_pages is the total pages spanned by the zone, including
 * holes, which is calculated as:
 * spanned_pages = zone_end_pfn - zone_start_pfn;
 *
 * present_pages is physical pages existing within the zone, which
 * is calculated as:
 * present_pages = spanned_pages - absent_pages(pages in holes);
 *
 * managed_pages is present pages managed by the buddy system, which
 * is calculated as (reserved_pages includes pages allocated by the
 * bootmem allocator):
 * managed_pages = present_pages - reserved_pages;
 *
 * So present_pages may be used by memory hotplug or memory power
 * management logic to figure out unmanaged pages by checking
 * (present_pages - managed_pages). And managed_pages should be used
 * by page allocator and vm scanner to calculate all kinds of watermarks
 * and thresholds.
 *
 * Locking rules:
 *
 * zone_start_pfn and spanned_pages are protected by span_seqlock.
 * It is a seqlock because it has to be read outside of zone->lock,
 * and it is done in the main allocator path. But, it is written
 * quite infrequently.
 *
 * The span_seq lock is declared along with zone->lock because it is
 * frequently read in proximity to zone->lock. It's good to
 * give them a chance of being in the same cacheline.
 *
 * Write access to present_pages at runtime should be protected by
 * mem_hotplug_begin/end(). Any reader who can't tolerant drift of
 * present_pages should get_online_mems() to get a stable value.
 *
 * Read access to managed_pages should be safe because it's unsigned
 * long. Write access to zone->managed_pages and totalram_pages are
 * protected by managed_page_count_lock at runtime. Idealy only
 * adjust_managed_page_count() should be used instead of directly
 * touching zone->managed_pages and totalram_pages.
 */
 unsigned long managed_pages;
 unsigned long spanned_pages;
 unsigned long present_pages;

 const char *name;// name of this zone

#ifdef CONFIG_MEMORY_ISOLATION
 /*
 * Number of isolated pageblock. It is used to solve incorrect
 * freepage counting problem due to racy retrieving migratetype
 * of pageblock. Protected by zone->lock.
 */
 unsigned long nr_isolate_pageblock;
#endif

#ifdef CONFIG_MEMORY_HOTPLUG
 /* see spanned/present_pages for more description */
 seqlock_t span_seqlock;
#endif

 int initialized;

 /* Write-intensive fields used from the page allocator */
 ZONE_PADDING(_pad1_)

 /* free areas of different sizes */
struct free_area free_area[MAX_ORDER];

 /* zone flags, see below */
 unsigned long flags;

 /* Primarily protects free_area */
 spinlock_t lock;

 /* Write-intensive fields used by compaction and vmstats. */
 ZONE_PADDING(_pad2_)

 /*
 * When free pages are below this point, additional steps are taken
 * when reading the number of free pages to avoid per-CPU counter
 * drift allowing watermarks to be breached
 */
 unsigned long percpu_drift_mark;

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
 /* pfn where compaction free scanner should start */
 unsigned long compact_cached_free_pfn;
 /* pfn where async and sync compaction migration scanner should start */
 unsigned long compact_cached_migrate_pfn[2];
#endif

#ifdef CONFIG_COMPACTION
 /*
 * On compaction failure, 1<<compact_defer_shift compactions
 * are skipped before trying again. The number attempted since
 * last failure is tracked with compact_considered.
 */
 unsigned int compact_considered;
 unsigned int compact_defer_shift;
 int compact_order_failed;
#endif

#if defined CONFIG_COMPACTION || defined CONFIG_CMA
 /* Set to true when the PG_migrate_skip bits should be cleared */
 bool compact_blockskip_flush;
#endif

 bool contiguous;

 ZONE_PADDING(_pad3_)
 /* Zone statistics */
 atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS];
} ____cacheline_internodealigned_in_smp;
```

以下是重要字段的总结表，每个字段都有简短的描述：

| 字段 | 描述 |
| --- | --- |
| `watermark` | 一个无符号长整型数组，具有`WRMARK_MIN`、`WRMARK_LOW`和`WRMARK_HIGH`的偏移量。这些偏移量中的值会影响`kswapd`内核线程执行的交换操作。 |
| `nr_reserved_highatomic` | 保留高阶原子页面的计数 |
| `lowmem_reserve` | 指定为每个*区域*保留用于关键分配的页面计数的数组。 |
| `zone_pgdat` | 指向此*区域*的节点描述符的指针。 |
| `pageset` | 指向每个 CPU 的热和冷页面列表。 |
| `free_area` | 一个`struct free_area`类型实例的数组，每个实例抽象出为伙伴分配器提供的连续空闲页面。更多关于伙伴分配器的内容将在后面的部分中介绍。 |
| `flags` | 用于存储*区域*当前状态的无符号长变量。 |
| `zone_start_pfn` | *区域*中第一个页面帧的索引 |
| `vm_stat` | *区域*的统计信息 |

# 内存分配器

在了解了物理内存是如何组织和通过核心数据结构表示之后，我们现在将把注意力转向处理分配和释放请求的物理内存管理。系统中的各种实体，如用户模式进程、驱动程序和文件系统，可以提出内存分配请求。根据提出分配请求的实体和上下文的类型，返回的分配可能需要满足某些特性，例如页面对齐的物理连续大块或物理连续小块、硬件缓存对齐内存，或映射到虚拟连续地址空间的物理碎片化块。

为了有效地管理物理内存，并根据选择的优先级和模式满足内存需求，内核与一组内存分配器进行交互。每个分配器都有一组不同的接口例程，这些例程由专门设计的算法支持，针对特定的分配模式进行了优化。

# 页面帧分配器

也称为分区页帧分配器，这用作以页面大小的倍数进行物理连续分配的接口。通过查找适当的区域以获取空闲页面来执行分配操作。每个*zone*中的物理页面由**伙伴系统**管理，该系统作为页面帧分配器的后端算法：

![](img/00022.jpeg)

内核代码可以通过内核头文件`linux/include/gfp.h`中提供的接口内联函数和宏来启动对该算法的内存分配/释放操作：

```
static inline struct page *alloc_pages(gfp_t gfp_mask, unsigned int order);
```

第一个参数`gfp_mask`用作指定属性的手段，根据这些属性来满足分配的需求；我们将在接下来的部分详细了解属性标志。第二个参数`order`用于指定分配的大小；分配的值被认为是 2^(order)。成功时，它返回第一个页面结构的地址，失败时返回 NULL。对于单页分配，还提供了一个备用宏，它再次回退到`alloc_pages()`：

```
#define alloc_page(gfp_mask) alloc_pages(gfp_mask, 0);
```

分配的页面被映射到连续的内核地址空间，通过适当的页表项（用于访问操作期间的分页地址转换）。在页表映射后生成的地址，用于内核代码中的使用，被称为**线性地址**。通过另一个函数接口`page_address()`，调用者代码可以检索分配块的起始线性地址。

分配也可以通过一组**包装器**例程和宏来启动到`alloc_pages()`的操作，这些例程和宏略微扩展了功能，并返回分配块的起始线性地址，而不是页面结构的指针。以下代码片段显示了一组包装器函数和宏：

```
/* allocates 2^(order) pages and returns start linear address */ unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order)
{
struct page *page;
/*
* __get_free_pages() returns a 32-bit address, which cannot represent
* a highmem page
*/
VM_BUG_ON((gfp_mask & __GFP_HIGHMEM) != 0);

page = alloc_pages(gfp_mask, order);
if (!page)
return 0;
return (unsigned long) page_address(page);
}

/* Returns start linear address to zero initialized page */
unsigned long get_zeroed_page(gfp_t gfp_mask)
{
return __get_free_pages(gfp_mask | __GFP_ZERO, 0);
}
 /* Allocates a page */
#define __get_free_page(gfp_mask) \
__get_free_pages((gfp_mask), 0)

/* Allocate page/pages from DMA zone */
#define __get_dma_pages(gfp_mask, order) \
 __get_free_pages((gfp_mask) | GFP_DMA, (order))

```

以下是释放内存返回到系统的接口。我们需要调用一个与分配例程匹配的适当接口；传递不正确的地址将导致损坏。

```
void __free_pages(struct page *page, unsigned int order);
void free_pages(unsigned long addr, unsigned int order);
void free_page(addr);
```

# 伙伴系统

虽然页面分配器用作内存分配的接口（以页面大小的倍数），但伙伴系统在后台运行以管理物理页面管理。该算法管理每个*zone*的所有物理页面。它经过优化，以最小化外部碎片化，实现大型物理连续块（页面）的分配。让我们探索其操作细节*.*

*zone*描述符结构包含一个*`struct free_area`*数组，数组的大小是通过内核宏`MAX_ORDER`定义的，默认值为`11`：

```
  struct zone {
          ...
          ...
          struct free_area[MAX_ORDER];
          ...
          ...
     };
```

每个偏移包含一个`free_area`结构的实例。所有空闲页面被分成 11 个（`MAX_ORDER`）列表，每个列表包含 2^(order)页的块列表，其中 order 的值在 0 到 11 的范围内（也就是说，2²的列表包含 16KB 大小的块，2³包含 32KB 大小的块，依此类推）。这种策略确保每个块都自然对齐。每个列表中的块大小恰好是低级列表中块大小的两倍，从而实现更快的分配和释放操作。它还为分配器提供了处理连续分配的能力，最多可达 8MB 的块大小（2¹¹列表）。

![](img/00023.jpeg)

当针对特定大小的分配请求时，*伙伴系统*会查找适当的空闲块列表，并返回其地址（如果有的话）。然而，如果找不到空闲块，它会移动到下一个高阶列表中查找更大的块，如果有的话，它会将高阶块分割成称为*伙伴*的相等部分，返回一个给分配器，并将第二个排入低阶列表。当两个伙伴块在将来某个时间变为空闲时，它们将合并为一个更大的块。算法可以通过它们对齐的地址来识别伙伴块，这使得它们可以合并。

让我们举个例子来更好地理解这一点，假设有一个请求来分配一个 8k 的块（通过页面分配器例程）。伙伴系统在`free_pages`数组的 8k 列表中查找空闲块（第一个偏移包含 2¹大小的块），如果有的话，返回块的起始线性地址；然而，如果在 8k 列表中没有空闲块，它会移动到下一个更高阶的列表，即 16k 块（`free_pages`数组的第二个偏移）中查找空闲块。假设在这个列表中也没有空闲块。然后它继续前进到大小为 32k 的下一个高阶列表（*free_pages*数组的第三个偏移）中查找空闲块；如果有的话，它将 32k 块分成两个相等的 16k 块（*伙伴*）。第一个 16k 块进一步分成两个 8k 的半块（*伙伴*），其中一个分配给调用者，另一个放入 8k 列表。第二个 16k 块放入 16k 空闲列表，当低阶（8k）伙伴在将来的某个时间变为空闲时，它们将合并为一个更高阶的 16k 块。当两个 16k 伙伴块都变为空闲时，它们再次合并为一个 32k 块，然后放回空闲列表。

当无法处理来自所需*区域*的分配请求时，伙伴系统使用回退机制来查找其他区域和节点：

![](img/00024.jpeg)

*伙伴系统*在各种*nix 操作系统中有着悠久的历史，并进行了广泛的实现和适当的优化。正如前面讨论的那样，它有助于更快的内存分配和释放，并且在一定程度上最小化了外部碎片化。随着提供了急需的性能优势的*大页*的出现，进一步努力以抵制碎片化变得更加重要。为了实现这一点，Linux 内核对伙伴系统的实现配备了通过页面迁移实现抵制碎片化的能力。

**页面迁移**是将虚拟页面的数据从一个物理内存区域移动到另一个的过程。这种机制有助于创建具有连续页面的更大块。为了实现这一点，页面被归类为以下类型：

**1. 不可移动页面**：被固定并保留用于特定分配的物理页面被视为不可移动。核心内核固定的页面属于这一类。这些页面是不可回收的。

2. 可回收页面：映射到动态分配的物理页面可以被驱逐到后备存储器，并且可以重新生成的页面被认为是*可回收*的。用于文件缓存，匿名页面映射以及内核的 slab 缓存持有的页面都属于这个类别。回收操作以两种模式进行：周期性回收和直接回收，前者通过称为*`kswapd`*的 kthread 实现。当系统内存严重不足时，内核进入*直接回收*。

3. 可移动页面：可以通过页面迁移机制*移动到*不同区域的物理页面。映射到用户模式*进程*的虚拟地址空间的页面被认为是可移动的，因为所有 VM 子系统需要做的就是复制数据并更改相关的页表条目。这是有效的，考虑到所有来自用户模式*进程*的访问操作都经过页表翻译。

伙伴系统根据页面的*可移动性*将页面分组为独立列表，并将它们用于适当的分配。这是通过将`struct free_area`中的每个 2^n 列表组织为基于页面移动性的自主列表组实现的。每个`free_area`实例都持有大小为`MIGRATE_TYPES`的列表数组。每个偏移量都持有相应页面组的`list_head`：

```
 struct free_area {
          struct list_head free_list[MIGRATE_TYPES];
          unsigned long nr_free;
    };
```

`nr_free`是一个计数器，它保存了此`free_area`（所有迁移列表放在一起）的空闲页面总数。以下图表描述了每种迁移类型的空闲列表：

![](img/00025.jpeg)

以下枚举定义了页面迁移类型：

```
enum {
 MIGRATE_UNMOVABLE,
 MIGRATE_MOVABLE,
 MIGRATE_RECLAIMABLE,
 MIGRATE_PCPTYPES, /* the number of types on the pcp lists */
 MIGRATE_HIGHATOMIC = MIGRATE_PCPTYPES,
#ifdef CONFIG_CMA
 MIGRATE_CMA,
#endif
#ifdef CONFIG_MEMORY_ISOLATION
 MIGRATE_ISOLATE, /* can't allocate from here */
#endif
 MIGRATE_TYPES
};  
```

我们已经讨论了关键的迁移类型`MIGRATE_MOVABLE`，`MIGRATE_UNMOVABLE`和`MIGRATE_RECLAIMABLE`类型。`MIGRATE_PCPTYPES`是一种特殊类型，用于提高系统性能；每个*区域*维护一个每 CPU 页缓存中的热缓存页面列表。这些页面用于为本地 CPU 提出的分配请求提供服务。*区域*描述符结构`pageset`元素指向每 CPU 缓存中的页面：

```
/* include/linux/mmzone.h */

struct per_cpu_pages {
 int count; /* number of pages in the list */
 int high; /* high watermark, emptying needed */
 int batch; /* chunk size for buddy add/remove */

 /* Lists of pages, one per migrate type stored on the pcp-lists */
 struct list_head lists[MIGRATE_PCPTYPES];
};

struct per_cpu_pageset {
 struct per_cpu_pages pcp;
#ifdef CONFIG_NUMA
 s8 expire;
#endif
#ifdef CONFIG_SMP
 s8 stat_threshold;
 s8 vm_stat_diff[NR_VM_ZONE_STAT_ITEMS];
#endif
};

struct zone {
 ...
 ...
 struct per_cpu_pageset __percpu *pageset;
 ...
 ...
};
```

`struct per_cpu_pageset`是一个表示*不可移动*，*可回收*和*可移动*页面列表的抽象。`MIGRATE_PCPTYPES`是按页面*移动性*排序的每 CPU 页面列表的计数。`MIGRATE_CMA`是连续内存分配器的页面列表，我们将在后续部分中讨论：

**![](img/00026.jpeg)**

当所需移动性的页面不可用时，伙伴系统实现了*回退*到备用列表，以处理分配请求。以下数组定义了各种迁移类型的回退顺序；我们不会进一步详细说明，因为它是不言自明的：

```
static int fallbacks[MIGRATE_TYPES][4] = {
 [MIGRATE_UNMOVABLE] = { MIGRATE_RECLAIMABLE, MIGRATE_MOVABLE, MIGRATE_TYPES },
 [MIGRATE_RECLAIMABLE] = { MIGRATE_UNMOVABLE, MIGRATE_MOVABLE, MIGRATE_TYPES },
 [MIGRATE_MOVABLE] = { MIGRATE_RECLAIMABLE, MIGRATE_UNMOVABLE, MIGRATE_TYPES },
#ifdef CONFIG_CMA
 [MIGRATE_CMA] = { MIGRATE_TYPES }, /* Never used */
#endif
#ifdef CONFIG_MEMORY_ISOLATION
 [MIGRATE_ISOLATE] = { MIGRATE_TYPES }, /* Never used */
#endif
};
```

# GFP 掩码

页面分配器和其他分配器例程（我们将在以下部分讨论）需要`gfp_mask`标志作为参数，其类型为`gfp_t`：

```
typedef unsigned __bitwise__ gfp_t;
```

Gfp 标志用于为分配器函数提供两个重要属性：第一个是分配的**模式**，它控制分配器函数的行为*，*第二个是分配的*来源*，它指示可以从中获取内存的*区域*或*区域*列表*。*内核头文件`gfp.h`定义了各种标志常量，这些常量被分类为不同的组，称为**区域修饰符，移动性和** **放置标志，水位标志，回收修饰符**和**操作修饰符。**

# 区域修饰符

以下是用于指定要从中获取内存的*区域*的修饰符的总结列表。回顾我们在前一节中对*区域*的讨论；对于每个*区域*，都定义了一个`gfp`标志：

```
#define __GFP_DMA ((__force gfp_t)___GFP_DMA)
#define __GFP_HIGHMEM ((__force gfp_t)___GFP_HIGHMEM)
#define __GFP_DMA32 ((__force gfp_t)___GFP_DMA32)
#define __GFP_MOVABLE ((__force gfp_t)___GFP_MOVABLE) /* ZONE_MOVABLE allowed */
```

# 页面移动性和放置

以下代码片段定义了页面移动性和放置标志：

```
#define __GFP_RECLAIMABLE ((__force gfp_t)___GFP_RECLAIMABLE)
#define __GFP_WRITE ((__force gfp_t)___GFP_WRITE)
#define __GFP_HARDWALL ((__force gfp_t)___GFP_HARDWALL)
#define __GFP_THISNODE ((__force gfp_t)___GFP_THISNODE)
#define __GFP_ACCOUNT ((__force gfp_t)___GFP_ACCOUNT)
```

以下是页面移动性和放置标志的列表：

+   `__GFP_RECLAIMABLE`：大多数内核子系统都设计为使用*内存缓存*来缓存频繁需要的资源，例如数据结构、内存块、持久文件数据等。内存管理器维护这些缓存，并允许它们根据需要动态扩展。但是，不能无限制地扩展这些缓存，否则它们最终会消耗所有内存。内存管理器通过**shrinker**接口处理此问题，这是一种内存管理器可以在需要时缩小缓存并回收页面的机制。在分配页面（用于缓存）时启用此标志表示向 shrinker 指示页面是*可回收*的。这个标志由后面的部分讨论的 slab 分配器使用。

+   `__GFP_WRITE`：当使用此标志时，表示向内核指示调用者打算污染页面。内存管理器根据公平区分配策略分配适当的页面，该策略在节点的本地*区域*之间轮流分配这些页面，以避免所有脏页面都在一个*区域*中。

+   `__GFP_HARDWALL`：此标志确保分配在与调用者绑定的相同节点或节点上进行；换句话说，它强制执行 CPUSET 内存分配策略。

+   `__GFP_THISNODE`：此标志强制满足分配请求来自请求的节点，没有回退或放置策略的强制执行。

+   `__GFP_ACCOUNT`：此标志导致分配被 kmem 控制组记录。

# 水印修饰符

以下代码片段定义了水印修饰符：

```
#define __GFP_ATOMIC ((__force gfp_t)___GFP_ATOMIC)
#define __GFP_HIGH ((__force gfp_t)___GFP_HIGH)
#define __GFP_MEMALLOC ((__force gfp_t)___GFP_MEMALLOC)
#define __GFP_NOMEMALLOC ((__force gfp_t)___GFP_NOMEMALLOC)
```

以下是水印修饰符的列表，它们可以控制内存的紧急保留池：

+   `__GFP_ATOMIC`：此标志表示分配具有高优先级，并且调用者上下文不能被置于等待状态。

+   `__GFP_HIGH`：此标志表示调用者具有高优先级，并且必须满足分配请求以使系统取得进展。设置此标志将导致分配器访问紧急池。

+   `__GFP_MEMALLOC`：此标志允许访问所有内存。只有在调用者保证分配很快就会释放更多内存时才应使用，例如，进程退出或交换。

+   `__GFP_NOMEMALLOC`：此标志用于禁止访问所有保留的紧急池。

# 页面回收修饰符

随着系统负载的增加，*区域*中的空闲内存量可能会低于*低水位标记*，导致内存紧缩，这将严重影响系统的整体性能*。为了处理这种可能性，内存管理器配备了**页面回收算法**，用于识别和回收页面。当使用适当的 GFP 常量调用内核内存分配器例程时，会启用回收算法，称为**页面回收修饰符**：

```
#define __GFP_IO ((__force gfp_t)___GFP_IO)
#define __GFP_FS ((__force gfp_t)___GFP_FS)
#define __GFP_DIRECT_RECLAIM ((__force gfp_t)___GFP_DIRECT_RECLAIM) /* Caller can reclaim */
#define __GFP_KSWAPD_RECLAIM ((__force gfp_t)___GFP_KSWAPD_RECLAIM) /* kswapd can wake */
#define __GFP_RECLAIM ((__force gfp_t)(___GFP_DIRECT_RECLAIM|___GFP_KSWAPD_RECLAIM))
#define __GFP_REPEAT ((__force gfp_t)___GFP_REPEAT)
#define __GFP_NOFAIL ((__force gfp_t)___GFP_NOFAIL)
#define __GFP_NORETRY ((__force gfp_t)___GFP_NORETRY)
```

以下是可以作为参数传递给分配例程的回收修饰符列表；每个标志都可以在特定内存区域上启用回收操作：

+   `__GFP_IO`：此标志表示分配器可以启动物理 I/O（交换）以回收内存。

+   `__GFP_FS`：此标志表示分配器可以调用低级 FS 进行回收。

+   `__GFP_DIRECT_RECLAIM`：此标志表示调用者愿意进行直接回收。这可能会导致调用者阻塞。

+   `__GFP_KSWAPD_RECLAIM`：此标志表示分配器可以唤醒`kswapd`内核线程来启动回收，当低水位标记达到时。

+   `__GFP_RECLAIM`：此标志用于启用直接和`kswapd`回收。

+   `__GFP_REPEAT`：此标志表示尝试努力分配内存，但分配尝试可能失败。

+   `__GFP_NOFAIL`：此标志强制虚拟内存管理器*重试*，直到分配请求成功。这可能会导致 VM 触发 OOM killer 来回收内存。

+   `__GFP_NORETRY`：当无法满足请求时，此标志将导致分配器返回适当的失败状态。

# 动作修饰符

以下代码片段定义了动作修饰符：

```
#define __GFP_COLD ((__force gfp_t)___GFP_COLD)
#define __GFP_NOWARN ((__force gfp_t)___GFP_NOWARN)
#define __GFP_COMP ((__force gfp_t)___GFP_COMP)
#define __GFP_ZERO ((__force gfp_t)___GFP_ZERO)
#define __GFP_NOTRACK ((__force gfp_t)___GFP_NOTRACK)
#define __GFP_NOTRACK_FALSE_POSITIVE (__GFP_NOTRACK)
#define __GFP_OTHER_NODE ((__force gfp_t)___GFP_OTHER_NODE)
```

以下是动作修饰符标志的列表；这些标志指定了在处理请求时分配器例程要考虑的附加属性：

+   **`__GFP_COLD`**：为了实现快速访问，每个*区域*中的一些页面被缓存在每个 CPU 的缓存中；缓存中保存的页面被称为**热**，而未缓存的页面被称为**冷**。此标志表示分配器应通过缓存冷页面来处理内存请求。

+   **`__GFP_NOWARN`**：此标志导致分配器以静默模式运行，导致警告和错误条件不被报告。

+   **`__GFP_COMP`**：此标志用于分配带有适当元数据的复合页面。复合页面是两个或更多个物理上连续的页面组成的，被视为单个大页面。元数据使复合页面与其他物理上连续的页面不同。复合页面的第一个物理页面称为**头页面**，其页面描述符中设置了`PG_head`标志，其余页面称为**尾页面**。

+   **`__GFP_ZERO`**：此标志导致分配器返回填充为零的页面。

+   **`__GFP_NOTRACK`**：kmemcheck 是内核中的一个调试器，用于检测和警告未初始化的内存访问。尽管如此，这些检查会导致内存访问操作被延迟。当性能是一个标准时，调用者可能希望分配不被 kmemcheck 跟踪的内存。此标志导致分配器返回这样的内存。

+   **`__GFP_NOTRACK_FALSE_POSITIVE`**：此标志是**`__GFP_NOTRACK`**的别名。

+   `__GFP_OTHER_NODE`：此标志用于分配透明巨大页面（THP）。

# 类型标志

由于有这么多类别的修饰符标志（每个都涉及不同的属性），程序员在选择相应分配的标志时要非常小心。为了使这个过程更容易、更快速，引入了类型标志，使程序员能够快速进行分配选择。**类型标志**是从各种修饰常量的组合（前面列出的）中派生出来的，用于特定的分配用例。然而，如果需要，程序员可以进一步自定义类型标志：

```
#define GFP_ATOMIC (__GFP_HIGH|__GFP_ATOMIC|__GFP_KSWAPD_RECLAIM)
#define GFP_KERNEL (__GFP_RECLAIM | __GFP_IO | __GFP_FS)
#define GFP_KERNEL_ACCOUNT (GFP_KERNEL | __GFP_ACCOUNT)
#define GFP_NOWAIT (__GFP_KSWAPD_RECLAIM)
#define GFP_NOIO (__GFP_RECLAIM)
#define GFP_NOFS (__GFP_RECLAIM | __GFP_IO)
#define GFP_TEMPORARY (__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_RECLAIMABLE)
#define GFP_USER (__GFP_RECLAIM | __GFP_IO | __GFP_FS | __GFP_HARDWALL)
#define GFP_DMA __GFP_DMA
#define GFP_DMA32 __GFP_DMA32
#define GFP_HIGHUSER (GFP_USER | __GFP_HIGHMEM)
#define GFP_HIGHUSER_MOVABLE (GFP_HIGHUSER | __GFP_MOVABLE)
#define GFP_TRANSHUGE_LIGHT ((GFP_HIGHUSER_MOVABLE | __GFP_COMP | __GFP_NOMEMALLOC | \ __GFP_NOWARN) & ~__GFP_RECLAIM)
#define GFP_TRANSHUGE (GFP_TRANSHUGE_LIGHT | __GFP_DIRECT_RECLAIM)
```

以下是类型标志的列表：

+   **`GFP_ATOMIC`**：指定非阻塞分配的标志，这种分配不会失败。此标志将导致从紧急储备中分配。通常在从原子上下文调用分配器时使用。

+   **`GFP_KERNEL`**：在为内核使用分配内存时使用此标志。这些请求是从正常*区域*处理的。此标志可能导致分配器进入直接回收。

+   **`GFP_KERNEL_ACCOUNT`**：与`GFP_KERNEL`相同，但额外增加了由 kmem 控制组跟踪分配的标志**。**

+   `GFP_NOWAIT`：此标志用于非阻塞的内核分配。

+   `GFP_NOIO`：此标志允许分配器在不需要物理 I/O（交换）的干净页面上开始直接回收。

+   `GFP_NOFS`：此标志允许分配器开始直接回收，但阻止调用文件系统接口。

+   **`GFP_TEMPORARY`**：在为内核缓存分配页面时使用此标志，通过适当的收缩器接口可以回收这些页面。此标志设置了我们之前讨论过的`__GFP_RECLAIMABLE`标志。

+   **`GFP_USER`**：此标志用于用户空间分配。分配的内存被映射到用户进程，并且也可以被内核服务或硬件访问，用于从设备到缓冲区或反之的 DMA 传输。

+   **`GFP_DMA`**：此标志导致从最低的*区域*`ZONE_DMA`中分配。这个标志仍然为向后兼容而支持。

+   `GFP_DMA32`：此标志导致从包含<4G 内存的`ZONE_DMA32`中处理分配。

+   `GFP_HIGHUSER`：此标志用于从**`ZONE_HIGHMEM`**（仅在 32 位平台上相关）分配用户空间分配。

+   `GFP_HIGHUSER_MOVABLE`：此标志类似于`GFP_HIGHUSER`，另外还可以从可移动页面中进行分配，这使得页面迁移和回收成为可能。

+   **`GFP_TRANSHUGE_LIGHT`**：这会导致透明巨大分配（THP）的分配，这是复合分配。这种类型的标志设置了`__GFP_COMP`，我们之前讨论过。

# 粘土块分配器

如前面的部分所讨论的，页面分配器（与伙伴系统协调）有效地处理了页面大小的多重内存分配请求。然而，内核代码发起的大多数分配请求用于其内部使用的较小块（通常小于一页）；为这样的分配请求启用页面分配器会导致*内部碎片*，导致内存浪费。粘土块分配器正是为了解决这个问题而实现的；它建立在伙伴系统之上，用于分配小内存块，以容纳内核服务使用的结构对象或数据。

粘土块分配器的设计基于*对象* *缓存*的概念。**对象缓存**的概念非常简单：它涉及保留一组空闲页面帧，将它们分割并组织成独立的空闲列表（每个列表包含一些空闲页面），称为**粘土块缓存**，并使用每个列表来分配一组固定大小的对象或内存块，称为**单元**。这样，每个列表被分配一个唯一的*单元*大小，并包含该大小的对象或内存块的池。当收到对给定大小的内存块的分配请求时，分配器算法会选择一个适当的*粘土块缓存*，其*单元*大小最适合请求的大小，并返回一个空闲块的地址。

然而，在低级别上，初始化和管理粘土块缓存涉及相当复杂的问题。算法需要考虑各种问题，如对象跟踪、动态扩展和通过 shrinker 接口进行安全回收。解决所有这些问题，并在增强性能和最佳内存占用之间取得适当的平衡是相当具有挑战性的。我们将在后续部分更多地探讨这些挑战，但现在我们将继续讨论分配器函数接口。

# Kmalloc 缓存

粘土块分配器维护一组通用粘土块缓存，以缓存 8 的倍数的*单元*大小的内存块。它为每个*单元*大小维护两组粘土块缓存，一组用于维护从`ZONE_NORMAL`页面分配的内存块池，另一组用于维护从`ZONE_DMA`页面分配的内存块池。这些缓存是全局的，并由所有内核代码共享。用户可以通过特殊文件`/proc/slabinfo`跟踪这些缓存的状态。内核服务可以通过`kmalloc`系列例程从这些缓存中分配和释放内存块。它们被称为`kmalloc`缓存：

```
#cat /proc/slabinfo 
slabinfo - version: 2.1
# name <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>dma-kmalloc-8192 0 0 8192 4 8 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-4096 0 0 4096 8 8 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-2048 0 0 2048 16 8 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-1024 0 0 1024 16 4 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-512 0 0 512 16 2 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-256 0 0 256 16 1 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-128 0 0 128 32 1 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-64 0 0 64 64 1 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-32 0 0 32 128 1 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-16 0 0 16 256 1 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-8 0 0 8 512 1 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-192 0 0 192 21 1 : tunables 0 0 0 : slabdata 0 0 0
dma-kmalloc-96 0 0 96 42 1 : tunables 0 0 0 : slabdata 0 0 0
kmalloc-8192 156 156 8192 4 8 : tunables 0 0 0 : slabdata 39 39 0
kmalloc-4096 325 352 4096 8 8 : tunables 0 0 0 : slabdata 44 44 0
kmalloc-2048 1105 1184 2048 16 8 : tunables 0 0 0 : slabdata 74 74 0
kmalloc-1024 2374 2448 1024 16 4 : tunables 0 0 0 : slabdata 153 153 0
kmalloc-512 1445 1520 512 16 2 : tunables 0 0 0 : slabdata 95 95 0
kmalloc-256 9988 10400 256 16 1 : tunables 0 0 0 : slabdata 650 650 0
kmalloc-192 3561 4053 192 21 1 : tunables 0 0 0 : slabdata 193 193 0
kmalloc-128 3588 5728 128 32 1 : tunables 0 0 0 : slabdata 179 179 0
kmalloc-96 3402 3402 96 42 1 : tunables 0 0 0 : slabdata 81 81 0
kmalloc-64 42672 45184 64 64 1 : tunables 0 0 0 : slabdata 706 706 0
kmalloc-32 15095 16000 32 128 1 : tunables 0 0 0 : slabdata 125 125 0
kmalloc-16 6400 6400 16 256 1 : tunables 0 0 0 : slabdata 25 25 0
kmalloc-8 6144 6144 8 512 1 : tunables 0 0 0 : slabdata 12 12 0
```

`kmalloc-96`和`kmalloc-192`是用于维护与一级硬件缓存对齐的内存块的缓存。对于大于 8k 的分配（大块），粘土块分配器会回退到伙伴系统。

以下是 kmalloc 系列分配器例程；所有这些都需要适当的 GFP 标志：

```
/**
 * kmalloc - allocate memory. 
 * @size: bytes of memory required.
 * @flags: the type of memory to allocate.
 */
/**
 * kmalloc_array - allocate memory for an array.
 * @n: number of elements.
 * @size: element size.
 * @flags: the type of memory to allocate (see kmalloc).
 */ 
/**
 * kcalloc - allocate memory for an array. The memory is set to zero.
 * @n: number of elements.
 * @size: element size.
 * @flags: the type of memory to allocate (see kmalloc).
 *//**
 * krealloc - reallocate memory. The contents will remain unchanged.
 * @p: object to reallocate memory for.
 * @new_size: bytes of memory are required.
 * @flags: the type of memory to allocate.
 *
 * The contents of the object pointed to are preserved up to the 
 * lesser of the new and old sizes. If @p is %NULL, krealloc()
 * behaves exactly like kmalloc(). If @new_size is 0 and @p is not a
 * %NULL pointer, the object pointed to is freed
 */
 void *krealloc(const void *p, size_t new_size, gfp_t flags) /**
 * kmalloc_node - allocate memory from a particular memory node.
 * @size: bytes of memory are required.
 * @flags: the type of memory to allocate.
 * @node: memory node from which to allocate
 */ void *kmalloc_node(size_t size, gfp_t flags, int node) /**
 * kzalloc_node - allocate zeroed memory from a particular memory node.
 * @size: how many bytes of memory are required.
 * @flags: the type of memory to allocate (see kmalloc).
 * @node: memory node from which to allocate
 */ void *kzalloc_node(size_t size, gfp_t flags, int node)
```

以下例程将分配的块返回到空闲池中。调用者需要确保作为参数传递的地址是有效的分配块：

```
/**
 * kfree - free previously allocated memory
 * @objp: pointer returned by kmalloc.
 *
 * If @objp is NULL, no operation is performed.
 *
 * Don't free memory not originally allocated by kmalloc()
 * or you will run into trouble.
 */
void kfree(const void *objp) /**
 * kzfree - like kfree but zero memory
 * @p: object to free memory of
 *
 * The memory of the object @p points to is zeroed before freed.
 * If @p is %NULL, kzfree() does nothing.
 *
 * Note: this function zeroes the whole allocated buffer which can be a good
 * deal bigger than the requested buffer size passed to kmalloc(). So be
 * careful when using this function in performance sensitive code.
 */ void kzfree(const void *p)
```

# 对象缓存

slab 分配器提供了用于设置 slab 缓存的函数接口，这些缓存可以由内核服务或子系统拥有。由于这些缓存是局部于内核服务（或内核子系统）的，因此被认为是私有的，例如设备驱动程序、文件系统、进程调度程序等。大多数内核子系统使用此功能来设置对象缓存和池化间歇性需要的数据结构。到目前为止，我们遇到的大多数数据结构（自第一章以来，*理解进程、地址空间和线程*），包括进程描述符、信号描述符、页面描述符等，都是在这样的对象池中维护的。伪文件`/proc/slabinfo`显示了对象缓存的状态：

```
# cat /proc/slabinfo 
slabinfo - version: 2.1
# name <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>
sigqueue 100 100 160 25 1 : tunables 0 0 0 : slabdata 4 4 0
bdev_cache 76 76 832 19 4 : tunables 0 0 0 : slabdata 4 4 0
kernfs_node_cache 28594 28594 120 34 1 : tunables 0 0 0 : slabdata 841 841 0
mnt_cache 489 588 384 21 2 : tunables 0 0 0 : slabdata 28 28 0
inode_cache 15932 15932 568 28 4 : tunables 0 0 0 : slabdata 569 569 0
dentry 89541 89817 192 21 1 : tunables 0 0 0 : slabdata 4277 4277 0
iint_cache 0 0 72 56 1 : tunables 0 0 0 : slabdata 0 0 0
buffer_head 53079 53430 104 39 1 : tunables 0 0 0 : slabdata 1370 1370 0
vm_area_struct 41287 42400 200 20 1 : tunables 0 0 0 : slabdata 2120 2120 0
files_cache 207 207 704 23 4 : tunables 0 0 0 : slabdata 9 9 0
signal_cache 420 420 1088 30 8 : tunables 0 0 0 : slabdata 14 14 0
sighand_cache 289 315 2112 15 8 : tunables 0 0 0 : slabdata 21 21 0
task_struct 750 801 3584 9 8 : tunables 0 0 0 : slabdata 89 89 0
```

`*kmem_cache_create()*`例程根据传递的参数设置一个新的*cache*。成功后，它将返回`*kmem_cache*`类型的缓存描述符结构的地址：

```
/*
 * kmem_cache_create - Create a cache.
 * @name: A string which is used in /proc/slabinfo to identify this cache.
 * @size: The size of objects to be created in this cache.
 * @align: The required alignment for the objects.
 * @flags: SLAB flags
 * @ctor: A constructor for the objects.
 *
 * Returns a ptr to the cache on success, NULL on failure.
 * Cannot be called within a interrupt, but can be interrupted.
 * The @ctor is run when new pages are allocated by the cache.
 *
 */
struct kmem_cache * kmem_cache_create(const char *name, size_t size, size_t align,
                                      unsigned long flags, void (*ctor)(void *))
```

缓存是通过分配空闲页面帧（来自伙伴系统）创建的，并且指定大小的数据对象（第二个参数）会被填充。尽管每个缓存在创建时都会托管固定数量的数据对象，但在需要时它们可以动态增长以容纳更多的数据对象。数据结构可能会很复杂（我们遇到了一些），并且可能包含各种元素，如列表头、子对象、数组、原子计数器、位字段等。设置每个对象可能需要将其所有字段初始化为默认状态；这可以通过分配给`*ctor`函数指针（最后一个参数）的初始化程序来实现。初始化程序会在分配每个新对象时调用，无论是在缓存创建时还是在增长以添加更多空闲对象时。然而，对于简单的对象，可以创建一个没有初始化程序的缓存。

```
kmem_cache_create():
```

```
/* net/core/skbuff.c */

struct kmem_cache *skbuff_head_cache;
skbuff_head_cache = kmem_cache_create("skbuff_head_cache",sizeof(struct sk_buff), 0, 
                                       SLAB_HWCACHE_ALIGN|SLAB_PANIC, NULL);                                                                                                   
```

标志用于启用调试检查，并通过将对象与硬件缓存对齐来增强对缓存的访问操作的性能。支持以下标志常量：

```
 SLAB_CONSISTENCY_CHECKS /* DEBUG: Perform (expensive) checks o alloc/free */
 SLAB_RED_ZONE /* DEBUG: Red zone objs in a cache */
 SLAB_POISON /* DEBUG: Poison objects */
 SLAB_HWCACHE_ALIGN  /* Align objs on cache lines */
 SLAB_CACHE_DMA  /* Use GFP_DMA memory */
 SLAB_STORE_USER  /* DEBUG: Store the last owner for bug hunting */
 SLAB_PANIC  /* Panic if kmem_cache_create() fails */

```

随后，*对象*可以通过相关函数进行分配和释放。释放后，*对象*将放回到*cache*的空闲列表中，使其可以重新使用；这可能会带来性能提升，特别是当*对象*是缓存热点时。

```
/**
 * kmem_cache_alloc - Allocate an object
 * @cachep: The cache to allocate from.
 * @flags: GFP mask.
 *
 * Allocate an object from this cache. The flags are only relevant
 * if the cache has no available objects.
 */
void *kmem_cache_alloc(struct kmem_cache *cachep, gfp_t flags);

/**
 * kmem_cache_alloc_node - Allocate an object on the specified node
 * @cachep: The cache to allocate from.
 * @flags: GFP mask.
 * @nodeid: node number of the target node.
 *
 * Identical to kmem_cache_alloc but it will allocate memory on the given
 * node, which can improve the performance for cpu bound structures.
 *
 * Fallback to other node is possible if __GFP_THISNODE is not set.
 */
void *kmem_cache_alloc_node(struct kmem_cache *cachep, gfp_t flags, int nodeid); /**
 * kmem_cache_free - Deallocate an object
 * @cachep: The cache the allocation was from.
 * @objp: The previously allocated object.
 *
 * Free an object which was previously allocated from this
 * cache.
 */ void kmem_cache_free(struct kmem_cache *cachep, void *objp);
```

当所有托管的数据对象都是*free*（未使用）时，可以通过调用`kmem_cache_destroy()`来销毁 kmem 缓存。

# 缓存管理

所有 slab 缓存都由**slab 核心**在内部管理，这是一个低级算法。它定义了描述每个**缓存列表**的物理布局的各种控制结构，并实现了由接口例程调用的核心缓存管理操作。slab 分配器最初是在 Solaris 2.4 内核中实现的，并且被大多数其他*nix 内核使用，基于 Bonwick 的一篇论文。

传统上，Linux 被用于具有中等内存的单处理器桌面和服务器系统，并且内核采用了 Bonwick 的经典模型，并进行了适当的性能改进。多年来，由于 Linux 内核被移植和使用的平台多样性，对于所有需求来说，slab 核心算法的经典实现都是低效的。虽然内存受限的嵌入式平台无法承受分配器的更高占用空间（用于管理元数据和分配器操作的密度），但具有大内存的 SMP 系统需要一致的性能、可伸缩性，并且需要更好的机制来生成分配的跟踪和调试信息。

为了满足这些不同的要求，当前版本的内核提供了 slab 算法的三种不同实现：**slob**，一个经典的 K&R 类型的链表分配器，设计用于内存稀缺的低内存系统，并且在 Linux 的最初几年（1991-1999）是默认的对象分配器；**slab**，一个经典的 Solaris 风格的 slab 分配器，自 1999 年以来一直存在于 Linux 中；以及**slub**，针对当前一代 SMP 硬件和巨大内存进行了改进，并提供了更好的控制和调试机制。大多数架构的默认内核配置都将**slub**作为默认的 slab 分配器；这可以在内核构建过程中通过内核配置选项进行更改。

`CONFIG_SLAB`：常规的 slab 分配器在所有环境中都已经建立并且运行良好。它将缓存热对象组织在每个 CPU 和每个节点队列中。

`CONFIG_SLUB`：**SLUB**是一个最小化缓存行使用而不是管理缓存对象队列（SLAB 方法）的 slab 分配器。使用对象的 slab 而不是对象队列来实现每个 CPU 的缓存。SLUB 可以高效地使用内存，并具有增强的诊断功能。SLUB 是 slab 分配器的默认选择。

`CONFIG_SLOB`：**SLOB**用一个极其简化的分配器替换了原始的分配器。SLOB 通常更节省空间，但在大型系统上的性能不如原始分配器。

无论选择了哪种分配器，编程接口都保持不变。实际上，在低级别，所有三种分配器共享一些公共代码基础：

![](img/00027.jpeg)

我们现在将研究*cache*的物理布局及其控制结构。

# 缓存布局 - 通用

每个缓存都由一个缓存描述符结构`kmem_cache`表示；这个结构包含了缓存的所有关键元数据。它包括一个 slab 描述符列表，每个描述符承载一个页面或一组页面帧。slab 下的页面包含对象或内存块，这些是缓存的分配单元。**slab 描述符**指向页面中包含的对象列表并跟踪它们的状态。根据它承载的对象的状态，一个 slab 可能处于三种可能的状态之一--满的、部分的或空的。当一个 slab 中的所有对象都被使用并且没有剩余的自由对象可供分配时，*slab*被认为是*full*。至少有一个自由对象的 slab 被认为处于*partial*状态，而所有对象都处于*free*状态的 slab 被认为是*empty*。

![](img/00028.jpeg)

这种安排使得对象分配更快，因为分配器例程可以查找*partial* slab 以获取一个自由对象，并在需要时可能转移到*empty* slab。它还有助于通过新的页面帧扩展缓存以容纳更多对象（在需要时），并促进安全和快速的回收（*empty*状态的 slab 可以被回收）。

# Slub 数据结构

在通用级别上查看了缓存的布局和涉及的描述符之后，让我们进一步查看**slub**分配器使用的特定数据结构，并探索空闲列表的管理。一个**slub**在内核头文件`/include/linux/slub-def.h`中定义了它的版本的缓存描述符`struct kmem_cache`：

```
struct kmem_cache {
 struct kmem_cache_cpu __percpu *cpu_slab;
 /* Used for retriving partial slabs etc */
 unsigned long flags;
 unsigned long min_partial;
 int size; /* The size of an object including meta data */
 int object_size; /* The size of an object without meta data */
 int offset; /* Free pointer offset. */
 int cpu_partial; /* Number of per cpu partial objects to keep around */
 struct kmem_cache_order_objects oo;

 /* Allocation and freeing of slabs */
 struct kmem_cache_order_objects max;
 struct kmem_cache_order_objects min;
 gfp_t allocflags; /* gfp flags to use on each alloc */
 int refcount; /* Refcount for slab cache destroy */
 void (*ctor)(void *);
 int inuse; /* Offset to metadata */
 int align; /* Alignment */
 int reserved; /* Reserved bytes at the end of slabs */
 const char *name; /* Name (only for display!) */
 struct list_head list; /* List of slab caches */
 int red_left_pad; /* Left redzone padding size */
 ...
 ...
 ...
 struct kmem_cache_node *node[MAX_NUMNODES];
};
```

`list`元素指的是一个 slab 缓存列表。当分配一个新的 slab 时，它被存储在缓存描述符的列表中，并被认为是*empty*，因为它的所有对象都是*free*并且可用的。在分配对象后，slab 变为*partial*状态。部分 slab 是分配器需要跟踪的唯一类型的 slab，并且在`kmem_cache`结构内部的列表中连接在一起。**SLUB**分配器对已分配所有对象的*full* slabs 或对象都是*free*的*empty* slabs 没有兴趣。**SLUB**通过`struct kmem_cache_node[MAX_NUMNODES]`类型的指针数组来跟踪每个节点的*partial* slabs，这个数组封装了*partial* slabs 的列表。

```
struct kmem_cache_node {
 spinlock_t list_lock;
 ...
 ...
#ifdef CONFIG_SLUB
 unsigned long nr_partial;
 struct list_head partial;
#ifdef CONFIG_SLUB_DEBUG
 atomic_long_t nr_slabs;
 atomic_long_t total_objects;
 struct list_head full;
#endif
#endif 
};
```

slab 中的所有*free*对象形成一个链表；当分配请求到达时，从列表中移除第一个空闲对象，并将其地址返回给调用者。通过链表跟踪空闲对象需要大量的元数据；而传统的**SLAB**分配器在 slab 头部维护了所有 slab 页面的元数据（导致数据对齐问题），**SLUB**通过将更多字段塞入页面描述符结构中，从而消除了 slab 头部的元数据，为 slab 中的页面维护每页元数据。**SLUB**页面描述符中的元数据元素仅在相应页面是 slab 的一部分时才有效。用于 slab 分配的页面已设置`PG_slab`标志。

以下是与 SLUB 相关的页面描述符的字段：

```
struct page {
      ...
      ...
     union {
      pgoff_t index; /* Our offset within mapping. */
      void *freelist; /* sl[aou]b first free object */
   };
     ...
     ...
   struct {
          union {
                  ...
                   struct { /* SLUB */
                          unsigned inuse:16;
                          unsigned objects:15;
                          unsigned frozen:1;
                     };
                   ...
                };
               ...
          };
     ...
     ...
       union {
             ...
             ...
             struct kmem_cache *slab_cache; /* SL[AU]B: Pointer to slab */
         };
    ...
    ...
};
```

`freelist`指针指向列表中的第一个空闲对象。每个空闲对象由一个包含指向列表中下一个空闲对象的指针的元数据区域组成。`index`保存到第一个空闲对象的元数据区域的偏移量（包含指向下一个空闲对象的指针）。最后一个空闲对象的元数据区域将包含下一个空闲对象指针设置为 NULL。`inuse`包含分配对象的总数，`objects`包含对象的总数。`frozen`是一个标志，用作页面锁定：如果页面被 CPU 核心冻结，只有该核心才能从页面中检索空闲对象。`slab_cache`是指向当前使用该页面的 kmem 缓存的指针：

![](img/00029.jpeg)

当分配请求到达时，通过`freelist`指针找到第一个空闲对象，并通过将其地址返回给调用者来从列表中移除它。`inuse`计数器也会递增，以指示分配对象的数量增加。然后，`freelist`指针将更新为列表中下一个空闲对象的地址。

为了实现增强的分配效率，每个 CPU 被分配一个私有的活动 slab 列表，其中包括每种对象类型的部分/空闲 slab 列表。这些 slab 被称为 CPU 本地 slab，并由 struct `kmem_cache_cpu`跟踪：

```
struct kmem_cache_cpu {
     void **freelist; /* Pointer to next available object */
     unsigned long tid; /* Globally unique transaction id */
     struct page *page; /* The slab from which we are allocating */
     struct page *partial; /* Partially allocated frozen slabs */
     #ifdef CONFIG_SLUB_STATS
        unsigned stat[NR_SLUB_STAT_ITEMS];
     #endif
};
```

当分配请求到达时，分配器会采用快速路径，并查看每个 CPU 缓存的`freelist`，然后返回空闲对象。这被称为快速路径，因为分配是通过中断安全的原子指令进行的，不需要锁竞争。当快速路径失败时，分配器会采用慢速路径，依次查看 CPU 缓存的`*page*`和`*partial*`列表。如果找不到空闲对象，分配器会移动到节点的*partial*列表；这个操作需要分配器争夺适当的排他锁。失败时，分配器从伙伴系统获取一个新的 slab。从节点列表获取或从伙伴系统获取新的 slab 都被认为是非常慢的路径，因为这两个操作都不是确定性的。

以下图表描述了 slub 数据结构和空闲列表之间的关系：

![](img/00030.jpeg)

# Vmalloc

页面和 slab 分配器都分配物理连续的内存块，映射到连续的内核地址空间。大多数情况下，内核服务和子系统更喜欢分配物理连续的块，以利用缓存、地址转换和其他与性能相关的好处。尽管如此，对于非常大的块的分配请求可能会因为物理内存的碎片化而失败，而且有一些情况需要分配大块，比如支持动态可加载模块、交换管理操作、大文件缓存等等。

作为解决方案，内核提供了**vmalloc**，这是一种分段内存分配器，通过虚拟连续地址空间将物理分散的内存区域连接起来进行内存分配。内核段内保留了一系列虚拟地址用于 vmalloc 映射，称为 vmalloc 地址空间。通过 vmalloc 接口可以映射的总内存量取决于 vmalloc 地址空间的大小，这由特定于架构的内核宏`VMALLOC_START`和`VMALLOC_END`定义；对于 x86-64 系统，vmalloc 地址空间的总范围达到了惊人的 32 TB。然而，另一方面，这个范围对于大多数 32 位架构来说太小了（只有 120 MB）。最近的内核版本使用 vmalloc 范围来设置虚拟映射的内核栈（仅限 x86-64），这是我们在第一章中讨论过的。

以下是 vmalloc 分配和释放的接口例程：

```
/**
  * vmalloc  -  allocate virtually contiguous memory
  * @size:   -  allocation size
  * Allocate enough pages to cover @size from the page level
  * allocator and map them into contiguous kernel virtual space.
  *
  */
    void *vmalloc(unsigned long size) 
/**
  * vzalloc - allocate virtually contiguous memory with zero fill
1 * @size:  allocation size
  * Allocate enough pages to cover @size from the page level
  * allocator and map them into contiguous kernel virtual space.
  * The memory allocated is set to zero.
  *
  */
 void *vzalloc(unsigned long size) 
/**
  * vmalloc_user - allocate zeroed virtually contiguous memory for userspace
  * @size: allocation size
  * The resulting memory area is zeroed so it can be mapped to userspace
  * without leaking data.
  */
    void *vmalloc_user(unsigned long size) /**
  * vmalloc_node  -  allocate memory on a specific node
  * @size:          allocation size
  * @node:          numa node
  * Allocate enough pages to cover @size from the page level
  * allocator and map them into contiguous kernel virtual space.
  *
  */
    void *vmalloc_node(unsigned long size, int node) /**
  * vfree  -  release memory allocated by vmalloc()
  * @addr:          memory base address
  * Free the virtually continuous memory area starting at @addr, as
  * obtained from vmalloc(), vmalloc_32() or __vmalloc(). If @addr is
  * NULL, no operation is performed.
  */
 void vfree(const void *addr) /**
  * vfree_atomic  -  release memory allocated by vmalloc()
  * @addr:          memory base address
  * This one is just like vfree() but can be called in any atomic context except NMIs.
  */
    void vfree_atomic(const void *addr)
```

大多数内核开发人员避免使用 vmalloc 分配，因为分配开销较大（因为它们不是身份映射的，并且需要特定的页表调整，导致 TLB 刷新）并且在访问操作期间涉及性能惩罚。

# 连续内存分配器（CMA）

尽管存在较大的开销，但虚拟映射的分配在很大程度上解决了大内存分配的问题。然而，有一些情况需要物理连续缓冲区的分配。DMA 传输就是这样一种情况。设备驱动程序经常需要物理连续缓冲区的分配（用于设置 DMA 传输），这是通过之前讨论过的任何一个物理连续分配器来完成的。

然而，处理特定类别设备的驱动程序，如多媒体，经常发现自己在搜索大块连续内存。为了实现这一目标，多年来，这些驱动程序一直通过内核参数`mem`在系统启动时*保留*内存，这允许在驱动程序运行时设置足够的连续内存，并且可以在线性地址空间中*重新映射*。尽管有价值，这种策略也有其局限性：首先，当相应设备未启动访问操作时，这些保留内存暂时未被使用，其次，根据需要支持的设备数量，保留内存的大小可能会大幅增加，这可能会严重影响系统性能，因为物理内存被挤压。

**连续内存分配器**（**CMA**）是一种内核机制，用于有效管理*保留*内存。*CMA*的核心是将*保留*内存引入分配器算法中，这样的内存被称为*CMA 区域。CMA*允许从*CMA 区域*为设备和系统的使用进行分配。这是通过为保留内存中的页面构建页面描述符列表，并将其列入伙伴系统来实现的，这使得可以通过页面分配器为常规需求（内核子系统）和通过 DMA 分配例程为设备驱动程序分配*CMA 页面*。

然而，必须确保 DMA 分配不会因为 CMA 页面用于其他目的而失败，这是通过`migratetype`属性来处理的，我们之前讨论过。CMA 列举的页面被分配给伙伴系统的`MIGRATE_CMA`属性，表示页面是可移动的。在为非 DMA 目的分配内存时，页面分配器只能使用 CMA 页面进行可移动分配（回想一下，这样的分配可以通过`__GFP_MOVABLE`标志进行）。当 DMA 分配请求到达时，内核分配的 CMA 页面会从保留区域中移出（通过页面迁移机制），从而为设备驱动程序的使用提供内存。此外，当为 DMA 分配页面时，它们的`migratetype`从`MIGRATE_CMA`更改为`MIGRATE_ISOLATE`，使它们对伙伴系统不可见。

*CMA 区域*的大小可以在内核构建过程中通过其配置界面进行选择；可选地，也可以通过内核参数`cma=`进行传递。

# 摘要

我们已经穿越了 Linux 内核最关键的一个方面，理解了内存表示和分配的各种微妙之处。通过理解这个子系统，我们也简洁地捕捉到了内核的设计才能和实现效率，更重要的是理解了内核在容纳更精细和更新的启发式和机制以持续增强方面的动态性。除了内存管理的具体细节，我们还评估了内核在最大化资源利用方面的效率，引领了所有经典的代码重用机制和模块化代码结构。

尽管内存管理的具体细节可能会根据底层架构而有所不同，但设计和实现风格的一般性大部分仍然保持一致，以实现代码稳定性和对变化的敏感性。

在下一章中，我们将进一步探讨内核的另一个基本抽象：*文件*。我们将浏览文件 I/O 并探索其架构和实现细节。
