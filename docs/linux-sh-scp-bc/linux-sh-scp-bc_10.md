# 第十章：脚本最佳实践

本章解释了一些实践和技术，这些实践和技术将帮助读者成为更好、更高效的程序员。

在本章中，我们将讨论我认为是脚本（或编程）最佳实践。自 1977 年以来，我一直在编程计算机，积累了相当丰富的经验。我很高兴教人们有关计算机的知识，希望我的想法能对一些人有所帮助。

涵盖的主题如下：

+   备份将再次被讨论，包括验证

+   我将解释如何选择一个你感到舒适的文本编辑器，并了解它的功能

+   我将涵盖一些基本的命令行项目，比如使用良好的提示符、命令完成、环境变量和别名

+   我将提供一些额外的脚本

# 验证备份

我已经在本书中至少两次谈到了备份，这将是我承诺的最后一次。创建您的备份脚本，并确保它们在应该运行时运行。但我还没有谈到的一件事是验证备份。您可能有 10 太拉夸德的备份存放在某个地方，但它们真的有效吗？您上次检查是什么时候？

使用`tar`命令时，它会在运行结束时报告是否遇到任何问题制作存档。一般来说，如果没有显示任何问题，备份可能是好的。使用带有`-t（tell）`选项的`tar`，或者在本地或远程机器上实际提取它，也是确定存档是否成功制作的好方法。

### 注意

注意：在使用 tar 时一个常见的错误是将当前正在更新的文件包含在备份中。

这是一个相当明显的例子：

```
guest1 /home # tar cvzf guest1.gz guest1/ | tee /home/guest1/temp/mainlogs`date '+%Y%m%d'`.gz
```

`tar`命令可能不认为这是一个错误，但通常会报告，所以一定要检查一下。

另一个常见的备份错误是不将文件复制到另一台计算机或外部设备。如果您擅长备份，但它们都在同一台机器上，最终硬盘和/或控制器将会失败。您可能能够恢复数据，但为什么要冒险呢？将文件复制到至少一个外部驱动器和/或计算机上，保险起见。

我将提到备份的最后一件事。确保您将备份发送到离岗位置，最好是在另一个城市、州、大陆或行星上。对于您宝贵的数据，您真的不能太小心。

# ssh 和 scp

使用`scp`到远程计算机也是一个非常好的主意，我的备份程序每天晚上也会这样做。以下是如何设置无人值守`ssh`/`scp`。在这种情况下，机器 1（M1）上的 root 帐户将能够将文件`scp`到机器 2（M2）上的 guest1 帐户。我之所以这样做，是因为出于安全原因，我总是在所有的机器上禁用`ssh`/`scp`的 root 访问。

1.  首先确保在每台机器上至少运行了一次`ssh`。这将设置一些必要的目录和文件。

1.  在 M1 上，在`root`下，运行`ssh-keygen -t rsa`命令。这将在`/root/.ssh`目录中创建文件`id_rsa.pub`。

1.  使用`scp`将该文件复制到 M2 的`/tmp`目录（或其他适当的位置）。

1.  在 M2 中转到`/home/guest1/.ssh`目录。

1.  如果已经有一个`authorized_keys`文件，请编辑它，否则创建它。

1.  将`/tmp/id_rsa.pub`文件中的行复制到`authorized_keys`文件中并保存。

通过使用`scp`将文件从 M1 复制到 M2 进行测试。它应该可以在不提示输入密码的情况下工作。如果有任何问题，请记住，这必须为每个想要执行无人值守`ssh`/`scp`的用户设置。

如果您的**互联网服务提供商**（**ISP**）为您的帐户提供 SSH，这种方法也可以在那里使用。我一直在使用它，它真的很方便。使用这种方法，您可以让脚本生成一个 HTML 文件，然后将其直接复制到您的网站上。动态生成 HTML 页面是程序真正擅长的事情。

# 找到并使用一个好的文本编辑器

如果你只是偶尔写脚本或程序，那么 vi 可能对你来说已经足够了。然而，如果你进行了一些真正深入的编程，无论是在 Bash、C、Java 还是其他语言，你都应该非常确定地了解一些其他可用的 Linux 文本编辑器。你几乎肯定会变得更有生产力。

正如我之前提到的，我已经使用计算机工作了很长时间。我最开始在 DOS 上使用一个叫做 Edlin 的编辑器，它相当弱（但仍然比穿孔卡好）。我最终转而开始在 AIX（IBM 的 UNIX 版本）上使用 vi。我在使用 vi 方面变得相当擅长，因为当时我们还没有其他选择。随着时间的推移，其他选择变得可用，我开始使用 IBM 个人编辑器。这些非常容易使用，比 vi 更高效，并且具有更多功能。随着我进行了越来越多的编程，我发现这些编辑器都不能满足我想要的一切，所以我用 C 编程语言编写了自己的编辑器。这是很久以前在 DOS 下，然而，我的编辑器现在已经被修改以在 Xenix、OS/2、AIX、Solaris、UNIX、FreeBSD、NetBSD 和当然 Linux 上运行。它在 Cygwin 环境下的 Windows 上也运行良好。

任何文本编辑器都应该具有标准功能，如复制、粘贴、移动、插入、删除、拆分、合并、查找/替换等。这些应该易于使用，不需要超过两个按键。`保存`命令只需要一个按键。

此外，一个好的编辑器还应该具有以下一个或多个功能：

+   能够同时编辑多个文件（文件环）

+   能够用单个按键切换到环中的下一个或上一个文件

+   能够显示环中的文件并立即切换到任何文件

+   能够将文件插入当前文件

+   能够记录和回放记住的按键序列。有时这被称为宏

+   撤销/恢复功能

+   自动保存文件选项

+   一个锁定文件的功能，以防止在编辑器的另一个实例中编辑同一个文件

+   绝对没有明显的缺陷或错误。这是强制性的

+   通过心灵感应接受输入

嗯，也许我还没有完全弄清楚最后一个。当然还有许多许多其他功能可以列出，但我觉得这些是最重要的。

这是我的编辑器的截图，显示了`ring`命令可能的样子：

![查找和使用好的文本编辑器](img/B07040_10_01.jpg)

还有很多功能可以展示，但这应该足以表达观点。我会提到 vi 是一个很好的编辑器，可能大多数 UNIX/Linux 用户都成功地使用它。然而，根据我的经验，如果要进行大量的编程，使用具有更多功能的不同编辑器将节省大量时间。这也更容易一些，这使得整个过程更有趣。

# 环境变量和别名

环境变量在第二章中有介绍，*变量处理*。这是我多年前学到的一个很酷的技巧，可以在使用命令行时真正帮助。大多数 Linux 系统通常在`$HOME`下有几个标准目录，如桌面、下载、音乐、图片等。我个人不喜欢一遍又一遍地输入相同的东西，所以这样做可以帮助更有效地使用系统。以下是我添加到`/home/guest1/.bashrc`文件的一些行：

```
export BIN=$HOME/bin
alias bin="cd $BIN"

export DOWN=$HOME/Downloads
alias down="cd $DOWN"

export DESK=$HOME/Desktop
alias desk="cd $DESK"

export MUSIC=$HOME/Music
alias music="cd $MUSIC"

export PICTURES=$HOME/Pictures
alias pictures="cd $PICTURES"

export BOOKMARKS=$HOME/Bookmarks
alias bookmarks="cd $BOOKMARKS"

# Packt- Linux Scripting Bootcamp
export LB=$HOME/LinuxScriptingBook
alias lb="cd $LB"

# Source lbcur
. $LB/source.lbcur.txt
```

使用这种方法，你可以通过只输入小写别名来 cd 到上述任何一个目录。更好的是，你还可以通过使用大写导出的环境变量来复制或移动文件到目录中或从目录中。看看下面的截图：

![环境变量和别名](img/B07040_10_02.jpg)

我花了好几年的时间才开始做这件事，我仍然为自己没有早点发现而感到后悔。记住将别名设为小写，环境变量设为大写，你就可以开始了。

注意我在“书签”目录中运行的命令。我实际上输入了`mv $DESK/`然后按了*Tab*键。这导致该行自动完成，然后我添加了句点`.`字符并按下*Enter*。

记住尽可能使用命令自动完成，这样可以节省大量时间。

需要解释的是`. $LB/source.lbcur.txt`这一行。你可以看到我有一个`lbcur`别名，它让我进入我当前撰写本书的目录。由于我同时使用我的 root 和`guest1`账户来写书，我只需在`source.lbcur.txt`文件中更改章节号。然后我为 root 和`guest1`源`.bashrc`文件，就完成了。否则，我将不得不在每个`.bashrc`文件中进行更改。也许只有两个文件可能不会那么糟糕，但假设你有几个用户呢？我在我的系统上经常使用这种技术，因为我是一个非常懒的打字员。

记住：当使用别名和环境变量时，需要在终端中更改之前先源用户的`.bashrc`文件。

# ssh 提示

当我运行 Linux 系统时，我倾向于至少打开 30 个终端窗口。其中一些登录到我家的其他机器上。在撰写本文时，我已登录到 laptop1、laptop4 和 gabi1（我女朋友运行 Fedora 20 的笔记本电脑）。我发现很久以前，如果这些终端的提示不同，我很难弄清楚并在错误的计算机上输入正确的命令。不用说，那可能是一场灾难。有一段时间我会手动更改提示，但这很快就厌倦了。有一天我几乎偶然发现了这个问题的一个非常酷的解决方案。我在 Red Hat Enterprise Linux、Fedora 和 CentOS 上使用了这种技术，所以它也应该适用于您的系统（可能需要稍微调整）。

这些行在我所有系统的`$HOME/.bashrc`文件中：

```
# Modified 1/17/2014
set | grep XAUTHORITY
rc=$?
if [ $rc -eq 0 ] ; then
 PS1="\h \w # "
else
 PS1="\h \h \h \h \w # "
fi
```

所以这个命令使用 set 命令来 grep 字符串`XAUTHORITY`。这个字符串只存在于本地机器的环境中。因此，当你在 big1 本地打开终端时，它使用正常的提示。然而，如果你`ssh`到另一个系统，该字符串就不存在，因此它使用长扩展提示。

这是我系统的屏幕截图：

![ssh 提示](img/B07040_10_03.jpg)

# 测试一个存档

这是我在几个计算机工作中遇到的问题。我的经理会要求我接手一个同事的项目。他会将文件`zip`或`tar`起来，然后给我存档。我会在我的系统上解压缩它并尝试开始工作。但总会有一个文件丢失。通常需要两次、三次或更多次尝试，我才最终拥有编译项目所需的每个文件。所以，这个故事的教训是，当制作一个要交给别人的存档时，一定要确保将其复制到另一台机器上并在那里进行测试。只有这样，你才能相对确定地包含了每个文件。

# 进度指示器

这是另一个光标移动脚本，它还计算了`$RANDOM` Bash 变量的低和高。这可能对每个人来说看起来并不那么酷，但它确实展示了我们在本书中涵盖的更多概念。我也对那个随机数生成器的范围有些好奇。

## 第十章 - 脚本 1

```
#!/bin/sh
#
# 6/11/2017
# Chapter 10 - Script 1
#

# Subroutines
trap catchCtrlC INT          # Initialize the trap

# Subroutines
catchCtrlC()
{
 loop=0                      # end the loop
}

cls()
{
 tput clear
}

movestr()                    # move cursor to row, col, display string
{
 tput cup $1 $2
 echo -n "$3"
}

# Code
if [ "$1" = "--help" ] ; then
 echo "Usage: script1 or script1 --help "
 echo " Shows the low and high count of the Bash RANDOM variable."
 echo " Press Ctrl-C to end."
 exit 255
fi

sym[0]='|'
sym[1]='/'
sym[2]='-'
sym[3]='\'

low=99999999
high=-1

cls
echo "Chapter 10 - Script 1"
echo "Calculating RANDOM low and high ..."
loop=1
count=0
x=0
while [ $loop -eq 1 ]
do
 r=$RANDOM
 if [ $r -lt $low ] ; then
  low=$r
 elif [ $r -gt $high ] ; then
  high=$r
 fi

# Activity indicator
 movestr 2 1 "${sym[x]}"     # row 2 col 1
 let x++
 if [ $x -gt 3 ] ; then
  x=0
 fi

 let count++
done

echo " "                     # carriage return
echo "Number of loops: $count"
echo "low: $low  high: $high"

echo "End of script1"
exit 0
```

我系统上的输出：

![第十章 - 脚本 1](img/B07040_10_04.jpg)

# 从模板创建新命令

由于您正在阅读本书，可以假定您将要编写大量脚本。这是我多年来学到的另一个方便的技巧。当我需要创建一个新脚本时，我不是从头开始做，而是使用这个简单的命令：

## 第十章 - 脚本 2

```
#!/bin/sh
#
# 1/26/2014
#
# create a command script

if [ $# -eq 0 ] ; then
 echo "Usage: mkcmd command"
 echo " Copies mkcmd.template to command and edits it with kw"
 exit 255
fi

if [ -f $1 ] ; then
  echo File already exists!
  exit 2
fi

cp $BIN/mkcmd.template $1
kw $1
exit 0

And here is the contents of the $BIN/mkcmd.template file:
#!/bin/sh
#
# Date
#
if [ $# -eq 0 ] ; then
 echo "Usage:                "
 echo "                      "
 exit 255
fi
```

确保在创建`mkcmd.template`文件后对其运行`chmod 755`。这样你就不必每次都记得这样做了。事实上，这就是我写这个脚本的主要原因。

随意修改这个脚本，当然也可以将`kw`更改为您正在使用的 vi 或其他编辑器。

# 提醒用户

当重要任务完成并且您想立刻知道时，让您的计算机响铃是很好的。以下是我用来响铃我的计算机内部扬声器的脚本：

### 第十章 - 脚本 3

```
#!/bin/sh
#
# 5/3/2017
#
# beep the PC speaker

lsmod | grep pcspkr > /dev/null
rc=$?
if [ $rc -ne 0 ] ; then
 echo "Please modprobe pcspkr and try again."
 exit 255
fi

echo -e '\a' > /dev/console
```

这个命令会响铃 PC 扬声器（如果有的话），并且驱动程序已经加载。请注意，这个命令可能只有在以 root 用户身份运行时才能在您的系统上工作。

# 总结

在这最后一章中，我展示了一些我学到的编程最佳实践。讨论了一个好的文本编辑器的特性，并包括了一个`$RANDOM`测试脚本。我还介绍了我多年来编写的一些脚本，以使我的系统更高效、更易于使用。
