# 第十四章：调度和日志记录

在本章中，我们将教您调度和记录脚本结果的基础知识。我们将首先解释如何使用`at`和`cron`来调度命令和脚本。在本章的第二部分，我们将描述如何记录脚本的结果。我们可以使用 Linux 的本地邮件功能和重定向来实现此目的。

本章将介绍以下命令：`at`、`wall`、`atq`、`atrm`、`sendmail`、`crontab`和`alias`。

本章将涵盖以下主题：

+   使用`at`和`cron`进行调度

+   记录脚本结果

# 技术要求

本章的所有脚本都可以在 GitHub 上找到：[`github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter14`](https://github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter14)。其余的示例和练习应该在您的 Ubuntu 虚拟机上执行。

# 使用 at 和 cron 进行调度

到目前为止，我们已经学习了 shell 脚本世界中的许多内容：变量、条件、循环、重定向，甚至函数。在本章中，我们将解释另一个与 shell 脚本密切相关的重要概念：调度。

简而言之，调度是确保您的命令或脚本在特定时间运行，而无需您每次都亲自启动它们。经典示例可以在清理日志中找到；通常，旧日志不再有用并且占用太多空间。例如，您可以使用清理脚本解决此问题，该脚本会删除 45 天前的日志。但是，这样的脚本可能应该每天运行一次。在工作日，这可能不是最大的问题，但在周末登录并不好玩。实际上，我们甚至不应该考虑这一点，因为调度允许我们定义脚本应该在*何时*或*多久*运行！

在 Linux 调度中，最常用的工具是`at`和`cron`。我们将首先描述使用`at`进行调度的原则，然后再继续使用更强大（因此更广泛使用）的`cron`。

# at

`at`命令主要用于临时调度。`at`的语法非常接近我们的自然语言。通过以下示例最容易解释：

```
reader@ubuntu:~/scripts/chapter_14$ date
Sat Nov 24 11:50:12 UTC 2018
reader@ubuntu:~/scripts/chapter_14$ at 11:51
warning: commands will be executed using /bin/sh
at> wall "Hello readers!"
at> <EOT>
job 6 at Sat Nov 24 11:51:00 2018
reader@ubuntu:~/scripts/chapter_14$ date
Sat Nov 24 11:50:31 UTC 2018

Broadcast message from reader@ubuntu (somewhere) (Sat Nov 24 11:51:00 2018):

Hello readers!

reader@ubuntu:~/scripts/chapter_14$ date
Sat Nov 24 11:51:02 UTC 2018
```

实质上，您在告诉系统：*在<时间戳>，执行某些操作*。当您输入`at 11:51`命令时，您将进入一个交互式提示符，允许您输入要执行的命令。之后，您可以使用*Ctrl* + *D*退出提示符；如果您使用*Ctrl* + *C*，作业将不会被保存！作为参考，在这里我们使用一个简单的命令`wall`，它允许您向当时登录到服务器的所有人广播消息。

# 时间语法

当您使用`at`时，可以绝对指定时间，就像我们在上一个示例中所做的那样，也可以相对指定。相对指定的示例可能是*5 分钟后*或*24 小时后*。这通常比检查当前时间，将所需的间隔添加到其中，并将其传递给`at`更容易。这可以使用以下语法：

```
reader@ubuntu:~/scripts/chapter_14$ at now + 1 min
warning: commands will be executed using /bin/sh
at> touch /tmp/at-file
at> <EOT>
job 10 at Sun Nov 25 10:16:00 2018
reader@ubuntu:~/scripts/chapter_14$ date
Sun Nov 25 10:15:20 UTC 2018
```

您总是需要指定相对于哪个时间要添加分钟、小时或天。幸运的是，我们可以使用 now 作为当前时间的关键字。请注意，处理分钟时，`at`将始终四舍五入到最近的整分钟。除分钟外，以下内容也是有效的（如`man at`中所述）：

+   小时

+   天

+   周

您甚至可以创建更复杂的解决方案，例如*3 天后的下午 4 点*。但是，我们认为`cron`更适合这类情况。就`at`而言，最佳用途似乎是在*接近*的时间运行一次性作业。

# at 队列

一旦您开始安排作业，您就会发现自己处于这样一种情况：您要么搞砸了时间，要么搞砸了作业内容。对于某些作业，您可以添加一个新的作业，让其他作业失败。但是，肯定有一些情况下，原始作业将对您的系统造成严重破坏。在这种情况下，删除错误的作业将是一个好主意。幸运的是，`at`的创建者预见到了这个问题（可能也经历过！）并创建了这个功能。`atq`命令（**at** **queue**的缩写）显示当前在队列中的作业。使用`atrm`（我们想不需要解释这个），您可以按编号删除作业。让我们看一个队列中有多个作业的示例，并删除其中一个：

```
reader@ubuntu:~/scripts/chapter_14$ vim wall.txt
reader@ubuntu:~/scripts/chapter_14$ cat wall.txt 
wall "Hello!"
reader@ubuntu:~/scripts/chapter_14$ at now + 5 min -f wall.txt 
warning: commands will be executed using /bin/sh
job 12 at Sun Nov 25 10:35:00 2018
reader@ubuntu:~/scripts/chapter_14$ at now + 10 min -f wall.txt 
warning: commands will be executed using /bin/sh
job 13 at Sun Nov 25 10:40:00 2018
reader@ubuntu:~/scripts/chapter_14$ at now + 4 min -f wall.txt 
warning: commands will be executed using /bin/sh
job 14 at Sun Nov 25 10:34:00 2018
reader@ubuntu:~/scripts/chapter_14$ atq
12    Sun Nov 25 10:35:00 2018 a reader
13    Sun Nov 25 10:40:00 2018 a reader
14    Sun Nov 25 10:34:00 2018 a reader
reader@ubuntu:~/scripts/chapter_14$ atrm 13
reader@ubuntu:~/scripts/chapter_14$ atq
12    Sun Nov 25 10:35:00 2018 a reader
14    Sun Nov 25 10:34:00 2018 a reader
```

正如您所看到的，我们为`at`使用了一个新的标志：`-f`。这允许我们运行在文件中定义的命令，而不必使用交互式 shell。这个文件以.txt 结尾（为了清晰起见，不需要扩展名），其中包含要执行的命令。我们使用这个文件来安排三个作业：5 分钟后，10 分钟后和 4 分钟后。在这样做之后，我们使用`atq`来查看当前队列：所有三个作业，编号为 12、13 和 14。此时，我们意识到我们只想让作业在 4 和 5 分钟后运行，而不是在 10 分钟后运行。现在我们可以使用`atrm`通过简单地将该数字添加到命令中来删除作业编号 13。然后我们再次查看队列时，只剩下作业 12 和 14。几分钟后，前两个 Hello！消息被打印到我们的屏幕上。如果我们等待完整的 10 分钟，我们将看到...什么也没有，因为我们已成功删除了我们的作业：

```
Broadcast message from reader@ubuntu (somewhere) (Sun Nov 25 10:34:00 2018):

Hello!

Broadcast message from reader@ubuntu (somewhere) (Sun Nov 25 10:35:00 2018):

Hello!

reader@ubuntu:~/scripts/chapter_14$ date
Sun Nov 25 10:42:07 UTC 2018
```

不要使用`atq`和`atrm`，`at`也有我们可以用于这些功能的标志。对于`atq`，这是`at -l`（*list*）。`atrm`甚至有两个可能的替代方案：`at -d`（*delete*）和`at -r`（*remove*）。无论您使用支持命令还是标志，底层都将执行相同的操作。使用对您来说最容易记住的方式！

# at 输出

正如您可能已经注意到的，到目前为止，我们只使用了不依赖于 stdout 的命令（有点狡猾，我们知道）。但是，一旦您考虑到这一点，这就会带来一个真正的问题。通常，当我们处理命令和脚本时，我们使用 stdout/stderr 来了解我们的操作结果。交互提示也是如此：我们使用键盘通过 stdin 提供输入。现在我们正在安排*非交互作业*，情况将会有所不同。首先，我们不能再使用诸如`read`之类的交互式结构。脚本将因为没有可用的 stdin 而简单地失败。但是，同样地，也没有可用的 stdout，因此我们甚至看不到脚本失败！还是有吗？

在`at`的 manpage 中的某个地方，您可以找到以下文本：

“用户将收到他的命令的标准错误和标准输出的邮件（如果有的话）。邮件将使用命令/usr/sbin/sendmail 发送。如果 at 是从 su(1) shell 执行的，则登录 shell 的所有者将收到邮件。”

似乎`at`的创建者也考虑到了这个问题。但是，如果您对 Linux 没有太多经验（但！），您可能会对前文中的邮件部分感到困惑。如果您在想邮票的那种，您就离谱了。但是，如果您想到*电子邮件*，您就接近了一些。

不详细介绍（这显然超出了本书的范围），Linux 有一个本地的*邮件存储箱*，允许您在本地系统内发送电子邮件。如果您将其配置为上游服务器，实际上也可以发送实际的电子邮件，但现在，请记住 Linux 系统上的内部电子邮件是可用的。有了这个邮件存储箱，电子邮件（也许不足为奇）是文件系统上的文件。这些文件可以在/var/spool/mail 找到，这实际上是/var/mail 的符号链接。如果您跟随安装 Ubuntu 18.04 机器的过程，这些目录将是空的。这很容易解释：默认情况下，`sendmail`未安装。当它未安装时，您安排一个具有 stdout 的作业时，会发生这种情况：

```
reader@ubuntu:/var/mail$ which sendmail # No output, so not installed.
reader@ubuntu:/var/mail$ at now + 1 min
warning: commands will be executed using /bin/sh
at> echo "Where will this go?" 
at> <EOT>
job 15 at Sun Nov 25 11:12:00 2018
reader@ubuntu:/var/mail$ date
Sun Nov 25 11:13:02 UTC 2018
reader@ubuntu:/var/mail$ ls -al
total 8
drwxrwsr-x  2 root mail 4096 Apr 26  2018 .
drwxr-xr-x 14 root root 4096 Jul 29 12:30 ..
```

是的，确实什么都不会发生。现在，如果我们安装`sendmail`并再次尝试，我们应该会看到不同的结果：

```
reader@ubuntu:/var/mail$ sudo apt install sendmail -y
[sudo] password for reader: 
Reading package lists... Done
<SNIPPED>
Setting up sendmail (8.15.2-10) ...
<SNIPPED>
reader@ubuntu:/var/mail$ which sendmail
/usr/sbin/sendmail
reader@ubuntu:/var/mail$ at now + 1 min
warning: commands will be executed using /bin/sh
at> echo "Where will this go?"
at> <EOT>
job 16 at Sun Nov 25 11:17:00 2018
reader@ubuntu:/var/mail$ date
Sun Nov 25 11:17:09 UTC 2018
You have new mail in /var/mail/reader
```

邮件，只给你！如果我们检查/var/mail/，我们将看到只有一个包含我们输出的文件：

```
reader@ubuntu:/var/mail$ ls -l
total 4
-rw-rw---- 1 reader mail 1341 Nov 25 11:18 reader
reader@ubuntu:/var/mail$ cat reader 
From reader@ubuntu.home.lan Sun Nov 25 11:17:00 2018
Return-Path: <reader@ubuntu.home.lan>
Received: from ubuntu.home.lan (localhost.localdomain [127.0.0.1])
  by ubuntu.home.lan (8.15.2/8.15.2/Debian-10) with ESMTP id wAPBH0Ix003531
  for <reader@ubuntu.home.lan>; Sun, 25 Nov 2018 11:17:00 GMT
Received: (from reader@localhost)
  by ubuntu.home.lan (8.15.2/8.15.2/Submit) id wAPBH0tK003528
  for reader; Sun, 25 Nov 2018 11:17:00 GMT
Date: Sun, 25 Nov 2018 11:17:00 GMT
From: Learn Linux Shell Scripting <reader@ubuntu.home.lan>
Message-Id: <201811251117.wAPBH0tK003528@ubuntu.home.lan>
Subject: Output from your job 16
To: reader@ubuntu.home.lan

Where will this go?
```

它甚至看起来像一个真正的电子邮件，有一个日期：、主题：、收件人：和发件人：（等等）。如果我们安排更多的作业，我们将看到新的邮件附加到这个单个文件中。Linux 有一些简单的基于文本的邮件客户端，允许您将这个单个文件视为多个电子邮件（`mutt`就是一个例子）；但是，我们不需要这些来实现我们的目的。

在处理系统通知时需要注意的一件事，比如您有新邮件时，它并不总是会推送到您的终端（而其他一些通知，比如`wall`，会）。这些消息会在下次更新终端时打印出来；这通常在您输入新命令时（或者只是一个空的*Enter*）时完成。如果您正在处理这些示例并等待输出，请随时按*Enter*几次，看看是否会有什么出现！

尽管获取我们作业的输出有时很棒，但往往会非常烦人，因为许多进程可能会发送本地邮件给您。通常情况下，这将导致您不查看邮件，甚至主动抑制命令的输出，以便您不再收到更多的邮件。在本章后面，介绍了`cron`之后，我们将花一些时间描述如何*正确处理输出*。作为一个小预览，这意味着我们不会依赖这种内置的能力，而是会使用重定向**将我们需要的输出写入我们知道的地方。**

# cron

现在，通过`at`进行调度的基础知识已经讨论过了，让我们来看看 Linux 上真正强大的调度工具：`cron`。`cron`的名称源自希腊词*chronos*，意思是*时间*，它是一个作业调度程序，由两个主要组件组成：*cron 守护进程*（有时称为*crond*）和*crontab*。cron 守护进程是运行预定作业的后台进程。这些作业是使用 crontab 进行预定的，它只是文件系统上的一个文件，通常使用同名命令`crontab`进行编辑。我们将首先看一下`crontab`命令和语法。

# crontab

Linux 系统上的每个用户都可以有自己的 crontab。还有一个系统范围的 crontab（不要与可以在 root 用户下运行的 crontab 混淆！），用于周期性任务；我们稍后会在本章中介绍这些。现在，我们将首先探索 crontab 的语法，并为我们的读者用户创建我们的第一个 crontab。

# crontab 的语法

虽然语法可能一开始看起来令人困惑，但实际上并不难理解，而且非常灵活：

<时间戳>命令

哇，这太容易了！如果真是这样的话，那是的。然而，我们上面描述的<时间戳>实际上由五个不同的字段组成，这些字段组成了运行作业多次的组合周期。实际上，时间戳的定义如下（按顺序）：

1.  一小时中的分钟

1.  一天中的小时

1.  一个月中的日期

1.  月份

1.  星期几

在任何这些值中，我们可以用一个通配符替换一个数字，这表示*所有值*。看一下下表，了解一下我们如何组合这五个字段来精确表示时间：

| ** Crontab     语法** | ** 语义含义** |
| --- | --- |
|  15 16 * * * |  每天 16:15。 |
|  30 * * * * |  每小时一次，xx:30（因为每小时都有效，所以通配符）。 |
|  * 20 * * * |  每天 60 次，从 20:00 到 20:59（小时固定，分钟有通配符）。 |
|  10 10 1 * * |  每个月 1 日的 10:10。 |
|  00 21 * * 1 |  每周一次，周一 21:00（1-7 代表周一到周日，周日也是 0）。 |
|  59 23 31 12 * |  新年前夜，12 月 31 日 23:59。 |
|  01 00 1 1 3 |  在 1 月 1 日 00:01，但仅当那天是星期三时（这将在 2020 年发生）。 |

你可能会对这种语法感到有些困惑。因为我们许多人通常写时间为 18:30，颠倒分钟和小时似乎有点不合常理。然而，这就是事实（相信我们，你很快就会习惯 crontab 格式）。现在，这种语法还有一些高级技巧：

+   8-16（连字符允许多个值，因此`00 8-16 * * *`表示从 08:00 到 16:00 的每个整点）。

+   */5 允许每 5 个*单位*（最常用于第一个位置，每 5 分钟一次）。小时的值*/6 也很有用，每天四次。

+   00,30 表示两个值，比如每小时的 30 分钟或半小时（也可以写成*/30）。

在我们深入理论之前，让我们使用`crontab`命令为我们的用户创建一个简单的第一个 crontab。`crontab`命令有三个最常用的有趣标志：`-l`用于列出，`-e`用于编辑，`-r`用于删除。让我们使用这三个命令创建（和删除）我们的第一个 crontab：

```
reader@ubuntu:~$ crontab -l
no crontab for reader
reader@ubuntu:~$ crontab -e
no crontab for reader - using an empty one

Select an editor.  To change later, run 'select-editor'.
  1\. /bin/nano        <---- easiest
  2\. /usr/bin/vim.basic
  3\. /usr/bin/vim.tiny
  4\. /bin/ed

Choose 1-4 [1]: 2
crontab: installing new crontab
reader@ubuntu:~$ crontab -l
# m h  dom mon dow   command
* * * * * wall "Crontab rules!"

Broadcast message from reader@ubuntu (somewhere) (Sun Nov 25 16:25:01 2018):

Crontab rules!

reader@ubuntu:~$ crontab -r
reader@ubuntu:~$ crontab -l
no crontab for reader
```

正如你所看到的，我们首先列出当前的 crontab 使用`crontab -l`命令。由于我们没有，我们看到消息没有读者的 crontab（没有什么意外的）。接下来，当我们使用`crontab -e`开始编辑 crontab 时，我们会得到一个选择：我们想使用哪个编辑器？像往常一样，选择最适合你的。我们有足够的经验使用`vim`，所以我们更喜欢它而不是`nano`。我们只需要为每个用户做一次，因为 Linux 会保存我们的偏好（查看~/.selected_editor 文件）。最后，我们会看到一个文本编辑器屏幕，在我们的 Ubuntu 机器上，上面填满了有关 crontab 的小教程。由于所有这些行都以#开头，都被视为注释，不会影响执行。通常情况下，我们会删除除了语法提示之外的所有内容：m h dom mon dow command。你可能会忘记这个语法几次，这就是为什么这个小提示在你需要快速编辑时非常有帮助的原因，尤其是如果你有一段时间没有与 crontab 交互了。

我们使用最简单的时间语法创建一个 crontab：在所有五个位置上都使用通配符。简单地说，这意味着指定的命令每分钟运行一次。保存并退出后，我们最多等待一分钟，然后我们就会看到`wall "Crontab rules!";`命令的结果，这是我们自己用户的广播，对系统上的所有用户可见。因为这种构造会严重干扰系统，我们使用`crontab -r`在单次广播后删除 crontab。或者，我们也可以删除那一行或将其注释掉。

一个 crontab 可以有很多条目。每个条目都必须放在自己的一行上，有自己的时间语法。这允许用户安排许多不同的作业，以不同的频率。因此，`crontab -r`并不经常使用，而且本身相当破坏性。我们建议您始终使用`crontab -e`来确保您不会意外删除整个作业计划，而只是您想要删除的部分。

如上所述，所有的 crontab 都保存在文件系统中的文件中。你可以在/var/spool/cron/crontabs/目录中找到它们。这个目录只有 root 用户才能访问；如果所有用户都能看到彼此的作业计划，那将会有一些很大的隐私问题。然而，如果你使用`sudo`成为 root 用户，你会看到以下内容：

```
reader@ubuntu:~$ sudo -i
[sudo] password for reader: 
root@ubuntu:~# cd /var/spool/cron/crontabs/
root@ubuntu:/var/spool/cron/crontabs# ls -l
total 4
-rw------- 1 reader crontab 1090 Nov 25 16:51 reader
```

如果我们打开这个文件（`vim`、`less`、`cat`，无论你喜欢哪个），我们会看到与读者用户的`crontab -e`显示的内容相同。然而，作为一个一般规则，总是使用可用的工具来编辑这样的文件！这样做的主要附加好处是，这些工具不允许你保存不正确的格式。如果我们手动编辑 crontab 文件并弄错了时间语法，整个 crontab 将不再工作。如果你用`crontab -e`做同样的事情，你会看到一个错误，crontab 将不会被保存，如下所示：

```
reader@ubuntu:~$ crontab -e
crontab: installing new crontab
"/tmp/crontab.ABXIt7/crontab":23: bad day-of-week
errors in crontab file, can't install.
Do you want to retry the same edit? (y/n)
```

在前面的例子中，我们输入了一行`* * * * true`。从错误中可以看出，cron 期望一个数字或通配符，但它找到了命令`true`（你可能还记得，这是一个简单返回退出码 0 的命令）。它向用户显示错误，并拒绝保存新的编辑，这意味着所有以前的计划任务都是安全的，将继续运行，即使我们这次搞砸了。

crontab 的时间语法允许几乎任何你能想到的组合。然而，有时你并不真的关心一个确切的时间，而更感兴趣的是确保某些东西每小时、每天、每周，甚至每月运行。Cron 为此提供了一些特殊的时间语法：而不是通常插入的五个值，你可以告诉 crontab`@hourly`、`@daily`、`@weekly`和`@monthly`。

# 记录脚本结果

按计划运行脚本是自动化重复任务的一种很好的方式。然而，在这样做时有一个很大的考虑因素：日志记录。通常，当你运行一个命令时，输出会直接显示给你。如果有什么问题，你就在键盘后面调查问题。然而，一旦我们开始使用`cron`（甚至`at`），我们就再也看不到命令的直接输出了。我们只能在登录后检查结果，如果我们没有做安排，我们只能寻找*脚本的结果*（例如，清理后的日志文件）。我们需要的是脚本的日志记录，这样我们就有一个简单的方法定期验证我们的脚本是否成功运行。

# Crontab 环境变量

在我们的 crontab 中，我们可以定义环境变量，这些变量将被我们的命令和脚本使用。crontab 的这个功能经常被使用，但大多数情况下只用于三个环境变量：PATH、SHELL 和 MAILTO。我们将看看这些变量的用例/必要性。

# 路径

通常，当你登录到 Linux 系统时，你会得到一个*登录 shell*。登录 shell 是一个完全交互的 shell，为你做了一些很酷的事情：它设置了 PS1 变量（决定了你的提示符的外观），正确设置了你的 PATH 等等。现在，你可能会想象，除了登录 shell 还有其他东西。从技术上讲，有两个维度构成了四种不同类型的 shell：

|  | **登录** | **非登录** |
| --- | --- | --- |
| **交互式** | 交互式登录 shell | 交互式非登录 shell |
| **非交互式** | 非交互式登录 shell | 非交互式非登录 shell |

大多数情况下，你会使用*交互式登录 shell*，比如通过（SSH）连接或直接通过终端控制台。另一个经常遇到的 shell 是*非交互式非登录 shell*，这是在通过`at`或`cron`运行命令时使用的。其他两种也是可能的，但我们不会详细讨论你何时会得到这些。

所以，现在你知道我们在`at`和`cron`中得到了不同类型的 shell，我们相信你想知道区别是什么（也就是说，你为什么关心这个问题？）。有一些文件在 Bash 中设置你的配置文件。其中一些在这里列出：

+   `/etc/profile`

+   `/etc/bash.bashrc`

+   `~/.profile`

+   `~/.bashrc`

前两个位于/etc/中，是系统范围的文件，因此对所有用户都是相同的。后两个位于你的主目录中，是个人的；这些可以被编辑，例如，添加你想使用的别名。`alias`命令用于为带有标志的命令创建一个简写。在 Ubuntu 18.04 上，默认情况下，~/.bashrc 文件包含一行`alias ll='ls -alF'`，这意味着你可以输入`ll`，而执行`ls -alF`。

不详细介绍（并且过于简化了很多），交互式登录 shell 读取和解析所有这些文件，而非交互式非登录 shell 不会（有关更深入的信息，请参见*进一步阅读*部分）。一如既往，一幅图值千言，所以让我们自己来看看区别：

```
reader@ubuntu:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
reader@ubuntu:~$ echo $PS1
\[\e]0;\u@\h: \w\a\]${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$
reader@ubuntu:~$ echo $0
-bash
reader@ubuntu:~$ at now
warning: commands will be executed using /bin/sh
at> echo $PATH
at> echo $PS1
at> echo $0
at> <EOT>
job 19 at Sat Dec  1 10:36:00 2018
You have mail in /var/mail/reader
reader@ubuntu:~$ tail -5 /var/mail/reader 
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
$
sh
```

正如我们在这里看到的，普通（SSH）shell 和`at`执行的命令之间的值是不同的。这对 PS1 和 shell 本身都是如此（我们可以通过$0 找到）。然而，对于`at`，PATH 与交互式登录会话的 PATH 相同。现在，看看如果我们在 crontab 中这样做会发生什么：

```
reader@ubuntu:~$ crontab -e
crontab: installing new crontab
reader@ubuntu:~$ crontab -l
# m h  dom mon dow   command
* * * * * echo $PATH; echo $PS1; echo $0
You have mail in /var/mail/reader
reader@ubuntu:~$ tail -4 /var/mail/reader 
/usr/bin:/bin
$
/bin/sh
reader@ubuntu:~$ crontab -r # So we don't keep doing this every minute!
```

首先，PS1 等于`at`看到的内容。由于 PS1 控制 shell 的外观，这只对交互式会话有趣；`at`和`cron`都是非交互式的。如果我们继续看**PATH**，我们会看到一个非常不同的故事：当在`cron`中运行时，我们得到的是/usr/bin:/bin，而不是/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin！简单地说，这意味着对于所有在/bin/和/usr/bin/之外的命令，我们需要使用完全限定的文件名。这甚至体现在$0 的差异（sh 与/bin/sh）。虽然这并不是严格必要的（因为/bin/实际上是 PATH 的一部分），但在与`cron`相关的任何事情上看到完全限定的路径仍然是很典型的。

现在，我们有两种选择来处理这个问题，如果我们想要防止诸如`sudo: command not found`之类的错误。我们可以确保对所有命令始终使用完全限定的路径（实际上，这样做肯定会失败几次），或者我们可以确保为 crontab 设置一个 PATH。第一种选择会给我们所有与`cron`相关的事情带来更多的额外工作。第二种选择实际上是确保我们消除这个问题的一个非常简单的方法。我们只需在 crontab 的顶部包含一个`PATH=...`，所有由 crontab 执行的事情都使用那个 PATH。试一下以下内容：

```
reader@ubuntu:~$ crontab -e
no crontab for reader - using an empty one
crontab: installing new crontab
reader@ubuntu:~$ crontab -l
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
# m h  dom mon dow   command
* * * * * echo $PATH
reader@ubuntu:~$
You have new mail in /var/mail/reader
reader@ubuntu:~$ crontab -r
reader@ubuntu:~$ tail -2 /var/mail/reader 
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```

很简单。如果你想亲自验证这一点，你可以保持默认的 PATH 并从/sbin/运行一些东西（比如`blkid`命令，它显示你的磁盘/分区的信息）。由于这不在 PATH 上，如果你不使用完全限定的方式运行它，你会遇到错误/bin/sh: 1: blkid: not found in your local mail。选择任何你通常可以运行的命令并尝试一下！

通过简单地添加到 crontab 中，你可以节省大量的时间和精力来排除错误。就像调度中的所有事情一样，你通常需要等待至少几分钟才能运行每个脚本尝试，这使得故障排除成为一种耗时的实践。请自己一个忙，确保在 crontab 的第一行包含一个相关的 PATH。

# SHELL

从我们看到的**PATH**的输出中，应该很清楚，`at`和`cron`默认使用/bin/sh。你可能很幸运，有一个/bin/sh 默认为 Bash 的发行版，但这并不一定是这样，尤其是如果你跟着我们的 Ubuntu 18.04 安装走的话！在这种情况下，如果我们检查/bin/sh，我们会看到完全不同的东西：

```
reader@ubuntu:~$ ls -l /bin/sh
lrwxrwxrwx 1 root root 4 Apr 26  2018 /bin/sh -> dash
```

Dash 是***D**ebian **A**lmquist **sh**ell*，它是最近 Debian 系统（你可能记得 Ubuntu 属于 Debian 发行系列）上的默认系统 shell。虽然 Dash 是一个很棒的 shell，有它自己的一套优点和缺点，但这本书是为 Bash 编写的。所以，对于我们的用例来说，让`cron`默认使用 Dash shell 并不实际，因为这将不允许我们使用酷炫的 Bash 4.x 功能，比如高级重定向、某些扩展等。幸运的是，当我们运行我们的命令时，我们可以很容易地设置`cron`应该使用的 shell：我们使用 SHELL 环境变量。设置这个非常简单：

```
reader@ubuntu:~$ crontab -e
crontab: installing new crontab
reader@ubuntu:~$ crontab -l
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
# m h  dom mon dow   command
* * * * * echo $0
reader@ubuntu:~$
You have mail in /var/mail/reader
reader@ubuntu:~$ tail -3 /var/mail/reader
/bin/bash
reader@ubuntu:~/scripts/chapter_14$ crontab -r
```

只需简单地添加 SHELL 环境变量，我们确保了不会因为某些 Bash 功能不起作用而感到困惑。预防这些问题总是一个好主意，而不是希望你能迅速发现它们，特别是如果你仍在掌握 shell 脚本。

# MAILTO

现在我们已经确定我们可以在 crontab 中使用环境变量，通过检查 PATH 和 SHELL，让我们看看另一个非常重要的变量 MAILTO。从名称上可以猜到，这个变量控制邮件发送的位置。你可能记得，当命令有 stdout 时（几乎所有命令都有），邮件会被发送。这意味着对于 crontab 执行的每个命令，你可能会收到一封本地邮件。你可能会怀疑，这很快就会变得很烦人。我们可以在我们放置在 crontab 中的所有命令后面加上一个不错的`&> /dev/null`（记住，`&>`是 Bash 特有的，对于默认的 Dash shell 不起作用）。然而，这意味着我们根本不会有任何输出，无论是邮件还是其他。除了这个问题，我们还需要将它添加到所有我们的行中；这并不是一个真正实用的、可行的解决方案。在接下来的几页中，我们将讨论如何将输出重定向到我们想要的地方。然而，在达到这一点之前，我们需要能够操纵默认的邮件。

一个选择是要么不安装或卸载`sendmail`。这对于你们中的一些人可能是一个很好的解决方案，但对于其他人来说，他们有另一个需要在系统上安装`sendmail`，所以它不能被移除。那么呢？我们可以像使用**PATH**一样使用 MAILTO 变量；我们在 crontab 的开头设置它，邮件将被正确重定向。如果我们清空这个变量，通过将它赋值为空字符串`""`，则不会发送邮件。这看起来像这样：

```
reader@ubuntu:~$ crontab -e
no crontab for reader - using an empty one
crontab: installing new crontab
reader@ubuntu:~$ crontab -l
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
MAILTO=""
# m h dom mon dow command
* * * * * echo "So, I guess we'll never see this :("
```

到目前为止，我们已经经常使用`tail`命令，但实际上它有一个很棒的小标志`--follow`（`-f`），它允许我们查看文件是否有新行被写入。这通常用于*tail a logfile*，但在这种情况下，它允许我们通过 tailing /var/mail/reader 文件来查看是否收到邮件。

```
reader@ubuntu:~$ tail -f /var/mail/reader 
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/home/reader>
X-Cron-Env: <PATH=/usr/bin:/bin>
X-Cron-Env: <LOGNAME=reader>

/bin/bash: 1: blkid: not found
```

如果一切都按我们的预期进行，这将是你看到的唯一的东西。由于 MAILTO 变量被声明为空字符串`""`，`cron`知道不发送邮件。使用*Ctrl* + *C*退出`tail -f`（但记住这个命令），现在你可以放心了，因为你已经阻止了自己被 crontab 垃圾邮件轰炸！

# 使用重定向进行日志记录

虽然邮件垃圾邮件已经消除，但现在你发现自己根本没有任何输出，这绝对也不是一件好事。幸运的是，我们在第十二章中学到了有关重定向的一切，*在脚本中使用管道和重定向**。*就像我们可以在脚本中使用*重定向*或*在命令行中*使用一样，我们可以在 crontab 中使用相同的结构。管道和 stdout/stderr 的顺序规则也适用，所以我们可以链接任何我们想要的命令。然而，在我们展示这个之前，我们将展示 crontab 的另一个很酷的功能：从文件实例化一个 crontab！

```
reader@ubuntu:~/scripts/chapter_14$ vim base-crontab
reader@ubuntu:~/scripts/chapter_14$ cat base-crontab 
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
MAILTO=""
# m h  dom mon dow   command
reader@ubuntu:~/scripts/chapter_14$ crontab base-crontab
reader@ubuntu:~/scripts/chapter_14$ crontab -l
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
MAILTO=""
# m h  dom mon dow   command
```

首先，我们创建 base-crontab 文件，其中包含我们的 Bash SHELL、我们修剪了一点的 PATH、MAILTO 变量和我们的语法头。接下来，我们使用`crontab base-crontab`命令。简单地说，这将用文件中的内容替换当前的 crontab。这意味着我们现在可以将 crontab 作为一个文件来管理；这包括对版本控制系统和其他备份解决方案的支持。更好的是，使用`crontab <filename>`命令时，语法检查是完整的。如果文件不是正确的 crontab 格式，你会看到错误“crontab 文件中的错误，无法安装”。如果你想将当前的 crontab 保存到一个文件中，`crontab -l > filename`命令会为你解决问题。

既然这样，我们将给出一些由 crontab 运行的命令的重定向示例。我们将始终从一个文件实例化，这样你就可以在 GitHub 页面上轻松找到这些材料：

```
reader@ubuntu:~/scripts/chapter_14$ cp base-crontab date-redirection-crontab
reader@ubuntu:~/scripts/chapter_14$ vim date-redirection-crontab 
reader@ubuntu:~/scripts/chapter_14$ cat date-redirection-crontab 
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
MAILTO=""
# m h  dom mon dow   command
* * * * * date &>> /tmp/date-file
reader@ubuntu:~/scripts/chapter_14$ crontab date-redirection-crontab 
reader@ubuntu:~/scripts/chapter_14$ tail -f /tmp/date-file
Sat Dec 1 15:01:01 UTC 2018
Sat Dec 1 15:02:01 UTC 2018
Sat Dec 1 15:03:01 UTC 2018
^C
reader@ubuntu:~/scripts/chapter_14$ crontab -r
```

现在，这很容易。只要我们的 SHELL、PATH 和 MAILTO 设置正确，我们就避免了在使用 crontab 进行调度时通常会遇到的很多问题。

我们还没有运行一个脚本来使用 crontab。到目前为止，只运行了单个命令。但是，脚本也可以很好地运行。我们将使用上一章的脚本 reverser.sh，它将显示我们也可以通过 crontab 向脚本提供参数。此外，它将显示我们刚学到的重定向对脚本输出同样有效：

```
reader@ubuntu:~/scripts/chapter_14$ cp base-crontab reverser-crontab
reader@ubuntu:~/scripts/chapter_14$ vim reverser-crontab 
reader@ubuntu:~/scripts/chapter_14$ cat reverser-crontab 
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin
MAILTO=""
# m h dom mon dow command
* * * * * /home/reader/scripts/chapter_13/reverser.sh 'crontab' &>> /tmp/reverser.log
reader@ubuntu:~/scripts/chapter_14$ crontab reverser-crontab 
reader@ubuntu:~/scripts/chapter_14$ cat /tmp/reverser.log
/bin/bash: /home/reader/scripts/chapter_13/reverser.sh: Permission denied
reader@ubuntu:~/scripts/chapter_14$ crontab -r
```

哎呀！尽管我们做了仔细的准备，但我们还是搞砸了。幸运的是，我们创建的输出文件（因为它是一个日志文件，所以扩展名为.log）也有 stderr 重定向（因为我们的 Bash 4.x `&>>`语法），我们看到了错误。在这种情况下，经典的错误“权限被拒绝”简单地意味着我们试图执行一个非可执行文件：

```
reader@ubuntu:~/scripts/chapter_14$ ls -l /home/reader/scripts/chapter_13/reverser.sh 
-rw-rw-r-- 1 reader reader 933 Nov 17 15:18 /home/reader/scripts/chapter_13/reverser.sh
```

所以，我们需要修复这个问题。我们可以做两件事：

+   使用（例如）`chmod 755 reverser.sh`使文件可执行。

+   将 crontab 从`reverser.sh`更改为`bash reverser.sh`。

在这种情况下，没有真正好坏之分。一方面，标记需要执行的文件为可执行文件总是一个好主意；这向看到系统的人表明你是有意这样做的。另一方面，如果在 crontab 中添加额外的`bash`命令可以避免这类问题，那又有什么坏处呢？

在我们看来，使文件可执行并在 crontab 中省略`bash`命令略有优势。这样可以保持 crontab 的清洁（并且根据经验，如果处理不当，crontab 很容易变得混乱，所以这是一个非常大的优点），并向查看脚本的其他人表明由于权限问题应该执行它。让我们在我们的机器上应用这个修复：

```
reader@ubuntu:~/scripts/chapter_14$ chmod 755 ../chapter_13/reverser.sh
reader@ubuntu:~/scripts/chapter_14$ crontab reverser-crontab
reader@ubuntu:~/scripts/chapter_14$ tail -f /tmp/reverser.log
/bin/bash: /home/reader/scripts/chapter_13/reverser.sh: Permission denied
Your reversed input is: _batnorc_
^C
reader@ubuntu:~/scripts/chapter_14$ crontab -r
```

好了，好多了。我们在 crontab 中运行的完整命令是`/home/reader/scripts/chapter_13/reverser.sh 'crontab' &>> /tmp/reverser.log`，其中包括单词 crontab 作为脚本的第一个参数。输出 _batnorc_ 确实是反转后的单词。看来我们可以通过 crontab 正确传递参数！虽然这个例子说明了这一点，但可能并不足以说明这可能是重要的。但是，如果你想象一个通用脚本，通常会使用不同的参数多次，那么它也可以在 crontab 中以不同的参数出现（可能在多行上，也许有不同的计划）。确实非常有用！

如果您需要快速查看 crontab 的情况，您当然会查看`man crontab`。但是，我们还没有告诉您的是，有些命令实际上有多个 man 页面！默认情况下，`man crontab`是`man <first-manpage> crontab`的简写。在该页面上，您将看到这样的句子：“SEE ALSO crontab(5), cron(8)”。通过向`man 5 crontab`提供此数字，您将看到一个不同的页面，其中本章的许多概念（语法、环境变量和示例）都很容易访问。

# 最终的日志记录考虑

您可能考虑让您的脚本自行处理其日志记录。虽然这当然是可能的（尽管有点复杂且不太可读），但我们坚信**调用者有责任处理日志记录**。如果您发现一个脚本自行处理其日志记录，您可能会遇到以下一些问题：

+   多个用户以不同的间隔运行相同的脚本，将输出到单个日志文件

+   日志文件需要具有健壮的用户权限，以确保正确的暴露

+   临时和定期运行都将出现在日志文件中

简而言之，将日志记录的责任委托给脚本本身是在自找麻烦。对于临时命令，您可以在终端中获得输出。如果您需要它用于其他任何目的，您可以随时将其复制并粘贴到其他地方，或者重定向它。更有可能的是使用管道运行脚本到`tee`，因此输出同时显示在您的终端上*并*保存到文件中。对于从`cron`进行的定期运行，您需要在创建计划时考虑重定向。在这种情况下，特别是如果您使用 Bash 4.x 的`&>>`构造，您将始终看到所有输出（stdout 和 stderr）都附加到您指定的文件中。在这种情况下，几乎没有错过任何输出的风险。记住：`tee`和重定向是您的朋友，当正确使用时，它们是任何脚本调度的重要补充！

如果您希望您的 cron 日志记录机制变得*非常花哨*，您可以设置`sendmail`（或其他软件，如`postfix`）作为实际的邮件传输代理（这超出了本书的范围，但请查看*进一步阅读*部分！）。如果正确配置，您可以在 crontab 中将 MAILTO 变量设置为实际的电子邮件地址（也许是`yourname@company.com`），并在您的常规电子邮件邮箱中接收来自定期作业的报告。这最适用于不经常运行的重要脚本；否则，您将只会收到大量令人讨厌的电子邮件。

# 关于冗长的说明

重要的是要意识到，就像直接在命令行上一样，只有输出（stdout/stderr）被记录。默认情况下，大多数成功运行的命令没有任何输出；其中包括`cp`、`rm`、`touch`等。如果您希望在脚本中进行信息记录，您有责任在适当的位置添加输出。最简单的方法是偶尔使用`echo`。使日志文件对用户产生信心的最简单方法是在脚本的最后一个命令中使用`echo "一切顺利，退出脚本。"`。只要您在脚本中正确处理了所有潜在的错误，您可以安全地说一旦达到最后一个命令，执行就已成功，您可以通知用户。如果不这样做，日志文件可能会保持空白，这可能有点可怕；它是空白的，因为一切都成功了*还是因为脚本甚至没有运行*？这不是您想冒险的事情，尤其是当一个简单的`echo`可以帮您省去所有这些麻烦。

# 摘要

我们通过展示新的`at`命令开始了本章，并解释了如何使用`at`来安排脚本。我们描述了`at`的时间戳语法以及它包含了所有计划作业的队列。我们解释了`at`主要用于临时安排的命令和脚本，然后继续介绍了更强大的`cron`调度程序。

`cron`守护程序负责系统上大多数计划任务，它是一个非常强大和灵活的调度程序，通常通过所谓的 crontab 来使用。这是一个用户绑定的文件，其中包含了关于`cron`何时以及如何运行命令和脚本的指令。我们介绍了在 crontab 中使用的时间戳语法。

本章的第二部分涉及记录我们的计划命令和脚本。当在命令行上交互运行命令时，不需要专门的记录，但计划的命令不是交互式的，因此需要额外的机制。计划命令的输出可以使用`sendmail`进程发送到本地文件，也可以使用我们之前概述的重定向可能性将其重定向到日志文件中。

我们在本章结束时对日志记录进行了一些最终考虑：始终由调用者负责安排日志记录，并且脚本作者有责任确保脚本足够详细以便非交互式地使用。

本章介绍了以下命令：`at`，`wall`，`atq`，`atrm`，`sendmail`，`crontab`和`alias`。

# 问题

1.  什么是调度？

1.  我们所说的临时调度是什么意思？

1.  使用`at`运行的命令的输出通常会去哪里？

1.  `cron`守护程序的调度最常见的实现方式是什么？

1.  哪些命令允许您编辑个人的 crontab？

1.  在 crontab 时间戳语法中有哪五个字段？

1.  crontab 的三个最重要的环境变量是哪些？

1.  我们如何检查我们使用`cron`计划的脚本或命令的输出？

1.  如果我们计划的脚本没有足够的输出让我们有效地使用日志文件，我们应该如何解决这个问题？

# 进一步阅读

如果您想更深入地了解本章的主题，以下资源可能会很有趣：

+   **配置文件和 Bashrc**：[`bencane.com/2013/09/16/understanding-a-little-more-about-etcprofile-and-etcbashrc/`](https://bencane.com/2013/09/16/understanding-a-little-more-about-etcprofile-and-etcbashrc/)

+   **使用 postfix 设置邮件传输代理**：[`www.hiroom2.com/2018/05/06/ubuntu-1804-postfix-en/`](https://www.hiroom2.com/2018/05/06/ubuntu-1804-postfix-en/)
