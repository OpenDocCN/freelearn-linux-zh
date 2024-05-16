# 第一章。外壳某事

在本章中，我们将涵盖：

+   在终端中打印

+   玩转变量和环境变量

+   使用 shell 进行数学计算

+   玩转文件描述符和重定向

+   数组和关联数组

+   访问别名

+   获取有关终端的信息

+   获取、设置日期和延迟

+   调试脚本

+   函数和参数

+   在变量中读取一系列命令的输出

+   在不按回车键的情况下读取“n”个字符

+   字段分隔符和迭代器

+   比较和测试

# 介绍

类 UNIX 系统是令人惊叹的操作系统设计。即使经过了许多年，类 UNIX 架构的操作系统仍然是最佳设计之一。这种架构的最重要特性之一是命令行界面或 shell。shell 环境帮助用户与操作系统的核心功能进行交互和访问。在这种情况下，术语脚本更相关。脚本通常由解释器支持的编程语言支持。Shell 脚本是我们编写需要执行的一系列命令的文件。脚本文件是使用 shell 实用程序执行的。

在本书中，我们处理的是 Bash（Bourne Again Shell），这是大多数 GNU/Linux 系统的默认 shell 环境。由于 GNU/Linux 是基于类 UNIX 架构的最突出的操作系统，大多数示例和讨论都是以 Linux 系统为基础编写的。

本章的主要目的是让读者了解 shell 环境，并熟悉围绕 shell 的基本功能。命令是在 shell 终端中输入和执行的。在打开终端时，会出现提示符。它通常是以下格式：

```
username@hostname$

```

或者：

```
root@hostname#

```

或者简单地为**$**或**#**。

`$`表示普通用户，`#`表示管理员用户 root。Root 是 Linux 系统中权限最高的用户。

Shell 脚本是一个文本文件，通常以 shebang 开头，如下所示：

```
#!/bin/bash
```

对于 Linux 环境中的任何脚本语言，脚本都以特殊行 shebang 开头。shebang 是一个前缀为`#!`的解释器路径的行。`/bin/bash`是 Bash 的解释器命令路径。

脚本的执行可以通过两种方式进行。我们可以将脚本作为`sh`的命令行参数运行，或者以执行权限运行自行可执行文件。

可以将脚本作为文件名的命令行参数运行，如下所示：

```
$ sh script.sh # Assuming script is in the current directory.

```

或者：

```
$ sh /home/path/script.sh # Using full path of script.sh.

```

如果脚本作为`sh`的命令行参数运行，则脚本中的 shebang 无效。

为了自行执行 shell 脚本，需要可执行权限。在作为自行可执行文件运行时，它使用 shebang。它使用 shebang 中附加到`#!`的解释器路径运行脚本。可以设置脚本的执行权限如下：

```
$ chmod a+x script.sh

```

此命令为`script.sh`文件赋予所有用户的可执行权限。脚本可以执行如下：

```
$ ./script.sh #./ represents the current directory

```

或：

```
$ /home/path/script.sh # Full path of the script is used

```

shell 程序将读取第一行，并查看 shebang 是`#!/bin/bash`。它将识别`/bin/bash`并在内部执行脚本：

```
$ /bin/bash script.sh

```

当打开终端时，它最初会执行一组命令来定义各种设置，如提示文本、颜色等。这组命令（运行命令）是从一个名为`.bashrc`的 shell 脚本中读取的，该脚本位于用户的主目录（`~/.bashrc`）中。bash shell 还会维护用户运行的命令历史记录。它位于文件`~/.bash_history`中。`~`是用户主目录路径的缩写。

在 Bash 中，每个命令或命令序列都是使用分号或换行符分隔的。例如：

```
$ cmd1 ; cmd2

```

这相当于： 

```
$ cmd1
$ cmd2

```

最后，`#`字符用于表示未处理注释的开始。注释部分以`#`开头，并一直延续到该行的末尾。注释行通常用于提供有关文件中代码的注释，或者阻止执行代码行。

现在让我们继续学习本章的基本配方。

# 在终端中打印

终端是用户与 shell 环境交互的交互式实用程序。在终端中打印文本是大多数 shell 脚本和实用程序需要定期执行的基本任务。可以通过各种方法和不同格式进行打印。

## 如何做...

`echo`是在终端中打印的基本命令。

`echo`默认情况下在每次调用结束时都会换行：

```
$ echo "Welcome to Bash"
Welcome to Bash

```

只需使用带有`echo`命令的双引号文本即可在终端中打印文本。同样，没有双引号的文本也会产生相同的输出：

```
$ echo Welcome to Bash
Welcome to Bash

```

另一种执行相同任务的方法是使用单引号：

```
$ echo 'text in quote'

```

这些方法可能看起来相似，但其中一些具有特定目的和副作用。考虑以下命令：

```
$ echo "cannot include exclamation - ! within double quotes"

```

这将返回以下内容：

```
bash: !: event not found error

```

因此，如果要打印`!`，不要在双引号内使用，或者可以使用特殊的转义字符（`\`）对`!`进行转义。

```
$ echo Hello world !

```

或者：

```
$ echo 'Hello world !'

```

或者：

```
$ echo "Hello world \!" #Escape character \ prefixed.

```

当使用双引号与`echo`一起使用时，你应该在发出`echo`之前添加`set +H`，这样你就可以使用`!`。

每种方法的副作用如下：

+   当不使用引号与`echo`一起使用时，我们不能使用分号，因为它在 bash shell 中充当命令之间的分隔符。

+   `echo hello;hello`将`echo hello`作为一个命令，第二个`hello`作为第二个命令。

+   当使用单引号与`echo`一起使用时，引号内的变量（例如，`$var`）不会被 Bash 展开，而是会按原样显示。

这意味着：

`$ echo '$var'`将返回`$var`

然而

如果定义了`$var`，`$ echo $var`将返回变量`$var`的值，如果未定义，则返回空。

在终端中打印的另一个命令是`printf`命令。`printf`使用与 C 编程语言中的`printf`命令相同的参数。例如：

```
$ printf "Hello world"

```

`printf`接受带引号的文本或由空格分隔的参数。我们可以使用带有`printf`的格式化字符串。我们可以指定字符串宽度，左对齐或右对齐等。默认情况下，`printf`不像`echo`命令那样具有换行符。我们必须在需要时指定换行符，如下面的脚本所示：

```
#!/bin/bash 
#Filename: printf.sh

printf  "%-5s %-10s %-4s\n" No Name  Mark 
printf  "%-5s %-10s %-4.2f\n" 1 Sarath 80.3456 
printf  "%-5s %-10s %-4.2f\n" 2 James 90.9989 
printf  "%-5s %-10s %-4.2f\n" 3 Jeff 77.564
```

我们将收到格式化的输出：

```
No    Name       Mark
1     Sarath     80.35
2     James      91.00
3     Jeff       77.56

```

`%s`、`%c`、`%d`和`%f`是格式替换字符，可以在引号格式字符串之后放置参数。

`%-5s`可以描述为具有左对齐的字符串替换（`-`表示左对齐），宽度等于`5`。如果未指定`-`，则字符串将对齐到右侧。宽度指定了为该变量保留的字符数。对于`Name`，保留的宽度为`10`。因此，任何名称都将位于为其保留的 10 个字符宽度内，其余字符将填充到总共 10 个字符。

对于浮点数，我们可以传递额外的参数来四舍五入小数位。

对于标记，我们已将字符串格式化为`%-4.2f`，其中`.2`指定四舍五入到小数点后两位。请注意，对于格式字符串的每一行，都会发出一个`\n`换行符。

## 还有更多...

应始终注意，`echo`和`printf`的标志（例如-e、-n 等）应出现在命令中的任何字符串之前，否则 Bash 将把标志视为另一个字符串。

### 在`echo`中转义换行符

默认情况下，`echo`在其输出文本的末尾附加换行符。可以通过使用`-n`标志来避免这种情况。`echo`也可以接受双引号字符串中的转义序列作为参数。要使用转义序列，请使用`echo`作为`echo -e "包含转义序列的字符串"`。例如：

```
echo -e "1\t2\t3"
123

```

### 打印彩色输出

在终端上生成彩色输出非常有趣。我们使用转义序列生成彩色输出。

颜色代码用于表示每种颜色。例如，reset=0，black=30，red=31，green=32，yellow=33，blue=34，magenta=35，cyan=36，white=37。

为了打印彩色文本，输入以下内容：

```
echo -e "\e[1;31m This is red text \e[0m"

```

这里`\e[1;31`是将颜色设置为红色的转义字符串，`\e[0m`将颜色重置。将`31`替换为所需的颜色代码。

对于彩色背景，reset = 0，black = 40，red = 41，green = 42，yellow = 43，blue = 44，magenta = 45，cyan = 46，white=47，是常用的颜色代码。

为了打印彩色背景，输入以下内容：

```
echo -e "\e[1;42m Green Background \e[0m"

```

# 玩弄变量和环境变量

变量是每种编程语言的基本组成部分，用于保存不同的数据。脚本语言通常不需要在使用之前声明变量类型。它可以直接分配。在 Bash 中，每个变量的值都是字符串。如果我们使用引号或不使用引号分配变量，它们将被存储为字符串。Shell 环境和操作系统环境使用特殊变量存储特殊值，这些变量称为环境变量。

让我们看看配方。

## 做好准备

变量使用通常的命名构造命名。当应用程序执行时，它将被传递一组称为环境变量的变量。从终端，要查看与该终端进程相关的所有环境变量，发出`env`命令。对于每个进程，可以通过其运行时查看环境变量：

```
cat /proc/$PID/environ

```

将`PID`设置为相关进程的进程 ID（PID 始终是整数）。

例如，假设正在运行一个名为 gedit 的应用程序。我们可以使用`pgrep`命令获取 gedit 的进程 ID，如下所示：

```
$ pgrep gedit
12501

```

您可以通过执行以下命令获取与进程关联的环境变量：

```
$ cat /proc/12501/environ
GDM_KEYBOARD_LAYOUT=usGNOME_KEYRING_PID=1560USER=slynuxHOME=/home/slynux

```

请注意，为了方便起见，许多环境变量被剥离。实际输出可能包含大量变量。

上述命令返回环境变量及其值的列表。每个变量表示为`name=value`对，并用空字符(`\0`)分隔。如果可以用`\n`替换`\0`字符，可以重新格式化输出，以在每行显示每个`variable=value`对。可以使用`tr`命令进行替换，如下所示：

```
$ cat /proc/12501/environ  | tr '\0' '\n'

```

现在，让我们看看如何分配和操作变量和环境变量。

## 如何做到…

变量可以按以下方式分配：

```
var=value
```

`var`是变量的名称，value 是要分配的值。如果`value`不包含任何空格字符（如空格），则无需用引号括起来，否则必须用单引号或双引号括起来。

请注意，`var = value`和`var=value`是不同的。常见错误是写`var =value`而不是`var=value`。后者是赋值操作，而前者是相等操作。

使用以下方式通过在变量名称前加上`$`来打印变量的内容：

```
var="value" #Assignment of value to variable var.

echo $var
```

或：

```
echo ${var}
```

输出如下：

```
value

```

我们可以在双引号中使用`printf`或`echo`中的变量值。

```
#!/bin/bash
#Filename :variables.sh
fruit=apple
count=5
echo "We have $count ${fruit}(s)"
```

输出如下：

```
We have 5 apple(s)

```

环境变量是未在当前进程中定义的变量，而是从父进程接收的变量。例如，`HTTP_PROXY`是一个环境变量。此变量定义了应该为 Internet 连接使用哪个代理服务器。

通常，它被设置为：

```
HTTP_PROXY=http://192.168.0.2:3128
export HTTP_PROXY

```

使用 export 命令设置`env`变量。现在，从当前 shell 脚本执行的任何应用程序都将接收到这个变量。我们可以为我们自己的目的在执行的应用程序或 shell 脚本中导出自定义变量。默认情况下，shell 提供了许多标准环境变量。

例如，`PATH`。典型的`PATH`变量将包含：

```
$ echo $PATH
/home/slynux/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games

```

当给出一个执行命令时，shell 会自动在`PATH`环境变量的目录列表中搜索可执行文件（目录路径由“:”字符分隔）。通常，`$PATH`在`/etc/environment`或`/etc/profile`或`~/.bashrc`中定义。当我们需要向`PATH`环境添加新路径时，我们使用：

```
export PATH="$PATH:/home/user/bin"

```

或者，我们也可以使用：

```
$ PATH="$PATH:/home/user/bin"
$ export PATH

$ echo $PATH
/home/slynux/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/home/user/bin

```

在这里，我们已经将`/home/user/bin`添加到`PATH`。

一些知名的环境变量是：`HOME`、`PWD`、`USER`、`UID`、`SHELL`等。

## 还有更多...

让我们看看与常规和环境变量相关的一些其他提示。

### 查找字符串的长度

获取变量值的长度如下：

```
length=${#var}

```

例如：

```
$ var=12345678901234567890
$ echo ${#var} 
20

```

`length`是字符串中的字符数。

### 识别当前的 shell

显示当前使用的 shell 如下：

```
echo $SHELL

```

或者，您也可以使用：

```
echo $0

```

例如：

```
$ echo $SHELL
/bin/bash

$ echo $0
bash

```

### 检查超级用户

UID 是一个重要的环境变量，可以用来检查当前脚本是否以 root 用户或常规用户身份运行。例如：

```
if [ $UID -ne 0 ]; then
echo Non root user. Please run as root.
else
echo "Root user"
fi
```

根用户的 UID 为 0。

### 修改 Bash 提示字符串（用户名@主机名:~$）

当我们打开终端或运行 shell 时，我们会看到一个提示字符串，如`user@hostname: /home/$`。不同的 GNU/Linux 发行版有稍微不同的提示和不同的颜色。我们可以使用`PS1`环境变量自定义提示文本。shell 的默认提示文本是在`~/.bashrc`文件中设置的。

+   我们可以列出用于设置`PS1`变量的行如下：

```
$ cat ~/.bashrc | grep PS1
PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '

```

+   为了设置自定义提示字符串，输入：

```
slynux@localhost: ~$ PS1="PROMPT>"
PROMPT> Type commands here # Prompt string changed.

```

+   我们可以使用特殊的转义序列如`\e[1;31`（参考本章的*在终端中打印*）来使用彩色文本。

还有一些特殊字符可以扩展到系统参数。例如，`\u`扩展到用户名，`\h`扩展到主机名，`\w`扩展到当前工作目录。

# 使用 shell 进行数学计算

算术运算是每种编程语言的基本要求。Bash shell 具有各种进行算术运算的方法。

## 准备就绪

Bash shell 环境可以使用`let`、`(( ))`和`[]`命令执行基本算术运算。两个实用程序`expr`和`bc`在执行高级操作时也非常有帮助。

## 如何做到...

可以将数值分配为常规变量赋值，它以字符串形式存储。但是，我们使用方法来操作数字。

```
#!/bin/bash
no1=4;
no2=5;
```

`let`命令可以直接执行基本操作。

在使用`let`时，我们使用不带`$`前缀的变量名，例如：

```
let result=no1+no2
echo $result
```

+   递增操作：

```
$ let no1++

```

+   递减操作：

```
$ let no1--

```

+   缩写：

```
let no+=6
let no-=6

```

这些分别等同于`let no=no+6`和`let no=no-6`。

+   备用方法：

`[]`操作符可以像`let`命令一样使用，如下所示：

```
result=$[ no1 + no2 ]
```

在`[]`操作符内部使用`$`前缀是合法的，例如：

```
result=$[ $no1 + 5 ]
```

`(( ))`也可以使用。当使用`(( ))`运算符时，变量名前面加上`$`，如下所示：

```
result=$(( no1 + 50 ))
```

`expr`也可以用于基本操作：

```
result=`expr 3 + 4`
result=$(expr $no1 + 5)
```

所有上述方法都不支持浮点数，只能操作整数。

`bc`精度计算器是一个用于数学运算的高级实用程序。它具有广泛的选项。我们可以执行浮点运算并使用高级函数，如下所示：

```
echo "4 * 0.56" | bc
2.24

no=54; 
result=`echo "$no * 1.5" | bc`
echo $result
81.0
```

可以通过`stdin`通过分号作为分隔符将附加参数传递给`bc`。

+   **指定十进制精度（scale）：**在下面的示例中，`scale=2`参数设置小数位数为`2`。因此，`bc`的输出将包含两位小数：

```
echo "scale=2;3/8" | bc
0.37

```

+   **使用 bc 进行基数转换：**我们可以从一个基数系统转换为另一个基数系统。让我们从十进制转换为二进制，再从二进制转换为八进制：

```
#!/bin/bash
Description: Number conversion

no=100
echo "obase=2;$no" | bc
1100100
no=1100100
echo "obase=10;ibase=2;$no" | bc
100
```

+   计算平方和平方根可以按如下方式进行：

```
echo "sqrt(100)" | bc #Square root
echo "10¹⁰" | bc #Square
```

# 玩转文件描述符和重定向

文件描述符是与文件输入和输出相关联的整数。它们跟踪已打开的文件。最常见的文件描述符是 `stdin`、`stdout` 和 `stderr`。我们可以将一个文件描述符的内容重定向到另一个文件描述符。以下示例将演示如何使用文件描述符进行操作和重定向。

## 准备工作

在编写脚本时，我们经常使用标准输入（`stdin`）、标准输出（`stdout`）和标准错误（`stderr`）。通过过滤内容将输出重定向到文件是我们需要执行的基本操作之一。当一个命令输出一些文本时，它可以是错误或输出（非错误）消息。我们无法仅仅通过查看文本来区分它是输出文本还是错误文本。但是，我们可以使用文件描述符来处理它们。我们可以提取附加到特定描述符的文本。

文件描述符是与打开的文件或数据流相关联的整数。文件描述符 0、1 和 2 分别保留如下：

+   0 – `stdin`（标准输入）

+   1 – `stdout`（标准输出）

+   2 – `stderr`（标准错误）

## 如何做到...

将输出文本重定向或保存到文件可以按以下方式完成：

```
$ echo "This is a sample text 1" > temp.txt

```

这将通过截断文件将回显文本存储在 `temp.txt` 中，在写入之前将清空文件内容。

接下来，考虑以下示例：

```
$ echo "This is sample text 2" >> temp.txt

```

这将把文本追加到文件中。

`>` 和 `>>` 运算符是不同的。它们两个都将文本重定向到文件，但第一个会清空文件然后写入，而后一个会将输出添加到现有文件的末尾。

查看文件内容如下：

```
$ cat temp.txt
This is sample text 1
This is sample text 2

```

当我们使用重定向运算符时，它不会在终端上打印，而是被重定向到文件。使用重定向运算符时，默认情况下会采用标准输出。为了显式地采用特定的文件描述符，必须在运算符前加上描述符号码。

`>` 相当于 `1>`，类似地，`>>` 也适用于 `1>>`。

让我们看看标准错误是什么，以及你如何重定向它。当命令输出错误消息时，`stderr` 消息会被打印。考虑以下示例：

```
$ ls +
ls: cannot access +: No such file or directory

```

这里 `+` 是一个无效的参数，因此会返回一个错误。

### 提示

**成功和不成功的命令**

当一个命令返回错误后，它会返回一个非零的退出状态。当成功完成后终止时，命令返回零。退出状态可以从特殊变量 `$?` 中读取（在执行命令后立即运行 `echo $?` 语句以打印退出状态）。

以下命令将 `stderr` 文本打印到屏幕而不是文件中：

```
$ ls + > out.txt 
ls: cannot access +: No such file or directory 

```

然而，在以下命令中，`stdout` 输出为空，因此会生成一个空文件 `out.txt`：

```
$ ls + 2> out.txt # works

```

你可以将 `stderr` 独占地重定向到一个文件，将 `stdout` 重定向到另一个文件，如下所示：

```
$ cmd 2>stderr.txt 1>stdout.txt

```

也可以通过将 `stderr` 转换为 `stdout` 的方式将 `stderr` 和 `stdout` 重定向到单个文件：

```
$ cmd 2>&1 output.txt

```

或者另一种方法：

```
$ cmd &> output.txt 

```

有时输出可能包含不必要的信息（如调试消息）。如果你不希望输出终端负担着 `stderr` 的细节，那么你应该将 `stderr` 输出重定向到 `/dev/null`，这样可以完全删除它。例如，假设我们有三个文件 a1、a2 和 a3。然而，a1 对用户没有读写执行权限。当你需要打印以 `a` 开头的文件的内容时，可以使用 `cat` 命令。

设置测试文件如下：

```
$ echo a1 > a1 
$ cp a1 a2 ; cp a2 a3;
$ chmod 000 a1  #Deny all permissions

```

使用通配符（`a*`）显示文件内容时，对于文件 `a1` 会显示错误消息，因为它没有适当的读取权限：

```
$ cat a*
cat: a1: Permission denied
a1
a1

```

这里的 `cat: a1: Permission denied` 属于 `stderr` 数据。我们可以将 `stderr` 数据重定向到文件，而 `stdout` 仍然会在终端上打印。考虑以下代码：

```
$ cat a* 2> err.txt #stderr is redirected to err.txt
a1
a1

$ cat err.txt
cat: a1: Permission denied

```

看一下以下代码：

```
$ some_command 2> /dev/null

```

在这种情况下，`stderr`输出被转储到`/dev/null`文件。`/dev/null`是一个特殊的设备文件，其中接收到的任何数据都被丢弃。空设备通常被称为位桶或黑洞。

当对`stderr`或`stdout`执行重定向时，重定向的文本流入文件。由于文本已经被重定向并且已经进入文件，没有文本剩下通过管道（`|`）流向下一个命令，它会出现在下一个命令序列的`stdin`中。

然而，有一种巧妙的方法可以将数据重定向到文件，并为下一组命令提供重定向数据的副本作为`stdin`。这可以使用`tee`命令来实现。例如，要在终端上打印`stdout`并将`stdout`重定向到文件，`tee`的语法如下：

`command | tee FILE1 FILE2`

在以下代码中，`tee`命令接收`stdin`数据。它将`stdout`的副本写入文件`out.txt`，并将另一个副本作为`stdin`发送给下一个命令。`cat –n`命令为从`stdin`接收的每一行添加行号，并将其写入`stdout`：

```
$ cat a* | tee out.txt | cat -n
cat: a1: Permission denied
 1a1
 2a1

```

如下所示检查`out.txt`的内容：

```
$ cat out.txt
a1
a1

```

请注意，`cat: a1: Permission denied`不会出现，因为它属于`stdin`。`tee`只能从`stdin`读取。

默认情况下，`tee`命令会覆盖文件，但可以通过提供`-a`选项来使用追加选项，例如：

```
$ cat a* | tee –a out.txt | cat –n.

```

命令以以下格式出现：`command FILE1 FILE2…`或者简单地`command FILE`。

我们可以使用`stdin`作为命令参数。可以通过使用`-`作为命令的文件名参数来实现：

```
$ cmd1 | cmd2 | cmd -

```

例如：

```
$ echo who is this | tee -
who is this
who is this

```

或者，我们可以使用`/dev/stdin`作为输出文件名来使用`stdin`。

类似地，使用`/dev/stderr`作为标准错误和`/dev/stdout`作为标准输出。这些是对应于`stdin`、`stderr`和`stdout`的特殊设备文件。

## 还有更多...

读取`stdin`输入的命令可以以多种方式接收数据。还可以使用`cat`和管道指定我们自己的文件描述符，例如：

```
$ cat file | cmd
$ cmd1 | cmd2

```

### 从文件重定向到命令

通过重定向，我们可以从文件中读取数据作为`stdin`，如下所示：

```
$ cmd < file

```

### 从脚本中包含的文本块进行重定向

有时我们需要将一块文本（多行文本）重定向为标准输入。考虑一个特殊情况，源文本放在 shell 脚本中。一个实际的用例是写入日志文件头数据。可以按照以下步骤执行：

```
#!/bin/bash
cat <<EOF>log.txt
LOG FILE HEADER
This is a test log file
Function: System statistics
EOF

```

出现在`cat <<EOF >log.txt`和下一个`EOF`行之间的行将作为`stdin`数据出现。如下所示打印`log.txt`的内容：

```
$ cat log.txt
LOG FILE HEADER
This is a test log file
Function: System statistics

```

### 自定义文件描述符

文件描述符是访问文件的抽象指示器。每个文件访问都与一个称为文件描述符的特殊数字相关联。0、1 和 2 是`stdin`、`stdout`和`stderr`的保留描述符号。

我们可以使用`exec`命令创建自定义文件描述符。如果您已经熟悉其他编程语言的文件编程，您可能已经注意到打开文件的模式。通常使用三种模式：

+   读取模式

+   使用截断模式写入

+   使用追加模式写入

`<`是一个用于从文件读取到`stdin`的操作符。`>`是用于写入文件并截断的操作符（数据被写入目标文件后截断内容）。`>>`是用于追加写入文件的操作符（数据被追加到现有文件内容中，目标文件的内容不会丢失）。文件描述符可以用三种模式之一创建。

创建用于读取文件的文件描述符，如下所示：

```
$ exec 3<input.txt # open for reading with descriptor number 3

```

我们可以按以下方式使用它：

```
$ echo this is a test line > input.txt
$ exec 3<input.txt

```

现在可以使用文件描述符`3`与命令一起使用。例如，`cat <&3`如下所示：

```
$ cat <&3
this is a test line

```

如果需要进行第二次读取，则不能重用文件描述符`3`。需要使用`exec`重新分配文件描述符`3`进行读取，以进行第二次读取。

创建一个用于写入的文件描述符（截断模式）如下：

```
$ exec 4>output.txt # open for writing

```

例如：

```
$ exec 4>output.txt
$ echo newline >&4
$ cat output.txt
newline

```

创建一个用于写入的文件描述符（追加模式）如下：

```
$ exec 5>>input.txt

```

例如：

```
$ exec 5>>input.txt
$ echo appended line >&5
$ cat input.txt 
newline
appended line

```

# 数组和关联数组

数组是存储使用索引作为单独实体的数据集合的非常重要的组件。

## 准备工作

Bash 支持常规数组和关联数组。常规数组是只能使用整数作为数组索引的数组。但是，关联数组是可以将字符串作为其数组索引的数组。

关联数组在许多类型的操作中非常有用。关联数组支持是在 Bash 的 4.0 版本中引入的。因此，旧版本的 Bash 将不支持关联数组。

## 如何做...

数组可以以许多方式定义。可以按照一行中的值列表定义数组，如下所示：

```
array_var=(1 2 3 4 5 6)
#Values will be stored in consecutive locations starting from index 0.
```

或者，可以按如下方式定义数组为一组索引-值对：

```
array_var[0]="test1"
array_var[1]="test2"
array_var[2]="test3"
array_var[3]="test4"
array_var[4]="test5"
array_var[5]="test6"
```

使用以下方法在给定索引处打印数组的内容：

```
$ echo ${array_var[0]}
test1
index=5
$ echo ${array_var[$index]}
test6

```

使用以下方法将数组中的所有值打印为列表：

```
$ echo ${array_var[*]}
test1 test2 test3 test4 test5 test6

```

或者，您可以使用：

```
$ echo ${array_var[@]}
test1 test2 test3 test4 test5 test6

```

按如下方式打印数组的长度（数组中的元素数）：

```
$ echo ${#array_var[*]}
6
```

## 还有更多...

从 4.0 版本开始，Bash 引入了关联数组。它们是使用哈希技术解决许多问题的有用实体。让我们深入了解一下。

### 定义关联数组

在关联数组中，我们可以使用任何文本数据作为数组索引。但是，普通数组只能使用整数作为数组索引。

首先需要声明语句来将变量名声明为关联数组。声明可以如下进行：

```
$ declare -A ass_array

```

声明后，可以使用以下两种方法向关联数组添加元素：

1.  通过使用内联索引-值列表方法，我们可以提供索引-值对的列表：

```
$ ass_array=([index1]=val1 [index2]=val2)

```

1.  或者，您可以使用单独的索引-值赋值：

```
$ ass_array[index1]=val1
$ ass_array[index2]=val2

```

例如，考虑使用关联数组为水果分配价格：

```
$ declare -A fruits_value
$ fruits_value=([apple]='100dollars' [orange]='150 dollars')

```

按如下方式显示数组的内容：

```
$ echo "Apple costs ${fruits_value[apple]}"
Apple costs 100 dollars

```

### 数组索引列表

数组具有用于索引每个元素的索引。普通数组和关联数组在索引类型方面有所不同。我们可以按如下方式获取数组中的索引列表：

```
$ echo ${!array_var[*]}

```

或者，我们也可以使用：

```
$ echo ${!array_var[@]}

```

在先前的`fruits_value`数组示例中，考虑以下内容：

```
$ echo ${!fruits_value[*]}
orange apple

```

这也适用于普通数组。

# 访问别名

别名基本上是一个快捷方式，可以代替输入一长串命令。

## 准备工作

别名可以通过多种方式实现，可以使用函数或使用`alias`命令。

## 如何做...

别名可以按如下方式实现：

```
$ alias new_command='command sequence'

```

为`install`命令提供快捷方式`apt-get install`，可以按如下方式完成：

```
$ alias install='sudo apt-get install'

```

因此，我们可以使用`install pidgin`代替`sudo apt-get install pidgin`。

`alias`命令是临时的；别名仅在关闭当前终端之前存在。为了使这些快捷方式永久存在，将此语句添加到`~/.bashrc`文件中。在新的 shell 进程生成时，`~/.bashrc`中的命令总是被执行。

```
$ echo 'alias cmd="command seq"' >> ~/.bashrc

```

要删除别名，从`~/.bashrc`中删除其条目或使用`unalias`命令。另一种方法是定义一个具有新命令名称的函数，并将其写入`~/.bashrc`。

我们可以将`rm`别名为删除原始文件并将其保留在备份目录中：

```
alias rm='cp $@ ~/backup; rm $@'

```

当创建别名时，如果要别名化的项目已经存在，则它将被新别名的命令替换为该用户的命令。

## 还有更多...

有时别名也可能会造成安全漏洞。看看如何识别它们：

### 转义别名

`alias`命令可用于为任何重要命令设置别名，并且您可能并不总是希望使用别名运行命令。我们可以通过转义要运行的命令来忽略当前定义的任何别名。例如：

```
$ \command
```

`\`字符转义命令，使其在没有任何别名更改的情况下运行。在不受信任的环境中运行特权命令时，通过在命令前加上`\`来忽略别名始终是一个良好的安全实践。攻击者可能已经使用自己的自定义命令将特权命令别名化，以窃取用户提供给命令的关键信息。

# 获取有关终端的信息

在编写命令行 shell 脚本时，我们经常需要大量操作有关当前终端的信息，例如列数、行数、光标位置、掩码密码字段等。本教程将帮助您了解收集和操作终端设置。

## 准备好

`tput`和`stty`是可用于终端操作的实用程序。让我们看看如何使用它们执行不同的任务。

## 如何做...

获取终端中的列数和行数如下：

```
tput cols
tput lines
```

为了打印当前的终端名称，使用：

```
tput longname

```

要将光标移动到位置 100,100，可以输入：

```
tput cup 100 100

```

将终端的背景颜色设置如下：

```
tput setb no

```

`no`可以是 0 到 7 范围内的值。

将文本的前景颜色设置如下：

```
tput setf no

```

`no`可以是 0 到 7 范围内的值。

为了使文本加粗使用：

```
tput bold

```

通过使用以下方式开始和结束下划线：

```
tput smul
tput rmul

```

为了从光标删除到行尾，请使用：

```
tput ed

```

在输入密码时，我们不应该显示已输入的字符。在以下示例中，我们将看到如何使用`stty`来实现：

```
#!/bin/sh
#Filename: password.sh
echo -e "Enter password: "
stty -echo
read password
stty echo
echo
echo Password read.
```

上面的`-echo`选项禁用了对终端的输出，而`echo`则启用了输出。

# 获取、设置日期和延迟

许多应用程序需要以不同的格式打印日期，设置日期和时间，并根据日期和时间执行操作。延迟通常用于在程序执行期间提供等待时间（例如，1 秒）。脚本上下文，例如每五秒执行一次监视任务，需要理解如何在程序中编写延迟。本教程将向您展示如何处理日期和时间延迟。

## 准备好

日期可以以各种格式打印。我们还可以从命令行设置日期。在类 UNIX 系统中，日期以自 1970-01-01 00:00:00 UTC 以来的秒数存储为整数。这称为纪元或 UNIX 时间。让我们看看如何读取日期并设置日期。

## 如何做...

您可以按如下方式读取日期：

```
$ date
Thu May 20 23:09:04 IST 2010

```

纪元时间可以打印如下：

```
$ date +%s
1290047248

```

纪元被定义为自 1970 年 1 月 1 日协调世界时（UTC）午夜以来经过的秒数，不包括闰秒。纪元时间在需要计算两个日期或时间之间的差异时非常有用。您可以找出两个给定时间戳的纪元时间，并计算纪元值之间的差异。因此，您可以找出两个日期之间的总秒数。

我们可以从给定的格式化日期字符串中找出纪元。您可以使用多种日期格式作为输入。通常，如果您从系统日志或任何标准应用程序生成的输出中收集日期，您不需要担心使用的日期字符串格式。您可以按如下方式将日期字符串转换为纪元：

```
$ date --date "Thu Nov 18 08:07:21 IST 2010" +%s
1290047841

```

`--date`选项用于提供日期字符串作为输入。但是，我们可以使用任何日期格式选项来打印输出。从字符串中提取输入日期可以用于找出给定日期的工作日。

例如：

```
$ date --date "Jan 20 2001" +%A
Saturday

```

日期格式字符串列在以下表中：

| 日期组件 | 格式 |
| --- | --- |
| 工作日 | `%a`（例如：Sat）`%A`（例如：Saturday） |
| 月 | `%b`（例如：Nov）`%B`（例如：November） |
| 天 | `%d`（例如：31） |
| 格式中的日期（mm/dd/yy） | `%D`（例如：10/18/10） |
| 年 | `%y`（例如：10）`%Y`（例如：2010） |
| 小时 | `%I`或`%H`（例如：08） |
| 分钟 | `%M`（例如：33） |
| 秒 | `%S`（例如：10） |
| 纳秒 | `%N`（例如：695208515） |
| 以秒为单位的 UNIX 时间戳 | `%s`（例如：1290049486） |

使用以`+`为前缀的格式字符串的组合作为`date`命令的参数，以打印所需格式的日期。例如：

```
$ date "+%d %B %Y"
20 May 2010

```

我们可以按如下方式设置日期和时间：

```
# date -s "Formatted date string"

```

例如：

```
# date -s "21 June 2009 11:01:22"

```

有时我们需要检查一组命令所花费的时间。我们可以按如下方式显示它：

```
#!/bin/bash
#Filename: time_take.sh
start=$(date +%s)
commands;
statements;

end=$(date +%s)
difference=$(( end - start))
echo Time taken to execute commands is $difference seconds.
```

另一种方法是使用`timescriptpath`来获取执行脚本所需的时间。

## 还有更多...

在编写循环执行的监控脚本时，生成时间间隔是必不可少的。让我们看看如何生成时间延迟。

### 在脚本中产生延迟

为了在脚本中延迟执行一段时间，使用`sleep:$ sleep no_of_seconds`。

例如，以下脚本使用`tput`和`sleep`从 0 计数到 40：

```
#!/bin/bash
#Filename: sleep.sh
echo -n Count:
tput sc

count=0;
while true;
do
if [ $x -lt 40 ];
then let count++;
sleep 1;
tput rc
tput ed
echo -n $count;
else exit 0;
fi
done
```

在上面的示例中，一个变量 count 被初始化为 0，并在每次循环执行时递增。`echo`语句打印文本。我们使用`tput sc`来存储光标位置。在每次循环执行时，我们通过恢复数字的光标位置在终端中写入新的计数。使用`tput rc`来恢复光标位置。`tput ed`清除当前光标位置到行尾的文本，以便清除旧数字并写入计数。使用`sleep`命令在循环中提供 1 秒的延迟。

# 调试脚本

调试是每种编程语言都应该实现的关键功能之一，以在发生意外情况时产生回溯信息。调试信息可用于阅读和理解导致程序崩溃或以意外方式行事的原因。Bash 提供了一些调试选项，每个系统管理员都应该知道。还有一些其他巧妙的调试方法。

## 准备工作

调试 shell 脚本不需要特殊的实用程序。Bash 带有一些标志，可以打印脚本接受的参数和输入。让我们看看如何做。

## 如何做...

添加`-x`选项以启用 shell 脚本的调试跟踪，如下所示：

```
$ bash -x script.sh

```

使用`-x`标志运行脚本将打印每个源代码行的当前状态。请注意，您还可以使用`sh –x script`。

`-x`标志会将脚本的每一行在执行时输出到`stdout`。但是，我们可能只需要观察源代码的某些部分，以便在某些部分打印命令和参数。在这种情况下，我们可以使用`set 内置`在脚本中启用和禁用调试打印。

+   `set -x`：在执行时显示参数和命令

+   `set +x`：禁用调试

+   `set –v`：在读取输入时显示输入

+   `set +v`：禁止打印输入

例如：

```
#!/bin/bash
#Filename: debug.sh
for i in {1..6}
do
set -x
echo $i
set +x
done
echo "Script executed"
```

在上述脚本中，`echo $i`的调试信息只会在该部分使用`-x`和`+x`限制调试时打印。

上述调试方法由 bash 内置提供。但它们总是以固定格式产生调试信息。在许多情况下，我们需要以自己的格式获得调试信息。我们可以通过传递`_DEBUG`环境变量来设置这样的调试样式。

看看以下示例代码：

```
#!/bin/bash
function DEBUG()
{
[ "$_DEBUG" == "on" ] && $@ || :
}

for i in {1..10}
do
DEBUG echo $i
done
```

我们可以按如下方式运行上述脚本，调试设置为“on”：

```
$ _DEBUG=on ./script.sh

```

我们在每个需要打印调试信息的语句前加上`DEBUG`前缀。如果未将`_DEBUG=on`传递给脚本，则不会打印调试信息。在 Bash 中，命令`:`告诉 shell 什么都不做。

## 还有更多...

我们还可以使用其他方便的方法来调试脚本。我们可以巧妙地利用 shebang 来调试脚本。

### Shebang hack

shebang 可以从`#!/bin/bash`更改为`#!/bin/bash –xv`，以启用调试而无需任何额外的标志（`-xv`标志本身）。

# 函数和参数

与其他脚本语言一样，Bash 也支持函数。让我们看看如何定义和使用函数。

## 如何做...

可以定义函数如下：

```
function fname()
{
statements;
}
```

或者，

```
fname()
{
statements;
}
```

函数可以通过使用其名称来调用：

```
$ fname ; # executes function

```

参数可以传递给函数，并且可以被我们的脚本访问：

```
fname arg1 arg2 ; # passing args

```

以下是函数`fname`的定义。在`fname`函数中，我们已经包含了各种访问函数参数的方式。

```
fname()
{
  echo $1, $2; #Accessing arg1 and arg2
  echo "$@"; # Printing all arguments as list at once
  echo "$*"; # Similar to $@, but arguments taken as single entity
  return 0; # Return value
}
```

类似地，参数可以传递给脚本，并且可以通过`script:$0`（脚本的名称）访问：

+   `$1`是第一个参数

+   `$2`是第二个参数

+   `$n`是第 n 个参数

+   `"$@"`扩展为`"$1" "$2" "$3"`等等

+   `"$*"`扩展为`"$1c$2c$3"`，其中`c`是 IFS 的第一个字符

+   `"$@"`是最常用的。`"$*"`很少使用，因为它将所有参数作为单个字符串。

## 还有更多...

让我们探索更多有关 Bash 函数的提示。

### 递归函数

Bash 中的函数也支持递归（可以调用自身的函数）。例如，`F() { echo $1; F hello; sleep 1; }`。

### 提示

**分叉炸弹**

:(){ :|:& };:

这个递归函数是一个调用自身的函数。它无限地生成进程，并最终导致拒绝服务攻击。在函数调用后加上`&`将子进程放入后台。这是一个危险的代码，因为它会分叉进程，因此被称为分叉炸弹。

你可能会发现难以解释上面的代码。请参阅维基百科页面[`en.wikipedia.org/wiki/Fork_bomb`](http://en.wikipedia.org/wiki/Fork_bomb)了解更多关于分叉炸弹的细节和解释。

可以通过限制从配置文件`/etc/security/limits.conf`生成的最大进程数来防止它。

### 导出函数

可以使用`export`导出函数，就像导出环境变量一样，这样函数的范围可以扩展到子进程，如下所示：

```
export -f fname

```

### 读取命令返回值（状态）

我们可以通过以下方式获取命令或函数的返回值：

```
cmd; 
echo $?;

```

`$?`将给出命令`cmd`的返回值。

返回值称为退出状态。它可以用来分析命令是否成功执行完成。如果命令成功退出，退出状态将为零，否则将为非零。

我们可以通过以下方式检查命令是否成功终止：

```
#!/bin/bash
#Filename: success_test.sh
CMD="command" #Substitute with command for which you need to test exit status
$CMD
if [ $? –eq 0 ];
then
echo "$CMD executed successfully"
else
echo "$CMD terminated unsuccessfully"
fi
```

### 将参数传递给命令

命令的参数可以以不同的格式传递。假设`-p`和`-v`是可用的选项，`-k NO`是另一个需要一个数字的选项。此外，命令需要一个文件名作为参数。可以按以下方式执行：

```
$ command -p -v -k 1 file
```

或者：

```
$ command -pv -k 1 file
```

或者：

```
$ command -vpk 1 file
```

或者：

```
$ command file -pvk 1
```

# 读取一系列命令的输出

Shell 脚本设计的最佳特性之一是轻松组合多个命令或实用程序以产生输出。一个命令的输出可以作为另一个命令的输入，后者将其输出传递给另一个命令，依此类推。这种组合的输出可以在一个变量中读取。本教程说明了如何组合多个命令以及如何读取其输出。

## 准备就绪

通常通过`stdin`或参数将输入提供给命令。输出显示为`stderr`或`stdout`。当我们组合多个命令时，我们通常使用`stdin`提供输入和`stdout`提供输出。

命令被称为过滤器。我们使用管道连接每个过滤器。管道操作符是"`|`"。例如：

```
$ cmd1 | cmd2 | cmd3 

```

这里我们结合了三个命令。`cmd1`的输出传递给`cmd2`，`cmd2`的输出传递给`cmd3`，最终输出（来自`cmd3`）将被打印或可以被重定向到文件。

## 如何做到...

看一下以下代码：

```
$ ls | cat -n > out.txt

```

这里`ls`的输出（当前目录的列表）被传递给`cat -n`。`cat –n`为通过`stdin`接收的输入添加行号。因此，它的输出被重定向到`out.txt`文件。

我们可以读取由管道组合的一系列命令的输出，如下所示：

```
cmd_output=$(COMMANDS)
```

这被称为子 shell 方法。例如：

```
cmd_output=$(ls | cat -n)
echo $cmd_output

```

另一种方法，称为反引号，也可以用来存储命令的输出，如下所示：

```
cmd_output=`COMMANDS`
```

例如：

```
cmd_output=`ls | cat -n`
echo $cmd_output

```

反引号与单引号字符不同。它是键盘上*~*按钮上的字符。

## 还有更多...

有多种组合命令的方法。让我们看一些。

### 使用子 shell 生成一个单独的进程

子 shell 是独立的进程。可以使用`( )`操作符定义子 shell，如下所示：

```
pwd;
(cd /bin; ls);
pwd;
```

当在子 shell 中执行某些命令时，当前 shell 中不会发生任何更改；更改仅限于子 shell。例如，在子 shell 中使用`cd`命令更改当前目录时，目录更改不会反映在主 shell 环境中。

`pwd`命令打印工作目录的路径。

`cd`命令将当前目录更改为给定的目录路径。

### 使用子 shell 引用以保留间距和换行字符

假设我们正在使用子 shell 或反引号方法将命令的输出读取到一个变量中，我们总是用双引号引起来以保留间距和换行字符（\n）。例如：

```
$ cat text.txt
1
2
3

$ out=$(cat text.txt)
$ echo $out
1 2 3 # Lost \n spacing in 1,2,3 

$ out="$(cat tex.txt)"
$ echo $out
1
2
3

```

# 读取“n”个字符而不按回车键

`read`是一个重要的 Bash 命令，可用于从键盘或标准输入读取文本。我们可以使用`read`来交互式地从用户那里读取输入，但`read`还可以做更多。让我们看一个新的示例来说明`read`命令提供的一些最重要的选项。

## 准备工作

任何编程语言中的大多数输入库都是从键盘读取输入；但是当按下*Return*时，字符串输入终止。在某些关键情况下，无法按下*Return*，但终止是基于字符数或单个字符完成的。例如，在游戏中，按下上+时，球会向上移动。每次按下`+`然后按下*Return*来确认+按下是不高效的。`read`命令提供了一种在不必按下*Return*的情况下完成此任务的方法。

## 如何做...

以下语句将从输入中读取“n”个字符到变量`variable_name`中：

`read -n number_of_chars variable_name`

例如：

```
$ read -n 2 var
$ echo $var

```

`read`还有许多其他选项。让我们看看这些。

按照以下方式以非回显模式读取密码：

```
read -s var
```

使用`read`显示消息：

```
read -p "Enter input:"  var
```

按照以下方式在超时后读取输入：

```
read -t timeout var
```

例如：

```
$ read -t 2 var
#Read the string that is typed within 2 seconds into variable var.

```

使用分隔符字符来结束输入行，如下所示：

```
read -d delim_charvar
```

例如：

```
$ read -d ":" var
hello:#var is set to hello

```

# 字段分隔符和迭代器

内部字段分隔符是 shell 脚本中的一个重要概念。在操作文本数据时非常有用。我们现在将讨论分隔符，它们将不同的数据元素从单个数据流中分隔开。内部字段分隔符是一个特殊目的的分隔符。**内部字段分隔符**（**IFS**）是一个存储分隔字符的环境变量。它是运行 shell 环境使用的默认分隔符字符串。

考虑需要在字符串或**逗号分隔值**（**CSV**）中迭代单词的情况。在第一种情况下，我们将使用`IFS=" "`，在第二种情况下，`IFS=","`。让我们看看如何做。

## 准备工作

考虑 CSV 数据的情况：

```
data="name,sex,rollno,location"
#To read each of the item in a variable, we can use IFS.
oldIFS=$IFS
IFS=, now,
for item in $data;
do
echo Item: $item
done

IFS=$oldIFS
```

输出如下：

```
Item: name
Item: sex
Item: rollno
Item: location

```

IFS 的默认值是一个空格组件（换行符、制表符或空格字符）。

当 IFS 设置为“,”时，shell 将逗号解释为分隔符字符，因此在迭代期间，`$item`变量将以逗号分隔的子字符串作为其值。

如果 IFS 未设置为“`,`”，那么它将打印整个数据作为单个字符串。

## 如何做...

让我们通过考虑`/etc/passwd`文件来看一下 IFS 的另一个示例用法。在`/etc/passwd`文件中，每一行包含由`":"`分隔的项目。文件中的每一行对应于与用户相关的属性。

考虑输入：`root:x:0:0:root:/root:/bin/bash`。每行的最后一个条目指定用户的默认 shell。为了打印用户及其默认 shell，我们可以使用 IFS hack，如下所示：

```
#!/bin/bash
#Description: Illustration of IFS
line="root:x:0:0:root:/root:/bin/bash" 
oldIFS=$IFS;
IFS=":"
count=0
for item in $line;
do

[ $count -eq 0 ]  && user=$item;
[ $count -eq 6 ]  && shell=$item;
let count++
done;
IFS=$oldIFS
echo $user\'s shell is $shell;
```

输出将是：

```
root's shell is /bin/bash

```

循环在迭代一系列值时非常有用。Bash 提供了许多类型的循环。让我们看看如何使用它们。

**For 循环**：

```
for var in list;
do
commands; # use $var
done
list can be a string, or a sequence.
```

我们可以轻松生成不同的序列。

`echo {1..50}`可以生成从 1 到 50 的数字列表

`echo {a..z}`或`{A..Z}`或我们可以使用`{a..h}`生成部分列表。类似地，通过组合这些，我们可以连接数据。

在以下代码中，每次迭代，变量`i`将保存范围为`a`到`z`的字符：

```
for i in {a..z}; do actions; done;
```

`for`循环也可以采用 C 语言中的`for`循环格式。例如：

```
for((i=0;i<10;i++))
{
commands; # Use $i
}
```

**While 循环**：

```
while condition
do
commands;
done
```

对于无限循环，使用`true`作为条件。

**Until 循环**：

Bash 中还有一个称为`until`的特殊循环。这将执行循环，直到给定条件成为真。例如：

```
x=0;
until [ $x -eq 9 ]; # [ $x -eq 9 ] is the condition
do let x++; echo $x;
done
```

# 比较和测试

程序中的流程控制由比较和测试语句处理。Bash 还提供了几种选项来执行与 UNIX 系统级特性兼容的测试。

## 准备工作

我们可以使用`if`，`if else`和逻辑运算符来执行测试，并使用某些比较运算符来比较数据项。还有一个称为`test`的命令可用于执行测试。让我们看看如何使用这些命令。

## 如何做...

如果条件：

```
if condition;
then
commands;
fi
```

else if 和 else：

```
if condition; 
then
commands;
elif condition; 
then
    commands
else
    commands
fi
```

嵌套也是可能的，使用 if 和 else。`if`条件可能很长。我们可以使用逻辑运算符使它们更短，如下所示：

```
[ condition ] && action; # action executes if condition is true.
[ condition ] || action; # action executes if condition is false.

```

`&&`是逻辑 AND 操作，`||`是逻辑 OR 操作。在编写 Bash 脚本时，这是一个非常有用的技巧。现在让我们进入条件和比较操作。

**数学比较**：

通常，条件被括在方括号`[]`中。请注意`[`或`]`和操作数之间有一个空格。如果没有提供空格，将会显示错误。示例如下：

```
[ $var -eq 0 ] or [ $var -eq 0 ]
```

可以按照以下方式执行变量或值的数学条件：

```
[ $var -eq 0 ]  # It returns true when $var equal to 0.
[ $var -ne 0 ] # It returns true when $var not equals 0
```

其他重要的操作符包括：

+   `-gt`：大于

+   `-lt`：小于

+   `-ge`：大于或等于

+   `-le`：小于或等于

多个测试条件可以组合如下：

```
[ $var1 -ne 0 -a $var2 -gt 2 ]  # using AND -a
[ $var -ne 0 -o var2 -gt 2 ] # OR -o
```

**与文件系统相关的测试**：

我们可以使用不同的条件标志来测试不同的文件系统相关属性，如下所示：

+   `[ -f $file_var ]`: 如果给定的变量保存了常规文件路径或文件名，则返回 true。

+   `[ -x $var ]`: 如果给定的变量保存了可执行文件路径或文件名，则返回 true。

+   `[ -d $var ]`: 如果给定的变量保存了目录路径或目录名称，则返回 true。

+   `[ -e $var ]`: 如果给定的变量保存了一个现有文件，则返回 true。

+   `[ -c $var ]`: 如果给定的变量保存了字符设备文件的路径，则返回 true。

+   `[ -b $var ]`: 如果给定的变量保存了块设备文件的路径，则返回 true。

+   `[ -w $var ]`: 如果给定的变量保存了可写文件的路径，则返回 true。

+   `[ -r $var ]`: 如果给定的变量保存了可读文件的路径，则返回 true。

+   `[ -L $var ]`: 如果给定的变量保存了符号链接的路径，则返回 true。

使用示例如下：

```
fpath="/etc/passwd"
if [ -e $fpath ]; then
echo File exists; 
else
echo Does not exist; 
fi
```

**字符串比较**：

在使用字符串比较时，最好使用双方括号，因为使用单方括号有时会导致错误。有时使用单方括号会导致错误。因此最好避免使用它们。

可以比较两个字符串是否相同，如下所示；

+   `[[ $str1 = $str2 ]]`: 当 str1 等于 str2 时返回 true，即 str1 和 str2 的文本内容相同

+   `[[ $str1 == $str2 ]]`: 这是字符串相等检查的替代方法

我们可以检查两个字符串是否不相同，如下所示：

+   `[[ $str1 != $str2 ]]`: 当 str1 和 str2 不匹配时返回 true

我们可以找出字母顺序较小或较大的字符串，如下所示：

+   `[[ $str1 > $str2 ]]`: 当 str1 按字母顺序大于 str2 时返回 true

+   `[[ $str1 < $str2 ]]`: 当 str1 按字母顺序小于 str2 时返回 true

### 注意

请注意，在`=`之后和之前都提供了一个空格。如果不提供空格，它不是一个比较，而是一个赋值语句。

+   `[[ -z $str1 ]]`：如果 str1 包含空字符串，则返回 true

+   `[[ -n $str1 ]]`：如果 str1 包含非空字符串，则返回 true

使用逻辑运算符`&&`和`||`可以更容易地组合多个条件，如下所示：

```
if [[ -n $str1 ]] && [[ -z $str2 ]] ;
then
commands;
fi
```

例如：

```
str1="Not empty "
str2=""
if [[ -n $str1 ]] && [[ -z $str2 ]];
then
echo str1 is non-empty and str2 is empty string.
fi
```

输出如下：

```
str1 is non-empty and str2 is empty string.

```

`test`命令可用于执行条件检查。它有助于避免使用许多大括号。可以在`test`命令中使用相同的一组条件，这些条件被包含在`[]`中。

例如：

```
if  [ $var -eq 0 ]; then echo "True"; fi
can be written as
if  test $var -eq 0 ; then echo "True"; fi
```
