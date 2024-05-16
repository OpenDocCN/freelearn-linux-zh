# 第四章：发短信和开车

在这一章中，我们将涵盖：

+   基本正则表达式入门

+   使用 grep 在文件中搜索和挖掘“文本”

+   使用 cut 按列切割文件

+   确定给定文件中使用的单词频率

+   基本 sed 入门

+   基本 awk 入门

+   从文本或文件中替换字符串

+   压缩或解压 JavaScript

+   在文件中迭代行、单词和字符

+   将多个文件合并为列

+   在文件或行中打印第 n 个单词或列

+   打印行号或模式之间的文本

+   使用脚本检查回文字符串

+   以相反的顺序打印行

+   从文本中解析电子邮件地址和 URL

+   在文件中打印模式之前或之后的一组行

+   从文件中删除包含特定单词的句子

+   使用 awk 实现 head、tail 和 tac

+   文本切片和参数操作

# 介绍

Shell 脚本语言中包含了 UNIX/Linux 系统的基本问题解决组件。Bash 总是可以为 UNIX 环境中的问题提供一些快速解决方案。文本处理是 Shell 脚本使用的关键领域之一。它带有诸如 sed、awk、grep、cut 等美丽的实用程序，可以组合使用以解决与文本处理相关的问题。大多数编程语言都设计为通用的，因此编写能够处理文本并产生所需输出的程序需要付出很大的努力。由于 Bash 是一种考虑到文本处理的语言，它具有许多功能。

各种实用程序帮助以字符、行、单词、列、行等细节处理文件。因此我们可以以多种方式操纵文本文件。正则表达式是模式匹配技术的核心。大多数文本处理实用程序都带有正则表达式支持。通过使用合适的正则表达式字符串，我们可以产生所需的输出，如过滤、剥离、替换、搜索等。

本章包括了一系列配方，涵盖了基于文本处理的许多问题背景，这将有助于编写真实脚本。

# 基本正则表达式入门

正则表达式是基于模式匹配的文本处理技术的核心。要流利地编写文本处理工具，必须对正则表达式有基本的理解。正则表达式是一种微型、高度专门化的编程语言，用于匹配文本。使用通配符技术，使用模式匹配文本的范围非常有限。这个配方是基本正则表达式的介绍。

## 准备工作

正则表达式是大多数文本处理实用程序中使用的语言。因此，您将在许多其他配方中使用在本配方中学到的技术。`[a-z0-9_]+@[a-z0-9]+\.[a-z]+` 是一个用于匹配电子邮件地址的正则表达式的示例。

这看起来奇怪吗？别担心，一旦你理解了概念，它就真的很简单。

## 如何做...

在本节中，我们将介绍正则表达式、POSIX 字符类和元字符。

让我们首先了解正则表达式（regex）的基本组件。

| regex | 描述 | 示例 |
| --- | --- | --- |
| `^` | 行首标记。 | `^tux` 匹配以`tux`开头的行的字符串。 |
| `$` | 行尾标记。 | `tux$` 匹配以`tux`结尾的行的字符串。 |
| `.` | 匹配任何一个字符。 | `Hack.` 匹配 `Hack1`，`Hacki` 但不匹配 `Hack12`，`Hackil`，只有一个额外的字符匹配。 |
| `[]` | 匹配`[chars]`中包含的任何一个字符。 | `coo[kl]` 匹配 `cook` 或 `cool`。 |
| `[^]` | 匹配除了`[^chars]`中包含的字符之外的任何一个字符。 | `9[⁰¹]` 匹配 `92`，`93` 但不匹配 `91` 或 `90`。 |
| `[-]` | 匹配`[]`中指定范围内的任何字符。 | `[1-5]` 匹配从 `1` 到 `5` 的任何数字。 |
| `?` | 前面的项目必须匹配一次或零次。 | `colou?r`匹配`color`或`colour`但不匹配`colouur`。 |
| `+` | 前面的项目必须匹配一次或多次。 | `Rollno-9+`匹配`Rollno-99`，`Rollno-9`但不匹配`Rollno-`。 |
| `*` | 前面的项目必须匹配零次或多次。 | `co*l`匹配`cl`，`col`，`coool`。 |
| `()` | 从正则表达式匹配中创建一个子字符串。 | `ma(tri)?x`匹配`max`或`matrix`。 |
| `{n}` | 前面的项目必须匹配 n 次。 | `[0-9]{3}`匹配任意三位数。`[0-9]{3}`可以扩展为:`[0-9][0-9][0-9]`。 |
| `{n,}` | 前面的项目应该至少匹配的次数。 | `[0-9]{2,}`匹配任何数字，即两位数或更多。 |
| `{n, m}` | 指定前面的项目应该匹配的最小和最大次数。 | `[0-9]{2,5}`匹配任何有两到五位数字的数字。 |
| `&#124;` | 交替-&#124;两侧的项目之一应该匹配。 | `Oct (1st &#124; 2nd)`匹配`Oct 1st`或`Oct 2nd`。 |
| `\` | 用于转义上述任何特殊字符的转义字符。 | `a\.b`匹配`a.b`但不匹配`ajb`。它通过前缀`\`忽略`.`的特殊含义。 |

POSIX 字符类是一种特殊的元序列，形式为`[:...:]`，可用于匹配指定字符范围。POSIX 类如下：

| 正则表达式 | 描述 | 例子 |
| --- | --- | --- |
| `[:alnum:]` | 字母数字字符 | `[[:alnum:]]+` |
| `[:alpha:]` | 字母字符（小写和大写） | `[[:alpha:]]{4}` |
| `[:blank:]` | 空格和制表符 | `[[:blank:]]*` |
| `[:digit:]` | 数字 | `[[:digit:]]?` |
| `[:lower:]` | 小写字母 | `[[:lower:]]{5,}` |
| `[:upper:]` | 大写字母 | `([[:upper:]]+)?` |
| `[:punct:]` | 标点符号 | `[[:punct:]]` |
| `[:space:]` | 包括换行符、回车符等所有空白字符。 | `[[:space:]]+` |

元字符是一种 Perl 风格的正则表达式，它受到一些文本处理实用程序的支持。并非所有实用程序都支持以下符号。但上述字符类和正则表达式是被普遍接受的。

| 正则表达式 | 描述 | 例子 |
| --- | --- | --- |
| `\b` | 单词边界 | `\bcool\b`只匹配`cool`而不匹配`coolant`。 |
| `\B` | 非单词边界 | `cool\B`匹配`coolant`而不是`cool`。 |
| `\d` | 单个数字字符 | `b\db`匹配`b2b`而不是`bcb`。 |
| `\D` | 单个非数字 | `b\Db`匹配`bcb`而不是`b2b`。 |
| `\w` | 单个单词字符（字母数字和 _） | `\w`匹配`1`或`a`而不是`&`。 |
| `\W` | 单个非单词字符 | `\w`匹配`&`而不是`1`或`a`。 |
| `\n` | 换行符 | `\n`匹配一个换行符。 |
| `\s` | 单个空格 | `x\sx`匹配`xx`而不是`xx`。 |
| `\S` | 单个非空格 | `x\Sx`匹配`xkx`而不是`xx`。 |
| `\r` | 回车 | `\r`匹配回车。 |

## 它是如何工作的...

在前一节中看到的表格是正则表达式的关键元素表。通过使用表中的合适键，我们可以构建任何适当的正则表达式字符串来根据上下文匹配文本。正则表达式是一种通用语言，用于匹配文本。因此，在本教程中我们不会介绍任何工具。但是，它遵循本章中的其他教程。

让我们看一些文本匹配的例子：

+   为了匹配给定文本中的所有单词，我们可以将正则表达式写成：

```
( ?[a-zA-Z]+ ?)
```

"?"是在单词前后表示可选空格的符号。`[a-zA-Z]+`表示一个或多个字母字符（a-z 和 A-Z）。

+   为了匹配 IP 地址，我们可以将正则表达式写成：

```
[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}

```

或者

```
[[:digit:]]{1,3}\.[[:digit:]]{1,3}\.[[:digit:]]{1,3}\.[[:digit:]]{1,3}

```

我们知道 IP 地址的格式是 192.168.0.2\. 它是由四个整数（每个从 0-255）用点分隔的形式（例如，192.168.0.2）。

`[0-9]`或`[:digit:]`表示匹配数字 0-9。`{1,3}`匹配一到三个数字，`\.`匹配“.”。

## 还有更多...

让我们看看正则表达式中特定字符的特殊含义是如何指定的。

### 特殊字符的处理

正则表达式使用一些特殊字符，如`$`、`^`、`.`、`*`、`+`、`{`和`}`。但如果我们想将这些字符用作非特殊字符（普通文本字符）呢？让我们看一个例子。

正则表达式：`[a-z]*.[0-9]`

这是如何解释的？

它可以是零个或多个[a-z] `([a-z]*)`，然后是任意一个字符（`.`），然后是集合`[0-9]`中的一个字符，使其匹配`abcdeO9`。

它也可以被解释为`[a-z]`之一，然后是一个字符`*`，然后是一个字符`.`（句号），然后是一个数字，使其匹配`x*.8`。

为了克服这个问题，我们在字符前加上反斜杠“\”（这样做称为“转义字符”）。具有多重含义的字符，如*，前缀为“\”，使其成为特殊含义或使其成为非特殊含义。是否转义特殊字符或非特殊字符取决于您正在使用的工具。

# 使用 grep 在文件中搜索和挖掘“文本”

在文件中搜索是文本处理中的一个重要用例。我们可能需要通过文件中的数千行进行搜索，以便通过使用某些规范找出一些所需的数据。这个示例将帮助您学习如何从数据池中定位给定规范的数据项。

## 做好准备

`grep`命令是在文本中搜索的主要 UNIX 实用程序。它接受正则表达式和通配符。我们可以使用`grep`提供的许多有趣选项以各种格式生成输出。让我们看看如何做到这一点。

## 如何做...

在文件中搜索单词如下：

```
$ grep match_pattern filename
this is the line containing match_pattern

```

或：

```
$ grep "match_pattern" filename
this is the line containing match_pattern

```

它将返回包含给定`match_pattern`的文本行。

我们也可以按如下方式从`stdin`读取：

```
$ echo -e "this is a word\nnext line" | grep word 
this is a word

```

使用单个`grep`调用在多个文件中执行搜索如下：

```
$ grep "match_text" file1 file2 file3 ... 

```

我们可以使用`--color`选项在行中突出显示单词如下：

```
$ grep word filename –-color=auto
this is the line containing word

```

通常，`grep`命令将`match_text`视为通配符。要将正则表达式用作输入参数，应添加`-E`选项，这意味着扩展的正则表达式。或者我们可以使用启用正则表达式的`grep`命令，`egrep`。例如：

```
$ grep -E "[a-z]+"

```

或：

```
$ egrep "[a-z]+"

```

为了仅输出文件中文本的匹配部分，请使用`-o`选项如下：

```
$ echo this is a line. | grep -o -E "[a-z]+\."
line

```

或：

```
$ echo this is a line. | egrep -o "[a-z]+\."
line.

```

为了打印除包含`match_pattern`的行之外的所有行，请使用：

```
$ grep -v  match_pattern file

```

在`grep`中添加`-v`选项可以反转匹配结果。

计算文件或文本中出现匹配字符串或正则表达式的行数如下：

```
$ grep -c "text" filename
10

```

应该注意，`-c`仅计算匹配行的数量，而不是匹配的次数。例如：

```
$ echo -e "1 2 3 4\nhello\n5 6" | egrep  -c "[0-9]"
2

```

尽管有 6 个匹配项，但它打印出`2`，因为只有`2`个匹配行。单行中的多个匹配只计算一次。

为了计算文件中匹配项的数量，请使用以下技巧：

```
$ echo -e "1 2 3 4\nhello\n5 6" | egrep  -o "[0-9]" | wc -l
6

```

打印匹配字符串的行号如下：

```
$ cat sample1.txt
gnu is not unix
linux is fun
bash is art
$ cat sample2.txt
planetlinux

$ grep linux -n sample1.txt
2:linux is fun

```

或：

```
$ cat sample1.txt | grep linux -n

```

如果使用多个文件，它还将打印带有结果的文件名如下：

```
$ grep linux -n sample1.txt sample2.txt
sample1.txt:2:linux is fun
sample2.txt:2:planetlinux

```

打印模式匹配的字符或字节偏移如下：

```
$ echo gnu is not unix | grep -b -o "not"
7:not

```

行中字符串的字符偏移是从 0 开始的计数器。在上面的例子中，`not`位于第七个偏移位置（即`not`从行中的第七个字符开始（`gnu is not unix`）。

`-b`选项始终与`-o`一起使用。

要在许多文件中搜索并找出某个文本匹配的文件，请使用：

```
$ grep -l linux sample1.txt sample2.txt
sample1.txt
sample2.txt

```

`-l`参数的反义词是`-L`。`-L`参数返回一个非匹配文件的列表。

## 还有更多...

我们已经使用了`grep`命令的基本用法示例。但是`grep`命令具有丰富的功能。让我们看看`grep`提供的不同选项。

### 递归搜索多个文件

要在许多后代目录中递归搜索文本，请使用：

```
$ grep "text" . -R -n

```

在此命令中，`"."`指定当前目录。

例如：

```
$ cd src_dir
$ grep "test_function()" . -R -n
./miscutils/test.c:16:test_function();

```

`test_function()`存在于`miscutils/test.c`的第 16 行。

### 注意

这是开发人员最常用的命令之一。它用于查找源代码文件中是否存在某个文本。

### 忽略模式的大小写

`-i`参数有助于匹配模式在不考虑字符是大写还是小写的情况下进行评估。例如：

```
$ echo hello world | grep -i "HELLO"
hello

```

### 通过匹配多个模式进行 grep

通常，我们可以指定单个模式进行匹配。但是，我们可以使用`-e`参数指定多个模式进行匹配，如下所示：

```
$ grep -e "pattern1" -e "pattern"

```

例如：

```
$ echo this is a line of text | grep -e "this" -e "line" -o
this
line

```

还有另一种指定多个模式的方法。我们可以使用一个模式文件来读取模式。按行编写模式以匹配并执行`grep`，并使用`-f`参数如下：

```
$ grep -f pattern_file source_filename

```

例如：

```
$ cat pat_file
hello
cool

$ echo hello this is cool | grep -f pat_file
hello this is cool

```

### 在 grep 搜索中包括和排除文件（通配符模式）

`grep`可以包括或排除要搜索的文件。我们可以使用通配符模式指定包含文件或排除文件。

要递归地仅在目录中搜索`.c`和`.cpp`文件，并排除所有其他文件类型，请使用：

```
$ grep "main()" . -r  --include *.{c,cpp}

```

请注意，`some{string1,string2,string3}`扩展为`somestring1 somestring2 somestring3`。

排除搜索中的所有 README 文件如下：

```
$ grep "main()" . -r –-exclude "README" 

```

要排除目录，请使用`--exclude-dir`选项。

要从文件中读取要排除的文件列表，请使用`--exclude-from FILE`。

### 使用带有零字节后缀的 xargs 的 grep

`xargs`命令通常用于将文件名列表作为命令行参数提供给另一个命令。当文件名用作命令行参数时，建议使用零字节终止符而不是空格终止符。一些文件名可能包含空格字符，它将被误解为终止符，并且一个文件名可能会被分成两个文件名（例如，`New file.txt`可能被解释为两个文件名`New`和`file.txt`）。通过使用零字节后缀可以避免这个问题。我们使用`xargs`来接受来自`grep`、`find`等命令的`stdin`文本。这些命令可以输出带有零字节后缀的文本到`stdout`。为了指定文件名的输入终止符是零字节（`\0`），我们应该在`xargs`中使用`-0`。

创建一些测试文件如下：

```
$ echo "test" > file1
$ echo "cool" > file2
$ echo "test" > file3

```

在以下命令序列中，`grep`输出带有零字节终止符（`\0`）的文件名。通过使用`grep`的`-Z`选项来指定。`xargs -0`读取输入并使用零字节终止符分隔文件名：

```
$ grep "test" file* -lZ | xargs -0 rm

```

通常，`-Z`与`-l`一起使用。

### grep 的静默输出

`grep`的先前提到的用法以不同的格式返回输出。有些情况下，我们需要知道文件是否包含指定的文本。我们必须执行一个返回 true 或 false 的测试条件。可以使用安静条件（`-q`）来执行。在安静模式下，`grep`命令不会将任何输出写入标准输出。相反，它运行命令并根据成功或失败返回退出状态。

我们知道，如果成功，命令返回 0，如果失败，则返回非零。

让我们通过一个脚本，以安静模式使用`grep`来测试文件中是否出现匹配文本。

```
#!/bin/bash 
#Filename: silent_grep.sh
#Description: Testing whether a file contain a text or not 

if [ $# -ne 2 ]; 
then
echo "$0 match_text filename"
fi

match_text=$1 
filename=$2 

grep -q $match_text $filename

if [ $? -eq 0 ];
then
echo "The text exists in the file"
else
echo "Text does not exist in the file"
fi
```

可以通过提供匹配单词（`Student`）和文件名（`student_data.txt`）作为命令参数来运行`silent_grep.sh`脚本：

```
$ ./silent_grep.sh Student student_data.txt 
The text exists in the file 

```

### 打印文本匹配之前和之后的行

基于上下文的打印是`grep`的一个很好的特性。假设找到了给定匹配文本的匹配行，`grep`通常只打印匹配行。但是我们可能需要在匹配行之后的“n”行或匹配行之前的“n”行或两者之间。可以使用`grep`中的上下文行控制来执行。让我们看看如何做到这一点。

为了在匹配后打印三行，请使用`-A`选项：

```
$ seq 10 | grep 5 -A 3
5
6
7
8

```

为了在匹配之前打印三行，请使用`-B`选项：

```
$ seq 10 | grep 5 -B 3
2
3
4
5

```

要在匹配项之后和之前打印三行，请使用`-C`选项如下：

```
$ seq 10 | grep 5 -C 3
2
3
4
5
6
7
8

```

如果有多个匹配项，每个部分由一行“--”分隔：

```
$ echo -e "a\nb\nc\na\nb\nc" | grep a -A 1
a
b
--
a
b

```

# 使用 cut 按列切割文件

我们可能需要按列而不是按行切割文本。假设我们有一个包含学生报告的文本文件，其中包含`编号`、`姓名`、`成绩`和`百分比`等列。我们需要提取学生的姓名到另一个文件或文件中的任何第 n 列，或提取两列或更多列。本教程将说明如何执行此任务。

## 准备好了

`cut`是一个小型实用程序，通常用于按列切割。它还可以指定分隔每一列的分隔符。在`cut`术语中，每一列被称为一个字段。

## 如何做...

为了提取第一个字段或列，使用以下语法：

```
cut -f FIELD_LIST filename

```

`FIELD_LIST`是要显示的列的列表。该列表由逗号分隔的列号组成。例如：

```
$ cut -f 2,3 filename

```

这里显示了第二列和第三列。

`cut`还可以从`stdin`中读取输入文本。

制表符是字段或列的默认分隔符。如果找到没有分隔符的行，它们也会被打印。为了避免打印没有分隔符字符的行，请在`cut`后面加上`-s`选项。以下是使用`cut`命令进行列处理的示例：

```
$ cat student_data.txt 
No  Name     Mark   Percent
1   Sarath    45     90
2   Alex      49     98
3   Anu       45     90

$ cut -f1 student_data.txt
No 
1 
2 
3 

```

提取多个字段如下：

```
$ cut -f2,4 student_data.txt
Name     Percent
Sarath   90
Alex     98
Anu      90

```

要打印多个列，请将由逗号分隔的列号列表作为`-f`的参数。

我们还可以使用`--complement`选项来补充提取的字段。假设您有许多字段，想要打印除第三列之外的所有列，请使用：

```
$ cut -f3 –-complement student_data.txt
No  Name    Percent 
1   Sarath  90
2   Alex    98
3   Anu     90

```

要指定字段的分隔符字符，请使用`-d`选项如下：

```
$ cat delimited_data.txt
No;Name;Mark;Percent
1;Sarath;45;90
2;Alex;49;98
3;Anu;45;90

$ cut -f2 -d";" delimited_data.txt
Name
Sarath
Alex
Anu

```

## 还有更多...

`cut` 命令有更多选项来指定要显示为列的字符序列。让我们看看`cut`提供的其他选项。

### 指定字符或字节范围作为字段

假设我们不依赖分隔符，但需要提取字段，以便定义一系列字符（从 0 开始计算为行的开头）作为字段，这样的提取可以使用`cut`来实现。

让我们看看有哪些可能的符号：

| N- | 从第 N 个字节、字符或字段到行尾 |
| --- | --- |
| N-M | 从第 N 到第 M（包括）个字节、字符或字段 |
| -M | 从第一个到第 M 个（包括）字节、字符或字段 |

我们使用上述符号来指定字段作为字节或字符的范围，具有以下选项：

+   `-b` 用于字节

+   `-c` 用于字符

+   `-f` 用于定义字段

例如：

```
$ cat range_fields.txt
abcdefghijklmnopqrstuvwxyz
abcdefghijklmnopqrstuvwxyz
abcdefghijklmnopqrstuvwxyz
abcdefghijklmnopqrstuvwxy

```

您可以按以下方式打印前五个字符：

```
$ cut -c1-5 range_fields.txt
abcde
abcde
abcde
abcde

```

前两个字符可以打印如下：

```
$ cut range_fields.txt -c-2
ab
ab
ab
ab

```

将`-c`替换为`-b`以按字节计数。

在使用`-c`、`-f`和`-b`时，可以指定输出分隔符如下：

```
--output-delimiter "delimiter string"
```

使用`-b`或`-c`提取多个字段时，`--output-delimiter`是必须的。否则，如果未提供它，您无法区分字段。例如：

```
$ cut range_fields.txt -c1-3,6-9 --output-delimiter ","
abc,fghi
abc,fghi
abc,fghi
abc,fghi

```

# 给定文件中使用的单词频率

查找文件中使用的单词频率是一个有趣的练习，可以应用文本处理技能。有很多不同的方法可以做到这一点。让我们看看如何做到这一点。

## 准备好了

我们可以使用关联数组、awk、sed、grep 等不同的方式来解决这个问题。

## 如何做...

单词是由空格和句点分隔的字母字符。首先，我们应该解析给定文件中的所有单词。因此，需要找出每个单词的计数。可以使用正则表达式和诸如 sed、awk 或 grep 之类的工具来解析单词。

要找出每个单词的计数，我们可以采用不同的方法。一种方法是循环遍历每个单词，然后使用另一个循环遍历单词并检查它们是否相等。如果它们相等，增加一个计数并在文件末尾打印它。这是一种低效的方法。在关联数组中，我们使用单词作为数组索引，计数作为数组值。我们只需要一个循环就可以通过循环遍历每个单词来实现这一点。`array[word] = array[word] + 1`，而最初它的值被设置为`0`。因此，我们可以得到一个包含每个单词计数的数组。

现在让我们来做吧。创建如下的 shell 脚本：

```
#!/bin/bash
#Name: word_freq.sh
#Description: Find out frequency of words in a file

if [ $# -ne 1 ];
then
echo "Usage: $0 filename";
exit -1
fi

filename=$1

egrep -o "\b[[:alpha:]]+\b" $filename | \

awk '{ count[$0]++ }
END{ printf("%-14s%s\n","Word","Count") ;
for(ind in count)
{  printf("%-14s%d\n",ind,count[ind]);  }

}'
```

一个示例输出如下：

```
$ ./word_freq.sh words.txt 
Word          Count 
used           1
this           2 
counting       1

```

## 它是如何工作的...

这里使用`egrep -o "\b[[:alpha:]]+\b" $filename`来仅输出单词。`-o`选项将打印由换行字符分隔的匹配字符序列。因此我们在每行收到单词。

`\b`是单词边界字符。`[:alpha:]`是字母的字符类。

`awk`命令用于避免对每个单词进行迭代。由于`awk`默认情况下执行`{}`块中的语句对每一行，我们不需要特定的循环来执行。因此，使用关联数组递增计数为`count[$0]++`。最后，在`END{}`块中，我们通过迭代单词来打印单词及其计数。

## 另请参阅

+   *数组和关联数组*第一章，解释了 Bash 中的数组

+   *基本的 awk 入门*，解释了 awk 命令

# 基本的 sed 入门

sed 代表流编辑器。这是文本处理的一个非常重要的工具。它是一个可以玩弄正则表达式的神奇实用程序。`sed`命令的一个众所周知的用法是文本替换。本教程将涵盖大多数经常使用的`sed`技术。

## 如何做…

`sed`可以用于在给定文本中用另一个字符串替换字符串的出现。它可以使用正则表达式进行匹配。

```
$ sed 's/pattern/replace_string/' file

```

或者

```
$ cat file | sed 's/pattern/replace_string/' file

```

这个命令从`stdin`中读取。

要将更改与替换保存到同一文件中，请使用-i 选项。大多数用户在进行替换后使用多个重定向来保存文件，如下所示：

```
$ sed 's/text/replace/' file > newfile
$ mv newfile file

```

然而，它可以在一行内完成，例如：

```
$ sed -i 's/text/replace/' file

```

先前看到的`sed`命令将替换每行中模式的第一个出现。但是为了替换每个出现，我们需要在末尾添加`g`参数，如下所示：

```
$ sed 's/pattern/replace_string/g' file

```

`/g`后缀表示它将替换每个出现。然而，有时我们不需要替换前 N 个出现，而只需要其余的。有一个内置选项可以忽略前 N 个出现并从第"N+1"次出现开始替换。

看看以下命令：

```
$ echo this thisthisthis | sed 's/this/THIS/2g' 
thisTHISTHISTHIS

$ echo this thisthisthis | sed 's/this/THIS/3g' 
thisthisTHISTHIS

$ echo this thisthisthis | sed 's/this/THIS/4g' 
thisthisthisTHIS

```

在需要从第 N 次出现开始替换时，放置`/Ng`。

`/`在`sed`中是一个分隔符字符。我们可以使用任何分隔符字符，如下所示：

```
sed 's:text:replace:g'
sed 's|text|replace|g'

```

当分隔符字符出现在模式内部时，我们必须使用`\`前缀进行转义，如下所示：

```
sed 's|te\|xt|replace|g'

```

`\|`是出现在替换中的分隔符。

## 还有更多...

`sed`命令具有许多用于文本处理的选项。通过将`sed`中可用的选项与逻辑序列结合，可以在一行中解决许多复杂的问题。让我们看看`sed`可用的一些不同选项。

### 删除空白行

使用`sed`删除空白行是一种简单的技术。空白可以与正则表达式`^$`匹配：

```
$ sed '/^$/d' file

```

`/pattern/d`将删除匹配模式的行。

对于空白行，行结束标记出现在行开始标记旁边。

### 匹配字符串符号（&）

在`sed`中，我们可以使用`&`作为替换模式的匹配字符串，以便我们可以在替换字符串中使用匹配的字符串。

例如：

```
$ echo this is an example | sed 's/\w\+/[&]/g'
[this] [is] [an] [example]

```

这里的正则表达式`\w\+`匹配每个单词。然后我们用`[&]`替换它。`&`对应于匹配的单词。

### 子字符串匹配符号（\1）

&是一个字符串，对应于给定模式的匹配字符串。但我们也可以匹配给定模式的子字符串。让我们看看如何做到这一点。

```
$ echo this is digit 7 in a number | sed 's/digit \([0-9]\)/\1/'
this is 7 in a number

```

它用`\(pattern\)`替换`digit 7`为`7`。匹配的子字符串是`7`。`()`中的模式用斜杠转义。对于第一个子字符串匹配，相应的表示是`\1`，对于第二个是`\2`，依此类推。看下面的多次匹配示例：

```
$ echo seven EIGHT | sed 's/\([a-z]\+\) \([A-Z]\+\)/\2 \1/'
EIGHT seven

```

`([a-z]\+\)`匹配第一个单词，`\([A-Z]\+\)`匹配第二个单词。`\1`和`\2`用于引用它们。这种引用方式称为回溯引用。在替换部分，它们的顺序被改变为`\2 \1`，因此它们以相反的顺序出现。

### 多个表达式的组合

使用管道替换多个`sed`的组合如下：

```
sed 'expression' | sed 'expression'
```

相当于：

```
$ sed 'expression; expression'
```

### 引用

通常情况下，`sed`表达式使用单引号引起来。但也可以使用双引号。双引号通过评估来扩展表达式。当我们想在`sed`表达式中使用某个变量字符串时，使用双引号是有用的。

例如：

```
$ text=hello
$ echo hello world | sed "s/$text/HELLO/" 
HELLO world 

```

`$text`被评估为"hello"。

# 基本 awk 入门

`awk`是一种设计用于处理数据流的工具。它非常有趣，因为它可以操作列和行。它支持许多内置功能，如数组、函数等，就像 C 编程语言一样。灵活性是它的最大优势。

## 如何做…

`awk`脚本的结构如下：

```
awk ' BEGIN{  print "start" } pattern { commands } END{ print "end" } file

```

`awk`命令也可以从`stdin`读取。

`awk`脚本通常由三部分组成：`BEGIN`、`END`和带有模式匹配选项的常用语句块。它们三者都是可选的，脚本中任何一个都可以不存在。脚本通常用单引号或双引号括起来，如下所示：

```
awk 'BEGIN { statements } { statements } END { end statements }'

```

或者，也可以使用：

```
awk "BEGIN { statements } { statements } END { end statements }"

```

例如：

```
$ awk 'BEGIN { i=0 } { i++ } END{ print i}' filename

```

或者：

```
$ awk "BEGIN { i=0 } { i++ } END{ print i }" filename

```

## 它是如何工作的…

`awk`命令的工作方式如下：

1.  执行`BEGIN { commands }`块中的语句。

1.  从文件或`stdin`读取一行，并执行`pattern { commands }`。重复此步骤，直到到达文件的末尾。

1.  当到达输入流的末尾时，执行`END { commands }`块。

`BEGIN`块在`awk`开始从输入流中读取行之前执行。这是一个可选块。在`BEGIN`块中编写的常见语句包括变量初始化、为输出表打印输出标题等。

`END`块类似于`BEGIN`块。当`awk`完成从输入流中读取所有行时，`END`块会被执行。在`END`块中，打印所有行的计算值后进行分析或打印结论等语句是常用的语句（例如，在比较所有行后，打印文件中的最大数）。这是一个可选块。

最重要的块是带有模式块的常用命令。这个块也是可选的。如果没有提供这个块，默认情况下会执行`{ print }`以打印读取的每一行。这个块对`awk`读取的每一行都会执行。

它就像一个 while 循环，用提供的语句在循环体内读取行。

读取一行，检查提供的模式是否与该行匹配。模式可以是正则表达式匹配、条件、行范围匹配等。如果当前读取的行与模式匹配，它会执行`{ }`中的语句。

模式是可选的。如果不使用模式，所有行都会匹配，并且执行`{ }`内的语句。

让我们看下面的例子：

```
$ echo -e "line1\nline2" | awk 'BEGIN{ print "Start" } { print } END{ print "End" } '
Start
line1
line2
End

```

当使用`print`而没有参数时，它将打印当前行。关于`print`有两件重要的事情需要记住。当 print 的参数用逗号分隔时，它们将以空格分隔打印。双引号在`awk`的`print`上下文中用作连接运算符。

例如：

```
$ echo | awk '{ var1="v1"; var2="v2"; var3="v3"; \
print var1,var2,var3 ; }'

```

上述语句将打印变量的值如下：

```
v1 v2 v3

```

`echo`命令将一行写入标准输出。因此，`awk`的`{ }`块中的语句将执行一次。如果`awk`的标准输入包含多行，则`awk`中的命令将执行多次。

连接可以如下使用：

```
$ echo | awk '{ var1="v1"; var2="v2"; var3="v3"; \
print var1"-"var2"-"var3 ; }'

```

输出将是：

```
v1-v2-v3

```

`{ }`就像循环中的块，遍历文件的每一行。

### 提示

通常，我们将初始变量赋值，比如`var=0;`和打印文件头的语句放在`BEGIN`块中。在`END{}`块中，我们放置打印结果等语句。

## 还有更多…

`awk`命令具有许多丰富的功能。为了掌握 awk 编程的艺术，您应该熟悉重要的`awk`选项和功能。让我们了解`awk`的基本功能。

### 特殊变量

可以使用的一些特殊变量与`awk`一起使用如下：

+   `NR`：它代表记录数，并对应于当前执行的行号。

+   `NF`：它代表字段数，并对应于当前执行行下的字段数（字段由空格分隔）。

+   `$0`：它是一个变量，包含当前执行行的文本内容。

+   `$1`：它是一个变量，保存第一个字段的文本。

+   `$2`：它是保存第二个字段文本的变量。

例如：

```
$ echo -e "line1 f2 f3\nline2 f4 f5\nline3 f6 f7" | \

awk '{
print "Line no:"NR",No of fields:"NF, "$0="$0, "$1="$1,"$2="$2,"$3="$3 
}' 
Line no:1,No of fields:3 $0=line1 f2 f3 $1=line1 $2=f2 $3=f3 
Line no:2,No of fields:3 $0=line2 f4 f5 $1=line2 $2=f4 $3=f5 
Line no:3,No of fields:3 $0=line3 f6 f7 $1=line3 $2=f6 $3=f7

```

我们可以将一行的最后一个字段打印为`print $NF`，倒数第二个字段为`$(NF-1)`等等。

`awk`提供了与 C 中相同语法的`printf()`函数。我们也可以使用它来代替 print。

让我们看一些基本的`awk`使用示例。

按如下方式打印每一行的第二个和第三个字段：

```
$awk '{ print $3,$2 }'  file

```

为了计算文件中的行数，使用以下命令：

```
$ awk 'END{ print NR }' file

```

这里我们只使用`END`块。`NR`将在进入每一行时由`awk`更新其行号。当它到达最后一行时，它将具有最后一行号的值。因此，在`END`块中，`NR`将具有最后一行号的值。

您可以将字段 1 的每一行的所有数字相加如下：

```
$ seq 5 | awk 'BEGIN{ sum=0; print "Summation:" } 
{ print $1"+"; sum+=$1 } END { print "=="; print sum }' 
Summation: 
1+ 
2+ 
3+ 
4+ 
5+ 
==
15

```

### 将变量值从外部传递给 awk

通过使用`-v`参数，我们可以将外部值（而不是来自`stdin`）传递给`awk`如下：

```
$ VAR=10000
$ echo | awk -v VARIABLE=$VAR'{ print VARIABLE }'
1

```

有一种灵活的替代方法可以从外部传递多个变量值给`awk`。例如：

```
$ var1="Variable1" ; var2="Variable2"
$ echo | awk '{ print v1,v2 }' v1=$var1 v2=$var2
Variable1 Variable2

```

当输入是通过文件而不是标准输入给出时，使用：

```
$ awk '{ print v1,v2 }' v1=$var1 v2=$var2 filename

```

在上述方法中，变量被指定为键值对，由空格分隔（`v1=$var1 v2=$var2`）作为命令参数传递给`awk`的 BEGIN、{ }和 END 块之后。

### 使用 getline 显式读取一行

通常，`grep`默认读取文件中的所有行。如果要读取特定行，可以使用`getline`函数。有时我们可能需要从`BEGIN`块中读取第一行。

语法是：`getline var`

变量`var`将包含该行的内容。

如果`getline`没有参数调用，我们可以使用`$0`，`$1`和`$2`来访问行的内容。

例如：

```
$ seq 5 | awk 'BEGIN { getline; print "Read ahead first line", $0 } { print $0 }'
Read ahead first line 1
2
3
4
5

```

### 使用过滤模式过滤由 awk 处理的行

我们可以为要处理的行指定一些条件。例如：

```
$ awk 'NR < 5' # Line number less than 5
$ awk 'NR==1,NR==4' #Line numbers from 1-5
$ awk '/linux/' # Lines containing the pattern linux (we can specify regex)
$ awk '!/linux/' # Lines not containing the pattern linux

```

### 设置字段的分隔符

默认情况下，字段的分隔符是空格。我们可以使用`-F "delimiter"`来明确指定分隔符：

```
$ awk -F: '{ print $NF }' /etc/passwd

```

或：

```
awk 'BEGIN { FS=":" } { print $NF }' /etc/passwd

```

我们可以通过在`BEGIN`块中设置`OFS="delimiter"`来设置输出字段分隔符。

### 从 awk 读取命令输出

在以下代码中，`echo`将产生一行空行。`cmdout`变量将包含`grep root /etc/passwd`命令的输出，并打印包含`root`的行：

在变量'output'中读取'command'的语法如下：

```
"command" | getline output ;
```

例如：

```
$ echo | awk '{ "grep root /etc/passwd" | getline cmdout ; print cmdout }'
root:x:0:0:root:/root:/bin/bash

```

通过使用`getline`，我们可以将外部 shell 命令的输出读入名为`cmdout`的变量中。

`awk`支持关联数组，可以使用文本作为索引。

### 在 awk 中使用循环

`awk`中有一个`for`循环。它的格式是：

```
for(i=0;i<10;i++) { print $i ; }
```

或者：

```
for(i in array) { print array[i]; }
```

`awk`带有许多内置的字符串操作函数。让我们来看看其中的一些：

+   `length(string)`: 它返回字符串的长度。

+   `index(string, search_string)`: 它返回`search_string`在字符串中的位置。

+   `split(string, array, delimiter)`: 它将使用分隔符生成的字符串列表存储在数组中。

+   `substr(string, start-position, end-position)`: 它返回通过使用起始和结束字符偏移创建的子字符串。

+   `sub(regex, replacement_str, string)`: 它用`replacment_str`替换字符串中第一次出现的正则表达式匹配。

+   `gsub(regex, replacment_str, string)`: 它类似于`sub()`。但它替换每一个正则表达式匹配。

+   `match(regex, string)`: 它返回正则表达式（regex）是否在字符串中找到匹配的结果。如果找到匹配，则返回非零，否则返回零。`match()`关联有两个特殊变量。它们是`RSTART`和`RLENGTH`。`RSTART`变量包含正则表达式匹配开始的位置。`RLENGTH`变量包含正则表达式匹配的字符串的长度。

# 从文本或文件中替换字符串

字符串替换是一个经常使用的文本处理任务。通过匹配所需的文本，可以很容易地使用正则表达式来完成。

## 准备就绪

当我们听到“替换”这个术语时，每个系统管理员都会想起 sed。 `sed`是 UNIX-like 系统下进行文本或文件替换的通用工具。让我们看看如何做到这一点。

## 如何做...

`sed`入门配方包含了大部分`sed`的用法。您可以按以下方式替换字符串或模式：

```
$ sed 's/PATTERN/replace_text/g' filename

```

或者：

```
$ stdin | sed 's/PATTERN/replace_text/g'

```

我们也可以使用双引号（"）而不是单引号（'）。当使用双引号（"）时，我们可以在`sed`模式和替换字符串中指定变量。例如：

```
$ p=pattern
$ r=replaced
$ echo "line containing apattern" | sed "s/$p/$r/g" 
line containing a replaced

```

我们也可以在`sed`中不使用`g`。

```
$ sed 's/PATTEN/replace_text/' filename

```

然后它将只替换`PATTERN`第一次出现的情况。`/g`代表全局。这意味着它将替换文件中`PATTERN`的每一个出现。

## 还有更多...

我们已经看到了使用`sed`进行基本文本替换。让我们看看如何将替换后的文本保存在源文件中。

### 将替换保存在文件中

当将文件名传递给`sed`时，它的输出将可用于`stdout`。为了将更改保存在文件中，而不是将输出流发送到`stdout`，请使用以下`-i`选项：

```
$ sed 's/PATTERN/replacement/' -i filename

```

例如，用以下方法在文件中替换所有三位数为另一个指定的数字：

```
$ cat sed_data.txt
11 abc 111 this 9 file contains 111 11 88 numbers 0000

$ cat sed_data.txt  | sed 's/\b[0-9]\{3\}\b/NUMBER/g'
11 abc NUMBER this 9 file contains NUMBER 11 88 numbers 0000

```

上面的一行只替换三位数。`\b[0-9]\{3\}\b`是用于匹配三位数的正则表达式。[0-9]是数字的范围，即从 0 到 9。{3}用于匹配前面的字符三次。`\`在`\{3\}`中用于给`{`和`}`赋予特殊含义。`\b`是单词边界标记。

## 参见

+   *基本的 sed 入门*，解释了 sed 命令

# 压缩或解压 JavaScript

JavaScript 在设计网站时被广泛使用。在编写 JavaScript 代码时，我们使用多个空格、注释和制表符来提高代码的可读性和维护性。但是在 JavaScript 中使用大量空格和制表符会导致文件大小增加。随着文件大小的增加，页面加载时间也会增加。因此，大多数专业网站都使用压缩的 JavaScript 来实现快速加载。压缩主要是挤压空格和换行字符。一旦 JavaScript 被压缩，可以通过添加足够的空格和换行字符来解压缩，从而使其可读。通常，混淆的代码也可以通过插入空格和换行符来实现可读。这个方法是在 shell 中尝试窃取类似功能的一种尝试。

## 准备工作

我们将编写一个 JavaScript 压缩器或混淆工具。也可以设计一个解压工具。我们将使用文本和字符替换工具`tr`和`sed`。让我们看看如何做到这一点。

## 如何做...

让我们按照逻辑顺序和所需的代码来压缩和解压 JavaScript。

```
$ cat sample.js
functionsign_out()
{ 

$("#loading").show(); 
$.get("log_in",{logout:"True"},

function(){ 

window.location="";

}); 

}

```

我们需要执行以下任务来压缩 JavaScript：

1.  删除换行符和制表符。

1.  挤压空格。

1.  替换注释/*内容*/。

1.  用替换替换以下内容：

+   将"{ "替换为"{"

+   " }"替换为"}"

+   " ("替换为"("

+   ") "替换为")"

+   ", "替换为","

+   " ; "替换为";"（我们需要删除所有额外的空格）

要解压缩或使 JavaScript 更易读，我们可以使用以下任务：

1.  将";"替换为";\n"。

1.  将"{"替换为"{\n"，将"}"替换为"\n}"。

## 它是如何工作的...

让我们通过执行以下任务来压缩 JavaScript：

1.  删除'\n'和'\t'字符：

```
tr -d '\n\t' 
```

1.  删除额外的空格：

```
tr -s ' ' or sed 's/[ ]\+/ /g'
```

1.  删除注释：

```
sed 's:/\*.*\*/::g'
```

+   `:`被用作 sed 分隔符，以避免需要转义`/`，因为我们需要使用`/*`和`*/`

+   在 sed 中，`*`被转义为`\*`

+   `.*`用于匹配`/*`和`*/`之间的所有文本

1.  删除所有在`{`、`}`、`(`、`)`、`;`、`:`和逗号之前和之后的空格。

```
sed 's/ \?\([{}();,:]\) \?/\1/g'
```

上述`sed`语句可以解析如下：

+   `/ \?\([{}();,:]\) \?/`在`sed`代码中是匹配部分，`/\1 /g`是替换部分。

+   `\([{}();,:]\)`用于匹配集合`[ { }( ) ; , : ]`中的任意一个字符（为了可读性插入了空格）。`\(`和`\)`是用于在替换部分中记忆匹配和回溯引用的组操作符。`(`和`)`被转义以赋予它们作为组操作符的特殊含义。`\?`在组操作符之前和之后。它是为了匹配可能在集合中的任何字符之前或之后的空格字符。

+   在替换部分，匹配字符串（即`:`一个空格（可选）、来自集合的字符，再次是可选空格）被替换为匹配的字符。它使用了一个回溯引用来匹配和记忆使用组操作符`()`的字符。通过使用`\1`符号，回溯引用的字符指的是组匹配。

使用管道结合上述任务如下：

```
$ catsample.js |  \
tr -d '\n\t' |  tr -s ' ' \
| sed 's:/\*.*\*/::g' \
| sed 's/ \?\([{}();,:]\) \?/\1/g' 

```

输出如下：

```
functionsign_out(){$("#loading").show();$.get("log_in",{logout:"True"},function(){window.location="";});}

```

让我们编写一个解压缩脚本，使混淆的代码可读，如下所示：

```
$ cat obfuscated.txt | sed 's/;/;\n/g; s/{/{\n\n/g; s/}/\n\n}/g' 

```

或：

```
$ cat obfuscated.txt | sed 's/;/;\n/g' | sed 's/{/{\n\n/g' | sed 's/}/\n\n}/g'

```

在上一个命令中：

+   `s/;/;\n/g`将`;`替换为`\n;`

+   `s/{/{\n\n/g`将`{`替换为`{\n\n`

+   `s/}/\n\n}/g`将`}`替换为`\n\n}`

## 另请参阅

+   *使用 tr 进行翻译* 第二章 ，解释了 tr 命令

+   *基本 sed 入门*，解释了 sed 命令

# 在文件中迭代行、单词和字符

在编写不同的文本处理和文件操作脚本时，经常需要对文件中的字符、单词和行进行迭代。尽管这很简单，但我们会犯一些错误，而且没有得到预期的输出。这个方法将帮助你学会如何做到这一点。

## 准备工作

使用简单循环和从`stdin`或文件重定向进行迭代是执行上述任务的基本组件。

## 如何做...

在这个配方中，我们讨论了执行遍历行、单词和字符的三个任务。让我们看看如何执行这些任务中的每一个。

1.  **遍历文件中的每一行：**

我们可以使用`while`循环从标准输入中读取。因此，它将在每次迭代中读取一行。

使用文件重定向到`stdin`如下：

```
while read line;
do
echo $line;
done < file.txt
```

如下使用子 shell：

```
cat file.txt | (  while read line; do echo $line; done )
```

这里`cat file.txt`可以替换为任何命令序列的输出。

1.  **遍历每个单词中的每个单词**

我们可以使用`while`循环来遍历行中的单词，如下所示：

```
for word in $line;
do
echo $word;
done

```

1.  **遍历单词中的每个字符**

我们可以使用`for`循环来迭代变量`i`从`0`到字符串的长度。在每次迭代中，可以使用特殊符号`${string:start_position:No_of_characters}`从字符串中提取一个字符。

```
for((i=0;i<${#word};i++))
do
echo ${word:i:1} ;
done

```

## 工作原理…

读取文件的行和读取行中的单词是直接的方法。但是读取单词的字符有点技巧。我们使用子字符串提取技术。

`${word:start_position:no_of_characters}`返回变量`word`中字符串的子字符串。

`${#word}`返回变量`word`的长度。

## 另请参阅

+   *字段分隔符和迭代器* 第一章，解释 Bash 中的不同循环。

+   *文本切片和参数操作*，解释从字符串中提取字符。

# 合并多个文件作为列

有不同的情况需要我们在列中连接文件。我们可能需要使每个文件的内容出现在单独的列中。通常，`cat`命令以行或行的方式连接。

## 如何做…

`paste`是可用于按列连接的命令。`paste`命令可使用以下语法：

`$ paste file1 file2 file3 …`

让我们尝试以下示例：

```
$ cat paste1.txt
1
2
3
4
5
$ cat paste2.txt
slynux
gnu
bash
hack
$ paste paste1.txt paste2.txt
1slynux
2gnu
3bash
4hack
5

```

默认分隔符是制表符。我们也可以使用`-d`显式指定分隔符。例如：

```
$ paste paste1.txt paste2.txt -d ","
1,slynux
2,gnu
3,bash
4,hack
5,

```

## 另请参阅

+   *使用 cut 按列切割文件*，解释从文本文件中提取数据

# 在文件或行中打印第 n 个单词或列

我们可能会得到一个具有许多列的文件，实际上只有少数列是有用的。为了仅打印相关列或字段，我们对其进行过滤。

## 准备工作

最广泛使用的方法是使用`awk`来执行此任务。也可以使用`cut`来完成。

## 如何做…

要打印第五列，请使用以下命令：

```
$ awk '{ print $5 }' filename

```

我们还可以打印多个列，并且可以在列之间插入自定义字符串。

例如，要打印当前目录中每个文件的权限和文件名，请使用：

```
$ ls -l | awk '{ print $1" :  " $8 }'
-rw-r--r-- :  delimited_data.txt
-rw-r--r-- :  obfuscated.txt
-rw-r--r-- :  paste1.txt
-rw-r--r-- :  paste2.txt

```

## 另请参阅

+   *基本 awk 入门*，解释 awk 命令

+   *使用 cut 按列切割文件*，解释从文本文件中提取数据

# 打印行号或模式之间的文本

我们可能需要根据条件打印文本行的某些部分，例如行号范围，开始和结束模式匹配的范围等。让我们看看如何做到这一点。

## 准备工作

我们可以使用诸如 awk、grep 和 sed 之类的实用程序根据条件执行部分打印。但我发现`awk`是最容易理解的。让我们使用`awk`来做。

## 如何做…

为了打印文本行的行号范围，从 M 到 N，使用以下语法：

```
$ awk 'NR==M, NR==N' filename

```

或者，它可以接受`stdin`输入如下：

```
$ cat filename | awk 'NR==M, NR==N'

```

将`M`和`N`替换为以下数字：

```
$ seq 100 | awk 'NR==4,NR==6'
4
5
6

```

要打印文本部分的行，使用以下语法：`start_pattern`和`end_pattern`。

```
$ awk '/start_pattern/, /end _pattern/' filename

```

例如：

```
$ cat section.txt 
line with pattern1 
line with pattern2 
line with pattern3 
line end with pattern4 
line with pattern5 

$ awk '/pa.*3/, /end/' section.txt 
line with pattern3 
line end with pattern4

```

`awk`中使用的模式是正则表达式。

## 另请参阅

+   *基本 awk 入门*，解释 awk 命令

# 使用脚本检查回文字符串

检查字符串是否回文是 C 编程课程中的第一个实验。但是，在这里，我们包含了这个配方，以便让您了解如何解决类似的问题，其中模式匹配可以扩展为以前出现的模式在文本中重复。

## 准备工作

`sed`命令具有记住先前匹配的子模式的能力。这被称为反向引用。我们可以通过使用反向引用来解决回文问题。我们可以在 Bash 中使用多种方法来解决这个问题。

## 如何做...

`sed`可以记住先前匹配的正则表达式模式，因此我们可以确定字符串中是否存在字符的重复。这种记住和引用先前匹配模式的能力称为反向引用。

让我们看看如何以更简单的方式应用反向引用来解决问题。例如：

```
$ sed -n '/\(.\)\1/p' filename

```

`\(.\)`对应于记住( )内的一个子字符串。这里是 . (句号)，它也是`sed`的单个字符通配符。

`\1`对应于`()`内的第一个匹配的记忆。`\2`对应于第二个匹配。因此，我们可以记住许多被`()`包围的块。`()`显示为`\( \)`，以赋予`(`和`)`特殊含义，而不仅仅是一个字符。

前面的`sed`语句将打印任何匹配两个完全相同的模式。

所有回文单词的结构如下：

+   偶数个字符和一个字符序列，与其反向序列连接

+   奇数个字符，带有字符序列，与相同字符的反向连接，但在第一个序列和其反向之间有一个公共字符

因此，为了匹配两者，我们可以在写正则表达式时在中间保留一个可选字符。

匹配三个字母回文单词的`sed`正则表达式将如下所示：

```
'/\(.\).\1/p'
```

我们可以在字符序列和其反向序列之间放置一个额外的字符(`.`)。

让我们编写一个可以匹配任意长度的回文字符串的脚本，如下所示：

```
#!/bin/bash
#Filename: match_palindrome.sh
#Description: Find out palindrome strings from a given file

if [ $# -ne 2 ];
then
echo "Usage: $0 filename string_length"
exit -1
fi

filename=$1 ;

basepattern='/^\(.\)'

count=$(( $2 / 2 ))

for((i=1;i<$count;i++))
do
basepattern=$basepattern'\(.\)' ;
done

if [ $(( $2 % 2 )) -ne 0 ];
then
basepattern=$basepattern'.' ;
fi

for((count;count>0;count--))
do
basepattern=$basepattern'\'"$count" ;
done

basepattern=$basepattern'$/p'
sed -n "$basepattern" $filename
```

使用字典文件作为输入文件，以获取给定字符串长度的回文单词列表。例如：

```
$ ./match_palindrome.sh /usr/share/dict/british-english 4
noon
peep
poop
sees

```

## 它是如何工作的...

上述脚本的工作很简单。大部分工作是为正则表达式和反向引用字符串生成`sed`脚本。

让我们通过一些示例来看看它的工作原理。

+   如果要匹配字符并进行反向引用，我们使用`\(.\)`来匹配一个字符，`\1`来引用它。因此，为了匹配并打印两个字母的回文，我们使用：

```
sed '/\(.\)\1/p'
```

现在，为了指定从行的开头匹配字符串，我们添加行开始标记^，这样它将变成`sed'/^\(.\)\1/p'`。`/p`用于打印匹配。

+   如果我们想匹配四个字符的回文，我们使用：

```
sed '/^\(.\)\(.\)\2\1/p'
```

我们使用了两个`\(.\)`来匹配两个字符并记住它们。任何在`\(`和`\)`之间的内容都将被`sed`记住并可以被反向引用。`\2\1`用于以匹配字符的相反顺序进行反向引用。

在上面的脚本中，我们有一个名为`basepattern`的变量，其中包含`sed`脚本。

该模式是根据回文字符串中的字符数使用`for`循环生成的。

最初，`basepattern`被初始化为`basepattern='/^\(.\)'`，它对应于一个字符匹配。使用`for`循环将`\(.\)`与`basepattern`连接起来，连接次数为回文字符串长度的一半。再次使用`for`循环以与回文字符串长度的一半相同的次数连接反向引用。最后，为了支持奇数长度的回文字符串，在匹配正则表达式和反向引用之间加入了一个可选字符(`.`)。

因此，`sed`回文匹配模式是精心制作的。这个精心制作的字符串用于从字典文件中找出回文字符串。

在上面的脚本中，我们使用了`for`循环生成`sed`模式。实际上没有必要单独生成模式。`sed`命令有自己的循环实现，使用标签和 goto。`sed`是一种广泛的语言。可以使用复杂的`sed`脚本在一行中进行回文检查。很难从头开始解释它。只需尝试以下脚本：

```
$ word="malayalam"
$ echo $word | sed ':loop ; s/^\(.\)\(.*\)\1/\2/; t loop; /^.\?$/{ s/.*/PALINDROME/ ; q; };  s/.*/NOT PALINDROME/ '
PALINDROME

```

如果您对使用`sed`进行深入脚本编写感兴趣，请参考完整的`sed`和`awk`参考书：*sed & awk*，作者 Dale Dougherty 和 Arnold Robbins 的第二版。

尝试解析上面的一行`sed`脚本以测试回文是否使用该书。

## 还有更多...

现在让我们看看其他选项，或者可能与此任务相关的一些一般信息片段。

### 最简单直接的方法

检查字符串是否是回文的最简单方法是使用 rev 命令。

`rev`命令接受文件或`stdin`作为输入，并打印每行的反转字符串。

让我们来做一下：

```
string="malayalam"
if [[ "$string" == "$(echo $string | rev )" ]];
then
echo "Palindrome"
else
echo "Not palindrome"
fi
```

`rev`命令可以与其他命令一起使用来解决不同的问题。让我们看一个有趣的例子，将句子中的单词反转：

```
sentence='this is line from sentence'
echo $sentence | rev | tr ' ' '\n' | tac | tr '\n' ' ' | rev

```

输出如下：

```
sentence from line is this

```

在上面的一行代码中，首先使用`rev`命令反转字符。然后通过使用`tr`命令将空格替换为`\n`字符，将单词分隔成每行一个单词。现在使用`tac`命令按顺序反转行。再次使用`tr`将行合并为一行。现在再次应用`rev`，使得带有单词的行以相反的顺序排列。

## 另请参阅

+   *基本的 sed 入门*，解释了 sed 命令

+   *比较和测试* 第一章的[比较和测试]，解释了字符串比较运算符

# 以相反的顺序打印行

这是一个简单的方法。它可能看起来并不是很有用，但它可以用来模拟 Bash 中的堆栈数据结构。这是一些有趣的东西。让我们以相反的顺序打印文件中的文本行。

## 准备工作

使用`awk`的小技巧可以完成任务。但是，也有一个直接的命令`tac`可以做同样的事情。`tac`是`cat`的反向。

## 如何做...

首先让我们用`tac`来做。语法如下：

```
tac file1 file2 …

```

它也可以按以下方式从`stdin`读取：

```
$ seq 5 | tac
5 
4 
3 
2 
1

```

在`tac`中，`\n`是行分隔符。但我们也可以使用`-s` "separator"选项指定自己的分隔符。

让我们用`awk`来做：

```
$ seq 9 | \
awk '{ lifo[NR]=$0; lno=NR } 
END{ for(;lno>-1;lno--){ print lifo[lno]; } 
}'

```

在 shell 脚本中，`\`用于方便地将单行命令序列分解为多行。

## 它是如何工作的...

`awk`脚本非常简单。我们将每行存储到一个关联数组中，行号作为数组索引（NR 给出行号）。最后，`awk`执行`END`块。为了获得最后一行行号，`lno=NR`在{ }块中使用。因此，它从最后一行号迭代到`0`，并以相反的顺序打印数组中存储的行。

## 另请参阅

+   *使用 awk 实现 head、tail 和 tac*，解释了使用 awk 编写 tac

# 从文本中解析电子邮件地址和 URL

从给定文件中解析所需的文本是我们在文本处理中经常遇到的常见任务。通过正确的正则表达式序列可以找到诸如电子邮件、URL 等项目。大多数情况下，我们需要从由许多不需要的字符和单词组成的电子邮件客户端的联系人列表或 HTML 网页中解析电子邮件地址。

## 准备工作

这个问题可以用 egrep 工具解决。

## 如何做...

匹配电子邮件地址的正则表达式模式是：

egrep 正则表达式：`[A-Za-z0-9.]+@[A-Za-z0-9.]+\.[a-zA-Z]{2,4}`

例如：

```
$ cat url_email.txt 
this is a line of text contains,<email> #slynux@slynux.com. </email> and email address, blog "http://www.google.com", test@yahoo.com dfdfdfdddfdf;cool.hacks@gmail.com<br />
<ahref="http://code.google.com"><h1>Heading</h1>

$ egrep -o '[A-Za-z0-9.]+@[A-Za-z0-9.]+\.[a-zA-Z]{2,4}'  url_email.txt
slynux@slynux.com 
test@yahoo.com 
cool.hacks@gmail.com

```

用于 HTTP URL 的`egrep regex`模式是：

```
http://[a-zA-Z0-9\-\.]+\.[a-zA-Z]{2,4}

```

例如：

```
$ egrep -o "http://[a-zA-Z0-9.]+\.[a-zA-Z]{2,3}" url_email.txt
http://www.google.com 
http://code.google.com

```

## 它是如何工作的...

设计正则表达式真的很容易。在电子邮件正则表达式中，我们都知道电子邮件地址的形式是`name@domain.some_2-4_letter`。在这里，相同的内容以正则表达式语言写成如下：

```
[A-Za-z0-9.]+@[A-Za-z0-9.]+\.[a-zA-Z]{2,4}

```

`[A-Za-z0-9.]+`表示在`[]`块中的一些字符组合应该在字面`@`字符出现之前出现一次或多次（这是`+`的含义）。然后`[A-Za-z0-9.]`也应该出现一次或多次（`+`）。模式`\.`表示应该出现一个字面上的句号，最后一部分应该是长度为 2 到 4 个字母字符。

HTTP URL 的情况类似于电子邮件地址，但没有电子邮件正则表达式的`name@`匹配部分。

```
http://[a-zA-Z0-9.]+\.[a-zA-Z]{2,3}
```

## 另请参阅

+   *基本 sed 入门*，解释了 sed 命令

+   *基本正则表达式入门*，解释了如何使用正则表达式

# 在文件中打印模式之前或之后的 n 行

通过模式匹配打印文本部分在文本处理中经常使用。有时，我们可能需要在文本中出现模式之前或之后的文本行。例如，考虑一个包含电影演员评分的文件，其中每行对应于电影演员的详细信息，我们需要找出演员的评分以及评分最接近他们的演员的详细信息。让我们看看如何做到这一点。

## 准备工作

`grep`是在文件中搜索和查找文本的最佳工具。通常，`grep`打印给定模式的匹配行或匹配文本。但是`grep`中的上下文行控制选项使其能够打印模式匹配行周围的行之前，之后和之前后。

## 如何做...

这种技术可以通过电影演员列表更好地解释。例如：

```
$ cat actress_rankings.txt | head -n 20
1 Keira Knightley
2 Natalie Portman 
3 Monica Bellucci
4 Bonnie Hunt 
5 Cameron Diaz 
6 Annie Potts 
7 Liv Tyler 
8 Julie Andrews 
9 Lindsay Lohan
10 Catherine Zeta-Jones 
11 CateBlanchett
12 Sarah Michelle Gellar 
13 Carrie Fisher 
14 Shannon Elizabeth 
15 Julia Roberts 
16 Sally Field 
17 TéaLeoni
18 Kirsten Dunst
19 Rene Russo 
20 JadaPinkett

```

为了打印匹配“卡梅隆·迪亚兹”之后的三行文本以及匹配行，请使用以下命令：

```
$ grep -A 3 "Cameron Diaz" actress_rankings.txt
5 Cameron Diaz
6 Annie Potts
7 Liv Tyler
8 Julie Andrews

```

为了打印匹配的行和前面的三行，请使用以下命令：

```
$ grep -B 3 "Cameron Diaz" actress_rankings.txt 
2 Natalie Portman 
3 Monica Bellucci
4 Bonnie Hunt 
5 Cameron Diaz

```

打印匹配的行以及匹配行之前和之后的两行如下：

```
$ grep -C 2 "Cameron Diaz" actress_rankings.txt 
3 Monica Bellucci
4 Bonnie Hunt 
5 Cameron Diaz
6 Annie Potts 
7 Liv Tyler

```

你想知道我从哪里得到这个排名吗？

我使用基本的 sed，awk 和 grep 命令解析了一个充满图像和 HTML 内容的网站。请参阅章节：*纷乱的网络？一点也不。*

## 另请参阅

+   使用 grep 在文件中搜索和挖掘“文本”解释了 grep 命令。

# 从包含单词的文件中删除句子

当确定了正确的正则表达式时，删除包含单词的句子是一个简单的任务。这只是解决类似问题的练习。

## 准备工作

`sed`是进行替换的最佳实用工具。因此让我们使用`sed`将匹配的句子替换为空白。

## 如何做...

让我们创建一个带有一些文本的文件进行替换。例如：

```
$ cat sentence.txt 
Linux refers to the family of Unix-like computer operating systems that use the Linux kernel. Linux can be installed on a wide variety of computer hardware, ranging from mobile phones, tablet computers and video game consoles, to mainframes and supercomputers. Linux is predominantly known for its use in servers. It has a server market share ranging between 20–40%. Most desktop computers run either Microsoft Windows or Mac OS X, with Linux having anywhere from a low of an estimated 1–2% of the desktop market to a high of an estimated 4.8%. However, desktop use of Linux has become increasingly popular in recent years, partly owing to the popular Ubuntu, Fedora, Mint, and openSUSE distributions and the emergence of netbooks and smart phones running an embedded Linux.

```

我们将删除包含“移动电话”一词的句子。使用以下`sed`表达式执行此任务：

```
$ sed 's/ [^.]*mobile phones[^.]*\.//g' sentence.txt
Linux refers to the family of Unix-like computer operating systems that use the Linux kernel. Linux is predominantly known for its use in servers. It has a server market share ranging between 20–40%. Most desktop computers run either Microsoft Windows or Mac OS X, with Linux having anywhere from a low of an estimated 1–2% of the desktop market to a high of an estimated 4.8%. However, desktop use of Linux has become increasingly popular in recent years, partly owing to the popular Ubuntu, Fedora, Mint, and openSUSE distributions and the emergence of netbooks and smart phones running an embedded Linux.

```

## 它是如何工作的...

让我们评估`sed`正则表达式`'s/ [^.]*mobile phones[^.]*\.//g'`。

它的格式是`s/替换模式/替换字符串/g`。

它用替换字符串替换每个`替换模式`的出现。

这里替换模式是句子的正则表达式。每个句子由“。”分隔，第一个字符是空格。因此，我们需要匹配格式为“空格”一些文本 MATCH_STRING 一些文本“点”的文本。句子可以包含除“点”之外的任何字符，这是分隔符。因此，我们使用了[^.]。[^.]*匹配除点之外的任何字符的组合。在文本匹配字符串“移动电话”之间放置。每个匹配句子都被替换为`//`（无）。

## 另请参阅

+   *基本 sed 入门*，解释了 sed 命令

+   *基本正则表达式入门*，解释了如何使用正则表达式

# 使用 awk 实现 head，tail 和 tac

掌握文本处理操作需要实践。这个配方将帮助我们练习结合我们刚刚学到的一些命令和我们已经知道的一些命令。

## 准备工作

命令`head`，`tail`，`uniq`和`tac`逐行操作。每当我们需要逐行处理时，我们总是可以使用`awk`。让我们用`awk`模拟这些命令。

## 如何做...

让我们看看如何用不同的基本文本处理命令来模拟不同的命令，比如 head、tail 和 tac。

`head`命令读取文件的前十行并将它们打印出来：

```
$ awk 'NR <=10' filename

```

`tail`命令打印文件的最后十行：

```
$ awk '{ buffer[NR % 10] = $0; } END { for(i=1;i<11;i++) { print buffer[i%10] } }' filename

```

`tac`命令以相反的顺序打印输入文件的行：

```
$ awk '{ buffer[NR] = $0; } END { for(i=NR; i>0; i--) { print buffer[i] } }' filename

```

## 它是如何工作的...

在使用`awk`实现`head`时，我们打印输入流中行号小于或等于`10`的行。行号可以使用特殊变量`NR`获得。

在`tail`命令的实现中使用了一种哈希技术。缓冲区数组索引由哈希函数`NR % 10`确定，其中`NR`是包含当前执行的 Linux 编号的变量。`$0`是文本变量中的行。因此`%`将哈希函数中具有相同余数的所有行映射到数组的特定索引。在`END{}`块中，它可以遍历数组的十个索引值并打印缓冲区中存储的行。

在`tac`命令的模拟中，它简单地将所有行存储在一个数组中。当它出现在`END{}`块中时，`NR`将保存最后一行的行号。然后它在`for`循环中递减，直到达到`1`，然后打印每个迭代语句中存储的行。

## 另请参阅

+   *基本 awk 入门*，解释了 awk 命令

+   *head 和 tail - 打印最后或前 10 行* of 第三章，解释了 head 和 tail 命令

+   *排序、唯一和重复* of 第二章, 解释了 uniq 命令

+   *以相反的顺序打印行*，解释了 tac 命令

# 文本切片和参数操作

这个教程介绍了 Bash 中一些简单的文本替换技术和参数扩展简写。一些简单的技巧通常可以帮助我们避免编写多行代码。

## 如何做...

让我们开始任务吧。

从变量中替换一些文本可以这样做：

```
$ var="This is a line of text"
$ echo ${var/line/REPLACED}
This is a REPLACED of text"

```

`line`被替换为`REPLACED`。

我们可以通过指定起始位置和字符串长度来生成子字符串，使用以下语法：

```
${variable_name:start_position:length}
```

要从第五个字符开始打印，请使用以下命令：

```
$ string=abcdefghijklmnopqrstuvwxyz
$ echo ${string:4}
efghijklmnopqrstuvwxyz

```

要从第五个字符开始打印八个字符，请使用：

```
$ echo ${string:4:8}
efghijkl

```

索引是通过将起始字母计为`0`来指定的。我们也可以指定从最后一个字母开始计数为`-1`。但它是在括号内使用的。`(-1)`是最后一个字母的索引。

```
echo ${string:(-1)}
z
$ echo ${string:(-2):2}
yz

```

## 另请参阅

+   *在文件中迭代行、单词和字符*，解释了从单词中切片一个字符
