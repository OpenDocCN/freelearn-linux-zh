# 第一章。开始使用 Shell 脚本

本章是关于 shell 脚本的简要介绍。它将假定读者对脚本基础知识大多熟悉，并将作为复习。

本章涵盖的主题如下：

+   脚本的一般格式。

+   如何使文件可执行。

+   创建良好的使用消息和处理返回代码。

+   展示如何从命令行传递参数。

+   展示如何使用条件语句验证参数。

+   解释如何确定文件的属性。

# 入门

您始终可以在访客账户下创建这些脚本，并且大多数脚本都可以从那里运行。当需要 root 访问权限来运行特定脚本时，将明确说明。

本书将假定用户已在该帐户的路径开头放置了(`.`)。如果没有，请在文件名前加上`./`来运行脚本。例如：

```
 $ ./runme
```

使用`chmod`命令使脚本可执行。

建议用户在其访客账户下创建一个专门用于本书示例的目录。例如，像这样的东西效果很好：

```
$ /home/guest1/LinuxScriptingBook/chapters/chap1
```

当然，随意使用最适合您的方法。

遵循 bash 脚本的一般格式，第一行将只包含此内容：

```
#!/bin/sh
```

请注意，在其他情况下，`#`符号后面的文本被视为注释。

例如，

# 整行都是注释

```
chmod 755 filename   # This text after the # is a comment
```

根据需要使用注释。有些人每行都加注释，有些人什么都不加注释。我试图在这两个极端之间取得平衡。

## 使用好的文本编辑器

我发现大多数人在 UNIX/Linux 环境下使用 vi 创建和编辑文本文档时感到舒适。这很好，因为 vi 是一个非常可靠的应用程序。我建议不要使用任何类型的文字处理程序，即使它声称具有代码开发选项。这些程序可能仍然会在文件中放入不可见的控制字符，这可能会导致脚本失败。除非您擅长查看二进制文件，否则可能需要花费数小时甚至数天来解决这个问题。

此外，我认为，如果您计划进行大量的脚本和/或代码开发，建议查看 vi 之外的其他文本编辑器。您几乎肯定会变得更加高效。

# 演示脚本的使用

这是一个非常简单的脚本示例。它可能看起来不起眼，但这是每个脚本的基础：

## 第一章 - 脚本 1

```
#!/bin/sh
#
#  03/27/2017
#
exit 0
```

### 注意

按照惯例，在本书中，脚本行通常会编号。这仅用于教学目的，在实际脚本中，行不会编号。

以下是带有行号的相同脚本：

```
1  #!/bin/sh
2  #
3  # 03/27/2017
4  #
5  exit 0
6
```

以下是每行的解释：

+   第 1 行告诉操作系统要使用哪个 shell 解释器。请注意，在某些发行版上，`/bin/sh`实际上是指向解释器的符号链接。

+   以`#`开头的行是注释。此外，`#`后面的任何内容也被视为注释。

+   在脚本中包含日期是一个好习惯，可以在注释部分和/或`Usage`部分（下一节介绍）中包含日期。

+   第 5 行是此脚本的返回代码。这是可选的，但强烈建议。

+   第 6 行是空行，也是脚本的最后一行。

使用您喜欢的文本编辑器，编辑一个名为`script1`的新文件，并将前面的脚本复制到其中，不包括行号。保存文件。

要将文件转换为可执行脚本，请运行以下命令：

```
$ chmod 755 script1
```

现在运行脚本：

```
$ script1
```

如果您没有像介绍中提到的那样在路径前加上`.`，则运行：

```
$ ./script1
```

现在检查返回代码：

```
$ echo $?
0
```

这是一个执行得更有用的脚本：

## 第一章 - 脚本 2

```
#!/bin/sh
#
# 3/26/2017
#
ping -c 1 google.com        # ping google.com just 1 time
echo Return code: $?
```

`ping`命令成功返回零，失败返回非零。如您所见，`echoing $?`显示了其前一个命令的返回值。稍后会详细介绍。

现在让我们传递一个参数并包括一个`Usage`语句：

## 第一章 - 脚本 3

```
  1  #!/bin/sh
  2  #
  3  # 6/13/2017
  4  #
  5  if [ $# -ne 1 ] ; then
  6   echo "Usage: script3 file"
  7   echo " Will determine if the file exists."
  8   exit 255
  9  fi
 10  
 11  if [ -f $1 ] ; then
 12   echo File $1 exists.
 13   exit 0
 14  else
 15   echo File $1 does not exist.
 16   exit 1
 17  fi
 18  
```

以下是每行的解释：

+   第`5`行检查是否给出了参数。如果没有，将执行第`6`到`9`行。请注意，通常最好在脚本中包含一个信息性的`Usage`语句。还要提供有意义的返回代码。

+   第`11`行检查文件是否存在，如果是，则执行第`12`-`13`行。否则运行第`14`-`17`行。

+   关于返回代码的说明：在 Linux/UNIX 下，如果命令成功，则返回零是标准做法，如果不成功则返回非零。这样返回的代码可以有一些有用的含义，不仅对人类有用，对其他脚本和程序也有用。但这并不是强制性的。如果你希望你的脚本返回不是错误而是指示其他条件的代码，那么请这样做。

下一个脚本扩展了这个主题：

## 第一章 - 脚本 4

```
  1  #!/bin/sh
  2  #
  3  # 6/13/2017
  4  #
  5  if [ $# -ne 1 ] ; then
  6   echo "Usage: script4 filename"
  7   echo " Will show various attributes of the file given."
  8   exit 255
  9  fi
 10  
 11  echo -n "$1 "                # Stay on the line
 12  
 13  if [ ! -e $1 ] ; then
 14   echo does not exist.
 15   exit 1                      # Leave script now
 16  fi
 17  
 18  if [ -f $1 ] ; then
 19   echo is a file.
 20  elif [ -d $1 ] ; then
 21   echo is a directory.
 22  fi
 23  
 24  if [ -x $1 ] ; then
 25   echo Is executable.
 26  fi
 27  
 28  if [ -r $1 ] ; then
 29   echo Is readable.
 30  else
 31   echo Is not readable.
 32  fi
 33  
 34  if [ -w $1 ] ; then
 35   echo Is writable.
 36  fi
 37  
 38  if [ -s $1 ] ; then
 39   echo Is not empty.
 40  else
 41   echo Is empty.
 42  fi
 43  
 44  exit 0                       # No error
 45  
```

以下是每行的解释：

+   第`5`-`9`行：如果脚本没有使用参数运行，则显示`Usage`消息并以返回代码`255`退出。

+   第`11`行显示了如何`echo`一个文本字符串但仍然保持在同一行（没有换行）。

+   第`13`行显示了如何确定给定的参数是否是现有文件。

+   第`15`行如果文件不存在，则退出脚本没有继续的理由。

剩下的行的含义可以通过脚本本身确定。请注意，可以对文件执行许多其他检查，这只是其中的一部分。

以下是在我的系统上运行`script4`的一些示例：

```
guest1 $ script4
Usage: script4 filename
 Will show various attributes of the file given.

guest1 $ script4 /tmp
/tmp is a directory.
Is executable.
Is readable.
Is writable.
Is not empty.

guest1 $ script4 script4.numbered
script4.numbered is a file.
Is readable.
Is not empty.

guest1 $ script4 /usr
/usr is a directory.
Is executable.
Is readable.
Is not empty.

guest1 $ script4 empty1
empty1 is a file.
Is readable.
Is writable.
Is empty.

guest1 $ script4 empty-noread
empty-noread is a file.
Is not readable.
Is empty.
```

下一个脚本显示了如何确定传递给它的参数数量：

## 第一章 - 脚本 5

```
#!/bin/sh
#
# 3/27/2017
#
echo The number of parameters is: $#
exit 0
```

让我们尝试一些例子：

```
guest1 $ script5
The number of parameters is: 0

guest1 $ script5 parm1
The number of parameters is: 1

guest1 $ script5 parm1 Hello
The number of parameters is: 2

guest1 $ script5 parm1 Hello 15
The number of parameters is: 3

guest1 $ script5 parm1 Hello 15 "A string"
The number of parameters is: 4

guest1 $ script5 parm1 Hello 15 "A string" lastone
The number of parameters is: 5
```

### 提示

记住，引用的字符串被计算为 1 个参数。这是传递包含空格的字符串的一种方法。

下一个脚本显示了如何更详细地处理多个参数：

## 第一章 - 脚本 6

```
#!/bin/sh
#
# 3/27/2017
#

if [ $# -ne 3 ] ; then
 echo "Usage: script6 parm1 parm2 parm3"
 echo " Please enter 3 parameters."

 exit 255
fi

echo Parameter 1: $1
echo Parameter 2: $2
echo Parameter 3: $3

exit 0
```

这个脚本的行没有编号，因为它相当简单。`$#`包含传递给脚本的参数数量。

# 总结

在本章中，我们讨论了脚本设计的基础知识。展示了如何使脚本可执行，以及创建信息性的`Usage`消息。还介绍了返回代码的重要性，以及参数的使用和验证。

下一章将更详细地讨论变量和条件语句。
