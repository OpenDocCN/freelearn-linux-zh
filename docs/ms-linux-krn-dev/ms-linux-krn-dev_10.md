# 第十章：时钟和时间管理

Linux 时间管理子系统管理各种与时间相关的活动，并跟踪时间数据，如当前时间和日期、自系统启动以来经过的时间（系统正常运行时间）和超时，例如，等待特定事件启动或终止的时间、在超时后锁定系统，或引发信号以终止无响应的进程。

Linux 时间管理子系统处理两种类型的定时活动：

+   跟踪当前时间和日期

+   维护定时器

# 时间表示

根据使用情况，Linux 以三种不同的方式表示时间：

1.  **墙上时间（或实时时间）：**这是真实世界中的实际时间和日期，例如 2017 年 8 月 10 日上午 07:00，用于文件和通过网络发送的数据包的时间戳。

1.  **进程时间：**这是进程在其生命周期中消耗的时间。它包括进程在用户模式下消耗的时间以及内核代码在代表进程执行时消耗的时间。这对于统计目的、审计和分析很有用。

1.  **单调时间：**这是自系统启动以来经过的时间。它是不断增加且单调的（系统正常运行时间）。

这三种时间可以用以下任一方式来衡量：

1.  **相对时间：**这是相对于某个特定事件的时间，例如自系统启动以来的 7 分钟，或自用户上次输入以来的 2 分钟。

1.  **绝对时间：**这是没有任何参考先前事件的唯一时间点，例如 2017 年 8 月 12 日上午 10:00。在 Linux 中，绝对时间表示为自 1970 年 1 月 1 日午夜 00:00:00（UTC）以来经过的秒数。

墙上的时间是不断增加的（除非用户修改了它），即使在重新启动和关机之间，但进程时间和系统正常运行时间始于某个预定义的时间点（*通常为零*），每次创建新进程或系统启动时。

# 计时硬件

Linux 依赖于适当的硬件设备来维护时间。这些硬件设备可以大致分为两类：系统时钟和定时器。

# 实时时钟（RTC）

跟踪当前时间和日期非常重要，不仅是为了让用户了解时间，还可以将其用作系统中各种资源的时间戳，特别是存储在辅助存储器中的文件。每个文件都有元数据信息，如创建日期和最后修改日期，每当创建或修改文件时，这两个字段都会使用系统中的当前时间进行更新。这些字段被多个应用程序用于管理文件，例如排序、分组，甚至删除（如果文件长时间未被访问）。*make*工具使用此时间戳来确定自上次访问以来源文件是否已被编辑；只有在这种情况下才会对其进行编译，否则保持不变。

系统时钟 RTC 跟踪当前时间和日期；由额外的电池支持，即使系统关闭，它也会继续运行。

RTC 可以定期在 IRQ8 上引发中断。通过编程 RTC 在达到特定时间时在 IRQ8 上引发中断，可以将此功能用作警报设施。在兼容 IBM 的个人电脑中，RTC 被映射到 0x70 和 0x71 I/O 端口。可以通过`/dev/rtc`设备文件访问它。

# 时间戳计数器（TSC）

这是通过 64 位寄存器 TSC 实现的计数器，每个 x86 微处理器都有，该寄存器称为 TSC 寄存器。它计算处理器的 CLK 引脚上到达的时钟信号数量。可以通过访问 TSC 寄存器来读取当前计数器值。每秒计数的时钟信号数可以计算为 1/(时钟频率)；对于 1 GHz 时钟，这相当于每纳秒一次。

知道两个连续 tick 之间的持续时间非常关键。一个处理器时钟的频率可能与其他处理器不同，这使得它在处理器之间变化。CPU 时钟频率是在系统引导期间通过`calibrate_tsc()`回调例程计算的，该例程定义在`arch/x86/include/asm/x86_init.h`头文件中的`x86_platform_ops`结构中：

```
struct x86_platform_ops {
        unsigned long (*calibrate_cpu)(void);
        unsigned long (*calibrate_tsc)(void);
        void (*get_wallclock)(struct timespec *ts);
        int (*set_wallclock)(const struct timespec *ts);
        void (*iommu_shutdown)(void);
        bool (*is_untracked_pat_range)(u64 start, u64 end);
        void (*nmi_init)(void);
        unsigned char (*get_nmi_reason)(void);
        void (*save_sched_clock_state)(void);
        void (*restore_sched_clock_state)(void);
        void (*apic_post_init)(void);
        struct x86_legacy_features legacy;
        void (*set_legacy_features)(void);
};
```

这个数据结构还管理其他计时操作，比如通过`get_wallclock()`从 RTC 获取时间或通过`set_wallclock()`回调在 RTC 上设置时间。

# 可编程中断定时器（PIT）

内核需要定期执行某些任务，比如：

+   更新当前时间和日期（在午夜）

+   更新系统运行时间（正常运行时间）

+   跟踪每个进程消耗的时间，以便它们不超过分配给 CPU 运行的时间

+   跟踪各种计时器活动

为了执行这些任务，必须定期引发中断。每次引发这种周期性中断时，内核都知道是时候更新前面提到的时间数据了。PIT 是负责发出这种周期性中断的硬件部件，称为定时器中断。PIT 会以大约 1000 赫兹的频率在 IRQ0 上定期发出定时器中断，即每毫秒一次。这种周期性中断称为**tick**，发出的频率称为**tick rate**。tick rate 频率由内核宏**HZ**定义，以赫兹为单位。

系统响应性取决于 tick rate：tick 越短，系统的响应性就越高，反之亦然。使用较短的 tick，`poll()`和`select()`系统调用将具有更快的响应时间。然而，较短的 tick rate 的相当大缺点是 CPU 将在内核模式下工作（执行定时器中断的中断处理程序）大部分时间，留下较少的时间供用户模式代码（程序）在其上执行。在高性能 CPU 中，这不会产生太多开销，但在较慢的 CPU 中，整体系统性能会受到相当大的影响。

为了在响应时间和系统性能之间取得平衡，在大多数机器上使用了 100 赫兹的 tick rate。除了*Alpha*和*m68knommu*使用 1000 赫兹的 tick rate 外，其余常见架构，包括*x86*（arm、powerpc、sparc、mips 等），使用了 100 赫兹的 tick rate。在*x86*机器中找到的常见 PIT 硬件是 Intel 8253。它是 I/O 映射的，并通过地址 0x40-0x43 进行访问。PIT 由`setup_pit_timer()`初始化，定义在`arch/x86/kernel/i8253.c`文件中。

```
void __init setup_pit_timer(void)
{
        clockevent_i8253_init(true);
        global_clock_event = &i8253_clockevent;
}
```

这在内部调用`clockevent_i8253_init()`，定义在`<drivers/clocksource/i8253.c>`中：

```
void __init clockevent_i8253_init(bool oneshot)
{
        if (oneshot)
                i8253_clockevent.features |= CLOCK_EVT_FEAT_ONESHOT;
        /*
        * Start pit with the boot cpu mask. x86 might make it global
        * when it is used as broadcast device later.
        */
        i8253_clockevent.cpumask = cpumask_of(smp_processor_id());

        clockevents_config_and_register(&i8253_clockevent, PIT_TICK_RATE,
                                        0xF, 0x7FFF);
}
#endif
```

# CPU 本地定时器

PIT 是一个全局定时器，由它引发的中断可以由 SMP 系统中的任何 CPU 处理。在某些情况下，拥有这样一个共同的定时器是有益的，而在其他情况下，每 CPU 定时器更可取。在 SMP 系统中，保持进程时间并监视每个 CPU 中进程的分配时间片将更加容易和高效。

最近的 x86 微处理器中的本地 APIC 嵌入了这样一个 CPU 本地定时器。CPU 本地定时器可以发出一次或定期中断。它使用 32 位计时器，可以以非常低的频率发出中断（这个更宽的计数器允许更多的 tick 发生在引发中断之前）。APIC 定时器与总线时钟信号一起工作。APIC 定时器与 PIT 非常相似，只是它是本地 CPU 的，有一个 32 位计数器（PIT 有一个 16 位计数器），并且与总线时钟信号一起工作（PIT 使用自己的时钟信号）。

# 高精度事件定时器（HPET）

HPET 使用超过 10 Mhz 的时钟信号，每 100 纳秒发出一次中断，因此被称为高精度。HPET 实现了一个 64 位的主计数器，以如此高的频率进行计数。它是由英特尔和微软共同开发的，用于需要新的高分辨率计时器。HPET 嵌入了一组定时器。每个定时器都能够独立发出中断，并可以由内核分配给特定应用程序使用。这些定时器被管理为定时器组，每个组最多可以有 32 个定时器。一个 HPET 最多可以实现 8 个这样的组。每个定时器都有一组*比较器*和*匹配寄存器*。当定时器的匹配寄存器中的值与主计数器的值匹配时，定时器会发出中断。定时器可以被编程为定期或周期性地生成中断。

寄存器是内存映射的，并具有可重定位的地址空间。在系统引导期间，BIOS 设置寄存器的地址空间并将其传递给内核。一旦 BIOS 映射了地址，内核就很少重新映射它。

# ACPI 电源管理计时器（ACPI PMT）

ACPI PMT 是一个简单的计数器，具有固定频率时钟，为 3.58 Mhz。它在每个时钟脉冲上递增。PMT 是端口映射的；BIOS 在引导期间的硬件初始化阶段负责地址映射。PMT 比 TSC 更可靠，因为它使用恒定的时钟频率。TSC 依赖于 CPU 时钟，根据当前负载可以被降频或超频，导致时间膨胀和不准确的测量。在所有情况下，HPET 是首选，因为它允许系统中存在非常短的时间间隔。

# 硬件抽象

每个系统至少有一个时钟计数器。与机器中的任何硬件设备一样，这个计数器也由一个结构表示和管理。硬件抽象由`include/linux/clocksource.h`头文件中定义的`struct clocksource`提供。该结构提供了回调函数来通过`read`、`enable`、`disable`、`suspend`和`resume`例程访问和处理计数器的电源管理：

```
struct clocksource {
        u64 (*read)(struct clocksource *cs);
        u64 mask;
        u32 mult;
        u32 shift;
        u64 max_idle_ns;
        u32 maxadj;
#ifdef CONFIG_ARCH_CLOCKSOURCE_DATA
        struct arch_clocksource_data archdata;
#endif
        u64 max_cycles;
        const char *name;
        struct list_head list;
        int rating;
        int (*enable)(struct clocksource *cs);
        void (*disable)(struct clocksource *cs);
        unsigned long flags;
        void (*suspend)(struct clocksource *cs);
        void (*resume)(struct clocksource *cs);
        void (*mark_unstable)(struct clocksource *cs);
        void (*tick_stable)(struct clocksource *cs);

        /* private: */
#ifdef CONFIG_CLOCKSOURCE_WATCHDOG
        /* Watchdog related data, used by the framework */
        struct list_head wd_list;
        u64 cs_last;
        u64 wd_last;
#endif
        struct module *owner;
};
```

成员`mult`和`shift`对于获取相关单位的经过时间非常有用。

# 计算经过的时间

到目前为止，我们知道在每个系统中都有一个自由运行的、不断递增的计数器，并且所有时间都是从中派生的，无论是墙上的时间还是任何持续时间。在这里计算时间（自计数器启动以来经过的秒数）的最自然的想法是将这个计数器提供的周期数除以时钟频率，如下式所示：

时间（秒）=（计数器值）/（时钟频率）

然而，这种方法有一个问题：它涉及除法（它使用迭代算法，使其成为四种基本算术运算中最慢的）和浮点计算，在某些体系结构上可能会更慢。在处理嵌入式平台时，浮点计算显然比在个人电脑或服务器平台上慢。

那么我们如何解决这个问题呢？与其使用除法，不如使用乘法和位移操作来计算时间。内核提供了一个辅助例程，以这种方式推导时间。`include/linux/clocksource.h`中定义的`clocksource_cyc2ns()`将时钟源周期转换为纳秒：

```
static inline s64 clocksource_cyc2ns(u64 cycles, u32 mult, u32 shift)
{
        return ((u64) cycles * mult) >> shift;
}
```

在这里，参数 cycles 是来自时钟源的经过的周期数，`mult`是周期到纳秒的乘数，而`shift`是周期到纳秒的除数（2 的幂）。这两个参数都是时钟源相关的。这些值是由之前讨论的时钟源内核抽象提供的。

时钟源硬件并非始终准确；它们的频率可能会变化。这种时钟变化会导致时间漂移（使时钟运行得更快或更慢）。在这种情况下，可以调整变量*mult*来弥补这种时间漂移。

在`kernel/time/clocksource.c`中定义的辅助例程`clocks_calc_mult_shift()`有助于评估`mult`和`shift`因子：

```
void
clocks_calc_mult_shift(u32 *mult, u32 *shift, u32 from, u32 to, u32 maxsec)
{
        u64 tmp;
        u32 sft, sftacc= 32;

        /*
        * Calculate the shift factor which is limiting the conversion
        * range:
        */
        tmp = ((u64)maxsec * from) >> 32;
        while (tmp) {
                tmp >>=1;
                sftacc--;
        }

        /*
        * Find the conversion shift/mult pair which has the best
        * accuracy and fits the maxsec conversion range:
        */
        for (sft = 32; sft > 0; sft--) {
                tmp = (u64) to << sft;
                tmp += from / 2;
                do_div(tmp, from);
                if ((tmp >> sftacc) == 0)
                        break;
        }
        *mult = tmp;
        *shift = sft;
}
```

两个事件之间的时间持续时间可以通过以下代码片段计算：

```
struct clocksource *cs = &curr_clocksource;
cycle_t start = cs->read(cs);
/* things to do */
cycle_t end = cs->read(cs);
cycle_t diff = end – start;
duration =  clocksource_cyc2ns(diff, cs->mult, cs->shift);
```

# Linux 时间保持数据结构、宏和辅助例程

现在我们将通过查看一些关键的时间保持结构、宏和辅助例程来扩大我们的认识，这些可以帮助程序员提取特定的与时间相关的数据。

# Jiffies

`jiffies`变量保存自系统启动以来经过的滴答数。每次发生滴答时，`jiffies`增加一。它是一个 32 位变量，这意味着对于 100 Hz 的滴答率，大约在 497 天后（对于 1000 Hz 的滴答率，在 49 天 17 小时后）会发生溢出。

为了解决这个问题，使用了 64 位变量`jiffies_64`，它允许在溢出发生之前经过数千万年。`jiffies`变量等于`jiffies_64`的 32 位最低有效位。之所以同时拥有`jiffies`和`jiffies_64`变量，是因为在 32 位机器中，无法原子地访问 64 位变量；在处理这两个 32 位半部分时需要一些同步，以避免在处理这两个 32 位半部分时发生任何计数器更新。在`/kernel/time/jiffies.c`源文件中定义的函数`get_jiffies_64()`返回`jiffies`的当前值：

```
u64 get_jiffies_64(void)
{
        unsigned long seq;
        u64 ret;

        do {
                seq = read_seqbegin(&jiffies_lock);
                ret = jiffies_64;
        } while (read_seqretry(&jiffies_lock, seq));
        return ret;
}
```

在处理`jiffies`时，必须考虑可能发生的回绕，因为在比较两个时间事件时会导致不可预测的结果。有四个宏在`include/linux/jiffies.h`中定义，用于此目的：

```
#define time_after(a,b)           \
       (typecheck(unsigned long, a) && \
        typecheck(unsigned long, b) && \
        ((long)((b) - (a)) < 0))
#define time_before(a,b)       time_after(b,a)

#define time_after_eq(a,b)     \
       (typecheck(unsigned long, a) && \
        typecheck(unsigned long, b) && \
        ((long)((a) - (b)) >= 0))
#define time_before_eq(a,b)    time_after_eq(b,a)
```

所有这些宏都返回布尔值；参数**a**和**b**是要比较的时间事件。如果 a 恰好是 b 之后的时间，`time_after()`返回 true，否则返回 false。相反，如果 a 恰好在 b 之前，`time_before()`返回 true，否则返回 false。`time_after_eq()`和`time_before_eq()`如果 a 和 b 都相等，则返回 true。可以使用`kernel/time/time.c`中定义的例程`jiffies_to_msecs()`、`jiffies_to_usecs()`将 jiffies 转换为其他时间单位，如毫秒、微秒和纳秒，以及`include/linux/jiffies.h`中的`jiffies_to_nsecs()`：

```
unsigned int jiffies_to_msecs(const unsigned long j)
{
#if HZ <= MSEC_PER_SEC && !(MSEC_PER_SEC % HZ)
        return (MSEC_PER_SEC / HZ) * j;
#elif HZ > MSEC_PER_SEC && !(HZ % MSEC_PER_SEC)
        return (j + (HZ / MSEC_PER_SEC) - 1)/(HZ / MSEC_PER_SEC);
#else
# if BITS_PER_LONG == 32
        return (HZ_TO_MSEC_MUL32 * j) >> HZ_TO_MSEC_SHR32;
# else
        return (j * HZ_TO_MSEC_NUM) / HZ_TO_MSEC_DEN;
# endif
#endif
}

unsigned int jiffies_to_usecs(const unsigned long j)
{
        /*
        * Hz doesn't go much further MSEC_PER_SEC.
        * jiffies_to_usecs() and usecs_to_jiffies() depend on that.
        */
        BUILD_BUG_ON(HZ > USEC_PER_SEC);

#if !(USEC_PER_SEC % HZ)
        return (USEC_PER_SEC / HZ) * j;
#else
# if BITS_PER_LONG == 32
        return (HZ_TO_USEC_MUL32 * j) >> HZ_TO_USEC_SHR32;
# else
        return (j * HZ_TO_USEC_NUM) / HZ_TO_USEC_DEN;
# endif
#endif
}

static inline u64 jiffies_to_nsecs(const unsigned long j)
{
        return (u64)jiffies_to_usecs(j) * NSEC_PER_USEC;
}
```

其他转换例程可以在`include/linux/jiffies.h`文件中探索。

# Timeval 和 timespec

在 Linux 中，当前时间是通过保持自 1970 年 1 月 1 日午夜以来经过的秒数（称为纪元）来维护的；这些中的每个第二个元素分别表示自上次秒数以来经过的时间，以微秒和纳秒为单位：

```
struct timespec {
        __kernel_time_t  tv_sec;                   /* seconds */
        long            tv_nsec;          /* nanoseconds */
};
#endif

struct timeval {
        __kernel_time_t          tv_sec;           /* seconds */
        __kernel_suseconds_t     tv_usec;  /* microseconds */
};
```

从时钟源读取的时间（计数器值）需要在某个地方累积和跟踪；`include/linux/timekeeper_internal.h`中定义的`struct tk_read_base`结构用于此目的：

```
struct tk_read_base {
        struct clocksource        *clock;
        cycle_t                  (*read)(struct clocksource *cs);
        cycle_t                  mask;
        cycle_t                  cycle_last;
        u32                      mult;
        u32                      shift;
        u64                      xtime_nsec;
        ktime_t                  base_mono;
};
```

`include/linux/timekeeper_internal.h`中定义的`struct timekeeper`结构保持各种时间保持值。它是用于维护和操作不同时间线的时间保持数据的主要数据结构，如单调和原始：

```
struct timekeeper {
        struct tk_read_base       tkr;
        u64                      xtime_sec;
        unsigned long           ktime_sec;
        struct timespec64 wall_to_monotonic;
        ktime_t                  offs_real;
        ktime_t                  offs_boot;
        ktime_t                  offs_tai;
        s32                      tai_offset;
        ktime_t                  base_raw;
        struct timespec64 raw_time;

        /* The following members are for timekeeping internal use */
        cycle_t                  cycle_interval;
        u64                      xtime_interval;
        s64                      xtime_remainder;
        u32                      raw_interval;
        u64                      ntp_tick;
        /* Difference between accumulated time and NTP time in ntp
        * shifted nano seconds. */
        s64                      ntp_error;
        u32                      ntp_error_shift;
        u32                      ntp_err_mult;
};
```

# 跟踪和维护时间

时间保持辅助例程`timekeeping_get_ns()`和`timekeeping_get_ns()`有助于获取通用时间和地球时间之间的校正因子（Δt），单位为纳秒：

```
static inline u64 timekeeping_delta_to_ns(struct tk_read_base *tkr, u64 delta)
{
        u64 nsec;

        nsec = delta * tkr->mult + tkr->xtime_nsec;
        nsec >>= tkr->shift;

        /* If arch requires, add in get_arch_timeoffset() */
        return nsec + arch_gettimeoffset();
}

static inline u64 timekeeping_get_ns(struct tk_read_base *tkr)
{
        u64 delta;

        delta = timekeeping_get_delta(tkr);
        return timekeeping_delta_to_ns(tkr, delta);
}
```

例程`logarithmic_accumulation()`更新 mono、raw 和 xtime 时间线；它将周期的移位间隔累积到纳秒的移位间隔中。例程`accumulate_nsecs_to_secs()`将`struct tk_read_base`的`xtime_nsec`字段中的纳秒累积到`struct timekeeper`的`xtime_sec`中。这些例程有助于跟踪系统中的当前时间，并在`kernel/time/timekeeping.c`中定义：

```
static u64 logarithmic_accumulation(struct timekeeper *tk, u64 offset,
                                    u32 shift, unsigned int *clock_set)
{
        u64 interval = tk->cycle_interval << shift;
        u64 snsec_per_sec;

        /* If the offset is smaller than a shifted interval, do nothing */
        if (offset < interval)
                return offset;

        /* Accumulate one shifted interval */
        offset -= interval;
        tk->tkr_mono.cycle_last += interval;
        tk->tkr_raw.cycle_last  += interval;

        tk->tkr_mono.xtime_nsec += tk->xtime_interval << shift;
        *clock_set |= accumulate_nsecs_to_secs(tk);

        /* Accumulate raw time */
        tk->tkr_raw.xtime_nsec += (u64)tk->raw_time.tv_nsec << tk->tkr_raw.shift;
        tk->tkr_raw.xtime_nsec += tk->raw_interval << shift;
        snsec_per_sec = (u64)NSEC_PER_SEC << tk->tkr_raw.shift;
        while (tk->tkr_raw.xtime_nsec >= snsec_per_sec) {
                tk->tkr_raw.xtime_nsec -= snsec_per_sec;
                tk->raw_time.tv_sec++;
        }
        tk->raw_time.tv_nsec = tk->tkr_raw.xtime_nsec >> tk->tkr_raw.shift;
        tk->tkr_raw.xtime_nsec -= (u64)tk->raw_time.tv_nsec << tk->tkr_raw.shift;

        /* Accumulate error between NTP and clock interval */
        tk->ntp_error += tk->ntp_tick << shift;
        tk->ntp_error -= (tk->xtime_interval + tk->xtime_remainder) <<
                                                (tk->ntp_error_shift + shift);

        return offset;
}
```

另一个例程`update_wall_time()`，在`kernel/time/timekeeping.c`中定义，负责维护壁钟时间。它使用当前时钟源作为参考递增壁钟时间。

# 时钟中断处理

为了提供编程接口，生成滴答的时钟设备通过`include/linux/clockchips.h`中定义的`struct clock_event_device`结构进行抽象：

```
struct clock_event_device {
        void                    (*event_handler)(struct clock_event_device *);
        int                     (*set_next_event)(unsigned long evt, struct clock_event_device *);
        int                     (*set_next_ktime)(ktime_t expires, struct clock_event_device *);
        ktime_t                  next_event;
        u64                      max_delta_ns;
        u64                      min_delta_ns;
        u32                      mult;
        u32                      shift;
        enum clock_event_state    state_use_accessors;
        unsigned int            features;
        unsigned long           retries;

        int                     (*set_state_periodic)(struct  clock_event_device *);
        int                     (*set_state_oneshot)(struct clock_event_device *);
        int                     (*set_state_oneshot_stopped)(struct clock_event_device *);
        int                     (*set_state_shutdown)(struct clock_event_device *);
        int                     (*tick_resume)(struct clock_event_device *);

        void                    (*broadcast)(const struct cpumask *mask);
        void                    (*suspend)(struct clock_event_device *);
        void                    (*resume)(struct clock_event_device *);
        unsigned long           min_delta_ticks;
        unsigned long           max_delta_ticks;

        const char               *name;
        int                     rating;
        int                     irq;
        int                     bound_on;
        const struct cpumask       *cpumask;
        struct list_head  list;
        struct module             *owner;
} ____cacheline_aligned;
```

在这里，`event_handler`是由框架分配的适当例程，由低级处理程序调用以运行滴答。根据配置，这个`clock_event_device`可以是`periodic`、`one-shot`或`ktime`基础的。在这三种情况中，滴答设备的适当操作模式是通过`unsigned int features`字段设置的，使用这些宏之一：

```
#define CLOCK_EVT_FEAT_PERIODIC 0x000001
#define CLOCK_EVT_FEAT_ONESHOT 0x000002
#define CLOCK_EVT_FEAT_KTIME  0x000004
```

周期模式配置硬件每*1/HZ*秒生成一次滴答，而单次模式使硬件在当前时间后经过特定数量的周期生成滴答。

根据用例和操作模式，event_handler 可以是这三个例程中的任何一个：

+   `tick_handle_periodic()`是周期性滴答的默认处理程序，定义在`kernel/time/tick-common.c`中。

+   `tick_nohz_handler()`是低分辨率中断处理程序，在低分辨率模式下使用。它在`kernel/time/tick-sched.c`中定义。

+   `hrtimer_interrupt()`在高分辨率模式下使用，并在调用时禁用中断。

通过`clockevents_config_and_register()`例程配置和注册时钟事件设备，定义在`kernel/time/clockevents.c`中。

# 滴答设备

`clock_event_device`抽象是为了核心定时框架；我们需要一个单独的抽象来处理每个 CPU 的滴答设备；这是通过`struct tick_device`结构和`DEFINE_PER_CPU()`宏来实现的，分别在`kernel/time/tick-sched.h`和`include/linux/percpu-defs.h`中定义：

```
enum tick_device_mode {
 TICKDEV_MODE_PERIODIC,
 TICKDEV_MODE_ONESHOT,
};

struct tick_device {
        struct clock_event_device *evtdev;
        enum tick_device_mode mode;
}
```

`tick_device`可以是周期性的或单次的。它通过`enum tick_device_mode`设置。

# 软件定时器和延迟函数

软件定时器允许在时间到期时调用函数。有两种类型的定时器：内核使用的动态定时器和用户空间进程使用的间隔定时器。除了软件定时器，还有另一种常用的定时函数称为延迟函数。延迟函数实现一个精确的循环，根据延迟函数的参数执行（通常是多次）。

# 动态定时器

动态定时器可以随时创建和销毁，因此称为动态定时器。动态定时器由`struct timer_list`对象表示，定义在`include/linux/timer.h`中：

```
struct timer_list {
        /*
        * Every field that changes during normal runtime grouped to the
        * same cacheline
        */
        struct hlist_node entry;
        unsigned long           expires;
        void                    (*function)(unsigned long);
        unsigned long           data;
        u32                      flags;

#ifdef CONFIG_LOCKDEP
        struct lockdep_map        lockdep_map;
#endif
};
```

系统中的所有定时器都由一个双向链表管理，并按照它们的到期时间排序，由 expires 字段表示。expires 字段指定定时器到期后的时间。一旦当前的`jiffies`值匹配或超过此字段的值，定时器就会过期。通过 entry 字段，定时器被添加到此定时器链表中。函数字段指向在定时器到期时要调用的例程，数据字段保存要传递给函数的参数（如果需要）。expires 字段不断与`jiffies_64`值进行比较，以确定定时器是否已经过期。

动态定时器可以按以下方式创建和激活：

+   创建一个新的`timer_list`对象，比如说`t_obj`。

+   使用宏`init_timer(&t_obj)`初始化此定时器对象，定义在`include/linux/timer.h`中。

+   使用函数字段初始化函数的地址，以在定时器到期时调用该函数。如果函数需要参数，则也初始化数据字段。

+   如果定时器对象已经添加到定时器列表中，则通过调用函数`mod_timer(&t_obj, <timeout-value-in-jiffies>)`更新 expires 字段，定义在`kernel/time/timer.c`中。

+   如果没有，初始化 expires 字段，并使用`add_timer(&t_obj)`将定时器对象添加到定时器列表中，定义在`/kernel/time/timer.c`中。

内核会自动从定时器列表中删除已过期的定时器，但也有其他方法可以从列表中删除定时器。`kernel/time/timer.c`中定义的`del_timer()`和`del_timer_sync()`例程以及宏`del_singleshot_timer_sync()`可以帮助实现这一点：

```
int del_timer(struct timer_list *timer)
{
        struct tvec_base *base;
        unsigned long flags;
        int ret = 0;

        debug_assert_init(timer);

        timer_stats_timer_clear_start_info(timer);
        if (timer_pending(timer)) {
                base = lock_timer_base(timer, &flags);
                if (timer_pending(timer)) {
                        detach_timer(timer, 1);
                        if (timer->expires == base->next_timer &&
                            !tbase_get_deferrable(timer->base))
                                base->next_timer = base->timer_jiffies;
                        ret = 1;
                }
                spin_unlock_irqrestore(&base->lock, flags);
        }

        return ret;
}

int del_timer_sync(struct timer_list *timer)
{
#ifdef CONFIG_LOCKDEP
        unsigned long flags;

        /*
        * If lockdep gives a backtrace here, please reference
        * the synchronization rules above.
        */
        local_irq_save(flags);
        lock_map_acquire(&timer->lockdep_map);
        lock_map_release(&timer->lockdep_map);
        local_irq_restore(flags);
#endif
        /*
        * don't use it in hardirq context, because it
        * could lead to deadlock.
        */
        WARN_ON(in_irq());
        for (;;) {
                int ret = try_to_del_timer_sync(timer);
                if (ret >= 0)
                        return ret;
                cpu_relax();
        }
}

#define del_singleshot_timer_sync(t) del_timer_sync(t)
```

`del_timer()` 删除活动和非活动的定时器。在 SMP 系统中特别有用，`del_timer_sync()` 会停止定时器，并等待处理程序在其他 CPU 上执行完成。

# 动态定时器的竞争条件

```
RESOURCE_DEALLOCATE() here could be any relevant resource deallocation routine:
```

```
...
del_timer(&t_obj);
RESOURCE_DEALLOCATE();
....
```

然而，这种方法仅适用于单处理器系统。在 SMP 系统中，当定时器停止时，其功能可能已经在另一个 CPU 上运行。在这种情况下，资源将在`del_timer()`返回时立即释放，而定时器功能仍在其他 CPU 上操作它们；这绝非理想的情况。`del_timer_sync()`解决了这个问题：在停止定时器后，它会等待定时器功能在其他 CPU 上执行完成。`del_timer_sync()`在定时器功能可以重新激活自身的情况下非常有用。如果定时器功能不重新激活定时器，则应该使用一个更简单和更快的宏`del_singleshot_timer_sync()`。

# 动态定时器处理

软件定时器复杂且耗时，因此不应由定时器 ISR 处理。而应该由一个可延迟的底半软中断例程`TIMER_SOFTIRQ`来执行，其例程在`kernel/time/timer.c`中定义：

```
static __latent_entropy void run_timer_softirq(struct softirq_action *h)
{
        struct timer_base *base = this_cpu_ptr(&timer_bases[BASE_STD]);

        base->must_forward_clk = false;

        __run_timers(base);
        if (IS_ENABLED(CONFIG_NO_HZ_COMMON) && base->nohz_active)
                __run_timers(this_cpu_ptr(&timer_bases[BASE_DEF]));
}
```

# 延迟函数

定时器在超时期相对较长时非常有用；在所有其他需要较短持续时间的用例中，使用延迟函数。在处理诸如存储设备（即*闪存*和*EEPROM*）等硬件时，设备驱动程序非常关键，需要等待设备完成写入和擦除等硬件操作，这在大多数情况下是在几微秒到毫秒的范围内。在不等待硬件完成这些操作的情况下继续执行其他指令将导致不可预测的读/写操作和数据损坏。在这种情况下，延迟函数非常有用。内核通过`ndelay()`、`udelay()`和`mdelay()`例程和宏提供这样的短延迟，分别接收纳秒、微秒和毫秒为参数。

以下函数可以在`include/linux/delay.h`中找到：

```
static inline void ndelay(unsigned long x)
{
        udelay(DIV_ROUND_UP(x, 1000));
}
```

这些函数可以在`arch/ia64/kernel/time.c`中找到：

```
static void
ia64_itc_udelay (unsigned long usecs)
{
        unsigned long start = ia64_get_itc();
        unsigned long end = start + usecs*local_cpu_data->cyc_per_usec;

        while (time_before(ia64_get_itc(), end))
                cpu_relax();
}

void (*ia64_udelay)(unsigned long usecs) = &ia64_itc_udelay;

void
udelay (unsigned long usecs)
{
        (*ia64_udelay)(usecs);
}
```

# POSIX 时钟

POSIX 为多线程和实时用户空间应用程序提供了软件定时器，称为 POSIX 定时器。POSIX 提供以下时钟：

+   `CLOCK_REALTIME`：该时钟表示系统中的实时时间。也称为墙上时间，类似于挂钟上的时间，用于时间戳和向用户提供实际时间。该时钟是可修改的。

+   `CLOCK_MONOTONIC`：该时钟保持系统启动以来经过的时间。它是不断增加的，并且不可被任何进程或用户修改。由于其单调性质，它是确定两个时间事件之间时间差的首选时钟。

+   `CLOCK_BOOTTIME`：该时钟与`CLOCK_MONOTONIC`相同；但它包括在挂起中花费的时间。

这些时钟可以通过以下 POSIX 时钟例程进行访问和修改（如果所选时钟允许）：

+   `int clock_getres(clockid_t clk_id, struct timespec *res);`

+   `int clock_gettime(clockid_t clk_id, struct timespec *tp);`

+   `int clock_settime(clockid_t clk_id, const struct timespec *tp);`

函数 `clock_getres()` 获取由 *clk_id* 指定的时钟的分辨率（精度）。如果分辨率非空，则将其存储在由分辨率指向的 `struct timespec` 中。函数 `clock_gettime()` 和 `clock_settime()` 读取和设置由 *clk_id* 指定的时钟的时间。*clk_id* 可以是任何 POSIX 时钟：`CLOCK_REALTIME`，`CLOCK_MONOTONIC` 等等。

`CLOCK_REALTIME_COARSE`

`CLOCK_MONOTONIC_COARSE`

每个这些 POSIX 例程都有相应的系统调用，即 `sys_clock_getres()，sys_clock_gettime()` 和 `sys_clock_settime`*.* 因此，每次调用这些例程时，都会发生从用户模式到内核模式的上下文切换。如果对这些例程的调用频繁，上下文切换可能会导致系统性能下降。为了避免上下文切换，POSIX 时钟的两个粗糙变体被实现为 vDSO（虚拟动态共享对象）库：

vDSO 是一个小型共享库，其中包含内核空间的选定例程，内核将其映射到用户空间应用程序的地址空间中，以便这些内核空间例程可以直接由它们在用户空间中的进程调用。C 库调用 vDSO，因此用户空间应用程序可以通过标准函数以通常的方式进行编程，并且 C 库将利用通过 vDSO 可用的功能，而不涉及任何系统调用接口，从而避免任何用户模式-内核模式上下文切换和系统调用开销。作为 vDSO 实现，这些粗糙的变体速度更快，分辨率为 1 毫秒。

# 总结

在本章中，我们详细了解了内核提供的大多数用于驱动基于时间的事件的例程，以及理解了 Linux 时间、其基础设施和其测量的基本方面。我们还简要介绍了 POSIX 时钟及其一些关键的时间访问和修改例程。然而，有效的时间驱动程序取决于对这些例程的谨慎和计算使用。

在下一章中，我们将简要介绍动态内核模块的管理。
