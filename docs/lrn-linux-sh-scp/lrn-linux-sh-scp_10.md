# 第十章：正则表达式

本章介绍了正则表达式以及我们可以用来利用其功能的主要命令。我们将首先了解正则表达式背后的理论，然后深入到使用`grep`和`sed`的正则表达式的实际示例中。

我们还将解释通配符及其在命令行上的使用方式。

本章将介绍以下命令：`grep`、`set`、`egrep`和`sed`。

本章将涵盖以下主题：

+   什么是正则表达式？

+   通配符

+   使用`egrep`和`sed`的正则表达式

# 技术要求

本章的所有脚本都可以在 GitHub 上找到：[`github.com/tammert/learn-linux-shell-scripting/tree/master/chapter_10`](https://github.com/tammert/learn-linux-shell-scripting/tree/master/chapter_10)。除此之外，Ubuntu 虚拟机仍然是我们在本章中测试和运行脚本的方式。

# 介绍正则表达式

您可能以前听说过*正则表达式*或*regex*这个术语。对于许多人来说，正则表达式似乎非常复杂，通常是从互联网或教科书中摘取的，而没有完全掌握它的作用。

虽然这对于完成一项任务来说是可以的，但是比普通系统管理员更好地理解正则表达式可以让你在创建脚本和在终端上工作时脱颖而出。

一个精心设计的正则表达式可以帮助您保持脚本简短、简单，并且能够适应未来的变化。

# 什么是正则表达式？

实质上，正则表达式是一段*文本*，它作为其他文本的*搜索模式*。正则表达式使得很容易地说，例如，我想选择所有包含五个字符的单词的行，或者查找所有以`.log`结尾的文件。

一个示例可能有助于您的理解。首先，我们需要一个可以用来探索正则表达式的命令。在 Linux 中与正则表达式一起使用的最著名的命令是`grep`。

`grep`是一个缩写，意思是***g**lobal **r**egular **e**xpression **p**rint*。您可以看到，这似乎是解释这个概念的一个很好的候选者！

# grep

我们将按以下方式立即深入：

```
reader@ubuntu:~/scripts/chapter_10$ vim grep-file.txt
reader@ubuntu:~/scripts/chapter_10$ cat grep-file.txt 
We can use this regular file for testing grep.
Regular expressions are pretty cool
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ grep 'cool' grep-file.txt 
Regular expressions are pretty cool
reader@ubuntu:~/scripts/chapter_10$ cat grep-file.txt | grep 'USA'
but in the USA they use color (and realize)!
```

首先，让我们探索`grep`的基本功能，然后再深入到正则表达式。`grep`的功能非常简单，如`man grep`中所述：*打印匹配模式的行*。

在前面的示例中，我们创建了一个包含一些句子的文件。其中一些以大写字母开头；它们大多以不同的方式结束；它们使用一些相似但不完全相同的单词。这些特征以及更多特征将在后续示例中使用。

首先，我们使用`grep`来匹配一个单词（默认情况下搜索区分大小写），并打印出来。`grep`有两种操作模式：

+   `grep <pattern> <file>`

+   `grep <pattern>`（需要以管道或`|`的形式输入）

第一种操作模式允许您指定一个文件名，从中您想要指定需要打印的行，如果它们匹配您指定的模式。`grep 'cool' grep-file.txt`命令就是一个例子。

还有另一种使用`grep`的方式：在流中。流是指*在传输中*到达您的终端的东西，但在移动过程中可以被更改。在这种情况下，对文件的`cat`通常会将所有行打印到您的终端上。

然而，通过管道符号（`|`），我们将`cat`的输出重定向到`grep`；在这种情况下，我们只需要指定要匹配的模式。任何不匹配的行将被丢弃，并且不会显示在您的终端上。

正如您所看到的，完整的语法是`cat grep-file.txt | grep 'USA'`。

管道是一种重定向形式，我们将在第十二章中进一步讨论，*在脚本中使用管道和重定向*。现在要记住的是，通过使用管道，`cat`的*输出*被用作`grep`的*输入*，方式与文件名被用作输入相同。在讨论`grep`时，我们（暂时）将使用首先解释的不使用重定向的方法。

因为单词*cool*和*USA*只在一行中找到，所以`grep`的两个实例都只打印那一行。但是如果一个单词在多行中找到，`grep`会按照它们遇到的顺序（通常是从上到下）打印它们：

```
reader@ubuntu:~/scripts/chapter_10$ grep 'use' grep-file.txt 
We can use this regular file for testing grep.
but in the USA they use color (and realize)!
```

使用`grep`，可以指定我们希望搜索是不区分大小写的，而不是默认的区分大小写的方法。例如，这是在日志文件中查找错误的一个很好的方法。一些程序使用单词*error*，其他使用*ERROR*，我们甚至偶尔会遇到*Error*。通过向`grep`提供`-i`标志，所有这些结果都可以返回：

```
reader@ubuntu:~/scripts/chapter_10$ grep 'regular' grep-file.txt 
We can use this regular file for testing grep.
reader@ubuntu:~/scripts/chapter_10$ grep -i 'regular' grep-file.txt 
We can use this regular file for testing grep.
Regular expressions are pretty cool
```

通过提供`-i`，我们现在看到了*regular*和*Regular*都已经匹配，并且它们的行已经被打印出来。

# 贪婪性

默认情况下，正则表达式被认为是贪婪的。这可能看起来是一个奇怪的术语来描述一个技术概念，但它确实非常合适。为了说明为什么正则表达式被认为是贪婪的，看看这个例子：

```
reader@ubuntu:~/scripts/chapter_10$ grep 'in' grep-file.txt 
We can use this regular file for testing grep.
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
reader@ubuntu:~/scripts/chapter_10$ grep 'the' grep-file.txt 
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
```

正如你所看到的，`grep`默认情况下不会寻找完整的单词。它查看文件中的字符，如果一个字符串匹配搜索（不管它们之前或之后是什么），那么该行就会被打印出来。

在第一个例子中，`in`匹配了正常的单词**in**，但也匹配了 test**in**g。在第二个例子中，两行都有两个匹配项，**the**和**the**y。

如果你只想返回整个单词，请确保在`grep`搜索模式中包含空格：

```
reader@ubuntu:~/scripts/chapter_10$ grep ' in ' grep-file.txt 
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
reader@ubuntu:~/scripts/chapter_10$ grep ' the ' grep-file.txt 
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
```

正如你所看到的，现在对' in '的搜索并没有返回包含单词**testing**的行，因为字符**in**没有被空格包围。

正则表达式只是一个特定搜索模式的定义，它在个别脚本/编程语言中的实现方式是不同的。我们在 Bash 中使用的正则表达式与 Perl 或 Java 中使用的不同。在一些语言中，贪婪性可以被调整甚至关闭，但是`grep`和`sed`下的正则表达式总是贪婪的。这并不是一个问题，只是在定义搜索模式时需要考虑的事情。

# 字符匹配

我们现在知道了如何搜索整个单词，即使我们对大写和小写不是很确定。

我们还看到，（大多数）Linux 应用程序下的正则表达式是贪婪的，因此我们需要确保通过指定空格和字符锚点来正确处理这一点，我们将很快解释。

在这两种情况下，我们知道我们在寻找什么。但是如果我们真的不知道我们在寻找什么，或者可能只知道一部分呢？这个困境的答案是字符匹配。

在正则表达式中，有两个字符可以用作其他字符的替代品：

+   `.`（点）匹配任何一个字符（除了换行符）

+   `*`（星号）匹配前面字符的任意重复次数（甚至零次）

一个例子将有助于理解这一点：

```
reader@ubuntu:~/scripts/chapter_10$ vim character-class.txt 
reader@ubuntu:~/scripts/chapter_10$ cat character-class.txt 
eee
e2e
e e
aaa
a2a
a a
aabb
reader@ubuntu:~/scripts/chapter_10$ grep 'e.e' character-class.txt 
eee
e2e
e e
reader@ubuntu:~/scripts/chapter_10$ grep 'aaa*' character-class.txt 
aaa
aabb
reader@ubuntu:~/scripts/chapter_10$ grep 'aab*' character-class.txt 
aaa
aabb
```

在那里发生了很多事情，其中一些可能会感觉非常违反直觉。我们将逐一讨论它们，并详细说明发生了什么：

```
reader@ubuntu:~/scripts/chapter_10$ grep 'e.e' character-class.txt 
eee
e2e
e e
```

在这个例子中，我们使用点来替代*任何字符*。正如我们所看到的，这包括字母（e**e**e）和数字（e**2**e）。但是，它也匹配了最后一行上两个 e 之间的空格字符。

这里是另一个例子：

```
reader@ubuntu:~/scripts/chapter_10$ grep 'aaa*' character-class.txt 
aaa
aabb
```

当我们使用`*`替代时，我们正在寻找**零个或多个**前面的字符。在搜索模式`aaa*`中，这意味着以下字符串是有效的：

+   `aa`

+   `aaa`

+   `aaaa`

+   `aaaaa`

...等等。在第一个结果之后的一切都应该是清楚的，为什么`aa`也匹配`aaa*`呢？因为*零或更多*中的零！在这种情况下，如果最后的`a`是零，我们只剩下`aa`。

在最后一个例子中发生了同样的事情：

```
reader@ubuntu:~/scripts/chapter_10$ grep 'aab*' character-class.txt 
aaa
aabb
```

模式`aab*`匹配**aa**a 中的 aa，因为`b*`可以是零，这使得模式最终变成`aa`。当然，它也匹配一个或多个 b（`aabb`完全匹配）。

当你对你要找的东西只有一个大概的想法时，这些通配符就非常有用。然而，有时你会对你需要的东西有更具体的想法。

在这种情况下，我们可以使用括号[...]来缩小我们的替换范围到某个字符集。以下示例应该让你对如何使用这个有一个很好的想法：

```
reader@ubuntu:~/scripts/chapter_10$ grep 'f.r' grep-file.txt 
We can use this regular file for testing grep.
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ grep 'f[ao]r' grep-file.txt 
We can use this regular file for testing grep.
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ grep 'f[abcdefghijklmnopqrstuvwxyz]r' grep-file.txt 
We can use this regular file for testing grep.
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ grep 'f[az]r' grep-file.txt 
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ grep 'f[a-z]r' grep-file.txt 
We can use this regular file for testing grep.
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ grep 'f[a-k]r' grep-file.txt 
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ grep 'f[k-q]r' grep-file.txt 
We can use this regular file for testing grep
```

首先，我们演示使用`.`（点）来替换任何字符。在这种情况下，模式**f.r**匹配**for**和**far**。

接下来，我们在`f[ao]r`中使用括号表示法，以表明我们将接受一个在`f`和`r`之间的单个字符，它在`ao`的字符集中。不出所料，这又返回了**far**和**for**。

如果我们用`f[az]r`模式来做这个，我们只能匹配**far**和**fzr**。由于字符串`fzr`不在我们的文本文件中（显然也不是一个单词），我们只看到打印出**far**的那一行。

接下来，假设你想匹配一个字母，但不是一个数字。如果你使用`.`（点）进行搜索，就像第一个例子中那样，这将返回字母和数字。因此，你也会得到，例如，**f2r**作为匹配（如果它在文件中的话，实际上并不是）。

如果你使用括号表示法，你可以使用以下表示法：`f[abcdefghijklmnopqrstuvwxyz]r`。这匹配`f`和`r`之间的任何字母 a-z。然而，在键盘上输入这个并不好（相信我）。

幸运的是，POSIX 正则表达式的创建者引入了一个简写：`[a-z]`，就像前面的例子中所示的那样。我们也可以使用字母表的一个子集，如：`f[a-k]r`。由于字母**o**不在 a 和 k 之间，它不匹配**for**。

最后，一个例子证明了这是一个强大而实用的模式：

```
reader@ubuntu:~/scripts/chapter_10$ grep reali[sz]e grep-file.txt 
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
```

希望这一切仍然是有意义的。在转向行锚之前，我们将进一步结合表示法。

在前面的例子中，你看到我们可以使用括号表示法来处理美式英语和英式英语之间的一些差异。然而，这只有在拼写的差异是一个字母时才有效，比如 realise/realize。

在颜色/colour 的情况下，有一个额外的字母我们需要处理。这听起来像是一个零或更多的情况，不是吗？

```
reader@ubuntu:~/scripts/chapter_10$ grep 'colo[u]*r' grep-file.txt 
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
```

通过使用模式`colo[u]*r`，我们搜索包含以**colo**开头的单词的行，可能包含任意数量的**u**，并以**r**结尾。由于`color`和`colour`都适用于这个模式，两行都被打印出来。

你可能会想要使用点字符和零或更多的`*`表示法。然而，仔细看看在这种情况下会发生什么：

```
reader@ubuntu:~/scripts/chapter_10$ grep 'colo.*r' grep-file.txt 
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
```

再次，两行都匹配。但是，由于第二行中包含另一个**r**，所以字符串`color (and r`被匹配，以及`colour`和`color`。

这是一个典型的例子，正则表达式模式对我们的目的来说太贪婪了。虽然我们不能告诉它变得不那么贪婪，但`grep`中有一个选项，让我们只寻找匹配的单词。

表示法`-w`评估空格和行尾/行首，以便只找到完整的单词。用法如下：

```
reader@ubuntu:~/scripts/chapter_10$ grep -w 'colo.*r' grep-file.txt 
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
```

现在，只有单词`colour`和`color`被匹配。之前，我们在单词周围放置了空格以促进这种行为，但由于单词`colour`在行尾，它后面没有空格。

自己尝试一下，看看为什么用`colo.*r`搜索模式括起来不起作用，但使用`-w`选项却起作用。

一些正则表达式的实现有`{3}`表示法，用来补充`*`表示法。在这种表示法中，你可以精确指定模式应该出现多少次。搜索模式`[a-z]{3}`将匹配所有恰好三个字符的小写字符串。在 Linux 中，这只能用扩展的正则表达式来实现，我们将在本章后面看到。

# 行锚

我们已经简要提到了行锚。根据我们目前为止提出的解释，我们只能在一行中搜索单词；我们还不能设置对单词在行中的位置的期望。为此，我们使用行锚。

在正则表达式中，`^`（插入符）字符表示行的开头，`$`（美元）表示行的结尾。我们可以在搜索模式中使用这些，例如，在以下情况下：

+   查找单词 error，但只在行的开头：`^error`

+   查找以句点结尾的行：`\.$`

+   查找空行：`^$`

第一个用法，查找行的开头，应该是很清楚的。下面的例子使用了`grep -i`（记住，这允许我们不区分大小写地搜索），展示了我们如何使用这个来按行位置进行过滤：

```
reader@ubuntu:~/scripts/chapter_10$ grep -i 'regular' grep-file.txt 
We can use this regular file for testing grep.
Regular expressions are pretty cool
reader@ubuntu:~/scripts/chapter_10$ grep -i '^regular' grep-file.txt 
Regular expressions are pretty cool
```

在第一个搜索模式`regular`中，我们返回了两行。这并不意外，因为这两行都包含单词*regular*（尽管大小写不同）。

现在，为了只选择以单词*Regular*开头的行，我们使用插入符字符`^`来形成模式`^regular`。这只返回单词在该行的第一个位置的行。（请注意，如果我们没有选择在`grep`上包括`-i`，我们可以使用`[Rr]egular`代替。）

下一个例子，我们查找以句点结尾的行，会有点棘手。你会记得，在正则表达式中，句点被认为是一个特殊字符；它是任何其他一个字符的替代。如果我们正常使用它，我们会看到文件中的所有行都返回（因为所有行都以*任何一个字符*结尾）。

要实际搜索文本中的句点，我们需要**转义**句点，即用反斜杠前缀它；这告诉正则表达式引擎不要将句点解释为特殊字符，而是搜索它：

```
reader@ubuntu:~/scripts/chapter_10$ grep '.$' grep-file.txt 
We can use this regular file for testing grep.
Regular expressions are pretty cool
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ grep '\.$' grep-file.txt 
We can use this regular file for testing grep.
Also, New Zealand is pretty far away.
```

由于`\`用于转义特殊字符，你可能会遇到在文本中寻找反斜杠的情况。在这种情况下，你可以使用反斜杠来转义反斜杠的特殊功能！在这种情况下，你的模式将是`\\`，它与`\`字符串匹配。

在这个例子中，我们遇到了另一个问题。到目前为止，我们总是用单引号引用所有模式。然而，并不总是需要这样！例如，`grep cool grep-file.txt` 和 `grep 'cool' grep-file.txt` 一样有效。

那么，我们为什么要这样做呢？提示：尝试前面的例子，使用点行结束，不用引号。然后记住，在 Bash 中，美元符号也用于表示变量。如果我们引用它，Bash 将不会扩展`$`，这将返回问题结果。

我们将在第十六章中讨论 Bash 扩展，*Bash 参数替换和扩展*。

最后，我们介绍了`^$`模式。这搜索一个行的开头，紧接着一个行的结尾。只有一种情况会发生这种情况：一个空行。

为了说明为什么你想要找到空行，让我们看一个新的`grep`标志：`-v`。这个标志是`--invert-match`的缩写，这应该给出一个关于它实际上做什么的好提示：它打印不匹配的行，而不是匹配的行。

通过使用`grep -v '^$' <文件名>`，你可以打印一个没有空行的文件。在一个随机的配置文件上试一试：

```
reader@ubuntu:/etc$ cat /etc/ssh/ssh_config 

# This is the ssh client system-wide configuration file.  See
# ssh_config(5) for more information.  This file provides defaults for
# users, and the values can be changed in per-user configuration files
# or on the command line.

# Configuration data is parsed as follows:
<SNIPPED>
reader@ubuntu:/etc$ grep -v '^$' /etc/ssh/ssh_config 
# This is the ssh client system-wide configuration file.  See
# ssh_config(5) for more information.  This file provides defaults for
# users, and the values can be changed in per-user configuration files
# or on the command line.
# Configuration data is parsed as follows:
<SNIPPED>
```

正如你所看到的，`/etc/ssh/ssh_config` 文件以一个空行开头。然后，在注释块之间，还有另一行空行。通过使用 `grep -v '^$'`，这些空行被移除了。虽然这是一个不错的练习，但这并没有真正为我们节省多少行。

然而，有一个搜索模式是广泛使用且非常强大的：过滤配置文件中的注释。这个操作可以快速概述实际配置了什么，并省略所有注释（尽管注释本身也有其价值，但在你只想看到配置选项时可能会妨碍）。

为了做到这一点，我们将行首的插入符号与井号结合起来，表示注释：

```
reader@ubuntu:/etc$ grep -v '^#' /etc/ssh/ssh_config 

Host *
    SendEnv LANG LC_*
    HashKnownHosts yes
    GSSAPIAuthentication yes
```

这仍然打印所有空行，但不再打印注释。在这个特定的文件中，共有 51 行，只有四行包含实际的配置指令！所有其他行要么是空的，要么包含注释。很酷，对吧？

使用 `grep`，也可以同时使用多个模式。通过使用这种方法，可以结合过滤空行和注释行，快速概述配置选项。使用 `-e` 选项定义多个模式。在这种情况下，完整的命令是 `grep -v -e '^$' -e '^#' /etc/ssh/ssh_config`。试试看！

# 字符类

我们现在已经看到了许多如何使用正则表达式的示例。虽然大多数事情都很直观，但我们也看到，如果我们想要过滤大写和小写字符串，我们要么必须为 `grep` 指定 `-i` 选项，要么将搜索模式从 `[a-z]` 更改为 `[a-zA-z]`。对于数字，我们需要使用 `[0-9]`。

有些人可能觉得这样工作很好，但其他人可能不同意。在这种情况下，可以使用另一种可用的表示法：`[[:pattern:]]`。

下一个例子同时使用了这种新的双括号表示法和旧的单括号表示法：

```
reader@ubuntu:~/scripts/chapter_10$ grep [[:digit:]] character-class.txt 
e2e
a2a
reader@ubuntu:~/scripts/chapter_10$ grep [0-9] character-class.txt 
e2e
a2a
```

正如你所看到的，这两种模式都导致相同的行：包含数字的行。同样的方法也适用于大写字符：

```
reader@ubuntu:~/scripts/chapter_10$ grep [[:upper:]] grep-file.txt 
We can use this regular file for testing grep.
Regular expressions are pretty cool
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ grep [A-Z] grep-file.txt 
We can use this regular file for testing grep.
Regular expressions are pretty cool
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
Also, New Zealand is pretty far away.
```

最终，使用哪种表示法是个人偏好的问题。不过，双括号表示法有一点值得一提：它更接近其他脚本/编程语言的实现。例如，大多数正则表达式实现使用 `\w`（单词）来选择字母，使用 `\d`（数字）来搜索数字。在 `\w` 的情况下，大写变体直观地是 `\W`。

为了方便起见，这里是一个包含最常见的 POSIX 双括号字符类的表格：

| **表示法** | **描述** | **单括号等效** |
| --- | --- | --- |
| `[[:alnum:]]` | 匹配小写字母、大写字母或数字 | [a-z A-Z 0-9] |
| `[[:alpha:]]` | 匹配小写字母和大写字母 | [a-z A-Z] |
| `[[:digit:]]` | 匹配数字 | [0-9] |
| `[[:lower:]]` | 匹配小写字母 | [a-z] |
| `[[:upper:]]` | 匹配大写字母 | [A-Z] |
| `[[:blank:]]` | 匹配空格和制表符 | [ \t] |

我们更喜欢使用双括号表示法，因为它更好地映射到其他正则表达式实现。在脚本中可以自由选择使用任何一种！但是，一如既往：确保你选择一种，并坚持使用它；不遵循标准会导致令人困惑的杂乱脚本。本书中的其余示例将使用双括号表示法。

# 通配符

我们现在已经掌握了正则表达式的基础知识。在 Linux 上，还有一个与正则表达式密切相关的主题：*通配符*。即使你可能没有意识到，你在本书中已经看到了通配符的示例。

更好的是，实际上你已经有很大的机会在实践中使用了*通配符模式*。如果在命令行上工作时，你曾经使用通配符字符 `*`，那么你已经在使用通配符！

# 什么是通配符？

简单地说，glob 模式描述了将通配符字符注入文件路径操作。所以，当你执行`cp * /tmp/`时，你将当前工作目录中的所有文件（不包括目录！）复制到`/tmp/`目录中。

`*`扩展到工作目录中的所有常规文件，然后所有这些文件都被复制到`/tmp/`中。

这是一个简单的例子：

```
reader@ubuntu:~/scripts/chapter_10$ ls -l
total 8
-rw-rw-r-- 1 reader reader  29 Oct 14 10:29 character-class.txt
-rw-rw-r-- 1 reader reader 219 Oct  8 19:22 grep-file.txt
reader@ubuntu:~/scripts/chapter_10$ cp * /tmp/
reader@ubuntu:~/scripts/chapter_10$ ls -l /tmp/
total 20
-rw-rw-r-- 1 reader reader   29 Oct 14 16:35 character-class.txt
-rw-rw-r-- 1 reader reader  219 Oct 14 16:35 grep-file.txt
<SNIPPED>
```

我们使用`*`来选择它们两个。相同的 glob 模式也可以用于`rm`：

```
reader@ubuntu:/tmp$ ls -l
total 16
-rw-rw-r-- 1 reader reader   29 Oct 14 16:37 character-class.txt
-rw-rw-r-- 1 reader reader  219 Oct 14 16:37 grep-file.txt
drwx------ 3 root root 4096 Oct 14 09:22 systemd-private-c34c8acb350...
drwx------ 3 root root 4096 Oct 14 09:22 systemd-private-c34c8acb350...
reader@ubuntu:/tmp$ rm *
rm: cannot remove 'systemd-private-c34c8acb350...': Is a directory
rm: cannot remove 'systemd-private-c34c8acb350...': Is a directory
reader@ubuntu:/tmp$ ls -l
total 8
drwx------ 3 root root 4096 Oct 14 09:22 systemd-private-c34c8acb350...
drwx------ 3 root root 4096 Oct 14 09:22 systemd-private-c34c8acb350...
```

默认情况下，`rm`只会删除文件而不是目录（正如你从前面的例子中的错误中看到的）。正如第六章所述，*文件操作*，添加`-r`将递归地删除目录。

再次，请考虑这样做的破坏性：没有警告，你可能会删除当前树位置内的每个文件（当然，如果你有权限的话）。前面的例子展示了`*` glob 模式有多么强大：它会扩展到它能找到的每个文件，无论类型如何。

# 与正则表达式的相似之处

正如所述，glob 命令实现了与正则表达式类似的效果。不过也有一些区别。例如，正则表达式中的`*`字符代表*前一个字符的零次或多次出现*。对于 globbing 来说，它是一个通配符，代表任何字符，更类似于正则表达式的`.*`表示。

与正则表达式一样，glob 模式可以由普通字符和特殊字符组合而成。看一个例子，其中`ls`与不同的参数/ globbing 模式一起使用：

```
reader@ubuntu:~/scripts/chapter_09$ ls -l
total 68
-rw-rw-r-- 1 reader reader  682 Oct  2 18:31 empty-file.sh
-rw-rw-r-- 1 reader reader 1183 Oct  1 19:06 file-create.sh
-rw-rw-r-- 1 reader reader  467 Sep 29 19:43 functional-check.sh
<SNIPPED>
reader@ubuntu:~/scripts/chapter_09$ ls -l *
-rw-rw-r-- 1 reader reader  682 Oct  2 18:31 empty-file.sh
-rw-rw-r-- 1 reader reader 1183 Oct  1 19:06 file-create.sh
-rw-rw-r-- 1 reader reader  467 Sep 29 19:43 functional-check.sh
<SNIPPED>
reader@ubuntu:~/scripts/chapter_09$ ls -l if-then-exit.sh 
-rw-rw-r-- 1 reader reader 416 Sep 30 18:51 if-then-exit.sh
reader@ubuntu:~/scripts/chapter_09$ ls -l if-*.sh
-rw-rw-r-- 1 reader reader 448 Sep 30 20:10 if-then-else-proper.sh
-rw-rw-r-- 1 reader reader 422 Sep 30 19:56 if-then-else.sh
-rw-rw-r-- 1 reader reader 535 Sep 30 19:44 if-then-exit-rc-improved.sh
-rw-rw-r-- 1 reader reader 556 Sep 30 19:18 if-then-exit-rc.sh
-rw-rw-r-- 1 reader reader 416 Sep 30 18:51 if-then-exit.sh
```

在上一章的`scripts`目录中，我们首先运行了一个普通的`ls -l`。如你所知，这会打印出目录中的所有文件。现在，如果我们使用`ls -l *`，我们会得到完全相同的结果。看起来，鉴于缺少参数，`ls`会为我们注入一个通配符 glob。

接下来，我们使用`ls`的替代模式，其中我们将文件名作为参数。在这种情况下，因为每个目录的文件名是唯一的，我们只会看到返回的单行。

但是，如果我们想要所有以`if-`开头的*scripts*（以`.sh`结尾）呢？我们使用`if-*.sh`的 globbing 模式。在这个模式中，`*`通配符被扩展为匹配，正如`man glob`所说，*任何字符串，包括空字符串*。

# 更多的 globbing

在 Linux 中，globbing 非常常见。如果你正在处理一个处理文件的命令（根据*一切皆为文件*原则，大多数命令都是如此），那么你很有可能可以使用 globbing。为了让你对此有所了解，考虑以下例子：

```
reader@ubuntu:~/scripts/chapter_10$ cat *
eee
e2e
e e
aaa
a2a
a a
aabb
We can use this regular file for testing grep.
Regular expressions are pretty cool
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
Also, New Zealand is pretty far away.
```

`cat`命令与通配符 glob 模式结合使用，打印出当前工作目录中**所有文件**的内容。在这种情况下，由于所有文件都是 ASCII 文本，这并不是真正的问题。正如你所看到的，文件都是紧挨在一起打印出来的；它们之间甚至没有空行。

如果你`cat`一个二进制文件，你的屏幕会看起来像这样：

```
reader@ubuntu:~/scripts/chapter_10$ cat /bin/chvt 
@H!@8    @@@�888�� �� �  H 88 8 �TTTDDP�td\\\llQ�tdR�td�� � /lib64/ld-linux-x86-64.so.2GNUGNU��H������)�!�@`��a*�K��9���X' Q��/9'~���C J
```

最糟糕的情况是二进制文件包含某个字符序列，这会对你的 Bash shell 进行临时更改，使其无法使用（是的，这种情况我们遇到过很多次）。这里的教训应该很简单：**在使用 glob 时要小心！**

到目前为止，我们看到的其他命令可以处理 globbing 模式的命令包括`chmod`、`chown`、`mv`、`tar`、`grep`等等。现在可能最有趣的是`grep`。我们已经在单个文件上使用了正则表达式与`grep`，但我们也可以使用 glob 来选择文件。

让我们来看一个最荒谬的`grep`与 globbing 的例子：在*everything*中找到*anything*。

```
reader@ubuntu:~/scripts/chapter_10$ grep .* *
grep: ..: Is a directory
character-class.txt:eee
character-class.txt:e2e
character-class.txt:e e
character-class.txt:aaa
character-class.txt:a2a
character-class.txt:a a
character-class.txt:aabb
grep-file.txt:We can use this regular file for testing grep.
grep-file.txt:Regular expressions are pretty cool
grep-file.txt:Did you ever realise that in the UK they say colour,
grep-file.txt:but in the USA they use color (and realize)!
grep-file.txt:Also, New Zealand is pretty far away.
```

在这里，我们使用了正则表达式`.*`的搜索模式（任何东西，零次或多次）与`*`的 glob 模式（任何文件）。正如你所期望的那样，这应该匹配每个文件的每一行。

当我们以这种方式使用`grep`时，它的功能基本上与之前的`cat *`相同。但是，当`grep`用于多个文件时，输出会包括文件名（这样您就知道找到该行的位置）。

请注意：globbing 模式总是与文件相关，而正则表达式是用于*文件内部*，用于实际内容。由于语法相似，您可能不会对此感到太困惑，但如果您曾经遇到过模式不按您的预期工作的情况，那么花点时间考虑一下您是在进行 globbing 还是正则表达式会很有帮助！

# 高级 globbing

基本的 globbing 主要是使用通配符，有时与部分文件名结合使用。然而，正如正则表达式允许我们替换单个字符一样，glob 也可以。

正则表达式通过点来实现这一点；在 globbing 模式中，问号被使用：

```
reader@ubuntu:~/scripts/chapter_09$ ls -l if-then-*
-rw-rw-r-- 1 reader reader 448 Sep 30 20:10 if-then-else-proper.sh
-rw-rw-r-- 1 reader reader 422 Sep 30 19:56 if-then-else.sh
-rw-rw-r-- 1 reader reader 535 Sep 30 19:44 if-then-exit-rc-improved.sh
-rw-rw-r-- 1 reader reader 556 Sep 30 19:18 if-then-exit-rc.sh
-rw-rw-r-- 1 reader reader 416 Sep 30 18:51 if-then-exit.sh
reader@ubuntu:~/scripts/chapter_09$ ls -l if-then-e???.sh
-rw-rw-r-- 1 reader reader 422 Sep 30 19:56 if-then-else.sh
-rw-rw-r-- 1 reader reader 416 Sep 30 18:51 if-then-exit.sh
```

现在，globbing 模式`if-then-e???.sh`应该不言自明了。在`?`出现的地方，任何字符（字母、数字、特殊字符）都是有效的替代。

在前面的例子中，所有三个问号都被字母替换。正如您可能已经推断出的那样，正则表达式`.`字符与 globbing 模式`?`字符具有相同的功能：它有效地代表一个字符。

最后，我们用于正则表达式的单括号表示法也可以用于 globbing。一个快速的例子展示了我们如何在`cat`中使用它：

```
reader@ubuntu:/tmp$ echo ping > ping # Write the word ping to the file ping.
reader@ubuntu:/tmp$ echo pong > pong # Write the word pong to the file pong.
reader@ubuntu:/tmp$ ls -l
total 16
-rw-rw-r-- 1 reader reader    5 Oct 14 17:17 ping
-rw-rw-r-- 1 reader reader    5 Oct 14 17:17 pong
reader@ubuntu:/tmp$ cat p[io]ng
ping
pong
reader@ubuntu:/tmp$ cat p[a-z]ng
ping
pong
```

# 禁用 globbing 和其他选项

尽管 globbing 功能强大，但这也是它危险的原因。因此，您可能希望采取激烈措施并关闭 globbing。虽然这是可能的，但我们并没有在实践中看到过。但是，对于一些工作或脚本，关闭 globbing 可能是一个很好的保障。

使用`set`命令，我们可以像 man 页面所述那样*更改 shell 选项的值*。在这种情况下，使用`-f`将关闭 globbing，正如我们在尝试重复之前的例子时所看到的：

```
reader@ubuntu:/tmp$ cat p?ng
ping
pong
reader@ubuntu:/tmp$ set -f
reader@ubuntu:/tmp$ cat p?ng
cat: 'p?ng': No such file or directory
reader@ubuntu:/tmp$ set +f
reader@ubuntu:/tmp$ cat p?ng
ping
pong
```

通过在前缀加上减号（`-`）来关闭选项，通过在前缀加上加号（`+`）来打开选项。您可能还记得，这不是您第一次使用这个功能。当我们调试 Bash 脚本时，我们开始的不是`bash`，而是`bash -x`。

在这种情况下，Bash 子 shell 在调用脚本之前执行了`set -x`命令。如果您在当前终端中使用`set -x`，您的命令将开始看起来像这样：

```
reader@ubuntu:/tmp$ cat p?ng
ping
pong
reader@ubuntu:/tmp$ set -x
reader@ubuntu:/tmp$ cat p?ng
+ cat ping pong
ping
pong
reader@ubuntu:/tmp$ set +x
+ set +x
reader@ubuntu:/tmp$ cat p?ng
ping
pong
```

请注意，我们现在可以看到 globbing 模式是如何解析的：从`cat p?ng`到`cat ping pong`。尽量记住这个功能；如果您曾经因为不知道脚本为什么不按照您的意愿执行而抓狂，一个简单的`set -x`可能会产生很大的不同！如果不行，您总是可以通过`set +x`恢复正常行为，就像例子中所示的那样。

`set`有许多有趣的标志，可以让您的生活更轻松。要查看您的 Bash 版本中`set`的功能概述，请使用`help set`命令。因为`set`是一个 shell 内置命令（您可以用`type set`来验证），所以不幸的是，查找`man set`的 man 页面是行不通的。

# 使用 egrep 和 sed 的正则表达式

我们现在已经讨论了正则表达式和 globbing。正如我们所看到的，它们非常相似，但仍然有一些需要注意的区别。在我们的正则表达式示例中，以及一些 globbing 示例中，我们已经看到了`grep`的用法。

在这部分中，我们将介绍另一个命令，它与正则表达式结合使用时非常方便：`sed`（不要与`set`混淆）。我们将从一些用于`grep`的高级用法开始。

# 高级 grep

我们已经讨论了一些用于更改`grep`默认行为的流行选项：`--ignore-case`（`-i`）、`--invert-match`（`-v`）和`--word-regexp`（`-w`）。作为提醒，这是它们的作用：

+   `-i`允许我们进行不区分大小写的搜索

+   `-v`只打印*不*匹配的行，而不是匹配的行

+   `-w`只匹配由空格和/或行锚和/或标点符号包围的完整单词

还有三个其他选项我们想和你分享。第一个新选项，`--only-matching`（`-o`）只打印匹配的单词。如果你的搜索模式不包含任何正则表达式，这可能是一个相当无聊的选项，就像在这个例子中所看到的：

```
reader@ubuntu:~/scripts/chapter_10$ grep -o 'cool' grep-file.txt 
cool
```

它确实如你所期望的那样：它打印了你要找的单词。然而，除非你只是想确认这一点，否则可能并不那么有趣。

现在，如果我们在使用一个更有趣的搜索模式（包含正则表达式）时做同样的事情，这个选项就更有意义了：

```
reader@ubuntu:~/scripts/chapter_10$ grep -o 'f.r' grep-file.txt 
for
far
```

在这个（简化的！）例子中，你实际上得到了新的信息：你搜索模式中的任何单词都会被打印出来。虽然对于这样一个短的单词在这样一个小的文件中来说可能并不那么令人印象深刻，但想象一下在一个更大的文件中使用一个更复杂的搜索模式！

这带来了另一个问题：`grep`非常*快*。由于 Boyer-Moore 算法，`grep`可以在非常大的文件（100 MB+）中进行非常快速的搜索。

第二个额外选项，`--count`（`-c`），不返回任何行。但是，它会返回一个数字：搜索模式匹配的行数。一个众所周知的例子是查看包安装的日志文件时：

```
reader@ubuntu:/var/log$ grep 'status installed' dpkg.log
2018-04-26 19:07:29 status installed base-passwd:amd64 3.5.44
2018-04-26 19:07:29 status installed base-files:amd64 10.1ubuntu2
2018-04-26 19:07:30 status installed dpkg:amd64 1.19.0.5ubuntu2
<SNIPPED>
2018-06-30 17:59:37 status installed linux-headers-4.15.0-23:all 4.15.0-23.25
2018-06-30 17:59:37 status installed iucode-tool:amd64 2.3.1-1
2018-06-30 17:59:37 status installed man-db:amd64 2.8.3-2
<SNIPPED>
2018-07-01 09:31:15 status installed distro-info-data:all 0.37ubuntu0.1
2018-07-01 09:31:17 status installed libcurl3-gnutls:amd64 7.58.0-2ubuntu3.1
2018-07-01 09:31:17 status installed libc-bin:amd64 2.27-3ubuntu1
```

在这个常规的`grep`中，我们看到显示了哪个包在哪个日期安装的日志行。但是，如果我们只想知道*某个日期安装了多少个包*呢？`--count`来帮忙！

```
reader@ubuntu:/var/log$ grep 'status installed' dpkg.log | grep '2018-08-26'
2018-08-26 11:16:16 status installed base-files:amd64 10.1ubuntu2.2
2018-08-26 11:16:16 status installed install-info:amd64 6.5.0.dfsg.1-2
2018-08-26 11:16:16 status installed plymouth-theme-ubuntu-text:amd64 0.9.3-1ubuntu7
<SNIPPED>
reader@ubuntu:/var/log$ grep 'status installed' dpkg.log | grep -c '2018-08-26'
40
```

我们将这个`grep`操作分为两个阶段。第一个`grep 'status installed'`过滤掉所有与成功安装相关的行，跳过中间步骤，比如*unpacked*和*half-configured*。

我们在管道后面使用`grep`的替代形式（我们将在第十二章中进一步讨论，*在脚本中使用管道和重定向*）来匹配另一个搜索模式到已经过滤的数据。第二个`grep '2018-08-26'`用于按日期过滤。

现在，如果没有`-c`选项，我们会看到 40 行。如果我们对包感兴趣，这可能是一个不错的选择，但否则，只打印数字比手动计算行数要好。

或者，我们可以将其写成一个单独的 grep 搜索模式，使用正则表达式。自己试一试：`grep '2018-08-26 .* status installed' dpkg.log`（确保用你运行更新/安装的某一天替换日期）。

最后一个选项非常有趣，特别是对于脚本编写，就是`--quiet`（`-q`）选项。想象一种情况，你想知道文件中是否存在某个搜索模式。如果找到了搜索模式，就删除文件。如果没有找到搜索模式，就将其添加到文件中。

你知道，你可以使用一个很好的`if-then-else`结构来完成这个任务。但是，如果你使用普通的`grep`，当你运行脚本时，你会在终端上看到文本被打印出来。

这并不是一个很大的问题，但是一旦你的脚本变得足够大和复杂，大量的输出到屏幕会使脚本难以使用。为此，我们有`--quiet`选项。看看这个示例脚本，看看你会如何做到这一点：

```
reader@ubuntu:~/scripts/chapter_10$ vim grep-then-else.sh 
reader@ubuntu:~/scripts/chapter_10$ cat grep-then-else.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-16
# Description: Use grep exit status to make decisions about file manipulation.
# Usage: ./grep-then-else.sh
#####################################

FILE_NAME=/tmp/grep-then-else.txt

# Touch the file; creates it if it does not exist.
touch ${FILE_NAME}

# Check the file for the keyword.
grep -q 'keyword' ${FILE_NAME}
grep_rc=$?

# If the file contains the keyword, remove the file. Otherwise, write 
# the keyword to the file.
if [[ ${grep_rc} -eq 0 ]]; then
  rm ${FILE_NAME}  
else
  echo 'keyword' >> ${FILE_NAME}
fi

reader@ubuntu:~/scripts/chapter_10$ bash -x grep-then-else.sh 
+ FILE_NAME=/tmp/grep-then-else.txt
+ touch /tmp/grep-then-else.txt
+ grep --quiet keyword /tmp/grep-then-else.txt
+ grep_rc='1'
+ [[ '1' -eq 0 ]]
+ echo keyword
reader@ubuntu:~/scripts/chapter_10$ bash -x grep-then-else.sh 
+ FILE_NAME=/tmp/grep-then-else.txt
+ touch /tmp/grep-then-else.txt
+ grep -q keyword /tmp/grep-then-else.txt
+ grep_rc=0
+ [[ 0 -eq 0 ]]
+ rm /tmp/grep-then-else.txt
```

正如你所看到的，关键在于退出状态。如果`grep`找到一个或多个搜索模式的匹配，就会返回退出代码 0。如果`grep`没有找到任何内容，返回代码将是 1。

你可以在命令行上自己看到这一点：

```
reader@ubuntu:/var/log$ grep -q 'glgjegeg' dpkg.log
reader@ubuntu:/var/log$ echo $?
1
reader@ubuntu:/var/log$ grep -q 'installed' dpkg.log 
reader@ubuntu:/var/log$ echo $?
0
```

在`grep-then-else.sh`中，我们抑制了`grep`的所有输出。但是，我们仍然可以实现我们想要的效果：脚本的每次运行在*then*和*else*条件之间变化，正如我们的`bash -x`调试输出清楚地显示的那样。

没有`--quiet`，脚本的非调试输出将如下所示：

```
reader@ubuntu:/tmp$ bash grep-then-else.sh 
reader@ubuntu:/tmp$ bash grep-then-else.sh 
keyword
reader@ubuntu:/tmp$ bash grep-then-else.sh 
reader@ubuntu:/tmp$ bash grep-then-else.sh 
keyword
```

它实际上并没有为脚本添加任何东西，是吗？更好的是，很多命令都有`--quiet`，`-q`或等效选项。

在编写脚本时，始终考虑命令的输出是否相关。如果不相关，并且可以使用退出状态，这几乎总是会使输出体验更清晰。

# 介绍`egrep`

到目前为止，我们已经看到`grep`与各种选项一起使用，这些选项改变了它的行为。有一个最后重要的选项我们想要和你分享：`--extended-regexp` (`-E`)。正如`man grep`页面所述，这意味着*将 PATTERN 解释为扩展正则表达式*。

与 Linux 中找到的默认正则表达式相比，扩展正则表达式具有更接近其他脚本/编程语言中的正则表达式的搜索模式（如果你已经有这方面的经验）。

具体来说，在使用扩展正则表达式而不是默认正则表达式时，以下构造是可用的：

| ? | 匹配前一个字符的重复*零次或多次* |
| --- | --- |
| + | 匹配前一个字符的重复*一次或多次* |
| {n} | 匹配前一个字符的重复*恰好 n 次* |
| {n,m} | 匹配前一个字符的重复*介于 n 和 m 次之间* |
| {,n} | 匹配前一个字符的重复*n 次或更少次* |
| {n,} | 匹配前一个字符的重复*n 次或更多次* |
| (xx&#124;yy) | 交替字符，允许我们在搜索模式中找到 xx *或* yy（对于具有多个字符的模式非常有用，否则，`[xy]`表示法就足够了） |

正如你可能已经看到的，`grep`的 man 页面包含了一个关于正则表达式和搜索模式的专门部分，你可能会发现它作为一个快速参考非常方便。

现在，在我们开始使用新的 ERE 搜索模式之前，我们将介绍一个*新*命令：`egrep`。如果你试图找出它的作用，你可能会从`which egrep`开始，结果是`/bin/egrep`。这可能会让你认为它是一个独立的二进制文件，而不是你现在已经使用了很多的`grep`。

然而，最终，`egrep`只不过是一个小小的包装脚本：

```
reader@ubuntu:~/scripts/chapter_10$ cat /bin/egrep
#!/bin/sh
exec grep -E "$@"
```

你可以看到，这只是一个 shell 脚本，但没有通常的`.sh`扩展名。它使用`exec`命令来*用新的进程映像替换当前进程映像*。

你可能还记得，通常情况下，命令是在当前环境的一个分支中执行的。在这种情况下，因为我们使用这个脚本来*包装*（这就是为什么它被称为包装脚本）`grep -E`作为`egrep`，所以替换它而不是再次分支是有意义的。

`"$@"`构造也是新的：它是一个*数组*（如果你对这个术语不熟悉，可以想象为一个有序列表）的参数。在这种情况下，它基本上将`egrep`接收到的所有参数传递到`grep -E`中。

因此，如果完整的命令是`egrep -w [[:digit:]] grep-file.txt`，它将被包装并最终作为`grep -E -w [[:digit:]] grep-file.txt`执行。

实际上，使用`egrep`或`grep -E`并不重要。我们更喜欢使用`egrep`，这样我们就可以确定我们正在处理扩展的正则表达式（因为在我们的经验中，扩展功能经常被使用）。但是，对于简单的搜索模式，不需要使用 ERE。

我们建议你找到自己的系统，决定何时使用每个命令。

现在我们来看一些扩展正则表达式搜索模式的例子：

```
reader@ubuntu:~/scripts/chapter_10$ egrep -w '[[:lower:]]{5}' grep-file.txt 
but in the USA they use color (and realize)!
reader@ubuntu:~/scripts/chapter_10$ egrep -w '[[:lower:]]{7}' grep-file.txt 
We can use this regular file for testing grep.
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
reader@ubuntu:~/scripts/chapter_10$ egrep -w '[[:alpha:]]{7}' grep-file.txt 
We can use this regular file for testing grep.
Regular expressions are pretty cool
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
Also, New Zealand is pretty far away.
```

第一个命令`egrep -w [[:lower:]]{5} grep-file.txt`，显示了所有恰好五个字符长的单词，使用小写字母。不要忘记这里需要`-w`选项，因为否则，任何五个字母连续在一起也会匹配，忽略单词边界（在这种情况下，**prett**y 中的**prett**也会匹配）。结果只有一个五个字母的单词：color。

接下来，我们对七个字母的单词做同样的操作。我们现在得到了更多的结果。然而，因为我们只使用小写字母，我们错过了两个也是七个字母长的单词：Regular 和 Zealand。我们通过使用`[[:alpha:]]`而不是`[[:lower:]]`来修复这个问题。（我们也可以使用`-i`选项使所有内容不区分大小写—`egrep -iw [[:lower:]]{7} grep-file.txt`。

虽然这在功能上是可以接受的，但请再考虑一下。在这种情况下，你将搜索由七个*小写*字母组成的*不区分大小写*单词。这实际上没有任何意义。在这种情况下，我们总是选择逻辑而不是功能，这意味着将`[[:lower:]]`改为`[[:alpha:]]`，而不是使用`-i`选项。

所以我们知道了如何搜索特定长度的单词（或行，如果省略了`-w`选项）。现在我们来搜索比最小长度或最大长度更长或更短的单词。

这里有一个例子：

```
reader@ubuntu:~/scripts/chapter_10$ egrep -w '[[:lower:]]{5,}' grep-file.txt
We can use this regular file for testing grep.
Regular expressions are pretty cool
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ egrep -w '[[:alpha:]]{,3}' grep-file.txt
We can use this regular file for testing grep.
Regular expressions are pretty cool
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ egrep '.{40,}' grep-file.txt
We can use this regular file for testing grep.
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
```

这个例子演示了边界语法。第一个命令，`egrep -w '[[:lower:]]{5,}' grep-file.txt`，寻找了至少五个字母的小写单词。如果你将这些结果与之前寻找确切五个字母长的单词的例子进行比较，你现在会发现更长的单词也被匹配到了。

接下来，我们反转边界条件：我们只想匹配三个字母或更少的单词。我们看到所有两个和三个字母的单词都被匹配到了（因为我们从`[[:lower:]]`切换到了`[[:alpha:]]`，UK 和行首大写字母也被匹配到了）。

在最后一个例子中，`egrep '.{40,}' grep-file.txt`，我们去掉了`-w`，所以我们匹配整行。我们匹配任何字符（由点表示），并且我们希望一行至少有 40 个字符（由`{40,}`表示）。在这种情况下，只有五行中的三行被匹配到了（因为其他两行较短）。

引用对于搜索模式非常重要。如果你在模式中不使用引号，特别是在使用{和}等特殊字符时，你将需要用反斜杠对它们进行转义。这可能会导致令人困惑的情况，你会盯着屏幕想知道为什么你的搜索模式不起作用，甚至会报错。只要记住：如果你始终对搜索模式使用单引号，你就会更有可能避免这些令人沮丧的情况。

我们想要展示的扩展正则表达式的最后一个概念是*alternation*。这使用了管道语法（不要与用于重定向的管道混淆，这将在第十二章中进一步讨论，*在脚本中使用管道和重定向*）来传达*匹配 xxx 或 yyy*的含义。

一个例子应该能说明问题：

```
reader@ubuntu:~/scripts/chapter_10$ egrep 'f(a|o)r' grep-file.txt 
We can use this regular file for testing grep.
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ egrep 'f[ao]r' grep-file.txt
We can use this regular file for testing grep.
Also, New Zealand is pretty far away.
reader@ubuntu:~/scripts/chapter_10$ egrep '(USA|UK)' grep-file.txt 
Did you ever realise that in the UK they say colour,
but in the USA they use color (and realize)!
```

在只有一个字母差异的情况下，我们可以选择使用扩展的 alternation 语法，或者之前讨论过的括号语法。我们建议使用最简单的语法来实现目标，这种情况下就是括号语法。

然而，一旦我们要寻找超过一个字符差异的模式，使用括号语法就变得非常复杂。在这种情况下，扩展的 alternation 语法是清晰而简洁的，特别是因为`|`或`||`在大多数脚本/编程逻辑中代表`OR`构造。对于这个例子，这就像是说：我想要找到包含单词 USA 或单词 UK 的行。

因为这种语法与语义视图相对应得很好，它感觉直观且易懂，这是我们在脚本中应该始终努力的事情！

# 流编辑器 sed

由于我们现在对正则表达式、搜索模式和（扩展）`grep`非常熟悉，是时候转向 GNU/Linux 领域中最强大的工具之一了：`sed`。这个术语是**s**tream **ed**itor 的缩写，它确实做到了它所暗示的：编辑流。

在这种情况下，流可以是很多东西，但通常是文本。这个文本可以在文件中找到，但也可以从另一个进程中*流式传输*，比如`cat grep-file.txt | sed ...`。在这个例子中，`cat`命令的输出（等同于`grep-file.txt`的内容）作为`sed`命令的输入。

我们将在我们的示例中查看就地文件编辑和流编辑。

# 流编辑

首先，我们将看一下使用`sed`进行实际流编辑。流编辑允许我们做一些很酷的事情：例如，我们可以更改文本中的一些单词。我们还可以删除我们不关心的某些行（例如，不包含单词 ERROR 的所有内容）。

我们将从一个简单的例子开始，搜索并替换一行中的一个单词：

```
reader@ubuntu:~/scripts/chapter_10$ echo "What a wicked sentence"
What a wicked sentence
reader@ubuntu:~/scripts/chapter_10$ echo "What a wicked sentence" | sed 's/wicked/stupid/'
What a stupid sentence
```

就像这样，`sed`将我的积极句子转变成了不太积极的东西。`sed`使用的模式（在`sed`术语中，这只是称为*script*）是`s/wicked/stupid/`。`s`代表搜索替换，*script*的第一个单词被第二个单词替换。

观察一下对于具有多个匹配项的多行会发生什么：

```
reader@ubuntu:~/scripts/chapter_10$ vim search.txt
reader@ubuntu:~/scripts/chapter_10$ cat search.txt 
How much wood would a woodchuck chuck
if a woodchuck could chuck wood?
reader@ubuntu:~/scripts/chapter_10$ cat search.txt | sed 's/wood/stone/'
How much stone would a woodchuck chuck
if a stonechuck could chuck wood?
```

从这个例子中，我们可以学到两件事：

+   默认情况下，`sed`只会替换每行中每个单词的第一个实例。

+   `sed`不仅匹配整个单词，还匹配部分单词。

如果我们想要替换每行中的所有实例怎么办？这称为*全局*搜索替换，语法只有非常轻微的不同：

```
reader@ubuntu:~/scripts/chapter_10$ cat search.txt | sed 's/wood/stone/g'
How much stone would a stonechuck chuck
if a stonechuck could chuck stone?
```

通过在`sed` *script*的末尾添加`g`，我们现在全局替换所有实例，而不仅仅是每行的第一个实例。

另一种可能性是，您可能只想在第一行上进行搜索替换。您可以使用`head -1`仅选择该行，然后将其发送到`sed`，但这意味着您需要在后面添加其他行。

我们可以通过在`sed`脚本前面放置行号来选择要编辑的行，如下所示：

```
reader@ubuntu:~/scripts/chapter_10$ cat search.txt | sed '1s/wood/stone/'
How much stone would a woodchuck chuck
if a woodchuck could chuck wood?
reader@ubuntu:~/scripts/chapter_10$ cat search.txt | sed '1s/wood/stone/g'
How much stone would a stonechuck chuck
if a woodchuck could chuck wood?
reader@ubuntu:~/scripts/chapter_10$ cat search.txt | sed '1,2s/wood/stone/g'
How much stone would a stonechuck chuck
if a stonechuck could chuck stone?
```

第一个脚本，`'1s/wood/stone/'`，指示`sed`将第一行中的第一个*wood*实例替换为*stone*。下一个脚本，`'1s/wood/stone/g'`，告诉`sed`将*wood*的所有实例替换为*stone*，但只在第一行上。最后一个脚本，`'1,2s/wood/stone/g'`，使`sed`替换所有行（包括！）中（和包括！）`1`和`2`之间的所有*wood*实例。

# 就地编辑

虽然在将文件发送到`sed`之前`cat`文件并不是*那么*大的问题，幸运的是，我们实际上不需要这样做。`sed`的用法如下：`sed [OPTION] {script-only-if-no-other-script} [input-file]`。正如您在最后看到的那样，还有一个选项`[input-file]`。

让我们拿之前的一个例子，然后去掉`cat`：

```
reader@ubuntu:~/scripts/chapter_10$ sed 's/wood/stone/g' search.txt 
How much stone would a stonechuck chuck
if a stonechuck could chuck stone?
reader@ubuntu:~/scripts/chapter_10$ cat search.txt 
How much wood would a woodchuck chuck
if a woodchuck could chuck wood?
```

如您所见，通过使用可选的`[input-file]`参数，`sed`根据脚本处理文件中的所有行。默认情况下，`sed`会打印它处理的所有内容。在某些情况下，这会导致行被打印两次，即当使用`sed`的`print`函数时（我们稍后会看到）。

这个例子展示的另一个非常重要的事情是：这种语法不会编辑原始文件；只有打印到`STDOUT`的内容会发生变化。有时，您可能希望编辑文件本身——对于这些情况，`sed`有`--in-place`（`-i`）选项。

确保您理解这**会对磁盘上的文件进行不可逆转的更改**。而且，就像 Linux 中的大多数事情一样，没有撤销按钮或回收站！

让我们看看如何使用`sed -i`来持久更改文件（当然，在我们备份之后）：

```
reader@ubuntu:~/scripts/chapter_10$ cat search.txt 
How much wood would a woodchuck chuck
if a woodchuck could chuck wood?
reader@ubuntu:~/scripts/chapter_10$ cp search.txt search.txt.bak
reader@ubuntu:~/scripts/chapter_10$ sed -i 's/wood/stone/g' search.txt
reader@ubuntu:~/scripts/chapter_10$ cat search.txt
How much stone would a stonechuck chuck
if a stonechuck could chuck stone?
```

这一次，不是将处理后的文本打印到屏幕上，而是`sed`悄悄地更改了磁盘上的文件。由于这种破坏性的本质，我们事先创建了一个备份。但是，`sed`的`--in-place`选项也可以提供这种功能，方法是添加文件后缀：

```
reader@ubuntu:~/scripts/chapter_10$ ls
character-class.txt  error.txt  grep-file.txt  grep-then-else.sh  search.txt  search.txt.bak
reader@ubuntu:~/scripts/chapter_10$ mv search.txt.bak search.txt
reader@ubuntu:~/scripts/chapter_10$ cat search.txt 
How much wood would a woodchuck chuck
if a woodchuck could chuck wood?
reader@ubuntu:~/scripts/chapter_10$ sed -i'.bak' 's/wood/stone/g' search.txt
reader@ubuntu:~/scripts/chapter_10$ cat search.txt
How much stone would a stonechuck chuck
if a stonechuck could chuck stone?
reader@ubuntu:~/scripts/chapter_10$ cat search.txt.bak 
How much wood would a woodchuck chuck
if a woodchuck could chuck wood?
```

`sed`的语法有点吝啬。如果在`-i`和`'.bak'`之间加上一个空格，您将会得到奇怪的错误（这通常对于选项带有参数的命令来说是正常的）。在这种情况下，因为脚本定义紧随其后，`sed`很难区分文件后缀和脚本字符串。

只要记住，如果您想使用这个，您需要小心这个语法！

# 行操作

虽然`sed`的单词操作功能很棒，但它也允许我们操作整行。例如，我们可以按行号删除某些行：

```
reader@ubuntu:~/scripts/chapter_10$ echo -e "Hi,\nthis is \nPatrick"
Hi,
this is 
Patrick
reader@ubuntu:~/scripts/chapter_10$ echo -e "Hi,\nthis is \nPatrick" | sed 'd'
reader@ubuntu:~/scripts/chapter_10$ echo -e "Hi,\nthis is \nPatrick" | sed '1d'
this is 
Patrick
```

通过使用`echo -e`结合换行符（`\n`），我们可以创建多行语句。`-e`在`man echo`页面上解释为*启用反斜杠转义的解释*。通过将这个多行输出传递给`sed`，我们可以使用删除功能，这是一个简单地使用字符`d`的脚本。

如果我们在行号前加上一个前缀，例如`1d`，则删除第一行。如果不这样做，所有行都将被删除，这对我们来说没有输出。

另一个，通常更有趣的可能性是删除包含某个单词的行：

```
reader@ubuntu:~/scripts/chapter_10$ echo -e "Hi,\nthis is \nPatrick" | sed '/Patrick/d'
Hi,
this is 
reader@ubuntu:~/scripts/chapter_10$ echo -e "Hi,\nthis is \nPatrick" | sed '/patrick/d'
Hi,
this is 
Patrick
```

与我们使用脚本进行单词匹配的`sed`搜索替换功能一样，如果存在某个单词，我们也可以删除整行。从前面的例子中可以看到，这是区分大小写的。幸运的是，如果我们想以不区分大小写的方式进行操作，总是有解决办法。在`grep`中，这将是`-i`标志，但对于`sed`，`-i`已经保留给了`--in-place`功能。

那我们该怎么做呢？当然是使用我们的老朋友正则表达式！请参阅以下示例：

```
reader@ubuntu:~/scripts/chapter_10$ echo -e "Hi,\nthis is \nPatrick" | sed '/[Pp]atrick/d'
Hi,
this is
reader@ubuntu:~/scripts/chapter_10$ echo -e "Hi,\nthis is \nPatrick" | sed '/.atrick/d'
Hi,
this is
```

虽然它不像`grep`提供的功能那样优雅，但在大多数情况下它确实完成了工作。它至少应该让您意识到，使用正则表达式与`sed`使整个过程更加灵活和更加强大。

与大多数事物一样，增加了灵活性和功能，也增加了复杂性。但是，我们希望通过这对正则表达式和`sed`的简要介绍，两者的组合不会感到难以管理的复杂。

与从文件或流中删除行不同，您可能更适合只显示一些文件。但是，这里有一个小问题：默认情况下，`sed`会打印它处理的所有行。如果您给`sed`指令打印一行（使用`p`脚本*），它将打印该行两次——一次是匹配脚本，另一次是默认打印。

这看起来有点像这样：

```
reader@ubuntu:~/scripts/chapter_10$ cat error.txt 
Process started.
Running normally.
ERROR: TCP socket broken.
ERROR: Cannot connect to database.
Exiting process.
reader@ubuntu:~/scripts/chapter_10$ sed '/ERROR/p' error.txt 
Process started.
Running normally.
ERROR: TCP socket broken.
ERROR: TCP socket broken.
ERROR: Cannot connect to database.
ERROR: Cannot connect to database.
Exiting process.
```

打印和删除脚本的语法类似：`'/word/d'`和`'/word/p'`。要抑制`sed`的默认行为，即打印所有行，添加`-n`（也称为`--quiet`或`--silent`）：

```
reader@ubuntu:~/scripts/chapter_10$ sed -n '/ERROR/p' error.txt 
ERROR: TCP socket broken.
ERROR: Cannot connect to database.
```

您可能已经发现，使用`sed`脚本打印和删除行与`grep`和`grep -v`具有相同的功能。在大多数情况下，您可以选择使用哪种。但是，一些高级功能，例如删除匹配的行，但仅从文件的前 10 行中删除，只能使用`sed`完成。作为一个经验法则，任何可以使用单个语句使用`grep`实现的功能都应该使用`grep`来处理；否则，转而使用`sed`。

有一个`sed`的最后一个用例我们想要强调：您有一个文件或流，您需要删除的不是整行，而只是这些行中的一些单词。使用`grep`，这是（很容易地）无法实现的。然而，`sed`有一种非常简单的方法来做到这一点。

搜索和替换与仅仅删除一个单词有什么不同？只是替换模式！

请参阅以下示例：

```
reader@ubuntu:~/scripts/chapter_10$ cat search.txt
How much stone would a stonechuck chuck
if a stonechuck could chuck stone?
reader@ubuntu:~/scripts/chapter_10$ sed 's/stone//g' search.txt
How much  would a chuck chuck
if a chuck could chuck ?
```

通过将单词 stone 替换为*nothing*（因为这正是在`sed`脚本中第二个和第三个反斜杠之间存在的内容），我们完全删除了单词 stone。然而，在这个例子中，你可以看到一个常见的问题，你肯定会遇到：删除单词后会有额外的空格。

这带我们来到了`sed`的另一个技巧，可以帮助你解决这个问题：

```
reader@ubuntu:~/scripts/chapter_10$ sed -e 's/stone //g' -e 's/stone//g' search.txt
How much would a chuck chuck
if a chuck could chuck ?
```

通过提供`-e`，后跟一个`sed`脚本，你可以让`sed`在你的流上运行多个脚本（按顺序！）。默认情况下，`sed`期望至少有一个脚本，这就是为什么如果你只处理一个脚本，你不需要提供`-e`。对于比这更多的脚本，你需要在每个脚本之前添加一个`-e`。

# 最后的话

正则表达式很**难**。在 Linux 上更难的是，正则表达式已经由不同的程序（具有不同的维护者和不同的观点）略有不同地实现。

更糟糕的是，一些正则表达式的特性被一些程序隐藏为扩展的正则表达式，而在其他程序中被认为是默认的。在过去的几年里，这些程序的维护者似乎已经朝着更全局的 POSIX 标准迈进，用于*正则*正则表达式和*扩展*正则表达式，但直到今天，仍然存在一些差异。

我们对处理这个问题有一些建议：**试一试**。也许你不记得星号在 globbing 中代表什么，与正则表达式不同，或者问号为什么会有不同的作用。也许你会忘记用`-E`来“激活”扩展语法，你的扩展搜索模式会返回奇怪的错误。

你肯定会忘记引用搜索模式一次，如果它包含像点或$这样的字符（由 Bash 解释），你的命令会崩溃，通常会有一个不太清晰的错误消息。

只要知道我们都犯过这些错误，只有经验才能让这变得更容易。事实上，在写这一章时，几乎没有一个命令像我们在脑海中想象的那样立即起作用！你并不孤单，你不应该因此感到难过。*继续努力，直到成功，并且直到你明白为什么第一次没有成功。*

# 总结

本章解释了正则表达式，以及在 Linux 下使用它们的两个常见工具：`grep`和`sed`。

我们首先解释了正则表达式是与文本结合使用的*搜索模式*，用于查找匹配项。这些搜索模式允许我们在文本中进行非常灵活的搜索，其中文本的内容在运行时不一定已知。

搜索模式允许我们，例如，仅查找单词而不是数字，查找行首或行尾的单词，或查找空行。搜索模式包括通配符，可以表示某个字符或字符类的一个或多个。

我们介绍了`grep`命令，以展示我们如何在 Bash 中使用正则表达式的基本功能。

本章的第二部分涉及 globbing。Globbing 用作文件名和路径的通配符机制。它与正则表达式有相似之处，但也有一些关键的区别。Globbing 可以与大多数处理文件的命令一起使用（而且，由于 Linux 下的大多数*东西*都可以被视为文件，这意味着几乎所有命令都支持某种形式的 globbing）。

本章的后半部分描述了如何使用`egrep`和`sed`的正则表达式。`egrep`是`grep -E`的简单包装器，允许我们使用扩展语法进行正则表达式，我们讨论了一些常用的高级`grep`功能。

与默认的正则表达式相比，扩展的正则表达式允许我们指定某些模式的长度以及它们重复的次数，同时还允许我们使用交替。

本章的最后部分描述了`sed`，流编辑器。`sed`是一个复杂但非常强大的命令，可以让我们做比`grep`更令人兴奋的事情。

本章介绍了以下命令：`grep`、`set`、`egrep`和`sed`。

# 问题

1.  什么是搜索模式？

1.  为什么正则表达式被认为是贪婪的？

1.  在搜索模式中，哪个字符被认为是除换行符外的任意一个字符的通配符？

1.  在 Linux 正则表达式搜索模式中，星号如何使用？

1.  什么是行锚点？

1.  列举三种字符类型。

1.  什么是 globbing？

1.  在 Bash 下，扩展正则表达式语法可以实现哪些普通正则表达式无法实现的功能？

1.  在决定使用`grep`还是`sed`时，有什么好的经验法则？

1.  为什么 Linux/Bash 上的正则表达式如此困难？

# 进一步阅读

如果您想更深入地了解本章主题，以下资源可能会很有趣：

+   Linux 文档项目关于正则表达式：[`www.tldp.org/LDP/abs/html/x17129.html`](http://www.tldp.org/LDP/abs/html/x17129.html)

+   Linux 文档项目关于 Globbing：[`www.tldp.org/LDP/abs/html/globbingref.html`](http://www.tldp.org/LDP/abs/html/globbingref.html)

+   Linux 文档项目关于 Sed：[`tldp.org/LDP/abs/html/x23170.html`](http://tldp.org/LDP/abs/html/x23170.html)
