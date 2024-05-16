# 第七章：Hello World!

在本章中，我们将终于开始编写 shell 脚本。在编写和运行我们自己的`Hello World!`脚本之后，我们将研究一些适用于所有未来脚本的最佳实践。我们将使用许多技术来提高脚本的可读性，并在可能的情况下遵循 KISS 原则（保持简单，愚蠢）。

本章将介绍以下命令：`head`，`tail`和`wget`。

本章将涵盖以下主题：

+   第一步

+   可读性

+   KISS

# 技术要求

我们将直接在虚拟机上创建我们的 shell 脚本；我们暂时不会使用 Atom/Notepad++。

本章的所有脚本都可以在 GitHub 上找到：[`github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter07`](https://github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter07)。

# 第一步

在获取有关 Linux 的一些背景信息，准备我们的系统，并了解 Linux 脚本编写的重要概念之后，我们终于到达了我们将编写实际 shell 脚本的地步！

总之，shell 脚本只不过是多个 Bash 命令的顺序排列。脚本通常用于自动化重复的任务。它们可以交互式或非交互式地运行（即带有或不带有用户输入），并且可以与他人共享。让我们创建我们的`Hello World`脚本！我们将在我们的`home`目录中创建一个文件夹，用于存储每个章节的所有脚本：

```
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader  0 Aug 19 11:54 emptyfile
-rw-rw-r-- 1 reader reader 23 Aug 19 11:54 textfile.txt
reader@ubuntu:~$ mkdir scripts
reader@ubuntu:~$ cd scripts/
reader@ubuntu:~/scripts$ mkdir chapter_07
reader@ubuntu:~/scripts$ cd chapter_07/
reader@ubuntu:~/scripts/chapter_07$ vim hello-world.sh
```

接下来，在`vim`屏幕中，输入以下文本（注意我们在两行之间*使用了空行*）：

```
#!/bin/bash

echo "Hello World!"
```

正如我们之前解释的，`echo`命令将文本打印到终端。让我们使用`bash`命令运行脚本：

```
reader@ubuntu:~/scripts/chapter_07$ bash hello-world.sh
Hello World!
reader@ubuntu:~/scripts/chapter_07
```

恭喜，你现在是一个 shell 脚本编写者！也许还不是一个非常优秀或全面的*编写者*，但无论如何都是一个 shell 脚本编写者。

请记住，如果`vim`还没有完全满足你的需求，你可以随时退回到`nano`。或者，更好的是，再次运行`vimtutor`并刷新那些`vim`操作！

# shebang

你可能想知道第一行是什么意思。第二行（或者第三行，如果你算上空行的话）应该很清楚，但第一行是新的。它被称为**shebang**，有时也被称为*sha-bang*，*hashbang*，*pound-bang*和/或*hash-pling*。它的功能非常简单：它告诉系统使用哪个二进制文件来执行脚本。它的格式始终是`#!<binary path>`。对于我们的目的，我们将始终使用`#!/bin/bash` shebang，但对于 Perl 或 Python 脚本，分别是`#!/usr/bin/perl`和`#!/usr/bin/python3`。乍一看，这似乎是不必要的。我们创建了名为`hello-world.sh`的脚本，而 Perl 或 Python 脚本将使用`hello-world.pl`和`hello-world.py`。那么，为什么我们需要 shebang 呢？

对于 Python，它允许我们轻松区分 Python 2 和 Python 3。通常情况下，人们会期望尽快切换到编程语言的新版本，但对于 Python 来说，这似乎需要付出更多的努力，这就是为什么今天我们会看到 Python 2 和 Python 3 同时在使用中的原因。

Bash 脚本不以`.bash`结尾，而是以`.sh`结尾，这是*shell*的一般缩写。因此，除非我们为 Bash 指定 shebang，否则我们将以*正常*的 shell 执行结束。虽然对于一些脚本来说这没问题（`hello-world.sh`脚本将正常工作），但当我们使用 Bash 的高级功能时，就会遇到问题。

# 运行脚本

如果您真的留心观察，您会注意到我们执行了一个没有可执行权限的脚本，使用了`bash`命令。如果我们已经指定了如何运行它，为什么还需要 shebang 呢？在这种情况下，我们不需要 shebang。但是，我们需要确切地知道它是哪种类型的脚本，并找到系统上正确的二进制文件来运行它，这可能有点麻烦，特别是当您有很多脚本时。幸运的是，我们有更好的方法来运行这些脚本：使用可执行权限。让我们看看如何通过设置可执行权限来运行我们的`hello-world.sh`脚本：

```
reader@ubuntu:~/scripts/chapter_07$ ls -l
total 4
-rw-rw-r-- 1 reader reader 33 Aug 26 12:08 hello-world.sh
reader@ubuntu:~/scripts/chapter_07$ ./hello-world.sh
-bash: ./hello-world.sh: Permission denied
reader@ubuntu:~/scripts/chapter_07$ chmod +x hello-world.sh 
reader@ubuntu:~/scripts/chapter_07$ ./hello-world.sh
Hello World! reader@ubuntu:~/scripts/chapter_07$ /home/reader/scripts/chapter_07/hello-world.sh Hello World!
reader@ubuntu:~/scripts/chapter_07$ ls -l
total 4
-rwxrwxr-x 1 reader reader 33 Aug 26 12:08 hello-world.sh
reader@ubuntu:~/scripts/chapter_07$
```

我们可以通过运行*完全限定*或在相同目录中使用`./`来执行脚本（或任何文件，只要对于该文件来说有意义）。只要设置了可执行权限，我们就需要前缀`./`。这是因为安全性的原因：通常当我们执行一个命令时，`PATH`变量会被探索以找到该命令。现在想象一下，有人在您的主目录中放置了一个恶意的名为`ls`的二进制文件。如果没有`./`规则，运行`ls`命令将导致运行该二进制文件，而不是`/bin/ls`（它在您的`PATH`上）。

因为我们只是使用`./hello-world.sh`来运行脚本，所以现在我们需要再次使用 shebang。否则，Linux 会默认使用`/bin/sh`，这不是我们在**Bash**脚本书中想要的，对吧？

# 可读性

在编写 shell 脚本时，您应该始终确保代码尽可能易读。当您正在创建脚本时，所有逻辑、命令和脚本流程对您来说可能是显而易见的，但如果您一段时间后再看脚本，这就不再是显而易见的了。更糟糕的是，您很可能会与其他人一起编写脚本；这些人在编写脚本时从未考虑过您的考虑（反之亦然）。我们如何在脚本中促进更好的可读性呢？注释和冗长是我们实现这一目标的两种方式。

# 注释

任何优秀的软件工程师都会告诉您，在代码中放置相关注释会提高代码的质量。注释只不过是一些解释您在做什么的文本，前面加上一个特殊字符，以确保您编写代码的语言不会解释这些文本。对于 Bash 来说，这个字符是*井号* `#`（目前更为人所熟知的是在#HashTags 中的使用）。在阅读其他来源时，它也可能被称为*井号*或*哈希*。其他注释字符的例子包括`//`（Java，C++），`--`（SQL），以及`<!-- comment here -->`（HTML，XML）。`#`字符也被用作 Python 和 Perl 的注释。

注释可以放在行的开头，以确保整行不被解释，或者放在行的其他位置。在这种情况下，直到`#`之前的所有内容都将被处理。让我们看一个修订后的`Hello World`脚本中这两种情况的例子。

```
#!/bin/bash

# Print the text to the Terminal.
echo "Hello World!"
```

或者，我们可以使用以下语法：

```
#!/bin/bash

echo "Hello World!" # Print the text to the Terminal.
```

一般来说，我们更喜欢将注释放在命令的上面单独的一行。然而，一旦我们引入循环、重定向和其他高级结构，*内联注释*可以确保比整行注释更好的可读性。然而，最重要的是：**任何相关的注释总比没有注释更好，无论是整行还是内联**。按照惯例，我们总是更喜欢保持注释非常简短（一到三个单词）或者使用带有适当标点的完整句子。在需要简短句子会显得过于夸张的情况下，使用一些关键词；否则，选择完整句子。我们保证这将使您的脚本看起来更加专业。

# 脚本头

在我们的脚本编写中，我们总是在脚本开头包含一个*标题*。虽然这对于脚本的功能来说并不是必需的，但当其他人使用您的脚本时（或者再次，当您使用其他人的脚本时），它可以帮助很大。标题可以包括您认为需要的任何信息，但通常我们总是从以下字段开始：

+   作者

+   版本

+   日期

+   描述

+   用法

通过使用注释实现简单的标题，我们可以让偶然发现脚本的人了解脚本是何时编写的，由谁编写的（如果他们有问题的话）。此外，简单的描述为脚本设定了一个目标，使用信息确保首次使用脚本时不会出现试错。让我们创建`hello-world.sh`脚本的副本，将其命名为`hello-world-improved.sh`，并实现标题和功能的注释：

```
reader@ubuntu:~/scripts/chapter_07$ ls -l
total 4
-rwxrwxr-x 1 reader reader 33 Aug 26 12:08 hello-world.sh
reader@ubuntu:~/scripts/chapter_07$ cp hello-world.sh hello-world-improved.sh
reader@ubuntu:~/scripts/chapter_07$ vi hello-world-improved.sh
```

确保脚本看起来像下面这样，但一定要输入*当前日期*和*您自己的名字*：

```
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-08-26
# Description: Our first script!
# Usage: ./hello-world-improved.sh
#####################################

# Print the text to the Terminal.
echo "Hello World!"
```

现在，看起来不错吧？唯一可能突出的是，我们现在有一个包含任何功能的 12 行脚本。在这种情况下，的确，这似乎有点过分。然而，我们正在努力学习良好的实践。一旦脚本变得更加复杂，我们用于 shebang 和标题的这 10 行将不会有任何影响，但可用性显著提高。顺便说一下，我们正在引入一个新的`head`命令。

```
reader@ubuntu:~/scripts/chapter_07$ head hello-world-improved.sh
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0

# Date: 2018-08-26
# Description: Our first script!
# Usage: ./hello-world-improved.sh
#####################################

reader@ubuntu:~/scripts/chapter_07$
```

`head`命令类似于`cat`，但它不会打印整个文件；默认情况下，它只打印前 10 行。巧合的是，这恰好与我们创建的标题长度相同。因此，任何想要使用您的脚本的人（老实说，**您**在 6 个月后也是*任何人*）只需使用`head`打印标题，并获取开始使用脚本所需的所有信息。

在引入`head`的同时，如果我们不介绍`tail`也是不负责任的。正如名称可能暗示的那样，`head`打印文件的顶部，而`tail`打印文件的末尾。虽然这对我们的脚本标题没有帮助，但在查看错误或警告的日志文件时非常有用：

```
reader@ubuntu:~/scripts/chapter_07$ tail /var/log/auth.log
Aug 26 14:45:28 ubuntu systemd-logind[804]: Watching system buttons on /dev/input/event1 (Sleep Button)
Aug 26 14:45:28 ubuntu systemd-logind[804]: Watching system buttons on /dev/input/event2 (AT Translated Set 2 keyboard)
Aug 26 14:45:28 ubuntu sshd[860]: Server listening on 0.0.0.0 port 22.
Aug 26 14:45:28 ubuntu sshd[860]: Server listening on :: port 22.
Aug 26 15:00:02 ubuntu sshd[1079]: Accepted password for reader from 10.0.2.2 port 51752 ssh2
Aug 26 15:00:02 ubuntu sshd[1079]: pam_unix(sshd:session): session opened for user reader by (uid=0)
Aug 26 15:00:02 ubuntu systemd: pam_unix(systemd-user:session): session opened for user reader by (uid=0)
Aug 26 15:00:02 ubuntu systemd-logind[804]: New session 1 of user reader.
Aug 26 15:17:01 ubuntu CRON[1272]: pam_unix(cron:session): session opened for user root by (uid=0)
Aug 26 15:17:01 ubuntu CRON[1272]: pam_unix(cron:session): session closed for user root
reader@ubuntu:~/scripts/chapter_07$
```

# 冗长

回到我们如何改善脚本的可读性。虽然注释是改善我们对脚本理解的好方法，但如果脚本中的命令使用许多晦涩的标志和选项，我们需要在注释中使用许多词语来解释一切。而且，正如您可能期望的那样，如果我们需要五行注释来解释我们的命令，那么可读性会降低而不是提高！冗长是在不要太多但也不要太少的解释之间取得平衡。例如，您可能不必向任何人解释您是否以及为什么使用`ls`命令，因为那是非常基本的。然而，`tar`命令可能相当复杂，因此简短地解释您要实现的目标可能是值得的。

在这种情况下，我们想讨论三种类型的冗长。它们分别是：

+   注释的冗长

+   命令的冗长

+   命令输出的冗长

# 注释的冗长

冗长的问题在于很难给出明确的规则。几乎总是非常依赖于上下文。因此，虽然我们可以说，确实，我们不必评论`echo`或`ls`，但情况并非总是如此。假设我们使用`ls`命令的输出来迭代一些文件；也许我们想在注释中提到这一点？或者甚至这种情况对我们的预期读者来说是如此清晰，以至于对整个循环进行简短的评论就足够了？

答案是，非常不令人满意，*这取决于情况*。如果您不确定，通常最好还是包括注释，但您可能希望保持更加简洁。例如，您可以选择*使用 ls 构建迭代列表*，而不是*此 ls 实例列出所有文件，然后我们可以用它来进行脚本的其余部分的迭代*。这在很大程度上是一种实践技能，所以一定要至少开始练习：随着您编写更多的 shell 脚本，您肯定会变得更好。

# 命令的冗长

命令输出的冗长是一个有趣的问题。在之前的章节中，您已经了解了许多命令，有时还有相应的标志和选项，可以改变该命令的功能。大多数选项都有短语法和长语法，可以实现相同的功能。以下是一个例子：

```
reader@ubuntu:~$ ls -R
.:
emptyfile  scripts  textfile.txt
./scripts:
chapter_07
./scripts/chapter_07:
hello-world-improved.sh  hello-world.sh
reader@ubuntu:~$ ls --recursive
.:
emptyfile  scripts  textfile.txt
./scripts:
chapter_07
./scripts/chapter_07:
hello-world-improved.sh  hello-world.sh
reader@ubuntu:~$
```

我们使用`ls`递归打印我们的主目录中的文件。我们首先使用简写选项`-R`，然后使用长`--recursive`变体。从输出中可以看出，命令完全相同，即使`-R`更短且输入更快。但是，`--recursive`选项更冗长，因为它比`-R`给出了更好的提示，说明我们在做什么。那么，何时使用哪个？简短的答案是：**在日常工作中使用简写选项，在编写脚本时使用长选项**。虽然这对大多数情况都适用，但这并不是一个绝对可靠的规则。有些简写命令使用得非常普遍，以至于使用长选项可能会更令读者困惑，尽管听起来有些违反直觉。例如，在使用 SELinux 或 AppArmor 时，`ls`的`-Z`命令会打印安全上下文。这个的长选项是`--context`，但是这个选项没有`-Z`选项那么出名（根据我们的经验）。在这种情况下，使用简写会更好。

然而，我们已经看到了一个复杂的命令，但是当我们使用长选项时，它会更加可读：`tar`。让我们看看创建存档的两种方法：

```
reader@ubuntu:~/scripts/chapter_07$ ls -l
total 8
-rwxrwxr-x 1 reader reader 277 Aug 26 15:13 hello-world-improved.sh
-rwxrwxr-x 1 reader reader  33 Aug 26 12:08 hello-world.sh
reader@ubuntu:~/scripts/chapter_07$ tar czvf hello-world.tar.gz hello-world.sh
hello-world.sh
reader@ubuntu:~/scripts/chapter_07$ tar --create --gzip --verbose --file hello-world-improved.tar.gz hello-world-improved.sh
hello-world-improved.sh
reader@ubuntu:~/scripts/chapter_07$ ls -l
total 16
-rwxrwxr-x 1 reader reader 277 Aug 26 15:13 hello-world-improved.sh
-rw-rw-r-- 1 reader reader 283 Aug 26 16:28 hello-world-improved.tar.gz
-rwxrwxr-x 1 reader reader  33 Aug 26 12:08 hello-world.sh
-rw-rw-r-- 1 reader reader 317 Aug 26 16:26 hello-world.tar.gz
reader@ubuntu:~/scripts/chapter_07$
```

第一个命令`tar czvf`只使用了简写。这样的命令非常适合作为完整的行注释或内联注释：

```
#!/bin/bash
<SNIPPED>
# Verbosely create a gzipped tarball.
tar czvf hello-world.tar.gz hello-world.sh
```

或者，您可以使用以下内容：

```
#!/bin/bash
<SNIPPED>
# Verbosely create a gzipped tarball.
tar czvf hello-world.tar.gz hello-world.sh
```

`tar --create --gzip --verbose --file` 命令本身已经足够冗长，不需要注释，因为适当的注释实际上与长选项所表达的意思相同！

简写用于节省时间。对于日常任务来说，这是与系统交互的好方法。但是，在 shell 脚本中，清晰和冗长更为重要。使用长选项是一个更好的主意，因为使用这些选项时可以避免额外的注释。然而，一些命令使用得非常频繁，以至于长标志实际上可能更加令人困惑；在这里要根据您的最佳判断，并从经验中学习。

# 命令输出的冗长

最后，当运行 shell 脚本时，您将看到脚本中命令的输出（除非您想使用*重定向*来删除该输出，这将在第十二章中解释，*在脚本中使用管道和重定向*）。一些命令默认是冗长的。这些命令的很好的例子是`ls`和`echo`命令：它们的整个功能就是在屏幕上打印一些东西。

如果我们回到`tar`命令，我们可以问自己是否需要看到正在存档的所有文件。如果脚本中的逻辑是正确的，我们可以假设正在存档正确的文件，并且这些文件的列表只会使脚本的其余输出变得混乱。默认情况下，`tar`不会打印任何内容；到目前为止，我们一直使用`-v`/`--verbose`选项。但是，对于脚本来说，这通常是不可取的行为，因此我们可以安全地省略此选项（除非我们有充分的理由不这样做）。

大多数命令默认具有适当的冗长性。`ls`的输出是打印的，但`tar`默认是隐藏的。对于大多数命令，可以通过使用`--verbose`或`--quiet`选项（或相应的简写，通常是`-v`或`-q`）来反转冗长性。`wget`就是一个很好的例子：这个命令用于从互联网上获取文件。默认情况下，它会输出大量关于连接、主机名解析、下载进度和下载目的地的信息。然而，很多时候，所有这些东西都不是很有趣！在这种情况下，我们使用`wget`的`--quiet`选项，因为对于这种情况来说，这是命令的**适当冗长性**。

在编写 shell 脚本时，始终考虑所使用命令的冗长性。如果不够，查看 man 页面以找到增加冗长性的方法。如果太多，同样查看 man 页面以找到更安静的选项。我们遇到的大多数命令都有一个或两个选项，有时在不同的级别（`-q`和`-qq`甚至更安静的操作！）。

# 保持简单，愚蠢（KISS）

KISS 原则是处理 shell 脚本的一个很好的方法。虽然它可能显得有点严厉，但给出它的精神是重要的：它应该被视为很好的建议。*Python 之禅*中还给出了更多的建议，这是 Python 的设计原则：

+   简单胜于复杂

+   复杂比复杂好

+   可读性很重要

*Python 之禅*中还有大约 17 个方面，但这三个对于 Bash 脚本编写也是最相关的。最后一个，'*可读性很重要'*，现在应该是显而易见的。然而，前两个，'*简单胜于复杂'*和'*复杂胜于复杂'*与 KISS 原则密切相关。保持简单是一个很好的目标，但如果不可能，复杂的解决方案总是比复杂的解决方案更好（没有人喜欢复杂的脚本！）。

在编写脚本时，有一些事情你可以记住：

+   如果你正在构思的解决方案似乎变得非常复杂，请做以下任一事情：

+   研究你的问题；也许有另一个工具可以代替你现在使用的工具。

+   看看是否可以将事情分成离散的步骤，这样它会变得更复杂但不那么复杂。

+   问问自己是否需要一行代码完成所有操作，或者是否可能将命令拆分成多行以增加可读性。在使用管道或其他形式的重定向时，如第十二章中更详细地解释的那样，*在脚本中使用管道和重定向*，这是需要牢记的事情。

+   如果它起作用，那*可能*不是一个坏解决方案。但是，请确保解决方案不要*太*简单，因为边缘情况可能会在以后造成麻烦。

# 总结

我们从创建和运行我们的第一个 shell 脚本开始了这一章。学习一门新的软件语言时，几乎是强制性的，我们在终端上打印了 Hello World！接着，我们解释了 shebang：脚本的第一行，它是对 Linux 系统的一条指令，告诉它在运行脚本时应该使用哪个解释器。对于 Bash 脚本，约定是文件名以.sh 结尾，带有`#!/bin/bash`的 shebang。

我们解释了可以运行脚本的多种方式。我们可以从解释器开始，并将脚本名称作为参数传递（例如：`bash hello-world.sh`）。在这种情况下，shebang 是不需要的，因为我们在命令行上指定了解释器。然而，通常情况下，我们通过设置可执行权限并直接调用文件来运行脚本；在这种情况下，shebang 用于确定使用哪个解释器。因为你无法确定用户将如何运行你的脚本，包含 shebang 应该被视为强制性的。

为了提高我们脚本的质量，我们描述了如何提高我们 shell 脚本的可读性。我们解释了何时以及如何在我们的脚本中使用注释，以及如何使用注释创建一个我们可以通过使用`head`命令轻松查看的脚本头。我们还简要介绍了与`head`密切相关的`tail`命令。除了注释，我们还解释了**冗长性**的概念。

冗长性可以在多个级别找到：注释的冗长性，命令的冗长性和命令输出的冗长性。我们认为，在脚本中使用命令的长选项几乎总是比使用简写更好的主意，因为它增加了可读性，并且可以防止需要额外的注释，尽管我们已经确定，太多的注释几乎总是比没有注释更好。

我们以简要描述 KISS 原则结束了本章，我们将其与 Python 中的一些设计原则联系起来。读者应该意识到，如果有一个简单的解决方案，它往往是最好的。如果简单的解决方案不可行，应优先选择复杂的解决方案而不是复杂的解决方案。

本章介绍了以下命令：`head`，`tail`和`wget`。

# 问题

1.  按照惯例，当我们学习一门新的编程或脚本语言时，我们首先要做什么？

1.  Bash 的 shebang 是什么？

1.  为什么需要 shebang？

1.  我们可以以哪三种方式运行脚本？

1.  为什么我们在创建 shell 脚本时要如此强调可读性？

1.  为什么我们使用注释？

1.  为什么我们建议为您编写的所有 shell 脚本包括脚本头？

1.  我们讨论了哪三种冗长性类型？

1.  KISS 原则是什么？

# 进一步阅读

如果您想更深入地了解本章主题，以下资源可能会有趣：

+   **你好，世界（长教程）**：[`bash.cyberciti.biz/guide/Hello,_World!_Tutorial`](https://bash.cyberciti.biz/guide/Hello,_World!_Tutorial)

+   **Bash 编码风格指南**：[`bluepenguinlist.com/2016/11/04/bash-scripting-tutorial/`](https://bluepenguinlist.com/2016/11/04/bash-scripting-tutorial/)

+   **KISS**：[`people.apache.org/%7Efhanik/kiss.html`](https://people.apache.org/%7Efhanik/kiss.html)
