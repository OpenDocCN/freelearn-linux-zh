# 第十二章：在脚本中使用管道和重定向

在本章中，我们将解释 Bash 的一个非常重要的方面：*重定向*。我们将从描述不同类型的输入和输出重定向开始，以及它们如何与 Linux 文件描述符相关联。在涵盖了重定向的基础知识之后，我们将继续介绍一些高级用法。

接下来是*管道*，这是 Shell 脚本中广泛使用的一个概念。我们将介绍一些管道的实际示例。最后，我们将展示*here documents*的工作原理，这也有一些很好的用途。

本章将介绍以下命令：`diff`、`gcc`、`fallocate`、`tr`、`chpasswd`、`tee`和`bc`。

本章将涵盖以下主题：

+   输入/输出重定向

+   管道

+   Here documents

# 技术要求

本章的所有脚本都可以在 GitHub 上找到，链接如下：[`github.com/tammert/learn-linux-shell-scripting/tree/master/chapter_12`](https://github.com/tammert/learn-linux-shell-scripting/tree/master/chapter_12)。对于所有其他练习，你的 Ubuntu 18.04 虚拟机仍然是你最好的朋友。

# 输入/输出重定向

在本章中，我们将详细讨论 Linux 中的重定向。

简而言之，重定向几乎完全就像字面意思一样：将*某物*重定向到*其他某物*。例如，我们已经看到我们可以使用一个命令的输出作为下一个命令的输入，使用管道。在 Linux 中，管道是使用`|`符号实现的。

然而，这可能会引发一个问题：Linux 如何处理输入和输出？我们将从一些关于**文件描述符**的理论开始我们的重定向之旅，这是使所有重定向成为可能的原因！

# 文件描述符

你可能已经厌倦了听到这一点，但它仍然是真的：在 Linux 中，一切都是一个文件。我们已经看到一个文件是一个文件，一个目录是一个文件，甚至硬盘也是文件；但现在，我们将再进一步：你用于*输入*的键盘也是一个文件！

与此相辅相成的是，你的终端，命令使用它作为*输出*，猜猜看：就是一个文件。

你可以在 Linux 文件系统树中找到这些文件，就像大多数特殊文件一样。让我们检查一下我们的虚拟机：

```
reader@ubuntu:~$ cd /dev/fd/
reader@ubuntu:/dev/fd$ ls -l
total 0
lrwx------ 1 reader reader 64 Nov  5 18:54 0 -> /dev/pts/0
lrwx------ 1 reader reader 64 Nov  5 18:54 1 -> /dev/pts/0
lrwx------ 1 reader reader 64 Nov  5 18:54 2 -> /dev/pts/0
lrwx------ 1 reader reader 64 Nov  5 18:54 255 -> /dev/pts/0
```

在这里找到的四个文件中，有三个很重要：`/dev/fd/0`、`/dev/fd/1`和`/dev/fd/2`。

从这段文字的标题中，你可能会怀疑**fd**代表**f**ile **d**escriptor。这些文件描述符在内部用于将用户的输入和输出与终端绑定在一起。你实际上可以看到文件描述符是如何做到这一点的：它们被符号链接到`/dev/pts/0`。

在这种情况下，**pts**代表**伪终端从属**，这是对 SSH 连接的定义。看看当我们从三个不同的位置查看`/dev/fd`时会发生什么：

```
# SSH connection 1
reader@ubuntu:~/scripts/chapter_12$ ls -l /dev/fd/
total 0
lrwx------ 1 reader reader 64 Nov  5 19:06 0 -> /dev/pts/0
lrwx------ 1 reader reader 64 Nov  5 19:06 1 -> /dev/pts/0
lrwx------ 1 reader reader 64 Nov  5 19:06 2 -> /dev/pts/0

# SSH connection 2
reader@ubuntu:/dev/fd$ ls -l
total 0
lrwx------ 1 reader reader 64 Nov  5 18:54 0 -> /dev/pts/1
lrwx------ 1 reader reader 64 Nov  5 18:54 1 -> /dev/pts/1
lrwx------ 1 reader reader 64 Nov  5 18:54 2 -> /dev/pts/1

# Virtual machine terminal
reader@ubuntu:/dev/fd$ ls -l
total 0
lrwx------ 1 reader reader 64 Nov  5 19:08 0 -> /dev/tty/1
lrwx------ 1 reader reader 64 Nov  5 19:08 1 -> /dev/tty/1
lrwx------ 1 reader reader 64 Nov  5 19:08 2 -> /dev/tty/1
```

每个连接都有自己的`/dev/`挂载（存储在内存中的`udev`类型），这就是为什么我们看不到一个连接的输出进入另一个连接的原因。

现在，我们一直在谈论输入和输出。但是，正如你无疑所见，前面的例子中分配了三个文件描述符。在 Linux（或类 Unix 系统）中，默认通过文件描述符公开的三个默认**流**：

+   *标准输入*流`stdin`默认绑定到`/dev/fd/0`

+   *标准输出*流`stdout`默认绑定到`/dev/fd/1`

+   *标准错误*流`stderr`默认绑定到`/dev/fd/2`

就这三个流而言，`stdin`和`stdout`应该相当直接：输入和输出。然而，正如你可能已经推断出的那样，输出实际上被分成*正常*输出和*错误*输出。正常输出被发送到`stdout`文件描述符，而错误输出通常被发送到`stderr`。

由于这两者都是符号链接到终端，所以无论如何你都会在那里看到它们。然而，正如我们将在本章后面看到的，一旦我们开始重定向，这种差异就变得重要起来。

你可能会看到一些其他文件描述符，比如第一个示例中的 255。除了在终端提供输入和输出时使用它们，文件描述符还在 Linux 打开文件系统中的文件时使用。文件描述符的这种其他用途超出了本书的范围；然而，我们在*进一步阅读*部分包含了一个链接，供感兴趣的读者参考。

在正常交互中，你在终端中输入的文本被写入到`/dev/fd/0`上的`stdin`，一个命令可以读取。使用该输入，命令通常会执行某些操作（否则，我们就不需要这个命令！）并将输出写入`stdout`或`stderr`。然后终端会读取这些输出并显示给你。简而言之：

+   一个*终端* **写入** `stdin`，**读取** `stdout`或`stderr`

+   一个*命令* **从** `stdin` **读取**，**写入** `stdout` 或 `stderr`

除了 Linux 内部使用的文件描述符之外，还有一些保留用于创建真正高级脚本的文件描述符；这些是 3 到 9。任何其他的*可能*被系统使用，但这些保证可以自由使用。正如所述，这是非常高级的，不太经常使用，我们不会详细介绍。然而，我们找到了一些可能有趣的进一步阅读材料，这些材料包含在本章的末尾。

# 重定向输出

现在输入、输出和文件描述符的理论应该是清楚的，我们将看到如何在命令行和脚本冒险中使用这些技术。

事实上，在没有使用重定向的情况下编写 shell 脚本是相当困难的；在本章之前的书中，我们实际上已经使用了几次重定向，因为我们当时真的需要它来完成我们的工作（例如第八章中的`file-create.sh`，*变量和用户输入*）。

现在，让我们先来体验一下重定向！

# stdout

大多数命令的输出将是*标准输出*，写入`/dev/fd/1`上的`stdout`。通过使用`>`符号，我们可以使用以下语法重定向输出：

```
command > output-file
```

重定向将始终指向一个文件（然而，正如我们所知，不是所有文件都是相等的，因此在常规示例之后，我们将向您展示一些 Bash 魔法，涉及非常规文件）。如果文件不存在，它将被创建。如果存在，它将被**覆盖**。

在其最简单的形式中，通常会打印到终端的所有内容都可以重定向到文件：

```
reader@ubuntu:~/scripts/chapter_12$ ls -l /var/log/dpkg.log 
-rw-r--r-- 1 root root 737150 Nov  5 18:49 /var/log/dpkg.log
reader@ubuntu:~/scripts/chapter_12$ cat /var/log/dpkg.log > redirected-file.log
reader@ubuntu:~/scripts/chapter_12$ ls -l
total 724
-rw-rw-r-- 1 reader reader 737150 Nov  5 19:45 redirected-file.log
```

如你所知，`cat`将整个文件内容打印到你的终端。实际上，它实际上将整个内容发送到`stdout`，它绑定到`/dev/fd/1`，它绑定到你的终端；这就是为什么你看到它。

现在，如果我们将文件的内容重定向回另一个文件，我们实际上已经做出了很大的努力...复制一个文件！从文件大小可以看出，实际上是相同的文件。如果你不确定，你可以使用`diff`命令来查看文件是否相同：

```
reader@ubuntu:~/scripts/chapter_12$ diff /var/log/dpkg.log redirected-file.log 
reader@ubuntu:~/scripts/chapter_12$ echo $?
0
```

如果`diff`没有返回任何输出，并且它的退出代码为`0`，则文件没有差异。

回到重定向示例。我们使用`>`将输出重定向到文件。实际上，`>`是`1>`的简写。你可能会认出这个`1`：它指的是文件描述符`/dev/fd/1`。正如我们将在处理`stderr`时看到的，它位于`/dev/fd/2`上，我们将使用`2>`而不是`1>`或`>`。

首先，让我们构建一个简单的脚本来进一步说明这一点：

```
reader@ubuntu:~/scripts/chapter_12$ vim redirect-to-file.sh 
reader@ubuntu:~/scripts/chapter_12$ cat redirect-to-file.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-05
# Description: Redirect user input to file.
# Usage: ./redirect-to-file.sh
#####################################

# Capture the users' input.
read -p "Type anything you like: " user_input

# Save the users' input to a file.
echo ${user_input} > redirect-to-file.txt
```

现在，当我们运行这个脚本时，`read`会提示我们输入一些文本。这将保存在`user_input`变量中。然后，我们将使用`echo`将`user_input`变量的内容发送到`stdout`。但是，它不会通过`/dev/fd/1`到达终端上的`/dev/pts/0`，而是重定向到`redirect-to-file.txt`文件中。

总的来说，它看起来像这样：

```
reader@ubuntu:~/scripts/chapter_12$ bash redirect-to-file.sh 
Type anything you like: I like dogs! And cats. Maybe a gecko?
reader@ubuntu:~/scripts/chapter_12$ ls -l
total 732
-rw-rw-r-- 1 reader reader 737150 Nov  5 19:45 redirected-file.log
-rw-rw-r-- 1 reader reader    383 Nov  5 19:58 redirect-to-file.sh
-rw-rw-r-- 1 reader reader     38 Nov  5 19:58 redirect-to-file.txt
reader@ubuntu:~/scripts/chapter_12$ cat redirect-to-file.txt
I like dogs! And cats. Maybe a gecko?
```

现在，这个脚本按照预期工作。然而，如果我们再次运行它，我们会看到这个脚本可能出现的两个问题：

```
reader@ubuntu:~/scripts$ bash chapter_12/redirect-to-file.sh
Type anything you like: Hello
reader@ubuntu:~/scripts$ ls -l
<SNIPPED>
drwxrwxr-x 2 reader reader 4096 Nov  5 19:58 chapter_12
-rw-rw-r-- 1 reader reader    6 Nov  5 20:02 redirect-to-file.txt
reader@ubuntu:~/scripts$ bash chapter_12/redirect-to-file.sh
Type anything you like: Bye
reader@ubuntu:~/scripts$ ls -l
<SNIPPED>
drwxrwxr-x 2 reader reader 4096 Nov  5 19:58 chapter_12
-rw-rw-r-- 1 reader reader    4 Nov  5 20:02 redirect-to-file.txt
```

第一件出错的事情，正如我们之前警告过的，是相对路径可能会搞乱文件的写入位置。

你可能已经设想到文件是在脚本旁边创建的；只有当你的*当前工作目录*在脚本所在的目录中时，才会发生这种情况。因为我们是从树的较低位置调用它，所以输出被写入那里（因为那是当前工作目录）。

另一个问题是，每次我们输入内容时，都会删除文件的旧内容！在我们输入`Hello`后，我们看到文件有六个字节（每个字符一个字节，加上一个换行符），在我们输入`Bye`后，我们现在看到文件只有四个字节（三个字符加上换行符）。

这可能是期望的行为，但更多时候，如果输出*追加*到文件中，而不是替换它，会更好。

让我们在脚本的新版本中解决这两个问题：

```
reader@ubuntu:~/scripts$ vim chapter_12/redirect-to-file.sh 
reader@ubuntu:~/scripts$ cat chapter_12/redirect-to-file.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
# Date: 2018-11-05
# Description: Redirect user input to file.
# Usage: ./redirect-to-file.sh
#####################################

# Since we're dealing with paths, set current working directory.
cd $(dirname $0)

# Capture the users' input.
read -p "Type anything you like: " user_input

# Save the users' input to a file. > for overwrite, >> for append.
echo ${user_input} >> redirect-to-file.txt
```

现在，如果我们运行它（无论在哪里），我们会看到新的文本被追加到第一句话，“我喜欢狗！还有猫。也许是壁虎？”在`/home/reader/chapter_12/redirect-to-file.txt`文件中：

```
reader@ubuntu:~/scripts$ cd /tmp/
reader@ubuntu:/tmp$ cat /home/reader/scripts/chapter_12/redirect-to-file.txt 
I like dogs! And cats. Maybe a gecko?
reader@ubuntu:/tmp$ bash /home/reader/scripts/chapter_12/redirect-to-file.sh
Type anything you like: Definitely a gecko, those things are awesome!
reader@ubuntu:/tmp$ cat /home/reader/scripts/chapter_12/redirect-to-file.txt 
I like dogs! And cats. Maybe a gecko?
Definitely a gecko, those things are awesome!
```

所以，`cd $(dirname $0)` 帮助我们处理相对路径，`>>` 而不是 `>` 确保追加而不是覆盖。正如你所期望的那样，`>>` 再次代表 `1>>`，当我们开始稍后重定向 `stderr` 流时，我们会看到这一点。

不久前，我们向你承诺了一些 Bash 魔法。虽然不完全是魔法，但可能会让你的头有点疼：

```
reader@ubuntu:~/scripts/chapter_12$ cat redirect-to-file.txt 
I like dogs! And cats. Maybe a gecko?
Definitely a gecko, those things are awesome!
reader@ubuntu:~/scripts/chapter_12$ cat redirect-to-file.txt > /dev/pts/0
I like dogs! And cats. Maybe a gecko?
Definitely a gecko, those things are awesome!
reader@ubuntu:~/scripts/chapter_12$ cat redirect-to-file.txt > /dev/fd/1
I like dogs! And cats. Maybe a gecko?
Definitely a gecko, those things are awesome!
reader@ubuntu:~/scripts/chapter_12$ cat redirect-to-file.txt > /dev/fd/2
I like dogs! And cats. Maybe a gecko?
Definitely a gecko, those things are awesome!
```

所以，我们成功地使用`cat`四次打印了我们的文件。你可能会想，我们也可以用`for`来做，但是这个教训不是我们打印消息的次数，而是我们是如何做到的！

首先，我们只是使用了`cat`；没有什么特别的。接下来，我们将`cat`与将`stdout`重定向到`/dev/pts/0`，也就是我们的终端，结合使用。同样，消息被打印出来。

第三和第四次，我们将`cat`的重定向`stdout`发送到`/dev/fd/1`和`/dev/fd/2`。由于这些是符号链接到`/dev/pts/0`，这也不奇怪，这些也最终出现在我们的终端上。

那么，我们如何实际区分 `stdout` 和 `stderr` 呢？

# stderr

如果你对前面的例子感到困惑，那可能是因为你误解了`stderr`消息的流向（我们不怪你，我们自己也搞混了！）。虽然我们将`cat`命令的输出发送到`/dev/fd/2`，但我们使用了`>`，这会发送`stdout`而不是`stderr`。

因此，在我们的示例中，我们滥用了 `stderr` 文件描述符来打印到终端；这是不好的做法。我们保证不会再这样做了。那么，我们如何*实际*处理 `stderr` 消息呢？

```
reader@ubuntu:/tmp$ cat /root/
cat: /root/: Permission denied
reader@ubuntu:/tmp$ cat /root/ 1> error-file
cat: /root/: Permission denied
reader@ubuntu:/tmp$ ls -l
-rw-rw-r-- 1 reader reader    0 Nov  5 20:35 error-file
reader@ubuntu:/tmp$ cat /root/ 2> error-file
reader@ubuntu:/tmp$ ls -l
-rw-rw-r-- 1 reader reader   31 Nov  5 20:35 error-file
reader@ubuntu:/tmp$ cat error-file 
cat: /root/: Permission denied
```

这种交互应该说明一些事情。首先，当`cat /root/`抛出`Permission denied`错误时，它将其发送到`stderr`而不是`stdout`。我们可以看到这一点，因为当我们执行相同的命令，但尝试用`1> error-file`重定向*标准* *输出*时，我们仍然在终端上看到输出，并且我们还看到`error-file`是空的。

当我们使用`2> error-file`时，它重定向`stderr`而不是常规的`stdout`，我们不再在终端上看到错误消息。

更好的是，我们现在看到`error-file`有 31 个字节的内容，当我们用`cat`打印它时，我们再次看到了我们重定向的错误消息！如前所述，并且与`1>>`的精神一样，如果你想*追加*而不是*覆盖*`stderr`流到一个文件，使用`2>>`。

现在，因为很难找到一个命令来在同一个命令中打印`stdout`和`stderr`，我们将创建我们自己的命令：一个非常简单的 C 程序，它打印两行文本，一行到`stdout`，一行到`stderr`。

作为对编程和编译的预览，请看这个（如果你不完全理解这个，不要担心）：

```
reader@ubuntu:~/scripts/chapter_12$ vim stderr.c 
reader@ubuntu:~/scripts/chapter_12$ cat stderr.c 
#include <stdio.h>
int main()
{
  // Print messages to stdout and stderr.
  fprintf(stdout, "This is sent to stdout.\n");
  fprintf(stderr, "This is sent to stderr.\n");
  return 0;
}

reader@ubuntu:~/scripts/chapter_12$ gcc stderr.c -o stderr
reader@ubuntu:~/scripts/chapter_12$ ls -l
total 744
-rw-rw-r-- 1 reader reader 737150 Nov  5 19:45 redirected-file.log
-rw-rw-r-- 1 reader reader    501 Nov  5 20:09 redirect-to-file.sh
-rw-rw-r-- 1 reader reader     84 Nov  5 20:13 redirect-to-file.txt
-rwxrwxr-x 1 reader reader   8392 Nov  5 20:46 stderr
-rw-rw-r-- 1 reader reader    185 Nov  5 20:46 stderr.c
```

`gcc stderr.c -o stderr`命令将在`stderr.c`中找到的源代码编译为二进制文件`stderr`。

`gcc`是 GNU 编译器集合，并不总是默认安装的。如果你想跟着这个例子并且收到关于找不到`gcc`的错误，请使用`sudo apt install gcc -y`来安装它。

如果我们运行我们的程序，我们会得到两行输出。因为这不是一个 Bash 脚本，我们不能用`bash stderr`来执行它。我们需要用`chmod`使二进制文件可执行，并用`./stderr`来运行它：

```
reader@ubuntu:~/scripts/chapter_12$ bash stderr
stderr: stderr: cannot execute binary file
reader@ubuntu:~/scripts/chapter_12$ chmod +x stderr
reader@ubuntu:~/scripts/chapter_12$ ./stderr 
This is sent to stdout.
This is sent to stderr.
```

现在，让我们看看当我们开始重定向部分输出时会发生什么：

```
reader@ubuntu:~/scripts/chapter_12$ ./stderr > /tmp/stdout
This is sent to stderr.
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/stdout 
This is sent to stdout.
```

因为我们只重定向了`stdout`（最后提醒：`>`等于`1>`）到完全限定的文件`/tmp/stdout`，`stderr`消息仍然被打印到终端上。

另一种方式会得到类似的结果：

```
reader@ubuntu:~/scripts/chapter_12$ ./stderr 2> /tmp/stderr
This is sent to stdout.
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/stderr 
This is sent to stderr.
```

现在，当我们只使用`2> /tmp/stderr`来重定向`stderr`时，我们会看到`stdout`消息出现在我们的终端上，而`stderr`被正确地重定向到`/tmp/stderr`文件中。

我相信你现在正在问自己这个问题：我们如何重定向**所有输出**，包括`stdout`和`stderr`，到一个文件？如果这是一本关于 Bash 3.x 的书，我们将会有一个困难的对话。这个对话将包括我们将`stderr`重定向到`stdout`，之后我们可以使用`>`将所有输出（因为我们已经将`stderr`重定向到`stdout`）发送到一个单独的文件。

尽管这是逻辑上的做法，将`stderr`重定向到`stdout`实际上是在命令的末尾。命令最终变成这样：`./stderr > /tmp/output 2>&1`。并不是*太复杂*，但足够难以一次记住（你可以相信我们）。

幸运的是，在 Bash 4.x 中，我们有一个新的重定向命令可供我们使用，可以以更易理解的方式完成相同的事情：`&>`。

# 重定向所有输出

在大多数情况下，发送到`stderr`而不是`stdout`的输出将包含明显表明你正在处理错误的单词。这将包括诸如`permission denied`、`cannot execute binary file`、`syntax error near unexpected token`等示例。

因此，通常并不真的需要将输出分成`stdout`和`stderr`（但显然，有时会是很好的功能）。在这些情况下，Bash 4.x 的新增功能允许我们用单个命令重定向`stdout`和`stderr`是完美的。这种重定向，你可以使用`&>`语法，与我们之前看到的例子没有不同。

让我们回顾一下我们之前的例子，看看这是如何让我们的生活变得更容易的：

```
reader@ubuntu:~/scripts/chapter_12$ ./stderr
This is sent to stdout.
This is sent to stderr.
reader@ubuntu:~/scripts/chapter_12$ ./stderr &> /tmp/output
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/output
This is sent to stderr.
This is sent to stdout.
```

太棒了！有了这个语法，我们就不再需要担心不同的输出流。当你使用新命令时，这是特别实用的；在这种情况下，你可能会错过一些有趣的错误消息，因为`stderr`流没有被保存。

冒昧地说一下，将`stdout`和`stderr`都追加到文件的语法再次是额外的`>`：`&>>`。

继续尝试之前的例子。我们不会在这里打印它，因为现在应该很明显这是如何工作的。

不确定是重定向所有输出，还是只重定向`stdout`或`stderr`？我们的建议：从将**两者**重定向到同一个文件开始。如果在您的用例中这会产生太多噪音（掩盖错误或正常日志消息），您可以决定将它们中的任何一个重定向到文件，并在终端中打印另一个。实际上，`stderr`消息通常需要`stdout`消息提供的上下文来理解错误，因此最好将它们方便地放在同一个文件中！

# 特殊的输出重定向

尽管发送所有输出通常是一件好事，但您经常会发现自己要做的另一件事是将错误（您期望在某些命令上出现）重定向到一个特殊的设备：`/dev/null`。

`null`类型透露了功能：它介于垃圾桶和黑洞之间。

# /dev/null

实际上，所有发送（实际上是写入）到`/dev/null`的数据都将被丢弃，但仍会生成一个*写操作成功*的返回给调用命令。在这种情况下，那将是重定向。

这很重要，因为当重定向无法成功完成时会发生什么：

```
reader@ubuntu:~/scripts/chapter_12$ ./stderr &> /root/file
-bash: /root/file: Permission denied
reader@ubuntu:~/scripts/chapter_12$ echo $?
1
```

这个操作失败了（因为`reader`用户显然无法在`root`超级用户的主目录中写入）。

看看当我们尝试使用`/dev/null`做同样的事情时会发生什么：

```
reader@ubuntu:~/scripts/chapter_12$ ./stderr &> /dev/null 
reader@ubuntu:~/scripts/chapter_12$ echo $?
0
reader@ubuntu:~/scripts/chapter_12$ cat /dev/null 
reader@ubuntu:~/scripts/chapter_12$
```

就是这样。所有的输出都消失了（因为`&>`重定向了`stdout`和`stderr`），但命令仍然报告了期望的退出状态`0`。当我们确保数据已经消失时，我们使用`cat /dev/null`，结果什么也没有。

我们将向您展示一个实际示例，您在脚本中经常会使用到：

```
reader@ubuntu:~/scripts/chapter_12$ vim find.sh 
reader@ubuntu:~/scripts/chapter_12$ cat find.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-06
# Description: Find a file.
# Usage: ./find.sh <file-name>
#####################################

# Check for the current number of arguments.
if [[ $# -ne 1 ]]; then
  echo "Wrong number of arguments!"
  echo "Usage: $0 <file-name>"
  exit 1
fi

# Name of the file to search for.
file_name=$1

# Redirect all errors to /dev/null, so they don't clutter the terminal.
find / -name "${file_name}" 2> /dev/null
```

这个脚本只包含我们之前介绍过的结构，除了对`stderr`进行`/dev/null`重定向。虽然这个`find.sh`脚本实际上只是`find`命令的一个简单包装器，但它确实有很大的区别。

看看当我们使用`find`查找文件`find.sh`时会发生什么（因为为什么不呢！）：

```
reader@ubuntu:~/scripts/chapter_12$ find / -name find.sh
find: ‘/etc/ssl/private’: Permission denied
find: ‘/etc/polkit-1/localauthority’: Permission denied
<SNIPPED>
find: ‘/sys/fs/pstore’: Permission denied
find: ‘/sys/fs/fuse/connections/48’: Permission denied
/home/reader/scripts/chapter_12/find.sh
find: ‘/data/devops-files’: Permission denied
find: ‘/data/dev-files’: Permission denied
<SNIPPED>
```

我们删掉了大约 95%的输出，因为您可能会同意，五页的`Permission denied`错误没有多少价值。因为我们是以普通用户身份运行`find`，所以我们无法访问系统的许多部分。这些错误反映了这一点。

我们确实找到了我们的脚本，正如之前所强调的，但在遇到它之前可能需要滚动几分钟。这正是我们所说的错误输出淹没相关输出的情况。

现在，让我们用我们的包装脚本寻找同一个文件：

```
reader@ubuntu:~/scripts/chapter_12$ bash find.sh find.sh
/home/reader/scripts/chapter_12/find.sh
```

我们走了！相同的结果，但没有那些让我们困惑的烦人错误。由于`Permission denied`错误被发送到`stderr`流，我们在`find`命令之后使用`2> /dev/null` *删除*了它们。

这实际上带我们到另一个观点：您也可以使用重定向来使命令静音。我们已经看到许多命令都包含`--quiet`或`-q`标志。但是，有些命令，比如`find`，却没有这个标志。

您可能会认为`find`有这个标志会很奇怪（为什么要搜索文件，当你不想知道它在哪里时，对吧？），但可能有其他命令的退出代码提供了足够的信息，但没有`--quiet`标志；这些都是将所有内容重定向到`/dev/null`的绝佳候选者。

所有的命令都是不同的。虽然现在大多数命令都有一个可用的`--quiet`标志，但总会有一些情况不适用。也许`--quiet`标志只静音`stdout`而不是`stderr`，或者它只减少输出。无论如何，当您真的对输出不感兴趣（只对退出状态感兴趣）时，了解将所有输出重定向到`/dev/null`是一件非常好的事情！

# /dev/zero

我们可以使用的另一个特殊设备是`/dev/zero`。当我们将输出重定向到`/dev/zero`时，它与`/dev/null`完全相同：数据消失。但是，在实践中，`/dev/null`最常用于此目的。

那么，为什么有这个特殊的设备呢？因为`/dev/zero`也可以用来读取空字节。在所有可能的 256 个字节中，空字节是第一个：十六进制`00`。空字节通常用于表示命令的终止，例如。

现在，我们还可以使用这些空字节来为磁盘分配字节：

```
reader@ubuntu:/tmp$ ls -l
-rw-rw-r-- 1 reader reader   48 Nov  6 19:26 output
reader@ubuntu:/tmp$ head -c 1024 /dev/zero > allocated-file
reader@ubuntu:/tmp$ ls -l
-rw-rw-r-- 1 reader reader 1024 Nov  6 20:09 allocated-file
-rw-rw-r-- 1 reader reader   48 Nov  6 19:26 output
reader@ubuntu:/tmp$ cat allocated-file 
reader@ubuntu:/tmp$ 
```

通过使用`head -c 1024`，我们指定要从`/dev/zero`中获取*前 1024 个字符*。因为`/dev/zero`只提供空字节，这些字节都将是相同的，但我们确切知道会有`1024`个。

我们使用`stdout`重定向将它们重定向到文件，然后我们看到一个大小为 1024 字节的文件（多么令人惊讶）。现在，如果我们`cat`这个文件，我们什么也看不到！同样，这不应该是一个惊喜，因为空字节就是这样：空的，无效的，空的。终端无法表示它们，因此它不会显示。

如果您在脚本中需要执行此操作，还有另一个选项：`fallocate`：

```
reader@ubuntu:/tmp$ fallocate --length 1024 fallocated-file
reader@ubuntu:/tmp$ ls -l
-rw-rw-r-- 1 reader reader 1024 Nov  6 20:09 allocated-file
-rw-rw-r-- 1 reader reader 1024 Nov  6 20:13 fallocated-file
-rw-rw-r-- 1 reader reader   48 Nov  6 19:26 output
reader@ubuntu:/tmp$ cat fallocated-file 
reader@ubuntu:/tmp$ 
```

从前面的输出中可以看出，这个命令与我们已经通过`/dev/zero`读取和重定向实现的功能完全相同（如果`fallocate`实际上是从`/dev/zero`读取的一个花哨的包装器，我们不会感到惊讶，但我们不能确定）。

# 输入重定向

另外两个著名的特殊设备`/dev/random`和`/dev/urandom`最好与*输入重定向*一起讨论。

输入通常来自您的键盘，通过终端传递给命令。最简单的例子是`read`命令：它从`stdin`读取，直到遇到换行符（按下*Enter*键时），然后将输入保存到`REPLY`变量（或者如果您提供了该参数，则保存到任何自定义变量）。它看起来有点像这样：

```
reader@ubuntu:~$ read -p "Type something: " answer
Type something: Something
reader@ubuntu:~$ echo ${answer}
something
```

简单。现在，假设我们以非交互方式运行此命令，这意味着我们无法使用键盘和终端提供信息（对于`read`来说不是真正的用例，但这是一个很好的例子）。

在这种情况下，我们可以使用输入重定向（`stdin`）来提供`read`的输入。这是通过`<`字符实现的，它是`<0`的简写。记住`stdin`文件描述符是`/dev/fd/0`？这不是巧合。

让我们通过将`stdin`重定向到文件而不是终端，以非交互方式使用`read`：

```
reader@ubuntu:/tmp$ echo "Something else" > answer-file
reader@ubuntu:/tmp$ read -p "Type something: " new_answer < answer-file
reader@ubuntu:/tmp$ echo ${new_answer}
Something else
```

为了表明我们没有作弊并重复使用`${answer}`变量中已经存储的答案，我们已经将`read`中的回复重命名为`${new_answer}`。

现在，在命令的末尾，我们将`stdin`从`answer-file`文件重定向，我们首先使用`echo` + `stdout`重定向创建了这个文件。这就像在命令之后添加`< answer-file`一样简单。

这种重定向使`read`从文件中读取，直到遇到换行符（这恰好是`echo`总是以字符串结尾的地方）。

现在基本的输入重定向应该是清楚的了，让我们回到我们的特殊设备：`/dev/random`和`/dev/urandom`。这两个特殊文件是伪随机数生成器，这是一个复杂的词，用于生成*几乎*随机的数据。

在这些特殊设备的情况下，它们从设备驱动程序、鼠标移动和其他大部分是随机的东西中收集*熵*（一个类似随机性的复杂词）。

`/dev/random`和`/dev/urandom`之间有一个细微的区别：当系统中的熵不足时，`/dev/random`停止生成随机输出，而`/dev/urandom`则继续生成。

如果您真的需要完全的熵，`/dev/random`可能是更好的选择（老实说，在这种情况下，您可能会采取其他措施），但通常情况下，在您的脚本中，`/dev/urandom`是更好的选择，因为阻塞可能会导致不可思议的等待时间。这来自第一手经验，可能非常不方便！

对于我们的示例，我们只会展示`/dev/urandom`；`/dev/random`的输出类似。

在实践中，`/dev/urandom`会*随机*地产生字节。虽然有些字节在可打印的 ASCII 字符范围内（1-9，a-z，A-Z），但其他字节用于空格（0x20）或换行符（0x0A）。

您可以通过使用`head -1`从`/dev/urandom`中抓取'第一行'来查看随机性。由于一行以换行符结尾，命令`head -1 /dev/urandom`将打印直到第一个换行符之前的所有内容：这可能是少量或大量字符：

```
reader@ubuntu:/tmp$ head -1 /dev/urandom 
~d=G1���RB�Ҫ��"@
                F��OJ2�%�=�8�#,�t�7���M���s��Oѵ�w��k�qݙ����W��E�h��Q"x8��l�d��P�,�.:�m�[Lb/A�J�ő�M�o�v��
                                                                                                        �
reader@ubuntu:/tmp$ head -1 /dev/urandom 
��o�u���'��+�)T�M���K�K����Y��G�g".!{R^d8L��s5c*�.đ�
```

我们第一次运行时打印了更多的字符（并非所有字符都可读）比第二次运行时；这可以直接与生成的字节的随机性联系起来。第二次我们运行`head -1 /dev/urandom`时，我们比第一次迭代更快地遇到了换行字节 0x0A。

# 生成密码

现在，您可能会想知道随机字符可能有什么用。一个主要的例子是生成密码。长而随机的密码总是很好；它们抵抗暴力破解攻击，无法被猜测，并且如果不重复使用，非常安全。而且坦率地说，使用您自己的 Linux 系统的熵来生成随机密码有多酷呢？

更好的是，我们可以使用来自`/dev/urandom`的输入重定向来完成这个任务，再加上`tr`命令。一个简单的脚本看起来是这样的：

```
reader@ubuntu:~/scripts/chapter_12$ vim password-generator.sh 
reader@ubuntu:~/scripts/chapter_12$ cat password-generator.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-06
# Description: Generate a password.
# Usage: ./password-generator.sh <length>
#####################################

# Check for the current number of arguments.
if [[ $# -ne 1 ]]; then
  echo "Wrong number of arguments!"
  echo "Usage: $0 <length>"
  exit 1
fi

# Verify the length argument.
if [[ ! $1 =~ ^[[:digit:]]+$ ]]; then
  echo "Please enter a length (number)."
  exit 1
fi

password_length=$1

# tr grabs readable characters from input, deletes the rest.
# Input for tr comes from /dev/urandom, via input redirection.
# echo makes sure a newline is printed.
tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c ${password_length}
echo
```

标题和输入检查，甚至包括使用正则表达式检查数字的检查，现在应该是清楚的。

接下来，我们使用`tr`命令从`/dev/urandom`重定向输入，以获取我们的 a-z、A-Z 和 0-9 字符集中的可读字符。这些被*管道*到`head`（本章后面将更多地介绍管道），这会导致前*x*个字符被打印给用户（如脚本参数中指定的那样）。

为了确保终端格式正确，我们在没有参数的情况下添加了一个快速的`echo`；这只是打印一个换行符。就像这样，我们建立了我们自己的*私人*、*安全*和*离线*密码生成器。甚至使用输入重定向！

# 高级重定向

我们现在已经看到了输入和输出重定向，以及两者的一些实际用途。但是，我们还没有结合这两种重定向形式，这是完全可能的！

您可能不会经常使用这个功能；大多数命令接受输入作为参数，并经常提供一个标志，允许您指定要输出到的文件。但知识就是力量，如果您遇到一个没有这些参数的命令，您知道您可以自己解决这个问题。

在命令行上尝试以下操作，并尝试理解为什么会得到您看到的结果：

```
reader@ubuntu:~/scripts/chapter_12$ cat stderr.c 
#include <stdio.h>
int main()
{
  // Print messages to stdout and stderr.
  fprintf(stdout, "This is sent to stdout.\n");
  fprintf(stderr, "This is sent to stderr.\n");
  return 0;
}

reader@ubuntu:~/scripts/chapter_12$ grep 'stderr' < stderr.c 
  // Print messages to stdout and stderr.
  fprintf(stderr, "This is sent to stderr.\n");
reader@ubuntu:~/scripts/chapter_12$ grep 'stderr' < stderr.c > /tmp/grep-file
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/grep-file 
  // Print messages to stdout and stderr.
  fprintf(stderr, "This is sent to stderr.\n");
```

正如您所看到的，我们可以在同一行上使用`<`和`>`来重定向输入和输出。首先，我们在`grep 'stderr' < stderr.c`命令中使用了输入重定向的`grep`（这在技术上也是`grep 'stderr' stderr.c`所做的）。我们在终端中看到了输出。

接下来，我们在该命令的后面添加了`> /tmp/grep-file`，这意味着我们将把我们的`stdout`重定向到`/tmp/grep-file`文件。我们不再在终端中看到输出，但当我们`cat`文件时，我们会得到它，所以它成功地写入了文件。

由于我们现在处于本章的高级部分，我们将演示输入重定向放在哪里实际上并不重要：

```
reader@ubuntu:~/scripts/chapter_12$ < stderr.c grep 'stdout' > /tmp/grep-file-stdout
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/grep-file-stdout 
 // Print messages to stdout and stderr.
 fprintf(stdout, "This is sent to stdout.\n");
```

在这里，我们在命令的开头指定了输入重定向。对我们来说，当考虑流程时，这似乎是更合乎逻辑的方法，但这会导致实际命令（`grep`）出现在命令的大致中间，这会破坏可读性。

这在实践中基本上是一个无用的观点，因为我们发现很少有用于输入和输出重定向；即使在这个例子中，我们也只需将命令写成`grep 'stdout' stderr.c > /tmp/grep-file-stdout`，混乱的构造就消失了。

但真正理解输入和输出的运作方式，以及一些命令如何为你做一些繁重的工作，是值得你花时间去理解的！这些正是你在更复杂的脚本中会遇到的问题，充分理解这一点将为你节省大量的故障排除时间。

# 重定向重定向

我们已经给你一个重定向重定向过程的概览。最著名的例子是在 Bash 4.x 之前大多数情况下使用的，即将`stderr`流重定向到`stdout`流。通过这样做，你可以只用`>`语法重定向*所有*输出。

你可以这样实现：

```
reader@ubuntu:/tmp$ cat /etc/shadow
cat: /etc/shadow: Permission denied
reader@ubuntu:/tmp$ cat /etc/shadow > shadow
cat: /etc/shadow: Permission denied
reader@ubuntu:/tmp$ cat shadow 
#Still empty, since stderr wasn't redirected to the file.
reader@ubuntu:/tmp$ cat /etc/shadow > shadow 2>&1 
#Redirect fd2 to fd1 (stderr to stdout).
reader@ubuntu:/tmp$ cat shadow 
cat: /etc/shadow: Permission denied
```

记住，你不再需要在 Bash 4.x 中使用这种语法，但是如果你想要使用自定义的文件描述符作为输入/输出流，这将是有用的知识。通过以`2>&1`结束命令，我们将所有`stderr`输出（`2>`）写入`stdout`描述符（`&1`）。

我们也可以反过来做：

```
reader@ubuntu:/tmp$ head -1 /etc/passwd
root:x:0:0:root:/root:/bin/bash
reader@ubuntu:/tmp$ head -1 /etc/passwd 2> passwd
root:x:0:0:root:/root:/bin/bash
reader@ubuntu:/tmp$ cat passwd
#Still empty, since stdout wasn't redirected to the file.
reader@ubuntu:/tmp$ head -1 /etc/passwd 2> passwd 1>&2
#Redirect fd1 to fd2 (stdout to stderr).
reader@ubuntu:/tmp$ cat passwd 
root:x:0:0:root:/root:/bin/bash
```

所以现在，我们将`stderr`流重定向到`passwd`文件。然而，`head -1 /etc/passwd`命令只提供了一个`stdout`流；我们看到它被打印到终端而不是文件中。

当我们使用`1>&2`（也可以写成`>&2`）时，我们将`stdout`重定向到`stderr`。现在它被写入文件，我们可以在那里使用`cat`命令！

记住，这是高级信息，主要用于你的理论理解以及当你开始使用自定义文件描述符时。对于所有其他输出重定向，还是安全地使用我们之前讨论过的`&>`语法。

# 命令替换

虽然在 Linux 意义上并不严格属于重定向，但在我们看来，*命令替换*是一种功能性重定向的形式：你使用一个命令的输出作为另一个命令的参数。如果我们需要使用输出作为下一个命令的输入，我们会使用管道（正如我们将在几页后看到的），但有时我们只需要将输出放在我们命令中的一个非常特定的位置。

这就是命令替换的用途。我们已经在一些脚本中看到了命令替换：`cd $(dirname $0)`。简单地说，这做的事情类似于`cd`到`dirname $0`的结果。

`dirname $0`返回脚本所在的目录（因为`$0`是脚本的完全限定路径），所以当我们在脚本中使用它时，我们将确保所有操作都相对于脚本所在的目录进行。

如果没有命令替换，我们需要在再次使用它之前将输出存储在某个地方：

```
dirname $0 > directory-file
cd < directory-file
rm directory-file
```

虽然这有时会起作用，但这里有一些陷阱：

+   你需要在你有写权限的地方写一个文件

+   在`cd`之后你需要清理文件

+   你需要确保文件不会与其他脚本冲突

长话短说，这远非理想的解决方案，最好避免使用。而且由于 Bash 提供了命令替换，使用它并没有真正的缺点。正如我们所见，`cd $(dirname $0)`中的命令替换为我们处理了这个问题，而不需要我们跟踪文件或变量或任何其他复杂的构造。

命令替换实际上在 Bash 脚本中经常使用。看看以下的例子，我们在其中使用命令替换来实例化和填充一个变量：

```
reader@ubuntu:~/scripts/chapter_12$ vim simple-password-generator.sh 
reader@ubuntu:~/scripts/chapter_12$ cat simple-password-generator.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-10
# Description: Use command substitution with a variable.
# Usage: ./simple-password-generator.sh
#####################################

# Write a random string to a variable using command substitution.
random_password=$(tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 20)

echo "Your random password is: ${random_password}"

reader@ubuntu:~/scripts/chapter_12$ bash simple-password-generator.sh 
Your random password is: T3noJ3Udf8a2eQbqPiad
reader@ubuntu:~/scripts/chapter_12$ bash simple-password-generator.sh 
Your random password is: wu3zpsrusT5zyvbTxJSn
```

在这个例子中，我们重用了我们之前的`password-generator.sh`脚本中的逻辑。这一次，我们不给用户提供输入长度的选项；我们保持简单，假设长度为 20（至少在 2018 年，这是一个相当好的密码长度）。

我们使用命令替换将结果（随机密码）写入一个变量，然后将其`echo`给用户。

实际上我们可以在一行中完成这个操作：

```
reader@ubuntu:~/scripts/chapter_12$ echo "Your random password is: $(tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 20)"
Your random password is: REzCOa11pA2846fvxsa
```

然而，正如我们现在已经讨论了很多次，*可读性很重要*（仍然！）。我们认为在实际使用之前首先将其写入具有描述性名称的变量，可以增加脚本的可读性。

此外，如果我们想要多次使用相同的随机值，我们无论如何都需要一个变量。因此，在这种情况下，脚本中的额外冗长帮助我们并且是可取的。

`$(..)`的前身是使用反引号，即`` ` ``字符（在英语国际键盘上的`1`旁边）。`$(cd dirname $0)`以前写为`` `cd dirname $0` ``。虽然这与新的（更好的）`$(..)`语法做的事情相同，有两件事经常与反斜线有关：单词拆分和换行。这些都是由空白引起的问题。使用新的语法要容易得多，而且不必担心这样的事情！

# 进程替换

与命令替换紧密相关的是*进程替换*。语法如下：

```
<(command)
```

它的工作原理与命令替换非常相似，但不是将命令的输出作为字符串发送到某个地方，而是可以将输出作为文件引用。这意味着一些命令，它们不期望字符串，而是期望文件引用，也可以使用动态输入。

虽然太高级了，无法详细讨论，但这里有一个简单的例子，应该能传达要点：

```
reader@ubuntu:~/scripts/chapter_12$ diff <(ls /tmp/) <(ls /home/)
1,11c1
< directory-file
< grep-file
< grep-file-stdout
< passwd
< shadow
---
> reader
```

`diff`命令通常比较两个文件并打印它们的差异。现在，我们使用进程替换，让`diff`比较`ls /tmp/`和`ls /home/`的结果，使用`<(ls /tmp/)`语法。

# 管道

最后，我们一直期待的**管道**终于来了。这些近乎神奇的结构在 Linux/Bash 中使用得如此频繁，以至于每个人都应该了解它们。任何比单个命令更复杂的东西几乎总是使用管道来达到解决方案。

现在揭晓大秘密：管道实际上只是将一个命令的`stdout`连接到另一个命令的`stdin`。

等等，什么？！

# 绑定 stdout 到 stdin

是的，就是这样。现在你知道了输入和输出重定向的所有知识，这可能有点令人失望。然而，仅仅因为概念简单，并不意味着管道不是**极其强大**且被广泛使用的。

让我们看一个例子，展示我们如何用管道替换输入/输出重定向：

```
reader@ubuntu:/tmp$ echo 'Fly into the distance' > file
reader@ubuntu:/tmp$ grep 'distance' < file
Fly into the distance reader@ubuntu:/tmp$ echo 'Fly into the distance' | grep 'distance'Fly into the distance 
```

对于正常的重定向，我们首先将一些文本写入文件（使用输出重定向），然后将其用作`grep`的输入。接下来，我们做完全相同的功能性事情，但没有文件作为中间步骤。

基本上，管道语法如下：

```
command-with-output | command-using-input
```

你可以在一行中使用多个管道，并且可以使用任何管道和输入/输出重定向的组合，只要它有意义。

通常，当你使用超过两个管道/重定向时，你可以通过额外的行来提高可读性，也许使用命令替换将中间结果写入变量。但是，从技术上讲，你可以让它变得*尽可能复杂*；只是要注意不要让它变得*过于复杂*。

如前所述，管道将`stdout`绑定到`stdin`。你可能已经想到即将出现的问题：`stderr`！看看这个例子，它展示了输出分为`stdout`和`stderr`是如何影响管道的：

```
reader@ubuntu:~/scripts/chapter_12$ cat /etc/shadow | grep 'denied'
cat: /etc/shadow: Permission denied
reader@ubuntu:~/scripts/chapter_12$ cat /etc/shadow | grep 'denied' > /tmp/empty-file
cat: /etc/shadow: Permission denied #Printed to stderr on terminal.
reader@ubuntu:~/scripts/chapter_12$ cat /etc/shadow | grep 'denied' 2> /tmp/error-file
cat: /etc/shadow: Permission denied #Printed to stderr on terminal.
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/empty-file
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/error-file
```

起初，这个例子可能会让你感到困惑。让我们一步一步地来弄清楚它。

首先，`cat /etc/shadow | grep 'denied'`。我们尝试在`cat /etc/shadow`的`stdout`中查找单词`denied`。我们实际上并没有找到它，但我们还是在终端上看到了它的打印。为什么？因为尽管`stdout`被管道传输到`grep`，但`stderr`直接发送到我们的终端（并且**不**通过`grep`）。

如果你通过 SSH 连接到 Ubuntu 18.04，默认情况下，当 `grep` 成功时，你应该会看到颜色高亮；在这个例子中，你不会遇到这种情况。

下一个命令，`cat /etc/shadow | grep 'denied' > /tmp/empty-file`，将 `grep` 的 `stdout` 重定向到一个文件。由于 `grep` 没有处理错误消息，文件保持空。

即使我们尝试在最后重定向 `stderr`，正如在 `cat /etc/shadow | grep 'denied' 2> /tmp/error-file` 命令中所见，我们仍然不会在文件中得到任何输出。这是因为重定向**是顺序的**：输出重定向仅适用于 `grep`，而不适用于 `cat`。

现在，正如输出重定向有一种方法可以重定向 `stdout` 和 `stderr`，管道也有一种方法使用 `|&` 语法。再次看一下相同的示例，现在使用正确的重定向：

```
reader@ubuntu:~/scripts/chapter_12$ cat /etc/shadow |& grep 'denied'
cat: /etc/shadow: Permission denied
reader@ubuntu:~/scripts/chapter_12$ cat /etc/shadow |& grep 'denied' > /tmp/error-file
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/error-file 
cat: /etc/shadow: Permission denied
reader@ubuntu:~/scripts/chapter_12$ cat /etc/shadow |& grep 'denied' 2> /tmp/error-file
cat: /etc/shadow: Permission denied
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/error-file
```

对于第一个命令，如果你启用了颜色语法，你会看到单词 `denied` 是加粗并着色的（在我们的例子中，是红色）。这意味着现在我们使用 `|&`，`grep` 确实成功地处理了输出。

接下来，当我们使用 `grep` 的 `stdout` 进行重定向时，我们看到我们成功地将输出写入文件。如果我们尝试使用 `2>` 进行重定向，我们看到它在终端中再次打印，但没有在文件中。这是因为重定向的顺序性质：一旦 `grep` 成功处理了输入（来自 `stderr`），`grep` 就将此输出到 `stdout`。

`grep` 实际上并不知道输入最初是来自 `stderr` 流；在它看来，这只是一个需要处理的 `stdin`。由于对于 `grep` 来说，成功的处理结果会输出到 `stdout`，所以最终我们在那里找到它！

如果我们想要安全，并且不需要区分 `stdout` 和 `stderr` 的功能，最安全的方法是像这样使用命令：`cat /etc/shadow |& grep 'denied' &> /tmp/file`。由于管道和输出重定向都处理 `stdout` 和 `stderr`，我们总能确保所有输出都在我们想要的地方。

# 实际示例

由于管道的理论现在应该相对简单（当我们讨论输入和输出重定向时，我们已经解决了大部分问题），我们将展示一系列实际示例，这些示例真正展示了管道的强大功能。

记住，管道只对那些接受来自 `stdin` 输入的命令有效；并非所有命令都如此。如果你将某些内容管道传输到一个完全忽略该输入的命令，你可能会对结果感到失望。

既然我们已经介绍了管道，我们将在本书的其余部分更自由地使用它们。虽然这些示例将展示一些使用管道的方法，但本书的其余部分将包含更多！

# 又一个密码生成器

因此，我们已经创建了两个密码生成器。既然三是一个神奇的数字，而且这是一个展示管道链的绝佳示例，我们将再创建一个（最后一个，我保证）：

```
reader@ubuntu:~/scripts/chapter_12$ vim piped-passwords.sh
reader@ubuntu:~/scripts/chapter_12$ cat piped-passwords.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-10
# Description: Generate a password, using only pipes.
# Usage: ./piped-passwords.sh
#####################################

password=$(head /dev/urandom | tr -dc 'a-zA-Z0-9' | head -c20)

echo "Your random password is: ${password}"
```

首先，我们从`/dev/urandom`获取前 10 行（`head`的默认行为）。我们将其发送到`tr`，它将其修剪为我们想要的字符集（因为它也输出不可读的字符）。然后，当我们有一个可用的字符集时，我们再次使用`head`从中获取前 20 个字符。

如果您只运行`head /dev/urandom | tr -dc 'a-zA-Z0-9'`几次，您会看到长度不同；这是因为换行字节的随机性。通过从`/dev/urandom`获取 10 行，没有足够的可读字符来创建 20 个字符的密码的可能性非常小。

（挑战读者：创建一个循环脚本，足够长时间地执行此操作以遇到此情况！）

这个例子说明了几个问题。首先，我们通常可以用几个巧妙的管道实现很多我们想做的事情。其次，多次使用同一个命令并不罕见。顺便说一下，我们也可以选择`tail -c20`作为链中的最后一个命令，但这与整个命令有很好的对称性！

最后，我们看到了三个不同的密码生成器，实际上它们做的是同样的事情。正如在 Bash 中一样，有很多方法可以实现相同的目标；由你来决定哪一个最适用。就我们而言，可读性和性能应该是这个决定中的两个主要因素。

# 在脚本中设置密码

您可能想要编写脚本的另一项任务是为本地用户设置密码。虽然从安全角度来看，这并不总是好的做法（尤其是对于个人用户帐户），但它用于功能性帐户（对应于软件的用户，例如运行`httpd`进程的 Apache 用户）。

这些用户中的大多数不需要密码，但有时他们需要。在这种情况下，我们可以使用带有`chpasswd`命令的管道来设置他们的密码：

```
reader@ubuntu:~/scripts/chapter_12$ vim password-setter.sh 
reader@ubuntu:~/scripts/chapter_12$ cat password-setter.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-11-10
# Description: Set a password using chpasswd.
# Usage: ./password-setter.sh
#####################################

NEW_USER_NAME=bob

# Verify this script is run with root privileges.
if [[ $(id -u) -ne 0 ]]; then
  echo "Please run as root or with sudo!"
  exit 1
fi

# We only need exit status, send all output to /dev/null.
id ${NEW_USER_NAME} &> /dev/null

# Check if we need to create the user.
if [[ $? -ne 0 ]]; then
  # User does not exist, create the user.
  useradd -m ${NEW_USER_NAME}
fi

# Set the password for the user.
echo "${NEW_USER_NAME}:password" | chpasswd
```

在运行此脚本之前，请记住，这会在您的系统上添加一个用户，其密码非常简单（糟糕）。我们为这个脚本更新了输入消毒：我们使用命令替换来检查脚本是否以 root 权限运行。因为`id -u`返回用户的数字 ID，对于 root 用户或 sudo 权限，它应该是 0，我们可以使用`-ne 0`进行比较。

如果我们运行脚本并且用户不存在，我们会在设置该用户的密码之前创建该用户。这是通过将`username:password`发送到`chpasswd`的`stdin`，通过管道实现的。请注意，我们使用了`-ne 0`两次，但用于非常不同的事情：第一次用于比较用户 ID，第二次用于退出状态。

你可能能想到对这个脚本进行多种改进。例如，能够指定用户名和密码而不是这些硬编码的占位值可能是个好主意。此外，在`chpasswd`命令之后进行健全性检查绝对是个好主意。在当前版本中，脚本没有给用户**任何**反馈；这是非常糟糕的做法。

看看你是否能解决这些问题，并确保记住，用户提供的任何输入都应该进行*彻底*检查！如果你真的想挑战自己，可以在一个`for`循环中为多个用户执行此操作，方法是从文件中获取输入。

需要注意的是，当进程运行时，系统上的任何用户都可以看到它。这通常不是什么大问题，但如果你直接将用户名和密码作为参数提供给脚本，那么这些信息也会对所有人可见。尽管这种情况通常只会持续很短的时间，但它们仍然会暴露。在处理如密码等敏感问题时，始终要牢记安全性。

# tee

一个看似为与管道协同工作而创建的命令是`tee`。手册页上的描述应该能说明大部分情况：

tee - 从标准输入读取并写入到标准输出和文件

因此，本质上，通过管道将某些内容发送到`tee`的`stdin`，允许我们将输出同时保存到终端和文件中。

这在使用交互式命令时通常最有用；它允许你实时跟踪输出，同时也将其写入（日志）文件以便稍后审查。系统更新提供了一个很好的`tee`使用案例：

```
sudo apt upgrade -y | tee /tmp/upgrade.log
```

我们可以通过将*所有*输出发送到`tee`，包括`stderr`，使其变得更好：

```
sudo apt upgrade -y |& tee /tmp/upgrade.log
```

输出将看起来像这样：

```
reader@ubuntu:~/scripts/chapter_12$ sudo apt upgrade -y |& tee /tmp/upgrade.log
WARNING: apt does not have a stable CLI interface. Use with caution in scripts.
Reading package lists...
<SNIPPED>
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
reader@ubuntu:~/scripts/chapter_12$ cat /tmp/upgrade.log 
WARNING: apt does not have a stable CLI interface. Use with caution in scripts.
Reading package lists...
<SNIPPED>
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

终端输出和日志文件的第一行是一个发送到`stderr`的`WARNING`；如果你使用的是`|`而不是`|&`，那么它就不会被写入日志文件，只会在屏幕上显示。如果你按照建议使用`|&`，你会发现屏幕上的输出和文件内容是完全匹配的。

默认情况下，`tee`会覆盖目标文件。与所有重定向形式一样，`tee`也有一种方式可以追加而不是覆盖：使用`--append`（`-a`）标志。根据我们的经验，这通常是一个明智的选择，与`|&`没有太大不同。

尽管`tee`是命令行工具库中的一个强大工具，它在脚本编写中同样有其用武之地。一旦你的脚本变得更加复杂，你可能希望将部分输出保存到文件中以便稍后审查。然而，为了保持用户对脚本状态的更新，将一些信息打印到终端也可能是个好主意。如果这两种情况重叠，你就需要使用`tee`来完成任务！

# 此处文档

本章我们将介绍的最后一个概念是*here document*。here document，也称为 heredocs，用于向某些命令提供输入，与 `stdin` 重定向略有不同。值得注意的是，它是向命令提供多行输入的一种简单方法。它使用以下语法：

```
cat << EOF
input
more input
the last input
EOF
```

如果你在终端中运行这个，你会看到以下内容：

```
reader@ubuntu:~/scripts/chapter_12$ cat << EOF
> input
> more input
> the last input
> EOF
input
more input
the last input
```

`<<` 语法让 Bash 知道你想要使用一个 heredoc。紧接着，你提供了一个*分隔标识符*。这可能看起来很复杂，但实际上意味着你提供了一个字符串，该字符串将终止输入。因此，在我们的例子中，我们提供了常用的 `EOF`（代表**结束**文件**结束**）。

现在，如果 heredoc 在输入中遇到与分隔标识符完全匹配的行，它将停止接收进一步的输入。这里有一个更接近的例子来说明这一点：

```
reader@ubuntu:~/scripts/chapter_12$ cat << end-of-file
> The delimiting identifier is end-of-file
> But it only stops when end-of-file is the only thing on the line
> end-of-file does not work, since it has text after it
> end-of-file
The delimiting identifier is end-of-file
But it only stops when end-of-file is the only thing on the line
end-of-file does not work, since it has text behind it
```

虽然使用 `cat` 说明了这一点，但它并不是一个非常实用的例子。然而，`wall` 命令是。`wall` 允许你向连接到服务器的每个人广播消息，到他们的终端。当与 heredoc 结合使用时，它看起来有点像这样：

```
reader@ubuntu:~/scripts/chapter_12$ wall << EOF
> Hi guys, we're rebooting soon, please save your work!
> It would be a shame if you lost valuable time...
> EOF

Broadcast message from reader@ubuntu (pts/0) (Sat Nov 10 16:21:15 2018):

Hi guys, we're rebooting soon, please save your work!
It would be a shame if you lost valuable time...
```

在这种情况下，我们收到自己的广播。但是，如果你使用你的用户多次连接，你也会在那里看到广播。

尝试使用终端控制台连接和 SSH 连接同时进行；如果你亲眼看到它，你会更好地理解它。

# Heredocs 和变量

使用 heredocs 时经常出现的混淆来源是使用变量。默认情况下，变量在 heredoc 中被解析，如下例所示：

```
reader@ubuntu:~/scripts/chapter_12$ cat << EOF
> Hi, this is $USER!
> EOF
Hi, this is reader!
```

然而，这可能并不总是理想的功能。你可能想使用它来写入一个文件，其中变量应该在以后解析。

在这种情况下，我们可以引用分隔标识符 EOF 以防止变量被替换：

```
reader@ubuntu:~/scripts/chapter_12$ cat << 'EOF'
> Hi, this is $USER!
> EOF
Hi, this is $USER!
```

# 使用 heredocs 进行脚本输入

由于 heredocs 允许我们简单地将以换行符分隔的输入传递给命令，我们可以使用它以非交互方式运行交互式脚本！我们在实践中使用了这一点，例如，在只能以交互方式运行的数据库安装脚本上。但是，一旦你知道问题的顺序和你想要提供的输入，你就可以使用 heredoc 将此输入提供给该交互式脚本。

更好的是，我们已经创建了一个使用交互式输入的脚本，`/home/reader/scripts/chapter_11/while-interactive.sh`，我们可以用它来展示这个功能：

```
reader@ubuntu:/tmp$ head /home/reader/scripts/chapter_11/while-interactive.sh
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.1.0
# Date: 2018-10-28
# Description: A simple riddle in a while loop.
# Usage: ./while-interactive.sh
#####################################

reader@ubuntu:/tmp$ bash /home/reader/scripts/chapter_11/while-interactive.sh << EOF
a mouse  #Try 1.
the sun  #Try 2.
keyboard #Try 3.
EOF

Incorrect, please try again. #Try 1.
Incorrect, please try again. #Try 2.
Correct, congratulations!    #Try 3.
Now we can continue after the while loop is done, awesome!
```

我们知道脚本会一直运行，直到得到正确答案，即 `keyboard` 或 `Keyboard`。我们使用 heredoc 按顺序发送三个答案给脚本：`a mouse`、`the sun`，最后是 `keyboard`。我们可以很容易地将输出与输入对应起来。

为了更详细地了解，可以运行带有 heredoc 输入的脚本，并使用 `bash -x`，这将明确显示谜题有三次尝试。

你可能想在嵌套函数（将在下一章解释）或循环内部使用此处文档。在这两种情况下，你都应该已经使用缩进来提高可读性。然而，这会影响你的 heredoc，因为空白被认为是输入的一部分。如果你发现自己处于这种情况，heredocs 有一个额外的选项：使用`<<-`而不是`<<`。当提供额外的`-`时，所有*制表符*都会被忽略。这允许你用制表符缩进 heredoc 结构，这既保持了可读性又保持了功能。

# 此处字符串

本章我们最后要讨论的是*此处字符串*。它与此处文档非常相似（因此得名），但它处理的是单个字符串，而不是文档（谁会想到呢！）。

这种使用`<<<`语法的结构，可以用来向可能通常只接受来自`stdin`或文件输入的命令提供文本输入。一个很好的例子是`bc`，它是一个简单的计算器（属于 GNU 项目的一部分）。

通常，你以两种方式使用它：通过管道将输入发送到`stdin`，或者通过指向`bc`到一个文件：

```
reader@ubuntu:/tmp$ echo "2^8" | bc
256

reader@ubuntu:/tmp$ echo "4*4" > math
reader@ubuntu:/tmp$ bc math
bc 1.07.1
Copyright 1991-1994, 1997, 1998, 2000, 2004, 2006, 2008, 2012-2017 Free Software Foundation, Inc.
This is free software with ABSOLUTELY NO WARRANTY.
For details type `warranty'. 
16
^C
(interrupt) use quit to exit.
quit
```

当与`stdin`一起使用时，`bc`返回计算结果。当与文件一起使用时，`bc`打开一个交互式会话，我们需要手动关闭它，方法是输入`quit`。这两种方式似乎对于我们想要实现的目标来说有点过于繁琐。

让我们看看此处字符串是如何解决这个问题的：

```
reader@ubuntu:/tmp$ bc <<< 2^8
256
```

就是这样。只是一个简单的输入字符串（发送到命令的`stdin`），我们得到了与使用管道的`echo`相同的功能。但是，现在只是一个命令，而不是一个链。简单但有效，正是我们喜欢的方式！

# 总结

这一章几乎解释了关于 Linux 上*重定向*的所有知识。我们从对重定向的一般描述开始，以及如何使用*文件描述符*来促进重定向。我们了解到文件描述符 0、1 和 2 分别用于`stdin`、`stdout`和`stderr`。

然后我们熟悉了重定向的语法。这包括`>`、`2>`、`&>`和`<`，以及它们的追加语法，`>>`、`2>>`、`&>>`和`<<`。

我们讨论了一些特殊的 Linux 设备，`/dev/null`、`/dev/zero`和`/dev/urandom`。我们展示了如何使用这些设备来删除输出、生成空字节和生成随机数据的示例。在高级重定向部分，我们展示了我们可以将`stdout`绑定到`stderr`，反之亦然。

此外，我们了解了*命令替换*和*进程替换*，它允许我们在另一个命令的参数中使用命令的结果，或者作为文件。

接下来是*管道*。管道是简单但非常强大的 Bash 结构，用于将一个命令的`stdout`（可能还有`stderr`）连接到另一个命令的`stdin`。这使我们能够链接命令，通过尽可能多的命令来进一步操作数据流。

我们还介绍了`tee`，它允许我们将流发送到我们的终端和一个文件，这种结构通常用于日志文件。

最后，我们解释了*文档*和*字符串*。这些概念允许我们将多行和单行输入直接从终端发送到其他命令的`stdin`，否则需要`echo`或`cat`。

本章介绍了以下命令：`diff`、`gcc`、`fallocate`、`tr`、`chpasswd`、`tee`和`bc`。

# 问题

1.  文件描述符是什么？

1.  术语`stdin`、`stdout`和`stderr`是什么意思？

1.  `stdin`、`stdout`和`stderr`如何映射到默认文件描述符？

1.  `>`、`1>`和`2>`之间的输出重定向有什么区别？

1.  `>`和`>>`之间有什么区别？

1.  如何同时重定向`stdout`和`stderr`？

1.  哪些特殊设备可以用作输出的黑洞？

1.  管道在重定向方面有什么作用？

1.  我们如何将输出发送到终端和日志文件？

1.  here string 的典型用例是什么？

# 进一步阅读

+   请点击以下链接了解有关文件描述符的更多信息：[`linuxmeerkat.wordpress.com/2011/12/02/file-descriptors-explained/`](https://linuxmeerkat.wordpress.com/2011/12/02/file-descriptors-explained/)。

+   请点击以下链接了解有关使用文件描述符的高级脚本的信息：[`bash.cyberciti.biz/guide/Reads_from_the_file_descriptor_(fd)`](https://bash.cyberciti.biz/guide/Reads_from_the_file_descriptor_(fd))。

+   请点击以下链接了解有关命令替换的更多信息：[`www.tldp.org/LDP/abs/html/commandsub.html`](http://www.tldp.org/LDP/abs/html/commandsub.html)。

+   请点击以下链接了解有关 here documents 的信息：[`www.tldp.org/LDP/abs/html/here-docs.html`](https://www.tldp.org/LDP/abs/html/here-docs.html)。
