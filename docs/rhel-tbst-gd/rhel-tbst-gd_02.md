# 第二章：故障排除命令和有用信息的来源

在第一章中，我们介绍了故障排除的最佳实践和涉及的高级流程。第一章是对故障排除的 20,000 英尺视图，而本章开始深入具体内容。

本章将回顾常见的故障排除命令以及查找有用信息的常见位置。在本书中，我们将使用 Red Hat Enterprise Linux 的 7 版（也称为 RHEL）。本章引用的所有命令都是 RHEL 7 默认安装包中包含的命令。

我们将引用默认安装的命令，因为我发现自己曾经处于这样的情况，我本可以使用特定的命令立即识别问题，但是这个命令对我不可用。通过将本章限制为默认命令，您可以确保本章涵盖的故障排除步骤不仅与大多数 RHEL 7 安装相关，而且与以前的版本和其他 Linux 发行版相关。

# 查找有用信息

在开始探索故障排除命令之前，我首先想要介绍有用信息的位置。有用信息是一个模糊的术语，几乎每个文件、目录或命令都可以提供*有用信息*。我真正打算介绍的是几乎可以找到几乎任何问题的信息的位置。

## 日志文件

日志文件通常是查找故障排除信息的第一个地方。每当服务或服务器遇到问题时，检查错误日志文件通常可以迅速回答许多问题。

### 默认位置

默认情况下，RHEL 和大多数 Linux 发行版将其日志文件保存在`/var/log/`中，这实际上是由 Linux 基金会维护的**文件系统层次结构标准**（**FHS**）的一部分。但是，虽然`/var/log/`可能是默认位置，并非所有日志文件都位于那里（[`en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard`](http://en.wikipedia.org/wiki/Filesystem_Hierarchy_Standard)）。

虽然`/var/log/httpd/`是 Apache 日志的默认位置，但可以通过 Apache 的配置文件更改此位置。当 Apache 安装在标准 RHEL 软件包之外时，这是非常常见的。

与 Apache 一样，大多数服务允许自定义日志位置。在`/var/log`之外找到专门用于日志文件的自定义目录或文件系统并不罕见。

### 常见日志文件

以下表格是常见日志文件的简要列表，以及您可以在其中找到的内容的描述。

### 提示

请记住，此列表特定于 Red Hat Enterprise Linux 7，而其他 Linux 发行版可能遵循类似的约定，但不能保证。

| 日志文件 | 描述 |
| --- | --- |
| `/var/log/messages` | 默认情况下，此日志文件包含所有`INFO`或更高优先级的 syslog 消息（除电子邮件）。 |

| `/var/log/secure` | 此日志文件包含与身份验证相关的消息项，例如：

+   SSH 登录

+   用户创建

+   Sudo 违规和权限提升

|

| `/var/log/cron` | 此日志文件包含`crond`执行的历史记录，以及`cron.daily`、`cron.weekly`和其他执行的开始和结束时间。 |
| --- | --- |
| `/var/log/maillog` | 这个日志文件是邮件事件的默认日志位置。如果使用 postfix，这是所有与 postfix 相关的消息的默认位置。 |
| `/var/log/httpd/` | 此日志目录是 Apache 日志的默认位置。虽然这是默认位置，但并不是所有 Apache 日志的保证位置。 |
| `/var/log/mysql.log` | 这个日志文件是 mysqld 的默认日志文件。与`httpd`日志一样，这是默认的，可以很容易地更改。 |
| `/var/log/sa/` | 此目录包含默认每 10 分钟运行一次的`sa`命令的结果。我们将在本章的后续部分以及本书的整个过程中更多地利用这些数据。 |

对于许多问题，要审查的第一个日志文件之一是`/var/log/messages`日志。在 RHEL 系统上，这个日志文件接收所有`INFO`优先级或更高级别的系统日志。一般来说，这意味着发送到`syslog`的任何重要事件都会在这个日志文件中被捕获。

以下是可以在`/var/log/messages`中找到的一些日志消息的示例：

```
Dec 24 18:03:51 localhost systemd: Starting Network Manager Script Dispatcher Service...
Dec 24 18:03:51 localhost dbus-daemon: dbus[620]: [system] Successfully activated service 'org.freedesktop.nm_dispatcher'
Dec 24 18:03:51 localhost dbus[620]: [system] Successfully activated service 'org.freedesktop.nm_dispatcher'
Dec 24 18:03:51 localhost systemd: Started Network Manager Script Dispatcher Service.
Dec 24 18:06:06 localhost kernel: e1000: enp0s3 NIC Link is Down
Dec 24 18:06:06 localhost kernel: e1000: enp0s8 NIC Link is Down
Dec 24 18:06:06 localhost NetworkManager[750]: <info> (enp0s3): link disconnected (deferring action for 4 seconds)
Dec 24 18:06:06 localhost NetworkManager[750]: <info> (enp0s8): link disconnected (deferring action for 4 seconds)
Dec 24 18:06:10 localhost NetworkManager[750]: <info> (enp0s3): link disconnected (calling deferred action)
Dec 24 18:06:10 localhost NetworkManager[750]: <info> (enp0s8): link disconnected (calling deferred action)
Dec 24 18:06:12 localhost kernel: e1000: enp0s3 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
Dec 24 18:06:12 localhost kernel: e1000: enp0s8 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
Dec 24 18:06:12 localhost NetworkManager[750]: <info> (enp0s3): link connected
Dec 24 18:06:12 localhost NetworkManager[750]: <info> (enp0s8): link connected
Dec 24 18:06:39 localhost kernel: atkbd serio0: Spurious NAK on isa0060/serio0\. Some program might be trying to access hardware directly.
Dec 24 18:07:10 localhost systemd: Starting Session 53 of user root.
Dec 24 18:07:10 localhost systemd: Started Session 53 of user root.
Dec 24 18:07:10 localhost systemd-logind: New session 53 of user root.
```

正如我们所看到的，在这个示例中有不止一条日志消息可能在故障排除问题时非常有用。

### 寻找不在默认位置的日志

很多时候日志文件不在`/var/log/`中，这可能是因为有人修改了日志位置到默认位置之外的某个地方，或者仅仅是因为相关服务默认使用另一个位置。

一般来说，有三种方法可以找到不在`/var/log/`中的日志文件。

#### 检查 syslog 配置

如果您知道某个服务正在使用 syslog 进行日志记录，查找其消息写入的日志文件的最佳位置是**rsyslog**配置文件。rsyslog 服务有两个配置位置。第一个是`/etc/rsyslog.d`目录。

`/etc/rsyslog.d`目录是自定义 rsyslog 配置的包含目录。第二个是`/etc/rsyslog.conf`配置文件。这是 rsyslog 的主配置文件，包含许多默认的 syslog 配置。

以下是`/etc/rsyslog.conf`的默认内容示例：

```
#### RULES ####

# Log all kernel messages to the console.
# Logging much else clutters up the screen.
#kern.*                              /dev/console

# Log anything (except mail) of level info or higher.
# Don't log private authentication messages!
*.info;mail.none;authpriv.none;cron.none  /var/log/messages

# The authpriv file has restricted access.
authpriv.*                           /var/log/secure

# Log all the mail messages in one place.
mail.*                              -/var/log/maillog

# Log cron stuff
cron.*                               /var/log/cron
```

通过审查这个文件的内容，很容易确定哪些日志文件包含所需的信息，如果不行，至少可以确定 syslog 管理的日志文件的可能位置。

#### 检查应用程序的配置

并非每个应用程序都使用 syslog；对于那些不使用的应用程序，找到应用程序的日志文件的最简单方法之一是阅读应用程序的配置文件。

从配置文件中查找日志文件位置的一种快速有用的方法是使用`grep`命令在文件中搜索单词`log`：

```
$ grep log /etc/samba/smb.conf
# files are rotated when they reach the size specified with "max log size".
  # log files split per-machine:
  log file = /var/log/samba/log.%m
  # maximum size of 50KB per log file, then rotate:
  max log size = 50
```

```
grep command is used to search the /etc/samba/smb.conf file for any instance of the pattern "log".
```

在审查上述`grep`命令的输出后，我们可以看到 samba 的配置日志位置是`/var/log/samba/log.%m`。需要注意的是，在这个例子中，`%m`实际上是在创建文件时用“机器名称”替换的。这实际上是 samba 配置文件中的一个变量。这些变量对每个应用程序都是唯一的，但这种动态配置值的方法是一种常见的做法。

##### 其他例子

以下是使用`grep`命令在 Apache 和 MySQL 配置文件中搜索单词“`log`”的示例：

```
$ grep log /etc/httpd/conf/httpd.conf
# ErrorLog: The location of the error log file.
# logged here.  If you *do* define an error logfile for a <VirtualHost>
# container, that host's errors will be logged there and not here.
ErrorLog "logs/error_log"

$ grep log /etc/my.cnf
# log_bin
log-error=/var/log/mysqld.log
```

在这两种情况下，这种方法能够识别服务日志文件的配置参数。通过前面的三个例子，很容易看出搜索配置文件的效果有多好。

#### 使用 find 命令

我们将在本章后面深入介绍的`find`命令是另一种查找日志文件的有用方法。`find`命令用于在目录结构中搜索指定的文件。查找日志文件的快速方法是简单地使用`find`命令搜索以“`.log`”结尾的任何文件：

```
# find /opt/appxyz/ -type f -name "*.log"
/opt/appxyz/logs/daily/7-1-15/alert.log
/opt/appxyz/logs/daily/7-2-15/alert.log
/opt/appxyz/logs/daily/7-3-15/alert.log
/opt/appxyz/logs/daily/7-4-15/alert.log
/opt/appxyz/logs/daily/7-5-15/alert.log
```

上述通常被认为是最后的解决方案，大多数情况下是在之前的方法没有产生结果时使用。

### 提示

执行`find`命令时，最好的做法是非常具体地指定要搜索的目录。当针对非常大的目录执行时，服务器的性能可能会下降。

## 配置文件

如前所述，应用程序或服务的配置文件可以是信息的绝佳来源。虽然配置文件不会提供特定的错误，比如日志文件，但它们可以提供关键信息（例如启用/禁用的功能、输出目录和日志文件位置）。

### 默认系统配置目录

一般来说，大多数 Linux 发行版的系统和服务配置文件位于`/etc/`目录中。但这并不意味着每个配置文件都位于`/etc/`目录中。事实上，应用程序通常会在应用程序的`home`目录中包含一个配置目录。

那么，如何知道何时在`/etc/`而不是应用程序目录中查找配置文件？一个经验法则是，如果软件包是 RHEL 发行版的一部分，可以安全地假设配置位于`/etc/`目录中。其他任何东西可能存在于`/etc/`目录中，也可能不存在。对于这些情况，你只需要去寻找它们。

### 查找配置文件

在大多数情况下，可以使用`ls`命令对`/etc/`目录进行简单的目录列表，以找到系统配置文件：

```
$ ls -la /etc/ | grep my
-rw-r--r--.  1 root root      570 Nov 17  2014 my.cnf
drwxr-xr-x.  2 root root       64 Jan  9  2015 my.cnf.d
```

```
ls to perform a directory listing and redirects that output to grep in order to search the output for the string "my". We can see from the output that there is a my.cnf configuration file and a my.cnf.d configuration directory. The MySQL processes use these for its configuration. We were able to find these by assuming that anything related to MySQL would have the string "my" in it.
```

#### 使用 rpm 命令

如果配置文件是作为 RPM 软件包的一部分部署的，可以使用`rpm`命令来识别配置文件。为此，只需执行带有`-q`（查询）标志和`-c`（configfiles）标志的`rpm`命令，然后跟上软件包的名称：

```
$ rpm -q -c httpd
/etc/httpd/conf.d/autoindex.conf
/etc/httpd/conf.d/userdir.conf
/etc/httpd/conf.d/welcome.conf
/etc/httpd/conf.modules.d/00-base.conf
/etc/httpd/conf.modules.d/00-dav.conf
/etc/httpd/conf.modules.d/00-lua.conf
/etc/httpd/conf.modules.d/00-mpm.conf
/etc/httpd/conf.modules.d/00-proxy.conf
/etc/httpd/conf.modules.d/00-systemd.conf
/etc/httpd/conf.modules.d/01-cgi.conf
/etc/httpd/conf/httpd.conf
/etc/httpd/conf/magic
/etc/logrotate.d/httpd
/etc/sysconfig/htcacheclean
/etc/sysconfig/httpd
```

`rpm`命令用于管理 RPM 软件包，在故障排除时非常有用。在下一节中，我们将进一步介绍这个命令，以探索故障排除的命令。

#### 使用 find 命令

与查找日志文件类似，要在系统上查找配置文件，可以利用`find`命令。在搜索日志文件时，`find`命令用于搜索所有文件名以“`.log`”结尾的文件。在下面的例子中，`find`命令用于搜索所有文件名以“`http`”开头的文件。这个`find`命令应该至少返回一些结果，这些结果将提供与 HTTPD（Apache）服务相关的配置文件：

```
# find /etc -type f -name "http*"

/etc/httpd/conf/httpd.conf
/etc/sysconfig/httpd
/etc/logrotate.d/httpd
```

前面的例子搜索了`/etc`目录；然而，这也可以用于搜索任何应用程序的主目录以查找用户配置文件。与搜索日志文件类似，使用`find`命令搜索配置文件通常被认为是最后的手段，不应该是第一个使用的方法。

## proc 文件系统

`proc`文件系统是一个非常有用的信息来源。这是由 Linux 内核维护的一个特殊文件系统。`proc`文件系统可用于查找有关运行进程以及其他系统信息的有用信息。例如，如果我们想要识别系统支持的文件系统，我们可以简单地读取`/proc/filesystems`文件：

```
$ cat /proc/filesystems
nodev  sysfs
nodev  rootfs
nodev  bdev
nodev  proc
nodev  cgroup
nodev  cpuset
nodev  tmpfs
nodev  devtmpfs
nodev  debugfs
nodev  securityfs
nodev  sockfs
nodev  pipefs
nodev  anon_inodefs
nodev  configfs
nodev  devpts
nodev  ramfs
nodev  hugetlbfs
nodev  autofs
nodev  pstore
nodev  mqueue
nodev  selinuxfs
  xfs
nodev  rpc_pipefs
nodev  nfsd
```

这个文件系统非常有用，包含了关于运行系统的大量信息。`proc 文件系统`将在本书的故障排除步骤中使用。它在故障排除各种问题时以不同的方式使用，从特定进程到只读文件系统。

# 故障排除命令

本节将介绍经常使用的故障排除命令，这些命令可用于从系统或运行的服务中收集信息。虽然不可能涵盖每个可能的命令，但所使用的命令确实涵盖了 Linux 系统的基本故障排除步骤。

## 命令行基础知识

本书中使用的故障排除步骤主要基于命令行。虽然可能可以从图形桌面环境执行许多这些操作，但更高级的项目是命令行特定的。因此，本书假设读者至少具有对 Linux 的基本理解。更具体地说，本书假设读者已经通过 SSH 登录到服务器，并熟悉基本命令，如`cd`、`cp`、`mv`、`rm`和`ls`。

对于那些可能不太熟悉的人，我想快速介绍一些基本的命令行用法，这将是本书所需的基本知识。

### 命令标志

许多读者可能熟悉以下命令：

```
$ ls -la
total 588
drwx------. 5 vagrant vagrant   4096 Jul  4 21:26 .
drwxr-xr-x. 3 root    root        20 Jul 22  2014 ..
-rw-rw-r--. 1 vagrant vagrant 153104 Jun 10 17:03 app.c
```

大多数人应该认识到这是`ls`命令，用于执行目录列表。可能不熟悉的是命令的“-la”部分是什么或者做什么。为了更好地理解这一点，让我们单独看一下 ls 命令：

```
$ ls
app.c  application  app.py  bomber.py  index.html  lookbusy-1.4  lookbusy-1.4.tar.gz  lotsofiles
```

先前执行的`ls`命令与以前的看起来非常不同。这是因为后者是`ls`的默认输出。命令标志允许用户更改命令的默认行为，为其提供特定选项。

实际上，“-la”标志是两个单独的选项，“-l”和“-a”;它们甚至可以分开指定:

```
 $ ls -l -a
total 588
drwx------. 5 vagrant vagrant   4096 Jul  4 21:26 .
drwxr-xr-x. 3 root    root        20 Jul 22  2014 ..
-rw-rw-r--. 1 vagrant vagrant 153104 Jun 10 17:03 app.c
```

```
ls –la is exactly the same as ls –l –a. For common commands, such as the ls command, it does not matter if the flags are grouped or separated, they will be parsed in the same way. Throughout this book, examples will show both grouped and ungrouped. If grouping or ungrouping is performed for any specific reason it will be called out; otherwise, the grouping or ungrouping used within this book is used for visual appeal and memorization.
```

除了分组和取消分组，本书还将以长格式显示标志。在前面的例子中，我们显示了标志`-a`，这被称为短标志。这个选项也可以以长格式`--all`提供：

```
$ ls -l --all
total 588
drwx------. 5 vagrant vagrant   4096 Jul  4 21:26 .
drwxr-xr-x. 3 root    root        20 Jul 22  2014 ..
-rw-rw-r--. 1 vagrant vagrant 153104 Jun 10 17:03 app.c
```

“-a”和`--all`标志本质上是相同的选项;它可以简单地以短格式和长格式表示。

一个重要的事情要记住的是，并非每个短标志都有长形式，反之亦然。每个命令都有自己的语法，有些命令只支持短形式，其他命令只支持长形式，但许多命令都支持两种形式。在大多数情况下，长标志和短标志都将在命令的手册页面中得到记录。

### 管道命令输出

本书中将多次使用的另一种常见的命令行实践是将输出“传输”。具体来说，例如以下示例：

```
$ ls -l --all | grep app
-rw-rw-r--. 1 vagrant vagrant 153104 Jun 10 17:03 app.c
-rwxrwxr-x. 1 vagrant vagrant  29390 May 18 00:47 application
-rw-rw-r--. 1 vagrant vagrant   1198 Jun 10 17:03 app.py
```

在前面的例子中，`ls -l --all`的输出被传输到`grep`命令。通过在两个命令之间放置`|`或管道字符，第一个命令的输出被“传输”到第二个命令的输入。将执行`ls`命令的示例;随后，`grep`命令将搜索该输出中的任何`app`模式的实例。

在本书中，将经常使用将输出传输到`grep`，因为这是将输出修剪为可维护大小的简单方法。许多时候，示例还将包含多个级别的管道:

```
$ ls -la | grep app | awk '{print $4,$9}'
vagrant app.c
vagrant application
vagrant app.py
```

在前面的代码中，`ls -la`的输出被传输到`grep`的输入;然而，这一次，`grep`的输出也被传输到`awk`的输入。

虽然许多命令可以进行管道传输，但并非每个命令都支持这一点。一般来说，接受来自文件或命令行的用户输入的命令也接受管道输入。与标志一样，命令的手册页面可用于确定命令是否接受管道输入。

## 收集一般信息

在长时间管理相同服务器时，您开始记住关于这些服务器的关键信息。例如物理内存的数量，文件系统的大小和布局，以及应该运行的进程。但是，当您不熟悉所讨论的服务器时，收集这种类型的信息总是一个好主意。

本节中的命令是用于收集此类一般信息的命令。

### w-显示谁登录了以及他们在做什么

在我系统管理职业生涯的早期，我有一个导师告诉我：*我每次登录服务器时都会运行 w*。这个简单的提示实际上在我的职业生涯中一次又一次地非常有用。`w`命令很简单;当执行时，它将输出系统正常运行时间，平均负载以及谁登录了：

```
# w
 04:07:37 up 14:26,  2 users,  load average: 0.00, 0.01, 0.05
USER     TTY        LOGIN@   IDLE   JCPU   PCPU WHAT
root     tty1      Wed13   11:24m  0.13s  0.13s -bash
root     pts/0     20:47    1.00s  0.21s  0.19s -bash
```

当与不熟悉的系统一起工作时，这些信息可能非常有用。即使您熟悉该系统，输出也可能很有用。通过这个命令，您可以看到：

+   上次系统重启时：

`04:07:37 up 14:26`：这些信息可能非常有用；无论是像 Apache 服务宕机的警报，还是用户因为被系统锁定而打进来。当这些问题是由意外重启引起时，报告的问题通常不包括这些信息。通过运行`w`命令，很容易看到自上次重启以来经过的时间。

+   系统的平均负载：

`平均负载：0.00, 0.01, 0.05`：平均负载是系统健康的一个非常重要的衡量标准。总结一下，平均负载是一段时间内处于`等待`状态的进程的平均数量。`w`命令输出中的三个数字代表不同的时间。

这些数字从左到右依次是 1 分钟、5 分钟和 15 分钟。

+   谁登录了以及他们在运行什么：

+   `USER TTY LOGIN@ IDLE JCPU PCPU WHAT`

+   `root tty1 Wed13 11:24m 0.13s 0.13s -bash`

`w`命令提供的最后一条信息是当前登录的用户以及他们正在执行的命令。

这基本上与`who`命令的输出相同，包括已登录的用户、他们登录的时间、他们已经空闲了多长时间，以及他们的 shell 正在运行的命令。列表中的最后一项非常重要。

在与大团队合作时，往往会有多个人响应一个问题或工单是很常见的。在登录后立即运行`w`命令，你将看到其他用户在做什么，避免你覆盖其他人已经采取的故障排除或纠正步骤。

### rpm – RPM 软件包管理器

`rpm`命令用于管理**Red Hat 软件包管理器**（**RPM**）。使用这个命令，你可以安装和删除 RPM 软件包，以及搜索已安装的软件包。

在本章的前面，我们看到`rpm`命令可以用来查找配置文件。以下是我们可以使用`rpm`命令查找关键信息的几种额外方式。

#### 列出所有安装的软件包

在故障排除服务时，一个关键的步骤是确定服务的版本以及它是如何安装的。要列出系统上安装的所有 RPM 软件包，只需执行带有`-q`（查询）和`-a`（所有）的`rpm`命令：

```
# rpm -q -a
kpatch-0.0-1.el7.noarch
virt-what-1.13-5.el7.x86_64
filesystem-3.2-18.el7.x86_64
gssproxy-0.3.0-9.el7.x86_64
hicolor-icon-theme-0.12-7.el7.noarch
```

`rpm`命令是一个非常多样化的命令，有很多标志。在前面的例子中使用了`-q`和`-a`标志。`-q`标志告诉`rpm`命令正在进行的操作是一个查询；你可以把它想象成进入了“搜索模式”。`-a`或`--all`标志告诉`rpm`命令列出所有软件包。

一个有用的功能是在前面的命令中添加`--last`标志，因为这会导致`rpm`命令按安装时间列出软件包，最新的排在最前面。

#### 列出软件包部署的所有文件

另一个有用的`rpm`功能是显示特定软件包部署的所有文件：

```
# rpm -q --filesbypkg kpatch-0.0-1.el7.noarch
kpatch                    /usr/bin/kpatch
kpatch                    /usr/lib/systemd/system/kpatch.service
```

在前面的例子中，我们再次使用`-q`标志来指定我们正在运行一个查询，以及`--filesbypkg`标志。`--filesbypkg`标志将导致`rpm`命令列出指定软件包部署的所有文件。

当试图确定服务的配置文件位置时，这个例子非常有用。

#### 使用软件包验证

在这第三个例子中，我们将使用`rpm`的一个非常有用的功能——验证。`rpm`命令有能力验证指定软件包部署的文件是否已经被更改。为了做到这一点，我们将使用`-V`（验证）标志：

```
# rpm -V httpd
S.5....T.  c /etc/httpd/conf/httpd.conf
```

在前面的例子中，我们只是运行了带有`-V`标志的`rpm`命令，后面跟着一个软件包名称。由于`-q`标志用于查询，`-V`标志用于验证。使用这个命令，我们可以看到只有`/etc/httpd/conf/httpd.conf`文件被列出；这是因为`rpm`只会输出已经被更改的文件。

在这个输出的第一列中，我们可以看到文件失败的验证检查。虽然这一列起初有点神秘，但 rpm 手册中有一个有用的表格（如下列表所示），解释了每个字符的含义：

+   `S`: 这意味着文件大小不同

+   `M`: 这意味着模式不同（包括权限和文件类型）

+   `5`: 这意味着摘要（以前是`MD5 校验和`）不同

+   `D`: 这意味着设备主/次编号不匹配

+   `L`: 这意味着`readLink(2)`路径不匹配

+   `U`: 这意味着用户所有权不同

+   `G`: 这意味着组所有权不同

+   `T`: 这意味着`mTime`不同

+   `P`: 这意味着`caPabilities`不同

使用这个列表，我们可以看到`httpd`.`conf`文件大小、`MD5`校验和和`mtime`（修改时间）与`httpd.rpm`部署的不同。这意味着很可能`httpd.conf`文件在安装后被修改过。

虽然`rpm`命令一开始可能不像是一个故障排除命令，但前面的例子显示了它实际上是一个多么强大的故障排除工具。通过这些例子，很容易识别重要文件以及这些文件是否已经从部署版本中修改。

### df - 报告文件系统空间使用情况

`df`命令在故障排除文件系统问题时非常有用。`df`命令用于输出已挂载文件系统的空间利用情况：

```
# df -h
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root  6.7G  1.6G  5.2G  24% /
devtmpfs               489M     0  489M   0% /dev
tmpfs                  498M     0  498M   0% /dev/shm
tmpfs                  498M   13M  485M   3% /run
tmpfs                  498M     0  498M   0% /sys/fs/cgroup
/dev/sdb1              212G   58G  144G  29% /repos
/dev/sda1              497M  117M  380M  24% /boot
```

在前面的例子中，`df`命令包括了`-h`标志。这个标志会导致`df`命令以“人类可读”的格式打印任何大小值。默认情况下，`df`会简单地以千字节打印这些值。从例子中，我们可以快速看到所有挂载文件系统的当前使用情况。具体来说，如果我们看输出，我们可以看到`/filesystem`目前使用了 24%：

```
Filesystem             Size  Used Avail Use% Mounted on
/dev/mapper/rhel-root  6.7G  1.6G  5.2G  24% /
```

这是一个非常快速简单的方法来识别任何文件系统是否已满。此外，`df`命令还非常有用，可以显示已挂载的文件系统的详细信息以及它们被挂载到哪里。从包含`/filesystem`的那一行，我们可以看到底层设备是`/dev/mapper/rhel-root`。

通过这个命令，我们能够识别出两个关键信息。

#### 显示可用的 inode

`df`的默认行为是显示已使用的文件系统空间量。但是，它也可以用来显示每个文件系统可用、已使用和空闲的**inodes**数量。要输出 inode 利用率，只需在执行`df`命令时添加`-i`（inode）标志：

```
# df -i
Filesystem              Inodes IUsed    IFree IUse% Mounted on
/dev/mapper/rhel-root  7032832 44318  6988514    1% /
devtmpfs                125039   347   124692    1% /dev
```

仍然可以使用`-h`标志与`df`一起以人类可读的格式打印输出。但是，使用`-i`标志，这将把输出缩写为`M`表示百万，`K`表示千，依此类推。这个输出很容易与兆字节或千字节混淆，所以一般情况下，我不会在与其他用户/管理员共享输出时使用人类可读的 inode 输出。

### free - 显示内存利用率

执行`free`命令时，将输出系统上可用内存和已使用内存的统计信息：

```
$ free
             total       used       free     shared    buffers     cached
Mem:       1018256     789796     228460      13116       3608     543484
-/+ buffers/cache:     242704     775552
Swap:       839676          4     839672
```

从前面的例子可以看出，`free`命令的输出提供了总可用内存、当前使用的内存量和空闲内存量。`free`命令是识别系统内存当前状态的一种简单快速的方式。

然而，`free`的输出一开始可能有点令人困惑。

#### 所谓的空闲，并不总是空闲

Linux 与其他操作系统相比，利用内存的方式不同。在前面的输出中，您将看到有 543,484 KB 被列为缓存。这个内存，虽然在技术上被使用，实际上是可用内存的一部分。系统可以根据需要重新分配这个缓存内存。

一个快速简单的方法来查看实际使用或空闲的内容可以在输出的第二行看到。前面的输出显示系统上有 775,552 KB 的内存可用。

#### /proc/meminfo 文件

在以前的 RHEL 版本中，`free`命令的第二行是识别可用内存量的最简单方法。但是，随着 RHEL 7，`/proc/meminfo`文件已经进行了一些改进。其中一个改进是增加了**MemAvailable**统计信息：

```
$ grep Available /proc/meminfo
MemAvailable:     641056 kB
```

`/proc/meminfo`文件是位于`/proc`文件系统中的许多有用文件之一。该文件由内核维护，包含系统当前的内存统计信息。在排除内存问题时，此文件非常有用，因为它包含的信息比`free`命令的输出要多得多。

### ps – 报告当前运行进程的快照

`ps`命令是任何故障排除活动的基本命令。执行此命令将输出运行进程的列表：

```
# ps
  PID TTY          TIME CMD
15618 pts/0    00:00:00 ps
17633 pts/0    00:00:00 bash
```

`ps`命令有许多标志和选项，可显示有关运行进程的不同信息。以下是一些在故障排除期间有用的`ps`命令示例。

#### 以长格式打印每个进程

以下`ps`命令使用`-e`（所有进程）、`-l`（长格式）和`-f`（完整格式）标志。这些标志将导致`ps`命令不仅打印每个进程，还将以提供相当多有用信息的格式打印它们：

```
# ps -elf
F S UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY   TIME CMD
1 S root   2     0   0  80  0 - 0 kthrea Dec24 ?   00:00:00 [kthreadd]
```

在`ps -elf`的前面输出中，我们可以看到`kthreadd`进程的许多有用信息，例如**父进程 ID**（**PPID**）、**优先级**（**PRI**）、**niceness 值**（**NI**）和运行进程的**驻留内存大小**（**SZ**）。

我发现前面的示例是一个非常通用的`ps`命令，可以在大多数情况下使用。

#### 打印特定用户的进程

前面的示例可能会变得非常庞大，使得难以识别特定进程。此示例使用`-U`标志来指定用户。这会导致`ps`命令打印作为指定用户运行的所有进程；在以下情况下是后缀：

```
ps -U postfix -l
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY       TIME CMD
4 S    89  1546  1536  0  80   0 - 23516 ep_pol ?    00:00:00 qmgr
4 S    89 16711  1536  0  80   0 - 23686 ep_pol ?  00:00:00 pickup
```

需要注意的是，`–U`标志也可以与其他标志结合使用，以提供有关运行进程的更多信息。在前面的示例中，`-l`标志再次用于以长格式打印输出。

#### 按进程 ID 打印进程

如果进程 ID 或 PID 已知，可以通过指定`–p`（进程 ID）标志来进一步缩小进程列表：

```
# ps -p 1236 -l
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY       TIME CMD
4 S     0  1236     1  0  80   0 - 20739 poll_s ?    00:00:00 sshd
```

当与`–L`（显示带有 LWP 列的线程）或`–m`（显示进程后的线程）标志结合使用时，这可能特别有用，这些标志用于打印进程线程。在排除多线程应用程序故障时，`-L`和`-m`标志可能至关重要。

#### 打印带有性能信息的进程

`ps`命令允许用户使用`-o`（用户定义格式）标志自定义打印的列：

```
# ps -U postfix -o pid,user,pcpu,vsz,cmd
  PID USER     %CPU    VSZ CMD
 1546 postfix   0.0  94064 qmgr -l -t unix -u
16711 postfix   0.0  94744 pickup -l -t unix -u
```

`–o`选项允许使用许多自定义列。在前面的版本中，我选择了与 top 命令中打印的类似的选项。

top 命令是最受欢迎的 Linux 故障排除命令之一。它用于按 CPU 使用率（默认情况下）显示前几个进程。在本章中，我选择省略 top 命令，因为我认为`ps`命令比 top 命令更基本和灵活。随着对`ps`命令的熟悉，学习和理解 top 命令将变得容易。

## 网络

网络对于任何系统管理员来说都是一项基本技能。没有正确配置的网络接口，服务器就没有多大用处。本节中的命令专门用于查找网络配置和当前状态。这些命令是必须学习的，因为它们不仅对故障排除有用，而且对日常设置和配置也很有用。

### ip – 显示和操作网络设置

`ip`命令用于管理网络设置，如接口配置、路由和基本上与网络相关的任何内容。虽然这些通常不被认为是故障排除任务，但`ip`命令也可以用于显示系统的网络配置。如果无法查找网络详细信息，如路由或设备配置，将很难排除与网络相关的问题。

以下示例展示了使用`ip`命令识别关键网络配置设置的各种方法。

#### 显示特定设备的 IP 地址配置

`ip`命令的一个核心用途是查找网络接口并显示其配置。为了做到这一点，我们将使用以下命令：

```
# ip addr show dev enp0s3
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:6e:35:18 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 45083sec preferred_lft 45083sec
    inet6 fe80::a00:27ff:fe6e:3518/64 scope link
       valid_lft forever preferred_lft forever
```

在前面的`ip`命令中，提供的第一个选项`addr`（地址）用于定义我们要查找的信息类型。第二个选项`show`告诉`ip`显示第一个选项的配置。第三个选项`dev`（设备）后面跟着所讨论的网络接口设备；`enp0s3`。如果省略了第三个选项，`ip`命令将显示所有网络设备的地址配置。

对于那些有经验的 RHEL 之前版本的人来说，设备名称`enp0s3`可能看起来有点奇怪。这个设备遵循了`systemd`引入的较新的网络设备命名方案。从 RHEL 7 开始，网络设备将使用基于设备驱动程序和 BIOS 详细信息的设备名称。

要了解更多关于 RHEL 7 的新命名方案的信息，请参考以下 URL：

[`access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/ch-Consistent_Network_Device_Naming.html`](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/ch-Consistent_Network_Device_Naming.html)

#### 显示路由配置

`ip`命令还可以用于显示路由配置。这些信息对于排除服务器之间的连接问题至关重要。

```
# ip route show
default via 10.0.2.2 dev enp0s3  proto static  metric 1024
10.0.2.0/24 dev enp0s3  proto kernel  scope link  src 10.0.2.15
192.168.56.0/24 dev enp0s8  proto kernel  scope link  src 192.168.56.101
```

前面的`ip`命令使用`route`选项，后面跟着`show`选项来显示此服务器的所有定义路由。与前面的例子一样，也可以通过添加`dev`（设备）选项后跟设备名称来限制此输出到特定设备：

```
# ip route show dev enp0s3
default via 10.0.2.2  proto static  metric 1024
10.0.2.0/24  proto kernel  scope link  src 10.0.2.15
```

#### 显示指定设备的网络统计信息

前面的例子展示了查找当前网络配置的方法，而这个命令使用`-s`（统计）标志来显示指定设备的网络统计信息：

```
# ip -s link show dev enp0s3
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DEFAULT qlen 1000
    link/ether 08:00:27:6e:35:18 brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast
    109717927  125911   0       0       0       0
    TX: bytes  packets  errors  dropped carrier collsns
    3944294    40127    0       0       0       0
```

在前面的例子中，使用了`link`（网络设备）选项来指定统计信息应该限制在指定的设备上。

显示的统计信息在排除丢包或识别哪个接口具有更高的网络利用率时非常有用。

### netstat - 网络统计

`netstat`命令是任何系统管理员工具包中的基本工具。这可以通过`netstat`命令普遍适用于即使不传统使用命令行进行管理的操作系统来看出。

#### 打印网络连接

`netstat`的主要用途之一是打印现有的已建立的网络连接。这可以通过简单执行`netstat`来完成；然而，如果使用了`-a`（所有）标志，输出也将包括监听端口：

```
# netstat -na
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address      Foreign Address    State
tcp        0      0 127.0.0.1:25       0.0.0.0:*          LISTEN
tcp        0      0 0.0.0.0:44969      0.0.0.0:*          LISTEN
tcp        0      0 0.0.0.0:111        0.0.0.0:*          LISTEN
tcp        0      0 0.0.0.0:22         0.0.0.0:*          LISTEN
tcp        0      0 192.168.56.101:22  192.168.56.1:50122 ESTABLISHED
tcp6       0      0 ::1:25               :::*               LISTEN
```

虽然在前面的`netstat`中使用了`-a`（所有）标志来打印所有监听端口，但`-n`标志用于强制输出为数字格式，例如打印 IP 地址而不是 DNS 主机名。

前面的例子将在第五章*网络故障排除*中大量使用，我们将在那里进行网络连接故障排除。

#### 打印所有监听 tcp 连接的端口

我曾经看到许多情况下，一个服务正在运行，并且可以通过`ps`命令看到；但是，客户端连接的端口没有绑定和监听。在解决服务的连接问题时，以下`netstat`命令非常有用：

```
# netstat -nlp --tcp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address State       PID/Program name
tcp        0      0 127.0.0.1:25            0.0.0.0:* LISTEN      1536/master
tcp        0      0 0.0.0.0:44969           0.0.0.0:* LISTEN      1270/rpc.statd
tcp        0      0 0.0.0.0:111             0.0.0.0:* LISTEN      1215/rpcbind
tcp        0      0 0.0.0.0:22              0.0.0.0:* LISTEN      1236/sshd
tcp6       0      0 ::1:25                  :::* LISTEN      1536/master
tcp6       0      0 :::111                  :::* LISTEN      1215/rpcbind
tcp6       0      0 :::22                   :::* LISTEN      1236/sshd
tcp6       0      0 :::46072                :::* LISTEN      1270/rpc.statd
```

前面的命令非常有用，因为它结合了三个有用的选项：

+   `–l`（监听）告诉`netstat`只列出正在监听的套接字

+   `--tcp`告诉`netstat`将输出限制为 TCP 连接

+   `–p`（程序）告诉`netstat`列出在该端口上监听的进程的 PID 和名称

#### 延迟

`netstat`的一个经常被忽视的选项是利用延迟功能。通过在命令的末尾添加一个数字，`netstat`将持续运行，并在执行之间休眠指定的秒数。

如果执行以下命令，`netstat`命令将每五秒打印所有正在监听的 TCP 套接字：

```
# netstat -nlp --tcp 5
```

延迟功能在调查网络连接问题时非常有用。因为它可以很容易地显示应用程序何时为新连接绑定端口。

## 性能

虽然我们稍微提到了使用`free`和`ps`等命令来解决性能问题，但本节将展示一些非常有用的命令，这些命令可以回答“为什么慢”的古老问题。

### iotop - 一个简单的类似 top 的 I/O 监视器

`iotop`命令是 Linux 上相对较新的命令。在以前的 RHEL 版本中，虽然可用，但默认情况下未安装`iotop`命令。`iotop`命令提供了一个类似 top 命令的界面，但它不是显示哪些进程正在使用最多的 CPU 时间或内存，而是显示按 I/O 利用率排序的进程：

```
# iotop
Total DISK READ :       0.00 B/s | Total DISK WRITE :       0.00 B/s
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       0.00 B/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN      IO COMMAND
 1536 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % master -w
    1 be/4 root        0.00 B/s    0.00 B/s  0.00 %  0.00 % systemd --switched-root --system --deserialize 23
```

与以前的一些命令不同，`iotop`非常专门用于显示正在使用 I/O 的进程。但是，有一些非常有用的标志可以改变`iotop`的默认行为。例如`–o`（仅）告诉`iotop`仅打印使用 I/O 的进程，而不是其默认行为打印所有进程。另一组有用的标志是`-q`（安静）和`–n`（迭代次数）。

连同`-o`标志一起，这些标志可以用来告诉`iotop`仅打印使用 I/O 的进程，而不清除下一次迭代的屏幕：

```
# iotop -o -q -n2
Total DISK READ :     0.00 B/s | Total DISK WRITE :       0.00 B/s
Actual DISK READ:     0.00 B/s | Actual DISK WRITE:       0.00 B/s
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN   IO   COMMAND
Total DISK READ :     0.00 B/s | Total DISK WRITE :       0.00 B/s
Actual DISK READ:     0.00 B/s | Actual DISK WRITE:       0.00 B/s
22965 be/4 root       0.00 B/s    0.00 B/s  0.00 %  0.03 % [kworker/0:3]
```

如果我们看一下前面的示例输出，我们可以看到`iotop`命令的两个独立迭代。但是，与以前的示例不同，输出是连续的，允许我们看到每次迭代时使用 I/O 的进程。

默认情况下，`iotop`迭代之间的延迟是 1 秒；但是，可以使用`-d`（延迟）标志进行修改。

### iostat - 报告 I/O 和 CPU 统计信息

`iotop`显示正在使用 I/O 的进程，`iostat`显示正在被利用的设备：

```
# iostat -t 1 2
Linux 3.10.0-123.el7.x86_64 (localhost.localdomain)   12/25/2014 _x86_64_  (1 CPU)

12/25/2014 03:20:10 PM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.11    0.00    0.17    0.01    0.00   99.72

Device:            tps    kB_read/s    kB_wrtn/s    kB_read kB_wrtn
sda               0.38         2.84         7.02     261526 646339
sdb               0.01         0.06         0.00       5449 12
dm-0              0.33         2.77         7.00     254948 644275
dm-1              0.00         0.01         0.00        936 4

12/25/2014 03:20:11 PM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.00    0.00    0.99    0.00    0.00   99.01

Device:            tps    kB_read/s    kB_wrtn/s    kB_read kB_wrtn
sda               0.00         0.00         0.00          0 0
sdb               0.00         0.00         0.00          0 0
dm-0              0.00         0.00         0.00          0 0
dm-1              0.00         0.00         0.00          0 0
```

前面的`iostat`命令使用`-t`（时间戳）标志在每个报告中打印时间戳。两个数字是间隔和计数值。在前面的示例中，`iostat`以一秒的间隔运行，总共迭代两次。

`iostat`命令对诊断与 I/O 相关的问题非常有用。但是，输出通常会产生误导。当执行时，第一个报告中提供的值是系统上次重启以来的平均值。随后的报告是自上一个报告以来的。在这个例子中，我们执行了两个报告，相隔一秒。您可以看到第一个报告中的数字比第二个报告中的数字要高得多。

因此，许多系统管理员简单地忽略第一个报告，但他们并不完全理解为什么。因此，对于不熟悉`iostat`的人来说，对第一个报告中的值做出反应并不罕见。

`iostat`命令有一个`-y`标志（省略第一个报告），这实际上会导致`iostat`省略第一个报告。这是一个很好的标志，可以教给那些可能不太熟悉使用`iostat`的用户。

#### 操纵输出

`iostat`命令也有一些非常有用的标志，允许您操纵它呈现数据的方式。例如`-p`（设备）标志允许您将统计信息限制为指定的设备，或者`-x`（扩展统计）将打印扩展统计信息：

```
# iostat -p sda -tx
Linux 3.10.0-123.el7.x86_64 (localhost.localdomain)   12/25/2014 _x86_64_  (1 CPU)

12/25/2014 03:38:00 PM
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.11    0.00    0.17    0.01    0.00   99.72

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.01     0.02    0.13    0.25     2.81     6.95 51.70     0.00    7.62    1.57   10.79   0.85   0.03
sda1              0.00     0.00    0.02    0.02     0.05     0.02 3.24     0.00    0.24    0.42    0.06   0.23   0.00
sda2              0.01     0.02    0.11    0.19     2.75     6.93 65.47     0.00    9.34    1.82   13.58   0.82   0.02
```

前面的示例使用了`-p`标志来指定`sda`设备，`-t`标志来打印时间戳，`-x`标志来打印扩展统计信息。在测量特定设备的 I/O 性能时，这些标志非常有用。

### vmstat - 报告虚拟内存统计信息

`iostat`用于报告有关磁盘 I/O 性能的统计信息，而`vmstat`用于报告有关内存使用和性能的统计信息：

```
# vmstat 1 3
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
2  0      4 225000   3608 544900    0    0     3     7   17   28 0  0 100  0  0
0  0      4 224992   3608 544900    0    0     0     0   19   19 0  0 100  0  0
0  0      4 224992   3608 544900    0    0     0     0    6    9 0  0 100  0  0
```

`vmstat`的语法与`iostat`非常相似，您可以在命令行参数中提供间隔和报告计数。与`iostat`一样，第一个报告实际上是自上次重启以来的平均值，随后的报告是自上一个报告以来的。不幸的是，与`iostat`命令不同，`vmstat`命令没有包含一个标志来省略第一个报告。因此，在大多数情况下，简单地忽略第一个报告是合适的。

虽然`vmstat`可能不包括省略第一个报告的标志，但它确实有一些非常有用的标志；例如`-m`（slabs），这会导致`vmstat`以定义的间隔输出系统的`slabinfo`，以及`-s`（stats），它会打印系统的内存统计的扩展报告：

```
# vmstat -stats
      1018256 K total memory
       793416 K used memory,
       290372 K active memory
       360660 K inactive memory
       224840 K free memory
         3608 K buffer memory
       544908 K swap cache
       839676 K total swap
            4 K used swap
       839672 K free swap
        10191 non-nice user cpu ticks
           67 nice user cpu ticks
        11353 system cpu ticks
      9389547 idle cpu ticks
          556 IO-wait cpu ticks
           33 IRQ cpu ticks
         4434 softirq cpu ticks
            0 stolen cpu ticks
       267011 pages paged in
       647220 pages paged out
            0 pages swapped in
            1 pages swapped out
      1619609 interrupts
      2662083 CPU context switches
   1419453695 boot time
        59061 forks
```

前面的代码是使用`-s`或`--stats`标志的示例。

### sar - 收集、报告或保存系统活动信息

一个非常有用的实用程序是`sar`命令，`sar`是`sysstat`软件包附带的实用程序。`sysstat`软件包包括各种实用程序，用于收集磁盘、CPU、内存和网络利用率等系统指标。默认情况下，这个收集将每 10 分钟运行一次，并作为`cron`作业在`/ettc/cron.d/sysstat`中执行。

虽然`sysstat`收集的数据可能非常有用，但在高性能环境中有时会删除这个软件包。因为系统利用率统计数据的收集可能会增加系统的利用率，导致性能下降。要查看`sysstat`软件包是否已安装，只需使用 rpm 命令和`-q`（查询）标志：

```
# rpm -q sysstat
sysstat-10.1.5-4.el7.x86_64
```

#### 使用 sar 命令

`sar`命令允许用户查看`sysstat`实用程序收集的信息。当不带标志执行`sar`命令时，将打印当天的 CPU 统计信息：

```
# sar | head -6
Linux 3.10.0-123.el7.x86_64 (localhost.localdomain)   12/25/2014   _x86_64_  (1 CPU)

12:00:01 AM     CPU     %user     %nice   %system   %iowait %steal     %idle
12:10:02 AM     all      0.05      0.00      0.20      0.01 0.00     99.74
12:20:01 AM     all      0.05      0.00      0.18      0.00 0.00     99.77
12:30:01 AM     all      0.06      0.00      0.25      0.00 0.00     99.69
```

每天午夜，`systat`收集器将创建一个新文件来存储收集的统计信息。要引用该文件中的统计信息，只需使用`-f`（文件）标志来针对指定的文件运行`sar`：

```
# sar -f /var/log/sa/sa13
Linux 3.10.0-123.el7.x86_64 (localhost.localdomain)   12/13/2014   _x86_64_  (1 CPU)

10:24:43 AM       LINUX RESTART

10:30:01 AM     CPU     %user     %nice   %system   %iowait %steal     %idle
10:40:01 AM     all      2.99      0.00      0.96      0.43 0.00     95.62
10:50:01 AM     all      9.70      0.00      2.17      0.00 0.00     88.13
11:00:01 AM     all      0.31      0.00      0.30      0.02 0.00     99.37
11:10:01 AM     all      1.20      0.00      0.41      0.01 0.00     98.38
11:20:01 AM     all      0.01      0.00      0.04      0.01 0.00     99.94
11:30:01 AM     all      0.92      0.07      0.42      0.01 0.00     98.59
11:40:01 AM     all      0.17      0.00      0.08      0.00 0.00     99.74
11:50:02 AM     all      0.01      0.00      0.03      0.00 0.00     99.96
```

在前面的代码中，指定的文件是`/var/log/sa/sa13`；这个文件包含了当月第 13 天的统计信息。

`sar`命令有许多有用的标志，远远不止在本章中列出的。以下列出了一些非常有用的标志：

+   -b：这会打印类似于`iostat`命令的 I/O 统计信息

+   `-n ALL`：这会打印所有网络设备的网络统计信息

+   `-R`：这会打印内存利用率统计信息

+   `-A`：这会打印所有收集的统计信息。它基本上等同于运行`sar -bBdHqrRSuvwWy -I SUM -I XALL -m ALL -n ALL -u ALL -P ALL`

虽然`sar`命令显示了许多统计信息，但我们已经涵盖了诸如`iostat`或`vmstat`之类的命令。`sar`命令最大的好处在于能够回顾过去的统计数据。当排除发生在短时间内或已经被缓解的性能问题时，这种能力是至关重要的。

# 总结

在本章中，您了解到日志文件、配置文件和`/proc`文件系统是故障排除过程中的关键信息来源。我们还介绍了许多基本故障排除命令的基本用法。

在阅读本章的过程中，您可能已经注意到，相当多的命令也用于日常生活中的非故障排除目的。如果我们回顾一下来自第一章 *故障排除最佳实践* 的故障排除过程，第一步包括信息收集。

虽然这些命令本身可能无法解释问题，但它们可以帮助收集有关问题的信息，从而实现更准确和快速的解决方案。熟悉这些基本命令对于您在故障排除过程中取得成功至关重要。

在接下来的几章中，我们将使用这些基本命令来解决现实世界中的问题。下一章将重点解决与基于 Web 的应用程序相关的问题。
