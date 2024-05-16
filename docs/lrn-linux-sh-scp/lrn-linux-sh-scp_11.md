# 第十一章：条件测试和脚本循环

本章将以对`if-then-else`的小结开始，然后介绍`if-then-else`条件的高级用法。我们将介绍`while`和`for`的脚本循环，并展示如何使用`exit`，`break`和`continue`来控制这些循环。

本章将介绍以下命令：`elif`，`help`，`while`，`sleep`，`for`，`basename`，`break`和`continue`。

本章将涵盖以下主题：

+   高级`if-then-else`

+   `while` 循环

+   `for` 循环

+   `loop` 控制

# 技术要求

本章的所有脚本都可以在 GitHub 上找到：[`github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter11`](https://github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter11)。所有其他工具仍然有效，无论是在您的主机上还是在您的 Ubuntu 虚拟机上。对于 break-x.sh，for-globbing.sh，square-number.sh，while-interactive.sh 脚本，只能在网上找到最终版本。在执行脚本之前，请务必验证头部中的脚本版本。

# 高级 if-then-else

本章致力于条件测试和脚本循环的所有内容，这两个概念经常交织在一起。我们已经在第九章中看到了`if-then-else`循环，*错误检查和处理*，它侧重于错误检查和处理。在继续介绍高级概念之前，我们将对我们描述的关于`if-then-else`的事情进行小结。

# 对 if-then-else 的小结

`If-then-else` 逻辑几乎完全符合其名称的含义：**如果** *某事是这样的*，**那么** *做某事* 或 **否则** *做其他事情*。在实践中，这可能是**如果** *磁盘已满*，**那么** *删除一些文件* 或 **否则** *报告磁盘空间看起来很好*。在脚本中，这可能看起来像这样：

```
reader@ubuntu:~/scripts/chapter_09$ cat if-then-else-proper.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-30
# Description: Use the if-then-else construct, now properly.
# Usage: ./if-then-else-proper.sh file-name
#####################################

file_name=$1

# Check if the file exists.
if [[ -f ${file_name} ]]; then 
  cat ${file_name} # Print the file content.
else
  echo "File does not exist, stopping the script!"
  exit 1
fi
```

如果文件存在，我们打印内容。否则（也就是说，如果文件不存在），我们以错误消息的形式给用户反馈，然后以`1`的退出状态退出脚本。请记住，任何不为 0 的退出代码都表示*脚本失败*。

# 在测试中使用正则表达式

在介绍了`if-then-else`之后的一章中，我们学到了关于正则表达式的一切。然而，那一章大部分是理论性的，只包含了一个脚本！现在，正如你可能意识到的那样，正则表达式主要是支持构造，应该与其他脚本工具一起使用。在我们描述的测试情况下，我们可以在`[[...]]`块中同时使用 globbing 和正则表达式！让我们更深入地看一下这一点，如下所示：

```
reader@ubuntu:~/scripts/chapter_11$ vim square-number.sh 
reader@ubuntu:~/scripts/chapter_11$ cat square-number.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-26
# Description: Return the square of the input number.
# Usage: ./square-number.sh <number>
#####################################

INPUT_NUMBER=$1

# Check the number of arguments received.
if [[ $# -ne 1 ]]; then
 echo "Incorrect usage, wrong number of arguments."
 echo "Usage: $0 <number>"
 exit 1
fi

# Check to see if the input is a number.
if [[ ! ${INPUT_NUMBER} =~ [[:digit:]] ]]; then 
 echo "Incorrect usage, wrong type of argument."
 echo "Usage: $0 <number>"
 exit 1
fi

# Multiple the input number with itself and return this to the user.
echo $((${INPUT_NUMBER} * ${INPUT_NUMBER}))
```

我们首先检查用户是否提供了正确数量的参数（这是我们应该始终做的）。接下来，我们在测试`[[..]]`块中使用`=~`运算符。这允许我们**使用正则表达式进行评估**。在这种情况下，它简单地允许我们验证用户输入是否为数字，而不是其他任何东西。

现在，如果我们调用这个脚本，我们会看到以下内容：

```
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh
Incorrect usage, wrong number of arguments.
Usage: square-number.sh <number>
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh 3 2
Incorrect usage, wrong number of arguments.
Usage: square-number.sh <number>
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh a
Incorrect usage, wrong type of argument.
Usage: square-number.sh <number>
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh 3
9
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh 11
121
```

我们可以看到我们的两个输入检查都有效。如果我们调用这个脚本而不是只有一个参数(`$# -ne 1`)，它会失败。这对于`0`和`2`个参数都是正确的。接下来，如果我们用一个字母而不是一个数字来调用脚本，我们会到达第二个检查和随之而来的错误消息：`错误的参数类型`。最后，为了证明脚本确实做到了我们想要的，我们将尝试使用单个数字：`3`和`11`。`9`和`121`的返回值是这些数字的平方，所以看起来我们实现了我们的目标！

然而，并不是一切都如表面所示。这是使用正则表达式时的一个常见陷阱，如下面的代码所示：

```
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh a3
0
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh 3a
square-number.sh: line 28: 3a: value too great for base (error token is "3a")
```

这是怎么发生的？我们检查了用户输入是否是一个数字，不是吗？实际上，与你可能认为的相反，我们实际上检查了用户输入是否“与数字匹配”。简单来说，如果输入包含一个数字，检查就会成功。我们真正想要检查的是输入是否是一个数字“从头到尾”。也许这听起来很熟悉，但它绝对有锚定行的味道！以下代码应用了这一点：

```
reader@ubuntu:~/scripts/chapter_11$ vim square-number.sh
reader@ubuntu:~/scripts/chapter_11$ head -5 square-number.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
reader@ubuntu:~/scripts/chapter_11$ grep 'digit' square-number.sh 
if [[ ! ${INPUT_NUMBER} =~ ^[[:digit:]]$ ]]; then
```

我们做了两个改变：我们匹配的搜索模式不再只是`[[:digit:]]`，而是`^[[:digit:]]$`，并且我们更新了版本号（直到现在我们还没有做太多）。因为我们现在将数字锚定到行的开头和结尾，我们不能再在随机位置插入字母。用错误的输入运行脚本来验证这一点：

```
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh a3
Incorrect usage, wrong type of argument.
Usage: square-number-improved.sh <number>
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh 3a
Incorrect usage, wrong type of argument.
Usage: square-number-improved.sh <number>
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh 3a3
Incorrect usage, wrong type of argument.
Usage: square-number-improved.sh <number>
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh 9
81
```

我很想告诉你，我们现在完全安全了。但是，不幸的是，就像正则表达式经常出现的那样，事情并不那么简单。脚本现在对单个数字（0-9）运行得很好，但是如果你尝试使用双位数，它会出现“错误的参数类型”（试一下！）。我们需要做最后的调整来确保它完全符合我们的要求：我们需要确保数字也接受多个连续的数字。正则表达式中的“一个或多个”构造是+号，我们可以将其附加到`[[:digit:]]`上：

```
reader@ubuntu:~/scripts/chapter_11$ vim square-number.sh 
reader@ubuntu:~/scripts/chapter_11$ head -5 square-number.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.2.0
reader@ubuntu:~/scripts/chapter_11$ grep 'digit' square-number.sh 
if [[ ! ${INPUT_NUMBER} =~ ^[[:digit:]]+$ ]]; then 
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh 15
225
reader@ubuntu:~/scripts/chapter_11$ bash square-number.sh 1x5
Incorrect usage, wrong type of argument.
Usage: square-number-improved.sh <number>
```

我们改变了模式，提高了版本号，并用不同的输入运行了脚本。最终的模式`^[[:digit:]]+$`可以解读为“从行首到行尾的一个或多个数字”，在这种情况下意味着“一个数字，没有其他东西”！

这里的教训是你确实需要彻底测试你的正则表达式。正如你现在所知道的，搜索模式是贪婪的，一旦有一点匹配，它就认为结果是成功的。就像前面的例子中所看到的那样，这并不够具体。实现（和学习！）的唯一方法是尝试破坏你自己的脚本。尝试错误的输入，奇怪的输入，非常具体的输入等等。除非你尝试很多次，否则你不能确定它会*可能*工作。

你可以在测试语法中使用所有正则表达式搜索模式。我们不会详细介绍其他例子，但应该考虑的有：

+   变量应该以`/`开头（用于完全限定的路径）

+   变量不能包含空格（使用`[[:blank:]]`搜索模式）

+   变量应该只包含小写字母（可以通过`^[[:lower:]]+$`模式实现）

+   变量应该包含一个带有扩展名的文件名（可以匹配`[[:alnum:]]\.[[:alpha:]]`）

# elif 条件

在我们到目前为止看到的情况中，只需要检查一个*if* *条件*。但是正如你所期望的那样，有时候有多个你想要检查的事情，每个事情都有自己的后续动作（*then* *block*）。你可以通过使用两个完整的`if-then-else`语句来解决这个问题，但至少你会有一个重复的*else* *block*。更糟糕的是，如果你有三个或更多的条件要检查，你将会有越来越多的重复代码！幸运的是，我们可以通过使用`elif`命令来解决这个问题，它是`if-then-else`逻辑的一部分。你可能已经猜到，`elif`是`else-if`的缩写。它允许我们做如下的事情：

如果条件 1，那么执行事情 1，否则如果条件 2，那么执行事情 2，否则执行最终的事情

你可以在初始的`if`命令之后链接尽可能多的`elif`命令，但有一件重要的事情需要考虑：一旦任何条件为真，只有该`then`语句会被执行；其他所有语句都会被跳过。

如果你在考虑多个条件可以为真，并且它们的`then`语句应该被执行，你需要使用多个`if-then-else`块。让我们看一个简单的例子，首先检查用户给出的参数是否是一个文件。如果是，我们使用`cat`打印文件。如果不是这种情况，我们检查它是否是一个目录。如果是这种情况，我们使用`ls`列出目录。如果也不是这种情况，我们将打印一个错误消息并以非零退出状态退出。看看以下命令：

```
reader@ubuntu:~/scripts/chapter_11$ vim print-or-list.sh 
reader@ubuntu:~/scripts/chapter_11$ cat print-or-list.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-26
# Description: Prints or lists the given path, depending on type.
# Usage: ./print-or-list.sh <file or directory path>
#####################################

# Since we're dealing with paths, set current working directory.
cd $(dirname $0)

# Input validation.
if [[ $# -ne 1 ]]; then
  echo "Incorrect usage!"
  echo "Usage: $0 <file or directory path>"
  exit 1
fi

input_path=$1

if [[ -f ${input_path} ]]; then
  echo "File found, showing content:"
  cat ${input_path} || { echo "Cannot print file, exiting script!"; exit 1; }
elif [[ -d ${input_path} ]]; then
  echo "Directory found, listing:"
  ls -l ${input_path} || { echo "Cannot list directory, exiting script!"; exit 1; }
else
  echo "Path is neither a file nor a directory, exiting script."
  exit 1
fi
```

如你所见，当我们处理用户输入的文件时，我们需要额外的净化。我们确保在脚本中设置当前工作目录为`cd $(dirname $0)`，并且我们假设每个命令都可能失败，因此我们使用||构造来处理这些失败，就像第九章中所解释的那样，*错误检查和处理*。让我们尝试看看我们是否可以找到这个逻辑可能走的大部分路径：

```
reader@ubuntu:~/scripts/chapter_11$ bash print-or-list.sh 
Incorrect usage!
Usage: print-or-list.sh <file or directory path>
reader@ubuntu:~/scripts/chapter_11$ bash print-or-list.sh /etc/passwd
File found, showing content:
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
<SNIPPED>
reader@ubuntu:~/scripts/chapter_11$ bash print-or-list.sh /etc/shadow
File found, showing content:
cat: /etc/shadow: Permission denied
Cannot print file, exiting script!
reader@ubuntu:~/scripts/chapter_11$ bash print-or-list.sh /tmp/
Directory found, listing:
total 8
drwx------ 3 root root 4096 Oct 26 08:26 systemd-private-4f8c34d02849461cb20d3bfdaa984c85...
drwx------ 3 root root 4096 Oct 26 08:26 systemd-private-4f8c34d02849461cb20d3bfdaa984c85...
reader@ubuntu:~/scripts/chapter_11$ bash print-or-list.sh /root/
Directory found, listing:
ls: cannot open directory '/root/': Permission denied
Cannot list directory, exiting script!
reader@ubuntu:~/scripts/chapter_11$ bash print-or-list.sh /dev/zero
Path is neither a file nor a directory, exiting script.
```

按顺序，我们已经看到了我们脚本的以下场景：

1.  **无参数**：`使用不正确`错误

1.  /etc/passwd 文件参数：文件内容已打印

1.  非可读文件/etc/shadow 上的文件参数：`无法打印文件`错误

1.  /tmp/上的目录参数：目录列表已打印

1.  非可列出目录/root/上的目录参数：`无法列出目录`错误

1.  特殊文件（块设备）参数/dev/zero：`路径既不是文件也不是目录`错误

这六种输入场景代表了我们的脚本可能采取的所有可能路径。虽然你可能认为对于（看似简单的）脚本的所有错误处理有点过分，但这些参数应该验证了为什么我们实际上需要所有这些错误处理。

虽然`elif`极大地增强了`if-then-else`语句的可能性，但太多的`if-elif-elif-elif-`.......`-then-else`将使你的脚本变得非常难以阅读。还有另一种构造（超出了本书的范围），叫做`case`。这处理许多不同的、独特的条件。在本章末尾的进一步阅读部分查看关于`case`的良好资源！

# 嵌套

另一个非常有趣的概念是嵌套。实质上，嵌套非常简单：就是在*外部*的`if-then-else`的`then`或`else`中放置另一个`if-then-else`语句。这使我们能够首先确定文件是否可读，然后确定文件的类型。通过使用嵌套的`if-then-else`语句，我们可以以不再需要||构造的方式重写先前的代码：

```
reader@ubuntu:~/scripts/chapter_11$ vim nested-print-or-list.sh 
reader@ubuntu:~/scripts/chapter_11$ cat nested-print-or-list.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-26
# Description: Prints or lists the given path, depending on type.
# Usage: ./nested-print-or-list.sh <file or directory path>
#####################################

# Since we're dealing with paths, set current working directory.
cd $(dirname $0)

# Input validation.
if [[ $# -ne 1 ]]; then
  echo "Incorrect usage!"
  echo "Usage: $0 <file or directory path>"
  exit 1
fi

input_path=$1

# First, check if we can read the file.
if [[ -r ${input_path} ]]; then
  # We can read the file, now we determine what type it is.
  if [[ -f ${input_path} ]]; then
    echo "File found, showing content:"
    cat ${input_path} 
  elif [[ -d ${input_path} ]]; then
    echo "Directory found, listing:"
    ls -l ${input_path} 
  else
    echo "Path is neither a file nor a directory, exiting script."
    exit 1
  fi
else
  # We cannot read the file, print an error.
  echo "Cannot read the file/directory, exiting script."
  exit 1
fi
```

尝试使用与前一个示例相同的输入运行上述脚本。在这种情况下，错误场景中的输出会更加友好，因为现在我们控制了这些（而不是默认输出`cat: /etc/shadow: Permission denied`，例如）。但从功能上讲，什么也没有改变！我们认为，这个使用嵌套的脚本比之前的例子更可读，因为我们现在自己处理错误场景，而不是依赖系统命令来为我们处理。

我们之前讨论过缩进，但在我们看来，像这样的脚本才是它真正发挥作用的地方。通过缩进内部的`if-then-else`语句，更清楚地表明第二个`else`属于外部的`if-then-else`语句。如果你使用多层缩进（因为理论上你可以嵌套多次），这确实有助于所有参与脚本编写的人遵循这个逻辑。

嵌套不仅仅适用于`if-then-else`。我们将在本章后面介绍的两个循环`for`和`while`也可以嵌套。而且，更实用的是，你可以将它们嵌套在其他所有循环中（从技术角度来看；当然，从逻辑角度来看也应该是有意义的！）。当我们解释`while`和`for`时，你会看到这样的例子。

# 获取帮助

到现在为止，你可能害怕自己永远记不住所有这些。虽然我们确信随着时间的推移，通过足够的练习，你肯定会记住，但我们理解当你经验不足时，这是很多东西要消化的。为了让这更容易些，除了`man`页面之外还有另一个有用的命令。你可能已经发现（并且在尝试时失败了），`man if`或`man [[`都不起作用。如果你用`type if`和`type [[`检查这些命令，你会发现它们实际上不是命令而是*shell 关键字*。对于大多数 shell 内置和 shell 关键字，你可以使用`help`命令打印一些关于它们的信息以及如何使用它们！使用`help`就像`help if`、`help [[`、`help while`等一样简单。对于`if-then-else`语句，只有`help if`有效：

```
reader@ubuntu:~/scripts/chapter_11$ help if
if: if COMMANDS; then COMMANDS; [ elif COMMANDS; then COMMANDS; ]... [ else COMMANDS; ] fi
    Execute commands based on conditional.

    The 'if COMMANDS' list is executed. If its exit status is zero,
     then the 'then COMMANDS' list is executed.  Otherwise, each 
     'elif COMMANDS' list is executed in turn, and if its 
     exit status is zero, the corresponding
    'then COMMANDS' list is executed and the if command completes.  Otherwise,
    the 'else COMMANDS' list is executed, if present. 
    The exit status of the entire construct is the 
     exit status of the last command executed, or zero
    if no condition tested true.

    Exit Status:
    Returns the status of the last command executed.
```

因此，总的来说，有三种方法可以让 Linux 为你打印一些有用的信息：

+   使用`man`命令的 man 页面

+   使用`help`命令获取帮助信息

+   命令本地帮助打印（通常作为`flag -h`、`--help`或`-help`）

根据命令的类型（二进制命令或 shell 内置/关键字），你将使用`man`、`help`或`--help`标志。记住，通过检查你正在处理的命令的类型（这样你就可以更加有根据地猜测首先尝试哪种帮助方法），使用`type -a <command>`。

# `while`循环

现在我们已经搞定了`if-then-else`的复习和高级用法，是时候讨论第一个脚本循环了：`while`。看一下下面的定义，在`if-then-else`之后应该看起来很熟悉：

当条件为真时执行的事情

`if`和`while`之间最大的区别是，`while`会执行动作多次，只要指定的条件仍然为真。因为通常不需要无休止地循环，动作将定期改变与条件相关的某些东西。这基本上意味着*do*中的动作最终会导致`while`条件变为 false 而不是 true。让我们看一个简单的例子：

```
reader@ubuntu:~/scripts/chapter_11$ vim while-simple.sh 
reader@ubuntu:~/scripts/chapter_11$ cat while-simple.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-27
# Description: Example of a while loop.
# Usage: ./while-simple.sh 
#####################################

# Infinite while loop.
while true; do
  echo "Hello!"
  sleep 1 # Wait for 1 second.
done
```

这个例子是`while`的最基本形式：一个无休止的循环（因为条件只是`true`），它打印一条消息，然后休眠一秒。这个新命令`sleep`经常在循环（`while`和`for`）中使用，等待指定的时间。在这种情况下，我们运行`sleep 1`，它在返回循环顶部并再次打印`Hello!`之前等待一秒。一定要尝试一下，并注意它永远不会停止（*Ctrl* + *C*会杀死进程，因为它是交互式的）。

现在我们将创建一个在特定时间结束的脚本。为此，我们将在`while`循环之外定义一个变量，我们将使用它作为计数器。这个计数器将在每次`while`循环运行时递增，直到达到条件中定义的阈值。看一下：

```
reader@ubuntu:~/scripts/chapter_11$ vim while-counter.sh 
reader@ubuntu:~/scripts/chapter_11$ cat while-counter.sh
cat while-counter.sh
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-27
# Description: Example of a while loop with a counter.
# Usage: ./while-counter.sh 
#####################################

# Define the counter outside of the loop so we don't reset it for 
# every run in the loop.
counter=0

# This loop runs 10 times.
while [[ ${counter} -lt 10 ]]; do
  counter=$((counter+1)) # Increment the counter by 1.
  echo "Hello! This is loop number ${counter}."
  sleep 1 
done

# After the while-loop finishes, print a goodbye message.
echo "All done, thanks for tuning in!"
```

由于我们添加了注释，这个脚本应该是不言自明的。`counter`被添加到`while`循环之外，否则每次循环运行都会以`counter=0`开始，这会重置进度。只要计数器小于 10，我们就会继续运行循环。经过 10 次运行后，情况就不再是这样了，而是继续执行脚本中的下一条指令，即打印再见消息。继续运行这个脚本。编辑 sleep 后面的数字（提示：它也接受小于一秒的值），或者完全删除 sleep。

# until 循环

`while`有一个孪生兄弟：`until`。`until`循环与`while`做的事情完全相同，唯一的区别是：只有在条件为**false**时循环才会运行。一旦条件变为**true**，循环就不再运行。我们将对上一个脚本进行一些小修改，看看`until`是如何工作的：

```
reader@ubuntu:~/scripts/chapter_11$ cp while-counter.sh until-counter.sh
reader@ubuntu:~/scripts/chapter_11$ vim until-counter.sh 
reader@ubuntu:~/scripts/chapter_11$ cat until-counter.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-27
# Description: Example of an until loop with a counter.
# Usage: ./until-counter.sh 
#####################################

# Define the counter outside of the loop so we don't reset it for 
# every run in the loop.
counter=0

# This loop runs 10 times.
until [[ ${counter} -gt 9 ]]; do
  counter=$((counter+1)) # Increment the counter by 1.
  echo "Hello! This is loop number ${counter}."
  sleep 1
done

# After the while-loop finishes, print a goodbye message.
echo "All done, thanks for tuning in!"
```

如你所见，对这个脚本的更改非常小（但重要，尽管如此）。我们用`until`替换了`while`，用`-gt`替换了`-lt`，用`9`替换了`10`。现在，它读作`当计数器大于 9 时运行循环`，而不是`只要计数器小于 10 时运行循环`。因为我们使用了小于和大于，我们必须改变数字，否则我们将会遇到著名的*off-by-one*错误（在这种情况下，这意味着我们将循环 11 次，如果我们没有将`10`改为`9`；试试看！）。

实际上，`while`和`until`循环是完全相同的。你会经常使用`while`循环而不是`until`循环：因为你可以简单地否定条件，`while`循环总是有效的。然而，有时，`until`循环可能更合理。无论如何，使用最容易理解的那个！如果有疑问，只使用`while`几乎永远不会错，只要你得到了正确的条件。

# 创建一个交互式 while 循环

实际上，你不会经常使用`while`循环。在大多数情况下，`for`循环更好（正如我们将在本章后面看到的）。然而，有一种情况`while`循环非常适用：处理用户输入。如果你使用`while true`结构，并在其中嵌套 if-then-else 块，你可以不断地向用户询问输入，直到得到你要找的答案。下面的例子是一个简单的谜语，应该能澄清问题：

```
reader@ubuntu:~/scripts/chapter_11$ vim while-interactive.sh 
reader@ubuntu:~/scripts/chapter_11$ cat while-interactive.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-27
# Description: A simple riddle in a while loop.
# Usage: ./while-interactive.sh
#####################################

# Infinite loop, only exits on correct answer.
while true; do
  read -p "I have keys but no locks. I have a space but no room. You can enter, but can’t go outside. What am I? " answer
  if [[ ${answer} =~ [Kk]eyboard ]]; then # Use regular expression so 'a keyboard' or 'Keyboard' is also a valid answer.
    echo "Correct, congratulations!"
    exit 0 # Exit the script.
  else
    # Print an error message and go back into the loop.
    echo "Incorrect, please try again."
  fi
done

reader@ubuntu:~/scripts/chapter_11$ bash while-interactive.sh 
I have keys but no locks. I have a space but no room. You can enter, but can’t go outside. What am I? mouse
Incorrect, please try again.
I have keys but no locks. I have a space but no room. You can enter, but can’t go outside. What am I? screen
Incorrect, please try again.
I have keys but no locks. I have a space but no room. You can enter, but can’t go outside. What am I? keyboard
Correct, congratulations!
reader@ubuntu:~/scripts/chapter_11$
```

在这个脚本中，我们使用`read -p`来询问用户一个问题，并将回答存储在`answer`变量中。然后我们使用嵌套的 if-then-else 块来检查用户是否给出了正确的答案。我们使用一个简单的正则表达式 if 条件，`${answer} =~ [Kk]eyboard`，这给用户在大写字母和也许单词`a`前面有一点灵活性。对于每个不正确的答案，*else*语句打印一个错误，循环重新开始`read -p`。如果答案是正确的，*then*块被执行，以`exit 0`表示脚本的结束。只要没有给出正确的答案，循环将永远继续。

你可能会看到这个脚本有一个问题。如果我们想在`while`循环之后做任何事情，我们需要在不退出脚本的情况下*中断*它。我们将看到如何使用——等待它——`break`关键字来实现这一点！但首先，我们将看看`for`循环。

# for 循环

`for`循环可以被认为是 Bash 脚本中更强大的循环。在实践中，`for`和`while`是可以互换的，但`for`有更好的简写语法。这意味着在`for`中编写循环通常需要比等效的`while`循环少得多的代码。

`for`循环有两种不同的语法：C 风格的语法和`regular` Bash 语法。我们首先看一下 Bash 语法：

FOR value IN list-of-values DO thing-with-value DONE

`for`循环允许我们*迭代*一个事物列表。每次循环将使用列表中的不同项目，按顺序。这个非常简单的例子应该说明这种行为：

```
reader@ubuntu:~/scripts/chapter_11$ vim for-simple.sh
reader@ubuntu:~/scripts/chapter_11$ cat for-simple.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-27
# Description: Simple for syntax.
# Usage: ./for-simple.sh
#####################################

# Create a 'list'.
words="house dog telephone dog"

# Iterate over the list and process the values.
for word in ${words}; do
  echo "The word is: ${word}"
done

reader@ubuntu:~/scripts/chapter_11$ bash for-simple.sh 
The word is: house
The word is: dog
The word is: telephone
The word is: dog
```

如你所见，`for`接受一个列表（在这种情况下，是由空格分隔的字符串），对于它找到的每个值，它执行`echo`操作。我们添加了一些额外的文本，这样你就可以看到它实际上进入循环四次，而不仅仅是打印带有额外换行符的列表。这里要注意的主要事情是，在 echo 中我们使用`${word}`变量，我们将其定义为`for`定义中的第二个单词。这意味着对于`for`循环的每次运行，`${word}`变量的值是不同的（这非常符合使用变量的意图，具有*variable*内容！）。你可以给它取任何名字，但我们更喜欢给出语义逻辑的名称；因为我们称列表为*words*，列表中的一个项目将是一个*word*。

如果你想用`while`做同样的事情，事情会变得更加复杂。通过使用计数器和`cut`这样的命令（它允许你剪切字符串的不同部分），这是完全可能的，但由于`for`循环以这种简单的方式完成，为什么要麻烦呢？

我们可以使用的第二种与 for 一起使用的语法对于那些有其他脚本编程语言经验的人来说更加熟悉。这种 C 风格的语法使用一个计数器，直到某个点递增，与我们在看`while`时看到的示例类似。其语法如下：

```
FOR ((counter=0; counter<=10; counter++)); DO something DONE
```

看起来很相似对吧？看看这个示例脚本：

```
reader@ubuntu:~/scripts/chapter_11$ vim for-counter.sh 
reader@ubuntu:~/scripts/chapter_11$ cat for-counter.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-27
# Description: Example of a for loop in C-style syntax.
# Usage: ./for-counter.sh 
#####################################

# This loop runs 10 times.
for ((counter=1; counter<=10; counter++)); do
  echo "Hello! This is loop number ${counter}."
  sleep 1
done

# After the for-loop finishes, print a goodbye message.
echo "All done, thanks for tuning in!"

reader@ubuntu:~/scripts/chapter_11$ bash for-counter.sh 
Hello! This is loop number 1.
Hello! This is loop number 2.
Hello! This is loop number 3.
Hello! This is loop number 4.
Hello! This is loop number 5.
Hello! This is loop number 6.
Hello! This is loop number 7.
Hello! This is loop number 8.
Hello! This is loop number 9.
Hello! This is loop number 10.
All done, thanks for tuning in!
```

由于 off-by-one 错误的性质，我们必须使用稍微不同的数字。由于计数器在循环结束时递增，我们需要从 1 开始而不是从 0 开始（或者我们可以在 while 循环中做同样的事情）。在 C 风格的语法中，**<=**表示*小于或等于*，++表示*递增 1*。因此，我们有一个计数器，从 1 开始，一直持续到达 10，并且每次循环运行时递增 1。我们发现这个`for`循环比等效的 while 循环更可取；它需要更少的代码，在其他脚本/编程语言中更常见。

更好的是，还有一种方法可以遍历数字范围（就像我们之前对 1-10 做的那样），也可以使用 for 循环 Bash 语法。因为数字范围只是一个*数字列表*，所以我们可以使用几乎与我们在第一个示例中对*单词列表*进行迭代的相同语法。看看下面的代码：

```
reader@ubuntu:~/scripts/chapter_11$ vim for-number-list.sh
reader@ubuntu:~/scripts/chapter_11$ cat for-number-list.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-27
# Description: Example of a for loop with a number range.
# Usage: ./for-number-list.sh
#####################################

# This loop runs 10 times.
for counter in {1..10}; do
  echo "Hello! This is loop number ${counter}."
  sleep 1
done

# After the for-loop finishes, print a goodbye message.
echo "All done, thanks for tuning in!"

reader@ubuntu:~/scripts/chapter_11$ bash for-number-list.sh 
Hello! This is loop number 1.
Hello! This is loop number 2.
Hello! This is loop number 3.
Hello! This is loop number 4.
Hello! This is loop number 5.
Hello! This is loop number 6.
Hello! This is loop number 7.
Hello! This is loop number 8.
Hello! This is loop number 9.
Hello! This is loop number 10.
All done, thanks for tuning in!
```

因此，`<list>`中的`<variable>`语法适用于`{1..10}`的列表。这称为**大括号扩展**，并在 Bash 版本 4 中添加。大括号扩展的语法非常简单：

```
{<starting value>..<ending value>}
```

大括号扩展可以以许多方式使用，但打印数字或字符列表是最为人熟知的：

```
reader@ubuntu:~/scripts/chapter_11$ echo {1..5}
1 2 3 4 5
reader@ubuntu:~/scripts/chapter_11$ echo {a..f}
a b c d e f
```

大括号扩展`{1..5}`返回字符串`1 2 3 4 5`，这是一个以空格分隔的值列表，因此可以在 Bash 风格的`for`循环中使用！另外，`{a..f}`打印字符串`a b c d e f`。范围实际上是由 ASCII 十六进制代码确定的；这也允许我们做以下操作：

```
reader@ubuntu:~/scripts/chapter_11$ echo {A..z}
A B C D E F G H I J K L M N O P Q R S T U V W X Y Z [  ] ^ _ ` a b c d e f g h i j k l m n o p q r s t u v w x y z
```

你可能会觉得奇怪，因为你会看到一些特殊字符在中间打印，但这些字符是大写和小写拉丁字母字符之间的。请注意，这种语法与使用`${variable}`获取变量值非常相似（但这是参数扩展，而不是大括号扩展）。

大括号扩展还有另一个有趣的功能：它允许我们定义增量！简而言之，这允许我们告诉 Bash 每次递增时要跳过多少步。其语法如下：

```
{<starting value>..<ending value>..<increment>}
```

默认情况下，增量值为 1。如果这是期望的功能，我们可以省略增量值，就像我们之前看到的那样。但是，如果我们设置了它，我们将看到以下内容：

```
reader@ubuntu:~/scripts/chapter_11$ echo {1..100..10}
1 11 21 31 41 51 61 71 81 91
reader@ubuntu:~/scripts/chapter_11$ echo {0..100..10}
0 10 20 30 40 50 60 70 80 90 100
```

现在，增量是以 10 的步长进行的。正如你在前面的示例中看到的，`<ending value>`被认为是*包含的*。这意味着*低于或等于*的值将被打印，但其他值不会。在前面示例中的第一个大括号扩展中，`{1..100..10}`，下一个值将是 101；因为这不是低于或等于 100，该值不会被打印，扩展被终止。

最后，因为我们承诺了我们可以用`while`做的任何事情，我们也可以用`for`做，我们想通过展示如何使用`for`创建无限循环来结束本章的部分。这是选择`while`而不是`for`的最常见原因，因为`for`的语法有点奇怪：

```
eader@ubuntu:~/scripts/chapter_11$ vim for-infinite.sh 
reader@ubuntu:~/scripts/chapter_11$ cat for-infinite.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-27
# Description: Example of an infinite for loop.
# Usage: ./for-infinite.sh 
#####################################

# Infinite for loop.
for ((;;)); do
  echo "Hello!"
  sleep 1 # Wait for 1 second.
done

reader@ubuntu:~/scripts/chapter_11$ bash for-infinite.sh 
Hello!
Hello!
Hello!
^C
```

我们使用 C 风格的语法，但省略了计数器的初始化、比较和递增。因此，它的读法如下：

for ((<nothing>;<no-comparison>;<no-increment>)); do

这最终变成了`((;;));`，只有将它放在正常语法的上下文中才有意义，就像我们在前面的例子中所做的那样。我们也可以省略增量或与相同效果的比较，但那样会增加更多的代码。通常情况下，更短更好，因为它会更清晰。

尝试复制无限的`for`循环，但只是通过省略`for`子句中的一个值。如果你成功了，你将更接近理解为什么你现在让它无休止地运行。如果你需要一点点提示，也许你想要在循环中打印`counter`的值，这样你就可以看到发生了什么。当然，你也可以用`bash -x`来运行它！

# 通配符和 for 循环

现在，让我们看一些更实际的例子。在 Linux 上，你将会处理大部分事情都与文件有关（还记得为什么吗？）。想象一下，你有一堆日志文件放在服务器上，你想对它们执行一些操作。如果只是一个命令执行一个动作，你可以很可能使用通配符模式和命令（比如`grep -i 'error' *.log`）。然而，想象一种情况，你想收集包含某个短语的日志文件，或者只想要这些文件的行。在这种情况下，使用通配符模式结合`for`循环将允许我们对许多文件执行许多命令，我们可以动态地找到它们！让我们试试看。因为这个脚本将结合我们迄今为止所学的许多课程，我们将从简单开始，逐渐扩展它：

```
reader@ubuntu:~/scripts/chapter_11$ vim for-globbing.sh 
reader@ubuntu:~/scripts/chapter_11$ cat for-globbing.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-27
# Description: Combining globbing patterns in a for loop.
# Usage: ./for-globbing.sh 
#####################################

# Create a list of log files.   
for file in $(ls /var/log/*.log); do
  echo ${file}
done

reader@ubuntu:~/scripts/chapter_11$ bash for-globbing.sh 
/var/log/alternatives.log
/var/log/auth.log
/var/log/bootstrap.log
/var/log/cloud-init.log
/var/log/cloud-init-output.log
/var/log/dpkg.log
/var/log/kern.log
```

通过使用`$(ls /var/log/*.log)`构造，我们可以创建一个在`/var/log/`目录中找到的所有以`.log`结尾的文件的列表。如果你手动运行`ls /var/log/*.log`命令，你会注意到格式与我们在 Bash 风格的 for 语法中使用时所见到的其他格式相同：单词，以空格分隔。因此，我们现在可以操作我们找到的所有文件！让我们看看如果我们尝试在这些文件中进行 grep 会发生什么：

```
reader@ubuntu:~/scripts/chapter_11$ cat for-globbing.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
# Date: 2018-10-27
# Description: Combining globbing patterns in a for loop.
# Usage: ./for-globbing.sh 
#####################################

# Create a list of log files.   
for file in $(ls /var/log/*.log); do
  echo "File: ${file}"
  grep -i 'error' ${file}
done
```

自从我们改变了脚本的内容，我们已经将版本从`v1.0.0`提升到`v1.1.0`。如果你现在运行这个脚本，你会发现一些文件在 grep 上返回了正匹配，而其他一些没有：

```
reader@ubuntu:~/scripts/chapter_11$ bash for-globbing.sh 
File: /var/log/alternatives.log
File: /var/log/auth.log
File: /var/log/bootstrap.log
Selecting previously unselected package libgpg-error0:amd64.
Preparing to unpack .../libgpg-error0_1.27-6_amd64.deb ...
Unpacking libgpg-error0:amd64 (1.27-6) ...
Setting up libgpg-error0:amd64 (1.27-6) ...
File: /var/log/cloud-init.log
File: /var/log/cloud-init-output.log
File: /var/log/dpkg.log
2018-04-26 19:07:33 install libgpg-error0:amd64 <none> 1.27-6
2018-04-26 19:07:33 status half-installed libgpg-error0:amd64 1.27-6
2018-04-26 19:07:33 status unpacked libgpg-error0:amd64 1.27-6
<SNIPPED>
File: /var/log/kern.log
Jun 30 18:20:32 ubuntu kernel: [    0.652108] RAS: Correctable Errors collector initialized.
Jul  1 09:31:07 ubuntu kernel: [    0.656995] RAS: Correctable Errors collector initialized.
Jul  1 09:42:00 ubuntu kernel: [    0.680300] RAS: Correctable Errors collector initialized.
```

太好了，现在我们用一个复杂的 for 循环实现了与直接使用`grep`相同的事情！现在，让我们充分利用它，在我们确定它们包含单词`error`之后，对文件做些什么：

```
reader@ubuntu:~/scripts/chapter_11$ vim for-globbing.sh 
reader@ubuntu:~/scripts/chapter_11$ cat for-globbing.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.2.0
# Date: 2018-10-27
# Description: Combining globbing patterns in a for loop.
# Usage: ./for-globbing.sh 
#####################################

# Create a directory to store log files with errors.
ERROR_DIRECTORY='/tmp/error_logfiles/'
mkdir -p ${ERROR_DIRECTORY}

# Create a list of log files. 
for file in $(ls /var/log/*.log); do
 grep --quiet -i 'error' ${file}

 # Check the return code for grep; if it is 0, file contains errors.
 if [[ $? -eq 0 ]]; then
 echo "${file} contains error(s), copying it to archive."
 cp ${file} ${ERROR_DIRECTORY} # Archive the file to another directory.
 fi

done

reader@ubuntu:~/scripts/chapter_11$ bash for-globbing.sh 
/var/log/bootstrap.log contains error(s), copying it to archive.
/var/log/dpkg.log contains error(s), copying it to archive.
/var/log/kern.log contains error(s), copying it to archive.
```

下一个版本，`v1.2.0`，执行了一个安静的`grep`（没有输出，因为我们只想要在找到东西时得到退出状态为 0）。在`grep`之后，我们使用了一个嵌套的`if-then`来将文件复制到我们在脚本开头定义的存档目录中。当我们现在运行脚本时，我们可以看到在上一个版本的脚本中生成输出的相同文件，但现在它复制整个文件。此时，`for`循环证明了它的价值：我们现在对使用通配符模式找到的单个文件执行多个操作。让我们再进一步，从存档文件中删除所有不包含错误的行：

```
reader@ubuntu:~/scripts/chapter_11$ vim for-globbing.sh 
reader@ubuntu:~/scripts/chapter_11$ cat for-globbing.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.3.0
# Date: 2018-10-27
# Description: Combining globbing patterns in a for loop.
# Usage: ./for-globbing.sh 
#####################################

# Create a directory to store log files with errors.
ERROR_DIRECTORY='/tmp/error_logfiles/'
mkdir -p ${ERROR_DIRECTORY}

# Create a list of log files.   
for file in $(ls /var/log/*.log); do
  grep --quiet -i 'error' ${file}

  # Check the return code for grep; if it is 0, file contains errors.
  if [[ $? -eq 0 ]]; then
    echo "${file} contains error(s), copying it to archive ${ERROR_DIRECTORY}."
    cp ${file} ${ERROR_DIRECTORY} # Archive the file to another directory.

    # Create the new file location variable with the directory and basename of the file.
    file_new_location="${ERROR_DIRECTORY}$(basename ${file})"
    # In-place edit, only print lines matching 'error' or 'Error'.
    sed --quiet --in-place '/[Ee]rror/p' ${file_new_location} 
  fi

done
```

版本 v1.3.0！为了使它稍微可读一些，我们没有在`cp`和`mkdir`命令上包含错误检查。然而，由于这个脚本的性质（在`/tmp/`中创建一个子目录并将文件复制到那里），那里出现问题的机会非常小。我们添加了两个新的有趣的东西：一个名为`file_new_location`的新变量，带有新位置的文件名和`sed`，它确保只有错误行保留在存档文件中。

首先，让我们考虑`file_new_location=${ERROR_DIRECTORY}$(basename ${file})`。我们正在将两个字符串拼接在一起：首先是存档目录，然后是*处理文件的基本名称*。`basename`命令会剥离文件的完全限定路径，只保留路径末端的文件名。如果我们要查看 Bash 将采取的步骤来解析这个新变量，它可能看起来是这样的：

+   `file_new_location=${ERROR_DIRECTORY}$(basename ${file})`

`-> 解析${file}`

+   `file_new_location=${ERROR_DIRECTORY}$(basename /var/log/bootstrap.log)`

`-> 解析$(basename /var/log/bootstrap.log)`

+   `file_new_location=${ERROR_DIRECTORY}bootstrap.log`

`-> 解析${ERROR_DIRECTORY}`

+   `file_new_location=/tmp/error_logfiles/bootstrap.log`

`-> 完成，变量的最终值！`

完成这些工作后，我们现在可以在这个新文件上运行`sed`。`sed --quiet --in-place '/[Ee]rror/p' ${file_new_location}`命令简单地用与正则表达式搜索模式`[Ee]rror`匹配的所有行替换文件的内容，这几乎就是我们最初使用 grep 搜索的内容。请记住，我们需要`--quiet`，因为默认情况下，`sed`会打印所有行。如果我们省略这一点，我们最终会得到文件中的所有行，但所有的错误文件都会被复制：一次来自`sed`的非静音输出，一次来自搜索模式匹配。然而，通过激活--quiet，`sed`只打印匹配的行并将其写入文件。让我们实际操作一下，验证结果：

```
reader@ubuntu:~/scripts/chapter_11$ bash for-globbing.sh 
/var/log/bootstrap.log contains error(s), copying it to archive /tmp/error_logfiles/.
/var/log/dpkg.log contains error(s), copying it to archive /tmp/error_logfiles/.
/var/log/kern.log contains error(s), copying it to archive /tmp/error_logfiles/.
reader@ubuntu:~/scripts/chapter_11$ ls /tmp/error_logfiles/
bootstrap.log  dpkg.log  kern.log
reader@ubuntu:~/scripts/chapter_11$ head -3 /tmp/error_logfiles/*
==> /tmp/error_logfiles/bootstrap.log <==
Selecting previously unselected package libgpg-error0:amd64.
Preparing to unpack .../libgpg-error0_1.27-6_amd64.deb ...
Unpacking libgpg-error0:amd64 (1.27-6) ...

==> /tmp/error_logfiles/dpkg.log <==
2018-04-26 19:07:33 install libgpg-error0:amd64 <none> 1.27-6
2018-04-26 19:07:33 status half-installed libgpg-error0:amd64 1.27-6
2018-04-26 19:07:33 status unpacked libgpg-error0:amd64 1.27-6

==> /tmp/error_logfiles/kern.log <==
Jun 30 18:20:32 ubuntu kernel: [    0.652108] RAS: Correctable Errors collector initialized.
Jul  1 09:31:07 ubuntu kernel: [    0.656995] RAS: Correctable Errors collector initialized.
Jul  1 09:42:00 ubuntu kernel: [    0.680300] RAS: Correctable Errors collector initialized.
```

正如你所看到的，每个文件顶部的三行都包含`error`或`Error`字符串。实际上，所有这些文件中的所有行都包含这两个字符串中的一个；请务必在您自己的系统上验证这一点，因为内容肯定会有所不同。

现在我们完成了这个示例，如果你愿意接受挑战，我们为读者提供了一些挑战：

+   使这个脚本接受输入。这可以是存档目录、路径通配符、搜索模式，甚至是这三者的组合！

+   通过为*可能*失败的命令添加异常处理，使这个脚本更加健壮。

+   通过使用`sed '/xxx/d'`语法来颠倒这个脚本的功能（提示：你可能需要重定向来实现这一点）。

虽然这个示例应该说明了很多东西，但我们意识到仅仅搜索`error`这个词实际上并不只返回错误。实际上，我们看到的大部分返回的内容都与一个已安装的软件包`liberror`有关！在实践中，你可能会处理在错误方面具有预定义结构的日志文件。在这种情况下，更容易确定一个只记录真正错误的搜索模式。

# 循环控制

在这一点上，你应该对使用`while`和`for`循环感到满意。关于循环，还有一个更重要的话题需要讨论：**循环控制**。循环控制是一个通用术语，用于控制循环的任何操作！然而，如果我们想要发挥循环的全部威力，有两个*关键字*是必须的：`break`和`continue`。我们将从`break`开始。

# 打破循环

对于一些脚本逻辑，有必要跳出循环。你可以想象，在你的某个脚本中，你正在等待某件事完成。一旦发生，你就想*做点什么*。在`while true`循环中等待并定期检查可能是一个选择，但是如果你回想一下`while-interactive.sh`脚本，我们在谜底得到成功答案时退出了。在退出时，我们不能运行任何超出`while`循环之外的命令！这就是`break`发挥作用的地方。它允许我们退出*循环*，但继续*脚本*。首先，让我们更新`while-interactive.sh`以利用这个循环控制关键字：

```
reader@ubuntu:~/scripts/chapter_11$ vim while-interactive.sh 
reader@ubuntu:~/scripts/chapter_11$ cat while-interactive.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
# Date: 2018-10-28
# Description: A simple riddle in a while loop.
# Usage: ./while-interactive.sh
#####################################

# Infinite loop, only exits on correct answer.
while true; do
  read -p "I have keys but no locks. I have a space but no room. You can enter, but can’t go outside. What am I? " answer
  if [[ ${answer} =~ [Kk]eyboard ]]; then # Use regular expression so 'a keyboard' or 'Keyboard' is also a valid answer.
    echo "Correct, congratulations!"
    break # Exit the while loop.
  else
    # Print an error message and go back into the loop.
    echo "Incorrect, please try again."
  fi
done

# This will run after the break in the while loop.
echo "Now we can continue after the while loop is done, awesome!"
```

我们做了三个更改：

+   采用了更高的版本号

+   将`exit 0`替换为`break`

+   在 while 循环后添加一个简单的`echo`

当我们仍然使用`exit 0`时，最终的`echo`将永远不会运行（但不要相信我们，一定要自己验证一下！）。现在，用`break`运行它并观察：

```
reader@ubuntu:~/scripts/chapter_11$ bash while-interactive.sh 
I have keys but no locks. I have a space but no room. You can enter, but can’t go outside. What am I? keyboard
Correct, congratulations!
Now we can continue after the while loop is done, awesome!
```

这就是，在一个中断的`while`循环之后的代码执行。通常，在一个无限循环之后，肯定有其他需要执行的代码，这就是做到的方式。

我们不仅可以在`while`循环中使用`break`，而且在`for`循环中当然也可以。下面的例子显示了我们如何在`for`循环中使用`break`：

```
reader@ubuntu:~/scripts/chapter_11$ vim for-loop-control.sh
reader@ubuntu:~/scripts/chapter_11$ cat for-loop-control.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-28
# Description: Loop control in a for loop.
# Usage: ./for-loop-control.sh
#####################################

# Generate a random number from 1-10.
random_number=$(( ( RANDOM % 10 )  + 1 ))

# Iterate over all possible random numbers.
for number in {1..10}; do

  if [[ ${number} -eq ${random_number} ]]; then
    echo "Random number found: ${number}."
    break # As soon as we have found the number, stop.
  fi

  # If we get here the number did not match.
  echo "Number does not match: ${number}."
done
echo "Number has been found, all done."
```

在此脚本功能的顶部，确定一个 1 到 10 之间的随机数（不用担心语法）。接下来，我们遍历 1 到 10 的数字，对于每个数字，我们将检查它是否等于随机生成的数字。如果是，我们打印一个成功的消息*并且我们中断循环*。否则，我们将跳出`if-then`块并打印失败消息。如果我们没有包括中断语句，输出将如下所示：

```
reader@ubuntu:~/scripts/chapter_11$ bash for-loop-control.sh 
Number does not match: 1.
Number does not match: 2.
Number does not match: 3.
Random number found: 4.
Number does not match: 4.
Number does not match: 5.
Number does not match: 6.
Number does not match: 7.
Number does not match: 8.
Number does not match: 9.
Number does not match: 10.
Number has been found, all done.
```

我们不仅看到数字被打印为匹配和不匹配（这当然是一个逻辑错误），而且当我们确定那些数字不会匹配时，脚本还会继续检查所有其他数字。现在，如果我们使用 exit 而不是 break，最终的语句将永远不会被打印：

```
reader@ubuntu:~/scripts/chapter_11$ bash for-loop-control.sh 
Number does not match: 1.
Number does not match: 2.
Number does not match: 3.
Number does not match: 4.
Number does not match: 5.
Number does not match: 6.
Random number found: 7.
```

只有使用`break`，我们才会得到我们需要的确切数量的输出；既不多也不少。你可能已经看到，我们也可以为`Number does not match:`消息使用`else`子句。但是，没有什么会阻止程序。所以即使随机数第一次就被找到（最终会发生的），它仍然会比较列表中的所有值，直到达到该列表的末尾。

这不仅是浪费时间和资源，而且想象一下，如果随机数在 1 到 1,000,000 之间！只要记住：如果你完成了循环，**跳出它。**

# 继续关键字

和 Bash（以及生活）中的大多数事情一样，`break`有一个对应的`continue`关键字。如果你使用 continue，你是告诉循环停止当前循环，但*继续*下一次运行。所以，不是停止整个循环，你只是停止当前迭代。让我们看看另一个例子是否能澄清这一点：

```
reader@ubuntu:~/scripts/chapter_11$ vim for-continue.sh
reader@ubuntu:~/scripts/chapter_11$ cat for-continue.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-28
# Description: For syntax with a continue.
# Usage: ./for-continue.sh
#####################################

# Look at numbers 1-20, in steps of 2.
for number in {1..20..2}; do
  if [[ $((${number}%5)) -eq 0 ]]; then
    continue # Unlucky number, skip this!
  fi

  # Show the user which number we've processed.
  echo "Looking at number: ${number}."

done
```

在这个例子中，所有可以被 5 整除的数字都被认为是不幸的，不应该被处理。这是通过`[[ $((${number}%5)) -eq 0 ]]`条件实现的：

+   **[[** $((${number}%5)) **-eq 0 ]]** -> 测试语法

+   [[ **$((**${number}%5**))** -eq 0 ]] -> 算术语法

+   [[ $((**${number}%5**)) -eq 0 ]] -> 变量**number**的模 5

如果数字通过了这个测试（因此可以被 5 整除，比如 5、10、15、20 等），将执行`continue`。当这发生时，循环的下一个迭代将运行（并且`echo`**不会**被执行！），当运行这个脚本时可以看到：

```
reader@ubuntu:~/scripts/chapter_11$ bash for-continue.sh 
Looking at number: 1.
Looking at number: 3.
Looking at number: 7.
Looking at number: 9.
Looking at number: 11.
Looking at number: 13.
Looking at number: 17.
Looking at number: 19.
```

如列表所示，数字`5`、`10`和`15`被处理，但我们在`echo`中看不到它们。我们还可以看到之后的一切，这在使用`break`时是不会发生的。使用`bash -x`验证这是否真的发生了（警告：大量输出！），并检查如果你用`break`或甚至`exit`替换`continue`会发生什么。 

# 循环控制和嵌套

在本章的最后部分，我们想向您展示如何使用循环控制来影响`嵌套`循环。`break`和`continue`都将带有一个额外的参数：指定要中断的循环。默认情况下，如果省略了此参数，就假定为`1`。因此，`break`命令等同于`break 1`，`continue 1`等同于`continue`。正如之前所述，我们理论上可以将我们的循环嵌套得很深；你可能会比你的现代系统的技术能力更早地遇到逻辑问题！我们将看一个简单的例子，向我们展示如何使用`break 2`不仅可以跳出`for`循环，还可以跳出外部的`while`循环：

```
reader@ubuntu:~/scripts/chapter_11$ vim break-x.sh 
reader@ubuntu:~/scripts/chapter_11$ cat break-x.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-10-28
# Description: Breaking out of nested loops.
# Usage: ./break-x.sh
#####################################

while true; do
  echo "This is the outer loop."
  sleep 1

  for iteration in {1..3}; do
    echo "This is inner loop ${iteration}."
    sleep 1
  done
done
echo "This is the end of the script, thanks for playing!"
```

这个脚本的第一个版本不包含`break`。当我们运行它时，我们永远看不到最终的消息，而且我们得到一个无休止的重复模式：

```
reader@ubuntu:~/scripts/chapter_11$ bash break-x.sh 
This is the outer loop.
This is inner loop 1.
This is inner loop 2.
This is inner loop 3.
This is the outer loop.
This is inner loop 1.
^C
```

现在，让我们在迭代达到`2`时中断内部循环：

```
reader@ubuntu:~/scripts/chapter_11$ vim break-x.sh 
reader@ubuntu:~/scripts/chapter_11$ cat break-x.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
# Date: 2018-10-28
# Description: Breaking out of nested loops.
# Usage: ./break-x.sh
#####################################
<SNIPPED>
  for iteration in {1..3}; do
    echo "This is inner loop ${iteration}."
    if [[ ${iteration} -eq 2 ]]; then
      break 1
    fi
    sleep 1
  done
<SNIPPED>
```

现在运行脚本时，我们仍然得到无限循环，但在三次迭代之后，我们缩短了内部 for 循环：

```
reader@ubuntu:~/scripts/chapter_11$ bash break-x.sh 
This is the outer loop.
This is inner loop 1.
This is inner loop 2.
This is the outer loop.
This is inner loop 1.
^C
```

现在，让我们使用`break 2`命令指示内部循环跳出外部循环：

```
reader@ubuntu:~/scripts/chapter_11$ vim break-x.sh 
reader@ubuntu:~/scripts/chapter_11$ cat break-x.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.2.0
# Date: 2018-10-28
# Description: Breaking out of nested loops.
# Usage: ./break-x.sh
#####################################
<SNIPPED>
    if [[ ${iteration} -eq 2 ]]; then
      break 2 # Break out of the outer while-true loop.
    fi
<SNIPPED>
```

看，内部循环成功地跳出了外部循环：

```
reader@ubuntu:~/scripts/chapter_11$ bash break-x.sh 
This is the outer loop.
This is inner loop 1.
This is inner loop 2.
This is the end of the script, thanks for playing!
```

我们可以完全控制我们的循环，即使我们需要嵌套尽可能多的循环来满足我们的脚本需求。相同的理论也适用于`continue`。在这个例子中，如果我们使用`continue 2`而不是`break 2`，我们仍然会得到一个无限循环（因为`while true`永远不会结束）。然而，如果您的其他循环也是`for`或非无限的`while`循环（根据我们的经验，这更常见，但不适合作为一个很好的简单例子），`continue 2`可以让您执行恰好符合情况需求的逻辑。

# 总结

本章是关于条件测试和脚本循环的。由于我们已经讨论了`if-then-else`语句，所以在展示条件测试工具包的更高级用法之前，我们回顾了这些信息。这些高级信息包括在条件测试场景中使用正则表达式，我们在上一章中学习过，以实现更灵活的测试。我们还向您展示了如何使用`elif`（`else if`的缩写）来顺序测试多个条件。我们解释了如何嵌套多个`if-then-else`语句以创建高级逻辑。

在本章的第二部分，我们介绍了`while`循环。我们向您展示了如何使用它来创建一个永远运行的脚本，或者如何使用条件来在满足某些条件时停止循环。我们介绍了`until`关键字，它与`while`具有相同的功能，但允许进行负检查，而不是`while`的正检查。我们通过向您展示如何在一个无休止的`while`循环中创建一个交互式脚本来结束对`while`的解释（使用我们的老朋友`read`）。

在`while`之后，我们介绍了更强大的`for`循环。这个循环可以做与`while`相同的事情，但通常更短的语法允许我们编写更少的代码（以及更可读的代码，这仍然是脚本编写中非常重要的一个方面！）。我们向您展示了`for`如何遍历列表，以及如何使用*大括号扩展*来创建数字列表。我们通过给出一个实际的例子，结合`for`和文件通配模式，来结束我们对`for`循环的讨论，以便我们可以动态查找、获取和处理文件。

我们通过解释循环控制来结束了本章，Bash 中使用`break`和`continue`关键字来实现。这些关键字允许我们从循环中*跳出*（甚至从嵌套循环中，直到我们需要的外部循环），并且还允许我们停止循环的当前迭代，*继续*到下一个迭代。

本章介绍了以下命令/关键字：`elif`、`help`、`while`、`sleep`、`for`、`basename`、`break`和`continue`。

# 问题

1.  `if-then`（`-else`）语句是如何结束的？

1.  我们如何在条件评估中使用正则表达式搜索模式？

1.  我们为什么需要`elif`关键字？

1.  什么是*嵌套*？

1.  我们如何获取有关如何使用 shell 内置命令和关键字的信息？

1.  `while`的相反关键字是什么？

1.  为什么我们会选择`for`循环而不是`while`循环？

1.  大括号扩展是什么，我们可以在哪些字符上使用它？

1.  哪两个关键字允许我们对循环有更精细的控制？

1.  如果我们嵌套循环，我们如何使用循环控制来影响内部循环的外部循环？

# 进一步阅读

如果您想更深入地了解本章的主题，以下资源可能会很有趣：

+   **case 语句**: [`tldp.org/LDP/Bash-Beginners-Guide/html/sect_07_03.html`](http://tldp.org/LDP/Bash-Beginners-Guide/html/sect_07_03.html)

+   **大括号扩展**: [`wiki.bash-hackers.org/syntax/expansion/brace`](http://wiki.bash-hackers.org/syntax/expansion/brace)

+   **Linux 文档项目关于循环**: [`www.tldp.org/LDP/abs/html/loops1.html`](http://www.tldp.org/LDP/abs/html/loops1.html)
