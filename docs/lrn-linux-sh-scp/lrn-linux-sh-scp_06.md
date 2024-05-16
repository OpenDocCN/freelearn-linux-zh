# 第六章：文件操作

本章专门讨论文件操作。就像*一切都是文件*系统一样，文件操作被认为是与 Linux 工作中最重要的方面之一。我们将首先探讨常见的文件操作，比如创建、复制和删除文件。接着我们会介绍一些关于存档的内容，这是在命令行工作时的另一个重要工具。本章的最后部分将致力于在文件系统中查找文件，这是 shell 脚本工具包中的另一个重要技能。

本章将介绍以下命令：`cp`、`rm`、`mv`、`ln`、`tar`、`locate`和`find`。

本章将涵盖以下主题：

+   常见的文件操作

+   存档

+   查找文件

# 技术要求

我们将使用我们在第二章中创建的虚拟机进行文件操作。此时不需要更多资源。

# 常见的文件操作

到目前为止，我们主要介绍了与 Linux 文件系统导航相关的命令。在早期的章节中，我们已经看到我们可以使用`mkdir`和`touch`分别创建目录和空文件。如果我们想给文件一些有意义的（文本）内容，我们使用`vim`或`nano`。然而，我们还没有讨论过删除文件或目录，或者复制、重命名或创建快捷方式。让我们从复制文件开始。

# 复制

实质上，在 Linux 上复制文件非常简单：使用`cp`命令，后面跟着要复制的文件名和要复制到的文件名。它看起来像这样：

```
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 testfile
drwxrwx--- 2 reader reader 4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games     0 Aug  4 16:18 umaskfile
reader@ubuntu:~$ cp testfile testfilecopy
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 testfile
-rwxr-xr-- 1 reader reader    0 Aug 18 14:00 testfilecopy
drwxrwx--- 2 reader reader 4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games     0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

正如你所看到的，在这个例子中，我们复制了一个（空的）*文件*，它已经是*我们拥有的*，而我们*在相同的目录*中。这可能会引发一些问题，比如：

+   我们是否总是需要在源文件和目标文件的相同目录中？

+   文件的权限呢？

+   我们是否也可以使用`cp`复制目录？

正如你所期望的，在 Linux 下，`cp`命令也是非常多才多艺的。我们确实可以复制不属于我们的文件；我们不需要在与文件相同的目录中，我们也可以复制目录！让我们尝试一些这些事情：

```
reader@ubuntu:~$ cd /var/log/
reader@ubuntu:/var/log$ ls -l
total 3688
<SNIPPED>
drwxr-xr-x  2 root      root               4096 Apr 17 20:22 dist-upgrade
-rw-r--r--  1 root      root             550975 Aug 18 13:35 dpkg.log
-rw-r--r--  1 root      root              32160 Aug 11 10:15 faillog
<SNIPPED>
-rw-------  1 root      root              64320 Aug 11 10:15 tallylog
<SNIPPED>
reader@ubuntu:/var/log$ cp dpkg.log /home/reader/
reader@ubuntu:/var/log$ ls -l /home/reader/
total 552
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 testfile
-rwxr-xr-- 1 reader reader      0 Aug 18 14:00 testfilecopy
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:/var/log$ cp tallylog /home/reader/
cp: cannot open 'tallylog' for reading: Permission denied
reader@ubuntu:/var/log$
```

那么，发生了什么？我们使用`cd`命令将目录更改为`/var/log/`。我们使用带有*长*选项的`ls`列出了那里的文件。我们复制了一个相对路径的文件，我们能够读取它，但它是由`root:root`拥有的，复制到了完全限定的`/home/reader/`目录。当我们使用完全限定路径列出`/home/reader/`时，我们看到复制的文件现在由`reader:reader`拥有。当我们尝试对`tallylog`文件做同样的操作时，我们得到了错误`cannot open 'tallylog' for reading: Permission denied`。这并不意外，因为我们对该文件没有任何读取权限，所以复制会很困难。

这应该回答了三个问题中的两个。但是对于目录呢？让我们尝试将`/tmp/`目录复制到我们的`home`目录中：

```
reader@ubuntu:/var/log$ cd
reader@ubuntu:~$ cp /tmp/ .
cp: -r not specified; omitting directory '/tmp/'
reader@ubuntu:~$ ls -l
total 552
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 testfile
-rwxr-xr-- 1 reader reader      0 Aug 18 14:00 testfilecopy
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:~$ cp -r /tmp/ .
cp: cannot access '/tmp/systemd-private-72bcf47b69464914b021b421d5999bbe-systemd-timesyncd.service-LeF05x': Permission denied
cp: cannot access '/tmp/systemd-private-72bcf47b69464914b021b421d5999bbe-systemd-resolved.service-ApdzhW': Permission denied
reader@ubuntu:~$ ls -l
total 556
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 testfile
-rwxr-xr-- 1 reader reader      0 Aug 18 14:00 testfilecopy
drwxrwxr-t 9 reader reader   4096 Aug 18 14:38 tmp
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

对于这样一个简单的练习，实际上发生了很多事情！首先，我们使用`cd`命令返回到我们的`home`目录，而不带任何参数；这本身就是一个很巧妙的小技巧。接下来，我们尝试将整个`/tmp/`目录复制到`.`（你应该记得，`.`是*当前目录*的缩写）。然而，这次失败了，出现了错误`-r not specified; omitting directory '/tmp/'`。我们列出目录来检查，确实，似乎什么都没发生。当我们添加了错误指定的`-r`并重新尝试命令时，出现了一些`Permission denied`的错误。这并不意外，因为并非所有`/tmp/`目录中的文件都对我们可读。尽管我们得到了错误，但当我们现在检查我们的`home`目录的内容时，我们可以看到`tmp`目录在那里！因此，使用`-r`选项（它是`--recursive`的缩写）允许我们复制目录和其中的所有内容。

# 删除

在将一些文件和目录复制到我们的`home`目录之后（这是一个安全的选择，因为我们确信可以在那里写入！），我们留下了一些混乱。让我们使用`rm`命令来删除一些重复的项目：

```
reader@ubuntu:~$ ls -l
total 556
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 testfile
-rwxr-xr-- 1 reader reader      0 Aug 18 14:00 testfilecopy
drwxrwxr-t 9 reader reader   4096 Aug 18 14:38 tmp
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:~$ rm testfilecopy
reader@ubuntu:~$ rm tmp/
rm: cannot remove 'tmp/': Is a directory
reader@ubuntu:~$ rm -r tmp/
reader@ubuntu:~$ ls -l
total 552
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 testfile
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

使用`rm`后跟文件名将删除它。您可能会注意到，这里没有“您确定吗？”的提示。实际上，可以通过使用`-i`标志来启用此功能，但默认情况下不是这样。请注意，`rm`还允许您使用通配符，例如`*`（匹配所有内容），这将删除所有匹配的文件（并且可以被用户删除）。简而言之，这是一种非常快速丢失文件的好方法！但是，当我们尝试使用`rm`命令和目录名称时，它会给出错误`cannot remove 'tmp/': Is a directory`。这与`cp`命令非常相似，幸运的是，解决方法也是一样的：添加`-r`进行*递归*删除！同样，这是一种丢失文件的好方法；一个命令就可以让您删除整个`home`目录及其中的所有内容，而不需要任何警告。请把这当作是您的警告！特别是在与`-f`标志结合使用时，它是`--force`的缩写，这将确保`rm`*永远不会提示*并立即开始删除。

# 重命名、移动和链接

有时，我们不仅想要创建或删除文件，还可能需要重命名文件。奇怪的是，Linux 没有任何听起来像重命名的东西；但是，`mv`命令（用于**m**o**v**e）确实实现了我们想要的功能。与`cp`命令类似，它接受源文件和目标文件作为参数，并且看起来像这样：

```
reader@ubuntu:~$ ls -l
total 552
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 testfile
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:~$ mv testfile renamedtestfile
reader@ubuntu:~$ mv testdir/ renamedtestdir
reader@ubuntu:~$ ls -l
total 552
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 renamedtestdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 renamedtestfile
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

正如您所看到的，`mv`命令非常简单易用。它甚至适用于目录，无需像`cp`和`rm`那样需要`-r`这样的特殊选项。但是，当我们引入通配符时，它会变得更加复杂，但现在不用担心。我们在前面的代码中使用的命令是相对的，但它们也可以完全限定或混合使用。

有时，您可能需要将文件从一个目录移动到另一个目录。如果您仔细考虑，这实际上是对完全限定文件名的重命名！没有触及任何数据，但您只是想在其他地方访问文件。因此，使用`mv umaskfile umaskdir/`将`umaskfile`移动到`umaskdir/`：

```
reader@ubuntu:~$ ls -l
total 16
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 renamedtestdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 renamedtestfile
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:~$ mv umaskfile umaskdir/
reader@ubuntu:~$ ls -l
total 16
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 renamedtestdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 renamedtestfile
drwxrwx--- 2 reader reader   4096 Aug 19 10:37 umaskdir
reader@ubuntu:~$ ls -l umaskdir/
total 0
-rw-rw---- 1 reader games 0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

最后，我们有`ln`命令，它代表**l**i**n**king。这是 Linux 创建文件之间链接的方式，最接近 Windows 使用的快捷方式。有两种类型的链接：符号链接（也称为软链接）和硬链接。区别在于文件系统的工作原理：符号链接指向文件名（或目录名），而硬链接链接到存储文件或目录内容的*inode*。对于脚本编写，如果您使用链接，您可能正在使用符号链接，因此让我们看看这些符号链接的操作：

```
reader@ubuntu:~$ ls -l
total 552
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 renamedtestdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 renamedtestfile
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:~$ ln -s /var/log/auth.log 
reader@ubuntu:~$ ln -s /var/log/auth.log link-to-auth.log
reader@ubuntu:~$ ln -s /tmp/
reader@ubuntu:~$ ln -s /tmp/ link-to-tmp
reader@ubuntu:~$ ls -l
total 552
lrwxrwxrwx 1 reader reader     17 Aug 18 15:07 auth.log -> /var/log/auth.log
-rw-r--r-- 1 reader reader 550975 Aug 18 14:20 dpkg.log
lrwxrwxrwx 1 reader reader     17 Aug 18 15:08 link-to-auth.log -> /var/log/auth.log
lrwxrwxrwx 1 reader reader      5 Aug 18 15:08 link-to-tmp -> /tmp/
-rw-rw-r-- 1 reader reader     69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader   4096 Aug  4 16:16 renamedtestdir
-rwxr-xr-- 1 reader reader      0 Aug  4 13:44 renamedtestfile
lrwxrwxrwx 1 reader reader      5 Aug 18 15:08 tmp -> /tmp/
drwxrwx--- 2 reader reader   4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games       0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

我们使用`ln -s`（这是`--symbolic`的缩写）创建了两种类型的符号链接：首先是到`/var/log/auth.log`文件，然后是到`/tmp/`目录。我们看到了两种不同的使用`ln -s`的方式：如果没有第二个参数，它将创建与我们要链接的内容相同名称的链接；否则，我们可以将我们自己的名称作为第二个参数（如`link-to-auth.log`和`link-to-tmp/`链接所示）。现在，我们可以通过与`/home/reader/auth.log`或`/home/reader/link-to-auth.log`交互来读取`/var/log/auth.log`的内容。如果我们想要导航到`/tmp/`，我们现在可以使用`/home/reader/tmp/`或`/home/reader/link-to-tmp/`与`cd`结合使用。虽然这个例子在日常工作中并不特别有用（除非输入`/var/log/auth.log`而不是`auth.log`可以为您节省大量时间），但链接可以防止重复复制文件，同时保持易于访问。

链接（以及 Linux 文件系统一般）中的一个重要概念是**inode**。每个文件（无论类型如何，包括目录）都有一个 inode，它描述了该文件的属性和*磁盘块位置*。在这个上下文中，属性包括所有权和权限，以及最后的更改、访问和修改时间戳。在链接中，*软链接*有它们自己的 inode，而*硬链接*指的是相同的 inode。

在继续本章的下一部分之前，使用`rm`清理四个链接和复制的`dpk.log`文件。如果你不确定如何做到这一点，请查看`rm`的 man 页面。一个小提示：删除符号链接就像`rm <name-of-link>`一样简单！

# 存档

现在我们对 Linux 中的常见文件操作有了一定的了解，我们将继续进行存档操作。虽然听起来可能很花哨，但存档简单地指的是**创建存档**。你们大多数人熟悉的一个例子是创建 ZIP 文件，这是一个存档。ZIP 并不是特定于 Windows 的；它是一种*存档文件格式*，在 Windows、Linux、macOS 等不同的实现中都有。

正如你所期望的，有许多存档文件格式。在 Linux 上，最常用的是**tarball**，它是通过使用`tar`命令创建的（这个术语来源于**t**ape **ar**chive）。以`.tar`结尾的 tarball 文件是未压缩的。在实践中，tarball 几乎总是使用 Gzip 进行压缩，Gzip 代表**G**NU **zip**。这可以直接使用`tar`命令（最常见）或之后使用`gzip`命令（不太常见，但也可以用于压缩除 tarball 以外的文件）。由于`tar`是一个复杂的命令，我们将更详细地探讨最常用的标志（描述取自`tar`手册页）：

| `-c`，`--create` | 创建一个新的存档。参数提供要存档的文件的名称。除非给出`--no-recursion`选项，否则将递归存档目录。 |
| --- | --- |
| `-x`，`--extract`，`--get` | 从存档中提取文件。参数是可选的。给定时，它们指定要提取的存档成员的名称。 |
| `-t`，`--list` | 列出存档的内容。参数是可选的。给定时，它们指定要列出的成员的名称。 |
| `-v`，`--verbose` | 详细列出处理的文件。 |
| `-f`，`--file=ARCHIVE` | 使用存档文件或设备 ARCHIVE。 |
| `-z`，`--gzip`，`--gunzip`，`--ungzip` | 通过 Gzip 过滤存档。 |
| `-C`，`--directory=DIR` | 在执行任何操作之前切换到 DIR。这个选项是有顺序的，也就是说，它影响后面的所有选项。 |

`tar`命令在指定这些选项的方式上非常灵活。我们可以逐个呈现它们，一起呈现，带有或不带有连字符，或者使用长选项或短选项。这意味着创建存档的以下方式都是正确的，都可以工作：

+   `tar czvf <archive name> <file1> <file2>`

+   `tar -czvf <archive name> <file1> <file2>`

+   `tar -c -z -v -f <archive name> <file1> <file2>`

+   `tar --create --gzip --verbose --file=<archive name> <file1> <file2>`

虽然这看起来很有帮助，但也可能令人困惑。我们的建议是：选择一个格式并坚持使用它。在本书中，我们将使用最简短的形式，因此这是所有没有连字符的短选项。让我们使用这种形式来创建我们的第一个存档！

```
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 renamedtestdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 renamedtestfile
drwxrwx--- 2 reader reader 4096 Aug  4 16:18 umaskdir
reader@ubuntu:~$ tar czvf my-first-archive.tar.gz \
nanofile.txt renamedtestfile
nanofile.txt
renamedtestfile
reader@ubuntu:~$ ls -l
total 16
-rw-rw-r-- 1 reader reader  267 Aug 19 10:29 my-first-archive.tar.gz
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 renamedtestdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 renamedtestfile
drwxrwx--- 2 reader reader 4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games     0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

使用这个命令，我们**v**erbosely **c**reated 了一个名为`my-first-archive.tar.gz`的 g**z**ipped **f**ile，其中包含了文件`nanofile.txt umaskfile`和`renamedtestfile`。

在这个例子中，我们只存档了文件。实际上，通常很好的存档整个目录。语法完全相同，只是你会给出一个目录名，整个目录将被存档（在`-z`选项的情况下也会被压缩）。当你解压存档了一个目录的 tarball 时，整个目录将被再次提取，而不仅仅是内容。

现在，让我们看看解压它是否能还原我们的文件！我们将 gzipped tarball 移动到`renamedtestdir`，并使用`tar xzvf`命令在那里解压它：

```
reader@ubuntu:~$ ls -l
total 16
-rw-rw-r-- 1 reader reader  226 Aug 19 10:40 my-first-archive.tar.gz
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 renamedtestdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 renamedtestfile
drwxrwx--- 2 reader reader 4096 Aug 19 10:37 umaskdir
reader@ubuntu:~$ mv my-first-archive.tar.gz renamedtestdir/
reader@ubuntu:~$ cd renamedtestdir/
reader@ubuntu:~/renamedtestdir$ ls -l
total 4
-rw-rw-r-- 1 reader reader 226 Aug 19 10:40 my-first-archive.tar.gz
reader@ubuntu:~/renamedtestdir$ tar xzvf my-first-archive.tar.gz 
nanofile.txt
renamedtestfile
reader@ubuntu:~/renamedtestdir$ ls -l
total 8
-rw-rw-r-- 1 reader reader 226 Aug 19 10:40 my-first-archive.tar.gz
-rw-rw-r-- 1 reader reader  69 Jul 14 13:18 nanofile.txt
-rwxr-xr-- 1 reader reader   0 Aug  4 13:44 renamedtestfile
reader@ubuntu:~/renamedtestdir$
```

正如我们所看到的，我们在`renamedtestdir`中找回了我们的文件！实际上，我们从未删除原始文件，所以这些是副本。在你开始提取和清理所有东西之前，你可能想知道 tarball 里面有什么。这可以通过使用`-t`选项而不是`-x`来实现：

```
reader@ubuntu:~/renamedtestdir$ tar tzvf my-first-archive.tar.gz 
-rw-rw-r-- reader/reader 69 2018-08-19 11:54 nanofile.txt
-rw-rw-r-- reader/reader  0 2018-08-19 11:54 renamedtestfile
reader@ubuntu:~/renamedtestdir$
```

`tar`广泛使用的最后一个有趣选项是`-C`或`--directory`选项。这个命令确保我们在提取之前不必移动存档。让我们使用它将`/home/reader/renamedtestdir/my-first-archive.tar.gz`从我们的`home`目录提取到`/home/reader/umaskdir/`中：

```
reader@ubuntu:~/renamedtestdir$ cd
reader@ubuntu:~$ tar xzvf renamedtestdir/my-first-archive.tar.gz -C umaskdir/
nanofile.txt
renamedtestfile
reader@ubuntu:~$ ls -l umaskdir/
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
-rwxr-xr-- 1 reader reader  0 Aug  4 13:44 renamedtestfile
-rw-rw---- 1 reader games   0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

通过在存档名称后指定`-C`和目录参数，我们确保`tar`将 gzipped tarball 的内容提取到指定的目录中。

这涵盖了`tar`命令的最重要方面。然而，还有一件小事要做：清理！我们在`home`目录下搞得一团糟，而且那里没有任何真正有用的文件。以下是一个实际示例，展示了带有`rm -r`命令的通配符有多危险：

```
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug 19 10:42 renamedtestdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 renamedtestfile
drwxrwx--- 2 reader reader 4096 Aug 19 10:47 umaskdir
reader@ubuntu:~$ rm -r *
reader@ubuntu:~$ ls -l
total 0
reader@ubuntu:~$
```

一个简单的命令，没有警告，所有文件，包括更多文件的目录，都消失了！如果你在想：不，Linux 也没有回收站。这些文件已经消失了；只有高级硬盘恢复技术*可能*还能够恢复这些文件。

确保你执行了前面的命令，以了解`rm`有多具有破坏性。然而，在你执行之前，请确保你在你的`home`目录下，并且不要意外地有任何你不想删除的文件。如果你遵循我们的示例，这不应该是问题，但如果你做了其他事情，请确保你知道自己在做什么！

# 查找文件

在学习了常见的文件操作和存档之后，还有一个在文件操作中至关重要的技能我们还没有涉及：查找文件。你知道如何复制或存档文件是非常好的，但如果你找不到你想要操作的文件，你将很难完成你的任务。幸运的是，有专门用于在 Linux 文件系统中查找和定位文件的工具。简单来说，这些工具就是`find`和`locate`。`find`命令更复杂，但更强大，而`locate`命令在你确切知道你要找的东西时更容易使用。首先，我们将向你展示如何使用`locate`，然后再介绍`find`更广泛的功能。

# 定位

在 locate 的 man 页面上，描述再合适不过了：“locate - 按名称查找文件”。`locate`命令默认安装在您的 Ubuntu 机器上，基本功能就是使用`locate <filename>`这么简单。让我们看看它是如何工作的：

```
reader@ubuntu:~$ locate fstab
/etc/fstab
/lib/systemd/system-generators/systemd-fstab-generator
/sbin/fstab-decode
/usr/share/doc/mount/examples/fstab
/usr/share/doc/mount/examples/mount.fstab
/usr/share/doc/util-linux/examples/fstab
/usr/share/doc/util-linux/examples/fstab.example2
/usr/share/man/man5/fstab.5.gz
/usr/share/man/man8/fstab-decode.8.gz
/usr/share/man/man8/systemd-fstab-generator.8.gz
/usr/share/vim/vim80/syntax/fstab.vim
reader@ubuntu:~$
```

在前面的例子中，我们搜索了文件名`fstab`。我们可能记得我们需要编辑这个文件来进行文件系统更改，但我们不确定在哪里可以找到它。`locate`向我们展示了磁盘上包含`fstab`的所有位置。如你所见，它不必是一个精确的匹配；包含`fstab`字符串的所有内容都将被打印出来。

你可能已经注意到`locate`命令几乎立即完成。这是因为它使用一个定期更新的数据库来存储所有文件，而不是在运行时遍历整个文件系统。因此，信息并不总是准确的，因为更改不会实时同步到数据库中。为了确保你使用的是文件系统的最新状态与数据库进行交互，请确保在运行`locate`之前运行`sudo updatedb`（需要 root 权限）。这也是在系统上首次运行`locate`之前所需的，否则就没有数据库可供查询！

Locate 有一些选项，但根据我们的经验，只有当你知道确切的文件名（或文件名的一部分）时才会使用它。对于其他搜索，最好默认使用`find`命令。

# find

find 是一个非常强大但复杂的命令。你可以使用`find`做以下任何一件事情：

+   按文件名搜索

+   按权限搜索（用户和组）

+   按所有权搜索

+   按文件类型搜索

+   按文件大小搜索

+   按时间戳搜索（创建时间，最后修改时间，最后访问时间）

+   仅在特定目录中搜索

解释`find`命令的所有功能需要一整章的篇幅。我们只会描述最常见的用法。真正的教训在于了解`find`的高级功能；如果你需要查找具有特定属性的文件，一定要首先考虑使用`find`命令，并查看`man file`页面，看看是否可以利用`find`进行搜索（剧透：**几乎总是**这样！）。

让我们从 find 的基本用法开始：`find <位置> <选项和参数>`。如果没有任何选项和参数，find 将打印出位置内找到的每个文件：

```
reader@ubuntu:~$ find /home/reader/
/home/reader/
/home/reader/.gnupg
/home/reader/.gnupg/private-keys-v1.d
/home/reader/.bash_logout
/home/reader/.sudo_as_admin_successful
/home/reader/.profile
/home/reader/.bashrc
/home/reader/.viminfo
/home/reader/.lesshst
/home/reader/.local
/home/reader/.local/share
/home/reader/.local/share/nano
/home/reader/.cache
/home/reader/.cache/motd.legal-displayed
/home/reader/.bash_history
reader@ubuntu:~$
```

你可能以为你的`home`目录是空的。实际上，它包含了相当多的隐藏文件或目录（以点开头），这些文件被`find`找到了。现在，让我们使用`-name`选项应用一个过滤器：

```
reader@ubuntu:~$ find /home/reader/ -name bash
reader@ubuntu:~$ find /home/reader/ -name *bash*
/home/reader/.bash_logout
/home/reader/.bashrc
/home/reader/.bash_history
reader@ubuntu:~$ find /home/reader/ -name .bashrc
/home/reader/.bashrc
reader@ubuntu:~$
```

与你可能期望的相反，`find`与`locate`在部分匹配文件方面的工作方式不同。除非在`-name`参数的参数周围添加通配符，否则它只会匹配完整的文件名，而不是部分匹配的文件。这绝对是需要记住的事情。那么，仅查找文件而不是目录呢？为此，我们可以使用`-type`选项和`d`参数表示目录，或者使用`f`表示文件：

```
reader@ubuntu:~$ find /home/reader/ -type d
/home/reader/
/home/reader/.gnupg
/home/reader/.gnupg/private-keys-v1.d
/home/reader/.local
/home/reader/.local/share
/home/reader/.local/share/nano
/home/reader/.cache
reader@ubuntu:~$ find /home/reader/ -type f
/home/reader/.bash_logout
/home/reader/.sudo_as_admin_successful
/home/reader/.profile
/home/reader/.bashrc
/home/reader/.viminfo
/home/reader/.lesshst
/home/reader/.cache/motd.legal-displayed
/home/reader/.bash_history
reader@ubuntu:~$
```

第一个结果显示了`/home/reader/`内的所有目录（包括`/home/reader/!`），而第二个结果打印了所有文件。你可以看到，没有重叠，因为在 Linux 下，文件*总是只有一种类型*。我们还可以组合多个选项，比如`-name`和`-type`：

```
reader@ubuntu:~$ find /home/reader/ -name *cache* -type f
reader@ubuntu:~$ find /home/reader/ -name *cache* -type d
/home/reader/.cache
reader@ubuntu:~$
```

我们首先在`/home/reader/`中寻找包含字符串 cache 的*文件*。`find`命令没有打印任何内容，这意味着我们没有找到任何东西。然而，如果我们寻找包含 cache 字符串的*目录*，我们会看到`/home/reader/.cache/`目录。

最后一个例子，让我们看看如何使用`find`来区分不同大小的文件。为此，我们将使用`touch`创建一个空文件，使用`vim`（或`nano`）创建一个非空文件：

```
reader@ubuntu:~$ ls -l
total 0
reader@ubuntu:~$ touch emptyfile
reader@ubuntu:~$ vim textfile.txt
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader  0 Aug 19 11:54 emptyfile
-rw-rw-r-- 1 reader reader 23 Aug 19 11:54 textfile.txt
reader@ubuntu:~
```

从屏幕上的`0`和`23`可以看出，`emptyfile`包含 0 字节，而`textfile.txt`包含 23 字节（这不是巧合，它包含了 23 个字符的句子）。让我们看看如何使用`find`命令找到这两个文件：

```
reader@ubuntu:~$ find /home/reader/ -size 0c
/home/reader/.sudo_as_admin_successful
/home/reader/.cache/motd.legal-displayed
/home/reader/emptyfile
reader@ubuntu:~$ find /home/reader/ -size 23c
/home/reader/textfile.txt
reader@ubuntu:~$
```

为此，我们使用了`-size`选项。我们给出了我们要查找的数字，后面跟着一个表示我们正在处理的范围的字母。`c`用于字节，`k`用于千字节，`M`用于兆字节，依此类推。您可以在手册页上找到这些值。正如结果所示，有三个文件的大小正好为 0 字节：我们的`emptyfile`就是其中之一。有一个文件的大小正好为 23 字节：我们的`textfile.txt`。您可能会想：23 字节，那非常具体！我们怎么知道文件的确切字节数呢？好吧，您不会知道。`find`的创建者还实现了*大于*和*小于*的结构，我们可以使用它们来提供更多的灵活性：

```
reader@ubuntu:~$ find /home/reader/ -size +10c
/home/reader/
/home/reader/.gnupg
/home/reader/.gnupg/private-keys-v1.d
/home/reader/.bash_logout
/home/reader/.profile
/home/reader/.bashrc
/home/reader/.viminfo
/home/reader/.lesshst
/home/reader/textfile.txt
/home/reader/.local
/home/reader/.local/share
/home/reader/.local/share/nano
/home/reader/.cache
/home/reader/.bash_history
reader@ubuntu:~$ find /home/reader/ -size +10c -size -30c
/home/reader/textfile.txt
reader@ubuntu:~$
```

假设我们正在寻找一个至少大于 10 字节的文件。我们在参数上使用`+`选项，它只打印大于 10 字节的文件。然而，我们仍然看到了太多的文件。现在，我们希望文件也小于 30 字节。我们添加另一个`-size`选项，这次指定`-30c`，意味着文件将小于 30 字节。并且，毫不意外，我们找到了我们的 23 字节的`testfile.txt`！

所有前述选项以及更多选项都可以组合在一起，形成一个非常强大的搜索查询。您是否正在寻找一个*文件*，它*至少*有 100 KB，但*不超过*10 MB，在`/var/`中的*任何位置*，在*上周*创建，并且对您是*可读*的？只需在`find`中组合选项，您肯定会在短时间内找到该文件！

# 总结

本章描述了 Linux 中的文件操作。我们从常见的文件操作开始。我们解释了如何在 Linux 中使用`cp`复制文件，以及如何使用`mv`移动或重命名文件。接下来，我们讨论了如何使用`rm`删除文件和目录，以及如何使用`ln -s`命令在 Linux 下创建符号链接。

在本章的第二部分中，我们讨论了归档。虽然有许多不同的工具可以进行归档，但我们专注于 Linux 中最常用的工具：`tar`。我们向您展示了如何在当前工作目录和文件系统的其他位置创建和提取归档。我们描述了`tar`可以归档文件和整个目录，并且我们可以使用`-t`选项在不实际提取它的情况下查看 tarball 中的内容。

我们以使用`file`和`locate`查找文件结束了本章。我们解释了`locate`是一个在特定情况下有用的简单命令，而`find`是一个更复杂但非常强大的命令，可以为掌握它的人带来巨大的好处。

本章介绍了以下命令：`cp`、`rm`、`mv`、`ln`、`tar`、`locate`和`find`。

# 问题

1.  我们在 Linux 中使用哪个命令来复制文件？

1.  移动和重命名文件之间有什么区别？

1.  用于在 Linux 下删除文件的`rm`命令为什么可能很危险？

1.  硬链接和符号（软）链接之间有什么区别？

1.  `tar`的三种最重要的操作模式是什么？

1.  `tar`用于选择输出目录的选项是什么？

1.  在搜索文件名时，`locate`和`find`之间最大的区别是什么？

1.  `find`有多少个选项可以组合使用？

# 进一步阅读

以下资源可能会很有趣，如果您想深入了解本章的主题：

+   **文件操作**：[`ryanstutorials.net/linuxtutorial/filemanipulation.php`](https://ryanstutorials.net/linuxtutorial/filemanipulation.php)

+   **Tar 教程**：[`www.poftut.com/linux-tar-command-tutorial-with-examples/`](https://www.poftut.com/linux-tar-command-tutorial-with-examples/)

+   **查找实际示例**：[`www.tecmint.com/35-practical-examples-of-linux-find-command/`](https://www.tecmint.com/35-practical-examples-of-linux-find-command/)
