# 第十五章：使用 `getopts` 解析 Bash 脚本参数

在本章中，我们将讨论向脚本传递参数的不同方法，特别关注标志。我们将首先回顾位置参数，然后继续讨论作为标志传递的参数。之后，我们将讨论如何使用 `getopts` shell 内建在你自己的脚本中使用标志。

本章将介绍以下命令：`getopts` 和 `shift`。

本章将涵盖以下主题：

+   位置参数与标志

+   `getopts` shell 内建

# 技术要求

本章的所有脚本都可以在 GitHub 上找到，链接如下：[`github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter15`](https://github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter15)。在你的 Ubuntu Linux 虚拟机上跟着示例进行—不需要其他资源。对于 `single-flag.sh` 脚本，只能在网上找到最终版本。在执行脚本之前，请务必验证头部中的脚本版本。

# 位置参数与标志

我们将从一个简短的位置参数回顾开始本章。你可能还记得来自第八章的*变量和用户输入*，我们可以使用位置参数来向我们的脚本传递参数。

简单来说，使用以下语法：

```
bash script.sh argument1 argument2 ...
```

在上述（虚构的）`script.sh` 中，我们可以通过查看参数的位置来获取用户提供的值：`$1` 是第一个参数，`$2` 是第二个参数，依此类推。记住 `$0` 是一个特殊的参数，它与脚本的名称有关：在这种情况下，是 `script.sh`。

这种方法相对简单，但也容易出错。当你编写这个脚本时，你需要对用户提供的输入进行广泛的检查；他们是否提供了足够的参数，但不要太多？或者，也许一些参数是可选的，所以可能有一些组合是可能的？所有这些事情都需要考虑，如果可能的话，需要处理。

除了脚本作者（你！），脚本调用者也有负担。在他们能够成功调用你的脚本之前，他们需要知道如何传递所需的信息。对于我们的脚本，我们应用了两种旨在减轻用户负担的做法：

+   我们的脚本头包含一个 `Usage:` 字段

+   当我们的脚本被错误调用时，我们会打印一个错误消息，带有一个与头部类似/相等的*使用提示*

然而，这种方法容易出错，而且并不总是很用户友好。不过，还有另一个选择：*选项*，更常被称为*标志*。

# 在命令行上使用标志

也许你还没有意识到，但你在命令行上使用的大多数命令都是使用位置参数和标志的组合。Linux 中最基本的命令 `cd` 使用了一个位置参数：你想要移动到的目录。

实际上它确实有两个标志，你也可以使用：`-L` 和 `-P`。这些标志的目的是小众的，不值得在这里解释。几乎所有命令都同时使用标志和位置参数。

那么，我们什么时候使用哪个？作为一个经验法则，标志通常用于*修改器*，而位置参数用于*目标*。目标很简单：你想要用命令操作的东西。在 `ls` 的情况下，这意味着位置参数是应该被列出（操作）的文件或目录。

对于`ls -l /tmp/`命令，`/tmp/`是目标，`-l`是用来修改`ls`行为的标志。默认情况下，`ls`列出所有文件，不包括所有者、权限、大小等额外信息。如果我们想要修改`ls`的行为，我们添加一个或多个标志：`-l`告诉`ls`使用长列表格式，这样每个文件都会单独打印在自己的行上，并打印有关文件的额外信息。

请注意，在`ls /tmp/`和`ls -l /tmp/`之间，目标没有改变，但输出却改变了，因为我们用标志*修改*了它！

有些标志甚至更特殊：它们需要自己的位置参数！因此，我们不仅可以使用标志来修改命令，而且标志本身还有多个选项来修改命令的行为。

一个很好的例子是`find`命令：默认情况下，它会在目录中查找所有文件，如下所示：

```
reader@ubuntu:~/scripts/chapter_14$ find
.
./reverser-crontab
./wall.txt
./base-crontab
./date-redirection-crontab
```

或者，我们可以使用`find`与位置参数一起使用，以便不在当前工作目录中搜索，而是在其他地方搜索，如下所示：

```
reader@ubuntu:~/scripts/chapter_14$ find ../chapter_10
../chapter_10
../chapter_10/error.txt
../chapter_10/grep-file.txt
../chapter_10/search.txt
../chapter_10/character-class.txt
../chapter_10/grep-then-else.sh
```

现在，`find`还允许我们使用`-type`标志只打印特定类型的文件。但是仅使用`-type`标志，我们还没有指定要打印的文件类型。通过在标志之后直接指定文件类型（这里*关键*是顺序），我们告诉标志要查找什么。它看起来像下面这样：

```
reader@ubuntu:/$ find /boot/ -type d
/boot/
/boot/grub
/boot/grub/i386-pc
/boot/grub/fonts
/boot/grub/locale
```

在这里，我们在`/boot/`目录中寻找了一种`d`（目录）类型。`-type`标志的其他参数包括`f`（文件）、`l`（符号链接）和`b`（块设备）。

像这样的事情会发生，如果你没有做对的话：

```
reader@ubuntu:/$ find -type d /boot/
find: paths must precede expression: '/boot/'
find: possible unquoted pattern after predicate '-type'?
```

不幸的是，不是所有的命令都是平等的。有些对用户更宽容，尽力理解输入的内容。其他则更加严格：它们会运行任何传递的内容，即使它没有任何功能上的意义。请务必确保您正确使用命令及其修改器！

前面的例子使用了与我们将学习如何在`getopts`中使用标志的方式不同。这些例子只是用来说明脚本参数、标志和带参数的标志的概念。这些实现是在没有使用`getopts`的情况下编写的，因此不完全对应我们以后要做的事情。

# 内置的 getopts shell

现在真正的乐趣开始了！在本章的第二部分中，我们将解释`getopts` shell 内置。`getopts`命令用于在脚本的开头获取您以标志形式提供的**选项**。它有一个非常特定的语法，一开始可能会让人感到困惑，但是，一旦我们完全了解了它，你应该就不会觉得太复杂了。

不过，在我们深入讨论之前，我们需要讨论两件事：

+   `getopts`和`getopt`之间的区别

+   短选项与长选项

如前所述，`getopts`是一个*shell 内置*。它在常规的 Bourne shell（`sh`）和 Bash 中都可用。它始于 1986 年左右，作为`getopt`的替代品，后者在 1980 年前后创建。

与`getopts`相比，`getopt`不是内置于 shell 中的：它是一个独立的程序，已经移植到许多不同的 Unix 和类 Unix 发行版。`getopts`和`getopt`之间的主要区别如下：

+   `getopt`不能很好地处理空标志参数；`getopts`可以

+   `getopts`包含在 Bourne shell 和 Bash 中；`getopt`需要单独安装

+   `getopt`允许解析长选项（`--help`而不是`-h`）；`getopts`不允许

+   `getopts`有更简单的语法；`getopt`更复杂（主要是因为它是一个外部程序，而不是内置的）。

一般来说，大多数情况下，使用`getopts`更可取（除非你真的想要长选项）。由于`getopts`是 Bash 内置的，我们也会使用它，特别是因为我们不需要长选项。

您在终端上使用的大多数命令都有短选项（在终端上交互工作时几乎总是使用，以节省时间）和长选项（更具描述性，更适合创建更易读的脚本）。根据我们的经验，短选项更常见，而且使用正确时更容易识别。

以下列表显示了最常见的短标志，对大多数命令起着相同的作用：

+   -h：打印命令的帮助/用法

+   -v：使命令详细

+   -q：使命令安静

+   -f <file>：将文件传递给<indexentry content="getopts shell builtin, flags:-f ">命令

+   -r：递归执行操作

+   -d：以调试模式运行命令

不要假设所有命令都解析短标志，如前所述。尽管对大多数命令来说是这样，但并非所有命令都遵循这些趋势。这里打印的内容是根据个人经验发现的，应始终在运行对您新的命令之前进行验证。也就是说，运行一个没有参数/标志或带有`-h`的命令，至少 90%的时间会打印正确的用法供您欣赏。

尽管长选项对我们的`getopts`脚本可用会很好，但是长选项永远不能替代编写可读性脚本和为使用您的脚本的用户创建良好提示。我们认为这比拥有长选项更重要！此外，`getopts`的语法比可比的`getopt`要干净得多，遵循 KISS 原则仍然是我们的目标之一。

# getopts 语法

我们不想在这一章中再花费更多时间而不看到实际的代码，我们将直接展示一个非常简单的`getopts`脚本示例。当然，我们会逐步引导您，以便您有机会理解它。

我们正在创建的脚本只做了一些简单的事情：如果找到`-v`标志，它会打印一个*详细*消息，告诉我们它找到了该标志。如果没有找到任何标志，它将不打印任何内容。如果找到任何其他标志，它将为用户打印错误。简单吧？

让我们来看一下：

```
reader@ubuntu:~/scripts/chapter_15$ vim single-flag.sh
reader@ubuntu:~/scripts/chapter_15$ cat !$
cat single-flag.sh
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-12-08
# Description: Shows the basic getopts syntax.
# Usage: ./single-flag.sh [flags]
#####################################

# Parse the flags in a while loop.
# After the last flag, getopts returns false which ends the loop.
optstring=":v"
while getopts ${optstring} options; do
  case ${options} in
    v)
      echo "-v was found!"
      ;;
    ?)
      echo "Invalid option: -${OPTARG}."
      exit 1
      ;; 
  esac
done
```

如果我们运行这个脚本，我们会看到以下情况发生：

```
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh # No flag, do nothing.
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -p 
Invalid option: -p. # Wrong flag, print an error.
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -v 
-v was found! # Correct flag, print the message.
```

因此，我们的脚本至少按预期工作！但是为什么它会这样工作呢？让我们来看看。我们将跳过标题，因为现在应该非常清楚。我们将从包含`getopts`命令和`optstring`的`while`行开始：

```
# Parse the flags in a while loop.
# After the last flag, getopts returns false which ends the loop.
optstring=":v"
while getopts ${optstring} options; do
```

`optstring`，很可能是***opt**ions **string***的缩写，告诉`getopts`应该期望哪些选项。在这种情况下，我们只期望`v`。然而，我们以一个冒号（`:`）开始`optstring`，这是`optstring`的一个特殊字符，它将`getopts`设置为*静默错误报告*模式。

由于我们更喜欢自己处理错误情况，我们将始终以冒号开头。但是，随时可以尝试删除冒号看看会发生什么。

之后，`getopts`的语法非常简单，如下所示：

```
getopts optstring name [arg]
```

我们可以看到命令，后面跟着`optstring`（我们将其抽象为一个单独的变量以提高可读性），最后是我们将存储解析结果的变量的名称。

`getopts`的最后一个可选方面允许我们传递我们自己的一组参数，而不是默认为传递给脚本的所有内容（$0 到$9）。我们在练习中不需要/使用这个，但这绝对是好事。与往常一样，因为这是一个 shell 内置命令，您可以通过执行`help getopts`来找到有关它的信息。

我们将此命令放在`while`循环中，以便它遍历我们传递给脚本的所有参数。如果`getopts`没有更多参数要解析，它将返回除`0`之外的退出状态，这将导致`while`循环退出。

然而，在循环中，我们将进入`case`语句。如你所知，`case`语句基本上是更好的语法，用于更长的`if-elif-elif-elif-else`语句。在我们的示例脚本中，它看起来像这样：

```
  case ${options} in
    v)
      echo "-v was found!"
      ;;
    ?)
      echo "Invalid option: -${OPTARG}."
      exit 1
      ;;
  esac
done
```

注意`case`语句以`esac`（case 反写）结束。对于我们定义的所有标志（目前只有`-v`），我们有一段代码块，只有对该标志才会执行。

当我们查看`${options}`变量时（因为我们在`getopts`命令中为*name*指定了它），我们还会发现`?`通配符。我们将它放在`case`语句的末尾，作为捕获错误的手段。如果它触发了`?)`代码块，我们向`getopts`提供了一个无法理解的标志。在这种情况下，我们打印一个错误并退出脚本。

最后一行的`done`结束了`while`循环，并表示我们所有的标志都应该已经处理完毕。

可能看起来有点多余，既有`optstring`又有所有可能选项的`case`。目前确实是这样，但在本章稍后的部分，我们将向您展示`optstring`用于指定除了字母之外的其他内容；到那时，`optstring`为什么在这里应该是清楚的。现在不要太担心它，只需在两个位置输入标志即可。

# 多个标志

幸运的是，我们不必满足于只有一个标志：我们可以定义许多标志（直到字母用完为止！）。

我们将创建一个新的脚本，向读者打印一条消息。如果没有指定标志，我们将打印默认消息。如果遇到`-b`标志或`-g`标志，我们将根据标志打印不同的消息。我们还将包括`-h`标志的说明，遇到时将打印帮助信息。

满足这些要求的脚本可能如下所示：

```
reader@ubuntu:~/scripts/chapter_15$ vim hey.sh 
reader@ubuntu:~/scripts/chapter_15$ cat hey.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-12-14
# Description: Getopts with multiple flags.
# Usage: ./hey.sh [flags]
#####################################

# Abstract the help as a function, so it does not clutter our script.
print_help() {
  echo "Usage: $0 [flags]"
  echo "Flags:"
  echo "-h for help."
  echo "-b for male greeting."
  echo "-g for female greeting."
}

# Parse the flags.
optstring=":bgh"
while getopts ${optstring} options; do
  case ${options} in
    b)
      gender="boy"
      ;;
    g)
      gender="girl"
      ;;
    h)
      print_help
      exit 0 # Stop script, but consider it a success.
      ;;
    ?)
      echo "Invalid option: -${OPTARG}."
      exit 1
      ;; 
  esac
done

# If $gender is n (nonzero), print specific greeting.
# Otherwise, print a neutral greeting.
if [[ -n ${gender} ]]; then
  echo "Hey ${gender}!"
else
  echo "Hey there!"
fi
```

在这一点上，这个脚本对你来说应该是可读的，尤其是包含的注释。从头开始，我们从标题开始，然后是`print_help()`函数，当遇到`-h`标志时打印我们的帮助信息（正如我们在几行后看到的那样）。

接下来是`optstring`，它仍然以冒号开头，以便关闭`getopts`的冗长错误（因为我们将自己处理这些错误）。在`optstring`中，我们将要处理的三个标志，即`-b`、`-g`和`-h`，定义为一个字符串：`bgh`。

对于每个标志，我们在`case`语句中都有一个条目：对于`b)`和`g)`，`gender`变量分别设置为`boy`或`girl`。对于`h)`，在调用`exit 0`之前，调用了我们定义的函数。（想想为什么我们要这样做！如果不确定，可以在不使用 exit 的情况下运行脚本。）

我们总是通过`?)`语法处理未知标志来结束`getopts`块。

继续，当我们的`case`语句以`esac`结束时，我们进入实际的功能。我们检查`gender`变量是否已定义：如果是，我们打印一个包含根据标志设置的值的消息。如果没有设置（即如果未指定`-b`和`-g`），我们打印一个省略性别的通用问候。

这也是为什么我们在找到`-h`后会`exit 0`：否则帮助信息和问候语都会显示给用户（这很奇怪，因为用户只是要求使用`-h`查看帮助页面）。

让我们看看我们的脚本是如何运行的：

```
reader@ubuntu:~/scripts/chapter_15$ bash hey.sh -h
Usage: hey.sh [flags]
Flags:
-h for help.
-b for male greeting.
-g for female greeting.
reader@ubuntu:~/scripts/chapter_15$ bash hey.sh
Hey there!
reader@ubuntu:~/scripts/chapter_15$ bash hey.sh -b
Hey boy!
reader@ubuntu:~/scripts/chapter_15$ bash hey.sh -g
Hey girl!
```

到目前为止，一切都很顺利！如果我们使用`-h`调用它，将看到打印的多行帮助信息。默认情况下，每个`echo`都以换行符结束，因此我们的五个`echo`将打印在五行上。我们可以使用单个`echo`和`\n`字符，但这样更易读。

如果我们在没有标志的情况下运行脚本，将看到通用的问候语。使用`-b`或`-g`运行它将给出特定性别的问候语。是不是很容易？

实际上是这样的！但是，情况即将变得更加复杂。正如我们之前解释过的，用户往往是相当不可预测的，可能会使用太多的标志，或者多次使用相同的标志。

让我们看看我们的脚本对此做出了怎样的反应：

```
reader@ubuntu:~/scripts/chapter_15$ bash hey.sh -h -b
Usage: hey.sh [flags]
Flags:
-h for help.
-b for male greeting.
-g for female greeting.
reader@ubuntu:~/scripts/chapter_15$ bash hey.sh -b -h
Usage: hey.sh [flags]
Flags:
-h for help.
-b for male greeting.
-g for female greeting.
reader@ubuntu:~/scripts/chapter_15$ bash hey.sh -b -h -g
Usage: hey.sh [flags]
Flags:
-h for help.
-b for male greeting.
-g for female greeting.
```

因此，只要指定了多少个标志，只要脚本遇到`-h`标志，它就会打印帮助消息并退出（由于`exit 0`）。为了您的理解，在调试模式下使用`bash -x`运行前面的命令，以查看它们实际上是不同的，即使用户看不到这一点（提示：检查`gender=boy`和`gender=girl`的赋值）。

这带我们来一个重要的观点：*标志是按用户提供的顺序解析的！*为了进一步说明这一点，让我们看另一个用户搞乱标志的例子：

```
reader@ubuntu:~/scripts/chapter_15$ bash hey.sh -g -b
Hey boy!
reader@ubuntu:~/scripts/chapter_15$ bash hey.sh -b -g
Hey girl!
```

当用户同时提供`-b`和`-g`标志时，系统会执行性别的两个变量赋值。然而，似乎最终的标志才是赢家，尽管我们刚刚说过标志是按顺序解析的！为什么会这样呢？

一如既往，一个不错的`bash -x`让我们对这种情况有了一个很好的了解：

```
reader@ubuntu:~/scripts/chapter_15$ bash -x hey.sh -b -g
+ optstring=:bgh
+ getopts :bgh options
+ case ${options} in
+ gender=boy
+ getopts :bgh options
+ case ${options} in
+ gender=girl
+ getopts :bgh options
+ [[ -n girl ]]
+ echo 'Hey girl!'
Hey girl!
```

最初，`gender`变量被赋予`boy`的值。然而，当解析下一个标志时，变量的值被*覆盖*为一个新值，`girl`。由于`-g`标志是最后一个，`gender`变量最终变成`girl`，因此打印出来的就是这个值。

正如您将在本章的下一部分中看到的，可以向标志提供参数。不过，对于没有参数的标志，有一个非常酷的功能，许多命令都在使用：标志链接。听起来可能很复杂，但实际上非常简单：如果有多个标志，可以将它们全部放在一个破折号后面。

对于我们的脚本，情况是这样的：

```
reader@ubuntu:~/scripts/chapter_15$ bash -x hey.sh -bgh
+ optstring=:bgh
+ getopts :bgh options
+ case ${options} in
+ gender=boy
+ getopts :bgh options
+ case ${options} in
+ gender=girl
+ getopts :bgh options
+ case ${options} in
+ print_help
<SNIPPED>
```

我们将所有标志都指定为一组：而不是`-b -g -h`，我们使用了`-bgh`。正如我们之前得出的结论，标志是按顺序处理的，这在我们连接的例子中仍然是这样（正如调试指令清楚地显示的那样）。这与`ls -al`并没有太大的不同。再次强调，这仅在标志没有参数时才有效。

# 带参数的标志

在`optstring`中，冒号除了关闭冗长的错误日志记录之外还有另一个意义：当放在一个字母后面时，它向`getopts`发出信号，表示期望一个*选项参数*。

如果我们回顾一下我们的第一个例子，`optstring`只是`:v`。如果我们希望`-v`标志接受一个参数，我们会在`v`后面放一个冒号，这将导致以下`optstring`：`:v:`。然后我们可以使用一个我们之前见过的特殊变量`OPTARG`来获取那个***选**项 **参**数*。

我们将对我们的`single-flag.sh`脚本进行修改，以向您展示它是如何工作的：

```
reader@ubuntu:~/scripts/chapter_15$ vim single-flag.sh 
reader@ubuntu:~/scripts/chapter_15$ cat single-flag.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
# Date: 2018-12-14
# Description: Shows the basic getopts syntax.
# Usage: ./single-flag.sh [flags]
#####################################

# Parse the flags in a while loop.
# After the last flag, getopts returns false which ends the loop.
optstring=":v:"
while getopts ${optstring} options; do
  case ${options} in
    v)
      echo "-v was found!"
      echo "-v option argument is: ${OPTARG}."
      ;;
    ?)
      echo "Invalid option: -${OPTARG}."
      exit 1
      ;; 
  esac
done
```

已更改的行已经为您突出显示。通过在`optstring`中添加一个冒号，并在`v)`块中使用`OPTARG`变量，我们现在看到了运行脚本时的以下行为：

```
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh 
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -v Hello
-v was found!
-v option argument is: Hello.
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -vHello
-v was found!
-v option argument is: Hello.
```

正如您所看到的，只要我们提供标志和标志参数，我们的脚本就可以正常工作。我们甚至不需要在标志和标志参数之间加上空格；由于`getopts`知道期望一个参数，它可以处理空格或无空格。我们始终建议在任何情况下都包括空格，以确保可读性，但从技术上讲并不需要。

这也证明了为什么我们需要一个单独的`optstring`：`case`语句是一样的，但是`getopts`现在期望一个参数，如果创建者省略了`optstring`，我们就无法做到这一点。

就像所有看起来太好以至于不真实的事情一样，这就是其中之一。如果用户对你的脚本友好，它可以正常工作，但如果他/她不友好，可能会发生以下情况：

```
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -v
Invalid option: -v.
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -v ''
-v was found!
-v option argument is: 
```

现在我们已经告诉`getopts`期望`-v`标志的参数，如果没有参数，它实际上将无法正确识别该标志。但是，空参数，如第二个脚本调用中的`''`，是可以的。 （从技术上讲是可以的，因为没有用户会这样做。）

幸运的是，有一个解决方案——`:)`块，如下所示：

```
reader@ubuntu:~/scripts/chapter_15$ vim single-flag.sh 
reader@ubuntu:~/scripts/chapter_15$ cat single-flag.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.2.0
# Date: 2018-12-14
# Description: Shows the basic getopts syntax.
# Usage: ./single-flag.sh [flags]
#####################################

# Parse the flags in a while loop.
# After the last flag, getopts returns false which ends the loop.
optstring=":v:"
while getopts ${optstring} options; do
  case ${options} in
    v)
      echo "-v was found!"
      echo "-v option argument is: ${OPTARG}."
      ;;
 :)
 echo "-${OPTARG} requires an argument."
 exit 1
 ;;
    ?)
      echo "Invalid option: -${OPTARG}."
      exit 1
      ;; 
  esac
done
```

可能有点令人困惑，错误的标志和缺少的选项参数都解析为`OPTARG`。不要把这种情况弄得比必要的更复杂，这一切取决于`case`语句块在那一刻包含`?)`还是`:)`。对于`?)`块，所有未被识别的内容（整个标志）都被视为选项参数，而`:)`块只有在`optstring`包含带参数选项的正确指令时才触发。

现在一切都应该按预期工作：

```
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -v
-v requires an argument.
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -v Hi
-v was found!
-v option argument is: Hi.
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -x Hi
Invalid option: -x.
reader@ubuntu:~/scripts/chapter_15$ bash single-flag.sh -x -v Hi
Invalid option: -x.
```

再次，由于标志的顺序处理，由于`?)`块中的`exit 1`，最终调用永远不会到达`-v`标志。但是，所有其他情况现在都得到了正确解决。不错！

`getopts`实际处理涉及多次传递和使用`shift`。这对于本章来说有点太技术性了，但对于你们中间感兴趣的人来说，*进一步阅读*部分包括了这个机制的*非常*深入的解释，你可以在空闲时阅读。

# 将标志与位置参数结合使用

可以将位置参数（在本章之前我们一直使用的方式）与选项和选项参数结合使用。在这种情况下，有一些事情需要考虑：

+   默认情况下，Bash 将识别标志（如`-f`）作为位置参数

+   就像标志和标志参数有一个顺序一样，标志和位置参数也有一个顺序

处理`getopts`和位置参数时，*标志和标志选项应始终在位置参数之前提供！*这是因为我们希望在到达位置参数之前解析和处理所有标志和标志参数。这对于脚本和命令行工具来说是一个相当典型的情况，但这仍然是我们必须考虑的事情。

前面的所有观点最好通过一个例子来说明，我们将创建一个简单的脚本，作为常见文件操作的包装器。有了这个脚本`file-tool.sh`，我们将能够做以下事情：

+   列出文件（默认行为）

+   删除文件（使用`-d`选项）

+   清空文件（使用`-e`选项）

+   重命名文件（使用`-m`选项，其中包括另一个文件名）

+   调用帮助函数（使用`-h`）

看一下脚本：

```
reader@ubuntu:~/scripts/chapter_15$ vim file-tool.sh 
reader@ubuntu:~/scripts/chapter_15$ cat file-tool.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-12-14
# Description: A tool which allows us to manipulate files.
# Usage: ./file-tool.sh [flags] <file-name>
#####################################

print_help() {
  echo "Usage: $0 [flags] <file-name>"
  echo "Flags:"
  echo "No flags for file listing."
  echo "-d to delete the file."
  echo "-e to empty the file."
  echo "-m <new-file-name> to rename the file."
  echo "-h for help."
}

command="ls -l" # Default command, can be overridden.

optstring=":dem:h" # The m option contains an option argument.
while getopts ${optstring} options; do
  case ${options} in
    d)
      command="rm -f";;
    e)
      command="cp /dev/null";;
    m)
      new_filename=${OPTARG}; command="mv";;
    h)
      print_help; exit 0;;
    :)
      echo "-${OPTARG} requires an argument."; exit 1;;
    ?)
      echo "Invalid option: -${OPTARG}." exit 1;; 
  esac
done

# Remove the parsed flags from the arguments array with shift.
shift $(( ${OPTIND} - 1 )) # -1 so the file-name is not shifted away.

filename=$1

# Make sure the user supplied a writable file to manipulate.
if [[ $# -ne 1 || ! -w ${filename} ]]; then
  echo "Supply a writable file to manipulate! Exiting script."
  exit 1 
fi

# Everything should be fine, execute the operation.
if [[ -n ${new_filename} ]]; then # Only set for -m.
  ${command} ${filename} $(dirname ${filename})/${new_filename}
else # Everything besides -m.
  ${command} ${filename}
fi
```

这是一个大的例子，不是吗？我们通过将多行压缩成单行（在`case`语句中）稍微缩短了一点，但它仍然不是一个短脚本。虽然一开始可能看起来令人生畏，但我们相信通过你到目前为止的接触和脚本中的注释，这对你来说应该是可以理解的。如果现在还不完全理解，不要担心——我们现在将解释所有新的有趣的行。

我们跳过了标题，`print_help()`函数和`ls -l`的默认命令。第一个有趣的部分将是`optstring`，它现在包含有和没有选项参数的选项：

```
optstring=":dem:h" # The m option contains an option argument.
```

当我们到达`m)`块时，我们将选项参数保存在`new_filename`变量中以供以后使用。

当我们完成`getopts`的`case`语句后，我们遇到了一个我们之前简要见过的命令：`shift`。这个命令允许我们移动我们的位置参数：如果我们执行`shift 2`，参数`$4`变成了`$2`，参数`$3`变成了`$1`，旧的`$1`和`$2`被移除了。

处理标志后面的位置参数时，所有标志和标志参数也被视为位置参数。在这种情况下，如果我们将脚本称为`file-tool.sh -m newfile /tmp/oldfile`，Bash 将解释如下：

+   `$1`：被解释为`-m`

+   `$2`：被解释为一个新文件

+   `$3`：被解释为`/tmp/oldfile`

幸运的是，`getopts`将它处理过的选项（和选项参数）保存在一个变量中：`$OPTIND`（来自***opt**ions **ind**ex*）。更准确地说，在解析了一个选项之后，它将`$OPTIND`设置为下一个可能的选项或选项参数：它从 1 开始，在找到传递给脚本的第一个非选项参数时结束。

在我们的示例中，一旦`getopts`到达我们的位置参数`/tmp/oldfile`，`$OPTIND`变量将为`3`。由于我们只需要将该点之前的所有内容`shift`掉，我们从`$OPTIND`中减去 1，如下所示：

```
shift $(( ${OPTIND} - 1 )) # -1 so the file-name is not shifted away.
```

记住，`$(( ... ))`是算术的简写；得到的数字用于`shift`命令。脚本的其余部分非常简单：我们将进行一些检查，以确保我们只剩下一个位置参数（我们想要操作的文件的文件名），以及我们是否对该文件具有写权限。

接下来，根据我们选择的操作，我们将为`mv`执行一个复杂的操作，或者为其他所有操作执行一个简单的操作。对于重命名命令，我们将使用一些命令替换来确定原始文件名的目录名称，然后我们将在重命名中重用它。

如果我们像应该做的那样进行了测试，脚本应该符合我们设定的所有要求。我们鼓励你尝试一下。

更好的是，看看你是否能想出一个我们没有考虑到的情况，破坏了脚本的功能。如果你找到了什么（剧透警告：我们知道有一些缺点！），试着自己修复它们。

正如你可能开始意识到的那样，我们正在进入一个非常难以为每个用户输入加固脚本的领域。例如，在最后一个例子中，如果我们提供了`-m`选项但省略了内容，我们提供的文件名将被视为选项参数。在这种情况下，我们的脚本将`shift`掉文件名并抱怨它没有。虽然这个脚本应该用于教育目的，但我们不会相信它用于我们的工作场所脚本。最好不要将`getopts`与位置参数混合使用，因为这样可以避免我们在这里面对的许多复杂性。只需让用户提供文件名作为另一个选项参数（`-f`，任何人？），你会更加快乐！

# 总结

本章以回顾 Bash 中如何使用位置参数开始。我们继续向您展示了到目前为止我们介绍的大多数命令行工具（以及我们没有介绍的那些）如何使用标志，通常作为脚本功能的*修饰符*，而位置参数则用于指示命令的*目标*。

然后，我们介绍了一种让读者在自己的脚本中结合选项和选项参数的方法：使用`getopts` shell 内置。我们从讨论传统程序`getopt`和较新的内置`getopts`之间的区别开始，然后我们在本章的其余部分重点讨论了`getopts`。

由于`getopts`只允许我们使用短选项（而`getopt`和其他一些命令行工具也使用长选项，用双破折号表示），我们向您展示了由于识别常见的短选项（如`-h`，`-v`等）而不是问题。

我们用几个例子正确介绍了`getopts`的语法。我们展示了如何使用带有和不带有标志参数的标志，以及我们如何需要一个`optstring`来向`getopts`发出信号，表明哪些选项有参数（以及期望哪些选项）。

我们通过聪明地使用`shift`命令来处理选项和选项参数与位置参数的组合，结束了这一章节。

本章介绍了以下命令：`getopts`和`shift`。

# 问题

1.  为什么标志经常被用作修饰符，而位置参数被用作目标？

1.  为什么我们在`while`循环中运行`getopts`？

1.  为什么我们在`case`语句中需要`?)`？

1.  为什么我们（有时）在`case`语句中需要`:)`？

1.  如果我们无论如何都要解析所有选项，为什么还需要一个单独的`optstring`？

1.  为什么我们在使用`shift`时需要从`OPTIND`变量中减去 1？

1.  将选项与位置参数混合使用是个好主意吗？

# 进一步阅读

请参考以下链接，了解本章主题的更多信息：

+   Bash-hackers 对`getopts`的解释：[`wiki.bash-hackers.org/howto/getopts_tutorial`](http://wiki.bash-hackers.org/howto/getopts_tutorial)

+   深入了解`getopts`：[`www.computerhope.com/unix/bash/getopts.htm`](https://www.computerhope.com/unix/bash/getopts.htm)
