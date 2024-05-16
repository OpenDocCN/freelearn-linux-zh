# 第十章。理解 Linux 用户和内核限制

在上一章中，我们使用了`lsof`和`strace`等工具来确定应用程序问题的根本原因。

在本章中，我们将再次确定应用程序相关问题的根本原因。但是，我们还将专注于学习和理解 Linux 用户和内核的限制。

# 一个报告的问题

就像上一章专注于自定义应用程序的问题一样，今天的问题也来自同一个自定义应用程序。

今天，我们将处理应用支持团队报告的一个问题。然而，这一次支持团队能够为我们提供相当多的信息。

我们在第九章中处理的应用程序，*使用系统工具来排除应用程序问题*，现在通过`端口 25`接收消息并将其存储在队列目录中。定期会运行一个作业来处理这些排队的消息，但是这个作业*似乎不再工作*。

应用支持团队已经注意到队列中积压了大量消息。然而，尽管他们已经尽可能地排除了问题，但他们卡住了，需要我们的帮助。

# 为什么作业失败了？

由于报告的问题是定时作业不起作用，我们应该首先关注作业本身。在这种情况下，我们有应用支持团队可以回答任何问题。所以，让我们再多了解一些关于这个作业的细节。

## 背景问题

以下是一系列快速问题，应该能够为您提供额外的信息：

+   作业是如何运行的？

+   如果需要，我们可以手动运行作业吗？

+   这个作业执行什么？

这三个问题可能看起来很基础，但它们很重要。让我们首先看一下应用团队提供的答案：

+   作业是如何运行的？

*作业是作为 cron 作业执行的。*

+   如果需要，我们可以手动运行作业吗？

*是的，可以根据需要手动执行作业。*

+   这个作业执行什么？

*作业以 vagrant 用户身份执行/opt/myapp/bin/processor 命令*。

前面的三个问题很重要，因为它们将为我们节省大量的故障排除时间。第一个问题关注作业是如何执行的。由于报告的问题是作业不起作用，我们还不知道问题是因为作业没有运行还是作业正在执行但由于某种原因失败。

第一个问题的答案告诉我们，这个作业是由在 Linux 上运行的**cron 守护程序**`crond`执行的。这很有用，因为我们可以使用这些信息来确定作业是否正在执行。一般来说，有很多方法可以执行定时作业。有时执行定时作业的软件在不同的系统上运行，有时在同一个本地系统上运行。

在这种情况下，作业是由`crond`在同一台服务器上执行的。

第二个问题也很重要。就像我们在上一章中需要手动启动应用程序一样，我们可能也需要对这个报告的问题执行这个故障排除步骤。根据答案，似乎我们可以根据需要多次执行这个命令。

第三个问题很有用，因为它不仅告诉我们正在执行哪个命令，还告诉我们要注意哪个作业。cron 作业是一种非常常见的调度任务的方法。一个系统通常会有许多已调度的 cron 作业。

## cron 作业是否在运行？

由于我们知道作业是由`crond`执行的，我们应该首先检查作业是否正在执行。为此，我们可以在相关服务器上检查 cron 日志。例如，考虑以下日志：

```
# ls -la /var/log/cron*
-rw-r--r--. 1 root root 30792 Jun 10 18:05 /var/log/cron
-rw-r--r--. 1 root root 28261 May 18 03:41 /var/log/cron-20150518
-rw-r--r--. 1 root root  6152 May 24 21:12 /var/log/cron-20150524
-rw-r--r--. 1 root root 42565 Jun  1 15:50 /var/log/cron-20150601
-rw-r--r--. 1 root root 18286 Jun  7 16:22 /var/log/cron-20150607

```

具体来说，在基于 Red Hat 的 Linux 系统上，我们可以检查`/var/log/cron`日志文件。我在前一句中指定了“基于 Red Hat 的”是因为在非 Red Hat 系统上，cron 日志可能位于不同的日志文件中。例如，基于 Debian 的系统默认为`/var/log/syslog`。

如果我们不知道哪个日志文件包含 cron 日志，有一个简单的技巧可以找到它。只需运行以下命令行：

```
# grep -ic cron /var/log/* | grep -v :0
/var/log/cron:400
/var/log/cron-20150518:379
/var/log/cron-20150524:86
/var/log/cron-20150601:590
/var/log/cron-20150607:248
/var/log/messages:1
/var/log/secure:1

```

前面的命令将使用`grep`在`/var/log`中的所有日志文件中搜索字符串`cron`。该命令还将搜索`Cron`、`CRON`、`cRon`等，因为我们在`grep`命令中添加了`-i`（不区分大小写）标志。这告诉`grep`在不区分大小写的模式下搜索。基本上，这意味着任何匹配单词`cron`的地方都会被找到，即使单词是大写或混合大小写。我们还在`grep`命令中添加了`-c`（计数）标志，这会导致它计算它找到的实例数：

```
/var/log/cron:400

```

如果我们看第一个结果，我们可以看到`grep`在`/var/log/cron`中找到了 400 个“cron”单词的实例。

最后，我们将结果重定向到另一个带有`-v`标志和`:0`的`grep`命令。这个`grep`将获取第一次执行的结果，并省略（-v）任何包含字符串`:0`的行。这对于将结果限制为只有包含其中的`cron`字符串的文件非常有用。

从前面的结果中，我们可以看到文件`/var/log/cron`中包含了最多的“cron”单词实例。这一事实本身就是`/var/log/cron`是`crond`守护程序的日志文件的一个很好的指示。

既然我们知道哪个日志文件包含了我们正在寻找的日志消息，我们可以查看该日志文件的内容。由于这个日志文件非常大，我们将使用`less`命令来读取这个文件：

```
# less /var/log/cron

```

由于这个日志中包含了相当多的信息，我们只会关注能帮助解释问题的日志条目。以下部分是一组有趣的日志消息，应该能回答我们的作业是否正在运行：

```
Jun 10 18:01:01 localhost CROND[2033]: (root) CMD (run-parts /etc/cron.hourly)
Jun 10 18:01:01 localhost run-parts(/etc/cron.hourly)[2033]: starting 0anacron
Jun 10 18:01:01 localhost run-parts(/etc/cron.hourly)[2042]: finished 0anacron
Jun 10 18:01:01 localhost run-parts(/etc/cron.hourly)[2033]: starting 0yum-hourly.cron
Jun 10 18:01:01 localhost run-parts(/etc/cron.hourly)[2048]: finished 0yum-hourly.cron
Jun 10 18:05:01 localhost CROND[2053]: (vagrant) CMD (/opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml > /dev/null)
Jun 10 18:10:01 localhost CROND[2086]: (root) CMD (/usr/lib64/sa/sa1 1 1)
Jun 10 18:10:01 localhost CROND[2087]: (vagrant) CMD (/opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml > /dev/null)
Jun 10 18:15:01 localhost CROND[2137]: (vagrant) CMD (/opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml > /dev/null)
Jun 10 18:20:01 localhost CROND[2147]: (root) CMD (/usr/lib64/sa/sa1 1 1)

```

前面的日志消息显示了相当多的行。让我们分解日志以更好地理解正在执行的内容。考虑以下行：

```
Jun 10 18:01:01 localhost CROND[2033]: (root) CMD (run-parts /etc/cron.hourly)
Jun 10 18:01:01 localhost run-parts(/etc/cron.hourly)[2033]: starting 0anacron
Jun 10 18:01:01 localhost run-parts(/etc/cron.hourly)[2042]: finished 0anacron
Jun 10 18:01:01 localhost run-parts(/etc/cron.hourly)[2033]: starting 0yum-hourly.cron
Jun 10 18:01:01 localhost run-parts(/etc/cron.hourly)[2048]: finished 0yum-hourly.cron

```

前几行似乎不是我们正在寻找的作业，而是`cron.hourly`作业。

在 Linux 系统上，有多种方法可以指定 cron 作业。在 RHEL 系统上，`/etc/`目录下有几个以`cron`开头的目录：

```
# ls -laF /etc/ | grep cron
-rw-------.  1 root root      541 Jun  9  2014 anacrontab
drwxr-xr-x.  2 root root       34 Jan 23 15:43 cron.d/
drwxr-xr-x.  2 root root       62 Jul 22  2014 cron.daily/
-rw-------.  1 root root        0 Jun  9  2014 cron.deny
drwxr-xr-x.  2 root root       44 Jul 22  2014 cron.hourly/
drwxr-xr-x.  2 root root        6 Jun  9  2014 cron.monthly/
-rw-r--r--.  1 root root      451 Jun  9  2014 crontab
drwxr-xr-x.  2 root root        6 Jun  9  2014 cron.weekly/

```

`cron.daily`、`cron.hourly`、`cron.monthly`和`cron.weekly`目录都是可以包含脚本的目录。这些脚本将按照目录名称中指定的时间运行。

例如，让我们看一下`/etc/cron.hourly/0yum-hourly.cron`：

```
# cat /etc/cron.hourly/0yum-hourly.cron
#!/bin/bash

# Only run if this flag is set. The flag is created by the yum-cron init
# script when the service is started -- this allows one to use chkconfig and
# the standard "service stop|start" commands to enable or disable yum-cron.
if [[ ! -f /var/lock/subsys/yum-cron ]]; then
 exit 0
fi

# Action!
exec /usr/sbin/yum-cron /etc/yum/yum-cron-hourly.conf

```

前面的文件是一个简单的`bash`脚本，`crond`守护程序将每小时执行一次，因为它在`cron.hourly`目录中。一般来说，这些目录中包含的脚本是由系统服务放在那里的。不过，这些目录也对系统管理员开放，可以放置他们自己的脚本。

## 用户 crontabs

如果我们继续查看日志文件，我们可以看到一个与我们自定义作业相关的条目：

```
Jun 10 18:10:01 localhost CROND[2087]: (vagrant) CMD (/opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml > /dev/null)

```

这一行显示了应用支持团队引用的`processor`命令。这一行必须是应用支持团队遇到问题的作业。日志条目告诉我们很多有用的信息。首先，它为我们提供了传递给这个作业的命令行选项：

```
/opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml > /dev/null

```

它还告诉我们作业是以`vagrant`身份执行的。不过，这个日志条目告诉我们最重要的是作业正在执行。

既然我们知道作业正在执行，我们应该验证作业是否成功。为了做到这一点，我们将采取一种简单的方法，手动执行作业：

```
$ /opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml
Initializing with configuration file /opt/myapp/conf/config.yml
- - - - - - - - - - - - - - - - - - - - - - - - - -
Starting message processing job
Traceback (most recent call last):
 File "app.py", line 28, in init app (app.c:1488)
IOError: [Errno 24] Too many open files: '/opt/myapp/queue/1433955823.29_0.txt'

```

我们应该从 cron 任务的末尾省略`> /dev/null`，因为这将把输出重定向到`/dev/null`。这是一种常见的丢弃 cron 作业输出的方法。对于此手动执行，我们可以利用输出来帮助解决问题。

一旦执行，作业似乎会失败。它不仅失败了，而且还产生了一个错误消息：

```
IOError: [Errno 24] Too many open files: '/opt/myapp/queue/1433955823.29_0.txt'

```

这个错误很有趣，因为它似乎表明应用程序打开了太多文件。*这有什么关系呢？*

# 了解用户限制

在 Linux 系统上，每个进程都受到限制。这些限制是为了防止进程使用太多的系统资源。

虽然这些限制适用于每个用户，但是可以为每个用户设置不同的限制。要检查`vagrant`用户默认设置的限制，我们可以使用`ulimit`命令：

```
$ ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 3825
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 3825
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited

```

当我们执行`ulimit`命令时，我们是以 vagrant 用户的身份执行的。这很重要，因为当我们以任何其他用户（包括 root）的身份运行`ulimit`命令时，输出将是该用户的限制。

如果我们查看`ulimit`命令的输出，我们可以看到有很多可以设置的限制。

## 文件大小限制

让我们来看一下并分解一些关键限制：

```
file size               (blocks, -f) unlimited

```

第一个有趣的项目是`文件大小`限制。这个限制将限制用户可以创建的文件的大小。vagrant 用户的当前设置是`无限制`，但如果我们将这个值设置为一个较小的数字会发生什么呢？

我们可以通过执行`ulimit -f`，然后跟上要限制文件的块数来做到这一点。例如，考虑以下命令行：

```
$ ulimit -f 10

```

将值设置为`10`后，我们可以通过再次运行`ulimit -f`来验证它是否生效，但这次不带值：

```
$ ulimit -f
10

```

现在我们的限制设置为 10 个块，让我们尝试使用`dd`命令创建一个 500MB 的文件：

```
$ dd if=/dev/zero of=/var/tmp/bigfile bs=1M count=500
File size limit exceeded

```

关于 Linux 用户限制的一个好处是通常提供的错误是不言自明的。我们可以从前面的输出中看到，`dd`命令不仅无法创建文件，还收到了一个错误，指出文件大小限制已超出。

## 最大用户进程限制

另一个有趣的限制是`最大进程`限制：

```
max user processes              (-u) 3825

```

这个限制防止用户一次运行*太多的进程*。这是一个非常有用和有趣的限制，因为它可以轻松地防止一个恶意应用程序接管系统。

这也可能是您经常会遇到的限制。这对于启动许多子进程或线程的应用程序尤其如此。要查看此限制如何工作，我们可以将设置更改为`10`：

```
$ ulimit -u 10
$ ulimit -u
10

```

与文件大小限制一样，我们可以使用`ulimit`命令修改进程限制。但这次，我们使用`-u`标志。每个用户限制都有自己独特的标志与`ulimit`命令。我们可以在`ulimit -a`的输出中看到这些标志，当然，每个标志都在`ulimit`的 man 页面中引用。

既然我们已经将我们的进程限制为`10`，我们可以通过运行一个命令来看到限制的执行：

```
$ man ulimit
man: fork failed: Resource temporarily unavailable

```

通过 SSH 登录 vagrant 用户，我们已经在使用多个进程。很容易遇到`10`个进程的限制，因为我们运行的任何新命令都会超出我们的登录限制。

从前面的例子中，我们可以看到当执行`man`命令时，它无法启动子进程，因此返回了一个错误，指出`资源暂时不可用`。

## 打开文件限制

我想要探索的最后一个有趣的用户限制是`打开文件`限制：

```
open files                      (-n) 1024

```

`打开文件`限制将限制进程打开的文件数不超过定义的数量。此限制可用于防止进程一次打开太多文件。当防止应用程序占用系统资源过多时，这是一种很有用的方法。

像其他限制一样，让我们看看当我们将这个限制减少到一个非常不合理的数字时会发生什么：

```
$ ulimit -n 2
$ ls
-bash: start_pipeline: pgrp pipe: Too many open files
ls: error while loading shared libraries: libselinux.so.1: cannot open shared object file: Error 24

```

与其他示例一样，我们在这种情况下收到了一个错误，即`Too many open files`。但是，这个错误看起来很熟悉。如果我们回顾一下从我们的计划作业收到的错误，我们就会明白为什么。

```
IOError: [Errno 24] Too many open files: '/opt/myapp/queue/1433955823.29_0.txt'

```

将我们的最大打开文件数设置为`2`后，`ls`命令产生了一个错误；这个错误与我们的应用程序之前执行时收到的完全相同的错误消息。

这是否意味着我们的应用程序试图打开的文件比我们的系统配置允许的要多？这是一个很有可能的情况。

# 更改用户限制

由于我们怀疑`open files`限制阻止了应用程序的执行，我们可以将其限制设置为更高的值。但是，这并不像执行`ulimit -n`那样简单；执行时得到的输出如下：

```
$ ulimit -n
1024
$ ulimit -n 5000
-bash: ulimit: open files: cannot modify limit: Operation not permitted
$ ulimit -n 4096
$ ulimit -n
4096

```

在我们的示例系统上，默认情况下，vagrant 用户被允许将`open files`限制提高到`4096`。从前面的错误中我们可以看到，任何更高的值都被拒绝；但是像大多数 Linux 一样，我们可以改变这一点。

## limits.conf 文件

我们一直在使用和修改的用户限制是 Linux 的 PAM 系统的一部分。PAM 或可插拔认证模块是一个提供模块化认证系统的系统。

例如，如果我们的系统要使用 LDAP 进行身份验证，`pam_ldap.so`库将用于提供此功能。但是，由于我们的系统使用本地用户进行身份验证，因此`pam_localuser.so`库处理用户身份验证。

如果我们阅读`/etc/pam.d/system-auth`文件，我们可以验证这一点：

```
$ cat /etc/pam.d/system-auth
#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authconfig is run.
auth        required      pam_env.so
auth        sufficient    pam_unix.so nullok try_first_pass
auth        requisite     pam_succeed_if.so uid >= 1000 quiet_success
auth        required      pam_deny.so

account     required      pam_unix.so
account     sufficient    pam_localuser.so
account     sufficient    pam_succeed_if.so uid < 1000 quiet
account     required      pam_permit.so

password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok
password    required      pam_deny.so

session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
-session     optional      pam_systemd.so
session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
session     required      pam_unix.so

```

如果我们看一下前面的例子，我们可以看到`pam_localuser.so`与`account`一起列在第一列：

```
account     sufficient    pam_localuser.so

```

这意味着`pam_localuser.so`模块是一个`sufficient`模块，允许账户被使用，这基本上意味着如果他们有正确的`/etc/passwd`和`/etc/shadow`条目，用户就能够登录。

```
session     required      pam_limits.so

```

如果我们看一下前面的行，我们可以看到用户限制是在哪里执行的。这行基本上告诉系统`pam_limits.so`模块对所有用户会话都是必需的。这有效地确保了`pam_limits.so`模块识别的用户限制在每个用户会话上都得到执行。

这个 PAM 模块的配置位于`/etc/security/limits.conf`和`/etc/security/limits.d/`中。

```
$ cat /etc/security/limits.conf
#This file sets the resource limits for the users logged in via PAM.
#        - core - limits the core file size (KB)
#        - data - max data size (KB)
#        - fsize - maximum filesize (KB)
#        - memlock - max locked-in-memory address space (KB)
#        - nofile - max number of open files
#        - rss - max resident set size (KB)
#        - stack - max stack size (KB)
#        - cpu - max CPU time (MIN)
#        - nproc - max number of processes
#        - as - address space limit (KB)
#        - maxlogins - max number of logins for this user
#        - maxsyslogins - max number of logins on the system
#        - priority - the priority to run user process with
#        - locks - max number of file locks the user can hold
#        - sigpending - max number of pending signals
#        - msgqueue - max memory used by POSIX message queues (bytes)
#        - nice - max nice priority allowed to raise to values: [-20, 19]
#        - rtprio - max realtime priority
#
#<domain>      <type>  <item>         <value>
#

#*               soft    core            0
#*               hard    rss             10000
#@student        hard    nproc           20
#@faculty        soft    nproc           20
#@faculty        hard    nproc           50
#ftp             hard    nproc           0
#@student        -       maxlogins       4

```

当我们阅读`limits.conf`文件时，我们可以看到关于用户限制的相当多有用的信息。

在这个文件中，列出了可用的限制以及该限制强制执行的描述。例如，在前面的命令行中，我们可以看到`open files`限制的数量：

```
#        - nofile - max number of open files

```

从这一行我们可以看到，如果我们想改变用户可用的打开文件数，我们需要使用`nofile`类型。除了列出每个限制的作用，`limits.conf`文件还包含了为用户和组设置自定义限制的示例：

```
#ftp             hard    nproc           0

```

通过这个例子，我们可以看到我们需要使用什么格式来设置限制；但我们应该将限制设置为多少呢？如果我们回顾一下作业中的错误，我们会发现错误列出了`/opt/myapp/queue`目录中的一个文件：

```
IOError: [Errno 24] Too many open files: '/opt/myapp/queue/1433955823.29_0.txt'

```

可以肯定地说，应用程序正在尝试打开此目录中的文件。因此，为了确定这个进程需要打开多少文件，让我们通过以下命令行找出这个目录中有多少文件存在：

```
$ ls -la /opt/myapp/queue/ | wc -l
492304

```

前面的命令使用`ls -la`列出`queue/`目录中的所有文件和目录，并将输出重定向到`wc -l`。`wc`命令将从提供的输出中计算行数（`-l`），这基本上意味着在`queue/`目录中有 492,304 个文件和/或目录。

鉴于数量很大，我们应该将`打开文件`限制数量设置为`500000`，足以处理`queue/`目录，以防万一再多一点。我们可以通过将以下行附加到`limits.conf`文件来实现这一点：

```
# vi /etc/security/limits.conf

```

在使用`vi`或其他文本编辑器添加我们的行之后，我们可以使用`tail`命令验证它是否存在：

```
$ tail /etc/security/limits.conf
#@student        hard    nproc           20
#@faculty        soft    nproc           20
#@faculty        hard    nproc           50
#ftp             hard    nproc           0
#@student        -       maxlogins       4

vagrant    soft  nofile    100000
vagrant    hard  nofile    500000

# End of file

```

更改这些设置并不意味着我们的登录 shell 立即具有`500000`的限制。我们的登录会话仍然设置了`4096`的限制。

```
$ ulimit -n
4096

```

我们还不能将其增加到该值以上。

```
$ ulimit -n 9000
-bash: ulimit: open files: cannot modify limit: Operation not permitted

```

为了使我们的更改生效，我们必须再次登录到我们的用户。

正如我们之前讨论的，这些限制是由 PAM 设置的，在我们的 shell 会话登录期间应用。由于这些限制是在登录期间设置的，所以我们仍然受到上次登录时采用的先前数值的限制。

要获取新的限制，我们必须注销并重新登录（或生成一个新的登录会话）。在我们的例子中，我们将注销我们的 shell 并重新登录。

```
$ ulimit -n
100000
$ ulimit -n 501000
-bash: ulimit: open files: cannot modify limit: Operation not permitted
$ ulimit -n 500000
$ ulimit -n
500000

```

如果我们看一下前面的命令行，我们会发现一些非常有趣的东西。

当我们这次登录时，我们的文件限制数量被设置为`100000`，这恰好是我们在`limits.conf`文件中设置的`soft`限制。这是因为`soft`限制是每个会话默认设置的限制。

`hard`限制是该用户可以设置的高于`soft`限制的最高值。我们可以在前面的例子中看到这一点，因为我们能够将`nofile`限制设置为`500000`，但不能设置为`501000`。

### 未来保护定时作业

我们将`soft`限制设置为`100000`的原因是因为我们计划在未来处理类似的情况。将`soft`限制设置为`100000`，运行这个定时作业的 cron 作业将被限制为 100,000 个打开文件。然而，由于`hard`限制设置为`500000`，某人可以在他们的登录会话中手动运行具有更高限制的作业。

只要`queue`目录中的文件数量不超过 500,000，就不再需要任何人编辑`/etc/security/limits.conf`文件。

## 再次运行作业

现在我们的限制已经增加，我们可以尝试再次运行作业。

```
$ /opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml
Initializing with configuration file /opt/myapp/conf/config.yml
- - - - - - - - - - - - - - - - - - - - - - - - - -
Starting message processing job
Traceback (most recent call last):
 File "app.py", line 28, in init app (app.c:1488)
IOError: [Errno 23] Too many open files in system: '/opt/myapp/queue/1433955989.86_5.txt'

```

我们再次收到了一个错误。然而，这次错误略有不同。

在上一次运行中，我们收到了以下错误。

```
IOError: [Errno 24] Too many open files: '/opt/myapp/queue/1433955823.29_0.txt'

```

然而，这一次我们收到了这个错误。

```
IOError: [Errno 23] Too many open files in system: '/opt/myapp/queue/1433955989.86_5.txt'

```

这种差异非常微妙，但在第二次运行时，我们的错误说明了**系统中打开的文件太多**，而我们第一次运行时没有包括`in system`。这是因为我们遇到了不同类型的限制，不是**用户**限制，而是**系统**限制。

# 内核可调参数

Linux 内核本身也可以对系统设置限制。这些限制是基于内核参数定义的。其中一些参数是静态的，不能在运行时更改；而其他一些可以。当内核参数可以在运行时更改时，这被称为**可调参数**。

我们可以使用`sysctl`命令查看静态和可调内核参数及其当前值。

```
# sysctl -a | head
abi.vsyscall32 = 1
crypto.fips_enabled = 0
debug.exception-trace = 1
debug.kprobes-optimization = 1
dev.hpet.max-user-freq = 64
dev.mac_hid.mouse_button2_keycode = 97
dev.mac_hid.mouse_button3_keycode = 100
dev.mac_hid.mouse_button_emulation = 0
dev.parport.default.spintime = 500
dev.parport.default.timeslice = 200

```

由于有许多参数可用，我使用`head`命令将输出限制为前 10 个。我们之前收到的错误提到了系统上的限制，这表明我们可能遇到了内核本身施加的限制。

唯一的问题是我们如何知道哪一个？最快的答案当然是搜索谷歌。由于有很多内核参数（我们正在使用的系统上有 800 多个），简单地阅读`sysctl –a`的输出并找到正确的参数是困难的。

一个更现实的方法是简单地搜索我们要修改的参数类型。我们的情况下，一个例子搜索可能是`Linux 参数最大打开文件数`。如果我们进行这样的搜索，很可能会找到参数以及如何修改它。然而，如果谷歌不是一个选择，还有另一种方法。

一般来说，内核参数的名称描述了参数控制的内容。

例如，如果我们要查找禁用 IPv6 的内核参数，我们首先会搜索`net`字符串，如网络：

```
# sysctl -a | grep -c net
556

```

然而，这仍然返回了大量结果。在这些结果中，我们可以看到字符串`ipv6`。

```
# sysctl -a | grep -c ipv6
233

```

尽管如此，还是有相当多的结果；但是，如果我们添加一个搜索字符串`disable`，我们会得到以下输出：

```
# sysctl -a | grep ipv6 | grep disable
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.enp0s3.disable_ipv6 = 0
net.ipv6.conf.enp0s8.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0

```

我们终于可以缩小可能的参数范围。但是，我们还不完全知道这些参数的作用。至少目前还不知道。

如果我们通过`/usr/share/doc`进行快速搜索，可能会找到一些解释这些设置作用的文档。我们可以通过使用`grep`在该目录中进行递归搜索来快速完成这个过程。为了保持输出简洁，我们可以添加`-l`（列出文件），这会导致`grep`只列出包含所需字符串的文件名：

```
# grep -rl net.ipv6 /usr/share/doc/
/usr/share/doc/grub2-tools-2.02/grub.html

```

在基于 Red Hat 的 Linux 系统中，`/usr/share/doc`目录用于系统手册之外的额外文档。如果我们只能使用系统本身的文档，`/usr/share/doc`目录是首要检查的地方之一。

## 查找打开文件的内核参数

由于我们喜欢用较困难的方式执行任务，我们将尝试识别潜在限制我们的内核参数，而不是在 Google 上搜索。这样做的第一步是在`sysctl`输出中搜索字符串`file`。

我们搜索`file`的原因是因为我们遇到了文件数量的限制。虽然这可能不会提供我们要识别的确切参数，但至少搜索会让我们有个起点：

```
# sysctl -a | grep file
fs.file-max = 48582
fs.file-nr = 1088  0  48582
fs.xfs.filestream_centisecs = 3000

```

事实上，搜索`file`可能实际上是一个非常好的选择。仅仅根据参数的名称，对我们可能感兴趣的两个参数是`fs.file-max`和`fs.file-nr`。在这一点上，我们不知道哪一个控制打开文件的数量，或者这两个参数是否有任何作用。

要了解更多信息，我们可以搜索`doc`目录。

```
# grep -r fs.file- /usr/share/doc/
/usr/share/doc/postfix-2.10.1/README_FILES/TUNING_README:
fs.file-max=16384

```

看起来一个名为`TUNING_README`的文档，位于 Postfix 服务文档中，提到了我们至少一个值的参考。让我们查看一下文件，看看这个文档对这个内核参数有什么说法：

```
* Configure the kernel for more open files and sockets. The details are
 extremely system dependent and change with the operating system version. Be
 sure to verify the following information with your system tuning guide:

 o Linux kernel parameters can be specified in /etc/sysctl.conf or changed
 with sysctl commands:

 fs.file-max=16384
 kernel.threads-max=2048

```

如果我们阅读文件中列出我们的内核参数周围的内容，我们会发现它明确指出了*配置内核以获得更多打开文件和套接字*的参数。

这个文档列出了两个内核参数，允许更多的打开文件和套接字。第一个被称为`fs.file-max`，这也是我们在`sysctl`搜索中识别出的一个。第二个被称为`kernel.threads-max`，这是相当新的。

仅仅根据名称，似乎我们想要修改的可调参数是`fs.file-max`参数。让我们看一下它的当前值：

```
# sysctl fs.file-max
fs.file-max = 48582

```

我们可以通过执行`sysctl`命令，后跟参数名称（如前面的命令行所示）来列出此参数的当前值。这将简单地显示当前定义的值；看起来设置为`48582`，远低于我们当前的用户限制。

### 提示

在前面的例子中，我们在一个 postfix 文档中找到了这个参数。虽然这可能很好，但并不准确。如果您经常需要在本地搜索内核参数，最好安装`kernel-doc`软件包。`kernel-doc`软件包包含了大量信息，特别是关于可调参数的信息。

## 更改内核可调参数

由于我们认为`fs.file-max`参数控制系统可以打开的最大文件数，我们应该更改这个值以允许我们的作业运行。

像大多数 Linux 上的系统配置项一样，有更改此值的临时和重新启动的选项。之前我们将`limits.conf`文件设置为允许 vagrant 用户能够以`软`限制打开 100,000 个文件，以`硬`限制打开 500,000 个文件。问题是我们是否希望这个用户能够正常操作打开 500,000 个文件？还是应该是一次性任务来纠正我们目前面临的问题？

答案很简单：*这取决于情况！*

如果我们看一下我们目前正在处理的情况，所讨论的工作已经有一段时间没有运行了。因此，队列中积压了大量的消息。但是这些并不是正常的情况。

早些时候，当我们将用户限制设置为 100,000 个文件时，我们这样做是因为这对于这个作业来说是一个相当合适的值。考虑到这一点，我们还应该将内核参数设置为略高于`100000`的值，但不要太高。

对于这种情况和环境，我们将执行两个操作。第一个是配置系统默认允许*125,000 个打开文件*。第二个是将当前参数设置为*525,000 个打开文件*，以便成功运行预定的作业。

### 永久更改可调整的值

由于我们想要将`fs.file-max`的值默认更改为`125000`，我们需要编辑`sysctl.conf`文件。`sysctl.conf`文件是一个系统配置文件，允许您为可调整的内核参数指定自定义值。在系统每次重新启动时，该文件都会被读取并应用其中的值。

为了将我们的`fs.file-max`值设置为`125000`，我们只需将以下行追加到这个文件中：

```
# vi /etc/sysctl.conf
fs.file-max=125000

```

现在我们已经添加了我们的自定义值，我们需要告诉系统应用它。

如前所述，`sysctl.conf`文件在重新启动时生效，但是我们也可以随时使用`sysctl`命令和`-p`标志将设置应用到这个文件。

```
# sysctl -p
fs.file-max = 125000

```

给定`-p`标志后，`sysctl`命令将读取并将值应用到指定的文件，或者如果没有指定文件，则应用到`/etc/sysctl.conf`。由于我们在`-p`标志后没有指定文件，`sysctl`命令将应用到`/etc/sysctl.conf`中添加的值，并打印修改的值。

让我们通过再次执行`sysctl`来验证它是否被正确应用。

```
# sysctl fs.file-max
fs.file-max = 125000

```

事实上，似乎值已经被正确应用了，但是将它设置为`525000`呢？

### 临时更改可调整的值

虽然更改`/etc/sysctl.conf`到一个更高的值，然后应用并恢复更改可能很简单。但是有一个更简单的方法可以临时更改可调整的值。

当提供`-w`选项时，`sysctl`命令允许修改可调整的值。为了看到这一点，我们将使用它将`fs.file-max`值设置为`525000`。

```
# sysctl -w fs.file-max=525000
fs.file-max = 525000

```

就像我们应用`sysctl.conf`文件的值一样，当我们执行`sysctl –w`时，它打印了应用的值。如果我们再次验证它们，我们会看到值被设置为`525000`个文件：

```
# sysctl fs.file-max
fs.file-max = 525000

```

## 最后再次运行作业

现在我们已经将 vagrant 用户的`打开文件`限制设置为`500000`，整个系统设置为`525000`。我们可以再次手动执行这个作业，这次应该会成功：

```
$ /opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml
Initializing with configuration file /opt/myapp/conf/config.yml
- - - - - - - - - - - - - - - - - - - - - - - - - -
Starting message processing job
Added 492304 to queue
Processing 492304 messages
Processed 492304 messages

```

这次作业执行时没有提供任何错误！我们可以从作业的输出中看到`/opt/myapp/queue`中的所有文件都被处理了。

# 回顾一下

现在我们已经解决了问题，让我们花点时间看看我们是如何解决这个问题的。

## 打开文件太多

为了排除我们的问题，我们手动执行了一个预定的 cron 作业。如果我们回顾之前的章节，这是一个复制问题并亲自看到的一个典型例子。

在这种情况下，作业没有执行它应该执行的任务。为了找出原因，我们手动运行了它。

在手动执行期间，我们能够识别出以下错误：

```
IOError: [Errno 24] Too many open files: '/opt/myapp/queue/1433955823.29_0.txt'

```

这种错误非常常见，是由作业超出用户限制而导致的，该限制阻止单个用户打开太多文件。为了解决这个问题，我们在`/etc/security/limits.conf`文件中添加了自定义设置。

这些更改将我们用户的“打开文件”限制默认设置为`100000`。我们还允许用户通过`hard`设置临时将“打开文件”限制增加到`500000`：

```
IOError: [Errno 23] Too many open files in system: '/opt/myapp/queue/1433955989.86_5.txt'

```

修改这些限制后，我们再次执行了作业，并遇到了类似但不同的错误。

这次，“打开文件”限制是系统本身施加的，这种情况下对系统施加了全局限制，即 48000 个打开文件。

为了解决这个问题，我们在`/etc/sysctl.conf`文件中设置了永久设置为`125000`，并临时将该值更改为`525000`。

从那时起，我们能够手动执行作业。然而，自从我们改变了默认限制以来，我们还为这个作业提供了更多资源以正常执行。只要没有超过 10 万个文件的积压，这个作业将来都应该能够正常执行。

## 稍微整理一下。

说到正常执行，为了减少内核对打开文件的限制，我们可以再次执行`sysctl`命令，并加上`-p`选项。这将把值重置为`/etc/sysctl.conf`文件中定义的值。

```
# sysctl -p
fs.file-max = 125000

```

这种方法的一个注意事项是，`sysctl -p`只会重置`/etc/sysctl.conf`中指定的值；*默认情况下只包含少量可调整的值*。如果修改了`/etc/sysctl.conf`中未指定的值，`sysctl -p`方法将不会将该值重置为默认值。

# 总结

在本章中，我们对 Linux 中强制执行的内核和用户限制非常熟悉。这些设置非常有用，因为任何利用许多资源的应用程序最终都会遇到其中之一。

在下一章中，我们将专注于一个非常常见但非常棘手的问题。我们将专注于故障排除和确定系统内存耗尽的原因。当系统内存耗尽时，会有很多后果，比如应用程序进程被终止。
