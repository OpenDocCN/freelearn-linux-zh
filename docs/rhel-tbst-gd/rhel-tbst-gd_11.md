# 第十一章：从常见故障中恢复

在上一章中，我们探讨了 Linux 服务器上存在的用户和系统限制。我们看了看现有的限制以及如何更改应用程序所需的默认值。

在本章中，我们将运用我们的故障排除技能来处理一个资源耗尽的系统。

# 报告的问题

今天的章节，就像其他章节一样，将以某人报告问题开始。报告的问题是 Apache 不再在服务器上运行，该服务器为公司的博客`blog.example.com`提供服务。

报告问题的另一位系统管理员解释说，有人报告博客宕机，当他登录服务器时，他发现 Apache 不再运行。在那时，我们的同行不确定接下来该怎么做，并请求我们的帮助。

## Apache 真的宕机了吗？

当报告某个服务宕机时，我们应该做的第一件事是验证它是否真的宕机。这本质上是我们故障排除过程中的*为自己复制*步骤。对于 Apache 这样的服务，我们也应该相当快地验证它是否真的宕机。

根据我的经验，我经常被告知服务宕机，而实际上并非如此。服务器可能出现问题，但技术上并没有宕机。上线或宕机的区别会改变我们需要执行的故障排除步骤。

因此，我总是首先执行的步骤是验证服务是否真的宕机，还是服务只是没有响应。

为了验证 Apache 是否真的宕机，我们将使用`ps`命令。正如我们之前学到的，这个命令将打印当前运行的进程列表。我们将将这个输出重定向到`grep`命令，以检查是否有`httpd`（Apache）服务的实例在运行：

```
# ps -elf | grep http
0 S root      2645  1974  0  80   0 - 28160 pipe_w 21:45 pts/0 00:00:00 grep --color=auto http

```

从上述`ps`命令的输出中，我们可以看到没有以`httpd`命名的进程在运行。在正常情况下，我们期望至少看到几行类似于以下示例的内容：

```
5 D apache    2383     1  0  80   0 - 115279 conges 20:58 ? 00:00:04 /usr/sbin/httpd -DFOREGROUND

```

由于在进程列表中找不到`httpd`进程，我们可以得出结论，Apache 实际上在这个系统上宕机了。现在的问题是，为什么？

## 为什么它宕机了？

在简单地启动 Apache 服务解决问题之前，我们将首先弄清楚为什么 Apache 服务没有运行。这是一个称为**根本原因分析**（**RCA**）的过程，这是一个正式的过程，用于了解最初导致问题的原因。

在下一章中，我们将对这个过程非常熟悉。在本章中，我们将保持简单，专注于为什么 Apache 没有运行。

我们要查看的第一个地方是`/var/log/httpd`中的 Apache 日志。在之前的章节中，我们在排除其他与 Web 服务器相关的问题时了解了这些日志。正如我们在之前的章节中看到的，应用程序和服务日志在确定服务发生了什么事情方面非常有帮助。

由于 Apache 不再运行，我们对最近发生的事件更感兴趣。如果服务遇到致命错误或被停止，应该在日志文件的末尾显示相应的消息。

因为我们只对最近发生的事件感兴趣，所以我们将使用`tail`命令显示`error_log`文件的最后 10 行。`error_log`文件是第一个要检查的日志，因为它是发生异常的最可能地方：

```
# tail /var/log/httpd/error_log
[Sun Jun 21 20:51:32.889455 2015] [mpm_prefork:notice] [pid 2218] AH00163: Apache/2.4.6  PHP/5.4.16 configured -- resuming normal operations
[Sun Jun 21 20:51:32.889690 2015] [core:notice] [pid 2218] AH00094: Command line: '/usr/sbin/httpd -D FOREGROUND'
[Sun Jun 21 20:51:33.892170 2015] [mpm_prefork:error] [pid 2218] AH00161: server reached MaxRequestWorkers setting, consider raising the MaxRequestWorkers setting
[Sun Jun 21 20:53:42.577787 2015] [mpm_prefork:notice] [pid 2218] AH00170: caught SIGWINCH, shutting down gracefully [Sun Jun 21 20:53:44.677885 2015] [core:notice] [pid 2249] SELinux policy enabled; httpd running as context system_u:system_r:httpd_t:s0
[Sun Jun 21 20:53:44.678919 2015] [suexec:notice] [pid 2249] AH01232: suEXEC mechanism enabled (wrapper: /usr/sbin/suexec)
[Sun Jun 21 20:53:44.703088 2015] [auth_digest:notice] [pid 2249] AH01757: generating secret for digest authentication ...
[Sun Jun 21 20:53:44.704046 2015] [lbmethod_heartbeat:notice] [pid 2249] AH02282: No slotmem from mod_heartmonitor
[Sun Jun 21 20:53:44.732504 2015] [mpm_prefork:notice] [pid 2249] AH00163: Apache/2.4.6  PHP/5.4.16 configured -- resuming normal operations
[Sun Jun 21 20:53:44.732568 2015] [core:notice] [pid 2249] AH00094: Command line: '/usr/sbin/httpd -D FOREGROUND'

```

从`error_log`文件内容中，我们可以看到一些有趣的信息。让我们快速浏览一下一些更具信息量的日志条目。

```
[Sun Jun 21 20:53:42.577787 2015] [mpm_prefork:notice] [pid 2218] AH00170: caught SIGWINCH, shutting down gracefully

```

前一行显示 Apache 进程在`Sunday, Jun 21`的`20:53`被关闭。我们可以看到错误消息清楚地说明了`优雅地关闭`。然而，接下来的几行似乎表明 Apache 服务只在`2`秒后重新启动：

```
[Sun Jun 21 20:53:44.677885 2015] [core:notice] [pid 2249] SELinux policy enabled; httpd running as context system_u:system_r:httpd_t:s0
[Sun Jun 21 20:53:44.678919 2015] [suexec:notice] [pid 2249] AH01232: suEXEC mechanism enabled (wrapper: /usr/sbin/suexec)
[Sun Jun 21 20:53:44.703088 2015] [auth_digest:notice] [pid 2249] AH01757: generating secret for digest authentication ...
[Sun Jun 21 20:53:44.704046 2015] [lbmethod_heartbeat:notice] [pid 2249] AH02282: No slotmem from mod_heartmonitor
[Sun Jun 21 20:53:44.732504 2015] [mpm_prefork:notice] [pid 2249] AH00163: Apache/2.4.6  PHP/5.4.16 configured -- resuming normal operations

```

关机日志条目显示了一个进程 ID 为`2218`，而前面的五行显示了一个进程 ID 为`2249`。第五行还声明了`恢复正常运行`。这四条消息似乎表明 Apache 进程只是重新启动了。很可能，这是 Apache 的优雅重启。

Apache 的优雅重启是在修改其配置时执行的一个相当常见的任务。这是一种在不完全关闭和影响 Web 服务的情况下重新启动 Apache 进程的方法。

```
[Sun Jun 21 20:53:44.732568 2015] [core:notice] [pid 2249] AH00094: Command line: '/usr/sbin/httpd -D FOREGROUND'

```

然而，这 10 行告诉我们最有趣的事情是，Apache 打印的最后一个日志只是一个通知。当 Apache 被优雅地停止时，它会在`error_log`文件中记录一条消息，以显示它正在被停止。

由于 Apache 进程不再运行，并且没有日志条目显示它是正常关闭或非正常关闭，我们得出结论，无论 Apache 为什么不运行，它都没有正常关闭。

如果一个人使用`apachectl`或`systemctl`命令关闭了服务，我们会期望看到类似于之前例子中讨论的消息。由于日志文件的最后一行没有显示关闭消息，我们只能假设这个进程是在异常情况下被终止或终止的。

现在，问题是*是什么导致了 Apache 进程以这种异常方式终止？*

Apache 发生了什么事情的线索可能在于 systemd 设施，因为 Red Hat Enterprise Linux 7 服务，比如 Apache，已经被迁移到了 systemd。在启动时，`systemd`设施会启动任何已经配置好的服务。

当`systemd`启动的进程被终止时，这个活动会被`systemd`捕获。根据进程终止后发生的情况，我们可以使用`systemctl`命令来查看`systemd`是否捕获了这个事件：

```
# systemctl status httpd
httpd.service - The Apache HTTP Server
 Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled)
 Active: failed (Result: timeout) since Fri 2015-06-26 21:21:38 UTC; 22min ago
 Process: 2521 ExecStop=/bin/kill -WINCH ${MAINPID} (code=exited, status=0/SUCCESS)
 Process: 2249 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=killed, signal=KILL)
 Main PID: 2249 (code=killed, signal=KILL)
 Status: "Total requests: 1649; Current requests/sec: -1.29; Current traffic:   0 B/sec"

Jun 21 20:53:44 blog.example.com systemd[1]: Started The Apache HTTP Server.
Jun 26 21:12:55 blog.example.com systemd[1]: httpd.service: main process exited, code=killed, status=9/KILL
Jun 26 21:21:20 blog.example.com systemd[1]: httpd.service stopping timed out. Killing.
Jun 26 21:21:38 blog.example.com systemd[1]: Unit httpd.service entered failed state.

```

`systemctl status`命令的输出显示了相当多的信息。由于我们在之前的章节中已经涵盖了这个问题，我将跳过这个输出的大部分，只看一些能告诉我们 Apache 服务发生了什么的部分。

看起来有趣的前两行是：

```
 Process: 2249 ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND (code=killed, signal=KILL)
 Main PID: 2249 (code=killed, signal=KILL)

```

在这两行中，我们可以看到进程 ID 为`2249`，这也是我们在`error_log`文件中看到的。这是在`6 月 21 日星期日`启动的 Apache 实例的进程 ID。我们还可以从这些行中看到，进程`2249`被终止了。这似乎表明有人或某物终止了我们的 Apache 服务：

```
Jun 21 20:53:44 blog.example.com systemd[1]: Started The Apache HTTP Server.
Jun 26 21:12:55 blog.example.com systemd[1]: httpd.service: main process exited, code=killed, status=9/KILL
Jun 26 21:21:20 blog.example.com systemd[1]: httpd.service stopping timed out. Killing.
Jun 26 21:21:38 blog.example.com systemd[1]: Unit httpd.service entered failed state.

```

如果我们看一下`systemctl`状态输出的最后几行，我们可以看到`systemd`设施捕获的事件。我们可以看到的第一个事件是 Apache 服务在`6 月 21 日 20:53`启动。这并不奇怪，因为它与我们在`error_log`中看到的信息相符。

然而，最后三行显示 Apache 进程随后在`6 月 26 日 21:21`被终止。不幸的是，这些事件并没有准确显示 Apache 进程被终止的原因或是谁终止了它。它告诉我们的是 Apache 被终止的确切时间。这也表明`systemd`设施不太可能停止了 Apache 服务。

## 那个时候还发生了什么？

由于我们无法从 Apache 日志或`systemctl status`中确定原因，我们需要继续挖掘以了解是什么导致了这个服务的停止。

```
# date
Sun Jun 28 18:32:33 UTC 2015

```

由于 26 号已经过去了几天，我们有一些有限的地方可以寻找额外的信息。我们可以查看`/var/log/messages`日志文件。正如我们在前面的章节中发现的，`messages`日志包含了系统中许多不同设施的各种信息。如果有一个地方可以告诉我们那个时候系统发生了什么，那就是那里。

### 搜索 messages 日志

`messages`日志非常庞大，在其中有许多日志条目：

```
# wc -l /var/log/messages
21683 /var/log/messages

```

因此，我们需要过滤掉与我们的问题无关或不在我们问题发生时的日志消息。我们可以做的第一件事是搜索日志中 Apache 停止的那一天的消息：`June 26`：

```
# tail -1 /var/log/messages
Jun 28 20:44:01 localhost systemd: Started Session 348 of user vagrant.

```

从前面提到的`tail`命令中，我们可以看到`/var/log/messages`文件中的消息格式是日期、主机名、进程，然后是消息。日期字段是一个三个字母的月份，后面跟着日期数字和 24 小时时间戳。

由于我们的问题发生在 6 月 26 日，我们可以搜索这个日志文件中字符串"`Jun 26`"的任何实例。这应该提供所有在 26 日写入的消息：

```
# grep -c "Jun 26" /var/log/messages
17864

```

显然这仍然是相当多的日志消息，太多了，无法全部阅读。鉴于这个数量，我们需要进一步过滤消息，也许可以按进程来过滤：

```
# grep "Jun 26" /var/log/messages | cut -d\  -f1,2,5 | sort -n | uniq -c | sort -nk 1 | tail
 39 Jun 26 journal:
 56 Jun 26 NetworkManager:
 76 Jun 26 NetworkManager[582]:
 76 Jun 26 NetworkManager[588]:
 78 Jun 26 NetworkManager[580]:
 79 Jun 26 systemd-logind:
 110 Jun 26 systemd[1]:
 152 Jun 26 NetworkManager[574]:
 1684 Jun 26 systemd:
 15077 Jun 26 kernel:

```

上面的代码通常被称为**bash**一行代码。这通常是一系列命令，它们将它们的输出重定向到另一个命令，以提供一个单独的命令无法执行或生成的功能或输出。在这种情况下，我们有一个一行代码，它显示了 6 月 26 日记录最多的进程。

### 分解这个有用的一行代码

上面提到的一行代码一开始可能有点复杂，但一旦我们分解这个一行代码，它就变得容易理解了。这是一个有用的一行代码，因为它使得在日志文件中识别趋势变得更容易。

让我们分解这个一行代码，以更好地理解它的作用：

```
# grep "Jun 26" /var/log/messages | cut -d\  -f1,2,5 | sort | uniq -c | sort -nk 1 | tail

```

我们已经知道第一个命令的作用；它只是在`/var/log/messages`文件中搜索字符串"`Jun 26`"的任何实例。其他命令是我们以前没有涉及过的命令，但它们可能是有用的命令。

#### cut 命令

这个一行代码中的`cut`命令用于读取`grep`命令的输出，并只打印每行的特定部分。要理解它是如何工作的，我们应该首先运行在`cut`命令结束的一行代码：

```
# grep "Jun 26" /var/log/messages | cut -d\  -f1,2,5
Jun 26 systemd:
Jun 26 systemd:
Jun 26 systemd:
Jun 26 systemd:
Jun 26 systemd:
Jun 26 systemd:
Jun 26 systemd:
Jun 26 systemd:
Jun 26 systemd:
Jun 26 systemd:

```

前面的`cut`命令通过指定分隔符并按该分隔符切割输出来工作。

分隔符是用来将行分解为多个字段的字符；我们可以用`-d`标志来指定它。在上面的例子中，`-d`标志后面跟着"`\`"；反斜杠是一个转义字符，后面跟着一个空格。这告诉`cut`命令使用一个空格字符作为分隔符。

`-f`标志用于指定应该显示的`fields`。这些字段是分隔符之间的文本字符串。

例如，让我们看看下面的命令：

```
$ echo "Apples:Bananas:Carrots:Dried Cherries" | cut -d: -f1,2,4
Apples:Bananas:Dried Cherries

```

在这里，我们指定"`:`"字符是`cut`的分隔符。我们还指定它应该打印第一、第二和第四个字段。这导致打印了 Apples（第一个字段）、Bananas（第二个字段）和 Dried Cherries（第四个字段）。第三个字段 Carrots 被省略了。这是因为我们没有明确告诉`cut`命令打印第三个字段。

现在我们知道了`cut`是如何工作的，让我们看看它是如何处理`messages`日志条目的。

这是一个日志消息的样本：

```
Jun 28 21:50:01 localhost systemd: Created slice user-0.slice.

```

当我们执行这个一行代码中的`cut`命令时，我们明确告诉它只打印第一、第二和第五个字段：

```
# grep "Jun 26" /var/log/messages | cut -d\  -f1,2,5
Jun 26 systemd:

```

通过在我们的`cut`命令中指定一个空格字符作为分隔符，我们可以看到这会导致`cut`只打印每个日志条目的月份、日期和程序。单独看可能并不那么有用，但随着我们继续查看这个一行代码，cut 提供的功能将变得至关重要。

#### sort 命令

接下来的`sort`命令在这个一行代码中实际上被使用了两次：

```
# grep "Jun 26" /var/log/messages | cut -d\  -f1,2,5 | sort | head
Jun 26 audispd:
Jun 26 audispd:
Jun 26 audispd:
Jun 26 audispd:
Jun 26 audispd:
Jun 26 auditd[539]:
Jun 26 auditd[539]:
Jun 26 auditd[542]:
Jun 26 auditd[542]:
Jun 26 auditd[548]:

```

这个命令实际上很简单，它的作用是对`cut`命令的输出进行排序。

为了更好地解释这一点，让我们看下面的例子：

```
# cat /var/tmp/fruits.txt 
Apples
Dried Cherries
Carrots
Bananas

```

上面的文件再次包含几种水果，这一次它们不是按字母顺序排列的。然而，如果我们使用`sort`命令来读取这个文件，这些水果的顺序将会改变：

```
# sort /var/tmp/fruits.txt 
Apples
Bananas
Carrots
Dried Cherries

```

正如我们所看到的，现在的顺序是按字母顺序排列的，尽管水果在文件中的顺序是不同的。`sort`的好处在于它可以用几种不同的方式对文本进行排序。实际上，在我们的一行命令中`sort`的第二个实例中，我们使用`-n`标志对文本进行了数字排序：

```
# cat /var/tmp/numbers.txt
10
23
2312
23292
1212
129191
# sort -n /var/tmp/numbers.txt 
10
23
1212
2312
23292
129191

```

### uniq 命令

我们的一行命令包含`sort`命令的原因很简单，就是为了对发送到`uniq -c`的输入进行排序：

```
# grep "Jun 26" /var/log/messages | cut -d\  -f1,2,5 | sort | uniq -c | head
 5 Jun 26 audispd:
 2 Jun 26 auditd[539]:
 2 Jun 26 auditd[542]:
 3 Jun 26 auditd[548]:
 2 Jun 26 auditd[550]:
 2 Jun 26 auditd[553]:
 15 Jun 26 augenrules:
 38 Jun 26 avahi-daemon[573]:
 19 Jun 26 avahi-daemon[579]:
 19 Jun 26 avahi-daemon[581]:

```

`uniq`命令可以用来识别匹配的行，并将这些行显示为单个唯一行。为了更好地理解这一点，让我们看看下面的例子：

```
$ cat /var/tmp/duplicates.txt 
Apple
Apple
Apple
Apple
Banana
Banana
Banana
Carrot
Carrot

```

我们的示例文件"`duplicates.txt`"包含多个重复的行。当我们用`uniq`读取这个文件时，我们只会看到每一行的唯一内容：

```
$ uniq /var/tmp/duplicates.txt 
Apple
Banana
Carrot

```

这可能有些有用；但是，我发现使用`-c`标志，输出可能会更有用：

```
$ uniq -c /var/tmp/duplicates.txt 
 4 Apple
 3 Banana
 2 Carrot

```

使用`-c`标志，`uniq`命令将计算它找到每行的次数。在这里，我们可以看到有四行包含单词苹果。因此，`uniq`命令在单词苹果之前打印了数字 4，以显示这行有四个实例：

```
$ cat /var/tmp/duplicates.txt 
Apple
Apple
Orange
Apple
Apple
Banana
Banana
Banana
Carrot
Carrot
$ uniq -c /var/tmp/duplicates.txt 
 2 Apple
 1 Orange
 2 Apple
 3 Banana
 2 Carrot

```

`uniq`命令的一个注意事项是，为了获得准确的计数，每个实例都需要紧挨在一起。当我们在苹果行的组之间添加单词橙子时，可以看到会发生什么。

### 把所有东西都联系在一起

如果我们再次看看我们的命令，现在我们可以更好地理解它在做什么：

```
# grep "Jun 26" /var/log/messages | cut -d\  -f1,2,5 | sort | uniq -c | sort -n | tail
 39 Jun 26 journal:
 56 Jun 26 NetworkManager:
 76 Jun 26 NetworkManager[582]:
 76 Jun 26 NetworkManager[588]:
 78 Jun 26 NetworkManager[580]:
 79 Jun 26 systemd-logind:
 110 Jun 26 systemd[1]:
 152 Jun 26 NetworkManager[574]:
 1684 Jun 26 systemd:
 15077 Jun 26 kernel:

```

上面的命令将过滤并打印`/var/log/messages`中与字符串"`Jun 26`"匹配的所有日志消息。然后输出将被发送到`cut`命令，该命令打印每行的月份、日期和进程。然后将此输出发送到`sort`命令，以将输出排序为相互匹配的组。排序后的输出然后被发送到`uniq -c`，它计算每行出现的次数并打印一个带有计数的唯一行。

然后，我们添加另一个`sort`来按`uniq`添加的数字对输出进行排序，并添加`tail`来将输出缩短到最后 10 行。

那么，这个花哨的一行命令到底告诉我们什么呢？嗯，它告诉我们`kernel`设施和`systemd`进程正在记录相当多的内容。实际上，与其他列出的项目相比，我们可以看到这两个项目的日志消息比其他项目更多。

然而，`systemd`和`kernel`在`/var/log/messages`中有更多的日志消息可能并不奇怪。如果有另一个写入许多日志的进程，我们将能够在一行输出中看到这一点。然而，由于我们的第一次运行没有产生有用的结果，我们可以修改一行命令来缩小输出范围：

```
Jun 26 19:51:10 localhost auditd[550]: Started dispatcher: /sbin/audispd pid: 562

```

如果我们看一下`messages`日志条目的格式，我们会发现在进程之后，可以找到日志消息。为了进一步缩小我们的搜索范围，我们可以在输出中添加一点消息。

我们可以通过将`cut`命令的字段列表更改为"`1,2,5-8`"来实现这一点。通过在`5`后面添加"`-8`"，我们发现`cut`命令显示从 5 到 8 的所有字段。这样做的效果是在我们的一行命令中包含每条日志消息的前三个单词：

```
# grep "Jun 26" /var/log/messages | cut -d\  -f1,2,5-8 | sort | uniq -c | sort -n | tail -30
 64 Jun 26 kernel: 131055 pages RAM
 64 Jun 26 kernel: 5572 pages reserved
 64 Jun 26 kernel: lowmem_reserve[]: 0 462
 77 Jun 26 kernel: [  579]
 79 Jun 26 kernel: Out of memory:
 80 Jun 26 kernel: [<ffffffff810b68f8>] ? ktime_get_ts+0x48/0xe0
 80 Jun 26 kernel: [<ffffffff81102e03>] ? proc_do_uts_string+0xe3/0x130
 80 Jun 26 kernel: [<ffffffff8114520e>] oom_kill_process+0x24e/0x3b0
 80 Jun 26 kernel: [<ffffffff81145a36>] out_of_memory+0x4b6/0x4f0
 80 Jun 26 kernel: [<ffffffff8114b579>] __alloc_pages_nodemask+0xa09/0xb10
 80 Jun 26 kernel: [<ffffffff815dd02d>] dump_header+0x8e/0x214
 80 Jun 26 kernel: [ pid ]
 81 Jun 26 kernel: [<ffffffff8118bc3a>] alloc_pages_vma+0x9a/0x140
 93 Jun 26 kernel: Call Trace:
 93 Jun 26 kernel: [<ffffffff815e19ba>] dump_stack+0x19/0x1b
 93 Jun 26 kernel: [<ffffffff815e97c8>] page_fault+0x28/0x30
 93 Jun 26 kernel: [<ffffffff815ed186>] __do_page_fault+0x156/0x540
 93 Jun 26 kernel: [<ffffffff815ed58a>] do_page_fault+0x1a/0x70
 93 Jun 26 kernel: Free swap 
 93 Jun 26 kernel: Hardware name: innotek
 93 Jun 26 kernel: lowmem_reserve[]: 0 0
 93 Jun 26 kernel: Mem-Info:
 93 Jun 26 kernel: Node 0 DMA:
 93 Jun 26 kernel: Node 0 DMA32:
 93 Jun 26 kernel: Node 0 hugepages_total=0
 93 Jun 26 kernel: Swap cache stats:
 93 Jun 26 kernel: Total swap =
 186 Jun 26 kernel: Node 0 DMA
 186 Jun 26 kernel: Node 0 DMA32
 489 Jun 26 kernel: CPU 

```

如果我们还增加`tail`命令以显示最后 30 行，我们可以看到一些有趣的趋势。非常有趣的第一行是输出中的第四行：

```
 79 Jun 26 kernel: Out of memory:

```

似乎`kernel`打印了以术语"`Out of memory`"开头的`79`条日志消息。虽然这似乎有点显而易见，但似乎这台服务器可能在某个时候耗尽了内存。

接下来的两行看起来也支持这个理论：

```
 80 Jun 26 kernel: [<ffffffff8114520e>] oom_kill_process+0x24e/0x3b0
 80 Jun 26 kernel: [<ffffffff81145a36>] out_of_memory+0x4b6/0x4f0

```

第一行似乎表明内核终止了一个进程；第二行再次表明出现了*内存耗尽*的情况。这个系统可能已经耗尽了内存，并在这样做时终止了 Apache 进程。这似乎非常可能。

## 当 Linux 系统内存耗尽时会发生什么？

在 Linux 上，内存的管理方式与其他操作系统有些不同。当系统内存不足时，内核有一个旨在回收已使用内存的进程；这个进程称为**内存耗尽终结者**（**oom-kill**）。

`oom-kill`进程旨在终止使用大量内存的进程，以释放这些内存供关键系统进程使用。我们将稍后讨论`oom-kill`，但首先，我们应该了解 Linux 如何定义内存耗尽。

### 最小空闲内存

在 Linux 上，当空闲内存量低于定义的最小值时，将启动 oom-kill 进程。这个最小值当然是一个名为`vm.min_free_kbytes`的内核可调参数。该参数允许您设置系统始终可用的内存量（以千字节为单位）。

当可用内存低于此参数的值时，系统开始采取行动。在深入讨论之前，让我们首先看看我们系统上设置的这个值，并重新了解 Linux 中的内存管理方式。

我们可以使用与上一章相同的`sysctl`命令查看当前的`vm.min_free_kbytes`值：

```
# sysctl vm.min_free_kbytes
vm.min_free_kbytes = 11424

```

当前值为`11424`千字节，约为 11 兆字节。这意味着我们系统的空闲内存必须始终大于 11 兆字节，否则系统将启动 oom-kill 进程。这似乎很简单，但正如我们从第四章中所知道的那样，Linux 管理内存的方式并不一定那么简单：

```
# free
 total       used       free     shared    buffers cached
Mem:        243788     230012      13776         60          0 2272
-/+ buffers/cache:     227740      16048
Swap:      1081340     231908     849432

```

如果我们在这个系统上运行`free`命令，我们可以看到当前的内存使用情况以及可用内存量。在深入讨论之前，我们将分解这个输出，以便重新理解 Linux 如何使用内存。

```
 total       used       free     shared    buffers  cached
Mem:        243788     230012      13776         60          0 2272

```

在第一行中，我们可以看到系统总共有 243MB 的物理内存。我们可以在第二列中看到目前使用了 230MB，第三列显示有 13MB 未使用。系统测量的正是这个未使用的值，以确定当前是否有足够的最小所需内存空闲。

这很重要，因为如果我们记得第四章中所说的，我们使用第二个“内存空闲”值来确定有多少内存可用。

```
 total       used       free     shared    buffers cached
Mem:        243788     230012      13776         60          0 2272
-/+ buffers/cache:     227740      16048

```

在`free`的第二行，我们可以看到系统在考虑缓存使用的内存量时的已使用和空闲内存量。正如我们之前学到的，Linux 系统非常积极地缓存文件和文件系统属性。所有这些缓存都存储在内存中，我们可以看到，在运行这个`free`命令的瞬间，我们的缓存使用了 2,272 KB 的内存。

当空闲内存（不包括缓存）接近`min_free_kbytes`值时，系统将开始回收一些用于缓存的内存。这旨在允许系统尽可能地缓存，但在内存不足的情况下，为了防止 oom-kill 进程的启动，这个缓存变得可丢弃：

```
Swap:      1081340     231908     849432

```

`free`命令的第三行将我们带到 Linux 内存管理的另一个重要步骤：交换。正如我们从前一行中看到的，当执行这个`free`命令时，系统将大约 231MB 的数据从物理内存交换到交换设备。

这是我们期望在运行内存不足的系统上看到的情况。当`free`内存开始变得稀缺时，系统将开始获取物理内存中的内存对象并将它们推送到交换内存中。

系统开始执行这些交换活动的侵略性取决于内核参数`vm.swappiness`中定义的值：

```
$ sysctl vm.swappiness
vm.swappiness = 30

```

在我们的系统上，`swappiness`值目前设置为`30`。这个可调参数接受 0 到 100 之间的值，其中 100 允许最激进的交换策略。

当`swappiness`值较低时，系统会更倾向于将内存对象保留在物理内存中尽可能长的时间，然后再将它们移动到交换设备上。

#### 快速回顾

在进入 oom-kill 之前，让我们回顾一下当 Linux 系统上的内存开始变得紧张时会发生什么。系统首先会尝试释放用于磁盘缓存的内存对象，并将已使用的内存移动到交换设备上。如果系统无法通过前面提到的两个过程释放足够的内存，内核就会启动 oom-kill 进程。

### oom-kill 的工作原理

如前所述，oom-kill 进程是在空闲内存不足时启动的一个进程。这个进程旨在识别使用大量内存并且对系统操作不重要的进程。

那么，oom-kill 是如何确定这一点的呢？嗯，实际上是由内核确定的，并且不断更新。

我们在前面的章节中讨论了系统上每个运行的进程都有一个在`/proc`文件系统中的文件夹。内核维护着这个文件夹，里面有很多有趣的文件。

```
# ls -la /proc/6689/oom_*
-rw-r--r--. 1 root root 0 Jun 29 15:23 /proc/6689/oom_adj
-r--r--r--. 1 root root 0 Jun 29 15:23 /proc/6689/oom_score
-rw-r--r--. 1 root root 0 Jun 29 15:23 /proc/6689/oom_score_adj

```

前面提到的三个文件与 oom-kill 进程及每个进程被杀死的可能性有关。我们要看的第一个文件是`oom_score`文件：

```
# cat /proc/6689/oom_score
40

```

如果我们`cat`这个文件，我们会发现它只包含一个数字。然而，这个数字对于 oom-kill 进程非常重要，因为这个数字就是进程 6689 的 OOM 分数。

OOM 分数是内核分配给一个进程的一个值，用来确定相应进程对 oom-kill 的优先级高低。分数越高，进程被杀死的可能性就越大。当内核为这个进程分配一个值时，它基于进程使用的内存和交换空间的数量以及对系统的重要性。

你可能会问自己，“我想知道是否有办法调整我的进程的 oom 分数。” 这个问题的答案是肯定的，有！这就是另外两个文件`oom_adj`和`oom_score_adj`发挥作用的地方。这两个文件允许您调整进程的 oom 分数，从而控制进程被杀死的可能性。

目前，`oom_adj`文件将被淘汰，取而代之的是`oom_score_adj`。因此，我们将只关注`oom_score_adj`文件。

#### 调整 oom 分数

`oom_score_adj`文件支持从-1000 到 1000 的值，其中较高的值将增加 oom-kill 选择该进程的可能性。让我们看看当我们为我们的进程添加 800 的调整时，我们的 oom 分数会发生什么变化：

```
# echo "800" > /proc/6689/oom_score_adj 
# cat /proc/6689/oom_score
840

```

仅仅通过改变内容为 800，内核就检测到了这个调整并为这个进程的 oom 分数增加了 800。如果这个系统在不久的将来内存耗尽，这个进程绝对会被 oom-kill 杀死。

如果我们将这个值改为-1000，这实际上会排除该进程被 oom-kill 杀死的可能性。

## 确定我们的进程是否被 oom-kill 杀死

现在我们知道了系统内存不足时会发生什么，让我们更仔细地看看我们的系统到底发生了什么。为了做到这一点，我们将使用`less`来读取`/var/log/messages`文件，并寻找`kernel: Out of memory`消息的第一个实例：

```
Jun 26 00:53:39 blog kernel: Out of memory: Kill process 5664 (processor) score 265 or sacrifice child

```

有趣的是，“内存不足”日志消息的第一个实例是在我们的 Apache 进程被杀死之前的 20 小时。更重要的是，被杀死的进程是一个非常熟悉的进程，即上一章的“处理器”cronjob。

这一条日志记录实际上可以告诉我们关于该进程以及为什么 oom-kill 选择了该进程的很多信息。在第一行，我们可以看到内核给了处理器进程一个`265`的分数。虽然不是最高分，但我们已经看到 265 分很可能比此时运行的大多数进程的分数都要高。

这似乎表明处理器作业在这个时候使用了相当多的内存。让我们继续查看这个文件，看看在这个系统上可能发生了什么其他事情：

```
Jun 26 00:54:31 blog kernel: Out of memory: Kill process 5677 (processor) score 273 or sacrifice child

```

在日志文件中再往下看一点，我们可以看到处理器进程再次被杀死。似乎每次这个作业运行时，系统都会耗尽内存。

为了节约时间，让我们跳到第 21 个小时，更仔细地看看我们的 Apache 进程被杀死的时间：

```
Jun 26 21:12:54 localhost kernel: Out of memory: Kill process 2249 (httpd) score 7 or sacrifice child
Jun 26 21:12:54 localhost kernel: Killed process 2249 (httpd) total-vm:462648kB, anon-rss:436kB, file-rss:8kB
Jun 26 21:12:54 localhost kernel: httpd invoked oom-killer: gfp_mask=0x200da, order=0, oom_score_adj=0

```

看起来`messages`日志一直都有我们的答案。从前面几行可以看到进程`2249`，这恰好是我们的 Apache 服务器进程 ID：

```
Jun 26 21:12:55 blog.example.com systemd[1]: httpd.service: main process exited, code=killed, status=9/KILL

```

在这里，我们看到`systemd`检测到该进程在`21:12:55`被杀死。此外，我们可以从消息日志中看到 oom-kill 在`21:12:54`针对该进程进行了操作。在这一点上，毫无疑问，该进程是被 oom-kill 杀死的。

## 系统为什么耗尽了内存？

在这一点上，我们能够确定 Apache 服务在内存耗尽时被系统杀死。不幸的是，oom-kill 并不是问题的根本原因，而是一个症状。虽然它是 Apache 服务停止的原因，但如果我们只是重新启动进程而不做其他操作，问题可能会再次发生。

在这一点上，我们需要确定是什么导致系统首先耗尽了内存。为了做到这一点，让我们来看看消息日志文件中“内存不足”消息的整个列表：

```
# grep "Out of memory" /var/log/messages* | cut -d\  -f1,2,10,12 | uniq -c
 38 /var/log/messages:Jun 28 process (processor)
 1 /var/log/messages:Jun 28 process (application)
 10 /var/log/messages:Jun 28 process (processor)
 1 /var/log/messages-20150615:Jun 10 process (python)
 1 /var/log/messages-20150628:Jun 22 process (processor)
 47 /var/log/messages-20150628:Jun 26 process (processor)
 32 /var/log/messages-20150628:Jun 26 process (httpd)

```

再次使用`cut`和`uniq -c`命令，我们可以在消息日志中看到一个有趣的趋势。我们可以看到内核已经多次调用了 oom-kill。我们可以看到即使今天系统也启动了 oom-kill 进程。

现在我们应该做的第一件事是弄清楚这个系统有多少内存。

```
# free -m
 total       used       free     shared    buffers cached
Mem:           238        206         32          0          0 2
-/+ buffers/cache:        203         34
Swap:         1055        428        627

```

使用`free`命令，我们可以看到系统有`238` MB 的物理内存和`1055` MB 的交换空间。然而，我们也可以看到只有`34` MB 的内存是空闲的，系统已经交换了`428` MB 的物理内存。

很明显，对于当前的工作负载，该系统分配的内存根本不够。

如果我们回顾一下 oom-kill 所针对的进程，我们可以看到一个有趣的趋势：

```
# grep "Out of memory" /var/log/messages* | cut -d\  -f10,12 | sort | uniq -c
 1 process (application)
 32 process (httpd)
 118 process (processor)
 1 process (python)

```

在这里，很明显，被最频繁杀死的两个进程是`httpd`和`processor`。我们之前了解到，oom-kill 根据它们使用的内存量来确定要杀死的进程。这意味着这两个进程在系统上使用了最多的内存，但它们到底使用了多少内存呢？

```
# ps -eo rss,size,cmd | grep processor
 0   340 /bin/sh -c /opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml > /dev/null
130924 240520 /opt/myapp/bin/processor --debug --config /opt/myapp/conf/config.yml
 964   336 grep --color=auto processor

```

使用`ps`命令来专门显示**rss**和**size**字段，我们在第四章中学到了，*故障排除性能问题*，我们可以看到`processor`作业使用了`130` MB 的常驻内存和`240` MB 的虚拟内存。

如果系统只有`238` MB 的物理内存，而进程使用了`240` MB 的虚拟内存，最终，这个系统的物理内存会不足。

# 长期和短期解决问题

像本章讨论的这种问题可能有点棘手，因为它们通常有两种解决路径。有一个长期解决方案和一个短期解决方案；两者都是必要的，但一个只是临时的。

## 长期解决方案

对于这个问题的长期解决方案，我们确实有两个选择。我们可以增加服务器的物理内存，为 Apache 和 Processor 提供足够的内存来完成它们的任务。或者，我们可以将 processor 移动到另一台服务器上。

由于我们知道这台服务器经常杀死 Apache 服务和`processor`任务，很可能是系统上的内存对于执行这两个角色来说太低了。通过将`processor`任务（以及很可能是其一部分的自定义应用程序）移动到另一个系统，我们将工作负载移动到一个专用服务器。

基于处理器的内存使用情况，增加新服务器的内存也可能是值得的。似乎`processor`任务使用了足够的内存，在目前这样的低内存服务器上可能会导致内存不足的情况。

确定哪个长期解决方案最好取决于环境和导致系统内存不足的应用程序。在某些情况下，增加服务器的内存可能是更好的选择。

在虚拟和云环境中，这个任务非常容易，但这并不总是最好的答案。确定哪个答案更好取决于你所使用的环境。

## 短期解决方案

假设两个长期解决方案都需要几天时间来实施。就目前而言，我们的系统上 Apache 服务仍然处于停机状态。这意味着我们的公司博客也仍然处于停机状态；为了暂时解决问题，我们需要重新启动 Apache。

然而，我们不应该只是用`systemctl`命令简单地重新启动 Apache。在启动任何内容之前，我们实际上应该首先重新启动服务器。

当大多数 Linux 管理员听到“让我们重启”这句话时，他们会感到沮丧。这是因为作为 Linux 系统管理员，我们很少需要重启系统。我们被告知在更新内核之外重启 Linux 服务器是一件不好的事情。

在大多数情况下，我们认为重新启动服务器不是正确的解决方案。然而，我认为系统内存不足是一个特殊情况。

我认为，在启动 oom-kill 时，应该在完全恢复到正常状态之前重新启动相关系统。

我这样说的原因是 oom-kill 进程可以杀死任何进程，包括关键的系统进程。虽然 oom-kill 进程确实会通过 syslog 记录被杀死的进程，但 syslog 守护程序只是系统上的另一个可以被 oom-kill 杀死的进程。

即使 oom-kill 没有在 oom-kill 杀死许多不同进程的情况下杀死 syslog 进程，要确保每个进程都正常运行可能会有些棘手。特别是当处理问题的人经验较少时。

虽然你可以花时间确定正在运行的进程，并确保重新启动每个进程，但简单地重新启动服务器可能更快，而且可以说更安全。因为你知道在启动时，每个定义为启动的进程都将被启动。

虽然并非每个系统管理员都会同意这种观点，但我认为这是确保系统处于稳定状态的最佳方法。但重要的是要记住，这只是一个短期解决方案，重新启动后，除非有变化，系统可能会再次出现内存不足的情况。

对于我们的情况，最好是在服务器的内存增加或作业可以移至专用系统之前禁用`processor`作业。然而，在某些情况下，这可能是不可接受的。像长期解决方案一样，防止再次发生这种情况是情境性的，并取决于你所管理的环境。

由于我们假设短期解决方案是我们示例的正确解决方案，我们将继续重新启动系统：

```
# reboot
Connection to 127.0.0.1 closed by remote host.

```

系统恢复在线后，我们可以使用`systemctl`命令验证 Apache 是否正在运行。

```
# systemctl status httpd
httpd.service - The Apache HTTP Server
 Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled)
 Active: active (running) since Wed 2015-07-01 15:37:22 UTC; 1min 29s ago
 Main PID: 1012 (httpd)
 Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
 CGroup: /system.slice/httpd.service
 ├─1012 /usr/sbin/httpd -DFOREGROUND
 ├─1439 /usr/sbin/httpd -DFOREGROUND
 ├─1443 /usr/sbin/httpd -DFOREGROUND
 ├─1444 /usr/sbin/httpd -DFOREGROUND
 ├─1445 /usr/sbin/httpd -DFOREGROUND
 └─1449 /usr/sbin/httpd -DFOREGROUND

Jul 01 15:37:22 blog.example.com systemd[1]: Started The Apache HTTP Server.

```

如果我们在这个系统上再次运行`free`命令，我们可以看到内存利用率要低得多，至少直到现在为止。

```
# free -m
 total       used       free     shared    buffers cached
Mem:           238        202         35          4          0 86
-/+ buffers/cache:        115        122
Swap:         1055          0       1055

```

# 总结

在本章中，我们运用了我们的故障排除技能，确定了影响公司博客的问题以及这个问题的根本原因。我们能够运用在之前章节学到的技能和技术，确定 Apache 服务已经停止。我们还确定了这个问题的根本原因是系统内存耗尽。

通过调查日志文件，我们发现系统上占用最多内存的两个进程是 Apache 和一个名为`processor`的自定义应用程序。此外，通过识别这些进程，我们能够提出长期建议，以防止此问题再次发生。

除此之外，我们还学到了当 Linux 系统内存耗尽时会发生什么。

在下一章中，我们将把你到目前为止学到的一切付诸实践，通过对一个无响应系统进行根本原因分析。
