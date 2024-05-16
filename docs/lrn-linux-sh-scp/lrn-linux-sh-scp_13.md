# 第十三章：函数

在本章中，我们将解释 Bash 脚本的一个非常实用的概念：函数。我们将展示它们是什么，我们如何使用它们，以及为什么我们想要使用它们。

在介绍了函数的基础知识之后，我们将进一步探讨函数如何具有自己的输入和输出。

将描述函数库的概念，并且我们将开始构建自己的个人函数库，其中包含各种实用函数。

本章将介绍以下命令：`top`、`free`、`declare`、`case`、`rev`和`return`。

本章将涵盖以下主题：

+   函数解释

+   使用参数增强函数

+   函数库

# 技术要求

本章的所有脚本都可以在 GitHub 上找到：[`github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter13`](https://github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter13)。除了您的 Ubuntu Linux 虚拟机外，在本章的示例中不需要其他资源。对于 argument-checker.sh、functions-and-variables.sh、library-redirect-to-file.sh 脚本，只能在网上找到最终版本。在执行脚本之前，请务必验证头部中的脚本版本。

# 函数解释

在本章中，我们将讨论函数以及这些如何增强你的脚本。函数的理论并不太复杂：函数是一组命令，可以被多次调用（执行），而无需再次编写整组命令。一如既往，一个好的例子胜过千言万语，所以让我们立即用我们最喜欢的例子之一来深入研究：打印“Hello world！”。

# Hello world！

我们现在知道，相对容易让单词“Hello world！”出现在我们的终端上。简单的`echo "Hello world!"`就可以做到。然而，如果我们想要多次这样做，我们该怎么做呢？你可以建议使用任何一种循环，这确实可以让我们多次打印。然而，该循环还需要一些额外的代码和提前规划。正如你将注意到的，实际上循环非常适合迭代项目，但并不完全适合以可预测的方式重用代码。让我们看看我们如何使用函数来代替这样做：

```
reader@ubuntu:~/scripts/chapter_13$ vim hello-world-function.sh
reader@ubuntu:~/scripts/chapter_13$ cat hello-world-function.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-11
# Description: Prints "Hello world!" using a function.
# Usage: ./hello-world-function.sh
#####################################

# Define the function before we call it.
hello_world() {
  echo "Hello world!"
}

# Call the function we defined earlier:
hello_world

reader@ubuntu:~/scripts/chapter_13$ bash hello-world-function.sh 
Hello world!
```

正如你所看到的，我们首先定义了函数，这只不过是写下应该在函数被调用时执行的命令。在脚本的末尾，你可以看到我们通过输入函数名来执行函数，就像执行任何其他命令一样。重要的是要注意，只有在之前定义了函数的情况下，你才能调用函数。这意味着整个函数定义需要在脚本中的调用之前。现在，我们将把所有函数放在脚本中的第一项。在本章的后面，我们将向你展示如何更有效地使用它。

在上一个例子中，你看到的是 Bash 中函数定义的两种可能语法中的第一种。如果我们只提取函数，语法如下：

```
function_name() {
   indented-commands
   further-indented-commands-as-needed
 }
```

第二种可能的语法，我们不太喜欢，是这样的：

```
function function_name {
   indented-commands
   further-indented-commands-as-needed
 }
```

两种语法的区别在于函数名之前没有`function`一词，或者在函数名后没有`()`。我们更喜欢第一种语法，它使用`()`符号，因为它更接近其他脚本/编程语言的符号，并且对大多数人来说应该更容易识别。而且，作为额外的奖励，它比第二种符号更短、更简单。正如你所期望的，我们将在本书的其余部分继续使用第一种符号；第二种符号是为了完整性而呈现的（如果你在研究脚本时在网上遇到它，了解它总是方便的！）。

记住，我们使用缩进来向脚本的读者传达命令嵌套的信息。在这种情况下，由于函数中的所有命令只有在调用函数时才运行，我们用两个空格缩进它们，这样就清楚地表明我们在函数内部。

# 更复杂

函数可以有尽可能多的命令。在我们简单的例子中，我们只添加了一个`echo`，然后只调用了一次。虽然这对于抽象来说很好，但并不真正需要创建一个函数（尚未）。让我们看一个更复杂的例子，这将让您更好地了解为什么在函数中抽象命令是一个好主意：

```
reader@ubuntu:~/scripts/chapter_13$ vim complex-function.sh 
reader@ubuntu:~/scripts/chapter_13$ cat complex-function.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-11
# Description: A more complex function that shows why functions exist.
# Usage: ./complex-function.sh
#####################################

# Used to print some current data on the system.
print_system_status() {
  date # Print the current datetime.
  echo "CPU in use: $(top -bn1 | grep Cpu | awk '{print $2}')"
  echo "Memory in use: $(free -h | grep Mem | awk '{print $3}')"
  echo "Disk space available on /: $(df -k / | grep / | awk '{print $4}')" 
  echo # Extra newline for readability.
}

# Print the system status a few times.
for ((i=0; i<5; i++)); do
  print_system_status
  sleep 5
done
```

现在我们在谈论！这个函数有五个命令，其中三个包括使用链式管道的命令替换。现在，我们的脚本开始变得复杂而强大。正如您所看到的，我们使用`()`符号来定义函数。然后我们在 C 风格的`for`循环中调用这个函数，这会导致脚本在每次系统状态之间暂停五秒钟后打印系统状态五次（由于`sleep`，我们在第十一章中看到过，*条件测试和脚本循环*）。当您运行这个脚本时，它应该看起来像这样：

```
reader@ubuntu:~/scripts/chapter_13$ bash complex-function.sh 
Sun Nov 11 13:40:17 UTC 2018
CPU in use: 0.1
Memory in use: 85M
Disk space available on /: 4679156

Sun Nov 11 13:40:22 UTC 2018
CPU in use: 0.2
Memory in use: 84M
Disk space available on /: 4679156
```

除了日期之外，其他输出发生显着变化的可能性很小，除非您有其他进程在运行。然而，函数的目的应该是清楚的：以透明的方式定义和抽象一组功能。

虽然不是本章的主题，但我们在这里使用了一些新命令。`top`和`free`命令通常用于检查系统的性能，并且可以在没有任何参数的情况下使用（`top`打开全屏，您可以使用*Ctrl *+ *C*退出）。在本章的*进一步阅读*部分，您可以找到有关 Linux 中这些（和其他）性能监控工具的更多信息。我们还在那里包括了`awk`的入门知识。

使用函数有许多优点；其中包括但不限于以下内容：

+   易于重用代码

+   允许代码共享（例如通过库）

+   将混乱的代码抽象为简单的函数调用

函数中的一个重要事项是命名。函数名应尽可能简洁，但仍需要告诉用户它的作用。例如，如果您将一个函数命名为`function1`这样的非描述性名称，任何人怎么知道它的作用呢？将其与我们在示例中看到的名称进行比较：`print_system_status`。虽然也许不完美（什么是系统状态？），但至少指引我们朝着正确的方向（如果您同意 CPU、内存和磁盘使用率被认为是系统状态的一部分的话）。也许函数的一个更好的名称是`print_cpu_mem_disk`。这取决于您的决定！确保在做出这个选择时考虑目标受众是谁；这通常会产生最大的影响。

虽然在函数命名中描述性非常重要，但遵守命名约定也同样重要。当我们处理变量命名时，我们已经在第八章中提出了同样的考虑。重申一下：最重要的规则是*保持一致*。如果您想要我们对函数命名约定的建议，那就坚持我们为变量制定的规则：小写，用下划线分隔。这就是我们在之前的例子中使用的方式，也是我们将在本书的其余部分继续展示的方式。

# 变量作用域

虽然函数很棒，但我们之前学到的一些东西在函数的范围内需要重新考虑，尤其是变量。我们知道变量存储的信息可以在脚本的多个地方多次访问或改变。然而，我们还没有学到的是变量总是有一个*作用域*。默认情况下，变量的作用域是*全局*的，这意味着它们可以在脚本的任何地方使用。随着函数的引入，还有一个新的作用域：*局部*。局部变量在函数内部定义，并随着函数调用而存在和消失。让我们看看这个过程：

```
reader@ubuntu:~/scripts/chapter_13$ vim functions-and-variables.sh
reader@ubuntu:~/scripts/chapter_13$ cat functions-and-variables.sh 
#!/bin/bash
#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-11
# Description: Show different variable scopes.
# Usage: ./functions-and-variables.sh <input>
#####################################

# Check if the user supplied at least one argument.
if [[ $# -eq 0 ]]; then
  echo "Missing an argument!"
  echo "Usage: $0 <input>"
  exit 1
fi

# Assign the input to a variable.
input_variable=$1
# Create a CONSTANT, which never changes.
CONSTANT_VARIABLE="constant"

# Define the function.
hello_variable() {
  echo "This is the input variable: ${input_variable}"
  echo "This is the constant: ${CONSTANT_VARIABLE}"
}

# Call the function.
hello_variable
reader@ubuntu:~/scripts/chapter_13$ bash functions-and-variables.sh teststring
This is the input variable: teststring
This is the constant: constant
```

到目前为止，一切都很好。我们可以在函数中使用我们的*全局*常量。这并不令人惊讶，因为它不是轻易被称为全局变量；它可以在脚本的任何地方使用。现在，让我们看看当我们在函数中添加一些额外的变量时会发生什么：

```
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
# Date: 2018-11-11
# Description: Show different variable scopes.
# Usage: ./functions-and-variables.sh <input>
#####################################
<SNIPPED>
# Define the function.
hello_variable() {
 FUNCTION_VARIABLE="function variable text!"
  echo "This is the input variable: ${input_variable}"
  echo "This is the constant: ${CONSTANT_VARIABLE}"
 echo "This is the function variable: ${FUNCTION_VARIABLE}"
}

# Call the function.
hello_variable

# Try to call the function variable outside the function.
echo "Function variable outside function: ${FUNCTION_VARIABLE}"
```

你认为现在会发生什么？试一试：

```
reader@ubuntu:~/scripts/chapter_13$ bash functions-and-variables.sh input
This is the input variable: input
This is the constant: constant
This is the function variable: function variable text!
Function variable outside function: function variable text!
```

与你可能怀疑的相反，我们在函数内部定义的变量实际上仍然是一个全局变量（对于欺骗你感到抱歉！）。如果我们想要使用局部作用域变量，我们需要添加内置的 local shell：

```
#!/bin/bash
#####################################
# Author: Sebastiaan Tammer
# Version: v1.2.0
# Date: 2018-11-11
# Description: Show different variable scopes.
# Usage: ./functions-and-variables.sh <input>
#####################################
<SNIPPED>
# Define the function.
hello_variable() {
 local FUNCTION_VARIABLE="function variable text!"
  echo "This is the input variable: ${input_variable}"
  echo "This is the constant: ${CONSTANT_VARIABLE}"
  echo "This is the function variable: ${FUNCTION_VARIABLE}"
}
<SNIPPED>
```

现在，如果我们这次执行它，我们实际上会看到脚本在最后一个命令上表现不佳：

```
reader@ubuntu:~/scripts/chapter_13$ bash functions-and-variables.sh more-input
This is the input variable: more-input
This is the constant: constant
This is the function variable: function variable text!
Function variable outside function: 
```

由于局部添加，我们现在只能在函数内部使用变量及其内容。因此，当我们调用`hello_variable`函数时，我们看到变量的内容，但当我们尝试在函数外部打印它时，在`echo "Function variable outside function: ${FUNCTION_VARIABLE}"`中，我们看到它是空的。这是预期的和理想的行为。实际上，你可以做的，有时确实很方便，是这样的：

```
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.3.0
# Date: 2018-11-11
# Description: Show different variable scopes.
# Usage: ./functions-and-variables.sh <input>
#####################################
<SNIPPED>
# Define the function.
hello_variable() {
 local CONSTANT_VARIABLE="maybe not so constant?"
  echo "This is the input variable: ${input_variable}"
  echo "This is the constant: ${CONSTANT_VARIABLE}"
}

# Call the function.
hello_variable

# Try to call the function variable outside the function.
echo "Function variable outside function: ${CONSTANT_VARIABLE}"
```

现在，我们已经定义了一个与我们已经初始化的全局作用域变量*同名*的局部作用域变量！你可能已经对接下来会发生什么有所想法，但一定要运行脚本并理解为什么会发生这种情况：

```
reader@ubuntu:~/scripts/chapter_13$ bash functions-and-variables.sh last-input
This is the input variable: last-input
This is the constant: maybe not so constant?
Function variable outside function: constant
```

所以，当我们在函数中使用`CONSTANT_VARIABLE`变量（记住，常量仍然被认为是变量，尽管是特殊的变量）时，它打印了局部作用域变量的值：`也许不那么常量？`。当在函数外，在脚本的主体部分，我们再次打印变量的值时，我们得到了最初定义的值：`constant`。

你可能很难想象这种情况的用例。虽然我们同意你可能不经常使用这个，但它确实有它的用处。例如，想象一个复杂的脚本，其中一个全局变量被多个函数和命令顺序使用。现在，你可能会遇到这样一种情况，你需要变量的值，但稍微修改一下才能在函数中正确使用它。你还知道后续的函数/命令需要原始值。现在，你可以将内容复制到一个新变量中并使用它，但是通过在函数内部*覆盖*变量，你让读者/用户更清楚地知道你有一个目的；这是一个经过深思熟虑的决定，你知道你需要这个例外*仅仅是为了那个函数*。使用局部作用域变量（最好还加上注释，像往常一样）将确保可读性！

变量可以通过使用内置的`declare` shell 设置为只读。如果你查看帮助，使用`help declare`，你会看到它被描述为“设置变量值和属性”。通过用`declare -r CONSTANT=VALUE`替换`CONSTANT=VALUE`，可以创建一个只读变量，比如常量。如果你这样做，你就不能再（临时）用本地实例覆盖变量；Bash 会给你一个错误。实际上，就我们遇到的情况而言，`declare`命令并没有被使用得太多，但它除了只读声明之外还可以有其他有用的用途，所以一定要看一看！

# 实际例子

在本章的下一部分介绍函数参数之前，我们将首先看一个不需要参数的函数的实际示例。我们将回到我们之前创建的脚本，并查看是否有一些功能可以抽象为一个函数。剧透警告：有一个很棒的功能，涉及到一点叫做错误处理的东西！

# 错误处理

在第九章中，*错误检查和处理*，我们创建了以下结构：`command || { echo "Something went wrong."; exit 1; }`。正如你（希望）记得的那样，`||`语法意味着只有在左侧命令的退出状态不是`0`时，右侧的所有内容才会被执行。虽然这种设置运行良好，但并没有增加可读性。如果我们能将错误处理抽象为一个函数，并调用该函数，那将会更好！让我们就这样做：

```
reader@ubuntu:~/scripts/chapter_13$ vim error-functions.sh
reader@ubuntu:~/scripts/chapter_13$ cat error-functions.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-11
# Description: Functions to handle errors.
# Usage: ./error-functions.sh
#####################################

# Define a function that handles minor errors.
handle_minor_error() {
 echo "A minor error has occured, please check the output."
}

# Define a function that handles fatal errors.
handle_fatal_error() {
 echo "A critical error has occured, stopping script."
 exit 1
}

# Minor failures.
ls -l /tmp/ || handle_minor_error
ls -l /root/ || handle_minor_error 

# Fatal failures.
cat /etc/shadow || handle_fatal_error
cat /etc/passwd || handle_fatal_error
```

这个脚本定义了两个函数：`handle_minor_error`和`handle_fatal_error`。对于轻微的错误，我们会打印一条消息，但脚本的执行不会停止。然而，致命错误被认为是如此严重，以至于脚本的流程预计会被中断；在这种情况下，继续执行脚本是没有用的，所以我们会确保函数停止它。通过使用这些函数与`||`结构，我们不需要在函数内部检查退出码；我们只有在退出码不是`0`时才会进入函数，所以我们已经知道我们处于错误的情况中。在执行这个脚本之前，花点时间反思一下*我们通过这些函数改进了多少可读性*。当你完成后，用调试输出运行这个脚本，这样你就可以跟踪整个流程。

```
reader@ubuntu:~/scripts/chapter_13$ bash -x error-functions.sh 
+ ls -l /tmp/
total 8
drwx------ 3 root root 4096 Nov 11 11:07 systemd-private-869037dc...
drwx------ 3 root root 4096 Nov 11 11:07 systemd-private-869037dc...
+ ls -l /root/
ls: cannot open directory '/root/': Permission denied
+ handle_minor_error
+ echo 'A minor error has occured, please check the output.'
A minor error has occured, please check the output.
+ cat /etc/shadow
cat: /etc/shadow: Permission denied
+ handle_fatal_error
+ echo 'A critical error has occured, stopping script.'
A critical error has occured, stopping script.
+ exit 1
```

正如你所看到的，第一个命令`ls -l /tmp/`成功了，我们看到了它的输出；我们没有进入`handle_minor_error`函数。下一个命令，我们确实希望它失败，它的确失败了。我们看到现在我们进入了函数，并且我们在那里指定的错误消息被打印出来。但是，由于这只是一个轻微的错误，我们继续执行脚本。然而，当我们到达`cat /etc/shadow`时，我们认为这是一个重要的组件，我们遇到了一个`Permission denied`的消息，导致脚本执行`handle_fatal_error`。因为这个函数有一个`exit 1`，脚本被终止，第四个命令就不会被执行。这应该说明另一个观点：一个`exit`，即使在函数内部，也是全局的，会终止脚本（不仅仅是函数）。如果你希望看到这个脚本成功，用`sudo bash error-functions.sh`来运行它。你会看到两个错误函数都没有被执行。

# 用参数增强函数

正如脚本可以接受参数的形式输入一样，函数也可以。实际上，大多数函数都会使用参数。静态函数，比如之前的错误处理示例，不如它们的参数化对应函数强大或灵活。

# 丰富多彩

在下一个示例中，我们将创建一个脚本，允许我们以几种不同的颜色打印文本到我们的终端。它基于一个具有两个参数的函数来实现：`string`和`color`。看一下以下命令：

```
reader@ubuntu:~/scripts/chapter_13$ vim colorful.sh 
reader@ubuntu:~/scripts/chapter_13$ cat colorful.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-17
# Description: Some printed text, now with colors!
# Usage: ./colorful.sh
#####################################

print_colored() {
  # Check if the function was called with the correct arguments.
  if [[ $# -ne 2 ]]; then
    echo "print_colored needs two arguments, exiting."
    exit 1
  fi

  # Grab both arguments.
  local string=$1
  local color=$2

  # Use a case-statement to determine the color code.
  case ${color} in
  red)
    local color_code="\e[31m";;
  blue)
    local color_code="\e[34m";;
  green)
    local color_code="\e[32m";;
  *)
    local color_code="\e[39m";; # Wrong color, use default.
  esac

  # Perform the echo, and reset color to default with [39m.
  echo -e ${color_code}${string}"\e[39m"
}

# Print the text in different colors.
print_colored "Hello world!" "red"
print_colored "Hello world!" "blue"
print_colored "Hello world!" "green"
print_colored "Hello world!" "magenta"
```

这个脚本中发生了很多事情。为了帮助你理解，我们将逐步地逐个部分地进行讲解，从函数定义的第一部分开始：

```
print_colored() {
  # Check if the function was called with the correct arguments.
  if [[ $# -ne 2 ]]; then
    echo "print_colored needs two arguments, exiting."
    exit 1
  fi

  # Grab both arguments.
  local string=$1
  local color=$2
```

在函数体内部，我们首先检查参数的数量。语法与我们通常对整个脚本传递的参数进行检查的方式相同，这可能会有所帮助，也可能会有些困惑。一个要意识到的好事是，`$#`结构适用于其所在的范围；如果它在主脚本中使用，它会检查那里传递的参数。如果像这里一样在函数内部使用，它会检查传递给函数的参数数量。对于`$1`、`$2`等也是一样：如果在函数内部使用，它们指的是传递给函数的有序参数，而不是一般脚本中的参数。当我们获取参数时，我们将它们写入*本地*变量；在这个简单的脚本中，我们不一定需要这样做，但是在本地范围内使用变量时，将变量标记为本地总是一个好习惯。您可能会想象，在更大、更复杂的脚本中，许多函数使用可能会意外地被称为相同的东西（在这种情况下，`string`是一个非常常见的词）。通过将它们标记为本地，您不仅提高了可读性，还防止了由具有相同名称的变量引起的错误；总的来说，这是一个非常好的主意。让我们回到脚本的下一部分，即`case`语句：

```
  # Use a case-statement to determine the color code.
  case ${color} in
  red)
    color_code="\e31m";;
  blue)
    color_code="\e[34m";;
  green)
    color_code="\e[32m";;
  *)
    color_code="\e[39m";; # Wrong color, use default.
  esac
```

现在是介绍`case`的绝佳时机。`case`语句基本上是一个非常长的`if-then-elif-then-elif-then...`链。变量的选项越多，链条就会变得越长。使用`case`，您只需说`对于${variable}中的特定值，执行<某些操作>`。在我们的例子中，这意味着如果`${color}`变量是`red`，我们将设置另一个`color_code`变量为`\e[31m`（稍后会详细介绍）。如果它是`blue`，我们将执行其他操作，对于`green`也是一样。最后，我们将定义一个通配符；未指定的变量值将通过这里，作为一种通用的构造。如果指定的颜色是一些不兼容的东西，比如**dog**，我们将只设置默认颜色。另一种选择是中断脚本，这对于错误的颜色有点反应过度。要终止`case`，您将使用`esac`关键字（这是`case`的反义词），类似于`if`被其反义词`fi`终止的方式。

现在，让我们来谈谈*终端上的颜色*的技术方面。虽然我们学到的大多数东西都是关于 Bash 或 Linux 特定的，但打印颜色实际上是由您的终端仿真器定义的。我们正在使用的颜色代码非常标准，应该被您的终端解释为*不要字面打印这个字符，而是改变`颜色`为`<颜色>`*。终端看到一个*转义序列*，`\e`，后面跟着一个*颜色代码*，`[31m`，并且知道您正在指示它打印一个与之前定义的颜色不同的颜色（通常是该终端仿真器的默认设置，除非您自己更改了颜色方案）。您可以使用转义序列做更多的事情（当然，只要您的终端仿真器支持），比如创建粗体文本、闪烁文本，以及为文本设置另一个背景颜色。现在，请记住*\e[31m 序列不会被打印，而是被解释*。对于`case`中的通配符，您不想显式设置颜色，而是向终端发出信号，以使用*默认*颜色打印。这意味着对于每个兼容的终端仿真器，文本都以用户选择的颜色（或默认分配的颜色）打印。

现在是脚本的最后部分：

```
  # Perform the echo, and reset color to default with [39m.
  echo -e ${color_code}${string}"\e[39m"
}

# Print the text in different colors.
print_colored "Hello world!" "red"
print_colored "Hello world!" "blue"
print_colored "Hello world!" "green"
print_colored "Hello world!" "magenta"
```

`print_colored`函数的最后一部分实际上打印了有颜色的文本。它通过使用带有`-e`标志的老式`echo`来实现这一点。`man echo`显示`-e`*启用反斜杠转义的解释*。如果您不指定此选项，您的输出将只是类似于`\e[31mHello world!\e[39m`。在这种情况下需要知道的一件好事是，一旦您的终端遇到颜色代码转义序列，*随后的所有文本都将以该颜色打印*！因此，我们用`"\e[39m"`结束 echo，将所有后续文本的颜色重置为默认值。

最后，我们多次调用函数，第一个参数相同，但第二个参数（颜色）不同。如果您运行脚本，输出应该类似于这样：

![

在前面的截图中，我的颜色方案设置为绿底黑字，这就是为什么最后的`Hello world!`是鲜绿色的原因。您可以看到它与`bash colorful.sh`的颜色相同，这应该足以让您确信`[39m`颜色代码实际上是默认值。

# 返回值

有些功能遵循*处理器*原型：它们接受输入，对其进行处理，然后将结果返回给调用者。这是经典功能的一部分：根据输入，生成不同的输出。我们将通过一个示例来展示这一点，该示例将用户指定的输入反转为脚本。通常使用`rev`命令来完成这个功能（实际上我们的函数也将使用`rev`来实现），但我们将创建一个包装函数，增加一些额外的功能：

```
reader@ubuntu:~/scripts/chapter_13$ vim reverser.sh 
reader@ubuntu:~/scripts/chapter_13$ cat reverser.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-17
# Description: Reverse the input for the user.
# Usage: ./reverser.sh <input-to-be-reversed>
#####################################

# Check if the user supplied one argument.
if [[ $# -ne 1 ]]; then
  echo "Incorrect number of arguments!"
  echo "Usage: $0 <input-to-be-reversed>"
  exit 1
fi

# Capture the user input in a variable.
user_input="_${1}_" # Add _ for readability.

# Define the reverser function.
reverser() {
  # Check if input is correctly passed.
  if [[ $# -ne 1 ]]; then
    echo "Supply one argument to reverser()!" && exit 1
  fi

  # Return the reversed input to stdout (default for rev).
  rev <<< ${1}
}

# Capture the function output via command substitution.
reversed_input=$(reverser ${user_input})

# Show the reversed input to the user.
echo "Your reversed input is: ${reversed_input}"
```

由于这又是一个更长、更复杂的脚本，我们将逐步查看它，以确保您完全理解。我们甚至在其中加入了一个小惊喜，证明了我们之前的说法之一，但我们稍后再谈。我们将跳过标题和输入检查，转而捕获变量：

```
# Capture the user input in a variable.
user_input="_${1}_" # Add _ for readability.
```

在以前的大多数示例中，我们总是直接将输入映射到变量。但是，这一次我们要表明您实际上也可以添加一些额外的文本。在这种情况下，我们通过用户输入并在其前后添加下划线。如果用户输入`rain`，那么变量实际上将包含`_rain_`。这将在后面证明有洞察力。现在，对于函数定义，我们使用以下代码：

```
# Define the reverser function.
reverser() {
  # Check if input is correctly passed.
  if [[ $# -ne 1 ]]; then
    echo "Supply one argument to reverser()!" && exit 1
  fi

  # Return the reversed input to stdout (default for rev).
  rev <<< ${1}
}
```

`reverser`函数需要一个参数：要反转的输入。与往常一样，我们首先检查输入是否正确，然后再执行任何操作。接下来，我们使用`rev`来反转输入。但是，`rev`通常期望从文件或`stdin`中获取输入，而不是作为参数的变量。因为我们不想添加额外的 echo 和管道，所以我们使用这里字符串（如第十二章中所述，*在脚本中使用管道和重定向*），它允许我们直接使用变量内容作为`stdin`。由于`rev`已经将结果输出到`stdout`，所以在那一点上我们不需要提供任何东西，比如 echo。

我们告诉过您我们将证明之前的说法，这在这种情况下与前面的片段中的`$1`有关。如果函数中的`$1`与脚本的第一个参数相关，而不是函数的第一个参数，那么我们在编写`user_input`变量时添加的下划线就不会出现。对于脚本，`$1`可能等于`rain`，而对于函数，`$1`等于`_rain_`。当您运行脚本时，您肯定会看到下划线，这意味着每个函数实际上都有自己的一组参数！

将所有内容绑在一起的是脚本的最后一部分：

```
# Capture the function output via command substitution.
reversed_input=$(reverser ${user_input})

# Show the reversed input to the user.
echo "Your reversed input is: ${reversed_input}"
```

由于`reverser`函数将反转的输入发送到`stdout`，我们将使用命令替换来将其捕获到一个变量中。最后，我们打印一些澄清文本和反转的输入给用户看。结果将如下所示：

```
reader@ubuntu:~/scripts/chapter_13$ bash reverser.sh rain
Your reversed input is: _niar_
```

下划线和所有，我们得到了`rain`的反转：`_nair_`。不错！

为了避免太多复杂性，我们将这个脚本的最后部分分成两行。但是，一旦你对命令替换感到舒适，你可以省去中间变量，并直接在 echo 中使用命令替换，就像这样：`echo "Your reversed input is: $(reverser ${user_input})"`。然而，我们建议不要让它变得比这更复杂，因为那将开始影响可读性。

# 函数库

当你到达书的这一部分时，你会看到超过 50 个示例脚本。这些脚本中有许多共享组件：输入检查、错误处理和设置当前工作目录在多个脚本中都被使用过。这段代码实际上并没有真正改变；也许注释或回显略有不同，但实际上只是重复的代码。再加上在脚本顶部定义函数的问题（或者至少在开始使用它们之前），你的可维护性就开始受到影响。幸运的是，我们有一个很好的解决方案：**创建你自己的函数库！**

# 源

函数库的想法是你定义的函数在不同的脚本之间是*共享的*。这些是可重复使用的通用函数，不太关心特定脚本的工作。当你创建一个新脚本时，你会在头部之后*包含来自库的函数定义*。库只是另一个 shell 脚本：但它只用于定义函数，所以它从不调用任何东西。如果你运行它，最终结果将与运行一个空脚本的结果相同。在我们看如何包含它之前，我们将首先创建我们自己的函数库。

创建函数库时只有一个真正的考虑：放在哪里。你希望它在你的文件系统中只出现一次，最好是在一个可预测的位置。就个人而言，我们更喜欢`/opt/`目录。然而，默认情况下`/opt/`只对`root`用户可写。在多用户系统中，把它放在那里可能不是一个坏主意，由`root`拥有并被所有人可读，但由于这是一个单用户情况，我们将直接把它放在我们的主目录下。让我们从那里开始建立我们的函数库：

```
reader@ubuntu:~$ vim bash-function-library.sh 
reader@ubuntu:~$ cat bash-function-library.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-17
# Description: Bash function library.
# Usage: source ~/bash-function-library.sh
#####################################

# Check if the number of arguments supplied is exactly correct.
check_arguments() {
  # We need at least one argument.
  if [[ $# -lt 1 ]]; then
    echo "Less than 1 argument received, exiting."
    exit 1
  fi  

  # Deal with arguments
  expected_arguments=$1
  shift 1 # Removes the first argument.

  if [[ ${expected_arguments} -ne $# ]]; then
    return 1 # Return exit status 1.
  fi
}
```

因为这是一个通用函数，我们需要首先提供我们期望的参数数量，然后是实际的参数。在保存期望的参数数量后，我们使用`shift`将所有参数向左移动一个位置：`$2`变成`$1`，`$3`变成`$2`，`$1`被完全移除。这样做，只有要检查的参数数量保留下来，期望的数量安全地存储在一个变量中。然后我们比较这两个值，如果它们不相同，我们返回退出码`1`。`return`类似于`exit`，但它不会停止脚本执行：如果我们想要这样做，调用函数的脚本应该处理这个问题。

要在另一个脚本中使用这个库函数，我们需要包含它。在 Bash 中，这称为*sourcing*。使用`source`命令来实现：

```
source <file-name>
```

语法很简单。一旦你`source`一个文件，它的所有内容都将被处理。在我们的库的情况下，当我们只定义函数时，不会执行任何内容，但我们将拥有这些函数。如果你`source`一个包含实际命令的文件，比如`echo`、`cat`或`mkdir`，这些命令将被执行。就像往常一样，一个例子胜过千言万语，所以让我们看看如何使用`source`来包含库函数：

```
reader@ubuntu:~/scripts/chapter_13$ vim argument-checker.sh
reader@ubuntu:~/scripts/chapter_13$ cat argument-checker.sh
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-17
# Description: Validates the check_arguments library function
# Usage: ./argument-checker.sh
#####################################

source ~/bash-function-library.sh

check_arguments 3 "one" "two" "three" # Correct.
check_arguments 2 "one" "two" "three" # Incorrect.
check_arguments 1 "one two three" # Correct.
```

很简单对吧？我们使用完全合格的路径（是的，即使`~`是简写，这仍然是完全合格的！）来包含文件，并继续使用在其他脚本中定义的函数。如果你以调试模式运行它，你会看到函数按我们的期望工作：

```
reader@ubuntu:~/scripts/chapter_13$ bash -x argument-checker.sh 
+ source /home/reader/bash-function-library.sh
+ check_arguments 3 one two three
+ [[ 4 -lt 1 ]]
+ expected_arguments=3
+ shift 1
+ [[ 3 -ne 3 ]]
+ check_arguments 2 one two three
+ [[ 4 -lt 1 ]]
+ expected_arguments=2
+ shift 1
+ [[ 2 -ne 3 ]]
+ return 1
+ check_arguments 1 'one two three'
+ [[ 2 -lt 1 ]]
+ expected_arguments=1
+ shift 1
+ [[ 1 -ne 1 ]]
```

第一个和第三个函数调用预期是正确的，而第二个应该失败。因为我们在函数中使用了`return`而不是`exit`，所以即使第二个函数调用返回了`1`的退出状态，脚本仍会继续执行。正如调试输出所示，第二次调用函数时，执行了`2 不等于 3`的评估并成功，导致了`return 1`。对于其他调用，参数是正确的，返回了默认的`0`返回代码（输出中没有显示，但这确实发生了；如果你想自己验证，可以添加`echo $?`）。

现在，要在实际脚本中使用这个，我们需要将用户给我们的所有参数传递给我们的函数。这可以使用`$@`语法来完成：其中`$#`对应于参数的数量，`$@`简单地打印出所有参数。我们将更新`argument-checker.sh`来检查脚本的参数：

```
reader@ubuntu:~/scripts/chapter_13$ vim argument-checker.sh 
reader@ubuntu:~/scripts/chapter_13$ cat argument-checker.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
# Date: 2018-11-17
# Description: Validates the check_arguments library function
# Usage: ./argument-checker.sh <argument1> <argument2>
#####################################

source ~/bash-function-library.sh

# Check user input. 
# Use double quotes around $@ to prevent word splitting.
check_arguments 2 "$@"
echo $?
```

我们传递了预期数量的参数`2`，以及脚本接收到的所有参数`$@`给我们的函数。用一些不同的输入运行它，看看会发生什么：

```
reader@ubuntu:~/scripts/chapter_13$ bash argument-checker.sh 
1
reader@ubuntu:~/scripts/chapter_13$ bash argument-checker.sh 1
1
reader@ubuntu:~/scripts/chapter_13$ bash argument-checker.sh 1 2
0
reader@ubuntu:~/scripts/chapter_13$ bash argument-checker.sh "1 2"
1
reader@ubuntu:~/scripts/chapter_13$ bash argument-checker.sh "1 2" 3
0
```

太棒了，一切似乎都在正常工作！最有趣的尝试可能是最后两个，因为它们展示了*单词分割*经常引起的问题。默认情况下，Bash 会将每个空白字符解释为分隔符。在第四个例子中，我们传递了`"1 2"`字符串，实际上*由于引号的存在是一个单独的参数*。如果我们没有在`$@`周围使用双引号，就会发生这种情况：

```
reader@ubuntu:~/scripts/chapter_13$ tail -3 argument-checker.sh 
check_arguments 2 $@
echo $?

reader@ubuntu:~/scripts/chapter_13$ bash argument-checker.sh "1 2"
0
```

在这个例子中，Bash 将参数传递给函数时没有保留引号。函数将会接收到`"1"`和`"2"`，而不是`"1 2"`。要时刻注意这一点！

现在，我们可以使用预定义的函数来检查参数的数量是否正确。然而，目前我们并没有使用我们的返回代码做任何事情。我们将对我们的`argument-checker.sh`脚本进行最后一次调整，如果参数的数量不正确，将停止脚本执行：

```
reader@ubuntu:~/scripts/chapter_13$ vim argument-checker.sh 
reader@ubuntu:~/scripts/chapter_13$ cat argument-checker.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.2.0
# Date: 2018-11-17
# Description: Validates the check_arguments library function
# Usage: ./argument-checker.sh <argument1> <argument2>
#####################################

source ~/bash-function-library.sh

# Check user input. 
# Use double quotes around $@ to prevent word splitting.
check_arguments 2 "$@" || \
{ echo "Incorrect usage! Usage: $0 <argument1> <argument2>"; exit 1; }

# Arguments are correct, print them.
echo "Your arguments are: $1 and $2"
```

由于本书的页面宽度，我们使用`\`将`check_arguments`一行分成两行：这表示 Bash 会继续到下一行。如果你喜欢，你可以省略这一点，让整个命令在一行上。如果我们现在运行脚本，将会看到期望的脚本执行：

```
reader@ubuntu:~/scripts/chapter_13$ bash argument-checker.sh 
Incorrect usage! Usage: argument-checker.sh <argument1> <argument2>
reader@ubuntu:~/scripts/chapter_13$ bash argument-checker.sh dog cat
Your arguments are: dog and cat
reader@ubuntu:~/scripts/chapter_13$ bash argument-checker.sh dog cat mouse
Incorrect usage! Usage: argument-checker.sh <argument1> <argument2>
```

恭喜，我们已经开始创建一个函数库，并成功在我们的一个脚本中使用它！

对于`source`有一个有点令人困惑的简写语法：一个点（`.`）。如果我们想在我们的脚本中使用这个简写，只需`. ~/bash-function-library.sh`。然而，我们并不是这种语法的铁杆支持者：`source`命令既不长也不复杂，而单个`.`如果你忘记在它后面加上空格（这很难看到！）就很容易被忽略或误用。我们的建议是：如果你在某个地方遇到这个简写，请知道它的存在，但在编写脚本时使用完整的内置`source`。

# 更多实际例子

我们将在本章的最后一部分扩展您的函数库，使用来自早期脚本的常用操作。我们将从早期章节中复制一个脚本，并使用我们的函数库来替换功能，然后可以使用我们的库中的函数来处理。

# 当前工作目录

我们自己的私有函数库中第一个候选是正确设置当前工作目录。这是一个非常简单的函数，所以我们将它添加进去，不做太多解释：

```
reader@ubuntu:~/scripts/chapter_13$ vim ~/bash-function-library.sh 
reader@ubuntu:~/scripts/chapter_13$ cat ~/bash-function-library.sh 
#!/bin/bash
#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
# Date: 2018-11-17
# Description: Bash function library.
# Usage: source ~/bash-function-library.sh
#####################################
<SNIPPED>
# Set the current working directory to the script location.
set_cwd() {
  cd $(dirname $0)
}
```

因为函数库是一个潜在频繁更新的东西，正确更新头部信息非常重要。最好（并且在企业环境中最有可能）将新版本的函数库提交到版本控制系统。在头部使用正确的语义版本将帮助您保持一个干净的历史记录。特别是，如果您将其与 Chef.io、Puppet 和 Ansible 等配置管理工具结合使用，您将清楚地了解您已经更改和部署到何处。

现在，我们将使用我们的库包含和函数调用更新上一章的脚本`redirect-to-file.sh`。最终结果应该是以下内容：

```
reader@ubuntu:~/scripts/chapter_13$ cp ../chapter_12/redirect-to-file.sh library-redirect-to-file.sh
reader@ubuntu:~/scripts/chapter_13$ vim library-redirect-to-file.sh 
reader@ubuntu:~/scripts/chapter_13$ cat library-redirect-to-file.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-17
# Description: Redirect user input to file.
# Usage: ./library-redirect-to-file.sh
#####################################

# Load our Bash function library.
source ~/bash-function-library.sh

# Since we're dealing with paths, set current working directory.
set_cwd

# Capture the users' input.
read -p "Type anything you like: " user_input

# Save the users' input to a file. > for overwrite, >> for append.
echo ${user_input} >> redirect-to-file.txt
```

为了教学目的，我们已将文件复制到当前章节的目录中；通常情况下，我们只需更新原始文件。我们只添加了对函数库的包含，并用我们的`set_cwd`函数调用替换了神奇的`cd $(dirname $0)`。让我们从脚本所在的位置运行它，看看目录是否正确设置：

```
reader@ubuntu:/tmp$ bash ~/scripts/chapter_13/library-redirect-to-file.sh
Type anything you like: I like ice cream, I guess
reader@ubuntu:/tmp$ ls -l
drwx------ 3 root root 4096 Nov 17 11:20 systemd-private-af82e37c...
drwx------ 3 root root 4096 Nov 17 11:20 systemd-private-af82e37c...
reader@ubuntu:/tmp$ cd ~/scripts/chapter_13
reader@ubuntu:~/scripts/chapter_13$ ls -l
<SNIPPED>
-rw-rw-r-- 1 reader reader 567 Nov 17 19:32 library-redirect-to-file.sh
-rw-rw-r-- 1 reader reader 26 Nov 17 19:35 redirect-to-file.txt
-rw-rw-r-- 1 reader reader 933 Nov 17 15:18 reverser.sh
reader@ubuntu:~/scripts/chapter_13$ cat redirect-to-file.txt 
I like ice cream, I guess
```

因此，即使我们使用了`$0`语法（你记得的，打印脚本的完全限定路径），我们在这里看到它指的是`library-redirect-to-file.sh`的路径，而不是你可能合理假设的`bash-function-library.sh`脚本的位置。这应该证实了我们的解释，即只有函数定义被包含，当函数在运行时被调用时，它们会采用包含它们的脚本的环境。

# 类型检查

我们在许多脚本中做的事情是检查参数。我们用一个函数开始了我们的库，允许检查用户输入的参数数量。我们经常对用户输入执行的另一个操作是验证输入类型。例如，如果我们的脚本需要一个数字，我们希望用户实际输入一个数字，而不是一个单词（或一个写出来的数字，比如'eleven'）。你可能记得大致的语法，但我敢肯定，如果你现在需要它，你会浏览我们的旧脚本找到它。这不是理想的库函数候选吗？我们创建并彻底测试我们的函数一次，然后我们可以放心地只是源和使用它！让我们创建一个检查传递参数是否实际上是整数的函数：

```
reader@ubuntu:~/scripts/chapter_13$ vim ~/bash-function-library.sh
reader@ubuntu:~/scripts/chapter_13$ cat ~/bash-function-library.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.2.0
# Date: 2018-11-17
# Description: Bash function library.
# Usage: source ~/bash-function-library.sh
#####################################
<SNIPPED>

# Checks if the argument is an integer.
check_integer() {
  # Input validation.
  if [[ $# -ne 1 ]]; then
    echo "Need exactly one argument, exiting."
    exit 1 # No validation done, exit script.
  fi

  # Check if the input is an integer.
  if [[ $1 =~ ^[[:digit:]]+$ ]]; then
    return 0 # Is an integer.
  else
    return 1 # Is not an integer.
  fi
}
```

因为我们正在处理一个库函数，为了可读性，我们可以多说一点。在常规脚本中过多的冗长将降低可读性，但是一旦有人查看函数库以便理解，你可以假设他们会喜欢一些更冗长的脚本。毕竟，当我们在脚本中调用函数时，我们只会看到`check_integer ${variable}`。

接下来是函数。我们首先检查是否收到了单个参数。如果没有收到，我们退出而不是返回。为什么我们要这样做呢？调用的脚本不应该困惑于`1`的返回代码意味着什么；如果它可以意味着我们没有检查任何东西，但也意味着检查本身失败了，我们会在不希望出现歧义的地方带来歧义。所以简单地说，返回总是告诉调用者有关传递参数的信息，如果脚本调用函数错误，它将看到完整的脚本退出并显示错误消息。

接下来，我们使用在第十章中构建的正则表达式，*正则表达式*，来检查参数是否实际上是整数。如果是，我们返回`0`。如果不是，我们将进入`else`块并返回`1`。为了向阅读库的人强调这一点，我们包括了`# 是整数`和`# 不是整数`的注释。为什么不让它对他们更容易呢？记住，你并不总是为别人写代码，但如果你在一年后看自己的代码，你肯定也会觉得自己像*别人*（相信我们吧！）。

我们将从我们早期的脚本中进行另一个搜索替换。来自上一章的一个合适的脚本，`password-generator.sh`，将很好地完成这个目的。将其复制到一个新文件中，加载函数库并替换参数检查（是的，两个！）：

```
reader@ubuntu:~/scripts/chapter_13$ vim library-password-generator.sh 
reader@ubuntu:~/scripts/chapter_13$ cat library-password-generator.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-17
# Description: Generate a password.
# Usage: ./library-password-generator.sh <length>
#####################################

# Load our Bash function library.
source ~/bash-function-library.sh

# Check for the correct number of arguments.
check_arguments 1 "$@" || \
{ echo "Incorrect usage! Usage: $0 <length>"; exit 1; }

# Verify the length argument.
check_integer $1 || { echo "Argument must be an integer!"; exit 1; }

# tr grabs readable characters from input, deletes the rest.
# Input for tr comes from /dev/urandom, via input redirection.
# echo makes sure a newline is printed.
tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c $1
echo
```

我们用我们的库函数替换了参数检查和整数检查。我们还删除了变量声明，并直接在脚本的功能部分使用了`$1`；这并不总是最好的做法。然而，当输入只使用一次时，首先将其存储在命名变量中会创建一些额外开销，我们可以跳过。即使有所有的空格和注释，我们仍然通过使用函数调用将脚本行数从 31 减少到 26。当我们调用我们的新改进的脚本时，我们看到以下内容：

```
reader@ubuntu:~/scripts/chapter_13$ bash library-password-generator.sh
Incorrect usage! Usage: library-password-generator.sh <length>
reader@ubuntu:~/scripts/chapter_13$ bash library-password-generator.sh 10
50BCuB835l
reader@ubuntu:~/scripts/chapter_13$ bash library-password-generator.sh 10 20
Incorrect usage! Usage: library-password-generator.sh <length>
reader@ubuntu:~/scripts/chapter_13$ bash library-password-generator.sh bob
Argument must be an integer!
```

很好，我们的检查按预期工作。看起来也好多了，不是吗？

# 是-否检查

在完成本章之前，我们将展示另一个检查。在本书的中间，在第九章中，*错误检查和处理*，我们介绍了一个处理用户可能提供的'yes'或'no'的脚本。但是，正如我们在那里解释的那样，用户也可能使用'y'或'n'，甚至可能在其中的某个地方使用大写字母。通过秘密使用一点 Bash 扩展，你将在第十六章中得到适当解释，我们能够对用户输入进行相对清晰的检查。让我们把这个东西放到我们的库中！

```
reader@ubuntu:~/scripts/chapter_13$ vim ~/bash-function-library.sh 
reader@ubuntu:~/scripts/chapter_13$ cat ~/bash-function-library.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.3.0
# Date: 2018-11-17
# Description: Bash function library.
# Usage: source ~/bash-function-library.sh
#####################################
<SNIPPED>

# Checks if the user answered yes or no.
check_yes_no() {
  # Input validation.
  if [[ $# -ne 1 ]]; then
    echo "Need exactly one argument, exiting."
    exit 1 # No validation done, exit script.
  fi

  # Return 0 for yes, 1 for no, exit 2 for neither.
  if [[ ${1,,} = 'y' || ${1,,} = 'yes' ]]; then
    return 0
  elif [[ ${1,,} = 'n' || ${1,,} = 'no' ]]; then
    return 1
  else
    echo "Neither yes or no, exiting."
    exit 2
  fi
}
```

通过这个例子，我们为你准备了一些稍微高级的脚本。现在我们不再有二进制返回，而是有四种可能的结果：

+   函数错误调用：`exit 1`

+   函数找到了 yes：`return 0`

+   函数找到了 no：`return 1`

+   函数找不到：`exit 2`

有了我们的新库函数，我们将把`yes-no-optimized.sh`脚本和复杂逻辑替换为（几乎）单个函数调用：

```
reader@ubuntu:~/scripts/chapter_13$ cp ../chapter_09/yes-no-optimized.sh library-yes-no.sh
reader@ubuntu:~/scripts/chapter_13$ vim library-yes-no.sh
reader@ubuntu:~/scripts/chapter_13$ cat library-yes-no.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-17
# Description: Doing yes-no questions from our library.
# Usage: ./library-yes-no.sh
#####################################

# Load our Bash function library.
source ~/bash-function-library.sh

read -p "Do you like this question? " reply_variable

check_yes_no ${reply_variable} && \
echo "Great, I worked really hard on it!" || \
echo "You did not? But I worked so hard on it!"
```

花一分钟看一下前面的脚本。起初可能有点混乱，但请记住`&&`和`||`的作用。由于我们应用了一些智能排序，我们可以使用`&&`和`||`来实现我们的结果。可以这样看待它：

1.  如果`check_yes_no`返回退出状态 0（找到**yes**时），则执行`&&`后面的命令。由于它回显了成功，而`echo`的退出代码为 0，因此下一个`||`后的失败`echo`不会被执行。

1.  如果`check_yes_no`返回退出状态 1（找到**no**时），则`&&`后面的命令不会被执行。然而，它会继续执行直到达到`||`，由于返回代码仍然不是 0，它会继续到失败的回显。

1.  如果`check_yes_no`在缺少参数或缺少 yes/no 时退出，则`&&`和`||`后面的命令都不会被执行（因为脚本被给予`exit`而不是`return`，所以代码执行立即停止）。

相当聪明对吧？然而，我们必须承认，这与我们教给你的大多数关于可读性的东西有点相悖。把这看作是一个使用`&&`和`||`链接的教学练习。如果你想要自己实现是-否检查，可能最好创建专门的`check_yes()`和`check_no()`函数。无论如何，让我们看看我们改进的脚本是否像我们希望的那样工作：

```
reader@ubuntu:~/scripts/chapter_13$ bash library-yes-no.sh 
Do you like this question? Yes
Great, I worked really hard on it!
reader@ubuntu:~/scripts/chapter_13$ bash library-yes-no.sh 
Do you like this question? n
You did not? But I worked so hard on it!
reader@ubuntu:~/scripts/chapter_13$ bash library-yes-no.sh 
Do you like this question? MAYBE 
Neither yes or no, exiting.
reader@ubuntu:~/scripts/chapter_13$ bash library-yes-no.sh 
Do you like this question?
Need exactly one argument, exiting.
```

我们定义的所有场景都能正常工作。非常成功！

通常，你不希望过多地混合退出和返回代码。此外，使用返回代码传达除了通过或失败之外的任何内容也是相当不常见的。然而，由于你可以返回 256 个不同的代码（从 0 到 255），这至少在设计上是可能的。我们的是非示例是一个很好的候选，可以展示如何使用它。然而，作为一个一般的建议，最好是以通过/失败的方式使用它，因为目前你把知道不同返回代码的负担放在了调用者身上。这至少不总是一个公平的要求。

我们想以一个小练习结束本章。在本章中，在我们引入函数库之前，我们已经创建了一些函数：两个用于错误处理，一个用于彩色打印，一个用于文本反转。你的练习很简单：获取这些函数并将它们添加到你的个人函数库中。请记住以下几点：

+   这些函数是否足够详细，可以直接包含在库中，还是需要更多的内容？

+   我们可以直接调用函数并处理输出，还是最好进行编辑？

+   返回和退出是否已经正确实现，还是需要调整以作为通用库函数工作？

这里没有对错之分，只是需要考虑的事情。祝你好运！

# 总结

在本章中，我们介绍了 Bash 函数。函数是可以定义一次，然后被多次调用的通用命令链。函数是可重用的，并且可以在多个脚本之间共享。

引入了变量作用域。到目前为止，我们看到的变量始终是*全局*作用域：它们对整个脚本都是可用的。然而，引入函数后，我们遇到了*局部*作用域的变量。这些变量只能在函数内部访问，并且用`local`关键字标记。

我们了解到函数可以有自己独立的参数集，可以在调用函数时作为参数传递。我们证明了这些参数实际上与传递给脚本的全局参数不同（当然，除非所有参数都通过函数传递）。我们举了一个例子，关于如何使用`stdout`从函数返回输出，我们可以通过将函数调用封装在命令替换中来捕获它。

在本章的下半部分，我们把注意力转向创建一个函数库：一个独立的脚本，没有实际命令，可以被包含（通过`source`命令）在另一个脚本中。一旦库在另一个脚本中被引用，库中定义的所有函数就可以被脚本使用。我们在本章的剩余部分展示了如何做到这一点，同时用一些实用的实用函数扩展了我们的函数库。

我们以一个练习结束了本章，以确保本章中定义的所有函数都包含在他们自己的个人函数库中。

本章介绍了以下命令：`top`，`free`，`declare`，`case`，`rev`和`return`。

# 问题

1.  我们可以用哪两种方式定义一个函数？

1.  函数的一些优点是什么？

1.  全局作用域变量和局部作用域变量之间有什么区别？

1.  我们如何给变量设置值和属性？

1.  函数如何使用传递给它的参数？

1.  我们如何从函数中返回一个值？

1.  `source`命令是做什么的？

1.  为什么我们想要创建一个函数库？

# 进一步阅读

+   **Linux 性能监控**：[`linoxide.com/monitoring-2/linux-performance-monitoring-tools/`](https://linoxide.com/monitoring-2/linux-performance-monitoring-tools/)

+   **AWK 基础教程**：[`mistonline.in/wp/awk-basic-tutorial-with-examples/`](https://mistonline.in/wp/awk-basic-tutorial-with-examples/)

+   **高级 Bash 变量**：[`www.thegeekstuff.com/2010/05/bash-variables/`](https://www.thegeekstuff.com/2010/05/bash-variables/)

+   **获取来源**: [`bash.cyberciti.biz/guide/Source_command`](https://bash.cyberciti.biz/guide/Source_command)
