# 第十一章：模块管理

内核模块（也称为 LKM）由于易用性而强调了内核服务的发展。本章的重点将是了解内核如何无缝地促进整个过程，使模块的加载和卸载变得动态和简单，我们将深入了解模块管理中涉及的所有核心概念、函数和重要数据结构。我们假设读者熟悉模块的基本用法。

在本章中，我们将涵盖以下主题：

+   内核模块的关键元素

+   模块布局

+   模块加载和卸载接口

+   关键数据结构

# 内核模块

内核模块是一种简单而有效的机制，可以在不重建整个内核的情况下扩展运行系统的功能，它们对于引入动态性和可扩展性到 Linux 操作系统至关重要。内核模块不仅满足了内核的可扩展性，还引入了以下功能：

+   允许内核仅保留必要的功能，从而提高容量利用率

+   允许专有/非 GPL 兼容服务加载和卸载

+   内核可扩展性的底线特性

# LKM 的元素

每个模块对象都包括*init（构造函数）*和*exit（析构函数）*例程。当模块部署到内核地址空间时，将调用*init*例程，而在模块被移除时将调用*exit*例程。正如名称本身所暗示的那样，*init*例程通常被编程为执行设置模块主体所必需的操作和动作，例如注册到特定的内核子系统或分配对加载的功能至关重要的资源。但是，*init*和*exit*例程中编程的特定操作取决于模块的设计目的以及它为内核带来的功能。以下代码摘录显示了*init*和*exit*例程的模板：

```
int init_module(void)
{
  /* perform required setup and registration ops */
    ...
    ...
    return 0;
}

void cleanup_module(void)
{
   /* perform required cleanup operations */
   ...
   ...
}
```

注意，*init*例程返回一个整数——如果模块已提交到内核地址空间，则返回零，如果失败则返回负数。这还为程序员提供了灵活性，只有在成功注册到所需子系统时才能提交模块。

*init*和*exit*例程的默认名称分别为`init_module()`和`cleanup_module()`。模块可以选择更改*init*和*exit*例程的名称以提高代码可读性。但是，它们必须使用`module_init`和`module_exit`宏进行声明：

```
int myinit(void)
{
        ...
        ...
        return 0;
}

void myexit(void)
{
        ...
        ...
}

module_init(myinit);
module_exit(myexit);
```

注释宏是模块代码的另一个关键元素。这些宏用于提供模块的用法、许可和作者信息。这很重要，因为模块来自各种供应商：

+   `MODULE_DESCRIPTION()`: 该宏用于指定模块的一般描述

+   `MODULE_AUTHOR()`: 用于提供作者信息

+   `MODULE_LICENSE()`: 用于指定模块中代码的合法许可证

通过这些宏指定的所有信息都保留在模块二进制文件中，并且可以通过名为*modinfo*的实用程序由用户访问。`MODULE_LICENSE()`是模块必须提到的唯一强制性宏。这非常方便，因为它通知用户模块中的专有代码容易受到调试和支持问题的影响（内核社区很可能会忽略专有模块引起的问题）。

模块可用的另一个有用功能是使用模块参数动态初始化模块数据变量。这允许在模块中声明的数据变量在模块部署期间或模块在内存中*live*时（通过 sysfs 接口）进行初始化。这可以通过通过适当的`module_param()`宏族（在内核头文件`<linux/moduleparam.h>`中找到）将选定的变量设置为模块参数来实现。在模块部署期间传递给模块参数的值在调用*init*函数之前进行初始化。

模块中的代码可以根据需要访问全局内核函数和数据。这使得模块的代码可以利用现有的内核功能。通过这样的函数调用，模块可以执行所需的操作，例如将消息打印到内核日志缓冲区，分配和释放内存，获取和释放排他锁，以及向适当的子系统注册和注销模块代码。

类似地，一个模块也可以将其符号导出到内核的全局符号表中，然后可以从其他模块中的代码中访问这些符号。这通过将内核服务组织在一组模块中，而不是将整个服务实现为单个 LKM，从而促进了内核服务的细粒度设计和实现。相关服务的堆叠会导致模块依赖，例如：如果模块 A 正在使用模块 B 的符号，则 A 依赖于 B，在这种情况下，必须在加载模块 A 之前加载模块 B，并且在卸载模块 A 之前不能卸载模块 B。

# LKM 的二进制布局

模块是使用 kbuild makefile 构建的；一旦构建过程完成，将生成一个带有*.ko*（内核对象）扩展名的 ELF 二进制文件。模块 ELF 二进制文件经过适当的调整，以添加新的部分，使其与其他 ELF 二进制文件区分开，并存储与模块相关的元数据。以下是内核模块中的部分：

| `.gnu.linkonce.this_module` | 模块结构 |
| --- | --- |
| `.modinfo` | 有关模块的信息（许可证等） |
| `__versions` | 编译时模块依赖的符号的预期版本 |
| `__ksymtab*` | 由此模块导出的符号表 |
| `__kcrctab*` | 由此模块导出的符号版本表 |
| `.init` | 初始化时使用的部分 |
| `.text, .data 等` | 代码和数据部分 |

# 加载和卸载操作

模块可以通过一个名为*modutils*的应用程序包中的特殊工具部署，其中*insmod*和*rmmod*被广泛使用。*insmod*用于将模块部署到内核地址空间，*rmmod*用于卸载活动模块。这些工具通过调用适当的系统调用来启动加载/卸载操作：

```
int finit_module(int fd, const char *param_values, int flags);
int delete_module(const char *name, int flags);
```

在这里，`finit_module()`（由`insmod`）被调用，带有指定模块二进制文件（.ko）的文件描述符和其他相关参数。此函数通过调用底层系统调用进入内核模式：

```
SYSCALL_DEFINE3(finit_module, int, fd, const char __user *, uargs, int, flags)
{
        struct load_info info = { };
        loff_t size;
        void *hdr;
        int err;

        err = may_init_module();
        if (err)
                return err;

        pr_debug("finit_module: fd=%d, uargs=%p, flags=%i\n", fd, uargs, flags);

        if (flags & ~(MODULE_INIT_IGNORE_MODVERSIONS
                      |MODULE_INIT_IGNORE_VERMAGIC))
                return -EINVAL;

        err = kernel_read_file_from_fd(fd, &hdr, &size, INT_MAX,
                                       READING_MODULE);
        if (err)
                return err;
        info.hdr = hdr;
        info.len = size;

        return load_module(&info, uargs, flags);
}
```

在这里，`may_init_module()`被调用来验证调用上下文的`CAP_SYS_MODULE`特权；此函数在失败时返回负数，在成功时返回零。如果调用者具有所需的特权，则通过使用`kernel_read_file_from_fd()`例程访问指定的模块映像，该例程返回模块映像的地址，然后将其填充到`struct load_info`的实例中。最后，通过将`load_info`的实例地址和从`finit_module()`调用传递下来的其他用户参数，调用`load_module()`核心内核例程：

```
static int load_module(struct load_info *info, const char __user *uargs,int flags)
{
        struct module *mod;
        long err;
        char *after_dashes;

        err = module_sig_check(info, flags);
        if (err)
                goto free_copy;

        err = elf_header_check(info);
        if (err)
                goto free_copy;

        /* Figure out module layout, and allocate all the memory. */
        mod = layout_and_allocate(info, flags);
        if (IS_ERR(mod)) {
                err = PTR_ERR(mod);
                goto free_copy;
        }

        ....
        ....
        ....

}
```

在这里，`load_module（）`是一个核心内核例程，它尝试将模块映像链接到内核地址空间。此函数启动一系列健全性检查，并最终通过将模块参数初始化为调用者提供的值并调用模块的*init*函数来提交模块。以下步骤详细说明了这些操作，以及调用的相关辅助函数的名称：

+   检查签名（`module_sig_check（）`）

+   检查 ELF 头（`elf_header_check（）`）

+   检查模块布局并分配必要的内存（`layout_and_allocate（）`）

+   将模块附加到模块列表（`add_unformed_module（）`）

+   为模块分配每个 CPU 区域（`percpu_modalloc（）`）

+   由于模块位于最终位置，需要找到可选部分（`find_module_sections（）`）

+   检查模块许可证和版本（`check_module_license_and_versions（）`）

+   解析符号（`simplify_symbols（）`）

+   根据 args 列表中传递的值设置模块参数

+   检查符号的重复（`complete_formation（）`）

+   设置 sysfs（`mod_sysfs_setup（）`）

+   释放*load_info*结构中的副本（`free_copy（）`）

+   调用模块的*init*函数（`do_init_module（）`）

卸载过程与加载过程非常相似；唯一不同的是，有一些健全性检查，以确保安全地从内核中移除模块，而不影响系统稳定性。模块的卸载是通过调用*rmmod*实用程序来初始化的，该实用程序调用`delete_module（）`例程，该例程进入底层系统调用：

```
SYSCALL_DEFINE2(delete_module, const char __user *, name_user,
                unsigned int, flags)
{
        struct module *mod;
        char name[MODULE_NAME_LEN];
        int ret, forced = 0;

        if (!capable(CAP_SYS_MODULE) || modules_disabled)
                return -EPERM;

        if (strncpy_from_user(name, name_user, MODULE_NAME_LEN-1) < 0)
                return -EFAULT;
        name[MODULE_NAME_LEN-1] = '\0';

        audit_log_kern_module(name);

        if (mutex_lock_interruptible(&module_mutex) != 0)
                return -EINTR;

        mod = find_module(name);
        if (!mod) {
                ret = -ENOENT;
                goto out;
        }

        if (!list_empty(&mod->source_list)) {
                /* Other modules depend on us: get rid of them first. */
                ret = -EWOULDBLOCK;
                goto out;
        }

        /* Doing init or already dying? */
        if (mod->state != MODULE_STATE_LIVE) {
                /* FIXME: if (force), slam module count damn the torpedoes */
                pr_debug("%s already dying\n", mod->name);
                ret = -EBUSY;
                goto out;
        }

        /* If it has an init func, it must have an exit func to unload */
        if (mod->init && !mod->exit) {
                forced = try_force_unload(flags);
                if (!forced) {
                        /* This module can't be removed */
                        ret = -EBUSY;
                        goto out;
                }
        }

        /* Stop the machine so refcounts can't move and disable module. */
        ret = try_stop_module(mod, flags, &forced);
        if (ret != 0)
                goto out;

        mutex_unlock(&module_mutex);
        /* Final destruction now no one is using it. */
        if (mod->exit != NULL)
                mod->exit();
        blocking_notifier_call_chain(&module_notify_list,
                                     MODULE_STATE_GOING, mod);
        klp_module_going(mod);
        ftrace_release_mod(mod);

        async_synchronize_full();

        /* Store the name of the last unloaded module for diagnostic purposes */
        strlcpy(last_unloaded_module, mod->name, sizeof(last_unloaded_module));

        free_module(mod);
        return 0;
out:
        mutex_unlock(&module_mutex);
        return ret;
}
```

在调用时，系统调用会检查调用者是否具有必要的权限，然后检查是否存在任何模块依赖项。如果没有，模块就可以被移除（否则，将返回错误）。之后，验证模块状态（*live*）。最后，调用模块的退出例程，最后调用`free_module（）`例程：

```
/* Free a module, remove from lists, etc. */
static void free_module(struct module *mod)
{
        trace_module_free(mod);

        mod_sysfs_teardown(mod);

        /* We leave it in list to prevent duplicate loads, but make sure
        * that no one uses it while it's being deconstructed. */
        mutex_lock(&module_mutex);
        mod->state = MODULE_STATE_UNFORMED;
        mutex_unlock(&module_mutex);

        /* Remove dynamic debug info */
        ddebug_remove_module(mod->name);

        /* Arch-specific cleanup. */
        module_arch_cleanup(mod);

        /* Module unload stuff */
        module_unload_free(mod);

        /* Free any allocated parameters. */
        destroy_params(mod->kp, mod->num_kp);

        if (is_livepatch_module(mod))
                free_module_elf(mod);

        /* Now we can delete it from the lists */
        mutex_lock(&module_mutex);
        /* Unlink carefully: kallsyms could be walking list. */
        list_del_rcu(&mod->list);
        mod_tree_remove(mod);
        /* Remove this module from bug list, this uses list_del_rcu */
        module_bug_cleanup(mod);
        /* Wait for RCU-sched synchronizing before releasing mod->list and buglist. */
        synchronize_sched();
        mutex_unlock(&module_mutex);

        /* This may be empty, but that's OK */
        disable_ro_nx(&mod->init_layout);
        module_arch_freeing_init(mod);
        module_memfree(mod->init_layout.base);
        kfree(mod->args);
        percpu_modfree(mod);

        /* Free lock-classes; relies on the preceding sync_rcu(). */
        lockdep_free_key_range(mod->core_layout.base, mod->core_layout.size);

        /* Finally, free the core (containing the module structure) */
        disable_ro_nx(&mod->core_layout);
        module_memfree(mod->core_layout.base);

#ifdef CONFIG_MPU
        update_protections(current->mm);
#endif
}
```

此调用将模块从加载期间放置的各种列表中删除（sysfs、模块列表等），以启动清理。调用特定于体系结构的清理例程（可以在`</linux/arch/<arch>/kernel/module.c>`*）*中找到。对所有依赖模块进行迭代，并从它们的列表中删除模块。一旦清理结束，将释放为模块分配的所有资源和内存。

# 模块数据结构

内核中部署的每个模块通常通过称为`struct module`的描述符表示。内核维护着模块实例的列表，每个实例代表内存中的特定模块：

```
struct module {
        enum module_state state;

        /* Member of list of modules */
        struct list_head list;

        /* Unique handle for this module */
        char name[MODULE_NAME_LEN];

        /* Sysfs stuff. */
        struct module_kobject mkobj;
        struct module_attribute *modinfo_attrs;
        const char *version;
        const char *srcversion;
        struct kobject *holders_dir;

        /* Exported symbols */
        const struct kernel_symbol *syms;
        const s32 *crcs;
        unsigned int num_syms;

        /* Kernel parameters. */
#ifdef CONFIG_SYSFS
        struct mutex param_lock;
#endif
        struct kernel_param *kp;
        unsigned int num_kp;

        /* GPL-only exported symbols. */
        unsigned int num_gpl_syms;
        const struct kernel_symbol *gpl_syms;
        const s32 *gpl_crcs;

#ifdef CONFIG_UNUSED_SYMBOLS
        /* unused exported symbols. */
        const struct kernel_symbol *unused_syms;
        const s32 *unused_crcs;
        unsigned int num_unused_syms;

        /* GPL-only, unused exported symbols. */
        unsigned int num_unused_gpl_syms;
        const struct kernel_symbol *unused_gpl_syms;
        const s32 *unused_gpl_crcs;
#endif

#ifdef CONFIG_MODULE_SIG
        /* Signature was verified. */
        bool sig_ok;
#endif

        bool async_probe_requested;

        /* symbols that will be GPL-only in the near future. */
        const struct kernel_symbol *gpl_future_syms;
        const s32 *gpl_future_crcs;
        unsigned int num_gpl_future_syms;

        /* Exception table */
        unsigned int num_exentries;
        struct exception_table_entry *extable;

        /* Startup function. */
        int (*init)(void);

        /* Core layout: rbtree is accessed frequently, so keep together. */
        struct module_layout core_layout __module_layout_align;
        struct module_layout init_layout;

        /* Arch-specific module values */
        struct mod_arch_specific arch;

        unsigned long taints;     /* same bits as kernel:taint_flags */

#ifdef CONFIG_GENERIC_BUG
        /* Support for BUG */
        unsigned num_bugs;
        struct list_head bug_list;
        struct bug_entry *bug_table;
#endif

#ifdef CONFIG_KALLSYMS
        /* Protected by RCU and/or module_mutex: use rcu_dereference() */
        struct mod_kallsyms *kallsyms;
        struct mod_kallsyms core_kallsyms;

        /* Section attributes */
        struct module_sect_attrs *sect_attrs;

        /* Notes attributes */
        struct module_notes_attrs *notes_attrs;
#endif

        /* The command line arguments (may be mangled).  People like
          keeping pointers to this stuff */
        char *args;

#ifdef CONFIG_SMP
        /* Per-cpu data. */
        void __percpu *percpu;
        unsigned int percpu_size;
#endif

#ifdef CONFIG_TRACEPOINTS
        unsigned int num_tracepoints;
        struct tracepoint * const *tracepoints_ptrs;
#endif
#ifdef HAVE_JUMP_LABEL
        struct jump_entry *jump_entries;
        unsigned int num_jump_entries;
#endif
#ifdef CONFIG_TRACING
        unsigned int num_trace_bprintk_fmt;
        const char **trace_bprintk_fmt_start;
#endif
#ifdef CONFIG_EVENT_TRACING
        struct trace_event_call **trace_events;
        unsigned int num_trace_events;
        struct trace_enum_map **trace_enums;
        unsigned int num_trace_enums;
#endif
#ifdef CONFIG_FTRACE_MCOUNT_RECORD
        unsigned int num_ftrace_callsites;
        unsigned long *ftrace_callsites;
#endif

#ifdef CONFIG_LIVEPATCH
        bool klp; /* Is this a livepatch module? */
        bool klp_alive;

        /* Elf information */
        struct klp_modinfo *klp_info;
#endif

#ifdef CONFIG_MODULE_UNLOAD
        /* What modules depend on me? */
        struct list_head source_list;
        /* What modules do I depend on? */
        struct list_head target_list;

        /* Destruction function. */
        void (*exit)(void);

        atomic_t refcnt;
#endif

#ifdef CONFIG_CONSTRUCTORS
        /* Constructor functions. */
        ctor_fn_t *ctors;
        unsigned int num_ctors;
#endif
} ____cacheline_aligned;
```

现在让我们看一下此结构的一些关键字段：

+   `list`：这是一个双向链表，其中包含内核中加载的所有模块。

+   `name`：指定模块的名称。这必须是一个唯一的名称，因为模块是通过此名称引用的。

+   `state`：表示模块的当前状态。模块可以处于`<linux/module.h>`下指定的任一状态中：

```
enum module_state {
        MODULE_STATE_LIVE,        /* Normal state. */
        MODULE_STATE_COMING,      /* Full formed, running module_init. */
        MODULE_STATE_GOING,       /* Going away. */
        MODULE_STATE_UNFORMED,    /* Still setting it up. */
};
```

在加载或卸载模块时，了解其当前状态很重要；例如，如果其状态指定模块已经存在，则无需插入现有模块。

`syms, crc 和 num_syms`：用于管理模块代码导出的符号。

`init`：这是指向在模块初始化时调用的函数的指针。

`arch`：表示特定于体系结构的结构，应填充体系结构特定数据，以便模块运行。但是，由于大多数体系结构不需要任何额外的信息，因此此结构大多数情况下保持为空。

`taints`：如果模块使内核受到污染，则使用此选项。这可能意味着内核怀疑模块会执行一些有害的操作或者是非 GPL 兼容的代码。

`percpu`：指向属于模块的每个 CPU 数据。它在模块加载时初始化。

`source_list 和 target_list`：这包含了模块依赖的详细信息。

`exit`：这只是 init 的相反。它指向调用模块清理过程的函数。它释放模块持有的内存并执行其他清理特定任务。

# 内存布局

模块的内存布局通过*<linux/module.h>*中定义的`struct module_layout`对象显示。

```
struct module_layout {
        /* The actual code + data. */
        void *base;
        /* Total size. */
        unsigned int size;
        /* The size of the executable code.  */
        unsigned int text_size;
        /* Size of RO section of the module (text+rodata) */
        unsigned int ro_size;

#ifdef CONFIG_MODULES_TREE_LOOKUP
        struct mod_tree_node mtn;
#endif
};
```

# 总结

在这一章中，我们简要介绍了模块的所有核心元素，其含义和管理细节。我们的目标是为您提供一个快速和全面的视角，了解内核如何通过模块实现其可扩展性。您还了解了促进模块管理的核心数据结构。内核在这个动态环境中保持安全和稳定的努力也是一个显著的特点。

我真诚地希望这本书能成为您去实验 Linux 内核的手段！
