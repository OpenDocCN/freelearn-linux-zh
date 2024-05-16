# 第三章：信号管理

信号提供了一个基本的基础设施，任何进程都可以异步地被通知系统事件。它们也可以作为进程之间的通信机制。了解内核如何提供和管理整个信号处理机制的平稳吞吐量，让我们对内核有更深入的了解。在本章中，我们将从进程如何引导信号到内核如何巧妙地管理例程以确保信号事件的发生，深入研究以下主题：

+   信号概述及其类型

+   进程级别的信号管理调用

+   进程描述符中的信号数据结构

+   内核的信号生成和传递机制

# 信号

**信号**是传递给进程或进程组的短消息。内核使用信号通知进程系统事件的发生；信号也用于进程之间的通信。Linux 将信号分为两组，即通用 POSIX（经典 Unix 信号）和实时信号。每个组包含 32 个不同的信号，由唯一的 ID 标识：

```
#define _NSIG 64
#define _NSIG_BPW __BITS_PER_LONG
#define _NSIG_WORDS (_NSIG / _NSIG_BPW)

#define SIGHUP 1
#define SIGINT 2
#define SIGQUIT 3
#define SIGILL 4
#define SIGTRAP 5
#define SIGABRT 6
#define SIGIOT 6
#define SIGBUS 7
#define SIGFPE 8
#define SIGKILL 9
#define SIGUSR1 10
#define SIGSEGV 11
#define SIGUSR2 12
#define SIGPIPE 13
#define SIGALRM 14
#define SIGTERM 15
#define SIGSTKFLT 16
#define SIGCHLD 17
#define SIGCONT 18
#define SIGSTOP 19
#define SIGTSTP 20
#define SIGTTIN 21
#define SIGTTOU 22
#define SIGURG 23
#define SIGXCPU 24
#define SIGXFSZ 25
#define SIGVTALRM 26
#define SIGPROF 27
#define SIGWINCH 28
#define SIGIO 29
#define SIGPOLL SIGIO
/*
#define SIGLOST 29
*/
#define SIGPWR 30
#define SIGSYS 31
#define SIGUNUSED 31

/* These should not be considered constants from userland. */
#define SIGRTMIN 32
#ifndef SIGRTMAX
#define SIGRTMAX _NSIG
#endif
```

通用 POSIX 类别中的信号与特定系统事件绑定，并通过宏适当命名。实时类别中的信号不与特定事件绑定，可以自由用于进程通信；内核用通用名称引用它们：`SIGRTMIN` 和 `SIGRTMAX`。

在生成信号时，内核将信号事件传递给目标进程，目标进程可以根据配置的操作（称为**信号处理方式**）对信号做出响应。

以下是进程可以设置为其信号处理方式的操作列表。进程可以在某个时间点设置任何一个操作为其信号处理方式，但可以在没有任何限制的情况下在这些操作之间任意切换任意次数。

+   **内核处理程序**: 内核为每个信号实现了默认处理程序。这些处理程序通过任务结构的信号处理程序表对进程可用。收到信号后，进程可以请求执行适当的信号处理程序。这是默认的处理方式。

+   **进程定义的处理程序:** 进程允许实现自己的信号处理程序，并设置它们以响应信号事件的执行。这是通过适当的系统调用接口实现的，允许进程将其处理程序例程与信号绑定。在发生信号时，进程处理程序将被异步调用。

+   **忽略:** 进程也可以忽略信号的发生，但需要通过调用适当的系统调用宣布其忽略意图。

内核定义的默认处理程序例程可以执行以下任何操作：

+   **Ignore**: 什么都不会发生。

+   **终止**: 终止进程，即组中的所有线程（类似于 `exit_group`）。组长（仅）向其父进程报告 `WIFSIGNALED` 状态。

+   **Coredump**: 写入描述使用相同 `mm` 的所有线程的核心转储文件，然后终止所有这些线程

+   **停止**: 停止组中的所有线程，即 `TASK_STOPPED` 状态。

以下是总结表，列出了默认处理程序执行的操作：

```
 +--------------------+------------------+
 * | POSIX signal     | default action |
 * +------------------+------------------+
 * | SIGHUP           | terminate 
 * | SIGINT           | terminate 
 * | SIGQUIT          | coredump 
 * | SIGILL           | coredump 
 * | SIGTRAP          | coredump 
 * | SIGABRT/SIGIOT   | coredump 
 * | SIGBUS           | coredump 
 * | SIGFPE           | coredump 
 * | SIGKILL          | terminate
 * | SIGUSR1          | terminate 
 * | SIGSEGV          | coredump 
 * | SIGUSR2          | terminate
 * | SIGPIPE          | terminate 
 * | SIGALRM          | terminate 
 * | SIGTERM          | terminate 
 * | SIGCHLD          | ignore 
 * | SIGCONT          | ignore 
 * | SIGSTOP          | stop
 * | SIGTSTP          | stop
 * | SIGTTIN          | stop
 * | SIGTTOU          | stop
 * | SIGURG           | ignore 
 * | SIGXCPU          | coredump 
 * | SIGXFSZ          | coredump 
 * | SIGVTALRM        | terminate 
 * | SIGPROF          | terminate 
 * | SIGPOLL/SIGIO    | terminate 
 * | SIGSYS/SIGUNUSED | coredump 
 * | SIGSTKFLT        | terminate 
 * | SIGWINCH         | ignore 
 * | SIGPWR           | terminate 
 * | SIGRTMIN-SIGRTMAX| terminate 
 * +------------------+------------------+
 * | non-POSIX signal | default action |
 * +------------------+------------------+
 * | SIGEMT           | coredump |
 * +--------------------+------------------+
```

# 信号管理 API

应用程序提供了各种 API 用于管理信号；我们将看一下其中一些重要的 API：

1.  `Sigaction()`: 用户模式进程使用 POSIX API `sigaction()` 来检查或更改信号的处理方式。该 API 提供了各种属性标志，可以进一步定义信号的行为：

```
 #include <signal.h>
 int sigaction(int signum, const struct sigaction *act, struct sigaction *oldact);

 The sigaction structure is defined as something like:

 struct sigaction {
 void (*sa_handler)(int);
 void (*sa_sigaction)(int, siginfo_t *, void *);
 sigset_t sa_mask;
 int sa_flags;
 void (*sa_restorer)(void);
 };
```

+   `int signum` 是已识别的 `signal` 的标识号。`sigaction()` 检查并设置与该信号关联的操作。

+   `const struct sigaction *act`可以被赋予一个`struct sigaction`实例的地址。在此结构中指定的操作成为与信号绑定的新操作。当*act*指针未初始化（NULL）时，当前的处理方式不会改变。

+   `struct sigaction *oldact`是一个 outparam，需要用未初始化的`sigaction`实例的地址进行初始化；`sigaction()`通过此参数返回当前与信号关联的操作。

+   以下是各种`flag`选项：

+   `SA_NOCLDSTOP`：此标志仅在绑定`SIGCHLD`的处理程序时相关。它用于禁用对子进程停止（`SIGSTP`）和恢复（`SIGCONT`）事件的`SIGCHLD`通知。

+   `SA_NOCLDWAIT`：此标志仅在绑定`SIGCHLD`的处理程序或将其设置为`SIG_DFL`时相关。设置此标志会导致子进程在终止时立即被销毁，而不是处于*僵尸*状态。

+   `SA_NODEFER`：设置此标志会导致生成的信号即使相应的处理程序正在执行也会被传递。

+   `SA_ONSTACK`：此标志仅在绑定信号处理程序时相关。设置此标志会导致信号处理程序使用备用堆栈；备用堆栈必须由调用进程通过`sigaltstack()`API 设置。如果没有备用堆栈，处理程序将在当前堆栈上被调用。

+   `SA_RESETHAND`：当与`sigaction()`一起应用此标志时，它使信号处理程序成为一次性的，也就是说，指定信号的操作对于该信号的后续发生被重置为`SIG_DFL`。

+   `SA_RESTART`：此标志使系统调用操作被当前信号处理程序中断后重新进入。

+   `SA_SIGINFO`：此标志用于向系统指示信号处理程序已分配--`sigaction`结构的`sa_sigaction`指针而不是`sa_handler`。分配给`sa_sigaction`的处理程序接收两个额外的参数：

```
      void handler_fn(int signo, siginfo_t *info, void *context);
```

第一个参数是`signum`，处理程序绑定的信号。第二个参数是一个 outparam，是指向`siginfo_t`类型对象的指针，提供有关信号来源的附加信息。以下是`siginfo_t`的完整定义：

```
 siginfo_t {
 int si_signo; /* Signal number */
 int si_errno; /* An errno value */
 int si_code; /* Signal code */
 int si_trapno; /* Trap number that caused hardware-generated signal (unused on most           architectures) */
 pid_t si_pid; /* Sending process ID */
 uid_t si_uid; /* Real user ID of sending process */
 int si_status; /* Exit value or signal */
 clock_t si_utime; /* User time consumed */
 clock_t si_stime; /* System time consumed */
 sigval_t si_value; /* Signal value */
 int si_int; /* POSIX.1b signal */
 void *si_ptr; /* POSIX.1b signal */
 int si_overrun; /* Timer overrun count; POSIX.1b timers */
 int si_timerid; /* Timer ID; POSIX.1b timers */
 void *si_addr; /* Memory location which caused fault */
 long si_band; /* Band event (was int in glibc 2.3.2 and earlier) */
 int si_fd; /* File descriptor */
 short si_addr_lsb; /* Least significant bit of address (since Linux 2.6.32) */
 void *si_call_addr; /* Address of system call instruction (since Linux 3.5) */
 int si_syscall; /* Number of attempted system call (since Linux 3.5) */
 unsigned int si_arch; /* Architecture of attempted system call (since Linux 3.5) */
 }
```

1.  `Sigprocmask()`：除了改变信号处理程序外，该处理程序还允许阻止或解除阻止信号传递。应用程序可能需要在执行关键代码块时进行这些操作，以防止被异步信号处理程序抢占。例如，网络通信应用程序可能不希望在进入启动与其对等体连接的代码块时处理信号：

+   `sigprocmask()`是一个 POSIX API，用于检查、阻塞和解除阻塞信号。

```
    int sigprocmask(int how, const sigset_t *set, sigset_t *oldset); 
```

任何被阻止的信号发生都会排队在每个进程的挂起信号列表中。挂起队列设计用于保存一个被阻止的通用信号的发生，同时排队每个实时信号的发生。用户模式进程可以使用`sigpending()`和`rt_sigpending()`API 来查询挂起信号。这些例程将挂起信号的列表返回到由`sigset_t`指针指向的实例中。

```
    int sigpending(sigset_t *set);
```

这些操作适用于除了`SIGKILL`和`SIGSTOP`之外的所有信号；换句话说，进程不允许改变默认的处理方式或阻止`SIGSTOP`和`SIGKILL`信号。

# 从程序中引发信号

`kill()`和`sigqueue()`是 POSIX API，通过它们，一个进程可以为另一个进程或进程组引发信号。这些 API 促进了信号作为**进程通信**机制的利用：

```
 int kill(pid_t pid, int sig);
 int sigqueue(pid_t pid, int sig, const union sigval value);

 union sigval {
 int sival_int;
 void *sival_ptr;
 };
```

虽然这两个 API 都提供了参数来指定接收者的`PID`和要提升的`signum`，`sigqueue()`通过一个额外的参数（联合信号）提供了*数据*可以与信号一起发送到接收进程。目标进程可以通过`struct siginfo_t`（`si_value`）实例访问数据。Linux 通过本机 API 扩展了这些函数，可以将信号排队到线程组，甚至到线程组中的轻量级进程（LWP）：

```
/* queue signal to specific thread in a thread group */
int tgkill(int tgid, int tid, int sig);

/* queue signal and data to a thread group */
int rt_sigqueueinfo(pid_t tgid, int sig, siginfo_t *uinfo);

/* queue signal and data to specific thread in a thread group */
int rt_tgsigqueueinfo(pid_t tgid, pid_t tid, int sig, siginfo_t *uinfo);

```

# 等待排队信号

在应用信号进行进程通信时，对于进程来说，暂停自身直到发生特定信号，然后在来自另一个进程的信号到达时恢复执行可能更合适。POSIX 调用`sigsuspend()`、`sigwaitinfo()`和`sigtimedwait()`提供了这种功能：

```
int sigsuspend(const sigset_t *mask);
int sigwaitinfo(const sigset_t *set, siginfo_t *info);
int sigtimedwait(const sigset_t *set, siginfo_t *info, const struct timespec *timeout);
```

虽然所有这些 API 允许进程等待指定的信号发生，`sigwaitinfo()`通过`info`指针返回的`siginfo_t`实例提供有关信号的附加数据。`sigtimedwait()`通过提供一个额外的参数扩展了功能，允许操作超时，使其成为一个有界等待调用。Linux 内核提供了一个替代 API，允许进程通过名为`signalfd()`的特殊文件描述符被通知信号的发生：

```
 #include <sys/signalfd.h>
 int signalfd(int fd, const sigset_t *mask, int flags);
```

成功时，`signalfd()`返回一个文件描述符，进程需要调用`read()`来阻塞，直到掩码中指定的任何信号发生。

# 信号数据结构

内核维护每个进程的信号数据结构，以跟踪*信号处理*、*阻塞信号*和*待处理信号队列*。进程任务结构包含对这些数据结构的适当引用：

```
struct task_struct {

....
....
....
/* signal handlers */
 struct signal_struct *signal;
 struct sighand_struct *sighand;

 sigset_t blocked, real_blocked;
 sigset_t saved_sigmask; /* restored if set_restore_sigmask() was used */
 struct sigpending pending;

 unsigned long sas_ss_sp;
 size_t sas_ss_size;
 unsigned sas_ss_flags;
  ....
  ....
  ....
  ....

};
```

# 信号描述符

回顾一下我们在第一章的早期讨论中提到的，Linux 通过轻量级进程支持多线程应用程序。线程应用程序的所有 LWP 都是*进程组*的一部分，并共享信号处理程序；每个 LWP（线程）维护自己的待处理和阻塞信号队列。

任务结构的**signal**指针指向`signal_struct`类型的实例，这是信号描述符。这个结构被线程组的所有 LWP 共享，并维护诸如共享待处理信号队列（对于排队到线程组的信号）之类的元素，这对进程组中的所有线程都是共同的。

以下图表示维护共享待处理信号所涉及的数据结构：

![](img/00014.jpeg)

以下是`signal_struct`的一些重要字段：

```
struct signal_struct {
 atomic_t sigcnt;
 atomic_t live;
 int nr_threads;
 struct list_head thread_head;

 wait_queue_head_t wait_chldexit; /* for wait4() */

 /* current thread group signal load-balancing target: */
 struct task_struct *curr_target;

 /* shared signal handling: */
 struct sigpending shared_pending; 
 /* thread group exit support */
 int group_exit_code;
 /* overloaded:
 * - notify group_exit_task when ->count is equal to notify_count
 * - everyone except group_exit_task is stopped during signal delivery
 * of fatal signals, group_exit_task processes the signal.
 */
 int notify_count;
 struct task_struct *group_exit_task;

 /* thread group stop support, overloads group_exit_code too */
 int group_stop_count;
 unsigned int flags; /* see SIGNAL_* flags below */

```

# 阻塞和待处理队列

任务结构中的`blocked`和`real_blocked`实例是被阻塞信号的位掩码；这些队列是每个进程的。线程组中的每个 LWP 都有自己的阻塞信号掩码。任务结构的`pending`实例用于排队私有待处理信号；所有排队到普通进程和线程组中特定 LWP 的信号都排队到这个列表中：

```
struct sigpending {
 struct list_head list; // head to double linked list of struct sigqueue
 sigset_t signal; // bit mask of pending signals
};
```

以下图表示维护私有待处理信号所涉及的数据结构：

![](img/00015.jpeg)

# 信号处理程序描述符

任务结构的`sighand`指针指向`struct sighand_struct`的一个实例，这是线程组中所有进程共享的信号处理程序描述符。这个结构也被所有使用`clone()`和`CLONE_SIGHAND`标志创建的进程共享。这个结构包含一个`k_sigaction`实例的数组，每个实例包装一个`sigaction`的实例，描述了每个信号的当前处理方式：

```
struct k_sigaction {
 struct sigaction sa;
#ifdef __ARCH_HAS_KA_RESTORER 
 __sigrestore_t ka_restorer;
#endif
};

struct sighand_struct {
 atomic_t count;
 struct k_sigaction action[_NSIG];
 spinlock_t siglock;
 wait_queue_head_t signalfd_wqh;
};

```

以下图表示信号处理程序描述符：

![](img/00016.jpeg)

# 信号生成和传递

当发生信号时，将其加入到接收进程或进程的任务结构中的挂起信号列表中。信号是在用户模式进程、内核或任何内核服务的请求下生成的（对于进程或组）。当接收进程或进程意识到其发生并被强制执行适当的响应处理程序时，信号被认为是**已传递**；换句话说，信号传递等同于相应处理程序的初始化。理想情况下，每个生成的信号都被假定立即传递；然而，存在信号生成和最终传递之间的延迟可能性。为了便于可能的延迟传递，内核为信号生成和传递提供了单独的函数。

# 信号生成调用

内核为信号生成提供了两组不同的函数：一组用于在单个进程上生成信号，另一组用于进程线程组。

+   以下是生成进程信号的重要函数列表：

`send_sig()`: 在进程上生成指定信号；这个函数被内核服务广泛使用

+   `end_sig_info()`: 用额外的`siginfo_t`实例扩展`send_sig()`

+   `force_sig()`: 用于生成无法被忽略或阻止的优先级非可屏蔽信号

+   `force_sig_info()`: 用额外的`siginfo_t`实例扩展`force_sig()`

所有这些例程最终调用核心内核函数`send_signal()`，该函数被设计用于生成指定的信号。

以下是生成进程组信号的重要函数列表：

+   `kill_pgrp()`: 在进程组中的所有线程组上生成指定信号

+   `kill_pid()`: 向由 PID 标识的线程组生成指定信号

+   `kill_pid_info()`: 用额外的`siginfo_t`实例扩展`kill_pid()`

所有这些例程调用一个名为`group_send_sig_info()`的函数，最终使用适当的参数调用`send_signal()`。

`send_signal()`函数是核心信号生成函数；它使用适当的参数调用`__send_signal()`例程：

```
 static int send_signal(int sig, struct siginfo *info, struct task_struct *t,
 int group)
{
 int from_ancestor_ns = 0;

#ifdef CONFIG_PID_NS
 from_ancestor_ns = si_fromuser(info) &&
 !task_pid_nr_ns(current, task_active_pid_ns(t));
#endif

 return __send_signal(sig, info, t, group, from_ancestor_ns);
}
```

以下是`__send_signal()`执行的重要步骤：

1.  从`info`参数中检查信号的来源。如果信号生成是由内核发起的，对于不可屏蔽的`SIGKILL`或`SIGSTOP`，它立即设置适当的 sigpending 位，设置`TIF_SIGPENDING`标志，并通过唤醒目标线程启动传递过程：

```
 /*
 * fast-pathed signals for kernel-internal things like SIGSTOP
 * or SIGKILL.
 */
 if (info == SEND_SIG_FORCED)
 goto out_set;
....
....
....
out_set:
 signalfd_notify(t, sig);
 sigaddset(&pending->signal, sig);
 complete_signal(sig, t, group);

```

1.  调用`__sigqeueue_alloc()`函数，检查接收进程的挂起信号数量是否小于资源限制。如果是，则增加挂起信号计数器并返回`struct sigqueue`实例的地址：

```
 q = __sigqueue_alloc(sig, t, GFP_ATOMIC | __GFP_NOTRACK_FALSE_POSITIVE,
 override_rlimit);
```

1.  将`sigqueue`实例加入到挂起列表中，并将信号信息填入`siginfo_t`：

```
if (q) {
 list_add_tail(&q->list, &pending->list);
 switch ((unsigned long) info) {
 case (unsigned long) SEND_SIG_NOINFO:
       q->info.si_signo = sig;
       q->info.si_errno = 0;
       q->info.si_code = SI_USER;
       q->info.si_pid = task_tgid_nr_ns(current,
       task_active_pid_ns(t));
       q->info.si_uid = from_kuid_munged(current_user_ns(), current_uid());
       break;
 case (unsigned long) SEND_SIG_PRIV:
       q->info.si_signo = sig;
       q->info.si_errno = 0;
       q->info.si_code = SI_KERNEL;
       q->info.si_pid = 0;
       q->info.si_uid = 0;
       break;
 default:
      copy_siginfo(&q->info, info);
      if (from_ancestor_ns)
      q->info.si_pid = 0;
      break;
 }

```

1.  在挂起信号的位掩码中设置适当的信号位，并通过调用`complete_signal()`尝试信号传递，进而设置`TIF_SIGPENDING`标志：

```
 sigaddset(&pending->signal, sig);
 complete_signal(sig, t, group);
```

# 信号传递

信号通过更新接收器任务结构中的适当条目生成后，内核进入传递模式。如果接收进程在 CPU 上并且未阻止指定的信号，则立即传递信号。即使接收方不在 CPU 上，也会传递优先级信号`SIGSTOP`和`SIGKILL`，通过唤醒进程；然而，对于其余的信号，传递将推迟直到进程准备好接收信号。为了便于推迟传递，内核在从中断和系统调用返回时检查进程的非阻塞挂起信号，然后允许进程恢复用户模式执行。当进程调度程序（在从中断和异常返回时调用）发现`TIF_SIGPENDING`标志设置时，它调用内核函数`do_signal()`来启动挂起信号的传递，然后恢复进程的用户模式上下文。

进入内核模式时，进程的用户模式寄存器状态存储在称为`pt_regs`的进程内核堆栈中（特定于体系结构）：

```
 struct pt_regs {
/*
 * C ABI says these regs are callee-preserved. They aren't saved on kernel entry
 * unless syscall needs a complete, fully filled "struct pt_regs".
 */
 unsigned long r15;
 unsigned long r14;
 unsigned long r13;
 unsigned long r12;
 unsigned long rbp;
 unsigned long rbx;
/* These regs are callee-clobbered. Always saved on kernel entry. */
 unsigned long r11;
 unsigned long r10;
 unsigned long r9;
 unsigned long r8;
 unsigned long rax;
 unsigned long rcx;
 unsigned long rdx;
 unsigned long rsi;
 unsigned long rdi;
/*
 * On syscall entry, this is syscall#. On CPU exception, this is error code.
 * On hw interrupt, it's IRQ number:
 */
 unsigned long orig_rax;
/* Return frame for iretq */
 unsigned long rip;
 unsigned long cs;
 unsigned long eflags;
 unsigned long rsp;
 unsigned long ss;
/* top of stack page */
};
```

`do_signal()`例程在内核堆栈中使用`pt_regs`的地址调用。虽然`do_signal()`旨在传递非阻塞的挂起信号，但其实现是特定于体系结构的。

以下是`do_signal()`的 x86 版本：

```
void do_signal(struct pt_regs *regs)
{
 struct ksignal ksig;
 if (get_signal(&ksig)) {
 /* Whee! Actually deliver the signal. */
 handle_signal(&ksig, regs);
 return;
 }
 /* Did we come from a system call? */
 if (syscall_get_nr(current, regs) >= 0) {
 /* Restart the system call - no handlers present */
 switch (syscall_get_error(current, regs)) {
 case -ERESTARTNOHAND:
 case -ERESTARTSYS:
 case -ERESTARTNOINTR:
 regs->ax = regs->orig_ax;
 regs->ip -= 2;
 break;
 case -ERESTART_RESTARTBLOCK:
 regs->ax = get_nr_restart_syscall(regs);
 regs->ip -= 2;
 break;
 }
 }
 /*
 * If there's no signal to deliver, we just put the saved sigmask
 * back.
 */
 restore_saved_sigmask();
}
```

`do_signal()`使用`struct ksignal`类型实例的地址调用`get_signal()`函数（我们将简要考虑此例程的重要步骤，跳过其他细节）。此函数包含一个循环，它调用`dequeue_signal()`直到从私有和共享挂起列表中取出所有非阻塞的挂起信号。它从最低编号的信号开始查找私有挂起信号队列，然后进入共享队列中的挂起信号，然后更新数据结构以指示该信号不再挂起并返回其编号：

```
 signr = dequeue_signal(current, &current->blocked, &ksig->info);
```

对于`dequeue_signal()`返回的每个挂起信号，`get_signal()`通过`struct ksigaction *ka`类型的指针检索当前的信号处理方式：

```
ka = &sighand->action[signr-1]; 
```

如果信号处理方式设置为`SIG_IGN`，则静默忽略当前信号并继续迭代以检索另一个挂起信号：

```
if (ka->sa.sa_handler == SIG_IGN) /* Do nothing. */
 continue;
```

如果处理方式不等于`SIG_DFL`，则检索**sigaction**的地址并将其初始化为参数`ksig->ka`，以便进一步执行用户模式处理程序。它进一步检查用户的**sigaction**中的`SA_ONESHOT (SA_RESETHAND)`标志，如果设置，则将信号处理方式重置为`SIG_DFL`，跳出循环并返回给调用者。`do_signal()`现在调用`handle_signal()`例程来执行用户模式处理程序（我们将在下一节详细讨论这个）。

```
  if (ka->sa.sa_handler != SIG_DFL) {
 /* Run the handler. */
 ksig->ka = *ka;

 if (ka->sa.sa_flags & SA_ONESHOT)
 ka->sa.sa_handler = SIG_DFL;

 break; /* will return non-zero "signr" value */
 }
```

如果处理方式设置为`SIG_DFL`，则调用一组宏来检查内核处理程序的默认操作。可能的默认操作是：

+   **Term**：默认操作是终止进程

+   **Ign**：默认操作是忽略信号

+   **Core**：默认操作是终止进程并转储核心

+   **Stop**：默认操作是停止进程

+   **Cont**：默认操作是如果当前停止则继续进程

```
get_signal() that initiates the default action as per the set disposition:
```

```
/*
 * Now we are doing the default action for this signal.
 */
 if (sig_kernel_ignore(signr)) /* Default is nothing. */
 continue;

 /*
 * Global init gets no signals it doesn't want.
 * Container-init gets no signals it doesn't want from same
 * container.
 *
 * Note that if global/container-init sees a sig_kernel_only()
 * signal here, the signal must have been generated internally
 * or must have come from an ancestor namespace. In either
 * case, the signal cannot be dropped.
 */
 if (unlikely(signal->flags & SIGNAL_UNKILLABLE) &&
 !sig_kernel_only(signr))
 continue;

 if (sig_kernel_stop(signr)) {
 /*
 * The default action is to stop all threads in
 * the thread group. The job control signals
 * do nothing in an orphaned pgrp, but SIGSTOP
 * always works. Note that siglock needs to be
 * dropped during the call to is_orphaned_pgrp()
 * because of lock ordering with tasklist_lock.
 * This allows an intervening SIGCONT to be posted.
 * We need to check for that and bail out if necessary.
 */
 if (signr != SIGSTOP) {
 spin_unlock_irq(&sighand->siglock);

 /* signals can be posted during this window */

 if (is_current_pgrp_orphaned())
 goto relock;

 spin_lock_irq(&sighand->siglock);
 }

 if (likely(do_signal_stop(ksig->info.si_signo))) {
 /* It released the siglock. */
 goto relock;
 }

 /*
 * We didn't actually stop, due to a race
 * with SIGCONT or something like that.
 */
 continue;
 }

 spin_unlock_irq(&sighand->siglock);

 /*
 * Anything else is fatal, maybe with a core dump.
 */
 current->flags |= PF_SIGNALED;

 if (sig_kernel_coredump(signr)) {
 if (print_fatal_signals)
 print_fatal_signal(ksig->info.si_signo);
 proc_coredump_connector(current);
 /*
 * If it was able to dump core, this kills all
 * other threads in the group and synchronizes with
 * their demise. If we lost the race with another
 * thread getting here, it set group_exit_code
 * first and our do_group_exit call below will use
 * that value and ignore the one we pass it.
 */
 do_coredump(&ksig->info);
 }

 /*
 * Death signals, no core dump.
 */
 do_group_exit(ksig->info.si_signo);
 /* NOTREACHED */
 }
```

首先，宏`sig_kernel_ignore`检查默认操作是否为忽略。如果为真，则继续循环迭代以查找下一个挂起信号。第二个宏`sig_kernel_stop`检查默认操作是否为停止；如果为真，则调用`do_signal_stop()`例程，将进程组中的每个线程置于`TASK_STOPPED`状态。第三个宏`sig_kernel_coredump`检查默认操作是否为转储；如果为真，则调用`do_coredump()`例程，生成转储二进制文件并终止线程组中的所有进程。接下来，对于默认操作为终止的信号，通过调用`do_group_exit()`例程杀死组中的所有线程。

# 执行用户模式处理程序

回顾我们在上一节中的讨论，`do_signal()` 调用 `handle_signal()` 例程以传递处于用户处理程序状态的挂起信号。用户模式信号处理程序驻留在进程代码段中，并需要访问进程的用户模式堆栈；因此，内核需要切换到用户模式堆栈以执行信号处理程序。成功从信号处理程序返回需要切换回内核堆栈以恢复用户上下文以进行正常的用户模式执行，但这样的操作将失败，因为内核堆栈不再包含用户上下文（`struct pt_regs`），因为在每次进程从用户模式进入内核模式时都会清空它。

为了确保进程在用户模式下正常执行时的平稳过渡（从信号处理程序返回），`handle_signal()` 将内核堆栈中的用户模式硬件上下文（`struct pt_regs`）移动到用户模式堆栈（`struct ucontext`）中，并设置处理程序帧以在返回时调用 `_kernel_rt_sigreturn()` 例程；此函数将硬件上下文复制回内核堆栈，并恢复当前进程的用户模式上下文以恢复正常执行。

以下图示了用户模式信号处理程序的执行：

![](img/00017.jpeg)

# 设置用户模式处理程序帧

为了为用户模式处理程序设置堆栈帧，`handle_signal()` 使用 `ksignal` 实例的地址调用 `setup_rt_frame()`，其中包含与信号相关的 `k_sigaction` 和当前进程内核堆栈中 `struct pt_regs` 的指针。

以下是 `setup_rt_frame()` 的 x86 实现：

```
setup_rt_frame(struct ksignal *ksig, struct pt_regs *regs)
{
 int usig = ksig->sig;
 sigset_t *set = sigmask_to_save();
 compat_sigset_t *cset = (compat_sigset_t *) set;

 /* Set up the stack frame */
 if (is_ia32_frame(ksig)) {
 if (ksig->ka.sa.sa_flags & SA_SIGINFO)
 return ia32_setup_rt_frame(usig, ksig, cset, regs); // for 32bit systems with SA_SIGINFO
 else
 return ia32_setup_frame(usig, ksig, cset, regs); // for 32bit systems without SA_SIGINFO
 } else if (is_x32_frame(ksig)) {
 return x32_setup_rt_frame(ksig, cset, regs);// for systems with x32 ABI
 } else {
 return __setup_rt_frame(ksig->sig, ksig, set, regs);// Other variants of x86
 }
}
```

它检查 x86 的特定变体，并调用适当的帧设置例程。在进一步讨论中，我们将专注于适用于 x86-64 的 `__setup_rt_frame()`。此函数使用一个名为 `struct rt_sigframe` 的结构的实例填充了处理信号所需的信息，设置了一个返回路径（通过 `_kernel_rt_sigreturn()` 函数），并将其推送到用户模式堆栈中。

```
/*arch/x86/include/asm/sigframe.h */
#ifdef CONFIG_X86_64

struct rt_sigframe {
 char __user *pretcode;
 struct ucontext uc;
 struct siginfo info;
 /* fp state follows here */
};

-----------------------  

/*arch/x86/kernel/signal.c */
static int __setup_rt_frame(int sig, struct ksignal *ksig,
 sigset_t *set, struct pt_regs *regs)
{
 struct rt_sigframe __user *frame;
 void __user *restorer;
 int err = 0;
 void __user *fpstate = NULL;

 /* setup frame with Floating Point state */
 frame = get_sigframe(&ksig->ka, regs, sizeof(*frame), &fpstate);

 if (!access_ok(VERIFY_WRITE, frame, sizeof(*frame)))
 return -EFAULT;

 put_user_try {
 put_user_ex(sig, &frame->sig);
 put_user_ex(&frame->info, &frame->pinfo);
 put_user_ex(&frame->uc, &frame->puc);

 /* Create the ucontext. */
 if (boot_cpu_has(X86_FEATURE_XSAVE))
 put_user_ex(UC_FP_XSTATE, &frame->uc.uc_flags);
 else 
 put_user_ex(0, &frame->uc.uc_flags);
 put_user_ex(0, &frame->uc.uc_link);
 save_altstack_ex(&frame->uc.uc_stack, regs->sp);

 /* Set up to return from userspace. */
 restorer = current->mm->context.vdso +
 vdso_image_32.sym___kernel_rt_sigreturn;
 if (ksig->ka.sa.sa_flags & SA_RESTORER)
 restorer = ksig->ka.sa.sa_restorer;
 put_user_ex(restorer, &frame->pretcode);

 /*
 * This is movl $__NR_rt_sigreturn, %ax ; int $0x80
 *
 * WE DO NOT USE IT ANY MORE! It's only left here for historical
 * reasons and because gdb uses it as a signature to notice
 * signal handler stack frames.
 */
 put_user_ex(*((u64 *)&rt_retcode), (u64 *)frame->retcode);
 } put_user_catch(err);

 err |= copy_siginfo_to_user(&frame->info, &ksig->info);
 err |= setup_sigcontext(&frame->uc.uc_mcontext, fpstate,
 regs, set->sig[0]);
 err |= __copy_to_user(&frame->uc.uc_sigmask, set, sizeof(*set));

 if (err)
 return -EFAULT;

 /* Set up registers for signal handler */
 regs->sp = (unsigned long)frame;
 regs->ip = (unsigned long)ksig->ka.sa.sa_handler;
 regs->ax = (unsigned long)sig;
 regs->dx = (unsigned long)&frame->info;
 regs->cx = (unsigned long)&frame->uc;

 regs->ds = __USER_DS;
 regs->es = __USER_DS;
 regs->ss = __USER_DS;
 regs->cs = __USER_CS;

 return 0;
}
```

`rt_sigframe` 结构的 `*pretcode` 字段被分配为信号处理程序函数的返回地址，该函数是 `_kernel_rt_sigreturn()` 例程。 `struct ucontext uc` 用 `sigcontext` 初始化，其中包含从内核堆栈的 `pt_regs` 复制的用户模式上下文，常规阻塞信号的位数组和浮点状态。在设置并将 `frame` 实例推送到用户模式堆栈后，`__setup_rt_frame()` 改变了进程的内核堆栈中的 `pt_regs`，以便在当前进程恢复执行时将控制权交给信号处理程序。**指令指针（ip）**设置为信号处理程序的基地址，**堆栈指针（sp）**设置为先前推送的帧的顶部地址；这些更改导致信号处理程序执行。

# 重新启动中断的系统调用

我们在第一章中了解到，用户模式进程调用 *系统调用* 以切换到内核模式执行内核服务。当进程进入内核服务例程时，有可能例程被阻塞以等待资源的可用性（例如，等待排他锁）或事件的发生（例如中断）。这些阻塞操作要求调用进程处于 `TASK_INTERRUPTIBLE`、`TASK_UNINTERRUPTIBLE` 或 `TASK_KILLABLE` 状态。所采取的具体状态取决于在系统调用中调用的阻塞调用的选择。

如果调用者任务被置于`TASK_UNINTERRUPTIBLE`状态，那么在该任务上发生的信号会导致它们进入挂起列表，并且仅在服务例程完成后（返回到用户模式时）才会传递给进程。然而，如果任务被置于`TASK_INTERRUPTIBLE`状态，那么在该任务上发生的信号会导致其状态被改变为`TASK_RUNNING`，从而导致任务在阻塞的系统调用上被唤醒，甚至在系统调用完成之前就被唤醒（导致系统调用操作失败）。这种中断通过返回适当的失败代码来指示。在`TASK_KILLABLE`状态下，信号对任务的影响与`TASK_INTERRUPTIBLE`类似，只是在发生致命的`SIGKILL`信号时才会唤醒。

`EINTR`、`ERESTARTNOHAND`、`ERESTART_RESTARTBLOCK`、`ERESTARTSYS`或`ERESTARTNOINTR`是各种内核定义的失败代码；系统调用被编程为在失败时返回适当的错误标志。错误代码的选择决定了在处理中断信号后是否重新启动失败的系统调用操作：

```
(include/uapi/asm-generic/errno-base.h)
 #define EPERM 1 /* Operation not permitted */
 #define ENOENT 2 /* No such file or directory */
 #define ESRCH 3 /* No such process */
 #define EINTR 4 /* Interrupted system call */
 #define EIO 5 /* I/O error */
 #define ENXIO 6 /* No such device or address */
 #define E2BIG 7 /* Argument list too long */
 #define ENOEXEC 8 /* Exec format error */
 #define EBADF 9 /* Bad file number */
 #define ECHILD 10 /* No child processes */
 #define EAGAIN 11 /* Try again */
 #define ENOMEM 12 /* Out of memory */
 #define EACCES 13 /* Permission denied */
 #define EFAULT 14 /* Bad address */
 #define ENOTBLK 15 /* Block device required */
 #define EBUSY 16 /* Device or resource busy */
 #define EEXIST 17 /* File exists */
 #define EXDEV 18 /* Cross-device link */
 #define ENODEV 19 /* No such device */
 #define ENOTDIR 20 /* Not a directory */
 #define EISDIR 21 /* Is a directory */
 #define EINVAL 22 /* Invalid argument */
 #define ENFILE 23 /* File table overflow */
 #define EMFILE 24 /* Too many open files */
 #define ENOTTY 25 /* Not a typewriter */
 #define ETXTBSY 26 /* Text file busy */
 #define EFBIG 27 /* File too large */
 #define ENOSPC 28 /* No space left on device */
 #define ESPIPE 29 /* Illegal seek */
 #define EROFS 30 /* Read-only file system */
 #define EMLINK 31 /* Too many links */
 #define EPIPE 32 /* Broken pipe */
 #define EDOM 33 /* Math argument out of domain of func */
 #define ERANGE 34 /* Math result not representable */
 linux/errno.h)
 #define ERESTARTSYS 512
 #define ERESTARTNOINTR 513
 #define ERESTARTNOHAND 514 /* restart if no handler.. */
 #define ENOIOCTLCMD 515 /* No ioctl command */
 #define ERESTART_RESTARTBLOCK 516 /* restart by calling sys_restart_syscall */
 #define EPROBE_DEFER 517 /* Driver requests probe retry */
 #define EOPENSTALE 518 /* open found a stale dentry */
```

从中断的系统调用返回时，用户模式 API 始终返回`EINTR`错误代码，而不管底层内核服务例程返回的具体错误代码是什么。其余的错误代码由内核的信号传递例程使用，以确定从信号处理程序返回时是否可以重新启动中断的系统调用。以下表格显示了系统调用执行被中断时的错误代码以及对各种信号处理的影响：

![](img/00018.jpeg)

这是它们的含义：

+   **不重新启动**：系统调用不会被重新启动。进程将从跟随系统调用的指令（int $0x80 或 sysenter）中的用户模式恢复执行。

+   **自动重启**：内核强制用户进程通过将相应的系统调用标识符加载到*eax*中并执行系统调用指令（int $0x80 或 sysenter）来重新启动系统调用操作。

+   **显式重启**：只有在进程设置中断信号的处理程序（通过 sigaction）时启用了`SA_RESTART`标志，系统调用才会被重新启动。

# 摘要

信号，虽然是进程和内核服务之间进行的一种基本形式的通信，但它们提供了一种简单有效的方式，以便在发生各种事件时从运行中的进程获得异步响应。通过理解信号使用的所有核心方面，它们的表示、数据结构和内核例程用于信号生成和传递，我们现在对内核更加了解，也更有准备在本书的后面部分更深入地研究进程之间更复杂的通信方式。在前三章中讨论了进程及其相关方面之后，我们现在将深入研究内核的其他子系统，以提高我们的可见性。在下一章中，我们将建立对内核的核心方面之一——内存子系统的理解。

在接下来的一章中，我们将逐步理解许多关键的内存管理方面，如内存初始化、分页和保护，以及内核内存分配算法等。
