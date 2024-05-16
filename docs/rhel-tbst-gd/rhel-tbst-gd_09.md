# 第九章 使用系统工具来排除应用程序问题

在上一章中，我们讨论了故障排除硬件问题。具体来说，您学会了当硬盘从 RAID 中移除并且无法读取时该怎么做。

在本章中，我们将回到排除应用程序问题，但与之前的例子不同，我们将不再排除像 WordPress 这样的流行开源应用程序。在本章中，我们将专注于一个自定义应用程序，这将比一个知名应用程序更难排除故障。

# 开源与自制应用程序

流行的开源项目通常有在线社区或错误/问题跟踪器。正如我们在第三章中所经历的，*故障排除 Web 应用程序*，这些资源对于排除应用程序问题非常有用。通常，问题已经在这些社区中报告或询问过，其中大多数帖子也包含了问题的解决方案。

这些解决方案被发布在互联网上的开放论坛上；应用程序的任何错误也可以直接在谷歌上搜索。大多数情况下，搜索结果会显示多个可能的答案。当一个流行的开源应用程序的错误在谷歌上产生零搜索结果时，这是一个非常罕见的情况。

然而，对于自定义应用程序，应用程序错误可能并不总是可以通过快速的谷歌搜索来解决。有时，应用程序会提供通用错误，比如**权限被拒绝**或**文件未找到**。然而，有时候它们不会产生错误，或者产生特定于应用程序的错误，比如我们今天将要处理的问题。

面对开源工具中不明确的错误时，您总是可以在某个在线网站上寻求帮助。然而，对于自定义应用程序，您可能并不总是有机会询问开发人员错误的含义。

有时，系统管理员需要在开发人员几乎没有帮助的情况下修复应用程序。

当出现这种情况时，管理员手头有很多工具可供使用。在今天的章节中，我们将探索其中一些工具，当然，也会排除自定义应用程序的故障。

# 当应用程序无法启动时

对于本章的问题，我们将像处理大多数其他问题一样开始，但今天，我们不是收到警报或电话，而是被另一位系统管理员问了一个问题。

系统管理员正在尝试在博客 Web 服务器上启动一个应用程序。当他们尝试启动应用程序时，它似乎正在启动；然而，在最后，它只是打印出一个错误消息并退出。

对于这种情况，我们的第一个反应当然是故障排除过程中的第一步——复制它。

另一位系统管理员告诉我们，他们通过执行以下步骤来启动应用程序：

1.  以`vagrant`用户登录服务器

1.  移动到目录`/opt/myapp`

1.  运行脚本`start.sh`

在进一步进行之前，让我们尝试同样的步骤：

```
$ whoami
vagrant
$ cd /opt/myapp/
$ ls -la
total 8
drwxr-xr-x. 5 vagrant vagrant  69 May 18 03:11 .
drwxr-xr-x. 4 root    root     50 May 18 00:48 ..
drwxrwxr-x. 2 vagrant vagrant  24 May 18 01:14 bin
drwxrwxr-x. 2 vagrant vagrant  23 May 18 00:51 conf
drwxrwxr-x. 2 vagrant vagrant   6 May 18 00:50 logs
-rwxr-xr-x. 1 vagrant vagrant 101 May 18 03:11 start.sh
$ ./start.sh 
Initializing with configuration file /opt/myapp/conf/config.yml
- - - - - - - - - - - - - - - - - - - - - - - - - -
Starting service: [Failed]

```

在前面的步骤中，我们按照之前的管理员的步骤进行，并得到了相同的结果。应用程序似乎未能启动。

在前面的示例中，使用`whoami`命令显示我们以`vagrant`用户登录。在处理应用程序时，这个命令非常方便，因为它可以用来确保正确的系统用户执行启动过程。

我们可以从前面的启动尝试中看到，应用程序未能启动，并显示以下消息：

```
Starting service: [Failed]

```

然而，我们需要知道为什么它无法启动，以及进程是否真的失败了

回答关于进程是否真正失败的问题实际上非常简单。为了做到这一点，我们可以简单地检查应用程序的退出代码，方法是在执行`start.sh`脚本后打印`$?`变量，如下所示：

```
$ echo $?
1

```

## 退出代码

在 Linux 和 Unix 系统上，程序在终止时有能力向其父进程传递一个值。这个值被称为**退出代码**。正在终止或“退出”的程序使用退出代码来告诉调用它的进程该程序是成功还是失败。

对于 POSIX 系统（如 Red Hat Enterprise Linux），标准约定是程序成功退出时以 0 状态代码退出，失败时以非零状态代码退出。由于我们前面的示例以状态代码 1 退出，这意味着应用程序以失败退出。

为了更好地理解退出代码，让我们编写一个快速的脚本来执行一个成功的任务：

```
$ cat /var/tmp/exitcodes.sh 
#!/bin/bash
touch /var/tmp/file.txt

```

这个快速的小 shell 脚本执行一个任务，它在文件`/var/tmp/file.txt`上运行`touch`命令。如果该文件存在，touch 命令只会更新该文件的访问时间。如果文件不存在，touch 命令将创建它。

由于`/var/tmp`是一个具有开放权限的临时目录，这个脚本在 vagrant 用户执行时应该是成功的：

```
$ /var/tmp/exitcodes.sh

```

执行命令后，我们可以通过使用 BASH 特殊变量`$?`来查看退出代码。这个变量是 BASH shell 中的一个特殊变量，只能用于读取上一个程序的退出代码。这个变量是 BASH shell 中的几个特殊变量之一，只能读取，不能写入。

要查看我们脚本的退出状态，我们可以将`$?`的值`echo`到屏幕上：

```
$ echo $?
0

```

看起来这个脚本返回了`0`退出状态。这意味着脚本成功执行，很可能更新或创建了文件`/var/tmp/file.txt`。我们可以通过对文件本身执行`ls -la`来验证文件是否已更新：

```
$ ls -la /var/tmp/file.txt 
-rw-rw-r--. 1 vagrant vagrant 0 May 25 14:25 /var/tmp/file.txt

```

从`ls`命令的输出中，看起来文件最近已更新或创建。

前面的示例展示了脚本成功时会发生什么，但是当脚本失败时会发生什么呢？通过前面脚本的修改版本，我们可以很容易地看到脚本失败时会发生什么：

```
$ cat /var/tmp/exitcodes.sh 
#!/bin/bash
touch /some/directory/that/doesnt/exist/file.txt

```

修改后的版本将尝试在不存在的目录中创建文件。该脚本将因此失败并以指示失败的退出代码退出：

```
$ /var/tmp/exitcodes.sh 
touch: cannot touch '/some/directory/that/doesnt/exist/file.txt': No such file or directory

```

从脚本的输出中，我们可以看到`touch`命令失败了，但是退出代码呢？

```
$ echo $?
1

```

退出代码还显示了脚本失败了。退出代码的标准是成功为`0`，任何非零值都表示失败。一般来说，你会看到`0`或`1`的退出代码。然而，一些应用程序会使用其他退出代码来指示特定的失败：

```
$ somecommand
-bash: somecommand: command not found
$ echo $?
127

```

例如，如果我们从 BASH shell 执行一个不存在的命令，提供的退出代码将是`127`。这个退出代码是一个用来指示命令未找到的约定。以下是用于特定目的的退出代码列表：

+   `0`：成功

+   `1`：发生了一般性失败

+   `2`：对 shell 内置的误用

+   `126`：无法执行调用的命令

+   `127`：命令未找到

+   `128`：传递给`exit`命令的无效参数

+   `130`：使用*Ctrl* + *C*键停止命令

+   `255`：提供的退出代码超出了`0 - 255`范围

这个列表是退出代码的一个很好的通用指南。然而，由于每个应用程序都可以提供自己的退出代码，你可能会发现一个命令或应用程序提供的退出代码不在上述列表中。对于开源应用程序，你通常可以查找退出代码的含义。然而，对于自定义应用程序，你可能有也可能没有查找退出代码含义的能力。

## 脚本失败了，还是应用程序失败了？

关于 shell 脚本和退出码的一个有趣的事情是，当执行 shell 脚本时，该脚本的退出码将是最后一个执行的命令的退出码。

为了更好地理解这一点，我们可以再次修改我们的测试脚本：

```
$ cat /var/tmp/exitcodes.sh 
#!/bin/bash
touch /some/directory/that/doesnt/exist/file.txt
echo "It works"

```

前面的命令应该产生一个有趣的结果。`touch`命令将失败；然而，echo 命令将成功。

这意味着当执行时，即使`touch`命令失败，`echo`命令也成功，因此命令行的退出码应该显示脚本成功：

```
$ /var/tmp/exitcodes.sh 
touch: cannot touch '/some/directory/that/doesnt/exist/file.txt': No such file or directory
It works
$ echo $?
0

```

前面的命令是一个不优雅处理错误的脚本的例子。如果我们依赖这个脚本仅通过退出码来提供正确的执行状态，我们将得到错误的结果。

对于系统管理员来说，对于未知脚本持有一些怀疑态度总是好的。我发现许多情况（并且自己写了一些）脚本没有错误检查。因此，我们应该执行的第一步是验证退出码 1 是否确实来自正在启动的应用程序。

为了做到这一点，我们需要阅读启动脚本：

```
$ cat ./start.sh 
#!/bin/bash

HOMEDIR=/opt/myapp

$HOMEDIR/bin/application --deamon --config $HOMEDIR/conf/config.yml

```

从外观上看，启动脚本非常基础。看起来脚本只是将`$HOMEDIR`变量设置为`/opt/myapp`，然后通过运行命令`$HOMEDIR/bin/application`来运行应用程序。

### 提示

在将`$HOMEDIR`的值设置为`/opt/myapp`之后，您可以假设将来对`$HOMEDIR`的任何引用实际上是值`/opt/myapp`。

从前面的脚本中，我们可以看到最后执行的命令是应用程序，这意味着我们收到的退出码来自应用程序而不是其他命令。这证明我们收到了这个应用程序的真实退出状态。

启动脚本确实为我们提供了比仅提供退出码的命令更多的信息。如果我们看一下应用程序的命令行参数，我们可以更多地了解这个应用程序：

```
$HOMEDIR/bin/application --deamon --config $HOMEDIR/conf/config.yml

```

这是实际在`start.sh`脚本中启动应用程序的命令。该脚本正在使用参数`--daemon`和`--config /opt/myapp/conf/config.yml`运行命令`/opt/myapp/bin/application`。虽然我们可能对这个应用程序了解不多，但我们可以做一些假设。

我们可以假设`--daemon`标志导致这个应用程序使自己成为守护进程。在 Unix 和 Linux 系统上，作为后台进程持续运行的进程被称为守护进程。

通常，守护进程是一个不需要用户输入的服务。一些容易识别的守护进程的例子是 Apache 或 MySQL。这些进程在后台运行并提供服务，而不是在用户的桌面或 shell 中运行。

通过前面的标志，我们可以安全地假设一旦成功启动，这个进程就被设计为在后台运行。

基于命令行参数，我们可以做出另一个假设，即文件`/opt/myapp/conf/config.yml`被用作应用程序的配置文件。考虑到标志被命名为`--config`，这似乎很简单明了。

前面的假设很容易识别，因为标志使用长格式`--option`。然而，并非所有应用程序或服务都使用命令行标志的长格式。通常，这些是单字符标志。

虽然每个应用程序都有自己的命令行标志，并且可能因应用程序而异，但常见的标志，如`--config`和`--deamon`通常被缩写为`-c`和`-d`或`-D`。如果我们的应用程序提供了单字符标志，它看起来会更像下面的样子：

```
$HOMEDIR/bin/application -d -c $HOMEDIR/conf/config.yml

```

即使使用了缩短的选项，我们仍然可以安全地确定`-c`指定了一个配置文件。

## 配置文件中包含大量信息

我们知道这个应用程序正在使用配置文件`/opt/myapp/conf/config.yml`。如果我们读取这个文件，我们可能会找到关于应用程序以及它正在尝试执行的任务的信息：

```
$ cat conf/config.yml 
port: 25
debug: True
logdir: /opt/myapp/logs

```

这个应用程序的配置文件非常简短，但其中包含了相当多有用的信息。第一个配置项很有趣，因为它似乎指定端口`25`作为应用程序使用的端口。不知道这个应用程序具体做什么，这个信息并不立即有用，但以后可能对我们有用。

第二项似乎表明应用程序处于调试模式。通常应用程序或服务可能有一个`debug`模式，导致它们记录或输出调试信息以进行故障排除。在我们的情况下，似乎调试选项已启用，因为这个项目的值是`True`。

第三项和最后一项是一个看起来是日志的目录路径。日志文件对于故障排除应用程序总是有用的。通常情况下，您可以在日志文件中找到有关应用程序问题的信息。如果应用程序处于`debug`状态，这对我们的应用程序似乎是正确的情况。

由于我们的应用似乎处于`debug`模式，并且我们知道日志目录的位置。我们可以检查日志目录是否有在应用启动过程中创建的日志文件：

```
$ ls -la /opt/myapp/logs/
total 4
drwxrwxr-x. 2 vagrant vagrant  22 May 30 03:51 .
drwxr-xr-x. 5 vagrant vagrant  53 May 30 03:49 ..
-rw-rw-r--. 1 vagrant vagrant 454 May 30 03:54 debug.out

```

如果我们在日志目录中运行`ls -la`，我们可以看到一个`debug.out`文件。根据名称，这个文件很可能是应用程序的调试输出，但不一定是应用程序的主要日志文件。然而，这个文件可能比标准日志文件更有用，因为它可能包含应用程序启动失败的原因：

```
$ cat debug.out 
Configuration file processed
--------------------------
Starting service: [Failed]
Configuration file processed
--------------------------
Starting service: [Success]
- - - - - - - - - - - - - - - - - - - - - - - - - 
Proccessed 5 messages
Proccessed 5 messages
Configuration file processed
--------------------------
Starting service: [Failed]
Configuration file processed
--------------------------
Starting service: [Failed]

```

根据这个文件的内容，似乎这个文件包含了多次执行该应用程序的日志。我们可以根据重复的模式看到这一点。

```
Configuration file processed
--------------------------

```

这似乎是每次应用程序启动时打印的第一项。我们总共可以看到这些行四次；很可能，这意味着这个应用程序过去至少启动了四次。

在这个文件中，我们可以看到一个重要的日志消息：

```
Starting service: [Success]

```

看起来，这个应用程序第二次启动时应用程序启动成功。然而，之后每次启动都失败。

### 在启动过程中观看日志文件

由于调试文件的内容不包括时间戳，很难知道调试输出是否是在我们启动应用程序时编写的，还是在以前的启动过程中编写的。

由于我们不知道哪些行是在我们上次尝试时写入的，而不是其他尝试，我们需要尝试确定每次启动应用程序时写入了多少日志条目。为此，我们可以使用`tail`命令与`-f`或`--follow`标志：

```
$ tail -f debug.out 
- - - - - - - - - - - - - - - - - - - - - - - - - 
Proccessed 5 messages
Proccessed 5 messages
 [Failed]
Configuration file processed
--------------------------
Starting service: [Failed]
Configuration file processed
--------------------------
Starting service: [Failed]

```

当首次使用`-f`（跟踪）标志启动`tail`命令时，将打印文件的最后 10 行。如果没有使用任何标志运行，这也是`tail`的默认行为。

然而，`-f`标志并不仅仅停留在最后 10 行。当使用`-f`标志运行时，`tail`将持续监视指定文件的新数据。一旦`tail`看到指定文件写入新数据，数据将被写入`tail`的输出。

通过对`debug.out`文件运行`tail -f`，我们将能够识别应用程序写入的任何新的调试日志。如果我们再次执行`start.sh`脚本，我们应该看到应用程序在启动过程中打印的任何可能的调试数据：

```
$ ./start.sh 
Initializing with configuration file /opt/myapp/conf/config.yml
- - - - - - - - - - - - - - - - - - - - - - - - - -
Starting service: [Failed]

```

`start.sh`脚本的输出与上次相同，这在这一点上并不奇怪。然而，现在我们正在观看`debug.out`文件，我们可能会找到一些有用的东西：

```
Configuration file processed
--------------------------
Starting service: [Failed]

```

从`tail`命令中，我们可以看到前面三行是在执行`start.sh`时打印的。虽然这本身并不能解释为什么应用程序无法启动，但它确实告诉了我们一些有趣的东西：

```
$ cat debug.out 
Configuration file processed
--------------------------
Starting service: [Failed]
Configuration file processed
--------------------------
Starting service: [Success]
- - - - - - - - - - - - - - - - - - - - - - - - - 
Processed 5 messages
Processed 5 messages
Configuration file processed
--------------------------
Starting service: [Failed]
Configuration file processed
--------------------------
Starting service: [Failed]
Configuration file processed
--------------------------
Starting service: [Failed]

```

考虑到当应用程序无法启动时，“失败”消息会从之前的命令中打印出来，我们可以看到`start.sh`脚本执行的最后三次都失败了。然而，在那之前的实例是成功的。

到目前为止，我执行了启动脚本两次，另一位管理员执行了一次。这可以解释我们在`debug.out`文件末尾看到的三次失败。有趣的是，在这些失败之前，应用程序成功启动了。

这很有趣，因为它表明应用程序的先前实例可能正在运行。

# 检查应用程序是否已经在运行

这种问题的一个非常常见的原因是应用程序已经在运行。有些应用程序应该只启动一次，在完成启动之前，应用程序本身会检查是否有另一个实例正在运行。

一般来说，如果是这种情况，我们期望应用程序会在屏幕上或`debug.out`文件中打印错误。然而，并非每个应用程序都有适当的错误处理或消息传递。这对于定制应用程序尤其如此，似乎也适用于我们正在处理的应用程序。

目前，我们假设我们的问题是由应用程序的另一个实例引起的。这是基于调试消息和以往经验的一个有根据的猜测。虽然我们还没有任何确凿的事实告诉我们是否有另一个实例正在运行，但这种情况是相当常见的。

这种情况是一个**有经验的猜测者**利用以往的经验来建立根本原因的假设的完美例子。当然，在形成假设之后，我们的下一步是验证它是否正确。即使我们的假设最终被证明是错误的，我们至少可以排除我们问题的一个潜在原因。

由于我们目前的假设是我们可能已经有一个应用程序的实例在运行，我们可以通过执行 ps 命令来验证它：

```
$ ps -elf | grep application
0 S vagrant   7110  5567  0  80   0 - 28160 pipe_w 15:22 pts/0    00:00:00 grep --color=auto application

```

从中可以看出，我们的假设可能是不正确的。然而，之前的命令只是执行进程列表，并在输出中搜索任何包含单词“应用程序”的实例。虽然这个命令可能足够了，但是一些应用程序在启动过程中（特别是那些变成守护进程的应用程序）会启动另一个进程，这个进程可能不匹配字符串“应用程序”。

由于我们一直以`vagrant`用户启动应用程序，即使应用程序变成守护进程，进程也会以 vagrant 用户的身份运行。使用相同的命令，我们还可以搜索以`vagrant`用户身份运行的进程列表：

```
$ ps -elf | grep vagrant
4 S root      4230   984  0  80   0 - 32881 poll_s May30 ?        00:00:00 sshd: vagrant [priv]
5 S vagrant   4233  4230  0  80   0 - 32881 poll_s May30 ?        00:00:00 sshd: vagrant@pts/1
0 S vagrant   4234  4233  0  80   0 - 28838 n_tty_ May30 pts/1    00:00:00 -bash
4 S root      5563   984  0  80   0 - 32881 poll_s May31 ?        00:00:00 sshd: vagrant [priv]
5 S vagrant   5566  5563  0  80   0 - 32881 poll_s May31 ?        00:00:01 sshd: vagrant@pts/0
0 S vagrant   5567  5566  0  80   0 - 28857 wait   May31 pts/0    00:00:00 -bash
0 R vagrant   7333  5567  0  80   0 - 30839 -      14:58 pts/0    00:00:00 ps -elf
0 S vagrant   7334  5567  0  80   0 - 28160 pipe_w 14:58 pts/0    00:00:00 grep --color=auto vagrant

```

这个命令给了我们更多的输出，但不幸的是，这些进程中没有一个是我们正在寻找的应用程序。

## 检查打开的文件

之前的进程列表命令没有提供任何结果，表明我们的应用程序的实例正在运行。然而，在假设它实际上没有运行之前，我们应该进行最后一次检查。

由于我们知道我们正在处理的应用程序似乎安装在`/opt/myapp`中，我们可以在该目录中看到配置文件和日志。可以很肯定地假设所讨论的应用程序可能会打开`/opt/myapp`目录中的一个或多个文件。

一个非常有用的命令是**lsof**命令。通过这个命令，我们可以列出系统上所有打开的文件。虽然这一开始可能听起来不太强大，但让我们详细看看这个命令，了解它实际上可以提供多少信息。

当运行`lsof`命令时，权限变得非常重要。当不带任何参数执行`lsof`时，该命令将打印出它能识别的每个进程的所有打开文件的列表。如果我们以非特权用户（如“`vagrant`”用户）身份运行此命令，输出将只包含作为 vagrant 用户运行的进程。然而，如果我们以 root 用户身份运行该命令，该命令将打印系统上所有进程的打开文件。

为了更好地理解这意味着多少文件，我们将运行`lsof`命令并将输出重定向到`wc -l`命令，这将计算输出中提供的行数：

```
# lsof | wc -l
3840

```

从`wc`命令中，我们可以看到当前系统上有`3840`个文件打开。现在，其中一些文件可能是重复的，因为可能有多个进程打开同一个文件。然而，当前系统上打开文件的数量非常大。为了进一步了解，这个系统也是一个相当未被充分利用的系统，一般并没有运行很多应用程序。如果在一个充分利用的系统上执行上述命令后，打开文件的数量呈指数级增长，也不要感到惊讶。

由于查看`3840`个打开文件并不是很实际，让我们通过查看`lsof`输出的前 10 个文件来更好地理解`lsof`。我们可以通过将命令的输出重定向到`head`命令来实现这一点，`head`命令将默认打印 10 行，就像`tail`命令一样。然而，`tail`命令打印最后 10 行，而`head`命令打印前 10 行：

```
# lsof | head
COMMAND    PID TID    USER   FD      TYPE             DEVICE  SIZE/OFF       NODE NAME
systemd      1        root  cwd       DIR              253,1      4096        128 /
systemd      1        root  rtd       DIR              253,1      4096        128 /
systemd      1        root  txt       REG              253,1   1214408   67629956 /usr/lib/systemd/systemd
systemd      1        root  mem       REG              253,1     58288  134298633 /usr/lib64/libnss_files-2.17.so
systemd      1        root  mem       REG              253,1     90632  134373166 /usr/lib64/libz.so.1.2.7
systemd      1        root  mem       REG              253,1     19888  134393597 /usr/lib64/libattr.so.1.1.0
systemd      1        root  mem       REG              253,1    113320  134298625 /usr/lib64/libnsl-2.17.so
systemd      1        root  mem       REG              253,1    153184  134801313 /usr/lib64/liblzma.so.5.0.99
systemd      1        root  mem       REG              253,1    398264  134373152 /usr/lib64/libpcre.so.1.2.0

```

正如我们所看到的，以 root 身份执行的`lsof`命令能够为我们提供相当多有用的信息。让我们只看一下输出的第一行，以了解`lsof`显示了什么：

```
COMMAND    PID TID    USER   FD      TYPE             DEVICE  SIZE/OFF       NODE NAME
systemd      1        root  cwd       DIR              253,1      4096        128 /

```

`lsof`命令打印 10 列，每个打开文件。

第一列是`COMMAND`列。这个字段包含打开文件的可执行文件的名称。当识别哪些进程打开了特定文件时，这是非常有用的。

对于我们的用例，这将告诉我们哪些进程打开了我们感兴趣的文件，并可能告诉我们正在寻找的应用程序的进程名称。

第二列是`PID`列。这个字段和第一个一样有用，因为它显示了打开显示的文件的应用程序的进程 ID。如果实际上正在运行，这个值将允许我们将应用程序缩小到特定的进程。

第三列是`TID`列，在我们的输出中是空白的。这一列包含了所讨论进程的线程 ID。在 Linux 中，多线程应用程序能够生成线程，这些线程也被称为轻量级进程。这些线程类似于常规进程，但能够共享资源，如文件描述符和内存映射。你可能听说过这些被称为线程或轻量级进程，但它们本质上是一样的。

为了看到`TID`字段，我们可以在`lsof`命令中添加`-K`（显示线程）标志。这将导致`lsof`打印所有轻量级进程以及完整进程。

`lsof`输出的第四列是`USER`字段。这个字段将打印打开文件的进程的用户名或`UID`（如果找不到用户名）。重要的是要知道，这个字段是进程正在执行的用户，而不是文件本身的所有者。

例如，如果作为`rotot`运行的进程打开了一个由`vagrant`拥有的文件，`lsof`中的 USER 字段将显示 root。这是因为`lsof`命令用于显示哪些进程打开了文件，并且用于显示有关进程的信息，而不一定是文件。

### 理解文件描述符

第五列非常有趣，因为这是**文件描述符**（**FD**）的字段；这是一个棘手的 Unix 和 Linux 主题。

文件描述符是 POSIX 应用程序编程接口（API）的一部分，这是所有现代 Linux 和 Unix 操作系统遵循的标准。从程序的角度来看，文件描述符是一个由非负数表示的对象。这个数字被用作内核在每个进程基础上管理的打开文件表的标识符。

由于内核在每个进程级别上维护这个数据，数据包含在`/proc`文件系统中。我们可以通过在`/proc/<process id>/fd`目录中执行`ls -la`来查看这个打开文件表：

```
# ls -la /proc/1/fd
total 0
dr-x------. 2 root root  0 May 17 23:07 .
dr-xr-xr-x. 8 root root  0 May 17 23:07 ..
lrwx------. 1 root root 64 May 17 23:07 0 -> /dev/null
lrwx------. 1 root root 64 May 17 23:07 1 -> /dev/null
lrwx------. 1 root root 64 Jun  1 15:08 10 -> socket:[7951]
lr-x------. 1 root root 64 Jun  1 15:08 11 -> /proc/1/mountinfo
lr-x------. 1 root root 64 Jun  1 15:08 12 -> /proc/swaps
lrwx------. 1 root root 64 Jun  1 15:08 13 -> socket:[11438]
lr-x------. 1 root root 64 Jun  1 15:08 14 -> anon_inode:inotify
lrwx------. 1 root root 64 May 17 23:07 2 -> /dev/null
lrwx------. 1 root root 64 Jun  1 15:08 20 -> socket:[7955]
lrwx------. 1 root root 64 Jun  1 15:08 21 -> socket:[13968]
lrwx------. 1 root root 64 Jun  1 15:08 22 -> socket:[13980]
lrwx------. 1 root root 64 May 17 23:07 23 -> socket:[13989]
lrwx------. 1 root root 64 Jun  1 15:08 24 -> socket:[7989]
lrwx------. 1 root root 64 Jun  1 15:08 25 -> /dev/initctl
lrwx------. 1 root root 64 Jun  1 15:08 26 -> socket:[7999]
lrwx------. 1 root root 64 May 17 23:07 27 -> socket:[6631]
lrwx------. 1 root root 64 May 17 23:07 28 -> socket:[6634]
lrwx------. 1 root root 64 May 17 23:07 29 -> socket:[6636]
lr-x------. 1 root root 64 May 17 23:07 3 -> anon_inode:inotify
lrwx------. 1 root root 64 May 17 23:07 30 -> socket:[8006]
lr-x------. 1 root root 64 Jun  1 15:08 31 -> anon_inode:inotify
lr-x------. 1 root root 64 Jun  1 15:08 32 -> /dev/autofs
lr-x------. 1 root root 64 Jun  1 15:08 33 -> pipe:[10502]
lr-x------. 1 root root 64 Jun  1 15:08 34 -> anon_inode:inotify
lrwx------. 1 root root 64 Jun  1 15:08 35 -> anon_inode:[timerfd]
lrwx------. 1 root root 64 Jun  1 15:08 36 -> socket:[8095]
lrwx------. 1 root root 64 Jun  1 15:08 37 -> /run/dmeventd-server
lrwx------. 1 root root 64 Jun  1 15:08 38 -> /run/dmeventd-client
lrwx------. 1 root root 64 Jun  1 15:08 4 -> anon_inode:[eventpoll]
lrwx------. 1 root root 64 Jun  1 15:08 43 -> socket:[11199]
lrwx------. 1 root root 64 Jun  1 15:08 47 -> socket:[14300]
lrwx------. 1 root root 64 Jun  1 15:08 48 -> socket:[14300]
lrwx------. 1 root root 64 Jun  1 15:08 5 -> anon_inode:[signalfd]
lr-x------. 1 root root 64 Jun  1 15:08 6 -> /sys/fs/cgroup/systemd
lrwx------. 1 root root 64 Jun  1 15:08 7 -> socket:[7917]
lrwx------. 1 root root 64 Jun  1 15:08 8 -> anon_inode:[timerfd]
lrwx------. 1 root root 64 Jun  1 15:08 9 -> socket:[7919]

```

这是`systemd`进程的文件描述符表。正如你所看到的，有一个数字，这个数字与一个文件/对象相关联。

这个输出中不容易表示的是这是一个不断变化的过程。当一个文件/对象被关闭时，文件描述符号就可以被内核重新分配给一个新的打开的文件/对象。根据进程打开和关闭文件的频率，如果我们重复相同的 ls 命令，我们可能会在这个表中看到完全不同的一组打开文件。

有了这个，我们期望`lsof`中的 FD 字段总是显示一个数字。然而，`lsof`输出中的 FD 字段实际上可以包含不止文件描述符号。这是因为`lsof`实际上显示的不仅仅是文件。

当执行时，`lsof`命令将打印许多不同类型的打开对象；并非所有这些都是文件。我们之前`lsof`命令输出的第一行就是一个例子：

```
COMMAND    PID TID    USER   FD      TYPE             DEVICE  SIZE/OFF       NODE NAME
systemd      1        root  cwd       DIR              253,1      4096        128 /

```

前面的项目不是一个文件，而是一个目录。因为这是一个目录，FD 字段显示`cwd`，用于表示打开对象的当前工作目录。这实际上与打开对象为文件时打印的输出非常不同。

为了更好地显示区别，我们可以通过将文件作为`lsof`的参数来运行`lsof`命令来针对特定文件运行`lsof`：

```
# lsof /dev/null | head
COMMAND    PID    USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
systemd      1    root    0u   CHR    1,3      0t0   23 /dev/null
systemd      1    root    1u   CHR    1,3      0t0   23 /dev/null
systemd      1    root    2u   CHR    1,3      0t0   23 /dev/null
systemd-j  436    root    0r   CHR    1,3      0t0   23 /dev/null
systemd-j  436    root    1w   CHR    1,3      0t0   23 /dev/null
systemd-j  436    root    2w   CHR    1,3      0t0   23 /dev/null
lvmetad    469    root    0r   CHR    1,3      0t0   23 /dev/null
systemd-u  476    root    0u   CHR    1,3      0t0   23 /dev/null
systemd-u  476    root    1u   CHR    1,3      0t0   23 /dev/null

```

在上面的输出中，我们不仅能够看到许多进程打开了`/dev/null`，而且每行的`FD`字段也有很大不同。如果我们看第一行，我们可以看到`systemd`进程打开了`/dev/null`，而`FD`字段的值是`0u`。

当`lsof`显示一个标准文件的打开对象时，`FD`字段将包含与内核表中该打开文件相关联的文件描述符号，本例中为`0`。

如果我们回顾一下`/proc/1/fd`目录，我们实际上可以在内核表中看到这个表示：

```
# ls -la /proc/1/fd/0
lrwx------. 1 root root 64 May 17 23:07 /proc/1/fd/0 -> /dev/null

```

文件描述符号可能会跟随两个值，这取决于文件的打开方式以及它是否被锁定。

第一个潜在的值显示了文件的打开模式。从我们的例子中，这由`0u`值中的`u`表示。小写的`u`表示文件同时以读写方式打开。

以下是`lsof`将显示的潜在模式列表：

+   `r`：小写的`r`表示文件只能读取

+   `w`：小写的`w`表示文件只能写入打开

+   `u`：小写的`u`表示文件同时以读写方式打开

+   <space>：空格用于表示文件打开的模式未知，并且当前文件上没有锁

+   `-`：连字符用于表示文件打开的模式未知，并且当前文件上有锁

最后两个值实际上非常有趣，因为它们将我们带到文件描述符号后的第二个潜在值。

Linux 和 Unix 系统上的进程在打开文件时允许请求文件被锁定。有多种类型的锁，这也在`lsof`输出中显示出来：

```
master    1586        root   10uW     REG              253,1        33  135127929 /var/spool/postfix/pid/master.pid

```

在前面的示例中，`FD`字段包含`10uW`。根据先前的示例，我们知道 10 是文件描述符号，`u`表示此文件已打开以进行读写，但`W`是新的。这个 W 显示了进程对该文件的锁的类型；对于这个示例来说，是写锁。

与文件打开模式一样，从`lsof`中可以看到许多不同类型的锁。以下是`lsof`显示的可能锁的列表：

+   `N`：用于 Solaris 未知类型的 NFS 锁

+   `r`：这是对文件的部分读取锁

+   `R`：这是对整个文件的读取锁

+   `w`：这是对文件的部分写锁

+   `W`：这是对整个文件的写锁

+   `u`：这是任意长度的读写锁

+   `U`：未知类型的读写锁

+   `x`：这是 SCO Openserver Xenix 对部分文件的锁

+   `X`：这是 SCO Openserver Xenix 对整个文件的锁

您可能会注意到有几种可能的锁并非特定于 Linux。这是因为`lsof`是一种广泛用于 Linux 和 Unix 的工具，并支持许多 Unix 发行版，如 Solaris 和 SCO。

现在我们已经了解了`lsof`如何显示实际文件的`FD`字段，让我们看看它如何显示不一定是文件的打开对象：

```
iprupdate  595        root  cwd       DIR              253,1      4096        128 /
iprupdate  595        root  rtd       DIR              253,1      4096        128 /
iprupdate  595        root  txt       REG              253,1    114784  135146206 /usr/sbin/iprupdate
iprupdate  595        root  mem       REG              253,1   2107600  134298615 /usr/lib64/libc-2.17.so

```

通过这个，我们可以在这个列表中看到很多不同的`FD`值，比如`cwd`、`rtd`、`txt`和`mem`。我们已经从之前的示例中知道，`cwd`用于显示`当前工作目录`，但其他的都是新的。实际上，根据打开的对象，可能有许多不同的可能文件类型。以下列表包含了所有可能的值，如果不使用文件描述符号，则可以显示：

+   `cwd`：当前工作目录

+   `Lnn`：AIX 系统的库引用（`nn`是一个数值）

+   `err`：文件描述符信息错误

+   `jld`：FreeBSD 监禁目录

+   `ltx`：共享库文本

+   `Mxx`：十六进制内存映射（xx 是类型编号）

+   `m86`：DOS 合并映射文件

+   `mem`：内存映射文件

+   `mmap`：内存映射设备

+   `pd`：父目录

+   `rtd`：根目录

+   `tr`：内核跟踪文件

+   `txt`：程序文本

+   `v86`：VP/ix 映射文件

我们可以看到`FD`字段有许多可能的值。既然我们已经看到了可能的值，让我们看一下前面的示例，以更好地理解显示的打开项目的类型：

```
iprupdate  595        root  cwd       DIR              253,1      4096        128 /
iprupdate  595        root  rtd       DIR              253,1      4096        128 /
iprupdate  595        root  txt       REG              253,1    114784  135146206 /usr/sbin/iprupdate
iprupdate  595        root  mem       REG              253,1   2107600  134298615 /usr/lib64/libc-2.17.so

```

前两行很有趣，因为它们都是针对"`/`"目录。但是，第一行显示"`/`"目录为`cwd`，这意味着它是当前工作目录。第二行显示"`/`"目录为`rtd`，这意味着这也是`iprupdate`程序的根目录。

第三行显示`/usr/sbin/iprupdate`是程序本身，因为它的`FD`字段值为`txt`。这意味着打开的文件是程序的代码。第四行打开项目`/usr/lib64/libc-2.17.so`显示了一个`mem`的 FD。这意味着文件`/usr/lib64/libc-2.17.so`已被读取并放入内存中供`iprupdate`进程使用。这意味着这个文件可以被当作内存对象访问。这对于诸如`libc-2.17.so`之类的库文件是一种常见做法。

## 回到`lsof`输出

现在我们已经彻底探讨了`lsof`输出的`FD`字段，让我们转到第六列，即`TYPE`字段。该字段显示正在打开的文件类型。由于可能的类型相当多，要在这里列出它们可能有点棘手；但是，您可以在`lsof`手册页中找到这些信息，该手册页可以在线访问，也可以通过"`man lsof`"命令访问。

虽然我们不会列出每种可能的文件类型，但我们可以快速查看一下从我们的示例系统中捕获的一些文件类型：

```
systemd      1        root  mem       REG              253,1    160240  134296681 /usr/lib64/ld-2.17.so
systemd      1        root    0u      CHR                1,3       0t0         23 /dev/null
systemd      1        root    6r      DIR               0,20         0       6404 /sys/fs/cgroup/systemd
systemd      1        root    7u     unix 0xffff88001d672580       0t0       7917 @/org/freedesktop/systemd1/notify

```

第一个示例项显示`TYPE`为`REG`。这种`TYPE`非常常见，因为被列出的项目是一个`Regular`文件。第二个示例项显示**Character special file** (**CHR**)。CHR 表示特殊文件，它们表现为文件，但实际上是设备的接口。列出的`/dev/null`就是一个字符文件的完美例子，因为它被用作输入到空。任何写入`/dev/null`的内容都会被清空，如果您读取此文件，将不会收到任何输出。

第三项显示`DIR`，这应该不足为奇，`DIR`代表目录。这是一个非常常见的`TYPE`，因为许多进程在某个级别上都需要打开一个目录。

第四项显示了`unix`，表明此打开项目是 Unix 套接字文件。Unix 套接字文件是用作进程通信的输入/输出设备的特殊文件。这些文件应该经常出现在`lsof`输出中。

正如我们从前面的示例中看到的，在 Linux 系统上有几种不同类型的文件。

现在我们已经查看了`lsof`输出中的第六列，即`TYPE`列，让我们快速看一下第七列，即`DEVICE`列：

```
COMMAND    PID TID    USER   FD      TYPE             DEVICE  SIZE/OFF       NODE NAME
systemd      1        root  cwd       DIR              253,1      4096        128 /

```

如果我们看前面的项目，我们可以看到`DEVICE`列的值为`253,1`。这些数字代表此项目所在设备的主设备号和次设备号。Linux 中的主设备号和次设备号被系统用来确定如何访问设备。主设备号，在本例中为`253`，用于确定系统应该使用哪个驱动程序。一旦选择了驱动程序，次设备号，在我们的情况下为 1，然后用于进一步确定如何访问这个设备。

### 提示

主设备号和次设备号实际上是 Linux 及其设备使用的重要部分。虽然我们不会在本书中深入讨论这个主题，但我建议您多了解一些，因为这些信息在故障排除硬件设备问题时非常有用。

```
systemd      1        root  mem       REG              253,1    160240  134296681 /usr/lib64/ld-2.17.so
systemd      1        root    0u      CHR                1,3       0t0         12 /dev/null

```

现在我们已经探索了`DEVICE`列，让我们来看一下`lsof`输出的第八列，`SIZE/OFF`。`SIZE/OFF`列用于显示打开项目的大小或**偏移量**。偏移通常与套接字文件和字符文件一起显示。当此列包含偏移量时，它将以"`0t`"开头。在上面的示例中，我们可以看到字符文件`/dev/null`的偏移值为`0t0`。

`SIZE`值用于指代常规文件等打开项目的大小。这个值实际上是文件的大小（以字节为单位）。例如，我们可以看到`/usr/lib64/ld-2.17.so`的`SIZE`列为`160240`。这意味着这个文件大约有 160 KB 大小。

`lsof`输出中的第九列是`NODE`列：

```
httpd     3205      apache    2w      REG              253,1       497  134812768 /var/log/httpd/error_log
httpd     3205      apache    4u     IPv6              16097       0t0        TCP *:http (LISTEN)

```

对于常规文件，`NODE`列将显示文件的**inode**编号。在文件系统中，每个文件都有一个 inode，这个 inode 被用作包含所有单个文件元数据的索引。这些元数据包括文件在磁盘上的位置、文件权限、创建时间和修改时间等。与主设备号和次设备号一样，我建议深入了解 inode 及其包含的内容，因为 inode 是文件在 Linux 系统上存在的核心组件。

您可以从前面示例中的第一项看到，`/var/log/httpd/error_log`的 inode 是`134812768`。

然而，第二行显示`NODE`为 TCP，这不是一个 inode。它显示 TCP 的原因是因为打开项目是 TCP 套接字，它不是文件系统上的文件。与`TYPE`列一样，`NODE`列将根据打开项目而改变。然而，在大多数系统上，您通常会看到一个 inode 编号、TCP 或 UDP（用于 UDP 套接字）。

`lsof`输出中的第十列是非常容易理解的，因为我们已经多次引用过它。第十列是`NAME`字段，就像它听起来那样简单；它列出了打开项目的名称：

```
COMMAND    PID TID    USER   FD      TYPE             DEVICE  SIZE/OFF       NODE NAME
systemd      1        root  cwd       DIR              253,1      4096        128 /

```

## 使用 lsof 来检查是否有先前运行的进程

现在我们对`lsof`的工作原理和它如何帮助我们有了更多了解，让我们使用这个命令来检查是否有任何正在运行的应用程序实例。

如果我们只是以 root 用户身份运行`lsof`命令，我们将看到系统上所有打开的文件。然而，即使我们将输出重定向到`less`或`grep`等命令，输出也可能会非常庞大。幸运的是，`lsof`允许我们指定要查找的文件和目录：

```
# lsof /opt/myapp/conf/config.yml 
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF      NODE NAME
less    3494 vagrant    4r   REG  253,1       45 201948450 /opt/myapp/conf/config.yml

```

正如我们所看到的，通过指定前面的命令中的一个文件，我们将输出限制为具有该文件打开的进程。

如果我们指定一个目录，输出是类似的：

```
# lsof /opt/myapp/
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF  NODE NAME
bash    3474 vagrant  cwd    DIR  253,1       53 25264 /opt/myapp
less    3509 vagrant  cwd    DIR  253,1       53 25264 /opt/myapp

```

从中我们可以看到两个进程打开了`/opt/myapp`目录。我们可以限制`lsof`的输出的另一种方法是指定`+D`（目录内容）标志，后跟一个目录。这个标志将告诉`lsof`查找该目录及其以下的任何打开项目。

例如，我们看到当使用`lsof`针对配置文件时，`less`进程已经打开了它。我们还可以看到，当用于`/opt/myapp/`目录时，两个进程打开了该目录。

我们可以使用`+D`标志一次查看所有这些项目：

```
# lsof +D /opt/myapp/
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF      NODE NAME
bash    3474 vagrant  cwd    DIR  253,1       53     25264 /opt/myapp
less    3509 vagrant  cwd    DIR  253,1       53     25264 /opt/myapp
less    3509 vagrant    4r   REG  253,1       45 201948450 /opt/myapp/conf/config.yml

```

这也会显示位于`/opt/myapp`目录下的任何其他项目。由于我们要检查应用程序是否有另一个实例正在运行，让我们看一下前面的`lsof`输出，并看看可以学到什么：

```
COMMAND  PID    USER   FD   TYPE DEVICE SIZE/OFF      NODE NAME
bash    3474 vagrant  cwd    DIR  253,1       53     25264 /opt/myapp

```

第一个打开的项目显示了一个`bash`进程，以`vagrant`用户身份运行，具有当前工作目录的文件描述符。这一行很可能是我们自己的`bash`进程，目前正在`/opt/myapp`目录中，当前正在执行`/opt/myapp/conf/config.yml`文件上的`less`命令。

我们可以通过使用`ps`命令并`grep`字符串`3474`来检查这一点，`bash`命令的进程 ID：

```
# ps -elf | grep 3474
0 S vagrant   3474  3473  0  80   0 - 28857 wait   20:09 pts/1    00:00:00 -bash
0 S vagrant   3509  3474  0  80   0 - 27562 n_tty_ 20:14 pts/1    00:00:00 less conf/config.yml
0 S root      3576  2978  0  80   0 - 28160 pipe_w 21:08 pts/0    00:00:00 grep --color=auto 3474

```

在这种情况下，我选择使用`grep`命令，因为我们还将能够看到引用进程 ID`3474`的任何子进程。也可以通过运行以下命令来执行相同的操作，而不使用`grep`命令：

```
# ps -lp 3474 --ppid 3474
F S   UID   PID  PPID  C PRI  NI ADDR SZ WCHAN  TTY          TIME CMD
0 S  1000  3474  3473  0  80   0 - 28857 wait   pts/1    00:00:00 bash
0 S  1000  3509  3474  0  80   0 - 27562 n_tty_ pts/1    00:00:00 less

```

总的来说，两种方法都会产生相同的结果；然而，第一种方法更容易记住。

如果我们查看进程列表输出，我们可以看到`bash`命令实际上与我们的 shell 相关，因为它的子进程是我们知道在另一个窗口中正在运行的`less`命令。

我们还可以看到`less`命令的进程 ID：`3509`。相同的进程 ID 在`lsof`输出中显示了`less`命令：

```
less    3509 vagrant  cwd    DIR  253,1       53     25264 /opt/myapp
less    3509 vagrant    4r   REG  253,1       45 201948450 /opt/myapp/conf/config.yml

```

由于输出只显示我们自己的进程，可以安全地假设在后台没有运行先前的应用程序实例。

# 了解更多关于应用程序的信息

我们现在知道问题不是另一个此应用程序实例正在运行。在这一点上，我们应该尝试并识别更多关于这个应用程序以及它在做什么的信息。

在尝试查找有关此应用程序的更多信息时，首先要做的是查看应用程序的文件类型。我们可以使用`file`命令来做到这一点：

```
$ file bin/application 
bin/application: setuid ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=0xbc4685b44eb120ff2252e21bd735933d51409ffa, not stripped

```

`file`命令是一个非常有用的命令，因为这个命令将识别指定文件的文件类型。在前面的例子中，我们可以看到"`application`"文件是一个已编译的二进制文件。我们可以看到它是由这个特定的输出编译的：`ELF 64 位 LSB 可执行文件`。

这行还告诉我们应用程序是作为 64 位应用程序编译的。这很有趣，因为 64 位应用程序和 32 位应用程序之间有很多区别。一个非常常见的情况是由于 64 位应用程序可以消耗的资源量；32 位应用程序通常比 64 位版本受限得多。

另一个常见问题是尝试在 32 位内核上执行 64 位应用程序。我们尚未验证是否在 64 位内核上运行；如果我们试图在 32 位内核上运行 64 位可执行文件，我们肯定会收到一些错误。

尝试在 32 位内核上执行 64 位应用程序时出现的错误类型非常具体，不太可能是我们问题的原因。尽管这不太可能是原因，我们可以使用`uname -a`命令来检查内核是否为 64 位内核：

```
$ uname -a
Linux blog.example.com 3.10.0-123.el7.x86_64 #1 SMP Mon Jun 30 12:09:22 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux

```

从`uname -a`命令的输出中，我们可以看到内核实际上是 64 位内核，因为存在这个字符串"`x86_64`"。

## 使用 strace 跟踪应用程序

由于我们知道应用程序是编译后的二进制文件，没有源代码，这使得在应用程序内部阅读代码相当困难。然而，我们可以追踪应用程序执行的系统调用，以查看是否能找到任何关于它为何无法启动的信息。

### 什么是系统调用？

**系统调用**是应用程序和内核之间的主要接口。简而言之，系统调用是请求内核执行操作的方法。

大多数应用程序不需要担心系统调用，因为系统调用通常由低级库（如 GNU C 库）调用。虽然程序员不需要担心系统调用，但重要的是要知道应用程序执行的每个操作都归结为某种系统调用。

这很重要，因为我们可以追踪这些系统调用来确定应用程序到底在做什么。就像我们使用`tcpdump`来追踪系统上的网络流量一样，我们可以使用一个叫做`strace`的命令来追踪进程的系统调用。

为了感受`strace`，让我们使用`strace`对之前的`exitcodes.sh`脚本进行系统调用跟踪。为此，我们将运行`strace`命令，然后是`exitcodes.sh`脚本。

执行时，`strace`命令将启动，然后执行`exitcodes.sh`脚本。在`exitcodes.sh`脚本运行时，`strace`命令将打印`exitcodes.sh`脚本中提供的每个系统调用和参数：

```
$ strace /var/tmp/exitcodes.sh 
execve("/var/tmp/exitcodes.sh", ["/var/tmp/exitcodes.sh"], [/* 26 vars */]) = 0
brk(0)                                  = 0x261a000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f890bd12000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=24646, ...}) = 0
mmap(NULL, 24646, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f890bd0b000
close(3)                                = 0
open("/lib64/libtinfo.so.5", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0@\316\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=174520, ...}) = 0
mmap(NULL, 2268928, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f890b8c9000
mprotect(0x7f890b8ee000, 2097152, PROT_NONE) = 0
mmap(0x7f890baee000, 20480, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x25000) = 0x7f890baee000
close(3)                                = 0
open("/lib64/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\320\16\0\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=19512, ...}) = 0

```

这只是`strace`的输出的一小部分。完整的输出实际上有好几页长。然而，`exitcodes.sh`脚本并不是很长。事实上，它只是一个简单的三行脚本：

```
$ cat /var/tmp/exitcodes.sh 
#!/bin/bash
touch /some/directory/that/doesnt/exist/file.txt
echo "It works"

```

这个脚本很好地展示了高级编程语言（如 bash）提供了多少重要的功能。现在我们知道`exitcodes.sh`脚本的作用，让我们来看一下它执行的一些系统调用。

我们将从前八行开始：

```
execve("/var/tmp/exitcodes.sh", ["/var/tmp/exitcodes.sh"], [/* 26 vars */]) = 0
brk(0)                                  = 0x261a000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f890bd12000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=24646, ...}) = 0
mmap(NULL, 24646, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f890bd0b000
close(3)                                = 0

```

由于系统调用非常广泛，有些系统调用很难理解。我们将把重点放在常见且较容易理解的系统调用上。

我们将要检查的第一个系统调用是`access()`系统调用：

```
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)

```

大多数系统调用都有一个名字，大致解释它执行的功能。`access()`系统调用也不例外，因为这个系统调用用于检查调用它的应用程序是否有足够的权限打开指定的文件。在前面的例子中，指定的文件是`/etc/ld.so.preload`。

关于`strace`的一个有趣的事情是它不仅显示系统调用，还显示返回值。在我们前面的示例中，`access()`系统调用收到了一个返回值`-1`，这是错误的典型值。当返回值是错误时，`strace`还会提供错误字符串。在这种情况下，`access()`调用收到了错误`-1 ENOENT (No such file or directory)`。

前面的错误非常容易理解，因为似乎文件`/etc/ld.so.preload`根本不存在。

下一个系统调用是一个经常见到的系统调用；它就是`open()`系统调用：

```
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3

```

`open()`系统调用执行了它所说的内容，它用于打开（或创建并打开）文件或设备。从前面的示例中，我们可以看到指定的文件是`/etc/ld.so.cache`文件。我们还可以看到传递给这个系统调用的参数之一是"`O_RDONLY`"。这个参数告诉`open()`调用以只读模式打开文件。

即使我们不知道`O_RDONLY`参数告诉打开命令以只读模式打开文件，这个名字几乎是自我描述的。对于那些不够自我描述的系统调用，可以通过相当快速的谷歌搜索找到相关信息，因为系统调用都有很好的文档记录：

```
fstat(3, {st_mode=S_IFREG|0644, st_size=24646, ...}) = 0

```

下一个要看的系统调用是`fstat()`系统调用。这个系统调用将获取文件的状态。这个系统调用提供的信息包括诸如 inode 号、用户所有权和文件大小等内容。单独看，`fstat()`系统调用可能看起来并不重要，但当我们看下一个系统调用`mmap()`时，它提供的信息可能就很重要了。

```
mmap(NULL, 24646, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f890bd0b000

```

这个系统调用可以用来将文件映射或取消映射到内存中。如果我们看一下`fstat()`行和`mmap()`行，我们会看到两个相符的数字。`fstat()`行有`st_size=24646`，这是提供给`mmap()`的第二个参数。

即使不知道这些系统调用的细节，也很容易得出这样的假设，即`mmap()`系统调用将文件从`fstat()`调用映射到内存中。

前面示例中的最后一个系统调用非常容易理解：

```
close(3)                                = 0

```

`close()`系统调用只是关闭打开的文件或设备。考虑到我们之前打开了文件`/etc/ld.so.cache`，这个`close()`系统调用被用来关闭那个文件是很合理的。在我们回到调试应用程序之前，让我们快速看一下最后四行放在一起的内容：

```
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=24646, ...}) = 0
mmap(NULL, 24646, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f890bd0b000
close(3) 

```

当我们看这四个系统调用时，我们可以开始看到一个模式。`open()`调用用于打开`/etc/ld.so.cache`文件，并返回值为`3`。`fstat()`命令提供了`3`作为输入，并得到了`st_size=24646`作为输出。`mmap()`函数被给予了`24646`和`3`作为输入，`close()`函数被提供了`3`作为输入。

考虑到`open()`调用的输出是`3`，并且这个值在这四个系统调用中被多次使用，可以安全地得出结论，即这个数字`3`是打开文件`/etc/ld.so.cache`的文件描述符号。有了这个结论，我们也可以相当肯定，前面的四个系统调用执行了打开文件`/etc/ld.so.cache`、确定文件大小、将文件映射到内存，然后关闭文件描述符的操作。

正如你所看到的，仅仅通过四个简单的系统调用就得到了相当多的信息。让我们将刚学到的知识付诸实践，使用`strace`来跟踪应用程序进程。

## 使用 strace 来确定应用程序为什么无法启动

早些时候，当我们运行`strace`时，我们只是提供了一个要执行的命令。这是你可以调用`strace`的一种方式，但如果进程已经在运行，你该怎么办呢？嗯，`strace`也可以跟踪正在运行的进程。

在跟踪现有进程时，我们可以使用`-p`（进程）标志加上要跟踪的进程 ID 来启动`strace`。这会导致`strace`绑定到该进程并开始跟踪它。为了跟踪我们的应用程序启动，我们将使用这种方法。

为了做到这一点，我们将在后台执行`start.sh`脚本，然后对`start.sh`脚本的进程 ID 运行`strace`：

```
$ ./start.sh &
[1] 3353

```

通过在命令行的末尾添加&，我们告诉启动脚本在后台运行。输出提供了正在运行的脚本的进程 ID，`3353`。然而，在另一个窗口中作为 root 用户，我们可以使用以下命令对该进程进行跟踪：

```
# strace -o /var/tmp/app.out -f -p 3353
Process 3353 attached
Process 3360 attached

```

前面的命令比只有`-p`和进程 ID 多了一些选项。我们还添加了`-o /var/tmp/app.out`参数。这个选项告诉`strace`将跟踪的数据保存到输出文件`/var/tmp/app.out`中。我们之前运行的`strace`提供了相当多的输出；通过指定数据应该写入文件，数据将更容易搜索。

我们添加的另一个新选项是`-f`；这个参数告诉`strace`跟踪子进程。由于启动脚本启动了应用程序，应用程序本身被认为是启动脚本的子进程。在前面的例子中，我们可以看到`strace`附加到了两个进程。我们可以假设第二个进程收到了进程 ID`3360`，这一点很重要，因为在浏览跟踪输出时我们需要引用该进程 ID：

```
# less /var/tmp/app.out

```

让我们开始阅读`strace`输出并尝试识别发生了什么。在浏览输出时，我们将限制它只包括对识别我们问题有用的部分：

```
3360  execve("/opt/myapp/bin/application", ["/opt/myapp/bin/application", "--deamon", "--config", "/opt/myapp/conf/config.yml"], [/* 28 vars */]) = 0

```

看起来有趣的第一个系统调用是`execve()`系统调用。这个特定的`execve()`调用似乎是在执行`/opt/myapp/bin/application`二进制文件。

需要指出的一个重要事项是，通过这个输出，我们可以看到系统调用之前有一个数字。这个数字`3360`是执行系统调用的进程 ID。只有在 strace 命令跟踪多个进程时才会显示进程 ID。

```
The next group of system calls that seem important are the following:
3360  open("/opt/myapp/conf/config.yml", O_RDONLY) = 3
3360  fstat(3, {st_mode=S_IFREG|0600, st_size=45, ...}) = 0
3360  fstat(3, {st_mode=S_IFREG|0600, st_size=45, ...}) = 0
3360  mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd0528df000
3360  read(3, "port: 25\ndebug: True\nlogdir: /op"..., 4096) = 45
3360  read(3, "", 4096)                 = 0
3360  read(3, "", 4096)                 = 0

```

从前面的一组中，我们可以看到应用程序以只读方式打开了`config.yml`文件，并且没有收到错误。我们还可以看到`read()`系统调用（似乎是从文件描述符 3 读取）正在读取`config.yml`文件。

```
3360  close(3)                          = 0

```

文件的更下方显示，使用`close()`系统调用关闭了这个文件描述符。这个信息很有用，因为它告诉我们我们能够读取`config.yml`文件，而我们的问题与配置文件的权限无关：

```
3360  open("/opt/myapp/logs/debug.out", O_WRONLY|O_CREAT|O_APPEND, 0666) = 3
3360  lseek(3, 0, SEEK_END)             = 1711
3360  fstat(3, {st_mode=S_IFREG|0664, st_size=1711, ...}) = 0
3360  fstat(3, {st_mode=S_IFREG|0664, st_size=1711, ...}) = 0
3360  mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fd0528df000
3360  write(1, "- - - - - - - - - - - - - - - - "..., 52) = 52

```

如果我们继续，我们还可以看到我们的配置也在生效，因为进程已经使用`open()`调用打开了`debug.out`文件进行写入，并使用`write()`调用写入了它。

对于有许多日志文件的应用程序，上述的系统调用等可以用于识别可能不太明显的日志消息。

在浏览系统调用时，您可以大致了解生成消息的上下文以及可能的原因。这个上下文可能会根据问题的严重程度非常有用。

```
3360  socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 4
3360  bind(4, {sa_family=AF_INET, sin_port=htons(25), sin_addr=inet_addr("0.0.0.0")}, 16) = -1 EADDRINUSE (Address already in use)
3360  open("/dev/null", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 5
3360  fstat(5, {st_mode=S_IFCHR|0666, st_rdev=makedev(1, 3), ...}) = 0
3360  write(1, "Starting service: [Failed]\n", 27) = 27
3360  write(3, "Configuration file processed\r\n--"..., 86) = 86
3360  close(3)                          = 0

```

说到上下文，前面的系统调用明确解释了我们的问题，一个系统调用。虽然`strace`文件包含了许多返回错误的系统调用，但其中大部分都像下面这样：

```
3360  stat("/usr/lib64/python2.7/encodings/ascii", 0x7fff8ef0d670) = -1 ENOENT (No such file or directory)

```

这是相当常见的，因为它只是意味着进程尝试访问一个不存在的文件。然而，在跟踪文件中，有一个错误比其他的更显眼：

```
3360  bind(4, {sa_family=AF_INET, sin_port=htons(25), sin_addr=inet_addr("0.0.0.0")}, 16) = -1 EADDRINUSE (Address already in use)

```

前面的系统调用`bind()`是一个绑定套接字的系统调用。前面的例子似乎是在绑定网络套接字。如果我们回想一下我们的配置文件，我们知道指定了端口`25`：

```
# cat /opt/myapp/conf/config.yml 
port: 25

```

在系统调用中，我们可以看到字符串`sin_port=htons(25)`，这可能意味着这个绑定系统调用正在尝试绑定到端口`25`。从提供的返回值中，我们可以看到`bind()`调用收到了一个错误。该错误的消息表明“地址已经在使用”。

由于我们知道应用程序配置为以某种方式利用端口`25`，并且我们可以看到一个`bind()`系统调用，因此可以推断出这个应用程序可能之所以没有启动，只是因为端口`25`已经被另一个进程使用，这在这一点上是我们的新假设。

# 解决冲突

正如您在网络章节中学到的，我们可以通过快速的`netstat`命令来验证进程是否使用端口`25`：

```
# netstat -nap | grep :25
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1588/master 
tcp6       0      0 ::1:25                  :::*                    LISTEN      1588/master

```

当我们以 root 用户身份运行`netstat`并添加`-p`标志时，该命令将包括每个 LISTEN-ing 套接字的进程 ID 和进程名称。从中，我们可以看到端口`25`实际上正在被使用，而进程 1588 是正在监听的进程。

为了更好地了解这个进程是什么，我们可以再次利用`ps`命令：

```
# ps -elf | grep 1588
5 S root      1588     1  0  80   0 - 22924 ep_pol 13:53 ?        00:00:00 /usr/libexec/postfix/master -w
4 S postfix   1616  1588  0  80   0 - 22967 ep_pol 13:53 ?        00:00:00 qmgr -l -t unix -u
4 S postfix   3504  1588  0  80   0 - 22950 ep_pol 20:36 ?        00:00:00 pickup -l -t unix -u

```

看起来`postfix`服务是在端口`25`上监听，这并不奇怪，因为这个端口通常用于 SMTP 通信，而 postfix 是一个电子邮件服务。

现在的问题是，后缀应该在这个端口上监听，还是应用程序？不幸的是，对于这个问题没有简单的答案，因为它确实取决于系统和它们正在做什么。

为了这个练习，我们将假设答案是自定义应用程序应该使用端口`25`，而后缀不应该运行。

为了阻止后缀在端口`25`上监听，我们将首先使用`systemctl`命令停止后缀：

```
 # systemctl stop postfix

```

这将停止后缀服务，下一个命令将禁止它在下次重新启动时再次启动：

```
# systemctl disable postfix
rm '/etc/systemd/system/multi-user.target.wants/postfix.service'

```

禁用后缀服务是解决此问题的重要步骤。目前，我们认为问题是由自定义应用程序和后缀之间的端口冲突引起的。如果我们不禁用后缀服务，下次系统重新启动时它将被重新启动。这将阻止自定义应用程序的启动。

虽然这可能看起来很基础，但我想强调这一步的重要性，因为在许多情况下，我曾见过一个问题反复发生，只是因为第一次解决它的人没有禁用一个服务。

如果我们运行`systemctl`状态命令，我们现在可以看到后缀服务已停止并禁用：

```
# systemctl status postfix
postfix.service - Postfix Mail Transport Agent
 Loaded: loaded (/usr/lib/systemd/system/postfix.service; disabled)
 Active: inactive (dead)

Jun 09 04:05:42 blog.example.com systemd[1]: Starting Postfix Mail Transport Agent...
Jun 09 04:05:43 blog.example.com postfix/master[1588]: daemon started -- version 2.10.1, configuration /etc/postfix
Jun 09 04:05:43 blog.example.com systemd[1]: Started Postfix Mail Transport Agent.
Jun 09 21:14:14 blog.example.com systemd[1]: Stopping Postfix Mail Transport Agent...
Jun 09 21:14:14 blog.example.com systemd[1]: Stopped Postfix Mail Transport Agent.

```

通过停止`postfix`服务，我们现在可以再次启动应用程序，看看问题是否已解决。

```
$ ./start.sh
Initializing with configuration file /opt/myapp/conf/config.yml
- - - - - - - - - - - - - - - - - - - - - - - - - -
Starting service: [Success]
- - - - - - - - - - - - - - - - - - - - - - - - - 
Proccessed 5 messages
Proccessed 5 messages
Proccessed 5 messages

```

看起来问题实际上是通过停止`postfix`服务解决的。我们可以通过启动过程中打印的`[Success]`消息来看到这一点。如果我们再次运行`lsof`命令，也可以看到这一点：

```
# lsof +D /opt/myapp/
COMMAND    PID    USER   FD   TYPE DEVICE SIZE/OFF      NODE NAME
bash      3332 vagrant  cwd    DIR  253,1       53     25264 /opt/myapp
start.sh  3585 vagrant  cwd    DIR  253,1       53     25264 /opt/myapp
start.sh  3585 vagrant  255r   REG  253,1      111     25304 /opt/myapp/start.sh
applicati 3588    root  cwd    DIR  253,1       53     25264 /opt/myapp
applicati 3588    root  txt    REG  253,1    36196  68112463 /opt/myapp/bin/application
applicati 3588    root    3w   REG  253,1     1797 134803515 /opt/myapp/logs/debug.out

```

现在应用程序正在运行，我们可以看到几个进程在`/opt/myapp`目录中有打开的项目。我们还可以看到其中一个进程是带有进程 ID`3588`的应用程序命令。为了更好地了解应用程序正在做什么，我们可以再次运行`lsof`，但这次我们只搜索进程 ID`3588`打开的文件：

```
# lsof -p 3588
COMMAND    PID USER   FD   TYPE DEVICE  SIZE/OFF      NODE NAME
applicati 3588 root  cwd    DIR  253,1        53     25264 /opt/myapp
applicati 3588 root  rtd    DIR  253,1      4096       128 /
applicati 3588 root  txt    REG  253,1     36196  68112463 /opt/myapp/bin/application
applicati 3588 root  mem    REG  253,1    160240 134296681 /usr/lib64/ld-2.17.so
applicati 3588 root    0u   CHR  136,2       0t0         5 /dev/pts/2
applicati 3588 root    1u   CHR  136,2       0t0         5 /dev/pts/2
applicati 3588 root    2u   CHR  136,2       0t0         5 /dev/pts/2
applicati 3588 root    3w   REG  253,1      1797 134803515 /opt/myapp/logs/debug.out
applicati 3588 root    4u  sock    0,6       0t0     38488 protocol: TCP

```

`-p`（进程）标志将`lsof`输出过滤到特定进程。在这种情况下，我们将输出限制为刚刚启动的自定义应用程序。

```
applicati 3588 root    4u  sock    0,6       0t0     38488 protocol: TCP

```

在最后一行中，我们可以看到应用程序有一个 TCP 套接字打开。根据应用程序的状态消息和`lsof`的结果，可以非常肯定地说应用程序已经启动并且启动正确。

# 总结

我们遇到了一个应用程序问题，并使用了常见的 Linux 工具，如`lsof`和`strace`来找到根本原因，即端口冲突。更重要的是，我们在没有关于应用程序或其尝试执行的任务的先前知识的情况下做到了这一点。

通过本章的示例，我们可以很容易地看到，拥有基本 Linux 工具的访问权限和知识，再加上对故障排除过程的理解，可以使您能够解决几乎任何问题，无论是应用程序问题还是系统问题。

在下一章中，我们将研究 Linux 用户和内核限制，以及它们有时可能会引起问题。
