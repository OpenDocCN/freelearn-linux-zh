# 第二章。拥有一个好的命令

在本章中，我们将涵盖：

+   使用 cat 进行连接

+   记录和回放终端会话

+   查找文件和文件列表

+   将命令输出作为命令的参数（xargs）

+   使用 tr 进行翻译

+   校验和和验证

+   排序、唯一和重复

+   临时文件命名和随机数

+   拆分文件和数据

+   基于扩展名切分文件名

+   使用 rename 和 mv 批量重命名文件

+   拼写检查和字典操作

+   自动化交互输入

# 介绍

命令是 UNIX-like 系统中美丽的组件。它们帮助我们完成许多任务，使我们的工作更加轻松。当你在任何地方练习使用命令时，你会喜欢它。许多情况会让你说“哇！”一旦你有机会尝试 Linux 提供的一些命令，使你的生活更轻松和更高效，你会想知道在没有使用它们之前你是如何做的。我个人最喜欢的一些命令是`grep`、`awk`、`sed`和`find`。

使用 UNIX/Linux 命令行是一门艺术。随着练习和经验的积累，你会变得更擅长使用它。本章将向您介绍一些最有趣和有用的命令。

# 使用 cat 进行连接

`cat`是命令行战士必须学习的第一批命令之一。`cat`是一个美丽而简单的命令。它通常用于读取、显示或连接文件的内容，但`cat`不仅仅是这样。

## 准备工作

当我们需要使用单行命令将标准输入数据与文件数据合并时，我们会感到困惑。将`stdin`数据与文件数据合并的常规方法是将`stdin`重定向到文件，然后追加两个文件。但是我们可以使用`cat`命令在单个调用中轻松完成。

## 如何做到…

`cat`命令是一个非常简单的命令，在日常生活中经常使用。`cat`代表连接。

读取文件内容的`cat`的一般语法是：

```
$ cat file1 file2 file3 ...
```

该命令输出作为命令行参数提供的文件的连接数据。例如：

```
$ cat file.txt
This is a line inside file.txt
This is the second line inside file.txt

```

## 它是如何工作的…

有很多功能与`cat`一起提供。让我们一起走过几种可能的`cat`使用技巧。

`cat`命令不仅可以从文件中读取和连接数据，还可以从标准输入中读取输入。

为了从标准输入中读取，使用管道操作符如下：

`某些命令的输出 | cat`

同样，我们可以使用`cat`将输入文件的内容与标准输入一起连接起来。将`stdin`和另一个文件的数据组合起来，如下所示：

```
$ echo 'Text through stdin' | cat – file.txt

```

在这段代码中，`-`充当`stdin`文本的文件名。

## 还有更多…

`cat`命令还有其他一些查看文件的选项。让我们一起来看看。

### 压缩空白行

有时文本中的许多空行需要压缩成一个以便阅读或其他目的。使用以下语法在文本文件中压缩相邻的空行：

```
$ cat -s file

```

例如：

```
$ cat multi_blanks.txt
line 1

line2

line3

line4

$ cat -s multi_blanks.txt # Squeeze adjacent blank lines
line 1

line2

line3

line4

```

或者，我们可以使用`tr`来删除所有空行，如下所示：

```
$ cat multi_blanks.txt | tr -s '\n'
line 1
line2
line3
line4

```

在上面的`tr`使用中，它将相邻的'`\n'`字符压缩为单个'`\n'`（换行符）。

### 显示制表符为^I

很难区分制表符和重复的空格字符。在编写像 Python 这样的语言的程序时，制表符和空格对缩进目的具有特殊意义。它们被不同对待。因此，使用制表符而不是空格会导致缩进问题。通过查看文本编辑器，可能很难跟踪制表符或空格的错误放置位置。`cat`有一个可以突出显示制表符的功能。这在调试缩进错误时非常有帮助。使用`cat`的`-T`选项来突出显示制表符字符为^I。例如：

```
$ cat file.py
def function():
 var = 5
 next = 6
 third = 7

$ cat -T file.py
def function():
^Ivar = 5
 next = 6
^Ithird = 7^I

```

### 行号

对`cat`命令使用`-n`标志将输出每一行的行号前缀。需要注意的是，`cat`命令永远不会更改文件；相反，它根据提供的选项对输入进行修改，并在`stdout`上产生输出。例如：

```
$ cat lines.txt
line 
line
line

$ cat -n lines.txt
 1	line
 2	line
 3	line

```

# 记录和重播终端会话

当您需要向某人展示如何在终端中执行某项操作，或者需要准备关于如何通过命令行执行某项操作的教程时，通常会手动输入命令并展示给他们。或者您可以录制屏幕录像并向他们播放视频。如果我们可以记录之前输入的命令的顺序和时间，并重新播放这些命令，以便其他人可以观看并仿佛他们正在输入呢？命令的输出将显示在终端上，直到播放完成。听起来有趣吗？可以使用`script`和`scriptreplay`命令来实现。

## 准备工作

`script`和`scriptreplay`命令在大多数 GNU/Linux 发行版中都可用。将终端会话记录到文件中会很有趣。您可以通过记录终端会话来创建命令行技巧和窍门的教程，以实现某些任务。您还可以分享记录的文件供他人回放，并了解如何使用命令行执行特定任务。

## 如何做…

我们可以开始记录终端会话，如下所示：

```
$ script -t 2> timing.log -a output.session
type commands;
…
..
exit

```

两个配置文件作为参数传递给`script`命令。一个文件用于存储每个命令运行的时间信息（`timing.log`），而另一个文件（`output.session`）用于存储命令输出。使用`-t`标志将时间数据转储到`stderr`。使用`2>`将`stderr`重定向到`timing.log`。

通过使用两个文件，`timing.log`（存储时间信息）和`output.session`（存储命令输出信息），我们可以按以下方式重播命令执行的顺序：

```
$ scriptreplay timing.log output.session
# Plays the sequence of commands and output

```

## 它是如何工作的…

通常，我们会录制桌面视频来准备教程。但是，视频需要大量的存储空间。但是终端脚本文件只是一个文本文件。因此，它的文件大小通常只有几千字节。

您可以与任何想要在他们的终端中重播终端会话的人分享文件`timing.log`和`output.session`。

`script`命令也可以用于设置可以广播给多个用户的终端会话。这是一种非常有趣的体验。让我们看看如何做。

打开两个终端，Terminal1 和 Terminal2。

1.  在 Terminal1 中输入以下命令：

```
$ mkfifo scriptfifo

```

1.  在 Terminal2 中输入以下命令：

```
$ cat scriptfifo

```

1.  返回到 Terminal1 并输入以下命令：

```
$ script -f scriptfifo
$ commands;

```

当您需要结束会话时，请输入`exit`并按*Return*。它将显示消息“脚本完成，文件为 scriptfifo”。

现在 Terminal1 是广播者，Terminal2 是接收者。

当您在 Terminal1 上实时输入任何内容时，它将在 Terminal2 或任何提供以下命令的终端上播放：

```
cat scriptfifo

```

在计算机实验室或互联网上处理多个用户的教程会话时，可以使用此方法。这将节省带宽，并提供实时体验。

# 查找文件和文件列表

`find`是 UNIX/Linux 命令行工具箱中的伟大实用程序之一。它是 shell 脚本的非常有用的命令，但由于缺乏理解，大多数人并不有效地使用它。本教程涵盖了`find`的大多数用例以及如何使用它来解决不同标准的问题。

## 准备工作

`find`命令使用以下策略：`find`通过文件层次结构进行匹配，匹配符合指定条件的文件，并执行一些操作。让我们看看`find`的不同用例和基本用法。

## 如何做…

为了列出当前目录到下降子目录中的所有文件和文件夹，请使用以下语法：

```
$ find base_path
```

`base_path`可以是`find`应该开始下降的任何位置（例如，`/home/slynux/`）。

此命令的一个示例如下：

```
$ find . -print
# Print lists of files and folders

```

`.`指定当前目录，`..`指定父目录。这个约定贯穿整个 UNIX 文件系统。

`-print`参数指定打印匹配文件的名称（路径）。当使用`-print`时，`'\n'`将是分隔每个文件的定界字符。

`-print0`参数指定每个匹配文件名都用定界字符`'\0'`打印。当文件名包含空格字符时，这是有用的。

## 还有更多...

在这个示例中，我们学习了最常用的`find`命令的用法。`find`命令是一个强大的命令行工具，它配备了各种有趣的选项。让我们来看看`find`命令的一些不同选项。

### 基于文件名或正则表达式匹配的搜索

`-name`参数指定文件名的匹配字符串。我们可以将通配符作为其参数文本传递。`*.txt`匹配所有以`.txt`结尾的文件名并将它们打印出来。`-print`选项在终端中打印与给定为`find`命令选项的条件（例如，`-name`）匹配的文件名或文件路径。

```
$ find /home/slynux -name "*.txt" –print

```

`find`命令有一个`–iname`（忽略大小写）选项，它类似于`-name`。`–iname`匹配名称时忽略大小写。

例如：

```
$ ls
example.txt  EXAMPLE.txt  file.txt
$ find . -iname "example*" -print
./example.txt
./EXAMPLE.txt

```

如果我们想匹配多个条件中的任何一个，我们可以使用 OR 条件，如下所示：

```
$ ls
new.txt  some.jpg  text.pdf
$ find . \( -name "*.txt" -o -name "*.pdf" \) -print
./text.pdf
./new.txt

```

上面的代码将打印所有`.txt`和`.pdf`文件，因为`find`命令匹配`.txt`和`.pdf`文件。`\(`和`\)`用于将`-name "*.txt" -o -name "*.pdf"`视为单个单元。

`-path`参数可用于匹配与通配符匹配的文件路径。`-name`始终使用给定的文件名进行匹配。但是，`-path`匹配整个文件路径。例如：

```
$ find /home/users -path "*slynux*" -print
This will match files as following paths.
/home/users/list/slynux.txt
/home/users/slynux/eg.css

```

`-regex`参数类似于`-path`，但`-regex`根据正则表达式匹配文件路径。

正则表达式是通配符匹配的高级形式。它使我们能够指定带有模式的文本。通过使用这些模式，我们可以匹配文本并将其打印出来。使用正则表达式进行文本匹配的典型示例是：从给定的文本池中解析所有电子邮件地址。电子邮件地址采用`name@host.root`的形式。因此，它可以概括为`[a-z0-9]+@[a-z0-9]+.[a-z0-9]+`。`+`表示前一个字符类可以在后面的字符中重复一次或多次。

以下命令匹配`.py`或`.sh`文件：

```
$ ls
new.PY  next.jpg  test.py
$ find . -regex ".*\(\.py\|\.sh\)$"
./test.py

```

类似地，使用`-iregex`忽略了可用的正则表达式的大小写。例如：

```
$ find . -iregex ".*\(\.py\|\.sh\)$"
./test.py
./new.PY

```

### 否定参数

`find`还可以使用“!”来否定参数。例如：

```
$ find . ! -name "*.txt" -print

```

上述`find`构造匹配所有文件名，只要名称不以`.txt`结尾。以下示例显示了该命令的结果：

```
$ ls
list.txt  new.PY  new.txt  next.jpg  test.py

$ find . ! -name "*.txt" -print
.
./next.jpg
./test.py
./new.PY

```

### 基于目录深度的搜索

当使用`find`命令时，它会递归地遍历所有子目录，直到尽可能地达到子目录树的叶子。我们可以通过给`find`命令一些深度参数来限制`find`命令遍历的深度。`-maxdepth`和`-mindepth`是这些参数。

在大多数情况下，我们只需要在当前目录中搜索。它不应该进一步进入当前目录的子目录。在这种情况下，我们可以通过深度参数限制`find`命令应该下降的深度。为了限制`find`不进入当前目录的子目录，深度可以设置为 1。当我们需要下降到两个级别时，深度设置为 2，依此类推。

为了指定最大深度，我们使用`-maxdepth`级别参数。同样，我们也可以指定下降开始的最小级别。如果我们想要从第二级开始搜索，可以使用`-mindepth`级别参数设置最小深度。通过使用以下命令，将`find`命令限制为最大深度为 1：

```
$ find . -maxdepth 1 -type f -print

```

此命令仅列出当前目录中的所有普通文件。如果有子目录，则不会打印或遍历它们。同样，`-maxdepth 2`最多遍历两个下降级别的子目录。

`-mindepth`类似于`-maxdepth`，但它设置了`find`遍历的最小深度级别。它可以用于查找和打印位于基本路径最小深度级别的文件。例如，要打印出距离当前目录至少两个子目录的所有文件，请使用以下命令：

```
$ find . -mindepth 2 -type f -print
./dir1/dir2/file1
./dir3/dir4/f2

```

即使当前目录或`dir1`和`dir3`中有文件，也不会被打印出来。

### 注意

`-maxdepth`和`-mindepth`应该作为`find`的第三个参数进行指定。如果它们作为第四个或更多的参数进行指定，可能会影响`find`的效率，因为它必须进行不必要的检查（例如，如果`-maxdepth`被指定为第四个参数，`-type`被指定为第三个参数，`find`命令首先找出所有具有指定`-type`的文件，然后找出所有匹配的文件具有指定的深度。然而，如果深度被指定为第三个参数，`-type`被指定为第四个参数，`find`可以收集所有具有最多指定深度的文件，然后检查文件类型，这是搜索的最有效方式。

### 基于文件类型进行搜索

类 UNIX 操作系统将每个对象都视为文件。有不同类型的文件，如普通文件、目录、字符设备、块设备、符号链接、硬链接、套接字、FIFO 等。

文件搜索可以使用`-type`选项进行过滤。通过使用`-type`，我们可以指定`find`命令只匹配具有指定类型的文件。

只列出包括后代的目录如下：

```
$ find . -type d -print

```

很难分别列出目录和文件。但`find`可以帮助做到。只列出普通文件如下：

```
$ find . -type f -print

```

只列出符号链接如下：

```
$ find . -type l -print

```

您可以使用以下表中的`type`参数来正确匹配所需的文件类型：

| 文件类型 | 类型参数 |
| --- | --- |
| 普通文件 | `f` |
| 符号链接 | `l` |
| 目录 | `d` |
| 字符特殊设备 | `c` |
| 块设备 | `b` |
| 套接字 | `s` |
| FIFO | `p` |

### 搜索文件时间

UNIX/Linux 文件系统的每个文件都有三种类型的时间戳。它们如下：

+   **访问时间**（-`atime`）：这是文件最后一次被某个用户访问的时间戳

+   **修改时间**（-`mtime`）：这是文件内容最后一次修改的时间戳

+   **更改时间**（-`ctime`）：这是文件的元数据（如权限或所有权）最后一次修改的时间戳

UNIX 中没有所谓的创建时间。

`-atime`、`-mtime`、`-ctime`是`find`中可用的时间参数选项。它们可以用"天数"的整数值来指定。这些整数值通常附加有`-`或`+`符号。`-`符号表示小于，而`+`表示大于。例如：

+   打印出在最近 7 天内访问过的所有文件如下：

```
$ find . -type f -atime -7 -print

```

+   打印出所有访问时间正好是 7 天前的文件如下：

```
$ find . -type f -atime 7 -print

```

+   打印出所有访问时间早于 7 天的文件如下：

```
$ find . -type f -atime +7 -print

```

同样，我们可以使用`-mtime`参数来搜索基于修改时间的文件，使用`-ctime`来搜索基于更改时间的文件。

`-atime`，`-mtime`和`-ctime`是基于时间的参数，使用以天为单位的时间度量。还有一些其他基于时间的参数，使用以分钟为单位的时间度量。这些如下：

+   `-amin`（访问时间）

+   `-mmin`（修改时间）

+   `-cmin`（更改时间）

例如：

为了打印所有访问时间早于七分钟的文件，请使用以下命令：

```
$ find . -type f -amin +7 -print

```

`find`的另一个很好的功能是`-newer`参数。通过使用`-newer`，我们可以指定一个参考文件来与时间戳进行比较。我们可以使用`-newer`参数找到所有比指定文件更新（修改时间更早）的文件。

例如，找出所有修改时间大于给定`file.txt`文件的修改时间的文件，如下所示：

```
$ find . -type f -newer file.txt -print

```

`find`命令的时间戳操作标志对于编写系统备份和维护脚本非常有用。

### 基于文件大小进行搜索

根据文件大小进行搜索，可以执行如下搜索：

```
$ find . -type f -size +2k
# Files having size greater than 2 kilobytes

$ find . -type f -size -2k
# Files having size less than 2 kilobytes

$ find . -type f -size 2k
# Files having size 2 kilobytes

```

我们可以使用不同的大小单位，而不是`k`，如下所示：

+   `b` – 512 字节块

+   `c` – 字节

+   `w` – 两个字节的字

+   `k` – 千字节

+   `M` – 兆字节

+   `G` – 千兆字节

### 基于文件匹配进行删除

`-delete`标志可用于删除由`find`匹配的文件。

从当前目录中删除所有`.swp`文件，方法如下：

```
$ find . -type f -name "*.swp" -delete

```

### 基于文件权限和所有权进行匹配

可以根据文件权限匹配文件。我们可以列出具有指定文件权限的文件，如下所示：

```
$ find . -type f -perm 644 -print
# Print files having permission 644

```

作为示例用例，我们可以考虑 Apache Web 服务器的情况。 Web 服务器中的 PHP 文件需要适当的权限才能执行。我们可以找出没有适当执行权限的 PHP 文件，如下所示：

```
$ find . –type f –name "*.php" ! -perm 644 –print

```

我们还可以根据文件的所有权搜索文件。使用`-user USER`选项可以找到特定用户拥有的文件。

`USER`参数可以是用户名或 UID。

例如，要打印所有由用户 slynux 拥有的文件的列表，可以使用以下命令：

```
$ find . -type f -user slynux -print

```

### 使用`find`执行命令或操作

`find`命令可以使用`-exec`选项与许多其他命令配合使用。`-exec`是`find`附带的最强大的功能之一。

让我们看看如何使用`-exec`选项。

考虑前一节中的示例。我们使用`-perm`找出没有适当权限的文件。类似地，在需要将所有由特定用户（例如`root`）拥有的文件的所有权更改为另一个用户（例如 Web 服务器中的默认 Apache 用户`www-data`）的情况下，我们可以使用`-user`选项找到所有由 root 拥有的文件，并使用`-exec`执行所有权更改操作。

### 注意

执行所有权更改时，必须以 root 身份运行`find`命令。

让我们看看以下示例：

```
# find . -type f –user root –exec chown slynux {} \;

```

在此命令中，`{}`是与`-exec`选项一起使用的特殊字符串。对于每个文件匹配，`{}`将被替换为文件名，以替代`-exec`。例如，如果`find`命令找到两个文件`test1.txt`和`test2.txt`，所有者为 slynux，`find`命令将执行：

```
chown slynux {}

```

这将解析为`chown slynux test1.txt`和`chown slynux test2.txt`。

另一个用法示例是将给定目录中的所有 C 程序文件连接起来，并将其写入单个文件`all_c_files.txt`。我们可以使用`find`递归匹配所有 C 文件，并使用`-exec`标志与`cat`命令，如下所示：

```
$ find . -type f -name "*.c" -exec cat {} \;>all_c_files.txt

```

`-exec`后面跟着任何命令。`{}`是一个匹配。对于每个匹配的文件名，`{}`将被替换为文件名。

为了将来自`find`的数据重定向到`all_c_files.txt`文件，我们使用了`>`运算符，而不是`>>`（追加），因为`find`命令的整个输出是单个数据流（`stdin`）。只有在需要将多个数据流追加到单个文件时才需要`>>`。

例如，要将所有早于 10 天的`.txt`文件复制到目录`OLD`中，请使用以下命令：

```
$ find . -type f -mtime +10 -name "*.txt" -exec cp {} OLD  \;

```

同样，`find`命令可以与许多其他命令配合使用。

### 提示

**-exec 与多个命令**

我们不能在`-exec`参数中使用多个命令。它只接受单个命令，但我们可以使用一个技巧。在 shell 脚本中编写多个命令（例如，`commands.sh`）并将其与`-exec`一起使用，如下所示：

`–exec ./commands.sh {} \;`

`-exec`可以与`printf`结合产生非常有用的输出。例如：

```
$ find . -type f -name "*.txt" -exec printf "Text file: %s\n" {} \;

```

### 从 find 中跳过指定的目录

有时在进行目录搜索和执行某些操作时，需要跳过某些子目录以提高性能。例如，当程序员在开发源代码树上查找特定文件时，该源代码树位于版本控制系统（如 Git）下，源代码层次结构将始终包含每个子目录中的`.git`目录（`.git`存储每个目录的版本控制相关信息）。由于版本控制相关目录不会产生有用的输出，因此应将其从搜索中排除。排除文件和目录的技术称为修剪。可以按以下方式执行：

```
$ find devel/source_path  \( -name ".git" -prune \) -o \( -type f -print \)

# Instead of \( -type -print \), use required filter.

```

上述命令打印所有不来自`.git`目录的文件的名称（路径）。

在这里，`\( -name ".git" -prune \)`是排除部分，指定了应该排除`.git`目录，`\( -type f -print \)`指定了要执行的操作。要执行的操作放在第二个块`-type f –print`中（这里指定的操作是打印所有文件的名称和路径）。

# 玩转 xargs

我们使用管道将命令的`stdout`（标准输出）重定向到另一个命令的`stdin`（标准输入）。例如：

```
cat foo.txt | grep "test"

```

但是，一些命令接受命令行参数作为数据，而不是通过 stdin（标准输入）的数据流。在这种情况下，我们不能使用管道通过命令行参数提供数据。

我们应该选择替代方法。`xargs`是一个非常有用的命令，可以处理标准输入数据并转换为命令行参数。`xargs`可以操作 stdin 并将其转换为指定命令的命令行参数。此外，`xargs`还可以将任何单行或多行文本输入转换为其他格式，例如多行（指定列数）或单行，反之亦然。

所有的 Bash 黑客都喜欢单行命令。一行命令是通过使用管道操作符连接的命令序列，但不使用分号终止符（;）在使用的命令之间。编写一行命令可以使任务更有效和更简单地解决。它需要适当的理解和实践来制定解决文本处理问题的一行命令。`xargs`是构建一行命令的重要组件之一。

## 准备工作

`xargs`命令应该始终出现在管道操作符之后。`xargs`使用标准输入作为主要数据流源。它使用`stdin`并通过使用 stdin 数据源为执行命令提供命令行参数。例如：

```
command | xargs

```

## 如何做...

`xargs`命令可以通过重新格式化通过`stdin`接收的数据来为命令提供参数。

`xargs`可以充当替代品，在`find`命令的情况下可以执行与`-exec`参数类似的操作。让我们看看可以使用`xargs`命令执行的各种技巧。

+   **将多行输入转换为单行输出：**

多行输入可以通过删除换行符并用" "（空格）字符替换来简单转换。`'\n'`被解释为换行符，这是行的分隔符。通过使用`xargs`，我们可以忽略所有带有空格的换行符，以便将多行转换为单行文本，如下所示：

```
$ cat example.txt # Example file
1 2 3 4 5 6 
7 8 9 10 
11 12

$ cat example.txt | xargs
1 2 3 4 5 6 7 8 9 10 11 12

```

+   **将单行转换为多行输出：**

给定一行中的最大参数数`= n`，我们可以将任何`stdin`（标准输入）文本分割为每个 n 个参数的行。参数是由“ ”（空格）分隔的字符串片段。空格是默认分隔符。一行可以分成多行，如下所示：

```
$ cat example.txt | xargs -n 3
1 2 3 
4 5 6 
7 8 9 
10 11 12

```

## 它是如何工作的...

`xargs`命令适用于许多问题场景，具有丰富和简单的选项。让我们看看如何明智地使用这些选项来解决问题。

我们还可以使用自己的分隔符来分隔参数。为了指定输入的自定义分隔符，请使用`-d`选项，如下所示：

```
$ echo "splitXsplitXsplitXsplit" | xargs -d X
split split split split

```

在上面的代码中，`stdin`包含一个由多个'X'字符组成的字符串。我们可以使用'X'作为输入分隔符，通过与`-d`一起使用它。在这里，我们已经明确指定 X 作为输入分隔符，而在默认情况下，`xargs`将内部字段分隔符（空格）作为输入分隔符。

通过与上述命令一起使用`-n`，我们可以将输入拆分为每行两个单词的多行，如下所示：

```
$ echo "splitXsplitXsplitXsplit" | xargs -d X -n 2
split split
split split

```

## 还有更多...

我们已经学会了如何将`stdin`格式化为不同的输出，作为上述示例的参数。现在让我们学习如何将这些格式化的输出作为参数提供给命令。

### **通过读取标准输入将格式化的参数传递给命令**

编写一个小的自定义 echo，以更好地理解使用 xargs 提供命令参数的示例用法。

```
#!/bin/bash
#Filename: cecho.sh

echo $*'#' 
```

当参数传递给`cecho.sh`时，它将打印以`#`字符终止的参数。例如：

```
$ ./cecho.sh arg1 arg2
arg1 arg2 #

```

让我们看一个问题：

+   我有一个文件中的参数列表（每行一个参数），要提供给一个命令（比如`cecho.sh`）。我需要用两种方法提供参数。在第一种方法中，我需要为命令提供一个参数，如下所示：

```
./cecho.sh arg1
./cecho.sh arg2
./cecho.sh arg3
```

或者，我需要为每次执行命令提供两个或三个参数。对于每两个参数，它将类似于以下内容：

```
./cecho.sh arg1 arg2
./cecho.sh arg3
```

+   在第二种方法中，我需要一次性提供所有参数给命令，如下所示：

```
./cecho.sh arg1 arg2 arg3
```

运行上述命令，并在阅读以下部分之前记下输出。

上述问题可以使用`xargs`解决。我们在一个名为`args.txt`的文件中有参数列表。内容如下：

```
$ cat args.txt
arg1
arg2
arg3

```

对于第一个问题，我们可以多次执行命令，每次执行一个参数，如下所示：

```
$ cat args.txt | xargs -n 1 ./cecho.sh
arg1 #
arg2 #
arg3 #

```

要执行每次执行 X 个参数的命令，请使用：

```
INPUT | xargs –n X
```

例如：

```
$ cat args.txt | xargs -n 2 ./cecho.sh 
arg1 arg2 #
arg3 #

```

对于第二个问题，我们可以一次执行命令，使用所有参数，如下所示：

```
$ cat args.txt | xargs ./ccat.sh
arg1 arg2 arg3 #

```

在上面的例子中，我们直接向特定命令（例如`cecho.sh`）提供了命令行参数。我们也可以从`args.txt`文件中提供参数。然而，在实时环境中，我们可能还需要在命令（例如`cecho.sh`）中添加一些常量参数，以及从`args.txt`中获取的参数。考虑以下格式的示例：

```
./cecho.sh –p arg1 –l
```

在上述命令执行中，`arg1`是唯一的可变文本。其他所有内容都应保持不变。我们应该从一个文件（`args.txt`）中读取参数并提供它：

```
./cecho.sh –p arg1 –l
./cecho.sh –p arg2 –l
./cecho.sh –p arg3 –l
```

为了提供如下所示的命令执行序列，`xargs`有一个`-I`选项。通过使用`-I`，我们可以指定一个替换字符串，当`xargs`扩展时将被替换。当`-I`与`xargs`一起使用时，它将执行每个参数的一个命令执行。

让我们这样做：

```
$ cat args.txt | xargs -I {} ./cecho.sh -p {} -l
-p arg1 -l #
-p arg2 -l #
-p arg3 -l #

```

`-I {}`指定替换字符串。对于为命令提供的每个参数，`{}`字符串将被通过`stdin`读取的参数替换。当与`-I`一起使用时，命令会像在循环中一样执行。当有三个参数时，命令将执行三次，以及命令`{}`。每次`{}`都会逐个替换参数。

### 使用 xargs 与 find

`xargs`和`find`是最好的朋友。它们可以结合起来轻松执行任务。通常，人们以错误的方式将它们结合在一起。例如：

```
$ find . -type f -name "*.txt"  -print | xargs rm -f 

```

这是危险的。有时可能会导致删除不必要的文件。在这里，我们无法预测定界字符（无论是`'\n'`还是`' '`）对于`find`命令的输出。许多文件名可能包含空格字符（' '），因此`xargs`可能会误解它作为定界符（例如，“hell text.txt”被`xargs`误解为“hell”和“text.txt”）。

因此，我们必须使用`-print0`以及`find`来生成一个带有分隔字符 null(`'\0'`)的输出，每当我们使用`find`的输出作为`xargs`的输入时。

让我们使用`find`来匹配和列出所有`.txt`文件，并使用`xargs`删除它们：

```
$ find . -type f -name "*.txt" -print0 | xargs -0 rm -f

```

这将删除所有`.txt`文件。`xargs -0`解释定界字符为`\0`。

### 在源代码目录中计算 C 代码的行数，涵盖了许多 C 文件。

这是大多数程序员所做的任务，即计算所有 C 程序文件的 LOC（代码行数）。此任务的代码如下：

```
$ find source_code_dir_path -type f -name "*.c" -print0 | xargs -0 wc -l

```

### 使用 while 和子 shell 技巧处理 stdin

`xargs`受限于以有限的方式提供参数以提供参数。此外，xargs 无法向多组命令提供参数。为了执行从标准输入收集的参数的命令，我们有一种非常灵活的方法。我称之为子 shell hack。可以使用带有`while`循环的子 shell 来读取参数并以以下更复杂的方式执行命令：

```
$ cat files.txt  | ( while read arg; do cat $arg; done )
# Equivalent to cat files.txt | xargs -I {} cat {}

```

在这里，通过使用`while`循环替换`cat $arg`，我们可以执行许多具有相同参数的命令操作。我们还可以将输出传递给其他命令，而不使用管道。子 shell`( )`技巧可以在各种问题环境中使用。当包含在子 shell 操作符中时，它作为一个具有多个命令的单个单元。

```
$ cmd0 | ( cmd1;cmd2;cmd3) | cmd4
```

如果`cmd1`是`cd /`，在子 shell 中，工作目录的路径会更改。但是，此更改仅存在于子 shell 中。`cmd4`将看不到目录更改。

# 使用 tr 进行转换

`tr`是 UNIX 命令战士工具包中的一个小而美丽的命令。它是经常用于制作美丽的单行命令的重要命令之一。

`tr`可用于执行标准输入中的字符替换、字符删除和重复字符的挤压。它通常被称为 translate，因为它可以将一组字符转换为另一组字符。

## 准备好

`tr`只通过`stdin`（标准输入）接受输入。它不能通过命令行参数接受输入。它具有以下调用格式：

```
tr [options] set1 set2
```

从`stdin`输入的字符从`set1`映射到`set2`，并将输出写入`stdout`（标准输出）。`set1`和`set2`是字符类或一组字符。如果集的长度不相等，则通过重复最后一个字符来扩展`set2`的长度到`set1`的长度，否则，如果`set2`的长度大于`set1`的长度，则从`set2`中忽略超出`set1`长度的所有字符。

## 如何做...

为了执行输入中的字符从大写转换为小写的转换，请使用以下命令：

```
$ echo "HELLO WHO IS THIS" | tr 'A-Z' 'a-z'

```

`'A-Z'`和`'a-z'`是集合。我们可以通过附加字符或字符类来指定自定义集合。

`'ABD-}'`，`'aA.,'`，`'a-ce-x'`，`'a-c0-9'`等都是有效的集合。我们可以轻松定义集合。我们可以使用`'startchar-endchar'`格式，而不是编写连续的字符序列。它也可以与任何其他字符或字符类结合使用。如果`startchar-endchar`不是有效的连续字符序列，则它们被视为三个字符的集合（例如，`startchar`，`-`和`endchar`）。您还可以使用特殊字符，如`'\t'`，`'\n'`或任何 ASCII 字符。

## 它是如何工作的...

通过使用`tr`与集合的概念，我们可以轻松地将字符从一个集合映射到另一个集合。让我们通过一个示例来了解如何使用`tr`来加密和解密数字字符：

```
$ echo 12345 | tr '0-9' '9876543210'
87654 #Encrypted

$ echo 87654 | tr '9876543210' '0-9'
12345 #Decrypted

```

让我们尝试另一个有趣的例子。

ROT13 是一个众所周知的加密算法。在 ROT13 方案中，相同的函数用于加密和解密文本。ROT13 方案对字符进行了 13 个字符的字母旋转。让我们使用`tr`执行 ROT13，如下所示：

```
$ echo "tr came, tr saw, tr conquered." | tr 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz' 'NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm'

```

输出将是：

```
ge pnzr, ge fnj, ge pbadhrerq.

```

通过再次将加密文本发送到相同的 ROT13 函数，我们得到：

```
$ echo ge pnzr, ge fnj, ge pbadhrerq. | tr 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz' 'NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm'

```

输出将是：

```
tr came, tr saw, tr conquered.

```

`tr`可以将制表符转换为空格，如下所示：

```
$ cat text | tr '\t' ' '

```

## 还有更多...

### 使用 tr 删除字符

`tr`有一个`-d`选项，可以通过使用指定的要删除的字符集在`stdin`上删除一组字符。

```
$ cat file.txt | tr -d  '[set1]'
#Only set1 is used, not set2
```

例如：

```
$ echo "Hello 123 world 456" | tr -d '0-9'
Hello  world
# Removes the numbers from stdin and print

```

### 补集字符集

我们可以使用`-c`标志通过`set1`来对`set1`进行补集。`-c [set]`等同于指定一个包含`[set]`的补集字符的集合。

```
tr -c [set1] [set2]
```

`set1`的补集意味着它是除`set1`中的字符之外的所有字符的集合。

最佳的用法示例是从输入文本中删除除补集中指定的字符之外的所有字符。例如：

```
$ echo hello 1 char 2 next 4 | tr -d -c '0-9 \n'
 1  2  4

```

在这里，补集是包含所有数字、空格字符和换行符的集合。由于`-d`与`tr`一起使用，所有其他字符都被删除。

### 使用 tr 挤压字符

`tr`命令在许多文本处理上下文中非常有用。在许多情况下，需要将连续重复的字符挤压成单个字符。挤压空格是一个经常发生的任务。

`tr`提供了`-s`选项，用于从输入中挤出重复的字符。可以按以下方式执行：

```
$ echo "GNU is       not     UNIX. Recursive   right ?" | tr -s ' '
GNU is not UNIX. Recursive right ?
# tr -s '[set]'

```

让我们以一种巧妙的方式使用`tr`从文件中添加给定的数字列表，如下所示：

```
$ cat sum.txt
1
2
3
4
5

$ cat sum.txt | echo $[ $(tr '\n' '+' ) 0 ]
15

```

这种黑客行为是如何工作的？

在这里，`tr`命令用于用`'+'`字符替换`'\n'`，因此我们形成字符串`"1+2+3+..5+"`，但在字符串末尾我们有一个额外的`+`运算符。为了抵消`+`运算符的影响，附加了`0`。

`$[ operation ]` 执行一个数字操作。因此，它形成如下字符串：

```
echo $[ 1+2+3+4+5+0 ]
```

如果我们使用循环从文件中读取数字执行加法，将需要几行代码。这里一行代码就能完成任务。通过实践可以掌握编写一行代码的技巧。

### 字符类

`tr`可以使用不同的字符类作为集合。不同的类如下：

+   `alnum`：字母数字字符

+   `alpha`：字母字符

+   `cntrl`：控制（不可打印）字符

+   `digit`：数字字符

+   `graph`：图形字符

+   `lower`：小写字母字符

+   `print`：可打印字符

+   `punct`：标点字符

+   `space`：空白字符

+   `upper`：大写字符

+   `xdigit`：十六进制字符

我们可以选择所需的类，并将它们与以下内容一起使用：

```
tr [:class:] [:class:]
```

例如：

```
tr '[:lower:]' '[:upper:]'
```

# 校验和和验证

校验和程序用于从文件生成校验和密钥字符串，并通过使用该校验和字符串稍后验证文件的完整性。文件可能分布在网络或任何存储介质上到不同的目的地。由于许多原因，文件可能因数据传输过程中丢失了一些位而损坏。这些错误经常发生在从互联网下载文件、通过网络传输、CD ROM 损坏等过程中。

因此，我们需要知道接收的文件是否正确，通过应用某种测试来确定。用于进行文件完整性测试的特殊密钥字符串称为**校验和**。

我们计算原始文件和接收文件的校验和。通过比较两个校验和，我们可以验证接收的文件是否正确。如果源位置的原始文件和目的地计算出的校验和相等，这意味着我们已经收到了正确的文件，没有在数据传输过程中造成任何错误的数据丢失，否则，用户必须重复数据传输并再次尝试校验和比较。

在编写备份脚本或由网络传输文件的维护脚本时，校验和至关重要。通过使用校验和验证，可以识别在网络数据传输过程中损坏的文件，并可以从源重新发送这些文件到目的地。因此，始终可以确保接收到的数据的完整性。

## 准备工作

最著名和广泛使用的校验和技术是 `md5sum` 和 `sha1sum`。它们通过将相应的算法应用于文件内容生成校验和字符串。让我们看看如何生成文件的校验和并验证文件的完整性。

## 如何做…

为了计算 `md5sum`，使用以下命令：

```
$ md5sum filename
68b329da9893e34099c7d8ad5cb9c940 filename

```

如上所示，`md5sum` 是一个 32 个字符的十六进制字符串。

我们将校验和输出重定向到文件，并使用该 MD5 文件进行验证，如下所示：

```
$ md5sum filename > file_sum.md5

```

## 工作原理…

`md5sum` 校验和计算的语法如下：

```
$ md5sum file1 file2 file3 ..

```

当使用多个文件时，输出将包含每个文件的校验和，每行一个校验和字符串，如下所示：

```
[checksum1]   file1
[checksum1]   file2
[checksum1]   file3

```

可以使用生成的文件验证文件的完整性，如下所示：

```
$ md5sum -c file_sum.md5
# It will output message whether checksum matches or not

```

或者，如果需要使用所有可用的 `.md5` 信息检查所有文件，可以使用：

```
$ md5sum *.md5

```

SHA1 是另一种常用的校验和算法，类似于 md5sum。它从给定的输入文件生成一个 40 个字符的十六进制代码。用于计算 SHA1 字符串的命令是 `sha1sum`。它的用法与 `md5sum` 非常相似。在本文中先前提到的所有命令中用 `sha1sum` 替换 `md5sum`。将输出文件名更改为 `file_sum.sha1`，而不是 `file_sum.md5`。

校验和验证非常有用，可以验证我们从互联网下载的文件的完整性。我们从互联网下载的 ISO 镜像通常更容易出现错误位。因此，为了检查我们是否正确接收了文件，广泛使用校验和。对于相同的文件数据，校验和程序将始终生成相同的校验和字符串。

## 还有更多…

当与多个文件一起使用时，校验和也非常有用。让我们看看如何对多个文件应用校验和并验证正确性。

### 目录的校验和

文件的校验和是针对文件计算的。计算目录的校验和意味着我们需要递归计算目录中所有文件的校验和。

可以通过 `md5deep` 或 `sha1deep` 命令实现。安装 `md5deep` 软件包以使这些命令可用。以下是此命令的示例：

```
$ md5deep -rl directory_path > directory.md5
# -r  for enable recursive.
# -l for using relative path. By default it writes absolute file path in output

```

或者，将其与 `find` 结合使用以递归计算校验和：

```
$ find directory_path -type f -print0 | xargs -0 md5sum >> directory.md5

```

要验证，请使用以下命令：

```
$ md5sum -c directory.md5

```

# 排序、唯一和重复项

排序是我们在文本文件中经常遇到的常见任务。因此，在文本处理任务中，sort 非常有用。`sort` 命令帮助我们对文本文件和 `stdin` 执行排序操作。它通常也可以与许多其他命令配合使用以生成所需的输出。`uniq` 是另一个经常与 `sort` 命令一起使用的命令。它有助于从文本或 `stdin` 中提取唯一行。`sort` 和 `uniq` 可以组合以查找重复项。本文介绍了 `sort` 和 `uniq` 命令的大多数用例。

## 准备工作

`sort` 命令接受文件名作为输入，也可以从 `stdin`（标准输入）中读取，并通过写入 `stdout` 输出结果。`uniq` 命令遵循相同的操作顺序。

## 如何做…

我们可以轻松地对给定的一组文件（例如 `file1.txt` 和 `file2.txt`）进行排序，如下所示：

```
$ sort file1.txt file2.txt .. > sorted.txt

```

或：

```
$ sort file1.txt file2.txt .. -o sorted.txt

```

为了从已排序的文件中查找唯一行，请使用：

```
$ cat sorted_file.txt | uniq> uniq_lines.txt

```

## 工作原理…

有许多情况可以使用 `sort` 和 `uniq` 命令。让我们看看各种选项和用法技巧。

要进行数字排序，请使用：

```
$ sort -n file.txt

```

要按相反顺序排序，请使用：

```
$ sort -r file.txt

```

按月份排序（按照 1 月、2 月、3 月的顺序）使用：

```
$ sort -M months.txt

```

可以按以下方式测试文件是否已排序：

```
#!/bin/bash
#Desc: Sort
sort -C file ;
if [ $? -eq 0 ]; then
   echo Sorted;
else
   echo Unsorted;
fi

# If we are checking numerical sort, it should be sort -nC
```

要合并两个已排序的文件而不再次排序，请使用：

```
$ sort -m sorted1 sorted2

```

## 还有更多...

### 根据键或列进行排序

如果需要对文本进行排序，则使用列排序如下：

```
$ cat data.txt
1  mac      2000
2  winxp    4000
3  bsd      1000
4  linux    1000

```

我们可以以多种方式对其进行排序；当前是按序列号（第一列）进行数字排序。我们还可以按第二列和第三列进行排序。

`-k`指定要执行排序的键。键是要进行排序的列号。`-r`指定 sort 命令以相反顺序排序。例如：

```
# Sort reverse by column1
$ sort -nrk 1  data.txt
4	linux		1000 
3	bsd		1000 
2	winxp		4000 
1	mac		2000 
# -nr means numeric and reverse

# Sort by column 2
$ sort -k 2  data.txt
3	bsd		1000 
4	linux		1000 
1	mac		2000 
2	winxp		4000

```

### 注意

始终要小心`-n`选项进行数字排序。sort 命令以不同的方式处理字母排序和数字排序。因此，为了指定数字排序，应提供`-n`选项。

通常，默认情况下，键是文本文件中的列。列由空格字符分隔。但在某些情况下，我们需要将键指定为给定字符编号范围内的一组字符（例如，key1= character4-character8）。在需要显式指定键为字符范围的情况下，我们可以指定键为字符位置的范围，如下所示：

```
$ cat data.txt
1010hellothis
2189ababbba
7464dfddfdfd
$ sort -nk 2,3 data.txt

```

要使用突出显示的字符作为数字键。要提取它们，请使用它们的起始位置和结束位置作为键格式。

要使用第一个字符作为键，请使用：

```
$ sort -nk 1,1 data.txt

```

通过使用以下命令，使 sort 的输出与`\0`终止符兼容，从而与`xargs`兼容：

```
$ sort -z data.txt | xargs -0
#Zero terminator is used to make safe use with xargs

```

有时文本可能包含不必要的多余字符，如空格。要按字典顺序忽略它们进行排序，忽略标点和折叠，使用：

```
$ sort -bd unsorted.txt

```

选项`-b`用于忽略文件中的前导空格，选项`-d`用于指定按字典顺序排序。

### uniq

`uniq`是一个用于从给定输入（`stdin`或从文件名作为命令参数）中找出唯一行的命令，通过消除重复项。它也可以用于从输入中找出重复行。`uniq`只能应用于排序后的数据输入。因此，`uniq`始终应与使用管道或使用排序文件作为输入的`sort`命令一起使用。

您可以从给定的输入数据中生成唯一行（唯一行表示打印输入中的所有行，但重复行仅打印一次）如下：

```
$ cat sorted.txt
bash 
foss 
hack 
hack

$ uniq sorted.txt
bash 
foss 
hack 

```

或者：

```
$ sort unsorted.txt | uniq

```

或者：

```
$ sort -u unsorted.txt

```

仅显示唯一行（输入文件中未重复或重复的行）如下：

```
$ uniq -u sorted.txt
bash
foss

```

或者：

```
$ sort unsorted.txt | uniq -u

```

要计算文件中每行出现的次数，请使用以下命令：

```
$ sort unsorted.txt | uniq -c
 1 bash
 1 foss
 2 hack

```

查找文件中的重复行如下：

```
$ sort unsorted.txt  | uniq -d
hack

```

要指定键，我们可以使用`-s`和`-w`参数的组合。

+   `-s`指定要跳过的前`N`个字符的数字

+   -w`指定要比较的最大字符数

此比较键用作`uniq`操作的索引，如下所示：

```
$ cat data.txt
u:01:gnu 
d:04:linux 
u:01:bash 
u:01:hack

```

我们需要使用突出显示的字符作为唯一性键。这用于忽略前 2 个字符（`-s 2`），并使用`-w`选项指定比较字符的最大数量（`-w 2`）：

```
$ sort data.txt | uniq -s 2 -w 2
d:04:linux 
u:01:bash 

```

当我们使用一个命令的输出作为 xargs 命令的输入时，最好为输出的每行使用零字节终止符，这充当`xargs`的源。在使用`uniq`命令的输出作为`xargs`的源时，我们应该使用零终止输出。如果不使用零字节终止符，则空格字符默认为`xargs`命令中的分隔符以拆分参数。例如，来自`stdin`的文本“this is a line”将被`xargs`视为四个单独的参数。但实际上，它是一行。当使用零字节终止符时，`\0`被用作分隔符字符，因此，包括空格的单行被解释为单个参数。

可以从`uniq`命令生成零字节终止的输出，如下所示：

```
$ uniq -z file.txt

```

以下命令从`files.txt`中读取的文件名中删除所有文件：

```
$ uniq –z file.txt | xargs -0 rm

```

如果文件中存在多行文件名条目，则`uniq`命令仅将文件名写入`stdout`一次。

### 使用 uniq 生成字符串模式

这里有一个有趣的问题：我们有一个包含重复字符的字符串。我们如何找出每个字符在字符串中出现的次数，并输出以下格式的字符串？

输入：ahebhaaa

输出：4a1b1e2h

每个字符都重复一次，并且它们中的每一个都以它们在字符串中出现的次数作为前缀。我们可以使用`uniq`和`sort`来解决这个问题，如下所示：

```
INPUT= "ahebhaaa"
OUTPUT=` echo $INPUT | sed 's/[^\n]/&\n/g' | sed '/^$/d' | sort | uniq -c | tr -d ' \n'`
echo $OUTPUT 
```

在上面的代码中，我们可以将每个管道命令拆分如下：

```
echo $INPUT  # Print the input to stdout
sed 's/./&\n/g'

```

为每个字符附加一个换行符，以便每行只出现一个字符。这样做是为了使字符可以通过`sort`命令进行排序。`sort`命令只能接受由换行符分隔的项目。

+   `sed '/^$/d'`：这里最后一个字符被替换为字符`+\n`。因此会形成一个额外的换行符，并且会在末尾形成一个空行。此命令删除末尾的空行。

+   `sort`：由于每个字符都出现在每行中，因此可以对其进行排序，以便作为 uniq 的输入。

+   `uniq -c`：此命令打印每行重复的次数。

+   `tr -d ' \n'`：这将从输入中删除空格字符和换行字符，以便以给定格式生成输出。

# 临时文件命名和随机数

在编写 shell 脚本时，我们经常需要存储临时数据。存储临时数据的最合适位置是`/tmp`（系统在重新启动时将清除该位置）。我们可以使用两种方法为临时数据生成标准文件名。

## 如何做…

`tempfile`在非 Debian Linux 发行版中看不到。`tempfile`命令随 Debian 系发行版一起提供，例如 Ubuntu、Debian 等。

以下代码将临时文件名分配给变量`temp_file`：

```
temp_file=$(tempfile)
```

使用`echo $temp_file`在终端中打印临时文件名。

输出将类似于`/tmp/fileaZWm8Y`。

有时我们可能会使用带有随机数字附加的文件名作为临时文件名。可以按以下方式完成：

```
temp_file="/tmp/file-$RANDOM"
```

`$RANDOM`环境变量始终返回一个随机数。

## 它是如何工作的…

我们可以使用自己的临时文件名，而不是使用`tempfile`命令。大多数经验丰富的 UNIX 程序员使用以下约定：

```
temp_file="/tmp/var.$$"
```

附加了`.$$`后缀。`$$`在执行时会扩展为当前脚本的进程 ID。

# 拆分文件和数据

在某些情况下，将文件拆分为许多较小的部分变得至关重要。在早期，当存储器受限于软盘等设备时，将文件拆分为较小的文件大小以在许多磁盘中传输数据至关重要。然而，如今我们拆分文件是为了其他目的，例如可读性，生成日志等。

## 如何做…

生成一个 100kb 的测试文件（`data.file`）如下：

```
$ dd if=/dev/zero bs=100k count=1 of=data.file

```

上述命令创建了一个大小为 100kb 的用零填充的文件。

您可以通过指定拆分大小将文件拆分为较小的文件，如下所示：

```
$ split -b 10k data.file
$ ls
data.file  xaa  xab  xac  xad  xae  xaf  xag  xah  xai  xaj

```

它将`data.file`拆分为许多文件，每个文件大小为 10k。这些块将以`xab`、`xac`、`xad`等方式命名。这意味着它们将具有字母后缀。要使用数字后缀，可以使用额外的`-d`参数。还可以使用`-a length`指定后缀长度，如下所示：

```
$ split -b 10k data.file -d  -a 4
$ ls
data.file x0009  x0019  x0029  x0039  x0049  x0059  x0069  x0079

```

我们可以使用`M`代替`k`（千字节）后缀，使用`M`代替`MB`，使用`G`代替`GB`，使用`c`代替字节，使用`w`代替字，等等。

## 还有更多…

`split`命令有更多选项。让我们来看看它们。

### 指定拆分文件的文件名前缀

上述拆分文件具有文件名前缀"x"。我们还可以通过提供前缀文件名来使用自己的文件名前缀。拆分命令的最后一个命令参数是`PREFIX`。它的格式是：

```
$ split [COMMAND_ARGS] PREFIX
```

让我们使用拆分文件的前缀文件名运行上一个命令：

```
$ split -b 10k data.file -d  -a 4 split_file
$ ls
data.file       split_file0002  split_file0005  split_file0008  strtok.c
split_file0000  split_file0003  split_file0006  split_file0009
split_file0001  split_file0004  split_file0007

```

为了根据每个拆分的行数而不是块大小拆分文件，使用`-l no_of_lines`如下：

```
$ split -l 10 data.file 
# Splits into files of 10 lines each.

```

还有另一个有趣的实用程序叫做`csplit`。它可以用于根据指定条件和字符串匹配选项拆分日志文件。让我们看看如何使用它。

`csplit`是`split`实用程序的一个变体。`split`实用程序只能根据块大小或行数拆分文件。`csplit`根据上下文拆分文件。它可以用于根据某个特定单词或文本内容的存在来拆分文件。

看一个示例日志：

```
$ cat server.log
SERVER-1 
[connection] 192.168.0.1 success 
[connection] 192.168.0.2 failed 
[disconnect] 192.168.0.3 pending 
[connection] 192.168.0.4 success 
SERVER-2 
[connection] 192.168.0.1 failed 
[connection] 192.168.0.2 failed 
[disconnect] 192.168.0.3 success 
[connection] 192.168.0.4 failed 
SERVER-3 
[connection] 192.168.0.1 pending 
[connection] 192.168.0.2 pending 
[disconnect] 192.168.0.3 pending 
[connection] 192.168.0.4 failed

```

我们可能需要从每个文件中的每个`SERVER`的内容中拆分文件为`server1.log`，`server2.log`和`server3.log`。可以按以下方式完成：

```
 $ csplit server.log /SERVER/ -n 2 -s {*}  -f server -b "%02d.log"  ; rm server00.log 

$ ls
server01.log  server02.log  server03.log  server.log

```

命令的详细信息如下：

+   `/SERVER/`是用于匹配拆分的行。

+   `/[REGEX]/` 是格式。它从当前行（第一行）复制到包含`"SERVER"`的匹配行，但不包括匹配行。

+   `{*}`用于指定根据匹配重复拆分直到文件的末尾。通过使用`{integer}`，我们可以指定要继续的次数。

+   `-s`是一个标志，使命令保持安静，而不是打印其他消息。

+   `-n`用于指定要用作后缀的数字位数。01、02、03 等。

+   `-f`用于指定拆分文件的文件名前缀（在上一个示例中的前缀是"server"）。

+   `-b`用于指定后缀格式。`"%02d.log"`类似于 C 中的`printf`参数格式。这里的文件名=前缀+后缀=`"server" + "%02d.log"`。

我们删除`server00.log`，因为第一个拆分文件是一个空文件（匹配单词是文件的第一行）。

# 根据扩展名切片文件名

几个自定义的 shell 脚本根据文件名执行操作。我们可能需要执行诸如保留扩展名重命名文件，将文件从一种格式转换为另一种格式（通过保留名称更改扩展名），提取文件名的一部分等操作。Shell 具有内置功能，可以根据不同条件对文件名进行切片。让我们看看如何做到这一点。

## 如何做…

从`name.extension`中的名称可以通过使用`%`运算符轻松提取。您可以按如下方式从`"sample.jpg"`中提取名称：

```
file_jpg="sample.jpg"
name=${file_jpg%.*}
echo File name is: $name
```

输出是：

```
File name is: sample
```

下一个任务是从文件名中提取文件的扩展名。可以使用`#`运算符提取扩展名。

从变量`file_jpg`中的文件名中提取`.jpg`如下：

```
extension=${file_jpg#*.}
echo Extension is: jpg
```

输出是：

```
Extension is: jpg
```

## 它是如何工作的..

在第一个任务中，为了从格式为`name.extension`的文件名中提取名称，我们使用了`%`运算符。

`${VAR%.*}` 可以解释为：

+   从`$VARIABLE`中删除通配符模式的字符串匹配，该模式出现在%的右侧（在上一个示例中为.*）。从右到左的方向进行评估应该使通配符匹配。

+   让`VAR=sample.jpg`。因此，从右到左的通配符匹配是`.jpg`。因此它从`$VAR`字符串中删除，并且输出将是"sample"。

`%`是一种非贪婪操作。它从右到左找到通配符的最小匹配。还有一个运算符`%%`，类似于`%`。但它是贪婪的。这意味着它匹配通配符的最大字符串。

例如，我们有：

```
VAR=hack.fun.book.txt
```

通过使用`%`运算符，我们有：

```
$ echo ${VAR%.*}

```

输出将是：`hack.fun.book`。

运算符`%`从右到左执行`.*`的非贪婪匹配（`.txt`）。

通过使用`%%`运算符，我们有：

```
$ echo ${VAR%%.*}

```

输出将是：`hack`

`%%`运算符从右到左进行贪婪匹配`.*`（.`fun.book.txt`）。

在第二个任务中，我们使用了`#`运算符从文件名中提取扩展名。它类似于`%`。但它是从左到右进行评估的。

`${VAR#*.}` 可以解释为：

从`#`右侧出现的通配符模式匹配的字符串匹配从左到右的方向应该使通配符匹配。

类似地，与`%%`一样，我们还有另一个贪婪运算符`#`，即`##`。

它通过从左到右评估并从指定变量中删除匹配字符串来进行贪婪匹配。

让我们使用这个例子：

```
VAR=hack.fun.book.txt
```

通过使用`#`运算符，我们有：

```
$ echo ${VAR#*.} 
```

输出将是：`fun.book.txt`。

运算符`#`对`hack.`进行了从左到右的非贪婪匹配。

通过使用##运算符，我们有：

```
$ echo ${VAR##*.}
```

输出将是：`txt`。

运算符##从左到右（txt）进行贪婪匹配。

### 注意

##运算符比#运算符更受欢迎，用于从文件名中提取扩展名，因为文件名可能包含多个'.'字符。由于##进行贪婪匹配，它总是只提取扩展名。

以下是一个实际示例，可用于提取域名的不同部分，给定 URL="[www.google.com](http://www.google.com)"：

```
$ echo ${URL%.*} # Remove rightmost .*
www.google

$ echo ${URL%%.*} # Remove right to leftmost  .* (Greedy operator)
www

$ echo ${URL#*.} # Remove leftmost  part before *.
google.com

$ echo ${URL##*.} # Remove left to rightmost  part before *. (Greedy operator)
com
```

# 批量重命名和移动文件

重命名多个文件是我们经常遇到的任务之一。一个简单的例子是，当您从数码相机下载照片到计算机时，您可能会删除不必要的文件，导致图像文件的编号不连续。有时您可能需要使用自定义前缀和连续编号对它们进行重命名。我们有时使用第三方工具执行重命名操作。我们可以使用 Bash 命令在几秒钟内执行重命名操作。

将所有文件名中包含特定子字符串（例如，文件名具有相同前缀）或具有特定文件类型的文件移动到给定目录是我们经常执行的另一个用例。让我们看看如何编写脚本来执行这些类型的操作。

## 准备工作

`rename`命令有助于使用 Perl 正则表达式更改文件名。通过组合`find`、`rename`和`mv`命令，我们可以执行很多操作。

## 如何做…

在当前目录中重命名图像文件为特定格式的最简单方法是使用以下脚本：

```
#!/bin/bash
#Filename: rename.sh
#Description: Rename jpg and png files

count=1;
for img in *.jpg *.png
do
new=image-$count.${img##*.}

mv "$img" "$new" 2> /dev/null

if [ $? -eq  0 ];
then

echo "Renaming $img to $new"
let count++

fi

done 
```

输出如下：

```
$ ./rename.sh
Renaming hack.jpg to image-1.jpg
Renaming new.jpg to image-2.jpg
Renaming next.jpg to image-3.jpg

```

该脚本将当前目录中的所有`.jpg`和`.png`文件重命名为格式为`image-1.jpg`、`image-2.jpg`、`image-3.jpg`、`image-4.png`等的新文件名。

## 它是如何工作的…

在上面的重命名脚本中，我们使用了`for`循环来遍历所有以`.jpg`扩展名结尾的文件名。通配符`*.jpg`和`*.png`用于匹配所有 JPEG 和 PNG 文件。我们可以对扩展名匹配进行一点改进。`.jpg`通配符只匹配小写的扩展名。然而，我们可以通过用`.[jJ][pP][gG]`替换`.jpg`来使其不区分大小写。因此它可以匹配文件`file.jpg`以及`file.JPG`或`file.Jpg`。在 Bash 中，当字符被包含在`[]`中时，它意味着从`[]`中包含的字符集中匹配一个字符。

上述代码中的`for img in *.jpg *.png`将被扩展为：

`for img in hack.jpg new.jpg next.jpg`

我们初始化了一个变量`count=1`，以便跟踪图像编号。下一步是使用`mv`命令重命名文件。文件的新名称应该经过制定。脚本中的`${img##*.}`解析了当前循环中的文件名的扩展名（有关`${img##*.}`的解释，请参见*基于扩展名切片文件名*配方）。

`let count++`用于递增每次循环执行的文件编号。

您可以看到使用`2>`运算符将错误重定向（`stderr`）到`/dev/null`是为了阻止错误消息打印到终端。

由于我们使用`*.png`和`*.jpg`，如果通配符匹配的至少一个图像不存在，shell 将解释通配符本身作为一个字符串。在上面的输出中，您可以看到`.png`文件不存在。因此它将把`*.png`作为另一个文件名并执行`mv *.png image-X.png`，这将导致错误。使用`[ $? –eq 0 ]`的`if`语句来检查退出状态(`$?`)。如果最后执行的命令成功，`$?`的值将为 0，否则返回非零。当`mv`命令失败时，它返回非零，因此用户将不会看到消息"重命名文件"，计数也不会增加。

有各种其他方法可以执行重命名操作。让我们来看看其中的一些。

将`*.JPG`重命名为`*.jpg`：

```
$ rename *.JPG *.jpg

```

将文件名中的空格替换为"`_"`字符如下：

```
$ rename 's/ /_/g' *

```

`# 's/ /_/g'`是文件名中的替换部分，`*`是目标文件的通配符。它可以是`*.txt`或任何其他通配符模式。

您可以按照以下方式将任何文件名从大写转换为小写，反之亦然：

```
$ rename 'y/A-Z/a-z/' *
$ rename 'y/a-z/A-Z/' *

```

为了递归地将所有`.mp3`文件移动到给定目录中，请使用：

```
$ find path -type f -name "*.mp3" -exec mv {} target_dir \;

```

通过以下方式递归重命名所有文件，将空格替换为"`_"`字符：

```
$ find path -type f -exec rename 's/ /_/g' {} \;

```

# 拼写检查和字典操作

大多数 Linux 发行版都附带一个字典文件。然而，我发现很少有人知道字典文件，因此许多人未能利用它们。有一个名为`aspell`的命令行实用程序，它可以作为拼写检查器。让我们看看一些利用字典文件和拼写检查器的脚本。

## 如何做...

`/usr/share/dict/`目录包含一些字典文件。字典文件是包含字典单词列表的文本文件。我们可以使用这个列表来检查一个单词是否是字典单词。

```
$ ls /usr/share/dict/ 
american-english  british-english

```

为了检查给定的单词是否是字典单词，请使用以下脚本：

```
#!/bin/bash
#Filename: checkword.sh
word=$1
grep "^$1$" /usr/share/dict/british-english -q 
if [ $? -eq 0 ]; then
  echo $word is a dictionary word;
else
  echo $word is not a dictionary word;
fi
```

用法如下：

```
$ ./checkword.sh ful 
ful is not a dictionary word 

$ ./checkword.sh fool 
fool is a dictionary word

```

## 它是如何工作的...

在`grep`中，`^`是单词起始标记字符，字符`$`是单词结束标记。

`-q`用于抑制任何输出并保持沉默。

或者，我们可以使用拼写检查`aspell`来检查一个单词是否在字典中，如下所示：

```
#!/bin/bash 
#Filename: aspellcheck.sh
word=$1 

output=`echo \"$word\" | aspell list` 

if [ -z $output ]; then 
        echo $word is a dictionary word; 
else 
        echo $word is not a dictionary word; 
fi 
```

`aspell list`命令在给定输入不是字典单词时返回输出文本，并且在输入为字典单词时不输出任何内容。`-z`检查确保`$output`是否为空字符串。

列出文件中以给定单词开头的所有单词如下：

```
$ look word filepath

```

或者，使用：

```
$ grep "^word" filepath

```

默认情况下，如果未给出文件名参数，则`look`命令使用默认字典(`/usr/share/dict/words`)并返回输出。

```
$ look word
# When used like this it takes default dictionary as file

```

例如：

```
$ look android
android
android's
androids

```

# 自动化交互式输入

自动化命令行实用程序的交互式输入对于编写自动化工具或测试工具非常有用。在处理需要交互式输入的命令时，会有许多情况。交互式输入是用户在命令要求输入时键入的输入。执行命令并提供交互式输入的示例如下：

```
$ command
Enter a number: 1
Enter name : hello
You have entered 1,hello

```

## 准备工作

自动化实用程序可以自动接受输入，就像上面提到的方式一样，对于提供输入给本地命令以及远程应用程序都是有用的。让我们看看如何自动化它们。

## 如何做...

考虑交互式输入的顺序。从前面的代码中，我们可以得出以下序列的步骤：

`1[Return]hello[Return]`

通过观察实际在键盘上键入的字符，将上述步骤`1,Return,hello,Return`转换为以下字符串。

`"1\nhello\n"`

当我们按下*Return*时，`\n`字符被发送。通过附加回车(`\n`)字符，我们得到传递给`stdin`（标准输入）的实际字符串。

因此，通过发送用户输入的等效字符串，我们可以自动化交互过程中的输入传递。

## 它是如何工作的...

让我们编写一个脚本，以交互方式读取输入，并将此脚本用于自动化示例：

```
#!/bin/bash
#Filename: interactive.sh
read -p "Enter number:" no ;
read -p "Enter name:" name
echo You have entered $no, $name;
```

让我们按以下方式自动发送输入到命令：

```
$ echo -e "1\nhello\n" | ./interactive.sh 
You have entered 1, hello 

```

因此，使用`\n`来制作输入是有效的。

我们使用了`echo -e`来生成输入序列。如果输入很大，我们可以使用输入文件和重定向运算符来提供输入。

```
$ echo -e "1\nhello\n"  > input.data
$ cat input.data
1
hello

```

您也可以手动制作输入文件，而不使用`echo`命令手动输入。例如：

```
$ ./interactive.sh < input.data

```

这将从文件中重定向交互式输入数据。

如果您是一名逆向工程师，您可能已经玩过缓冲区溢出漏洞。为了利用它们，我们需要重定向 shellcode，比如`"\xeb\x1a\x5e\x31\xc0\x88\x46"`，它是用十六进制编写的。这些字符不能直接通过键盘输入，因为键盘上没有这些字符的键。因此我们应该使用：

`echo -e "\xeb\x1a\x5e\x31\xc0\x88\x46"`

这将把 shellcode 重定向到一个有漏洞的可执行文件。

我们已经描述了一种通过将预期输入文本重定向通过`stdin`（标准输入）来自动化交互式输入程序的方法。我们发送输入而不检查程序要求的输入。我们通过期望程序按特定（静态）顺序要求输入来发送输入。如果程序随机或以不同顺序要求输入，或者有时根本不要求某些输入，则上述方法将失败。它将向程序的不同输入提示发送错误的输入。为了处理动态输入供应并通过检查程序在运行时的输入要求来提供输入，我们有一个称为`expect`的实用工具。`expect`命令通过程序发送正确的输入到正确的输入提示。让我们看看如何使用`expect`。

## 还有更多...

交互式输入的自动化也可以使用其他方法。Expect 脚本是另一种自动化方法。让我们来看看。

### 使用 expect 进行自动化

`expect`实用程序在大多数常见的 Linux 发行版中默认不包含。您必须使用软件包管理器手动安装 expect 软件包。

`expect`期望特定的输入提示，并通过检查输入提示中的消息发送数据。

```
#!/usr/bin/expect 
#Filename: automate_expect.sh
spawn ./interactive .sh 
expect "Enter number:" 
send "1\n" 
expect "Enter name:" 
send "hello\n" 
expect eof 
```

运行如下：

```
$ ./automate_expect.sh

```

在这个脚本中：

+   `spawn`参数指定要自动化的命令

+   `expect`参数提供了预期的消息

+   `send`是要发送的消息。

+   `expect eof`定义了命令交互的结束
