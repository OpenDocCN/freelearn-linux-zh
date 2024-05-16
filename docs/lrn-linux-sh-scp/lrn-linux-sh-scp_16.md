# 第十六章：Bash 参数替换和扩展

本章专门介绍了 Bash 的一个特殊功能：参数扩展。参数扩展允许我们对变量进行许多有趣的操作，我们将进行广泛的介绍。

我们将首先讨论变量的默认值、输入检查和变量长度。在本章的第二部分，我们将更仔细地看一下我们如何操作变量。这包括替换和删除文本中的模式，修改变量的大小写，并使用子字符串。

本章将介绍以下命令：`export`和`dirname`。

本章将涵盖以下主题：

+   参数扩展

+   变量操作

# 技术要求

本章的所有脚本都可以在 GitHub 上找到，链接如下：[`github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter16`](https://github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter16)。对于这最后一个常规章节，你的 Ubuntu 虚拟机应该能再次帮助你度过难关。

# 参数扩展

在倒数第二章中，最后一章是技巧和窍门，我们将讨论 Bash 的一个非常酷的功能：*参数扩展*。

我们将首先对术语进行一些说明。首先，在 Bash 中被认为是*参数扩展*的东西不仅仅涉及到脚本提供的参数/参数：我们将在本章讨论的所有特殊操作都适用于 Bash *变量*。在官方 Bash 手册页（`man bash`）中，所有这些都被称为参数。

对于脚本的位置参数，甚至带参数的选项，这是有意义的。然而，一旦我们进入由脚本创建者定义的常量领域，常量/变量和参数之间的区别就有点模糊了。这并不重要；只要记住，当你在`man page`中看到*参数*这个词时，它可能是指一般的变量。

其次，人们对术语*参数扩展*和*参数替换*有些困惑，在互联网上这两个术语经常被交替使用。在官方文档中，*替换*这个词只用在*命令替换*和*进程替换*中。

命令替换是我们讨论过的：它是`$(...)`的语法。进程替换非常高级，还没有描述过：如果你遇到`<(...)`的语法，那就是在处理进程替换。我们在本章的*进一步阅读*部分包括了一篇关于进程替换的文章，所以一定要看一下。

我们认为混淆的根源在于*参数替换*，也就是在运行时用变量名替换其值，只被认为是 Bash 中更大的*参数扩展*的一小部分。这就是为什么你会看到一些文章或来源将参数扩展的所有伟大功能（默认值、大小写操作和模式删除等）称为参数替换。

再次强调，这些术语经常被互换使用，人们（可能）谈论的是同一件事。如果你自己有任何疑问，我们建议在任何一台机器上打开 Bash 的`man page`，并坚持使用官方的称呼：*参数扩展*。

# 参数替换-回顾

虽然在这一点上可能并不是必要的，但我们想快速回顾一下参数替换，以便将其放在参数扩展的更大背景中。

正如我们在介绍中所述，并且你在整本书中都看到了，参数替换只是在运行时用变量的值替换变量。在命令行中，这看起来有点像下面这样：

```
reader@ubuntu:~/scripts/chapter_16$ export word=Script
reader@ubuntu:~/scripts/chapter_16$ echo ${word}
Script
reader@ubuntu:~/scripts/chapter_16$ echo "You're reading: Learn Linux Shell ${word}ing"
You're reading: Learn Linux Shell Scripting
reader@ubuntu:~/scripts/chapter_16$ echo "You're reading: Learn Linux Shell $wording"
You're reading: Learn Linux Shell 
```

通常在回顾中你不会学到任何新东西，但因为我们只是为了背景，我们设法在这里偷偷加入了一些新东西：`export`命令。`export`是一个 shell 内置命令（可以用`type -a export`找到），我们可以使用`help export`来了解它（这是获取所有 shell 内置命令信息的方法）。

当设置变量值时，我们并不总是需要使用`export`：在这种情况下，我们也可以只使用`word=Script`。通常情况下，当我们设置一个变量时，它只在当前的 shell 中可用。在我们的 shell 的分支中运行的任何进程都不会将环境的这一部分与它们一起分叉：它们无法看到我们为变量分配的值。

虽然这并不总是必要的，但你可能会在网上寻找答案时遇到`export`的使用，所以了解它是很好的！

其余的示例应该不言自明。我们为一个变量赋值，并在运行时使用参数替换（在这种情况下，使用`echo`）来替换变量名为实际值。

作为提醒，我们将向你展示为什么我们建议*始终*在变量周围包含花括号：这样可以确保 Bash 知道变量的名称从何处开始和结束。在最后的`echo`中，我们可能会忘记这样做，我们会发现变量被错误解析，文本打印不正确。虽然并非所有脚本都需要，但我们认为这样做看起来更好，是一个你应该始终遵循的良好实践。

就我们而言，只有我们在这里涵盖的内容属于*参数替换*。本章中的所有其他特性都是*参数扩展*，我们将相应地引用它们！

# 默认值

接下来是参数扩展！正如我们所暗示的，Bash 允许我们直接对变量进行许多酷炫的操作。我们将从看似简单的示例开始，为变量定义默认值。

在处理用户输入时，这样做会让你和脚本用户的生活都变得更加轻松：只要有一个合理的默认值，我们就可以确保使用它，而不是在用户没有提供我们想要的信息时抛出错误。

我们将重用我们最早的一个脚本，`interactive.sh`，来自第八章，*变量和用户输入*。这是一个非常简单的脚本，没有验证用户输入，因此容易出现各种问题。让我们更新一下，并包括我们的参数的新默认值，如下所示：

```
reader@ubuntu:~/scripts/chapter_16$ cp ../chapter_08/interactive-arguments.sh default-interactive-arguments.sh
reader@ubuntu:~/scripts/chapter_16$ vim default-interactive-arguments.sh 
reader@ubuntu:~/scripts/chapter_16$ cat default-interactive-arguments.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-12-16
# Description: Interactive script with default variables.
# Usage: ./interactive-arguments.sh <name> <location> <food>
#####################################

# Initialize the variables from passed arguments.
character_name=${1:-Sebastiaan}
location=${2:-Utrecht}
food=${3:-frikandellen}

# Compose the story.
echo "Recently, ${character_name} was seen in ${location} eating ${food}!"
```

我们现在不再仅仅使用`$1`，`$2`和`$3`来获取用户输入，而是使用`man bash`中定义的更复杂的语法，如下所示：

${parameter:-word}

**使用默认值。** 如果参数未设置或为空，将替换为 word 的扩展。否则，将替换为参数的值。

同样，你应该在这个上下文中将*参数*读作*变量*（即使在用户提供时，它实际上是参数的一个参数，但它也很可能是一个常量）。使用这种语法，如果变量未设置或为空（空字符串），则在破折号后面提供的值（在`man`页面中称为*word*）将被插入。

我们已经为所有三个参数做了这个，所以让我们看看这在实践中是如何工作的：

```
reader@ubuntu:~/scripts/chapter_16$ bash default-interactive-arguments.sh 
Recently, Sebastiaan was seen in Utrecht eating frikandellen!
reader@ubuntu:~/scripts/chapter_16$ bash default-interactive-arguments.sh '' Amsterdam ''
Recently, Sebastiaan was seen in Amsterdam eating frikandellen!
```

如果我们没有向脚本提供任何值，所有默认值都会被插入。如果我们提供了三个参数，其中两个只是空字符串（`''`），我们可以看到 Bash 仍然会为我们替换空字符串的默认值。然而，实际的字符串`Amsterdam`被正确输入到文本中，而不是`Utrecht`。

以这种方式处理空字符串通常是期望的行为，你也可以编写你的脚本以允许空字符串作为变量的默认值。具体如下：

```
reader@ubuntu:~/scripts/chapter_16$ cat /tmp/default-interactive-arguments.sh 
<SNIPPED>
character_name=${1-Sebastiaan}
location=${2-Utrecht}
food=${3-frikandellen}
<SNIPPED>

reader@ubuntu:~/scripts/chapter_16$ bash /tmp/default-interactive-arguments.sh '' Amsterdam
Recently,  was seen in Amsterdam eating frikandellen!
```

在这里，我们创建了一个临时副本来说明这个功能。当您从默认声明中删除冒号（`${1-word}`而不是`${1:-word}`）时，它不再为空字符串插入默认值。但是，对于根本没有设置的值，它会插入默认值，当我们使用`'' Amsterdam`而不是`'' Amsterdam ''`调用它时可以看到。

根据我们的经验，在大多数情况下，默认值应忽略空字符串，因此`man page`中呈现的语法更可取。不过，如果您有一个特殊情况，现在您已经意识到了这种可能性！

对于您的一些脚本，您可能会发现仅替换默认值是不够的：您可能更愿意将变量设置为可以更细致评估的值。这也是可能的，使用参数扩展，如下所示：

${parameter:=word}

分配默认值。如果参数未设置或为空，则将单词的扩展分配给参数。然后替换参数的值。不能以这种方式分配位置参数和特殊参数。

我们从未见过需要使用此功能，特别是因为它与位置参数不兼容（因此，我们只在这里提到它，不详细介绍）。但是，与所有事物一样，了解参数扩展在这个领域提供的可能性是很好的。

# 输入检查

与使用参数扩展设置默认值密切相关，我们还可以使用参数扩展来显示如果变量为空或为空则显示错误。到目前为止，我们通过在脚本中实现 if-then 逻辑来实现这一点。虽然这是一个很好且灵活的解决方案，但有点冗长，特别是如果您只对用户提供参数感兴趣的话。

让我们创建我们之前示例的新版本：这个版本不提供默认值，但会在缺少位置参数时提醒用户。

我们将使用以下语法：

${parameter:?word}

如果参数为空或未设置，则将单词的扩展（或者如果单词不存在，则写入相应的消息）写入标准错误和 shell，如果不是交互式的，则退出。否则，替换参数的值。

当我们在脚本中使用这个时，它可能看起来像这样：

```
reader@ubuntu:~/scripts/chapter_16$ cp default-interactive-arguments.sh check-arguments.sh
reader@ubuntu:~/scripts/chapter_16$ vim check-arguments.sh eader@ubuntu:~/scripts/chapter_16$ cat check-arguments.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-12-16
# Description: Script with parameter expansion input checking.
# Usage: ./check-arguments.sh <name> <location> <food>
#####################################

# Initialize the variables from passed arguments.
character_name=${1:?Name not supplied!}
location=${2:?Location not supplied!}
food=${3:?Food not supplied!}

# Compose the story.
echo "Recently, ${character_name} was seen in ${location} eating ${food}!"
```

再次注意冒号。与前面的示例中冒号的工作方式相同，它还会强制此参数扩展将空字符串视为 null/未设置值。

当我们运行这个脚本时，我们会看到以下内容：

```
reader@ubuntu:~/scripts/chapter_16$ bash check-arguments.sh 
check-arguments.sh: line 12: 1: Name not supplied!
reader@ubuntu:~/scripts/chapter_16$ bash check-arguments.sh Sanne
check-arguments.sh: line 13: 2: Location not supplied!
reader@ubuntu:~/scripts/chapter_16$ bash check-arguments.sh Sanne Alkmaar
check-arguments.sh: line 14: 3: Food not supplied!
reader@ubuntu:~/scripts/chapter_16$ bash check-arguments.sh Sanne Alkmaar gnocchi
Recently, Sanne was seen in Alkmaar eating gnocchi!
reader@ubuntu:~/scripts/chapter_16$ bash check-arguments.sh Sanne Alkmaar ''
check-arguments.sh: line 14: 3: Food not supplied!
```

虽然这样做效果很好，但看起来并不是那么好，对吧？打印了脚本名称和行号，这对于脚本的用户来说似乎是太多深入的信息。

您可以决定您是否认为这些是可以接受的反馈消息给您的用户；就个人而言，我们认为一个好的 if-then 通常更好，但是在简洁的脚本方面，这是无法超越的。

还有另一个与此密切相关的参数扩展：`${parameter:+word}`。这允许您仅在参数不为空时使用*word*。根据我们的经验，这并不常见，但对于您的脚本需求可能会有用；在`man bash`中查找`Use Alternate Value`以获取更多信息。

# 参数长度

到目前为止，我们在书中进行了很多检查。然而，我们没有进行的一个是所提供参数的长度。在这一点上，您可能不会感到惊讶的是我们如何实现这一点：当然是通过参数扩展。语法也非常简单：

${#parameter}

参数长度。替换参数值的字符数。如果参数是*或@，则替换的值是位置参数的数量。

所以，我们将使用`${#variable}`而不是`${variable}`来打印，后者会在运行时替换值，而前者会给我们一个数字：值中的字符数。这可能有点棘手，因为空格等内容也可以被视为字符。

看看下面的例子：

```
reader@ubuntu:~/scripts/chapter_16$ variable="hello"
reader@ubuntu:~/scripts/chapter_16$ echo ${#variable}
5
reader@ubuntu:~/scripts/chapter_16$ variable="hello there"
reader@ubuntu:~/scripts/chapter_16$ echo ${#variable}
11
```

正如你所看到的，单词`hello`被识别为五个字符；到目前为止一切顺利。当我们看看句子`hello there`时，我们可以看到两个分别有五个字母的单词。虽然你可能期望参数扩展返回`10`，但实际上它返回的是`11`。由于单词之间用空格分隔，你不应感到惊讶：这个空格是第 11 个字符。

如果我们回顾一下`man bash`页面上的语法定义，我们会看到以下有趣的细节：

如果参数是*或@，则替换的值是位置参数的数量。

还记得我们在本书的其余部分中使用`$#`来确定传递给脚本的参数数量吗？这实际上就是 Bash 参数扩展的工作，因为`${#*}`等于`$#!`

为了加深这些观点，让我们创建一个快速脚本，处理三个字母的首字母缩略词（我们个人最喜欢的缩略词类型）。目前，这个脚本的功能将仅限于验证和打印用户输入，但当我们到达本章的末尾时，我们将稍作修改，使其更加酷炫：

```
reader@ubuntu:~/scripts/chapter_16$ vim acronyms.sh 
reader@ubuntu:~/scripts/chapter_16$ cat acronyms.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-12-16
# Description: Verify argument length.
# Usage: ./acronyms.sh <three-letter-acronym>
#####################################

# Use full syntax for passed arguments check.
if [[ ${#*} -ne 1 ]]; then
  echo "Incorrect number of arguments!"
  echo "Usage: $0 <three-letter-acronym>"
  exit 1
fi

acronym=$1 # No need to default anything because of the check above.

# Check acronym length using parameter expansion.
if [[ ${#acronym} -ne 3 ]]; then
  echo "Acronym should be exactly three letters!"
  exit 2
fi

# All checks passed, we should be good.
echo "Your chosen three letter acronym is: ${acronym}. Nice!"
```

在这个脚本中，我们做了两件有趣的事情：我们使用了`${#*}`的完整语法来确定传递给我们脚本的参数数量，并使用`${#acronym}`检查了首字母缩略词的长度。因为我们使用了两种不同的检查，所以我们使用了两种不同的退出代码：对于错误的参数数量，我们使用`exit 1`，对于不正确的首字母缩略词长度，我们使用`exit 2`。

在更大、更复杂的脚本中，使用不同的退出代码可能会节省大量的故障排除时间，因此我们在这里提供了相关信息。

如果我们现在用不同的不正确和正确的输入运行我们的脚本，我们可以看到它按计划运行。

```
reader@ubuntu:~/scripts/chapter_16$ bash acronyms.sh 
Incorrect number of arguments!
Usage: acronyms.sh <three-letter-acronym>
reader@ubuntu:~/scripts/chapter_16$ bash acronyms.sh SQL
Your chosen three letter acronym is: SQL. Nice!
reader@ubuntu:~/scripts/chapter_16$ bash acronyms.sh SQL DBA
Incorrect number of arguments!
Usage: acronyms.sh <three-letter-acronym>
reader@ubuntu:~/scripts/chapter_16$ bash acronyms.sh TARDIS
Acronym should be exactly three letters
```

没有参数，太多参数，参数长度不正确：我们已经准备好处理用户可能抛给我们的一切。一如既往，永远不要指望用户会按照你的期望去做，只需确保你的脚本只有在输入正确时才会执行！

# 变量操作

Bash 中的参数扩展不仅涉及默认值、输入检查和参数长度，它实际上还允许我们在使用变量之前操纵这些变量。在本章的第二部分中，我们将探讨参数扩展中处理*变量操作*（我们的术语；就 Bash 而言，这些只是普通的参数扩展）的能力。

我们将以*模式替换*开始，这是我们在第十章中对`sed`的解释后应该熟悉的内容。

# 模式替换

简而言之，模式替换允许我们用其他东西替换模式（谁会想到呢！）。这就是我们之前用`sed`已经能做的事情：

```
reader@ubuntu:~/scripts/chapter_16$ echo "Hi"
Hi
reader@ubuntu:~/scripts/chapter_16$ echo "Hi" | sed 's/Hi/Bye/'
Bye
```

最初，我们的`echo`包含单词`Hi`。然后我们通过`sed`进行管道传输，在其中查找*模式* `Hi`，我们将用`Bye` *替换*它。`sed`指令前面的`s`表示我们正在搜索和替换。

看吧，当`sed`解析完流之后，我们的屏幕上就会出现`Bye`。

如果我们想在使用变量时做同样的事情，我们有两个选择：要么像之前一样通过`sed`解析它，要么转而使用我们的新朋友进行另一次很棒的参数扩展：

${parameter/pattern/string}

**模式替换。** 模式会扩展成与路径名扩展中一样的模式。参数会被扩展，模式与其值的最长匹配将被替换为字符串。如果模式以/开头，则所有模式的匹配都将被替换为字符串。

因此，对于`${sentence}`变量，我们可以用`${sentence/pattern/string}`替换模式的第一个实例，或者用`${sentence//pattern/string}`替换所有实例（注意额外的斜杠）。

在命令行上，它可能看起来像这样：

```
reader@ubuntu:~$ sentence="How much wood would a woodchuck chuck if a woodchuck could chuck wood?"
reader@ubuntu:~$ echo ${sentence}
How much wood would a woodchuck chuck if a woodchuck could chuck wood?
reader@ubuntu:~$ echo ${sentence/wood/stone}
How much stone would a woodchuck chuck if a woodchuck could chuck wood?
reader@ubuntu:~$ echo ${sentence//wood/stone}
How much stone would a stonechuck chuck if a stonechuck could chuck stone reader@ubuntu:~$ echo ${sentence}
How much wood would a woodchuck chuck if a woodchuck could chuck wood?
```

再次强调，这是非常直观和简单的。

一个重要的事实是，这种参数扩展实际上并不编辑变量的值：它只影响当前的替换。如果您想对变量进行永久操作，您需要再次将结果写入变量，如下所示：

```
reader@ubuntu:~$ sentence_mutated=${sentence//wood/stone}
reader@ubuntu:~$ echo ${sentence_mutated}
How much stone would a stonechuck chuck if a stonechuck could chuck stone?
```

或者，如果您希望在变异后保留变量名称，可以将变异值一次性赋回变量，如下所示：

```
reader@ubuntu:~$ sentence=${sentence//wood/stone}
reader@ubuntu:~$ echo ${sentence}
How much stone would a stonechuck chuck if a stonechuck could chuck stone?
```

想象在脚本中使用这种语法应该不难。举个简单的例子，我们创建了一个小型交互式测验，在其中，如果用户给出了错误答案，我们将*帮助*他们：

```
reader@ubuntu:~/scripts/chapter_16$ vim forbidden-word.sh
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-12-16
# Description: Blocks the use of the forbidden word!
# Usage: ./forbidden-word.sh
#####################################

read -p "What is your favorite shell? " answer

echo "Great choice, my favorite shell is also ${answer/zsh/bash}!"

reader@ubuntu:~/scripts/chapter_16$ bash forbidden-word.sh 
What is your favorite shell? bash
Great choice, my favorite shell is also bash!
reader@ubuntu:~/scripts/chapter_16$ bash forbidden-word.sh 
What is your favorite shell? zsh
Great choice, my favorite shell is also bash!
```

在这个脚本中，如果用户暂时*困惑*并且没有给出想要的答案，我们将简单地用*正确*答案`bash`替换他们的*错误*答案（`zsh`）。

开玩笑的时候到此为止，其他 shell（如`zsh`，`ksh`，甚至较新的 fish）都有自己独特的卖点和优势，使一些用户更喜欢它们而不是 Bash 进行日常工作。这显然很好，也是使用 Linux 的心态的一部分：您有自由选择您喜欢的软件！

然而，当涉及到脚本时，我们（显然）认为 Bash 仍然是 shell 之王，即使只是因为它已经成为大多数发行版的事实标准 shell。这在可移植性和互操作性方面非常有帮助，这些特性通常对脚本有益。

# 模式删除

与模式替换紧密相关的一个主题是*模式删除*。让我们面对现实，模式删除基本上就是用空白替换模式。

如果模式删除与模式替换具有完全相同的功能，我们就不需要它。但是，模式删除有一些很酷的技巧，使用模式替换可能会很困难，甚至不可能做到。

模式删除有两个选项：删除匹配模式的*前缀*或*后缀*。简单来说，它允许您从开头或结尾删除内容。它还有一个选项，可以在找到第一个匹配模式后停止，或者一直持续到最后。

没有一个好的例子，这可能有点太抽象（对我们来说，第一次遇到这种情况时肯定是这样）。然而，这里有一个很好的例子：这一切都与文件有关：

```
reader@ubuntu:/tmp$ touch file.txt
reader@ubuntu:/tmp$ file=/tmp/file.txt
reader@ubuntu:/tmp$ echo ${file}
/tmp/file.txt
```

我们创建了一个包含对文件的引用的变量。如果我们想要目录，或者不带目录的文件，我们可以使用`basename`或`dirname`，如下所示：

```
reader@ubuntu:/tmp$ basename ${file}
file.txt
reader@ubuntu:/tmp$ dirname ${file}
/tmp
```

我们也可以通过参数扩展来实现这一点。前缀和后缀删除的语法如下：

${parameter#word}

${parameter##word}

**删除匹配前缀模式。** ${parameter%word}${parameter%%word} **删除匹配后缀模式。**

对于我们的`${file}`变量，我们可以使用参数扩展来删除所有目录，只保留文件名，如下所示：

```
reader@ubuntu:/tmp$ echo ${file#/}
tmp/file.txt
reader@ubuntu:/tmp$ echo ${file#*/}
tmp/file.txt
reader@ubuntu:/tmp$ echo ${file##/}
tmp/file.txt
reader@ubuntu:/tmp$ echo ${file##*/}
file.txt
```

第一条和第二条命令之间的区别很小：我们使用了可以匹配任何内容零次或多次的星号通配符。在这种情况下，由于变量的值以斜杠开头，它不匹配。然而，一旦我们到达第三个命令，我们就看到了需要包括它：我们需要匹配*我们想要删除的所有内容*。

在这种情况下，`*/`模式匹配`/tmp/`，而`/`模式仅匹配第一个正斜杠（正如第三个命令的结果清楚显示的那样）。

值得记住的是，在这种情况下，我们仅仅是使用参数扩展来替换`basename`命令的功能。然而，如果我们不是在处理文件引用，而是（例如）下划线分隔的文件，我们就无法用`basename`来实现这一点，参数扩展就会派上用场！

既然我们已经看到了前缀的用法，让我们来看看后缀。功能是一样的，但是不是从值的开头解析，而是先从值的末尾开始。例如，我们可以使用这个功能从文件中删除扩展名：

```
reader@ubuntu:/tmp$ file=file.txt
reader@ubuntu:/tmp$ echo ${file%.*}
file
```

这使我们能够获取文件名，不包括扩展名。如果你的脚本中有一些逻辑可以应用到文件的这一部分，这可能是可取的。根据我们的经验，这比你想象的要常见！

例如，你可能想象一下备份文件名中有一个日期，你想将其与今天的日期进行比较，以确保备份成功。一点点的参数扩展就可以让你得到你想要的格式，这样日期的比较就变得微不足道了。

就像我们能够替换`basename`命令一样，我们也可以使用后缀模式删除来找到`dirname`，如下所示：

```
reader@ubuntu:/tmp$ file=/tmp/file.txt
reader@ubuntu:/tmp$ echo ${file%/*}
/tmp
```

再次强调，这些示例主要用于教育目的。有许多情况下这可能会有用；由于这些情况非常多样化，很难给出一个对每个人都有趣的例子。

然而，我们介绍的关于备份的情况可能对你有用。作为一个基本的脚本，它看起来会是这样的：

```
reader@ubuntu:~/scripts/chapter_16$ vim check-backup.sh
reader@ubuntu:~/scripts/chapter_16$ cat check-backup.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-12-16
# Description: Check if daily backup has succeeded.
# Usage: ./check-backup.sh <file>
#####################################

# Format the date: yyyymmdd.
DATE_FORMAT=$(date +%Y%m%d)

# Use basename to remove directory, expansion to remove extension.
file=$(basename ${1%%.*}) # Double %% so .tar.gz works too.

if [[ ${file} == "backup-${DATE_FORMAT}" ]]; then
  echo "Backup with todays date found, all good."
  exit 0 # Successful.
else
  echo "No backup with todays date found, please double check!"
  exit 1 # Unsuccessful.
fi

reader@ubuntu:~/scripts/chapter_16$ touch /tmp/backup-20181215.tar.gz
reader@ubuntu:~/scripts/chapter_16$ touch /tmp/backup-20181216.tar.gz
reader@ubuntu:~/scripts/chapter_16$ bash -x check-backup.sh /tmp/backup-20181216.tar.gz 
++ date +%Y%m%d
+ DATE_FORMAT=20181216
++ basename /tmp/backup-20181216
+ file=backup-20181216
+ [[ backup-20181216 == backup-20181216 ]]
+ echo 'Backup with todays date found, all good.'
Backup with todays date found, all good.
+ exit 0
reader@ubuntu:~/scripts/chapter_16$ bash check-backup.sh /tmp/backup-20181215.tar.gz 
No backup with todays date found, please double check!
```

为了说明这一点，我们正在创建虚拟备份文件。在实际情况下，你更有可能在目录中挑选最新的文件（例如使用`ls -ltr /backups/ | awk '{print $9}' | tail -1`）并将其与当前日期进行比较。

与 Bash 脚本中的大多数事物一样，还有其他方法可以完成这个日期检查。你可以说我们可以保留文件变量中的扩展名，并使用解析日期的正则表达式：这样也可以，工作量几乎相同。

这个例子（以及整本书）的要点应该是使用对你和你的组织有用的东西，只要你以稳固的方式构建它，并为每个人添加必要的注释，让大家都能理解你做了什么！

# 大小写修改

接下来是另一个参数扩展，我们已经简要看到了：*大小写修改*。在这种情况下，大小写是指小写和大写字母。

在我们最初在第九章中创建的`yes-no-optimized.sh`脚本中，*错误检查和处理*，我们有以下指令：

```
reader@ubuntu:~/scripts/chapter_09$ cat yes-no-optimized.sh 
<SNIPPED>
read -p "Do you like this question? " reply_variable

# See if the user responded positively.
if [[ ${reply_variable,,} = 'y' || ${reply_variable,,} = 'yes' ]]; then
  echo "Great, I worked really hard on it!"
  exit 0
fi

# Maybe the user responded negatively?
if [[ ${reply_variable^^} = 'N' || ${reply_variable^^} = 'NO' ]]; then
  echo "You did not? But I worked so hard on it!"
  exit 0
fi
```

正如你所期望的那样，在变量的花括号中找到的`,,`和`^^`是我们所讨论的参数扩展。

如`man bash`中所述的语法如下：

${parameter^pattern}

${parameter^^pattern}

${parameter,pattern}

${parameter,,pattern}

**大小写修改。** 这个扩展修改参数中字母字符的大小写。模式被扩展以产生一个与路径名扩展中一样的模式。参数的扩展值中的每个字符都与模式进行匹配，如果匹配模式，则其大小写被转换。模式不应尝试匹配多于一个字符。 

在我们的第一个脚本中，我们没有使用模式。当不使用模式时，暗示着模式是通配符（在这种情况下是`?`），这意味着一切都匹配。

快速的命令行示例可以清楚地说明如何进行大小写修改。首先，让我们看看如何将变量转换为大写：

```
reader@ubuntu:~/scripts/chapter_16$ string=yes
reader@ubuntu:~/scripts/chapter_16$ echo ${string}
yes
reader@ubuntu:~/scripts/chapter_16$ echo ${string^}
Yes
reader@ubuntu:~/scripts/chapter_16$ echo ${string^^}
YES
```

如果我们使用单个插入符（`^`），我们可以看到我们变量值的第一个字母将变成大写。如果我们使用双插入符，`^^`，我们现在有了全部大写的值。

以类似的方式，逗号也可以用于小写：

```
reader@ubuntu:~/scripts/chapter_16$ STRING=YES
reader@ubuntu:~/scripts/chapter_16$ echo ${STRING}
YES
reader@ubuntu:~/scripts/chapter_16$ echo ${STRING,}
yES
reader@ubuntu:~/scripts/chapter_16$ echo ${STRING,,}
yes
```

因为我们可以选择将整个值大写或小写，所以现在我们可以更容易地将用户输入与预定义值进行比较。无论用户输入`YES`，`Yes`还是`yes`，我们都可以通过单个检查来验证所有这些情况：`${input,,} == 'yes'`。

这可以减少用户的头疼，而一个快乐的用户正是我们想要的（记住，你经常是你自己脚本的用户，你也应该快乐！）。

现在，关于*模式*，就像`man page`指定的那样。根据我们的个人经验，我们还没有使用过这个选项，但它是强大和灵活的，所以多解释一点也没有坏处。

基本上，只有在模式匹配时才会执行大小写修改。这可能有点棘手，但你可以看到它是如何工作的：

```
reader@ubuntu:~/scripts/chapter_16$ animal=salamander
reader@ubuntu:~/scripts/chapter_16$ echo ${animal^a}
salamander
reader@ubuntu:~/scripts/chapter_16$ echo ${animal^^a}
sAlAmAnder
reader@ubuntu:~/scripts/chapter_16$ echo ${animal^^ae}
salamander
reader@ubuntu:~/scripts/chapter_16$ echo ${animal^^[ae]}
sAlAmAndEr
```

我们运行的第一个命令`${animal^a}`，只有在匹配模式`a`时才会将第一个字母大写。由于第一个字母实际上是`s`，整个单词被打印为小写。

对于下一个命令`${animal^^a}`，*所有匹配的字母*都会被大写。因此，单词`salamander`中的所有三个`a`实例都会变成大写。

在第三个命令中，我们尝试向模式添加一个额外的字母。由于这不是正确的做法，参数扩展（可能）试图找到一个单个字母来匹配模式中的两个字母。剧透警告：这是不可能的。一旦我们将一些正则表达式专业知识融入其中，我们就可以做我们想做的事情：通过使用`[ae]`，我们指定`a`和`e`都是大小写修改操作的有效目标。

最后，返回的动物现在是`sAlAmAndEr`，所有元音字母都使用自定义模式和大小写修改参数扩展为大写！

作为一个小小的奖励，我们想分享一个甚至在`man bash`页面上都没有的大小写修改！它也不是那么复杂。如果你用波浪号`~`替换逗号`,`或插入符`^`，你将得到一个*大小写反转*。正如你可能期望的那样，单个波浪号只会作用于第一个字母（如果匹配模式的话），而双波浪号将匹配模式的所有实例（如果没有指定模式并且使用默认的`?`）。

看一下：

```
reader@ubuntu:~/scripts/chapter_16$ name=Sebastiaan
reader@ubuntu:~/scripts/chapter_16$ echo ${name}
Sebastiaan
reader@ubuntu:~/scripts/chapter_16$ echo ${name~}
sebastiaan
reader@ubuntu:~/scripts/chapter_16$ echo ${name~~}
sEBASTIAAN reader@ubuntu:~/scripts/chapter_16$ echo ${name~~a}
SebAstiAAn
```

这应该足够解释大小写修改，因为所有的语法都是相似和可预测的。

现在你知道如何将变量转换为小写、大写，甚至反转大小写，你应该能够以任何你喜欢的方式改变它们，特别是如果你加入一个模式，这个参数扩展提供了许多可能性！

# 子字符串扩展

关于参数扩展，只剩下一个主题：子字符串扩展。虽然你可能听说过子字符串，但它也可能是一个非常复杂的术语。

幸运的是，这实际上是*非常非常*简单的。如果我们拿一个字符串，比如*今天是一个伟大的一天*，那么这个句子的任何部分，只要顺序正确但不是完整的句子，都可以被视为完整字符串的子字符串。例如：

+   今天是

+   一个伟大的一天

+   day is a gre

+   今天是一个伟大的一天

+   o

+   （<- 这里有一个空格，你只是看不到它）

从这些例子中可以看出，我们并不关注句子的语义意义，而只是关注字符：任意数量的字符按正确的顺序可以被视为子字符串。这包括整个句子减去一个字母，但也包括单个字母，甚至是单个空格字符。

因此，让我们最后一次看一下这个参数扩展的语法：

${parameter:offset}

${parameter:offset:length}

**子字符串扩展。** 从偏移量指定的字符开始，将参数值的长度扩展到长度个字符。

基本上，我们指定了子字符串应该从哪里开始，以及应该有多长（以字符为单位）。与大多数计算机一样，第一个字符将被视为`0`（而不是任何非技术人员可能期望的`1`）。如果我们省略长度，我们将得到偏移量之后的所有内容；如果我们指定了长度，我们将得到确切数量的字符。

让我们看看这对我们的句子会怎么样：

```
reader@ubuntu:~/scripts/chapter_16$ sentence="Today is a great day"
reader@ubuntu:~/scripts/chapter_16$ echo ${sentence}
Today is a great day
reader@ubuntu:~/scripts/chapter_16$ echo ${sentence:0:5}
Today
reader@ubuntu:~/scripts/chapter_16$ echo ${sentence:1:6}
oday is
reader@ubuntu:~/scripts/chapter_16$ echo ${sentence:11}
great day
```

在我们的命令行示例中，我们首先创建包含先前给定文本的`${sentence}`变量。首先，我们完全`echo`它，然后我们使用`${sentence:0:5}`只打印前五个字符（记住，字符串从 0 开始！）。

接下来，我们打印从第二个字符开始的前六个字符（由`:1:6`表示）。在最后一个命令中，`echo ${sentence:11}`显示我们也可以在不指定长度的情况下使用子字符串扩展。在这种情况下，Bash 将简单地打印从偏移量到变量值结束的所有内容。

我们想以前面承诺的方式结束本章：我们的三个字母缩写脚本。现在我们知道如何轻松地从用户输入中提取单独的字母，创建一个咒语会很有趣！

让我们修改脚本：

```
reader@ubuntu:~/scripts/chapter_16$ cp acronyms.sh acronym-chant.sh
reader@ubuntu:~/scripts/chapter_16$ vim acronym-chant.sh
reader@ubuntu:~/scripts/chapter_16$ cat acronym-chant.sh
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-12-16
# Description: Verify argument length, with a chant!
# Usage: ./acronym-chant.sh <three-letter-acronym>
#####################################
<SNIPPED>

# Split the string into three letters using substring expansion.
first_letter=${acronym:0:1}
second_letter=${acronym:1:1}
third_letter=${acronym:2:1}

# Print our chant.
echo "Give me the ${first_letter^}!"
echo "Give me the ${second_letter^}!"
echo "Give me the ${third_letter^}!"

echo "What does that make? ${acronym^^}!"
```

我们还加入了一些大小写修改以确保万无一失。在我们使用子字符串扩展拆分字母之后，我们无法确定用户呈现给我们的大小写。由于这是一首咒语，我们假设大写不是一个坏主意，我们将所有内容都转换为大写。

对于单个字母，一个插入符就足够了。对于完整的首字母缩写，我们使用双插入符，以便所有三个字符都是大写。使用`${acronym:0:1}`、`${acronym:1:1}`和`${acronym:2:1}`的子字符串扩展，我们能够获得单个字母（因为*长度*总是 1，但偏移量不同）。

为了重要的可读性，我们将这些字母分配给它们自己的变量，然后再使用它们。我们也可以直接在`echo`中使用`${acronym:0:1}`，但由于这个脚本不太长，我们选择了更冗长的额外变量选项，其中名称透露了我们通过子字符串扩展实现的目标。

最后，让我们运行这个最后的脚本，享受我们的个人咒语：

```
reader@ubuntu:~/scripts/chapter_16$ bash acronym-chant.sh Sql
Give me the S!
Give me the Q!
Give me the L!
What does that make? SQL!
reader@ubuntu:~/scripts/chapter_16$ bash acronym-chant.sh dba
Give me the D!
Give me the B!
Give me the A!
What does that make? DBA!
reader@ubuntu:~/scripts/chapter_16$ bash acronym-chant.sh USA
Give me the U!
Give me the S!
Give me the A!
What does that make? USA!
```

大小写混合，小写，大写，都无所谓：无论用户输入什么，只要是三个字符，我们的咒语就能正常工作。好东西！谁知道子字符串扩展可以如此方便呢？

一个非常高级的参数扩展功能是所谓的*参数转换*。它的语法`${parameter@operator}`允许对参数执行一些复杂的操作。要了解这可以做什么，转到`man bash`并查找参数转换。你可能永远不需要它，但功能确实很酷，所以绝对值得一看！

# 总结

在本章中，我们讨论了 Bash 中的参数扩展。我们首先回顾了我们如何在本书的大部分内容中使用参数替换，以及参数替换只是 Bash 参数扩展的一小部分。

我们继续向你展示如何使用参数扩展来包括变量的默认值，以防用户没有提供自己的值。这个功能还允许我们在输入缺失时向用户呈现错误消息，尽管不是最干净的方式。

我们通过展示如何使用这个来确定变量值的长度来结束了参数扩展的介绍，并且我们向你展示了我们在书中已经广泛使用了这个形式的`$#`语法。

我们在“变量操作”标题下继续描述参数扩展的功能。这包括“模式替换”的功能，它允许我们用另一个字符串替换变量值的一部分（“模式”）。在非常相似的功能中，“模式删除”允许我们删除与模式匹配的部分值。

接下来，我们向您展示了如何将字符从小写转换为大写，反之亦然。这个功能在本书的早期已经提到，但现在我们已经更深入地解释了它。

我们以“子字符串扩展”结束了本章，它允许我们从“偏移量”和/或指定的“长度”中获取变量的部分。

本章介绍了以下命令：`export`和`dirname`。

# 问题

1.  什么是参数替换？

1.  我们如何为已定义的变量包含默认值？

1.  我们如何使用参数扩展来处理缺失的参数值？

1.  `${#*}`是什么意思？

1.  在谈论参数扩展时，模式替换是如何工作的？

1.  模式删除与模式替换有什么关系？

1.  我们可以执行哪些类型的大小写修改？

1.  我们可以使用哪两种方法从变量的值中获取子字符串？

# 进一步阅读

有关本章主题的更多信息，请参考以下链接：

+   TLDP 关于进程替换：[`www.tldp.org/LDP/abs/html/process-sub.html`](http://www.tldp.org/LDP/abs/html/process-sub.html)

+   TLDP 关于参数替换的内容：[`www.tldp.org/LDP/abs/html/parameter-substitution.html`](https://www.tldp.org/LDP/abs/html/parameter-substitution.html)

+   GNU 关于参数扩展：[`www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html`](https://www.gnu.org/software/bash/manual/html_node/Shell-Parameter-Expansion.html)
