# 第七章。高级 Procmail

现在我们已经掌握了 Procmail 的基础知识，我们可以继续开始组建一个更完整的邮件处理系统。本章中的高级技术只有在您需要非常专业的邮件处理时才需要，并且不需要为设置基本电子邮件服务器而需要。您可能希望跳过本章，在服务器完全配置和运行后再返回。

在本章中，我们将使用一些更高级的 Procmail 功能。本章将涵盖：

+   投递和非投递配方之间的区别

+   在高级配方中使用变量、替换和伪变量

+   锁定和使用各种标志来控制执行

+   如何应用条件来测试消息的各个部分

+   高级操作，将消息转发、保存或传递给外部程序进行处理

+   正则表达式简介

+   使用 Procmail 宏简化电子邮件头部分析

+   详细分析一些高级配方，包括一些示例配方

到本章结束时，您应该具有一套有用的例程工具箱，用于组合您自己的一套 Procmail 配方，并控制您的邮件。

# 投递和非投递配方

到目前为止，我们只涵盖了那些要么最终将邮件传递给程序或文件，要么将消息转发给另一个邮件用户的配方。还有另一个选项可用，引用 Procmail 文档：

> 有两种类型的配方——投递和非投递配方。如果找到匹配的投递配方，Procmail 会认为邮件（你猜对了）已经投递，并在成功执行配方的动作行后停止处理`.procmailrc`文件。如果找到匹配的非投递配方，此配方的动作行执行后，`.procmailrc`文件的处理将继续。

## 非投递示例

我们在上一章中介绍了一个示例，旨在备份邮件项，以防正在测试的配方删除所有邮件。这是一个非常有用的非投递配方示例，可以在 Procmail 手册页`procmailex`中找到。

如果您对 Procmail 相当陌生，并计划进行一些实验，通常有某种安全网会有所帮助。在插入前面提到的两个配方之前，所有其他配方将确保最后到达的 32 封邮件始终被保留。为了使其按预期工作，我们必须在插入这两个配方之前在`$MAILDIR`中创建一个名为`backup`的目录：

```
:0 c
backup
:0 ic
cd backup && rm -f dummy `ls -t msg.* | sed -e 1,32d`

```

第二个配方使用了 Procmail 的几个特性，我们将在本章的后续部分中更详细地探讨。

如果我们按照这个配方一步一步地工作，最终会得到一个有用的存档实用程序，记录最后接收的 32 封邮件，并允许我们手动恢复邮件，如果我们创建了一个最终销毁邮件而不是存储它的配方。在繁忙的邮件服务器上，增加这个数字可能是明智的，以保留更大的消息存档。

第一个配方通过将邮件的副本或克隆传递到`backup`目录来执行简单的备份操作：

```
:0 c
backup

```

在添加第二个配方之前，在`.procmailrc`文件中创建上述配方，并给自己发送几封邮件。我们可以看到每个邮件项都存储在备份目录中（前提是它存在并具有正确的权限）。

第二个配方同样简单，但使用了 Linux 系统命令的一些更复杂的特性，以删除除`backup`目录中最近的 32 个邮件项之外的所有邮件项。

```
:0 ic
| cd backup && rm -f dummy `ls -t msg.* | sed -e 1,32d`

```

让我们看看这个配方是如何工作的。首先，我们将看到规则标志及其含义：

| 标志 | 含义 |
| --- | --- |
| `i` | 忽略后续管道命令的返回代码 |
| `c` | 克隆或复制传入数据，以便不影响原始数据 |

`|`指示 Procmail 将匹配的数据传递给以下管道命令。每个命令执行特定的动作。

| 命令 | 动作 |
| --- | --- |
| `cd backup` | 进入`backup`目录。 |
| `ls -t msg.*` | 获取以`msg`开头的文件列表，并按时间顺序排序。 |
| `sed -e 1,32d` | 删除除最后 32 行之外的所有内容，即最近的 32 封邮件。 |
| `rm -f dummy...` | 参数`dummy`是为了防止出现没有要删除的文件时的错误消息，然后`rm`命令继续删除`sed`过滤器列出的文件。 |

这两个配方是无条件配方的例子，它们在每封传入的邮件消息上运行。没有条件行，即以星号（*）开头的行，这意味着这些配方是无条件的。由于这两个配方都包括配方中的`c`标志，它们也被定义为非投递配方。

一旦我们收集了一些 Procmail 配方，我们会发现处理配方的顺序可能很重要。通过正确设置处理顺序，我们可以提高性能，减少处理传入邮件所需的时间。我们还可以确保更重要的消息适用于重要消息，而不是适用于用于批量消息的更一般的规则。

一个典型的情景可能是按以下顺序应用规则：

1.  首先处理守护程序或服务器消息。

1.  邮件列表应尽早处理，但在服务器消息之后，因为我们希望我们的服务首先得到处理。

1.  应用`kill file`来阻止任何已知的垃圾邮件发送者。

1.  在我们处理邮件列表之前，请不要发送度假回复，以防止向邮件列表发送令人讨厌的度假回复。

1.  保存私人消息。

1.  检查**未经请求的大量电子邮件**（**UBE**）-垃圾邮件。这避免了在已知有效的电子邮件上处理垃圾邮件检查的高开销。

# Formail

**Formail**是一个外部实用程序（来自 Procmail），几乎总是在安装了 Procmail 的系统上可用。它的功能是处理邮件消息并从消息头中提取信息。它充当过滤器，可用于强制将邮件格式化为适合存储在 Linux 邮件系统中的格式。它还可以执行许多其他有用的功能，如'From'转义、生成自动回复头、简单的头提取或拆分邮箱/摘要/文章文件。

需要使用标准输入提供输入数据邮件/邮箱/文章内容。因此，`formail`非常适合在管道命令链中使用。输出数据在标准输出上提供。

在本章中，我们不打算深入讨论`formail`的微妙之处，但由于它是一个有用的工具，我们将在一些示例中提到它的一些功能。更多信息可以从系统手册页面中获得。

# 高级配方分析

这里有一个更复杂的配方，实现了一种度假服务，通知发件人你不在，无法回复电子邮件。乍一看，这可能是一个简单的非投递配方，向所有收到的消息发送一条消息。然而，这并不理想，因为有些人可能最终会收到多个投递确认消息，而你也可能会向无法理解你善意回复的系统实用程序发送消息。

该示例基于 Procmail `procmailex`手册页面上的“度假示例”。

`vacation.cache`文件由 Formail 维护。它通过提取发件人的姓名并将其插入`vacation.cache`文件来维护度假数据库。这确保它始终包含最新的名称。文件的大小限制为最大约 8192 字节。如果发件人的姓名是新的，将发送自动回复。

以下配方实现了度假自动回复：

```
SHELL=/bin/sh # for other shells, this might need adjustment
:0 Whc: vacation.lock
# Perform a quick check to see if the mail was addressed to us
$TO_:.*\<$\LOGNAME\>
# Filter out the mail senders we don't want to send replies to - Ever
* !^FROM_DAEMON
# Make sure that we do not create an endless loop that keeps
# replying to the reply by checking to see if we have already processed
# this message and inserted a loop detection header
* !^X-Loop: your@own.mail.address
| formail -rD 8192 vacation.cache
:0 ehc
# We are pretty certain it's OK to send a reply to the sender of this message
| (
formail -rA"Precedence: junk" \
-A"X-Loop: your@own.mail.address" ; \
echo "Hi, Your message was delivered to my mailbox,"; \
echo "but I won't be back until Monday."; \
echo "-- "; cat $HOME/.signature \
) | $SENDMAIL -oi -t

```

我们将在本节末尾回到这个配方，并通过使用我们在 Procmail 中学到的一些东西，创建一个稍微更新的版本。目前，这个示例将作为一个参考，以便理解我们在下面对一般配方结构进行分解时探讨的一些概念。

## 添加注释

对我们的规则和配方进行文档编写或添加注释始终是一项重要任务。所有注释都以`#`字符开头，并持续到行尾。在大多数情况下，将注释放在行首或在我们希望记录的单行之后的一两个制表符处是有用的。

然而，有一个规则文件的部分，注释*必须*单独占一行，那就是*条件*部分。

```
# Here is a full line comment
MAILDIR=${HOME}/Maildir # This comment spans multiple
# lines for clarity.
:0: # Comment OK here
* condition # BAD comment. NOT allowed.
# Old versions of Procmail don't understand this.
* condition
{ # Comment OK
# Comment OK
do_something # Comment OK
}

```

## 变量赋值

为了跟踪设置、测试结果、默认值等信息，我们可以将这些信息存储在变量中。赋值操作很简单，遵循与其他 Linux 脚本语言相同的格式。基本格式是`VARIABLENAME=VALUE。`

### 提示

变量名称中不能有空格。如果在分配的值中有空格，则整个变量应存储在双引号之间。

访问变量的正确方式是将`VARIABLENAME`括在大括号`{}`中，并在所有内容前加上美元（$）符号。在其他赋值中使用变量是完全可以接受的。以下是一些示例：

```
MAILDIR=${HOME}/Maildir # Set the value of the MAILDIR
LOGFILE=${MAILDIR}/log # Store logfiles in the MAILDIR

```

请注意在前面的示例中，`${HOME}`获取了 shell 环境设置的值，就像在启动进程时设置的那样。

谨慎使用变量及其命名可以使配方更容易阅读和维护。

### 执行替换

有时，将文字元素替换为仅在运行时可以计算或评估的变量将是必要或有用的。Procmail 允许作者在大多数地方用变量替换或命令替换替换大多数文字元素。使用变量的最简单方式是使用`$varname`格式，这是许多脚本语言共有的。

| 变量/命令 | 替换 |
| --- | --- |
| `$VAR` | 在配方中，无论何时出现`$VAR`，都用该变量持有的值替换它。 |
| `${VAR}iable *` | 当我们需要将变量与文字连接在一起时，使用`{}`来强制指定名称为`${VAR}`而不是`$VARiable`。 |

### 注意

如果需要将变量与固定文本或值组合，`{}`元素允许绝对定义变量名称。请注意，除非包含`$`修饰符，否则这不会发生在条件行中。

#### 使用默认值赋值变量

Procmail 借用了一些标准的 shell 语法来初始化变量。

如果我们希望能够为变量分配默认值，以便在变量未设置或由于某种原因无法计算时使用，可以使用或`:-`分隔符。如果希望在变量已设置或非空时应用替代值，则使用`+`或`:+`分隔符。

| 分隔符 | 操作 |
| --- | --- |
| `${VAR:-value}` | 如果`VAR`未设置或为空，则替换为`value`的扩展；否则，替换为`VAR`的值。 |
| `${VAR-value}` | 如果`VAR`未设置，则替换为`value`的扩展；否则，替换为`VAR`的值。 |
| `${VAR:+value}` | 如果`VAR`已设置或非空，则替换为`value`的扩展；否则，替换为`VAR`的值。 |
| `${VAR+value}` | 如果`VAR`已设置，则替换为`value`的扩展；否则，替换为`VAR`的值。 |

以下是一些示例：

```
VAR = "" # Set VAR to null
VAR = ${VAR:-"val1"} # VAR = "val1"
VAR = ""
VAR = ${VAR-"val2"} # VAR = ""
VAR = ""
VAR = ${VAR:+"val3"} # VAR = ""
VAR = ""
VAR = ${VAR+"val4"} # VAR = "val4"
VAR = "val"
VAR = ${VAR:+"val3"} # VAR = "val3"
VAR = "val"
VAR = ${VAR+"val4"} # VAR = "val4"
VAR # unset VAR
VAR = ${VAR:-"val1"} # VAR = "val1"
VAR
VAR = ${VAR-"val2"} # VAR = "val2"
VAR
VAR = ${VAR:+"val3"} # no action
VAR
VAR = ${VAR+"val4"} # no action

```

#### 将命令输出赋值给变量

可以通过使用（反引号）`操作符将命令的输出赋值给变量——反引号（`）是 ASCII 值 96，而不是普通的撇号（'），其 ASCII 值为 39。

```
  `cmd1 | cmd2`

```

此示例将将两个反引号之间的管道输出分配给相应的变量或内联到代码中。

### 伪变量

有许多特殊变量或伪变量直接由 Procmail 分配。更改其中一些值实际上可以改变 Procmail 的操作方式。

#### 邮箱变量

以下变量由 Procmail 用于确定它将存储任何已交付邮件的位置。

| 名称 | 操作 |
| --- | --- |
| `MAILDIR` | `MAILDIR`的默认值取自`$HOME`环境变量的值。它也是 Procmail 在执行期间的当前工作目录的值。除非输出文件名包括路径组件，否则它们将被创建在这个默认目录中。 |
| `MSGPREFIX` | 当我们希望将文件按顺序写入目录时使用此选项。`MSGPREFIX`将作为前缀添加到使用此选项创建的文件的名称中。默认前缀是`msg.`，因此文件将被命名为`msg.xyz`。将文件传递到`maildir`或`MH`目录时不使用此选项。 |
| `DEFAULT` | 这是系统上默认邮件存储区的位置。通常情况下，我们不会修改这个变量。 |
| `ORGMAIL` | 这用作`DEFAULT`无法使用的情况下的灾难恢复位置。这绝对不应该被修改。 |

#### 程序变量

Procmail 在编译时有合理的默认值。大多数情况下，这些值不需要更改。

| 名称 | 操作 |
| --- | --- |
| `SHELL` | 这是一个标准的环境变量，指定了 Procmail 需要调用子进程的 shell 环境。分配给它的值应该是与 Bourne shell 兼容的，例如`/bin/sh`。 |
| `SHELLFLAGS` | 这指定了在启动`SHELL`时应传递的任何可选标志。 |
| `SENDMAIL` | 这指示 Procmail 在哪里找到用于将邮件发送给其他用户的`sendmail`程序。（通常不应该被玩弄）。 |
| `SENDMAILFLAGS` | 与`SHELLFLAGS`一样，指定在执行`SENDMAIL`程序时应传递的任何标志或命令行参数。 |

#### 系统交互变量

在执行食谱期间，Procmail 可能需要运行外部命令，处理错误或创建文件。这些变量控制了 Procmail 与 shell 的交互方式。

| 名称 | 操作 |
| --- | --- |
| `UMASK` | 创建任何文件时使用的文件权限模式。有关详细信息，请参阅`man umask`。 |
| `SHELLMETAS` | 在执行之前，shell 管道将与`SHELLMETAS`的内容进行比较。如果在管道命令中找到了来自`SHELLMETAS`的任何字符，则认为该命令对于 Procmail 来说太复杂，无法自行管理，并且会生成一个子 shell 进程。如果我们知道特定的管道始终对 Procmail 来说足够简单，但包含在`SHELLMETAS`中的字符，我们可以在处理管道时临时将空字符串分配给`SHELLMETAS`，然后恢复`SHELLMETAS`。这将避免生成子 shell 的开销。 |
| `TRAP` | 在 Procmail 执行结束时，我们可以分配一个代码段来执行。例如，可以用它来删除在执行食谱期间创建的临时文件。`TEMPORARY=$HOME/tmp/pmail.$$``TRAP="/bin/rm -f $TEMPORARY"` |
| `EXITCODE` | 当 Procmail 退出时，此值将返回给启动 Procmail 的进程。通常，成功返回值为`0`，非零值表示某种形式的失败。通过修改`EXITCODE`值，我们可以返回有关执行的特定信息。由 Procmail 启动的程序的退出代码存储在变量`$?`中。 |

#### 日志变量

在食谱执行期间，任何日志输出的冗长和位置由以下变量控制：

| 名称 | 操作 |
| --- | --- |
| `LOGFILE` | 这指定了 Procmail 应将其所有日志和调试信息写入的位置。如果此值为空，则输出将发送到**标准错误输出**，这意味着除非程序以交互方式运行或`stderr`被重定向到其他位置，否则它将丢失。 |
| `LOG` | 如果我们希望直接写入日志文件，我们可以为`LOG`变量分配一个值，并且它将附加到`LOGFILE`。如果我们想要格式化输出并在我们的日志消息之后包括一个空行，我们必须记得在输出的消息中包含一个空行。`LOG="Procmail is great"` |
| `VERBOSE` | 这允许输出为基本默认或提供详细信息。设置`VERBOSE=1`将包括详细的日志信息，有助于调试我们的配方。为了减少输出信息的数量，请记得在配方运行后设置`VERBOSE=0`。 |
| `LOGABSTRACT` | 如果`LOGABSTRACT`设置为`all`，所有投递将包含有关发送者、主题和投递邮件大小的信息。如果您希望停止此日志记录，请将`LOGABSTRACT=no`。 |
| `COMSAT` | 如果设置为`yes`，Procmail 将生成 comsat/biff 通知。有关更多信息，请参阅`comsat`和`biff`手册页。 |

#### Procmail 的状态变量

在处理配方时，Procmail 使用当前配方的当前状态更新以下变量：

| 名称 | 操作 |
| --- | --- |
| `PROCMAIL_OVERFLOW` | 如果 Procmail 在启动时在 Procmail 配方文件中发现任何行长于缓冲区大小，它将将`PROCMAIL_OVERFLOW`的值设置为`yes`。如果正在读取的行是条件或操作行，则操作将被视为失败。但是，如果它是变量赋值或配方开始，Procmail 将停止读取文件并以异常终止退出。 |
| `HOST` | 这保存了进程正在运行的主机的名称。 |
| `DELIVERED` | 如果邮件成功投递，这将设置为`yes`，并且 Procmail 将通知调用进程。如果我们手动将其设置为`yes` *并且*消息未被投递，它将无迹可寻地丢失，但调用进程仍将认为它已成功投递。 |
| `LASTFOLDER` | 这给出了最后一个写入消息的文件或目录的名称。 |
| `MATCH` | 这保存了上一个正则表达式操作提取的信息。 |
| `$=` | 这保存了最新评分配方的结果。有关更多信息，请参阅*procmailsc*手册页。 |

| `$1，$2，...; $@; $#` | 就像标准 shell 一样，这指定了 Procmail 启动时使用的命令行参数。

+   `$1`是第一个命令行参数，依此类推。

+   `$@`包含所有参数。

+   `$#`包含参数的数量。

另请参阅`SHIFT`伪变量。 |

| `$$` | 这保存了当前进程 ID。这对于创建与进程唯一相关的临时文件非常有用。 |
| --- | --- |
| `$?` | 这保存了上一个 shell 命令的退出代码。 |
| `$_` | 这保存了当前正在处理的 Procmail 文件的名称。 |
| `$-` | 这是`LASTFOLDER`的别名。`$=`和`$@`不能直接使用；在可以用于任何有用的东西之前，我们必须将值分配给另一个变量。 |

#### 消息内容变量

这些变量的主要用途是访问保存在适当部分中的数据，但是当配方具有限制处理到消息的另一部分的标志时。通过使用`HB`，我们可以访问整个消息的信息。

| 名称 | 操作 |
| --- | --- |
| `H` | 这保存了当前正在处理的消息的头信息。 |
| `B` | 这保存当前正在处理的消息的正文。 |

#### 锁定变量

以下表中的每个变量控制任何锁定文件的名称以及配方应等待锁定变为自由的时间。

| 名称 | 行动 |
| --- | --- |
| `LOCKFILE` | 为此变量分配一个值会创建一个全局锁定文件，直到为`LOCKFILE`分配另一个值为止。此值可以是要创建的另一个锁定文件的名称，也可以是一个空值，以删除任何锁定。 |
| `LOCKEXT` | 为此分配一个值允许我们覆盖作为锁定文件名一部分使用的扩展名。这在识别创建锁定文件的进程方面可能很有用。 |
| `LOCKSLEEP` | 如果 Procmail 想要在另一个进程已经锁定的文件上创建一个锁定，它将进入`retry`循环。`LOCKSLEEP`变量指定在重试获取锁之前睡眠和等待的秒数。 |
| `LOCKTIMEOUT` | 这指定了锁定文件必须达到的秒龄，然后假定锁定文件无效并将被覆盖。如果值为`0`，则永远不会覆盖锁定文件。默认值为`1024`秒。 |

#### 错误处理变量

在我们的配方中出现错误时，我们可以使用这些变量中的任何一个来决定采取什么行动。

| 名称 | 行动 |
| --- | --- |
| `TIMEOUT` | 这指定了在告知子进程终止之前等待子进程的时间。默认值为`960`秒。 |
| `SUSPEND` | 这指定了在`NORESRETRY`重试之间等待的时间。默认值为`16`秒。 |
| `NORESRETRY` | 当发生严重的系统资源短缺时（例如磁盘空间不足或系统已达到最大进程数），Procmail 将重试的次数。默认值为`4`，如果数字为负数，Procmail 将永远重试。如果在重试期间资源仍然不可用，消息将被丢弃并被分类为不可投递。 |

#### 杂项变量

以下表格包含有关可能在我们的配方中有用的各种 Procmail 变量的信息：

| 名称 | 行动 |
| --- | --- |
| `LINEBUF` | 这为 Procmail 准备处理的配方行的长度设置了一个限制。如果我们需要处理非常大的正则表达式或将大量数据存储到`MATCH`中，增加此值。 |
| `SHIFT` | 这类似于正常 shell 处理中的`shift`功能。将正数分配给此变量会使 Procmail 的命令行参数下移。 |
| `INCLUDERC` | 这指示 Procmail 加载另一个包含 Procmail 配方的文件。在 Procmail 继续处理当前文件之前，将加载和处理这个新文件。 |
| `DROPPRIVS` | 这确保了在 Procmail 以`setuid`或`setgid`执行时不会有根特权可用。将此值设置为`yes`将使 Procmail 放弃所有特权。 |

#### 打印 Procmail 变量

以下示例将打印大部分环境设置的响应，并提供一些在尝试调试 Procmail 问题时可能有用的信息。不希望将其包含在任何生产文件中，否则我们的日志文件可能会迅速变得非常庞大。

在与其他 Procmail 配方文件相同的目录中创建一个名为`rc.dump`的文件，并将以下行放入文件中：

### 注意

请注意，下一个示例开头和结尾的引号（"）是必需的，以确保配方正常运行。

```
#
# Simple Procmail recipe to dump variables to a log file
#
LOG="Dump of ProcMail Variables
MAILDIR is currently :${MAILDIR}:
MSGPREFIX is currently :${MSGPREFIX}:
DEFAULT is currently :${DEFAULT}:
ORGMAIL is currently :${ORGMAIL}:
SHELL is currently :${SHELL}:
SHELLFLAGS is currently :${SHELLFLAGS}:
SENDMAIL is currently :${SENDMAIL}:
SENDMAILFLAGS is currently :${SENDMAILFLAGS}:
UMASK is currently :${UMASK}:
SHELLMETAS is currently :${SHELLMETAS}:
TRAP is currently :${TRAP}:
EXITCODE is currently :${EXITCODE}:
LOGFILE is currently :${LOGFILE}:
LOG is currently :${LOG}:
VERBOSE is currently :${VERBOSE}:
LOGABSTRACT is currently :${LOGABSTRACT}:
COMSAT is currently :${COMSAT}:
PROCMAIL_OVERFLOW is currently :${PROCMAIL_OVERFLOW}:
TODO is currently :${TODO}:
HOST is currently :${HOST}:
DELIVERED is currently :${DELIVERED}:
LASTFOLDER is currently :${LASTFOLDER}:
\$= is currently :$=:
\$1 is currently :$1:
\$2 is currently :$2:
\$$ is currently :$$:
\$? is currently :$?:
\$_ is currently :$_:
\$- is currently :$-:
LOCKFILE is currently :${LOCKFILE}:
LOCKEXT is currently :${LOCKEXT}:
LOCKSLEEP is currently :${LOCKSLEEP}:
LOCKTIMEOUT is currently :${LOCKTIMEOUT}:
TIMEOUT is currently :${TIMEOUT}:
NORESRETRY is currently :${NORESRETRY}:
SUSPEND is currently :${SUSPEND}:"

```

运行以下命令：

```
# procmail ./rc.dump
<CTRL-D>

```

这将创建以下输出：

```
# procmail ./rc.dump
<CTRL-D>
"Dump of ProcMail Variables
MAILDIR is currently :.:
MSGPREFIX is currently :msg.:
DEFAULT is currently :/var/spool/mail/root:
ORGMAIL is currently :/var/spool/mail/root:
SHELL is currently :/bin/bash:
SHELLFLAGS is currently :-c:
SENDMAIL is currently :/usr/sbin/sendmail:
SENDMAILFLAGS is currently :-oi:
UMASK is currently ::
SHELLMETAS is currently :&|<>~;?*:
TRAP is currently ::
EXITCODE is currently ::
LOGFILE is currently ::
LOG is currently ::
VERBOSE is currently :1:
LOGABSTRACT is currently ::
COMSAT is currently :no:
PROCMAIL_OVERFLOW is currently ::
TODO is currently ::
HOST is currently :delta.adepteo.net:
DELIVERED is currently ::
LASTFOLDER is currently ::
$= is currently :0:
$1 is currently ::
$2 is currently ::
$$ is currently :9014:
$? is currently :0:
$_ is currently :./rc.dump:
$- is currently ::
LOCKFILE is currently ::
LOCKEXT is currently :.lock:
LOCKSLEEP is currently ::
LOCKTIMEOUT is currently ::
TIMEOUT is currently ::
NORESRETRY is currently ::
SUSPEND is currently ::

```

## 配方

Procmail 配方遵循简单的格式。但是，有许多方法可以指示 Procmail 根据许多标志以及规则和配方的编写方式来解释或实施规则中的指令。

### 冒号行

正如我们已经发现的，到目前为止，所有规则都以`:0`开头，后面跟着一个或多个标志和指令。历史上，冒号（`:`）后面跟着一个数字，用来指定规则中存在的条件数量。当前版本的 Procmail 会自动确定条件的数量，因此始终使用值`0`。

#### 锁定

我们已经讨论过，为了阻止多个进程同时尝试写入同一文件，我们需要使用锁定机制。当然，这个要求随着过滤器试图调用的进程类型而变化。例如，仅仅更改或分配值的过滤器对任何物理文件都没有影响，因此不需要锁定。同样，仅仅将数据转发到另一个进程或另一个接收者的过滤器本质上不需要应用锁定。在大多数情况下，当 Procmail 意识到它正在写入文件并提供文件本身的锁定时，将自动应用锁定。在某些情况下，可能需要显式锁定资源。

以下是一些示例，以便了解何时自动应用锁定，根本不需要，或需要强制手动锁定。

##### 自动锁定

任何以`:0:`开头的规则都将应用自动文件锁定。在这种情况下，Procmail 将自动确定邮件被传递到的文件的名称，并创建一个锁定文件。如果锁定文件已经存在，它将等待一段时间并重试创建锁定。当它最终创建了锁定文件，它将继续处理。如果无法创建锁定文件，它将报告错误并继续下一个规则。

以下规则使用自动锁定：

```
:0 <flags>:

```

##### 强制锁定

可能会有一段时间，特别是在通过外部脚本处理邮件时，需要强制锁定。在大多数情况下，Procmail 将通过检查进程命令行并查看输出的位置来确定最终数据被写入的文件的名称。然而，如果脚本负责选择输出位置本身，或者依赖于可能被另一个 Procmail 进程更改的文件，必须明确请求锁定文件如下：

```
:0 <flags> :scriptname.lock

```

在大多数情况下，您不太可能需要在编写的大多数脚本中强制锁定。

##### 无锁定

当转发到执行自己的文件或记录锁定过程的管道时，例如将问题报告存储在数据库中，不需要记录锁定。同样，如果消息被转发给另一个用户，最终交付将处理记录锁定。简单的规则定义是：

```
:0 <flags>

```

#### 标志

在我们迄今为止看过的例子中，我们已经允许 Procmail 的默认设置生效。然而，有许多标志可以设置以控制 Procmail 的工作方式。

![标志

##### 默认标志

如果在食谱的冒号行上没有声明标志，Procmail 将假定以下标志（`H，hb`）已被用作默认值。

| 标志 | 动作 |
| --- | --- |
| `H` | 仅扫描邮件标题。 |
| `hb` | 动作行同时传递邮件数据的标题和正文。 |

##### 匹配范围：HB

通常，匹配将在整个邮件包括标题和邮件正文之间进行。如果邮件正文可能很大，并且我们知道我们需要对标题进行匹配，那么使用`H`标志限制匹配范围仅限于标题是明智的。

相反，有时我们可能正在寻找信息项，例如文档正文中仅出现的重复页脚或签名，这种情况下我们可以使用`B`标志将匹配限制为仅限于正文。

| 标志 | 动作 |
| --- | --- |
| `H` | 仅在邮件标题中进行匹配。 |
| `B` | 仅在邮件正文中进行匹配。 |
| `HB` | 在整个邮件项目中包括标题和正文进行匹配。 |

##### 操作范围：hb

默认情况下，操作行处理整个电子邮件项目，包括标题和正文。如果需要仅处理邮件数据的一部分，则可以指定将哪一部分传递给操作行。

| 旗帜 | 行动 |
| --- | --- |
| `h` | 仅将标题传递给操作行进行处理。 |
| `b` | 仅将消息正文传递给操作行进行处理。 |
| `hb` | 仅将标题和消息正文传递给操作行进行处理。这是默认范围。 |

### 注意

注意在“匹配范围”和“操作范围”之间的区别很重要。在第一种情况下，标志的值确定必须扫描邮件的哪一部分（标题、正文或整个邮件）以进行匹配。在第二种情况下，标志的值确定必须处理邮件的哪一部分。

##### 流程控制：aAeEc

这可能是所有 Procmail 标志中最复杂的一组标志。本章后面的示例将解释使用这些标志的各种方式。简而言之，可以假定每个标志的以下内容：

| 旗帜 | 行动 |
| --- | --- |
| `A` | 只有前一个配方的条件得到满足时，才会处理该配方。 |
| `a` | 如果前一个配方的条件得到满足，并且操作已经完成而没有错误，则将处理该配方。 |
| `E` | 这是`A`的相反。如果前一个配方的条件未满足，则将处理该配方。 |
| `e` | 如果前一个配方的条件得到满足，但处理未能成功完成，将处理该配方。 |
| `c` | 这指示配方创建原始消息的副本或克隆，并使用任何操作在子进程中处理此副本。父进程继续处理消息的原始副本。 |

`c`标志应该被理解为`克隆`或`复制`。普遍的误解是该标志应该被解释为`继续`。`克隆`或`复制`操作创建数据的单独副本，并创建一个单独的执行流程来处理该数据，有时作为完全独立的子进程。当此克隆配方完成时，父进程将继续执行，并保持原始数据不变。

##### 区分大小写：D

顽固的 Linux 用户非常了解区分大小写，并始终将`大写`视为与`小写`完全不同。然而，Procmail 的默认操作是在匹配字符串时不区分大小写。这意味着对于 Procmail 来说，`大写`和`小写`是相同的，除非告知应用`D`标志应用区分大小写。

##### 执行模式：fwWir

我们可以指示 Procmail 如何处理或执行配方，以及在处理过程中遇到错误时要采取的操作。当处理仅在数据的前几行上进行时，较小的邮件消息可能不会出现错误。然而，对于较大的消息，当管道仅读取可用数据的一部分时，Linux shell 可能会认为存在错误。

执行模式的**过滤模式**很重要。这个术语可能会让人困惑，因为 Procmail 的设计目的只是过滤邮件。以以下方式考虑执行模式“过滤”：我们正在处理的邮件消息将在实际传递到 Procmail（或至少我们配方的其余部分）之前通过操作行上的任何内容进行管道传输。查看过滤模式的另一种方式是将其视为一种转换模式，在这种模式下，数据以某种方式进行修改，然后返回到控制 Procmail 配方以进行进一步执行。

| 旗帜 | 行动 |
| --- | --- |
| `f` | 将消息内容通过配方传递给外部管道进程进行处理，然后获取处理过程的输出行，准备替换原始消息内容。 |
| `i` | 如果 Linux 管道进程只读取其输入的一部分，然后终止，shell 将向 Procmail 程序发送`SIGPIPE`错误信号——`i`标志指示 Procmail 忽略此信号。这应该在预期管道进程只处理部分消息后返回时使用。 |
| `r` | 传递给管道进程的数据应该原样传递，不做任何修改。 |
| `w` | 默认情况下，Procmail 进程将生成一个子进程并继续自己的处理。`w`标志指示 Procmail 等待子进程管道完成后再继续自己的处理。 |
| `W` | 这与`w`的工作方式相同，但也隐藏了管道进程的任何错误或其他输出消息。 |

### 条件

有许多条件类型可以应用于决定给定配方是否适用于特定的邮件项。正确应用条件的想法是减少执行的不必要处理量。

条件行始终以星号（*）字符开头，后面跟着一个或多个空格。可以在一个配方内应用多个条件行，但它们必须全部分组在连续的行上。分组的逻辑操作是执行`AND`操作，以便在执行动作之前必须应用所有条件。

```
:0
* condition1
* condition2
action_on_condition1_and_condition2

```

#### 无条件应用规则

可能需要应用规则到所有消息，而不考虑任何条件。例如，这样的规则可以备份邮件消息到邮件文件夹，或者出于法律或公司政策原因归档所有邮件。

无条件规则是由缺少条件行而暗示的。也就是说，规则总是匹配的。

```
# Save all remaining messages to DEFAULT
:0:
${DEFAULT}/

```

无条件规则经常用于嵌套配方链的末尾，以执行最终的默认操作，如果配方未传递邮件。请记住，一旦邮件被传递，处理就会停止。

#### 使用正则表达式进行测试

我们中的一些人熟悉简单的模式匹配操作，比如在文件列表操作中常用的`?`或`*`，可能会想知道是否可以创建类似的测试来匹配邮件头部或正文的部分。好消息是，有一个称为**正则表达式**或**regex**的优秀功能。这些提供了一个执行非常复杂的模式匹配操作的机制。总的来说，这个功能与`egrep`命令行正则表达式非常相似。然而，有一些重要的区别，有经验的`regex`用户一定要了解，以便了解如何编写适用于 Procmail 操作的表达式。本章后面有一个完整的关于编写`regex`的部分。

正则表达式可以针对邮件消息的数据部分（标题、正文或两者）运行，如标志所定义，也可以用于测试先前分配的变量。

| 条件 | 动作 |
| --- | --- |
| `* 正则表达式` | 根据标志测试消息的部分与正则表达式。通常这只会处理标题，除非给出`B`标志以指示匹配范围是处理消息的正文。 |
| `* 变量 ?? 正则表达式` | 这是为了将分配的变量与`正则表达式`进行比较。 |

本章前面列出了各种伪变量，代表了访问 Procmail 应用程序中包含的信息的方式。这些伪变量可以与普通变量相同方式进行比较。

以下示例将复制所有包含关键短语的邮件项的副本。

```
VERBOSE=1
:0cB:
* [0-9]+ Linux Rules [ok!]
${MAILDIR}/linuxrules/
VERBOSE=0

```

以下是对前面示例操作的快速解释：

+   我们指定`:0cB:`以确保我们只搜索邮件正文，并进行复制，以便我们仍然可以得到原始邮件的处理。

+   如果消息正文中有一个短语，该短语有一个或多个数字，后面跟着`<SPACE>Linux Rules<SPACE>`，然后是`o, k`或`!`，则会将副本存储在`linuxrules`文件夹中。

在处理规则之前设置和取消设置`VERBOSE`选项允许在日志中更详细地显示该规则，这意味着在调试时需要搜索的日志文件更少。

#### 测试消息部分的大小

在某些情况下，我们可能不希望处理大型消息的配方。在这种情况下，我们可以设置一个限制，使得配方不匹配超过一定大小的消息。如果我们有使用缓慢数据连接的用户，也许使用手机连接的连接，将所有大型邮件项目移动到一个单独的文件夹中以便在用户回到更好的互联网连接时检索，这可能是有用的。

| 条件 | 操作 |
| --- | --- |
| `* > number` | 如果消息大小大于给定的字节数，则返回`true`。 |
| `* < number` | 如果消息大小小于给定的字节数，则返回`true`。 |

#### 测试外部程序的退出代码

如果运行外部程序来提供处理的一部分，则可能需要检查退出代码，以确保进程正确完成或执行次要操作以完成整体处理。

```
*? /unix/command/line | another/command*emphasis>
```

`?`指示 Procmail 将当前消息数据作为标准输入传递到 Linux 命令行。如果命令行以零退出代码退出，则条件成功满足。虽然命令行是几个进程的管道，但返回的退出代码是管道中最后一个程序的退出代码。

管道打印到标准错误的任何输出都会显示在日志中。

在这个例子中，消息的主体被传递到命令管道中，如果短语确切地在第三行中找到（退出代码为 0），则消息将被存储在文件夹中。

在`VERBOSE=1`和`VERBOSE=0`之间的行的操作将被记录，但此范围之外的所有行将不被记录。这使我们能够控制正在进行的日志记录量，因此更容易跟踪日志文件活动。

```
VERBOSE=1
:0B:
* ? /bin/sed -n 3p | /bin/egrep "Linux Rules"
${MAILDIR}/linuxrules/
VERBOSE=0

```

#### 否定

有时，检查特定条件是否不存在以便以某种方式继续处理是有用的。**感叹号**（!）或有时称为**Bang**，用于颠倒条件的值，使得假变为真，反之亦然。

```
* ! condition

```

这测试条件中的负结果，并且如果条件不满足则返回`true`。

在这里，我们正在寻找任何未直接发送给我们的项目，并将其存储在一个文件夹中以供以后查看。

```
:0:
* !^TO.*cjtaylor
${MAILDIR}/not_sent_to_me/

```

#### 条件中的变量替换

多个`$`标志可以用于强制应用多个替换传递。

```
* $ condition

```

`$`指示 Procmail 使用正常的`sh`规则处理条件，以执行变量和反引号替换，然后再实际评估条件。替换过程将解析变量（`$VAR`）为它们的值，而不是将它们作为文字处理。任何带引号的字符串都将删除其引号，所有其他 shell 元字符也将被评估。要使这些字符通过此替换过程，它们应该使用标准的反斜杠（\）转义机制进行转义。

下面的例子摘自*procmailex*手册页，即使在那里也被描述为相当奇异，但它确实作为一个例子。假设您的主目录中有一个名为`.urgent`的文件，并且该文件中列出的（一个）人是传入邮件的发件人。您希望该邮件存储在`$MAILDIR/urgent`中，而不是存储在任何正常的邮件文件夹中。那么您可以这样做（注意，`$HOME/.urgent`的文件长度应远低于`$LINEBUF`；如有必要，增加`LINEBUF`）：

```
URGMATCH=`cat $HOME/.urgent`
:0:
* $^From.*${URGMATCH}
$MAILDIR/urgent/

```

### 操作行

这是执行所有处理活动的行。在大多数情况下，这意味着写入物理文件或文件夹。但它也可以包括将邮件转发给其他用户，将数据传递给命令或一系列命令的管道，或者在某些情况下，作为复合食谱的一部分执行一系列连续的操作。如果要执行多个操作，不能只是将它们堆叠在一起 - 您需要多个食谱（可能是无条件的，和/或分组在一对大括号中）和每个冒号行（当然还可以是条件）。

还要注意，影响操作行的标志实际上直到尝试执行操作时才会生效。特别是，`c`标志直到其所有条件都得到满足之前才生成消息的克隆。

#### 转发到其他地址

将用户帐户的所有消息全局转发到另一个用户帐户是由 Postfix 本身处理效率要高得多的过程。但是，如果需要应用一些逻辑来决定发送消息的内容或位置，那么 Procmail 可以提供帮助。

大多数邮件传输都允许我们传递多个电子邮件地址以进行进一步传输。

```
! user1@domain2.net user2@domain1.com user3 ...

```

上述操作在功能上与将消息传递到以下管道相同：

```
| $SENDMAIL "$SENDMAILFLAGS"

```

这是转发邮件的特殊情况，并指示 Procmail 从原始消息的实际标头中提取收件人列表：

```
! -t

```

在这里，我们将邮件转发给我们的支持团队，而不是自己处理。邮件的主题行包含短语**support**。

```
:0:
* ^Subject.*support
! support@adepteo.net

```

#### 喂给 shell 或命令管道

Procmail 允许在电子邮件中做几乎无限的事情。与 Procmail 一起工作的更强大的功能之一是其能够根据给定的条件将电子邮件转发到应用程序或脚本。一个可能的例子是跟踪支持请求，并将条目直接存储到数据库系统中，以便在专用应用程序中进行跟踪。

管道过程负责保存其输出。食谱的标志能够告诉 Procmail 期望其他内容。通过使用`>>`语法，Procmail 可以确定要使用的锁定文件。在写入文件时始终使用锁定非常重要，以避免两个操作同时写入同一个文件并损坏彼此的数据。

```
| cmd1 param1 | cmd2 -opt param2 >>file

```

可以将命令管道的输出存储在变量中。这本身使得食谱成为非交付食谱。

```
VAR=| cmd1 | cmd2 ...

```

请注意，此语法仅允许在操作行上使用。要在普通赋值中获得相同的结果，我们可以使用反引号（`）运算符。

```
VERBOSE=1
#Copy the data and pass the headers to the process
:0hc:
* ^Subject: Book Pipeline Example
#Copy so that the next recipe will still work
| cat - > /tmp/cjt_header.txt
#Final recipe so do not copy here, but pass the body
:0b:
| cat - > /tmp/cjt_body.txt
VERBOSE=0

```

#### 保存到文件夹

这将输出保存到普通文件中。如果只提供文件名，文件将在`MAILDIR`设置中指定的目录中创建。在写入普通文件时，一定要确保使用某种形式的锁定。

```
/path/to/filename

```

保存到目录时，文件将在目录中创建具有顺序编号的文件。

在路径名的末尾使用尾部（/）斜杠指示 Procmail 将项目存储在`maildir`格式的文件夹中。子文件夹`cur, new`和`tmp`将自动创建。

```
directory/

```

在路径名的末尾使用`/`指示 Procmail 将项目存储在`MH`格式的文件夹中。

```
directory/.

```

如果要将数据存储到多个`MH`或`maildir`文件夹中，可以同时列出它们。结果将是实际只写入一个文件，其余将作为硬链接创建。

#### 复合食谱

如果我们想对匹配项执行一系列条件进程或操作，那么我们可以使用`{`和`}`字符指定要使用的一组食谱块，而不是单个操作行。`{`和`}`字符后必须至少有一个空格。

```
{
# ... more recipes
}

```

大括号之间的代码可以是任何有效的 Procmail 结构。

### 注意

请注意，变量赋值的操作必须始终放在一组大括号内：`{ VAR=value }`。如果只使用`VAR=value`而不使用大括号，数据将保存在名为`VAR=value`的文件夹中。

如果我们想要一个实际上不进行任何处理的配方，也许作为`if…else`操作的一部分，我们可以使用一个空的`{ }`，但是空格的规则仍然适用，我们需要确保两个大括号之间至少有一个空格字符。

下面的例子是基于前面的例子稍作修改，以便只执行一个测试，然后如果测试通过，则运行一系列无条件的测试：

```
VERBOSE=1
:0:
* ^Subject: Book Pipeline Example
{
#Copy so that the next recipe will still work
:0hc:
| cat - > /tmp/cjt_header.txt
#Final recipe so do not copy here
:0b:
| cat - > /tmp/cjt_body.txt
}
VERBOSE=0

```

# 正则表达式

Procmail 实现了一种与其他 UNIX 实用程序略有不同的正则表达式形式。在这里，我们将介绍基本的区别，并引导新用户进入正则表达式的强大世界，它们的含义、实现和用途。

我们已经看到，Procmail 的匹配是不区分大小写的，除非使用了`D`标志。对于正则表达式也是如此。Procmail 还默认使用多行匹配。

## 正则表达式简介

对于 Linux 和编程的新用户来说，可能不清楚正则表达式为处理数据带来的强大功能。在其最简单的形式中，正则表达式可以理解为在一组数据中搜索短语或模式。以下简单的例子展示了如何匹配所有邮件项目，其中标题和/或正文包含短语`mystical monsters`，并将邮件放在相关文件夹中。

```
:0 HB:
* mystical monsters
${MAILDIR}/monsters/

```

然而，这个过滤器将无法匹配包含短语`mystical monster`或`mystical-monsters`的项目，例如。因此，正则表达式的真正威力在于能够以简化的格式描述文本或数据模式，然后在一组数据中搜索与这些模式匹配的内容。但是，你应该小心不要被“简化”这个词所误导。在现实生活中，你遇到的大多数正则表达式可能并不简单，如果用原生格式编写的话。以下面的例子为例，它的目的是确定邮件项目是否是 MIME 编码，并在适当的文件夹中存储它：

```
:0:
* ^Content-Type: multipart/[^;]+;[ ]*boundary="?\/[^"]+
${MAILDIR}/mime/

```

字符`., [, ^, ;, ], +, ?, \, /`和`"`是特殊指令，而不是它们通常表示的字面 ASCII 字符。为了理解这些字符及其含义，我们将快速浏览一下最重要的例子。

### 句点

这是最简单和最常见的正则表达式形式，简单地意味着匹配任何单个字符（不包括换行符，换行符被视为特殊情况）。考虑以下表达式：

```
:0
* Dragons ... mystical monsters
${MAILDIR}/result/

```

这将匹配以下任何短语：

```
Dragons are mystical monsters
Dragons and mystical monsters
Dragons but mystical monsters

```

实际上，它将匹配`Dragons`和`mystical`之间有一个三个字符的单词的任何短语。如果我们想匹配`Dragons`和`mystical`之间长度为三个或更多字符的任何单词，我们可以使用`?`或量词操作。

如果我们想匹配一个字面上的'.'或多个'.'，我们可以通过在特殊意义的字符前加上反斜杠'\'来转义正则表达式字符串，使得'\.'可以字面上匹配'.'（句号），'\\'可以字面上匹配'\'（反斜杠）字符。

### 量词操作

问号表示前面的字符应该匹配零次或一次。因此，以下代码行将满足我们的要求：

```
:0
* Dragons ....? Mystical monsters
${MAILDIR}/result/

```

这个表达式可以理解为，“匹配由三个或更多字符组成的任何单词，后面跟着空白或任何一个字符”。

`?`前面的字符也可以是一个简单的 ASCII 字符，这样表达式将匹配如下：

```
:0
* Dragons ..d? Mystical monsters
${MAILDIR}/result/

```

这可以理解为，“任意两个字符后面跟着空白或字母`d.`”因此，这将匹配`an`和`and`，但不会匹配`are.`

### 星号

星号修饰符的工作方式类似于量词运算符，但意思是匹配前面的字符的零个或多个实例，当然不包括换行符。`.*`是一个非常常见的序列，你会在很多配方中找到它。

以下示例将匹配所有包含单词`choose`后跟其他单词再跟单词`online`的消息：

```
:0
* ^Subject: Choose.*online
${MAILDIR}/result/
Subject: Choose discount pharmacy and expedite the service online.
Subject: Choose hassle free online shopping
Subject: Choose reliable online shopping site for reliable service and quality meds
Subject: Choose reliable service provider and save more online.
Subject: Choose the supplier for more hot offers online
Subject: Choose to shop online and choose to save

```

下一个例子将寻找“任何东西”（.*）后跟两个或更多感叹号（!!）和（!*）：

```
:0
* ^Subject: .*!!!*
${MAILDIR}/result/
Subject: Breathtaking New Year sale on now!!! Get ready for it!! Subject: Hey Ya!! New Year Sale on right now!! Subject: It Doesn't Matter!!

```

### 加号

加号与`*`非常相似，只是它要求正则表达式中`+`之前必须至少有一个字符的实例。

如果我们考虑我们之前的例子，下一个例子将寻找“任何东西”`.*`后跟两个`!!`和至少一个以上的`!`感叹号。

```
:0
* ^Subject: .*!!!+
${MAILDIR}/result/

```

这将给我们一个更受限制的输出，至少需要三个`!`连续出现。

```
Subject: Breathtaking New Year sale on now!!! Get ready for it!!

```

### 使用括号进行限制匹配

到目前为止，我们能够创建的匹配模式功能强大，但工作方式不够集中。例如，我们可以轻松地编写一个规则来查找以`t`结尾的任何三个字母的单词，但无法将匹配限制为仅包含一组给定的以`t`结尾的单词。为了克服这一点，我们可以用一组字符或字符组的集合替换`.`或单个字符，然后应用量词操作来准确说明这些字符可以应用多少次。

通过小括号`( )`的谨慎使用，我们可以创建字符串组，这些组将用于模式匹配规则。例如，假设我们试图拆分由系统脚本频繁发送的电子邮件。脚本格式化主题行，使其包含以下短语之一。

```
There is only one problem
There are 10 problems

```

下面的正则表达式将匹配我们要查找的特定字符串，方法是匹配任何字符串，该字符串在`there`和`problem`之间有一个或多个`is only one`短语的出现。

```
There (is only one)+ problem

```

如果我们想要过滤一个词或短语列表，我们需要使用**Alternation**功能。

```
There (is only one|are)+ problem

```

`|`字符分隔了可以用来匹配模式的单词列表。

以下简单的垃圾邮件过滤器使用了**Alternation**功能，以搜索常用的文本替换，以避免简单的基于单词的过滤器。

### 创建一个简单的垃圾邮件过滤器

随着我们每天收到的垃圾邮件数量不断增加，我相信到目前为止阅读到这里的一些人已经想出了我们可以开始过滤我们每天收到的一些常规消息的方法。有许多专门设计用于与 Procmail 紧密配合并为垃圾邮件过滤提供更多测试和覆盖范围的特定垃圾邮件过滤器。其中一个应用程序 SpamAssassin，在第八章中有介绍。

以在线赌场为例——这是垃圾邮件发送者鼓励我们探索的热门主题。这不是我们通常感兴趣的东西，所以我们很高兴地将所有包含“在线”和“赌场”这两个词的消息过滤到一个单独的文件夹中。

```
Subject: Online Casino

```

垃圾邮件发送者面临的挑战之一是编写我们可以阅读而垃圾邮件过滤器难以处理的主题行。一个简单的方法是用常见的打字错误字符替换，比如数字零（`0`）替换字母`O`或`o`，数字`1`替换`L`或`l`，数字`4`替换`A`或`a`。

因此，我们可以继续编写规则：

```
Subject: (o|0)n(1|l)ine casin(o|0)

```

下一个迭代将具体显示这个配方，我们特别寻找包含“在线”和“赌场”这两个词的主题行，但要包括单词可能以不同顺序出现的情况，每个单词都会分别测试。

```
:0
* ^Subject: (o|0)n(1|l)ine
* ^Subject: casin(o|0)
${MAILDIR}/_maybespam/

```

虽然这样做效果不错，但是以这种方式工作的规则并不是真正高效的，因为这种替换是正则表达式的常见需求，所以有一种特殊的方式来表达这些术语，即**字符类**。

### 字符类

方括号`[ ]`中包含的任何字符序列表示要在表达式中检查每个列出的字符。对于字母表中的字母或数字范围等常见字符序列，可以使用`[a-z]`或`[0-9]`：

+   `[a-e]` 意味着匹配所有字母`a, b, c, d, e`。

+   `[1,3,5-9]` 意味着匹配数字`1, 3, 5, 6, 7, 8`或`9`中的任何一个。

以下示例将查找在文本字符串中嵌入数字`0`和`1`的消息，以便它们看起来像`O`和`L`或`I`。

```
:0
* ^Subject: [a-z]*[01]+[a-z]*
${MAILDIR}/_maybespam
Subject: Hot Shot St0ckInfo VCSC loadstone Subject: M1CR0S0FT, SYMANNTEC, MACR0MEDIA, PC GAMES FROM $20 EACH Subject: R0LEX Replica - make your first impressions count! Subject: Small-Cap DTOI St0cks reimburse Subject: TimelySt0ck DTOI Buy of the Week evasive

```

### 行首

如果我们想匹配所有范围广泛的字符而不匹配少量范围，那么使用`^`字符指定负匹配会更容易。

```
[⁰-9]

```

这意味着匹配任何以不是`0`到`9`之间的数字开头的字符串。

当我们知道模式应该从行首开始时，将行首锚添加到我们正在搜索的模式中是很有用的。例如，所有标题必须从行首开始，因此搜索以下短语：

```
Subject: any subject message

```

也将匹配以短语开头的标题，例如：

```
Old-Subject:

```

为了停止这个，我们可以添加**行首锚字符**（^）并将正则表达式更改为：

```
^Subject: any subject message

```

### 行尾

当我们计划匹配我们知道应该终止的字符串时，我们可以将**行尾锚字符**（$）添加到模式中，以确保我们匹配到字符串的末尾，如下所示：

```
^Subject:.* now$

```

这将匹配任何以单词`now`结尾的主题行。

## 进一步阅读

正则表达式是一个庞大的主题，但是非常值得学习，因为它们被许多 Linux 工具和应用程序使用。有许多与正则表达式相关的在线资源。以下是一些入门链接：

+   [`www.regular-expressions.info/`](http://www.regular-expressions.info/)

+   [`en.wikipedia.org/wiki/Regular_expression`](http://en.wikipedia.org/wiki/Regular_expression)

正如我们在上一章中简要介绍的那样，Procmail 有许多有用的“预先准备好的”正则表达式或宏，提供了一系列在 Procmail 配方中常用的匹配。

## ^TO 和^TO_

`^TO`是处理“To”地址的原始 Procmail 宏。这已经被新的`^TO_`宏所取代，该宏是在 Procmail 版本 3.11pre4 中引入的。

这个万能匹配包括大多数标题，这些标题中可能包含您的地址，例如`To:, Apparently-To:, Cc:, Resent-To:`等等。

在大多数情况下，您应该使用`^TO_`选项，因为它的覆盖范围要好得多。

### 注意

尽管似乎有逻辑地有一个类似的宏来覆盖源地址详细信息，但请注意，*没有*相应的`^FROM`或`^FROM_`宏。

这是来自 Procmail 源代码的正则表达式字符串：

```
"(^((Original-)?(Resent-)?(To|Cc|Bcc)|\
(X-Envelope|Apparently(-Resent)?)-To):(.*[^-a-zA-Z0-9_.])?)"

```

## ^FROM_MAILER

这个宏识别了广泛的邮件生成程序，并且是一个有用的万能匹配。但是，新程序一直在被创建，因此几乎总是需要额外的过滤器。

Procmail 将这个简短的宏扩展为从 Procmail 源代码中获取的以下正则表达式。

```
"(^(Mailing-List:|Precedence:.*(junk|bulk|list)|\
To: Multiple recipients of |\
(((Resent-)?(From|Sender)|X-Envelope-From):|>?From )([^>]*[^(.%@a-z0-9])?(\
Post(ma?(st(e?r)?|n)|office)|(send)?Mail(er)?|daemon|m(mdf|ajordomo)|n?uucp|\
LIST(SERV|proc)|NETSERV|o(wner|ps)|r(e(quest|sponse)|oot)|b(ounce|bs\\.smtp)|\
echo|mirror|s(erv(ices?|er)|mtp(error)?|ystem)|\
A(dmin(istrator)?|MMGR|utoanswer)\
)(([^).!:a-z0-9][-_a-z0-9]*)?[%@> ][^<)]*(\\(.*\\).*)?)?$([^>]|$)))"

```

## ^FROM_DAEMON

这采用了与`^FROM_MAILER`类似的方法，但旨在识别更常见的 Linux 守护程序和系统进程的消息。

给出了来自 Procmail 源代码的正则表达式字符串：

```
"(^(((Resent-)?(From|Sender)|X-Envelope-From):|\
>?From )([^>]*[^(.%@a-z0-9])?(\
Post(ma(st(er)?|n)|office)|(send)?Mail(er)?|daemon|mmdf|n?uucp|ops|\
r(esponse|oot)|(bbs\\.)?smtp(error)?|s(erv(ices?|er)|ystem)|A(dmin(istrator)?|\
MMGR)\
)(([^).!:a-z0-9][-_a-z0-9]*)?[%@> ][^<)]*(\\(.*\\).*)?)?$([^>]|$))"

```

以下示例将存储接收到的守护程序消息在包含年份和月份作为路径的文件夹中。这些变量`${YY}`和`${MM}`在 Procmail 文件中之前分配，并且必要的目录也已创建。

```
:0:
* ^FROM_DAEMON
${YY}/${MM}/daemon

```

# 高级配方

在这里，我们将把 Procmail 的各种功能组合成几个有用的食谱，作为我们自己组织中工具的基础。第一个例子是基于传统的`度假`食谱，通知发件人可能需要一段时间才能被收件人阅读的电子邮件。第二个示例展示了如何创建支持，根据日期和可能的处理时间自动归档消息。最后，我们将完成上一章开始的规则，通知用户已被过滤到单独文件夹中的大型邮件项目。

## 创建度假自动回复

这个例子是基于`man procmailex`中给出的度假示例，并在本章中稍作提及。

正如我们已经讨论过的那样，盲目自动回复电子邮件是一个非常糟糕的主意，并且具有重大影响。首先，我们必须决定是否发送自动回复。为此，我们需要确保条件是合理且满足的。如果是这样，当前消息的标题（由`h`标志表示）被提供给`formail`，这是 Procmail 工具套件的一部分。然后，`formail`检查`vacation.cache`文件，以查看发件人是否已经收到自动回复。这是为了确保我们不会向用户发送多个报告。在进行这部分处理的同时，我们的食谱将创建一个名为`vacation.lock`的锁。

这样做的主要原因是为了避免在更新缓存时发生冲突，这可能导致缓存信息损坏。

这个食谱实际上包括两个单独的食谱。第一个提供了检查和记录已发送的回复，以确保我们不发送重复的回复。

这个食谱`W`，等待`formail`的返回。没有`c`，Procmail 在完成这个食谱后会停止处理，因为它是一个投递食谱。它将标题传递给`formail`。

`TO_`和`^FROM_DAEMON`条件的含义远不止表面上看到的那样。

如果用户的登录名出现在任何接收者标题**To:, Cc:, Bcc:**中，`TO_ $<logname>`就会被满足。这样可以避免向地址为别名或邮件列表的消息发送自动回复，而不是明确发送给我们的用户。

`!^FROM_DAEMON`确保我们不会自动回复来自各种守护程序的消息。

`!^X-Loop: $RECIPIENT`避免回复我们自己的自动回复；请注意，我们发送的自动回复中插入了这个`X-Loop`标题。

```
:0 Whc: vacation.lock
# Perform a quick check to see if the mail was addressed to us
* $^To_:.*\<$\LOGNAME\>
# Don't reply to daemons and mailinglists
* !^FROM_DAEMON
# Mail loops are evil
* !^X-Loop: $RECIPIENT
| formail -rD 8192 vacation.cache

```

如果第一个食谱在缓存中找不到匹配项，那么食谱的第二部分就会执行。地址找不到的原因有两个——要么从未见过，因此没有发送过回复，要么是很久以前见过，以至于条目已经被强制从缓存中移除。在任何一种情况下，都会发送一份度假消息的副本。发件人永远不会收到他们发送的每一条消息的自动回复——这可能会让频繁发送邮件的人感到不安。

```
:0 ehc
# if the name was not in the cache
| (
formail -rA"Precedence: junk" \
-A"X-Loop: $RECIPIENT" ; \
cat $HOME/.vacation_message \
) | $SENDMAIL -oi -t

```

由于`e`，如果前一个食谱返回错误状态，就会执行前一个食谱。在这种情况下，这并不是真正的错误，只是来自`formail`的信号，表示地址在缓存文件中不存在，我们可以继续发送自动回复。请注意，如果在前一个食谱中条件不满足导致跳过`formail`缓存检查，Procmail 足够聪明，会跳过这个食谱。

当前消息的标题被提供给这个食谱中的`formail`，以便构建自动回复的标题。

这个食谱中的`c`导致在这个食谱之后处理整个当前消息。通常，这意味着它将在没有进一步食谱的情况下被处理，这就是我们在邮箱中得到一份副本的方式。在执行这个食谱时不需要锁，因此没有使用锁。

向原始消息的发送者发送回复所需的只是消息的副本，该副本保存在用户的主目录中的`.vacation_message`文件中。

在 Procmail 配方之外存储消息信息使得系统用户可以轻松更新他们发送的消息，而不会破坏实际的配方本身。

## 按日期组织邮件

您可能不想删除您认为将来可能有用的邮件。这很容易导致大量数据存储在各种位置。可以根据年份、月份和主题的组合将我们的一些或所有传入邮件过滤到文件夹中，以便轻松追踪。

应用于每个邮件过程的通用规则确保必要的目录结构存在。

```
#Assign the name of the folder by extracting the year and month
# parts from the external date command.
MONTHFOLDER=`date +%y/%m`
#Unconditional rule to create the folder. Using the test
#command. we create the monthly folder if it does not exist.
:0 ic
* ? test ! -d ${MONTHFOLDER}
| mkdir -p ${MONTHFOLDER}
#Alternative way of creating the folder using an assignment operation
DUMMY=`test -d $MONTHFOLDER || mkdir $MONTHFOLDER`
#Now store any email matching 'meeting' in an appropriate folder
:0:
* meeting
${MONTHFOLDER}/meeting/

```

如果您更喜欢对输出格式或位置有更多控制，可以使用以下规则：

```
#This obtains the date formatted as YYYY MM DD, e.g. 2009 09 08 date = `date "+%Y %m %d"`
#Now assign the Year YYYY style :0 * date ?? ^^()\/ { YYYY = $MATCH }
#Now assign the Year YY style :0 * date ?? ^^..\/ { YY = $MATCH }
#Now assign the Month MM style :0 * date ?? ^^.....\/ { MM = $MATCH }
#Now assign the Day DD style :0 * date ?? ()\/..^^ { DD = $MATCH }
#Create the various directory formats you are going to use
DUMMY=`test -d ${YYYY}/${MM}/${DD} || mkdir -p ${YYYY}/${MM}/${DD}`
DUMMY=`test -d ${YY}/${MM} || mkdir -p ${YY}/${MM}`
#Now store the data in an appropriate folder using the variables
#YYYY, MM and DD setup above.
:0:
* ^FROM_DAEMON
${YYYY}/${MM}/${DD}/daemon/

```

## 通知用户有关大型邮件

在上一章中，我们介绍了一个非常简单的规则，将所有大小超过 100KB 的传入邮件存储在`largemail`文件夹中。这对于保持单个传入邮件文件夹的大小不会变得太大很有用，但意味着需要定期进行特殊检查，以查看是否已过滤任何邮件。

在这个规则中，我们现在将提取标题和主题行，以及原始大型电子邮件消息的前几行，并创建一个带有修改主题行的新消息。这个修改后的消息将与将大型原始项目过滤到其单独的`largemail`文件夹中同时存储在用户的收件箱中。

测试的主要部分仅在消息大小超过 100,000 字节时才会应用，因此我们需要类似以下配方的结构来进行初始测试，并决定这是否是一个大项目：

```
:0:
* >100000
{
MAIN PROCESS WILL GO HERE
}

```

假设我们确实有一个大项目，我们需要使用`c`标志复制消息，并将此副本存储在`largemail`文件夹中：

```
#Place a copy in the largemail folder
:0 c:
largemail/

```

接下来是提取消息正文的第一部分，可以使用各种选项来完成。在这种情况下，我们将通过等待传递消息正文的结果来剥离消息的前 1024 字节，只告诉系统头命令返回前 1024 字节。这里使用的标志告诉 Procmail 等待命令行进程的结果，并忽略任何管道错误，因为头命令只会读取提供给它的数据的一部分。

```
#Strip the body to 1kb
:0 bfwi
| /usr/bin/head -c1024

```

现在我们需要重写主题行，这是使用`formail`程序完成的。这一次，我们只传递标题到命令行，并等待响应。

不过，在这种情况下，我们需要获取当前的主题行，以便将其作为修改后的主题行的一部分传递给`formail`程序。我们通过对主题内容进行简单匹配，然后将`$MATCH`变量（现在保存主题行内容）作为`formail`程序的参数传递。为了整洁起见，我们在原始主题行之前添加`{* -BIG- *}`的措辞，以便轻松排序和识别这些消息。

```
#ReWrite the subject line
:0 fhw
* ^Subject:\/.*
| formail -I "Subject: {* -BIG- *} $MATCH"

```

然后将进行消息的正常传递，并将新的较短消息存储在收件箱中。

如果我们将所有这些放在一起，我们最终得到以下完整的配方。

```
:0: * >100000
{
#Place a copy in the largemail folder
:0 c: largemail/
#Strip the body to 1kb :0 bfwi | /usr/bin/head -c1024 #ReWrite the subject line :0 fhw * ^Subject:\/.* | formail -I "Subject: {* -BIG- *} $MATCH" }

```

# Procmail 模块库

作为一个避免重复造轮子的社区努力的一部分，Procmail 模块库提供了一系列由 Procmail 用户贡献的有用配方。来自 Procmail 模块库的以下介绍[`freshmeat.net/projects/procmail-lib`](http://freshmeat.net/projects/procmail-lib)描述了该软件包：

> Procmail 模块库是 Procmail 邮件处理实用程序的许多插件模块的集合。这些模块允许常见任务，如解析日期、时间、MIME 和电子邮件地址，转发邮件，处理 POP3，垃圾邮件屏蔽，运行电子邮件 cron 作业，处理守护程序消息等。

每个模块或 Procmail 包含的文件都有全面的文档和示例用法。它们可以按原样使用，也可以使用各种可配置选项，或者作为您自己食谱的基础。我们在本章中演示的许多技术都在库中使用，还有一些基于消息内容类型的更复杂的过滤方法。

# 把所有东西整合起来

在本章中，我们涵盖了各种主题，现在我们可以把它们整合起来。以下示例使用了本章中展示的每种技术，并且通常用于电子邮件处理。我希望你能在创建自己的邮件过滤策略时找到它有用。

## 创建一个基于您自己规则的结构

将 Procmail 规则和配置的相关方面分组将使您的安装更容易维护，并在进行更改时更不太可能出现问题。

在主 Procmail 目录中，创建遵循一致命名约定的单独文件，例如`rc.main, rc.spam, rc.lists`等。然后将这些文件包含到您的主`.procmailrc`文件中，如下所示。

```
#This obtains the date formatted as YYYY MM DD date = `date "+%Y %m %d"`
#Now assign the Year YYYY style :0 * date ?? ^^()\/ { YYYY = $MATCH }
#Now assign the Year YY style :0 * date ?? ^^..\/ { YY = $MATCH }
#Now assign the Month MM style :0 * date ?? ^^.....\/ { MM = $MATCH }
#Now assign the Day DD style :0 * date ?? ()\/..^^ { DD = $MATCH }
#Create the various directory formats you are going to use
DUMMY=`test -d ${YYYY}/${MM}/${DD} || mkdir -p ${YYYY}/${MM}/${DD}`
DUMMY=`test -d ${YY}/${MM} || mkdir -p ${YY}/${MM}`
#Make a backup copy of all incoming mail
:0 c
backup/
#Restrict the history to just 32 mail items
:0 ic
| cd backup && rm -f dummy `ls -t msg.* | sed -e 1,32d`
#Make sure that all mails have a valid From value
:0 fhw
| formail -I "From " -a "From "
#
## Don't include this unless we need to
## INCLUDERC=${HOME}/Procmail/rc.testing
##
## Now include the various process listings
INCLUDERC=${HOME}/Procmail/rc.system
INCLUDERC=${HOME}/Procmail/rc.lists
INCLUDERC=${HOME}/Procmail/rc.killspam
INCLUDERC=${HOME}/Procmail/rc.vacation
INCLUDERC=${HOME}/Procmail/rc.largefiles
INCLUDERC=${HOME}/Procmail/rc.virusfilter
INCLUDERC=${HOME}/Procmail/rc.spamfilter

```

现在，对于列出的每个`include`文件，创建与该文件相关的规则，并将其包含在该文件中。然后，暂时隔离用于处理传入邮件的部分的`INCLUDERC`引用就成了问题。在生产环境中，要小心不要盲目地剪切和粘贴这些示例，而不检查每个食谱是否按预期执行。

### Rc.system

将信息系统和守护程序消息存档在一个带日期的文件夹结构中，可以给出如下：

```
# Filter system mail messages into a dated folder structure.
# The variables YY and MM are defined in the calling recipe
# and each of the directories will have been created if necessary.
:0:
* ^From:.*root@delta.adepteo.net
${YY}/${MM}/daemon/
:0:
* ^From:.*root@ramsbottom.adepteo.net
${YY}/${MM}/daemon/
:0:
* ^TO_pager@adepteo.net
${YY}/${MM}/daemon/
:0:
* ^From:.*MAILER-DAEMON@delta.adepteo.net
${YY}/${MM}/daemon/
:0:
* ^From:.*me@localhost.com
${YY}/${MM}/daemon/

```

### Rc.lists

将所有订阅的邮件列表存档在带日期的文件夹中以供以后阅读。

```
# Mailing lists
# Store by date folder
# The variables DD and MM are defined in the calling recipe.
# and each of the directories will have been created if necessary.
:0:
* ^From:.*mapserver-users-admin@lists.gis.umn.edu
${YY}/${MM}/mapserver/
:0:
* ^TO_mapserver-users@lists.gis.umn.edu
${YY}/${MM}/mapserver/
:0:
* ^From:.*yourtopjob@topjobs.co.uk
${YY}/${MM}/jobs/
:0:
* ^Subject: silicon Jobs-by-Email Alert
${YY}/${MM}/jobs/
:0:
* ^Reply-To: Axandra Search Engine Facts <facts@Axandra.com>
${YY}/${MM}/lists/
:0:
* ^Subject: A Joke A Day
${YY}/${MM}/lists/
:0:
* ^List-Owner: <mailto:owner-tribune@lists.sitepoint.com>
${YY}/${MM}/lists/
:0:
* ^Reply-To: newsletter@192.com
${YY}/${MM}/lists/
:0:
* ^Subject: Developer Shed Weekly Update
${YY}/${MM}/lists/

```

### Rc.killspam

删除来自与我们的屏蔽文件中地址匹配的发件人的任何邮件。

```
#Kill file for known spammers
# If the sender is in the killfile then discard the mail into the bit bucket
# Here we use the external command 'grep' to search our killfile for a
# matching sending sending by testing the return status from grep.
:0:
* ? grep -i `formail -rtzxTo:` $HOME/.killfile
/dev/null

```

### Rc.vacation

我们的假期自动回复食谱：

```
#Vacation Replies
:0 Whc: vacation.lock
# Perform a quick check to see if the mail was addressed to us
* $^To_:.*\<$\LOGNAME\>
# Don't reply to daemons and mailinglists
* !^FROM_DAEMON
# Mail loops are evil
* !^X-Loop: $RECIPIENT
| formail -rD 8192 vacation.cache
:0 ehc
# if the name was not in the cache reply with the contents
# of our vacation message in the body of the email.
| (
formail -rA"Precedence: junk" \
-A"X-Loop: $RECIPIENT" ; \
cat $HOME/.vacation_message \
) | $SENDMAIL -oi -t

```

### Rc.largefiles

为了避免用大量消息堵塞我们的收件箱，我们将大消息存档在一个文件夹中，并发送通知给自己，告知我们收到了一条超大消息。

```
#Assume that files larger than 100k are not spam
:0: * >100000
{
#Place a copy in the largemail folder
:0 c: largemail/
#Strip the body to 1kb :0 bfwi | /usr/bin/head -c1024
#ReWrite the subject line :0 fhw * ^Subject:\/.* | formail -I "Subject: {* -BIG- *} $MATCH" }

```

### Rc.viruses

任何带有表明该消息为病毒的电子邮件标题的内容，都放在一个文件夹中。

```
#Virus Filter
#X-Virus-Status: Infected
:0:
* ^X-Virus-Status: Infected
_virus/

```

### Rc.spamfilter

任何带有表明该消息为垃圾邮件的电子邮件标题的内容，都放在一个文件夹中。

```
#Spam Filter
:0fw
* < 256000
| spamc
# Mails with a score of 15 or higher are almost certainly
# spam (with 0.05% false positives according to
# rules/STATISTICS.txt). Let's put them in a
# different mbox. (This one is optional.)
#
# The regular expression below matches the SpamAssassin
# header with 15 asterisks or more.
#
:0:
* ^X-Spam-Level: \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*
_almost-certainly-spam/
# All mail tagged as spam (eg. with a score higher than the
# set threshold)
is moved to "probably-spam".
:0:
* ^X-Spam-Status: Yes
_probably-spam/

```

# 总结

在本章中，我们探索了 Procmail，发现了大量的服务和大量的功能，它可以提供帮助我们控制邮件。使用 Procmail 的高级功能，我们发现了：

+   交付和非交付食谱之间的区别

+   如何订购每个食谱以避免交货时间过长

+   使用 Procmail 变量和条件标志来控制交付

+   使用正则表达式进行复杂的模式匹配

+   可用的大量 Procmail 宏及其用法

+   最后，一些示例食谱来有效管理我们的邮件

虽然我们已经涵盖了很多内容，但仍有很多东西有待学习，网上有大量资源专门致力于这一特定应用程序。

希望现在您已经对 Procmail 的核心功能有了深刻的理解，知道如何实施它，以及如何探索您的现实需求，并创建食谱集，以便组合成您自己独特的邮件过滤策略。
