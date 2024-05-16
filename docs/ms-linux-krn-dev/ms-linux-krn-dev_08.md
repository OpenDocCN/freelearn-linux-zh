# 第八章：内核同步和锁定

内核地址空间由所有用户模式进程共享，这使得可以并发访问内核服务和数据结构。为了系统的可靠运行，内核服务必须实现为可重入的。访问全局数据结构的内核代码路径需要同步，以确保共享数据的一致性和有效性。在本章中，我们将详细介绍内核程序员用于同步内核代码路径和保护共享数据免受并发访问的各种资源。

本章将涵盖以下主题：

+   原子操作

+   自旋锁

+   标准互斥锁

+   等待/伤害互斥锁

+   信号量

+   序列锁

+   完成

# 原子操作

计算操作被认为是**原子的**，如果它在系统的其余部分看起来是瞬间发生的。原子性保证了操作的不可分割和不可中断的执行。大多数 CPU 指令集架构定义了可以在内存位置上执行原子读-修改-写操作的指令操作码。这些操作具有成功或失败的定义，即它们要么成功地改变内存位置的状态，要么失败而没有明显的影响。这些操作对于在多线程场景中原子地操作共享数据非常有用。它们还用作实现排他锁的基础构建块，这些锁用于保护共享内存位置免受并行代码路径的并发访问。

Linux 内核代码使用原子操作来处理各种用例，例如共享数据结构中的引用计数器（用于跟踪对各种内核数据结构的并发访问），等待-通知标志，以及为特定代码路径启用数据结构的独占所有权。为了确保直接处理原子操作的内核服务的可移植性，内核提供了丰富的与体系结构无关的接口宏和内联函数库，这些函数库用作处理器相关的原子指令的抽象。这些中立接口下的相关 CPU 特定原子指令由内核代码的体系结构分支实现。

# 原子整数操作

通用原子操作接口包括对整数和位操作的支持。整数操作被实现为操作特殊的内核定义类型，称为`atomic_t`（32 位整数）和`atomic64_t`（64 位整数）。这些类型的定义可以在通用内核头文件`<linux/types.h>`中找到：

```
typedef struct {
        int counter;
} atomic_t;

#ifdef CONFIG_64BIT
typedef struct {
        long counter;
} atomic64_t;
#endif
```

该实现提供了两组整数操作；一组适用于 32 位，另一组适用于 64 位原子变量。这些接口操作被实现为一组宏和内联函数。以下是适用于`atomic_t`类型变量的操作的摘要列表：

| **接口宏/内联函数** | **描述** |
| --- | --- |
| `ATOMIC_INIT(i)` | 用于初始化原子计数器的宏 |
| `atomic_read(v)` | 读取原子计数器`v`的值 |
| `atomic_set(v, i)` | 原子性地将计数器`v`设置为`i`中指定的值 |
| `atomic_add(int i, atomic_t *v)` | 原子性地将`i`添加到计数器`v`中 |
| `atomic_sub(int i, atomic_t *v)` | 原子性地从计数器`v`中减去`i` |
| `atomic_inc(atomic_t *v)` | 原子性地增加计数器`v` |
| `atomic_dec(atomic_t *v)` | 原子性地减少计数器`v` |

以下是执行相关**读-修改-写**（**RMW**）操作并返回结果的函数列表（即，它们返回修改后写入内存地址的值）：

| **操作** | **描述** |
| --- | --- |
| `bool atomic_sub_and_test(int i, atomic_t *v)` | 原子性地从`v`中减去`i`，如果结果为零则返回`true`，否则返回`false` |
| `bool atomic_dec_and_test(atomic_t *v)` | 原子性地将`v`减 1，并在结果为 0 时返回`true`，否则对所有其他情况返回`false` |
| `bool atomic_inc_and_test(atomic_t *v)` | 原子地将`i`添加到`v`，如果结果为 0 则返回`true`，否则返回`false` |
| `bool atomic_add_negative(int i, atomic_t *v)` | 原子地将`i`添加到`v`，如果结果为负数则返回`true`，如果结果大于或等于零则返回`false` |
| `int atomic_add_return(int i, atomic_t *v)` | 原子地将`i`添加到`v`，并返回结果 |
| `int atomic_sub_return(int i, atomic_t *v)` | 原子地从`v`中减去`i`，并返回结果 |
| `int atomic_fetch_add(int i, atomic_t *v)` | 原子地将`i`添加到`v`，并返回`v`中的加法前值 |
| `int atomic_fetch_sub(int i, atomic_t *v)` | 原子地从`v`中减去`i`，并返回`v`中的减法前值 |
| `int atomic_cmpxchg(atomic_t *v, int old,` int new) | 读取位置`v`处的值，并检查它是否等于`old`；如果为`true`，则交换`v`处的值与`*new*`，并始终返回在`v`处读取的值 |
| `int atomic_xchg(atomic_t *v, int new)` | 用`new`交换存储在位置`v`处的旧值，并返回旧值`v` |

对于所有这些操作，都存在用于`atomic64_t`的 64 位变体；这些函数的命名约定为`atomic64_*()`

# 原子位操作

内核提供的通用原子操作接口还包括位操作。与整数操作不同，整数操作被实现为在`atomic(64)_t`类型上操作，这些位操作可以应用于任何内存位置。这些操作的参数是位的位置或位数，以及一个具有有效地址的指针。32 位机器的位范围为 0-31，64 位机器的位范围为 0-63。以下是可用的位操作的摘要列表：

| **操作接口** | **描述** |
| --- | --- |
| `set_bit(int nr, volatile unsigned long *addr)` | 在从`addr`开始的位置上原子设置位`nr` |
| `clear_bit(int nr, volatile unsigned long *addr)` | 在从`addr`开始的位置上原子清除位`nr` |
| `change_bit(int nr, volatile unsigned long *addr)` | 在从`addr`开始的位置上原子翻转位`nr` |
| `int test_and_set_bit(int nr, volatile unsigned long *addr)` | 在从`addr`开始的位置上原子设置位`nr`，并返回`nr^(th)`位的旧值 |
| `int test_and_clear_bit(int nr, volatile unsigned long *addr)` | 在从`addr`开始的位置上原子清除位`nr`，并返回`nr``^(th)`位的旧值 |
| `int test_and_change_bit(int nr, volatile unsigned long *addr)` | 在从`addr`开始的位置上原子翻转位`nr`，并返回`nr^(th)`位的旧值 |

对于所有具有返回类型的操作，返回的值是在指定修改发生之前从内存地址中读取的位的旧状态。这些操作也存在非原子版本；它们对于可能需要位操作的情况是高效且有用的，这些情况是从互斥临界块中的代码语句发起的。这些在内核头文件`<linux/bitops/non-atomic.h>`中声明。

# 引入排他锁

硬件特定的原子指令只能操作 CPU 字和双字大小的数据；它们不能直接应用于自定义大小的共享数据结构。对于大多数多线程场景，通常可以观察到共享数据是自定义大小的，例如，一个具有*n*个不同类型元素的结构。访问这些数据的并发代码路径通常包括一堆指令，这些指令被编程为访问和操作共享数据；这样的访问操作必须被*原子地*执行，以防止竞争。为了确保这些代码块的原子性，使用了互斥锁。所有多线程环境都提供了基于排他协议的互斥锁的实现。这些锁定实现是建立在硬件特定的原子指令之上的。

Linux 内核实现了标准排斥机制的操作接口，如互斥和读写排斥。它还包含对各种其他当代轻量级和无锁同步机制的支持。大多数内核数据结构和其他共享数据元素，如共享缓冲区和设备寄存器，都通过内核提供的适当排斥锁接口受到并发访问的保护。在本节中，我们将探讨可用的排斥机制及其实现细节。

# 自旋锁

**自旋锁**是大多数并发编程环境中广泛实现的最简单和轻量级的互斥机制之一。自旋锁实现定义了一个锁结构和操作，用于操作锁结构。锁结构主要包含原子锁计数器等元素，操作接口包括：

+   一个**初始化例程**，用于将自旋锁实例初始化为默认（解锁）状态

+   一个**锁例程**，通过原子地改变锁计数器的状态来尝试获取自旋锁

+   一个**解锁例程**，通过将计数器改变为解锁状态来释放自旋锁

当调用者尝试在锁定时（或被另一个上下文持有）获取自旋锁时，锁定函数会迭代地轮询或自旋直到可用，导致调用者上下文占用 CPU 直到获取锁。正是由于这个事实，这种排斥机制被恰当地命名为自旋锁。因此建议确保关键部分内的代码是原子的或非阻塞的，以便锁定可以持续一个短暂的、确定的时间，因为显然持有自旋锁很长时间可能会造成灾难。

正如讨论的那样，自旋锁是围绕处理器特定的原子操作构建的；内核的架构分支实现了核心自旋锁操作（汇编编程）。内核通过一个通用的平台中立接口包装了架构特定的实现，该接口可以直接被内核服务使用；这使得使用自旋锁保护共享资源的服务代码具有可移植性。

通用自旋锁接口可以在内核头文件 `<linux/spinlock.h>` 中找到，而特定架构的定义是 `<asm/spinlock.h>` 的一部分。通用接口提供了一系列针对特定用例实现的 `lock()` 和 `unlock()` 操作。我们将在接下来的章节中讨论这些接口中的每一个；现在，让我们从接口提供的标准和最基本的 `lock()` 和 `unlock()` 操作变体开始我们的讨论。以下代码示例展示了基本自旋锁接口的使用：

```
DEFINE_SPINLOCK(s_lock);
spin_lock(&s_lock);
/* critical region ... */
spin_unlock(&s_lock);
```

让我们来看看这些函数的实现细节：

```
static __always_inline void spin_lock(spinlock_t *lock)
{
        raw_spin_lock(&lock->rlock);
}

...
...

static __always_inline void spin_unlock(spinlock_t *lock)
{
        raw_spin_unlock(&lock->rlock);
}
```

内核代码实现了两种自旋锁操作的变体；一种适用于 SMP 平台，另一种适用于单处理器平台。自旋锁数据结构和与架构和构建类型（SMP 和 UP）相关的操作在内核源树的各个头文件中定义。让我们熟悉一下这些头文件的作用和重要性：

`<include/linux/spinlock.h>` 包含了通用的自旋锁/rwlock 声明。

以下头文件与 SMP 平台构建相关：

+   `<asm/spinlock_types.h>` 包含了 `arch_spinlock_t/arch_rwlock_t` 和初始化程序

+   `<linux/spinlock_types.h>` 定义了通用类型和初始化程序

+   `<asm/spinlock.h>` 包含了 `arch_spin_*()` 和类似的低级操作实现

+   `<linux/spinlock_api_smp.h>` 包含了 `_spin_*()` API 的原型

+   `<linux/spinlock.h>` 构建了最终的 `spin_*()` API

以下头文件与单处理器（UP）平台构建相关：

+   `<linux/spinlock_type_up.h>` 包含了通用的、简化的 UP 自旋锁类型

+   `<linux/spinlock_types.h>` 定义了通用类型和初始化程序

+   `<linux/spinlock_up.h>`包含了`arch_spin_*()`和 UP 版本的类似构建（在非调试、非抢占构建上是 NOP）

+   `<linux/spinlock_api_up.h>`构建了`_spin_*()`API

+   `<linux/spinlock.h>`构建了最终的`spin_*()`APIs

通用内核头文件`<linux/spinlock.h>`包含一个条件指令，以决定拉取适当的（SMP 或 UP）API。

```
/*
 * Pull the _spin_*()/_read_*()/_write_*() functions/declarations:
 */
#if defined(CONFIG_SMP) || defined(CONFIG_DEBUG_SPINLOCK)
# include <linux/spinlock_api_smp.h>
#else
# include <linux/spinlock_api_up.h>
#endif
```

`raw_spin_lock()`和`raw_spin_unlock()`宏会根据构建配置中选择的平台类型（SMP 或 UP）动态扩展为适当版本的自旋锁操作。对于 SMP 平台，`raw_spin_lock()`会扩展为内核源文件`kernel/locking/spinlock.c`中实现的`__raw_spin_lock()`操作。以下是使用宏定义的锁定操作代码：

```
/*
 * We build the __lock_function inlines here. They are too large for
 * inlining all over the place, but here is only one user per function
 * which embeds them into the calling _lock_function below.
 *
 * This could be a long-held lock. We both prepare to spin for a long
 * time (making _this_ CPU preemptable if possible), and we also signal
 * towards that other CPU that it should break the lock ASAP.
 */

#define BUILD_LOCK_OPS(op, locktype)                                    \
void __lockfunc __raw_##op##_lock(locktype##_t *lock)                   \
{                                                                       \
        for (;;) {                                                      \
                preempt_disable();                                      \
                if (likely(do_raw_##op##_trylock(lock)))                \
                        break;                                          \
                preempt_enable();                                       \
                                                                        \
                if (!(lock)->break_lock)                                \
                        (lock)->break_lock = 1;                         \
                while (!raw_##op##_can_lock(lock) && (lock)->break_lock)\
                        arch_##op##_relax(&lock->raw_lock);             \
        }                                                               \
        (lock)->break_lock = 0;                                         \
} 
```

这个例程由嵌套的循环结构组成，一个外部`for`循环结构和一个内部`while`循环，它会一直旋转，直到指定的条件满足为止。外部循环的第一个代码块通过调用特定于体系结构的`##_trylock()`例程来原子地尝试获取锁。请注意，此函数在本地处理器上禁用内核抢占时被调用。如果成功获取锁，则跳出循环结构，并且调用返回时关闭了抢占。这确保了持有锁的调用者上下文在执行临界区时不可抢占。这种方法还确保了在当前所有者释放锁之前，没有其他上下文可以在本地 CPU 上争夺相同的锁。

然而，如果它未能获取锁，通过`preempt_enable()`调用启用了抢占，并且调用者上下文进入内部循环。这个循环是通过一个条件`while`实现的，它会一直旋转，直到发现锁可用为止。循环的每次迭代都会检查锁，并且当它检测到锁还不可用时，会调用一个特定于体系结构的放松例程（执行特定于 CPU 的 nop 指令），然后再次旋转以检查锁。请记住，在此期间抢占是启用的；这确保了调用者上下文是可抢占的，并且不会长时间占用 CPU，尤其是在锁高度争用的情况下可能发生。这也允许同一 CPU 上调度的两个或更多线程争夺相同的锁，可能通过相互抢占来实现。

当旋转上下文检测到锁可用时，它会跳出`while`循环，导致调用者迭代回外部循环（`for`循环）的开始处，再次尝试通过`##_trylock()`来抓取锁，同时禁用抢占：

```
/*
 * In the UP-nondebug case there's no real locking going on, so the
 * only thing we have to do is to keep the preempt counts and irq
 * flags straight, to suppress compiler warnings of unused lock
 * variables, and to add the proper checker annotations:
 */
#define ___LOCK(lock) \
  do { __acquire(lock); (void)(lock); } while (0)

#define __LOCK(lock) \
  do { preempt_disable(); ___LOCK(lock); } while (0)

#define _raw_spin_lock(lock) __LOCK(lock)
```

与 SMP 变体不同，UP 平台的自旋锁实现非常简单；实际上，锁例程只是禁用内核抢占并将调用者放入临界区。这是因为在暂停抢占的情况下，没有其他上下文可能会争夺锁。

# 备用自旋锁 API

到目前为止我们讨论的标准自旋锁操作适用于仅从进程上下文内核路径访问的共享资源的保护。然而，可能存在一些场景，其中特定的共享资源或数据可能会从内核服务的进程上下文和中断上下文代码中访问。例如，考虑一个设备驱动程序服务，可能包含进程上下文和中断上下文例程，都编程来访问共享的驱动程序缓冲区以执行适当的 I/O 操作。

假设使用自旋锁来保护驱动程序的共享资源免受并发访问，并且驱动程序服务的所有例程（包括进程和中断上下文）都使用标准的`spin_lock()`和`spin_unlock()`操作编程了适当的临界区。这种策略将通过强制排斥来确保共享资源的保护，但可能会导致 CPU 在随机时间出现*硬锁定条件*，因为中断路径代码在同一 CPU 上争夺*锁*。为了进一步理解这一点，让我们假设以下事件按相同顺序发生：

1.  驱动程序的进程上下文例程获取*锁（*使用标准的`spin_lock()`调用*）。

1.  关键部分正在执行时，发生中断并被驱动到本地 CPU，导致进程上下文例程被抢占并让出 CPU 给中断处理程序。

1.  驱动程序的中断上下文路径（ISR）开始并尝试获取*锁（*使用标准的`spin_lock()`调用*），*然后开始自旋等待*锁*可用。

在 ISR 的持续时间内，进程上下文被抢占并且永远无法恢复执行，导致*锁*永远无法释放，并且 CPU 被一个永远不会放弃的自旋中断处理程序硬锁定。

为了防止这种情况发生，进程上下文代码需要在获取*锁*时禁用当前处理器上的中断。这将确保中断在临界区和锁释放*之前*永远无法抢占当前上下文。请注意，中断仍然可能发生，但会路由到其他可用的 CPU 上，在那里中断处理程序可以自旋，直到*锁*变为可用。自旋锁接口提供了另一种锁定例程`spin_lock_irqsave()`，它会禁用当前处理器上的中断以及内核抢占。以下代码片段显示了该例程的基础代码：

```
unsigned long __lockfunc __raw_##op##_lock_irqsave(locktype##_t *lock)  \
{                                                                       \
        unsigned long flags;                                            \
                                                                        \
        for (;;) {                                                      \
                preempt_disable();                                      \
                local_irq_save(flags);                                  \
                if (likely(do_raw_##op##_trylock(lock)))                \
                        break;                                          \
                local_irq_restore(flags);                               \
                preempt_enable();                                       \
                                                                        \
                if (!(lock)->break_lock)                                \
                        (lock)->break_lock = 1;                         \
                while (!raw_##op##_can_lock(lock) && (lock)->break_lock)\
                        arch_##op##_relax(&lock->raw_lock);             \
        }                                                               \
        (lock)->break_lock = 0;                                         \
        return flags;                                                   \
} 
```

调用`local_irq_save()`来禁用当前处理器的硬中断；请注意，如果未能获取锁，则通过调用`local_irq_restore()`来启用中断。请注意，使用`spin_lock_irqsave()`获取的锁需要使用`spin_lock_irqrestore()`来解锁，这会在释放锁之前为当前处理器启用内核抢占和中断。

与硬中断处理程序类似，软中断上下文例程（如*softirqs，tasklets*和其他*bottom halves*）也可能争夺同一处理器上由进程上下文代码持有的*锁*。这可以通过在进程上下文中获取*锁*时禁用*bottom halves*的执行来防止。`spin_lock_bh()`是另一种锁定例程的变体，它负责挂起本地 CPU 上的中断上下文 bottom halves 的执行。

```
void __lockfunc __raw_##op##_lock_bh(locktype##_t *lock)                \
{                                                                       \
        unsigned long flags;                                            \
                                                                        \
        /* */                                                           \
        /* Careful: we must exclude softirqs too, hence the */          \
        /* irq-disabling. We use the generic preemption-aware */        \
        /* function: */                                                 \
        /**/                                                            \
        flags = _raw_##op##_lock_irqsave(lock);                         \
        local_bh_disable();                                             \
        local_irq_restore(flags);                                       \
} 
```

`local_bh_disable()`挂起本地 CPU 的 bottom half 执行。要释放由`spin_lock_bh()`获取的*锁*，调用者上下文将需要调用`spin_unlock_bh()`，这将释放本地 CPU 的自旋锁和 BH 锁。

以下是内核自旋锁 API 接口的摘要列表：

| **函数** | **描述** |
| --- | --- |
| `spin_lock_init()` | 初始化自旋锁 |
| `spin_lock()` | 获取锁，在竞争时自旋 |
| `spin_trylock()` | 尝试获取锁，在竞争时返回错误 |
| `spin_lock_bh()` | 通过挂起本地处理器上的 BH 例程来获取锁，在竞争时自旋 |
| `spin_lock_irqsave()` | 通过保存当前中断状态来挂起本地处理器上的中断来获取锁，在竞争时自旋 |
| `spin_lock_irq()` | 通过挂起本地处理器上的中断来获取锁，在竞争时自旋 |
| `spin_unlock()` | 释放锁 |
| `spin_unlock_bh()` | 释放本地处理器的锁并启用 bottom half |
| `spin_unlock_irqrestore()` | 释放锁并将本地中断恢复到先前的状态 |
| `spin_unlock_irq()` | 释放锁并恢复本地处理器的中断 |
| `spin_is_locked()` | 返回锁的状态，如果锁被持有则返回非零，如果锁可用则返回零 |

# 读写器自旋锁

到目前为止讨论的自旋锁实现通过强制并发代码路径之间的标准互斥来保护共享数据的访问。这种形式的排斥不适合保护经常被并发代码路径读取的共享数据，而写入或更新很少。读写锁强制在读取器和写入器路径之间进行排斥；这允许并发读取器共享锁，而读取任务将需要等待锁，而写入器拥有锁。Rw-locks 强制在并发写入器之间进行标准排斥，这是期望的。

Rw-locks 由在内核头文件`<linux/rwlock_types.h>`中声明的`struct rwlock_t`表示：

```
typedef struct {
        arch_rwlock_t raw_lock;
#ifdef CONFIG_GENERIC_LOCKBREAK
        unsigned int break_lock;
#endif
#ifdef CONFIG_DEBUG_SPINLOCK
        unsigned int magic, owner_cpu;
        void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
        struct lockdep_map dep_map;
#endif
} rwlock_t;
```

rwlocks 可以通过宏`DEFINE_RWLOCK(v_rwlock)`静态初始化，也可以通过`rwlock_init(v_rwlock)`在运行时动态初始化。

读取器代码路径将需要调用`read_lock`例程。

```
read_lock(&v_rwlock);
/* critical section with read only access to shared data */
read_unlock(&v_rwlock);
```

写入器代码路径使用以下内容：

```
write_lock(&v_rwlock);
/* critical section for both read and write */
write_unlock(&v_lock);
```

当锁有争用时，读取和写入锁例程都会自旋。该接口还提供了称为`read_trylock()`和`write_trylock()`的非自旋版本的锁函数。它还提供了锁定调用的中断禁用版本，当读取或写入路径恰好在中断或底半部上下文中执行时非常方便。

以下是接口操作的摘要列表：

| **函数** | **描述** |
| --- | --- |
| `read_lock()` | 标准读锁接口，当有争用时会自旋 |
| `read_trylock()` | 尝试获取锁，如果锁不可用则返回错误 |
| `read_lock_bh()` | 通过挂起本地 CPU 的 BH 执行来尝试获取锁，当有争用时会自旋 |
| `read_lock_irqsave()` | 通过保存本地中断的当前状态来尝试通过挂起当前 CPU 的中断来获取锁，当有争用时会自旋 |
| `read_unlock()` | 释放读锁 |
| `read_unlock_irqrestore()` | 释放持有的锁并将本地中断恢复到先前的状态 |
| `read_unlock_bh()` | 释放读锁并在本地处理器上启用 BH |
| `write_lock()` | 标准写锁接口，当有争用时会自旋 |
| `write_trylock()` | 尝试获取锁，如果有争用则返回错误 |
| `write_lock_bh()` | 尝试通过挂起本地 CPU 的底半部来获取写锁，当有争用时会自旋 |
| `wrtie_lock_irqsave()` | 通过保存本地中断的当前状态来尝试通过挂起本地 CPU 的中断来获取写锁，当有争用时会自旋 |
| `write_unlock()` | 释放写锁 |
| `write_unlock_irqrestore()` | 释放锁并将本地中断恢复到先前的状态 |
| `write_unlock_bh()` | 释放写锁并在本地处理器上启用 BH |

所有这些操作的底层调用与自旋锁实现的类似，并且可以在前面提到的自旋锁部分指定的头文件中找到。

# 互斥锁

自旋锁的设计更适用于锁定持续时间短、固定的情况，因为无限期的忙等待会对系统的性能产生严重影响。然而，有许多情况下锁定持续时间较长且不确定；**睡眠锁**正是为这种情况而设计的。内核互斥锁是睡眠锁的一种实现：当调用任务尝试获取一个不可用的互斥锁（已被另一个上下文拥有），它会被置于休眠状态并移出到等待队列，强制进行上下文切换，从而允许 CPU 运行其他有生产力的任务。当互斥锁变为可用时，等待队列中的任务将被唤醒并通过互斥锁的解锁路径移动，然后尝试*锁定*互斥锁。

互斥锁由`include/linux/mutex.h`中定义的`struct mutex`表示，并且相应的操作在源文件`kernel/locking/mutex.c`中实现：

```
 struct mutex {
          atomic_long_t owner;
          spinlock_t wait_lock;
 #ifdef CONFIG_MUTEX_SPIN_ON_OWNER
          struct optimistic_spin_queue osq; /* Spinner MCS lock */
 #endif
          struct list_head wait_list;
 #ifdef CONFIG_DEBUG_MUTEXES
          void *magic;
 #endif
 #ifdef CONFIG_DEBUG_LOCK_ALLOC
          struct lockdep_map dep_map;
 #endif
 }; 
```

在其基本形式中，每个互斥锁都包含一个 64 位的`atomic_long_t`计数器（`owner`），用于保存锁定状态，并存储当前拥有锁的任务结构的引用。每个互斥锁都包含一个等待队列（`wait_list`）和一个自旋锁（`wait_lock`），用于对`wait_list`进行串行访问。

互斥锁 API 接口提供了一组宏和函数，用于初始化、锁定、解锁和访问互斥锁的状态。这些操作接口在`<include/linux/mutex.h>`中定义。

可以使用宏`DEFINE_MUTEX(name)`声明和初始化互斥锁。

还有一种选项，可以通过`mutex_init(mutex)`动态初始化有效的互斥锁。

如前所述，在争用时，锁操作会将调用线程置于休眠状态，这要求在将其移入互斥锁等待列表之前，将调用线程置于`TASK_INTERRUPTIBLE`、`TASK_UNINTERRUPTIBLE`或`TASK_KILLABLE`状态。为了支持这一点，互斥锁实现提供了两种锁操作的变体，一种用于**不可中断**，另一种用于**可中断**休眠。以下是每个标准互斥锁操作的简要描述：

```
/**
 * mutex_lock - acquire the mutex
 * @lock: the mutex to be acquired
 *
 * Lock the mutex exclusively for this task. If the mutex is not
 * available right now, Put caller into Uninterruptible sleep until mutex 
 * is available.
 */
    void mutex_lock(struct mutex *lock);

/**
 * mutex_lock_interruptible - acquire the mutex, interruptible
 * @lock: the mutex to be acquired
 *
 * Lock the mutex like mutex_lock(), and return 0 if the mutex has
 * been acquired else put caller into interruptible sleep until the mutex  
 * until mutex is available. Return -EINTR if a signal arrives while sleeping
 * for the lock.                               
 */
 int __must_check mutex_lock_interruptible(struct mutex *lock); /**
 * mutex_lock_Killable - acquire the mutex, interruptible
 * @lock: the mutex to be acquired
 *
 * Similar to mutex_lock_interruptible(),with a difference that the call
 * returns -EINTR only when fatal KILL signal arrives while sleeping for the     
 * lock.                              
 */
 int __must_check mutex_lock_killable(struct mutex *lock); /**
 * mutex_trylock - try to acquire the mutex, without waiting
 * @lock: the mutex to be acquired
 *
 * Try to acquire the mutex atomically. Returns 1 if the mutex
 * has been acquired successfully, and 0 on contention.
 *
 */
    int mutex_trylock(struct mutex *lock); /**
 * atomic_dec_and_mutex_lock - return holding mutex if we dec to 0,
 * @cnt: the atomic which we are to dec
 * @lock: the mutex to return holding if we dec to 0
 *
 * return true and hold lock if we dec to 0, return false otherwise. Please 
 * note that this function is interruptible.
 */
    int atomic_dec_and_mutex_lock(atomic_t *cnt, struct mutex *lock); 
/**
 * mutex_is_locked - is the mutex locked
 * @lock: the mutex to be queried
 *
 * Returns 1 if the mutex is locked, 0 if unlocked.
 */
/**
 * mutex_unlock - release the mutex
 * @lock: the mutex to be released
 *
 * Unlock the mutex owned by caller task.
 *
 */
 void mutex_unlock(struct mutex *lock);
```

尽管可能会阻塞调用，但互斥锁定函数已经针对性能进行了大幅优化。它们被设计为在尝试获取锁时采用快速路径和慢速路径方法。让我们深入了解锁定调用的代码，以更好地理解快速路径和慢速路径。以下代码摘录是来自`<kernel/locking/mutex.c>`中的`mutex_lock()`例程：

```
void __sched mutex_lock(struct mutex *lock)
{
  might_sleep();

  if (!__mutex_trylock_fast(lock))
    __mutex_lock_slowpath(lock);
}
```

首先通过调用非阻塞的快速路径调用`__mutex_trylock_fast()`来尝试获取锁。如果由于争用而无法获取锁，则通过调用`__mutex_lock_slowpath()`进入慢速路径：

```
static __always_inline bool __mutex_trylock_fast(struct mutex *lock)
{
  unsigned long curr = (unsigned long)current;

  if (!atomic_long_cmpxchg_acquire(&lock->owner, 0UL, curr))
    return true;

  return false;
}
```

如果可用，此函数被设计为原子方式获取锁。它调用`atomic_long_cmpxchg_acquire()`宏，该宏尝试将当前线程分配为互斥锁的所有者；如果互斥锁可用，则此操作将成功，此时函数返回`true`。如果某些其他线程拥有互斥锁，则此函数将失败并返回`false`。在失败时，调用线程将进入慢速路径例程。

传统上，慢速路径的概念一直是将调用任务置于休眠状态，同时等待锁变为可用。然而，随着多核 CPU 的出现，人们对可伸缩性和性能的需求不断增长，因此为了实现可伸缩性，互斥锁慢速路径实现已经重新设计，引入了称为**乐观自旋**的优化，也称为**中间路径**，可以显著提高性能。

乐观自旋的核心思想是将竞争任务推入轮询或自旋，而不是在发现互斥体所有者正在运行时休眠。一旦互斥体变为可用（因为发现所有者正在运行，所以预计会更快），就假定自旋任务始终可以比互斥体等待列表中的挂起或休眠任务更快地获取它。但是，只有当没有其他处于就绪状态的更高优先级任务时，才有可能进行这种自旋。有了这个特性，自旋任务更有可能是缓存热点，从而产生可预测的执行，从而产生明显的性能改进：

```
static int __sched
__mutex_lock(struct mutex *lock, long state, unsigned int subclass,
       struct lockdep_map *nest_lock, unsigned long ip)
{
  return __mutex_lock_common(lock, state, subclass, nest_lock, ip, NULL,     false);
}

...
...
...

static noinline void __sched __mutex_lock_slowpath(struct mutex *lock) 
{
        __mutex_lock(lock, TASK_UNINTERRUPTIBLE, 0, NULL, _RET_IP_); 
}

static noinline int __sched
__mutex_lock_killable_slowpath(struct mutex *lock)
{
  return __mutex_lock(lock, TASK_KILLABLE, 0, NULL, _RET_IP_);
}

static noinline int __sched
__mutex_lock_interruptible_slowpath(struct mutex *lock)
{
  return __mutex_lock(lock, TASK_INTERRUPTIBLE, 0, NULL, _RET_IP_);
}

```

`__mutex_lock_common()`函数包含一个带有乐观自旋的慢路径实现；这个例程由所有互斥锁定函数的睡眠变体调用，带有适当的标志作为参数。这个函数首先尝试通过与互斥体关联的可取消的 mcs 自旋锁（互斥体结构中的 osq 字段）实现乐观自旋来获取互斥体。当调用者任务无法通过乐观自旋获取互斥体时，作为最后的手段，这个函数切换到传统的慢路径，导致调用者任务进入睡眠，并排队进入互斥体的`wait_list`，直到被解锁路径唤醒。

# 调试检查和验证

错误使用互斥操作可能导致死锁、排除失败等。为了检测和防止这种可能发生的情况，互斥子系统配备了适当的检查或验证，这些检查默认情况下是禁用的，可以通过在内核构建过程中选择配置选项`CONFIG_DEBUG_MUTEXES=y`来启用。

以下是受检的调试代码强制执行的检查列表：

+   互斥体在给定时间点只能由一个任务拥有

+   互斥体只能由有效所有者释放（解锁），任何尝试由不拥有锁的上下文释放互斥体的尝试都将失败

+   递归锁定或解锁尝试将失败

+   互斥体只能通过初始化调用进行初始化，并且任何对*memset*互斥体的尝试都不会成功

+   调用者任务可能不会在持有互斥锁的情况下退出

+   不得释放包含持有的锁的动态内存区域

+   互斥体只能初始化一次，任何尝试重新初始化已初始化的互斥体都将失败

+   互斥体可能不会在硬/软中断上下文例程中使用

死锁可能由许多原因触发，例如内核代码的执行模式和锁定调用的粗心使用。例如，让我们考虑这样一种情况：并发代码路径需要通过嵌套锁定函数来拥有*L[1]*和*L[2]*锁。必须确保所有需要这些锁的内核函数都被编程为以相同的顺序获取它们。当没有严格强制执行这样的顺序时，总会有两个不同的函数尝试以相反的顺序锁定*L1*和*L2*的可能性，这可能会触发锁反转死锁，当这些函数并发执行时。

内核锁验证器基础设施已经实施，以检查并证明在内核运行时观察到的任何锁定模式都不会导致死锁。此基础设施打印与锁定模式相关的数据，例如：

+   获取点跟踪、函数名称的符号查找和系统中所有持有的锁列表

+   所有者跟踪

+   检测自递归锁并打印所有相关信息

+   检测锁反转死锁并打印所有受影响的锁和任务

可以通过在内核构建过程中选择`CONFIG_PROVE_LOCKING=y`来启用锁验证器。

# 等待/伤害互斥体

如前一节所讨论的，在内核函数中无序的嵌套锁定可能会导致锁反转死锁的风险，内核开发人员通过定义嵌套锁定顺序的规则并通过锁验证器基础设施执行运行时检查来避免这种情况。然而，存在动态锁定顺序的情况，无法将嵌套锁定调用硬编码或根据预设规则强加。

一个这样的用例与 GPU 缓冲区有关；这些缓冲区应该由各种系统实体拥有和访问，比如 GPU 硬件、GPU 驱动程序、用户模式应用程序和其他与视频相关的驱动程序。用户模式上下文可以以任意顺序提交 dma 缓冲区进行处理，GPU 硬件可以在任意时间处理它们。如果使用锁来控制缓冲区的所有权，并且必须同时操作多个缓冲区，则无法避免死锁。等待/伤害互斥锁旨在促进嵌套锁的动态排序，而不会导致锁反转死锁。这是通过强制争用的上下文*伤害*来实现的，意味着强制它释放持有的锁。

例如，假设有两个缓冲区，每个缓冲区都受到锁的保护，进一步考虑两个线程，比如`T[1]`和`T[2]`，它们通过以相反的顺序尝试锁定来寻求对缓冲区的所有权：

```
Thread T1       Thread T2
===========    ==========
lock(bufA);     lock(bufB);
lock(bufB);     lock(bufA);
 ....            ....
 ....            ....
unlock(bufB);   unlock(bufA);
unlock(bufA);   unlock(bufB);
```

`T[1]`和`T[2]`的并发执行可能导致每个线程等待另一个持有的锁，从而导致死锁。等待/伤害互斥锁通过让*首先抓住锁的线程*保持睡眠，等待嵌套锁可用来防止这种情况。另一个线程被*伤害*，导致它释放其持有的锁并重新开始。假设`T[1]`在`bufA`上获得锁之前，`T[2]`可以在`bufB`上获得锁。`T[1]`将被视为*首先到达的线程*，并被放到`bufB`的锁上睡眠，`T[2]`将被伤害，导致它释放`bufB`上的锁并重新开始。这样可以避免死锁，当`T[1]`释放持有的锁时，`T[2]`将重新开始。

# 操作接口：

等待/伤害互斥锁通过在头文件`<linux/ww_mutex.h>`中定义的`struct ww_mutex`来表示：

```
struct ww_mutex {
       struct mutex base;
       struct ww_acquire_ctx *ctx;
# ifdef CONFIG_DEBUG_MUTEXES
       struct ww_class *ww_class;
#endif
};
```

使用等待/伤害互斥锁的第一步是定义一个*类*，这是一种表示一组锁的机制。当并发任务争夺相同的锁时，它们必须通过指定这个类来这样做。

可以使用宏定义一个类：

```
static DEFINE_WW_CLASS(bufclass);
```

声明的每个类都是`struct ww_class`类型的实例，并包含一个原子计数器`stamp`，用于记录哪个竞争任务*首先到达*的序列号。其他字段由内核的锁验证器用于验证等待/伤害机制的正确使用。

```
struct ww_class {
       atomic_long_t stamp;
       struct lock_class_key acquire_key;
       struct lock_class_key mutex_key;
       const char *acquire_name;
       const char *mutex_name;
};
```

每个竞争的线程在尝试嵌套锁定调用之前必须调用`ww_acquire_init()`。这通过分配一个序列号来设置上下文以跟踪锁。

```
/**
 * ww_acquire_init - initialize a w/w acquire context
 * @ctx: w/w acquire context to initialize
 * @ww_class: w/w class of the context
 *
 * Initializes a context to acquire multiple mutexes of the given w/w class.
 *
 * Context-based w/w mutex acquiring can be done in any order whatsoever 
 * within a given lock class. Deadlocks will be detected and handled with the
 * wait/wound logic.
 *
 * Mixing of context-based w/w mutex acquiring and single w/w mutex locking 
 * can result in undetected deadlocks and is so forbidden. Mixing different
 * contexts for the same w/w class when acquiring mutexes can also result in 
 * undetected deadlocks, and is hence also forbidden. Both types of abuse will 
 * will be caught by enabling CONFIG_PROVE_LOCKING.
 *
 */
   void ww_acquire_init(struct ww_acquire_ctx *ctx, struct ww_clas *ww_class);
```

一旦上下文设置和初始化，任务可以开始使用`ww_mutex_lock()`或`ww_mutex_lock_interruptible()`调用获取锁：

```
/**
 * ww_mutex_lock - acquire the w/w mutex
 * @lock: the mutex to be acquired
 * @ctx: w/w acquire context, or NULL to acquire only a single lock.
 *
 * Lock the w/w mutex exclusively for this task.
 *
 * Deadlocks within a given w/w class of locks are detected and handled with 
 * wait/wound algorithm. If the lock isn't immediately available this function
 * will either sleep until it is(wait case) or it selects the current context
 * for backing off by returning -EDEADLK (wound case).Trying to acquire the
 * same lock with the same context twice is also detected and signalled by
 * returning -EALREADY. Returns 0 if the mutex was successfully acquired.
 *
 * In the wound case the caller must release all currently held w/w mutexes  
 * for the given context and then wait for this contending lock to be 
 * available by calling ww_mutex_lock_slow. 
 *
 * The mutex must later on be released by the same task that
 * acquired it. The task may not exit without first unlocking the mutex.Also,
 * kernel memory where the mutex resides must not be freed with the mutex 
 * still locked. The mutex must first be initialized (or statically defined) b
 * before it can be locked. memset()-ing the mutex to 0 is not allowed. The
 * mutex must be of the same w/w lock class as was used to initialize the 
 * acquired context.
 * A mutex acquired with this function must be released with ww_mutex_unlock.
 */
    int ww_mutex_lock(struct ww_mutex *lock, struct ww_acquire_ctx *ctx);

/**
 * ww_mutex_lock_interruptible - acquire the w/w mutex, interruptible
 * @lock: the mutex to be acquired
 * @ctx: w/w acquire context
 *
 */
   int  ww_mutex_lock_interruptible(struct ww_mutex *lock, 
                                             struct  ww_acquire_ctx *ctx);
```

当任务抓取与类相关的所有嵌套锁（使用这些锁定例程中的任何一个）时，需要使用函数`ww_acquire_done()`通知所有权的获取。这个调用标志着获取阶段的结束，任务可以继续处理共享数据：

```
/**
 * ww_acquire_done - marks the end of the acquire phase
 * @ctx: the acquire context
 *
 * Marks the end of the acquire phase, any further w/w mutex lock calls using
 * this context are forbidden.
 *
 * Calling this function is optional, it is just useful to document w/w mutex
 * code and clearly designated the acquire phase from actually using the 
 * locked data structures.
 */
 void ww_acquire_done(struct ww_acquire_ctx *ctx);
```

当任务完成对共享数据的处理时，可以通过调用`ww_mutex_unlock()`例程开始释放所有持有的锁。一旦所有锁都被释放，*上下文*必须通过调用`ww_acquire_fini()`来释放：

```
/**
 * ww_acquire_fini - releases a w/w acquire context
 * @ctx: the acquire context to free
 *
 * Releases a w/w acquire context. This must be called _after_ all acquired 
 * w/w mutexes have been released with ww_mutex_unlock.
 */
    void ww_acquire_fini(struct ww_acquire_ctx *ctx);
```

# 信号量

在 2.6 内核早期版本之前，信号量是睡眠锁的主要形式。典型的信号量实现包括一个计数器、等待队列和一组可以原子地增加/减少计数器的操作。

当信号量用于保护共享资源时，其计数器被初始化为大于零的数字，被视为解锁状态。寻求访问共享资源的任务首先通过对信号量进行减操作来开始。此调用检查信号量计数器；如果发现大于零，则将计数器减一，并返回成功。但是，如果计数器为零，则减操作将调用者任务置于睡眠状态，直到计数器增加到大于零为止。

这种简单的设计提供了很大的灵活性，允许信号量适应和应用于不同的情况。例如，对于需要在任何时候对特定数量的任务可访问的资源的情况，信号量计数可以初始化为需要访问的任务数量，比如 10，这允许最多 10 个任务在任何时候访问共享资源。对于其他情况，例如需要互斥访问共享资源的任务数量，信号量计数可以初始化为 1，导致在任何给定时刻最多一个任务访问资源。

信号量结构及其接口操作在内核头文件`<include/linux/semaphore.h>`中声明：

```
struct semaphore {
        raw_spinlock_t     lock;
        unsigned int       count;
        struct list_head   wait_list;
};
```

自旋锁（`lock`字段）用作对`count`的保护，也就是说，信号量操作（增加/减少）被编程为在操作`count`之前获取`lock`。`wait_list`用于将任务排队等待，直到信号量计数增加到零以上为止。

信号量可以通过宏`DEFINE_SEMAPHORE(s)`声明和初始化为 1。

信号量也可以通过以下方式动态初始化为任何正数：

```
void sema_init(struct semaphore *sem, int val)
```

以下是一系列操作接口及其简要描述。命名约定为`down_xxx()`的例程尝试减少信号量，并且可能是阻塞调用（除了`down_trylock()`），而例程`up()`增加信号量并且总是成功：

```
/**
 * down_interruptible - acquire the semaphore unless interrupted
 * @sem: the semaphore to be acquired
 *
 * Attempts to acquire the semaphore.  If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the sleep is interrupted by a signal, this function will return -EINTR.
 * If the semaphore is successfully acquired, this function returns 0.
 */
 int down_interruptible(struct semaphore *sem); /**
 * down_killable - acquire the semaphore unless killed
 * @sem: the semaphore to be acquired
 *
 * Attempts to acquire the semaphore.  If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the sleep is interrupted by a fatal signal, this function will return
 * -EINTR.  If the semaphore is successfully acquired, this function returns
 * 0.
 */
 int down_killable(struct semaphore *sem); /**
 * down_trylock - try to acquire the semaphore, without waiting
 * @sem: the semaphore to be acquired
 *
 * Try to acquire the semaphore atomically.  Returns 0 if the semaphore has
 * been acquired successfully or 1 if it it cannot be acquired.
 *
 */
 int down_trylock(struct semaphore *sem); /**
 * down_timeout - acquire the semaphore within a specified time
 * @sem: the semaphore to be acquired
 * @timeout: how long to wait before failing
 *
 * Attempts to acquire the semaphore.  If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the semaphore is not released within the specified number of jiffies,
 * this function returns -ETIME.  It returns 0 if the semaphore was acquired.
 */
 int down_timeout(struct semaphore *sem, long timeout); /**
 * up - release the semaphore
 * @sem: the semaphore to release
 *
 * Release the semaphore.  Unlike mutexes, up() may be called from any
 * context and even by tasks which have never called down().
 */
 void up(struct semaphore *sem);
```

与互斥锁实现不同，信号量操作不支持调试检查或验证；这个约束是由于它们固有的通用设计，允许它们被用作排他锁、事件通知计数器等。自从互斥锁进入内核（2.6.16）以来，信号量不再是排他性的首选，信号量作为锁的使用大大减少，而对于其他目的，内核有备用接口。大部分使用信号量的内核代码已经转换为互斥锁，只有少数例外。然而，信号量仍然存在，并且至少在所有使用它们的内核代码转换为互斥锁或其他合适的接口之前，它们可能会继续存在。

# 读写信号量

该接口是睡眠读写排他的实现，作为自旋的替代。读写信号量由`struct rw_semaphore`表示，在内核头文件`<linux/rwsem.h>`中声明：

```
struct rw_semaphore {
        atomic_long_t count;
        struct list_head wait_list;
        raw_spinlock_t wait_lock;
#ifdef CONFIG_RWSEM_SPIN_ON_OWNER
       struct optimistic_spin_queue osq; /* spinner MCS lock */
       /*
       * Write owner. Used as a speculative check to see
       * if the owner is running on the cpu.
       */
      struct task_struct *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
     struct lockdep_map dep_map;
#endif
};
```

该结构与互斥锁的结构相同，并且设计为支持通过`osq`进行乐观自旋；它还通过内核的*lockdep*包括调试支持。`Count`用作排他计数器，设置为 1，允许最多一个写者在某一时刻拥有锁。这是因为互斥仅在竞争写者之间执行，并且任意数量的读者可以同时共享读锁。`wait_lock`是一个自旋锁，用于保护信号量`wait_list`。

`rw_semaphore`可以通过`DECLARE_RWSEM(name)`静态实例化和初始化，也可以通过`init_rwsem(sem)`动态初始化。

与 rw 自旋锁一样，该接口也为读者和写者路径的锁获取提供了不同的例程。以下是接口操作的列表：

```
/* reader interfaces */
   void down_read(struct rw_semaphore *sem);
   void up_read(struct rw_semaphore *sem);
/* trylock for reading -- returns 1 if successful, 0 if contention */
   int down_read_trylock(struct rw_semaphore *sem);
   void up_read(struct rw_semaphore *sem);

/* writer Interfaces */
   void down_write(struct rw_semaphore *sem);
   int __must_check down_write_killable(struct rw_semaphore *sem);

/* trylock for writing -- returns 1 if successful, 0 if contention */
   int down_write_trylock(struct rw_semaphore *sem); 
   void up_write(struct rw_semaphore *sem);
/* downgrade write lock to read lock */
   void downgrade_write(struct rw_semaphore *sem); 

/* check if rw-sem is currently locked */  
   int rwsem_is_locked(struct rw_semaphore *sem);

```

这些操作是在源文件`<kernel/locking/rwsem.c>`中实现的；代码相当自解释，我们不会进一步讨论它。

# 序列锁

传统的读写锁设计为读者优先，它们可能导致写入任务等待非确定性的持续时间，这在具有时间敏感更新的共享数据上可能不合适。这就是顺序锁派上用场的地方，因为它旨在提供对共享资源的快速和无锁访问。当需要保护的资源较小且简单，写访问快速且不频繁时，顺序锁是最佳选择，因为在内部，顺序锁会退回到自旋锁原语。

顺序锁引入了一个特殊的计数器，每当写入者获取顺序锁时都会增加该计数器，并附带一个自旋锁。写入者完成后，释放自旋锁并再次增加计数器，为其他写入者打开访问。对于读取，有两种类型的读取者：序列读取者和锁定读取者。**序列读取者**在进入临界区之前检查计数器，然后在不阻塞任何写入者的情况下在临界区结束时再次检查。如果计数器保持不变，这意味着在读取期间没有写入者访问该部分，但如果在部分结束时计数器增加，则表明写入者已访问，这要求读取者重新读取临界部分以获取更新的数据。**锁定读取者**会获得锁并在进行时阻塞其他读取者和写入者；当另一个锁定读取者或写入者进行时，它也会等待。

序列锁由以下类型表示：

```
typedef struct {
        struct seqcount seqcount;
        spinlock_t lock;
} seqlock_t;
```

我们可以使用以下宏静态初始化序列锁：

```
#define DEFINE_SEQLOCK(x) \
               seqlock_t x = __SEQLOCK_UNLOCKED(x)
```

实际初始化是使用`__SEQLOCK_UNLOCKED(x)`来完成的，其定义在这里：

```
#define __SEQLOCK_UNLOCKED(lockname)                 \
       {                                               \
               .seqcount = SEQCNT_ZERO(lockname),     \
               .lock = __SPIN_LOCK_UNLOCKED(lockname)   \
       }
```

要动态初始化序列锁，我们需要使用`seqlock_init`宏，其定义如下：

```
  #define seqlock_init(x)                                     \
       do {                                                   \
               seqcount_init(&(x)->seqcount);                 \
               spin_lock_init(&(x)->lock);                    \
       } while (0)
```

# API

Linux 提供了许多用于使用序列锁的 API，这些 API 在`</linux/seqlock.h>`中定义。以下是一些重要的 API：

```
static inline void write_seqlock(seqlock_t *sl)
{
        spin_lock(&sl->lock);
        write_seqcount_begin(&sl->seqcount);
}

static inline void write_sequnlock(seqlock_t *sl)
{
        write_seqcount_end(&sl->seqcount);
        spin_unlock(&sl->lock);
}

static inline void write_seqlock_bh(seqlock_t *sl)
{
        spin_lock_bh(&sl->lock);
        write_seqcount_begin(&sl->seqcount);
}

static inline void write_sequnlock_bh(seqlock_t *sl)
{
        write_seqcount_end(&sl->seqcount);
        spin_unlock_bh(&sl->lock);
}

static inline void write_seqlock_irq(seqlock_t *sl)
{
        spin_lock_irq(&sl->lock);
        write_seqcount_begin(&sl->seqcount);
}

static inline void write_sequnlock_irq(seqlock_t *sl)
{
        write_seqcount_end(&sl->seqcount);
        spin_unlock_irq(&sl->lock);
}

static inline unsigned long __write_seqlock_irqsave(seqlock_t *sl)
{
        unsigned long flags;

        spin_lock_irqsave(&sl->lock, flags);
        write_seqcount_begin(&sl->seqcount);
        return flags;
}
```

以下两个函数用于通过开始和完成读取部分：

```
static inline unsigned read_seqbegin(const seqlock_t *sl)
{
        return read_seqcount_begin(&sl->seqcount);
}

static inline unsigned read_seqretry(const seqlock_t *sl, unsigned start)
{
        return read_seqcount_retry(&sl->seqcount, start);
}
```

# 完成锁

**完成锁**是一种有效的方式来实现代码同步，如果需要一个或多个执行线程等待某个事件的完成，比如等待另一个进程达到某个点或状态。完成锁可能比信号量更受欢迎，原因有几点：多个执行线程可以等待完成，并且使用`complete_all()`，它们可以一次性全部释放。这比信号量唤醒多个线程要好得多。其次，如果等待线程释放同步对象，信号量可能导致竞争条件；使用完成时，这个问题就不存在。

通过包含`<linux/completion.h>`并创建一个`struct completion`类型的变量来使用完成结构，这是一个用于维护完成状态的不透明结构。它使用 FIFO 来排队等待完成事件的线程：

```
struct completion {
        unsigned int done;
        wait_queue_head_t wait;
};
```

完成基本上包括初始化完成结构，通过`wait_for_completion()`调用的任何变体等待，最后通过`complete()`或`complete_all()`调用发出完成信号。在其生命周期中还有函数来检查完成的状态。

# 初始化

以下宏可用于静态声明和初始化完成结构：

```
#define DECLARE_COMPLETION(work) \
       struct completion work = COMPLETION_INITIALIZER(work)
```

以下内联函数将初始化动态创建的完成结构：

```
static inline void init_completion(struct completion *x)
{
        x->done = 0;
        init_waitqueue_head(&x->wait);
}
```

以下内联函数将用于在需要重用时重新初始化完成结构。这可以在`complete_all()`之后使用：

```
static inline void reinit_completion(struct completion *x)
{
        x->done = 0;
}
```

# 等待完成

如果任何线程需要等待任务完成，它将在初始化的完成结构上调用`wait_for_completion（）`。如果`wait_for_completion`操作发生在调用`complete（）`或`complete_all（）`之后，则线程将简单地继续，因为它想要等待的原因已经得到满足；否则，它将等待直到`complete（）`被发出信号。对于`wait_for_completion（）`调用有可用的变体：

```
extern void wait_for_completion_io(struct completion *);
extern int wait_for_completion_interruptible(struct completion *x);
extern int wait_for_completion_killable(struct completion *x);
extern unsigned long wait_for_completion_timeout(struct completion *x,
                                                   unsigned long timeout);
extern unsigned long wait_for_completion_io_timeout(struct completion *x,
                                                    unsigned long timeout);
extern long wait_for_completion_interruptible_timeout(
        struct completion *x, unsigned long timeout);
extern long wait_for_completion_killable_timeout(
        struct completion *x, unsigned long timeout);
extern bool try_wait_for_completion(struct completion *x);
extern bool completion_done(struct completion *x);

extern void complete(struct completion *);
extern void complete_all(struct completion *);
```

# 完成信号

希望发出完成预期任务的执行线程调用`complete（）`向等待的线程发出信号，以便它可以继续。线程将按照它们排队的顺序被唤醒。在有多个等待者的情况下，它调用`complete_all（）`：

```
void complete(struct completion *x)
{
        unsigned long flags;

        spin_lock_irqsave(&x->wait.lock, flags);
        if (x->done != UINT_MAX)
                x->done++;
        __wake_up_locked(&x->wait, TASK_NORMAL, 1);
        spin_unlock_irqrestore(&x->wait.lock, flags);
}
EXPORT_SYMBOL(complete);
void complete_all(struct completion *x)
{
        unsigned long flags;

        spin_lock_irqsave(&x->wait.lock, flags);
        x->done = UINT_MAX;
        __wake_up_locked(&x->wait, TASK_NORMAL, 0);
        spin_unlock_irqrestore(&x->wait.lock, flags);
}
EXPORT_SYMBOL(complete_all);
```

# 总结

在本章中，我们不仅了解了内核提供的各种保护和同步机制，还试图欣赏这些选项的有效性，以及它们的各种功能和缺陷。本章的收获必须是内核处理这些不同复杂性以提供数据保护和同步的坚韧性。另一个值得注意的事实是内核在处理这些问题时保持了编码的便利性和设计的优雅。

在我们的下一章中，我们将看一下中断如何由内核处理的另一个关键方面。
