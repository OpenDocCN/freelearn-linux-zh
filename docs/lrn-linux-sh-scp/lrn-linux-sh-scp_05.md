# 第五章：理解 Linux 权限方案

在本章中，我们将探讨 Linux 权限方案是如何实现的。文件和目录的读取、写入和执行权限将被讨论，我们将看到它们如何不同地影响文件和目录。我们将看到多个用户如何使用组一起工作，以及一些文件和目录也对其他人可用。

本章将介绍以下命令：`id`，`touch`，`chmod`，`umask`，`chown`，`chgrp`，`sudo`，`useradd`，`groupadd`，`usermod`，`mkdir`和`su`。

本章将涵盖以下主题：

+   读取，写入和执行

+   用户，组和其他人

+   与多个用户一起工作

+   高级权限

# 技术要求

我们将使用我们在第二章中创建的虚拟机来探索 Linux 权限方案，*设置您的本地环境*。在本章中，我们将向该系统添加新用户，但目前只有作为第一个用户（具有管理或*root*权限）的访问权限就足够了。

# 读取，写入和执行

在上一章中，我们讨论了 Linux 文件系统以及 Linux 实现“一切皆文件”哲学的不同类型。然而，我们没有看文件的权限。你可能已经猜到，在一个多用户系统，比如 Linux 服务器中，用户可以访问其他用户拥有的文件并不是一个特别好的主意。隐私在哪里呢？

Linux 权限方案实际上是 Linux 体验的核心，就我们而言。就像在 Linux 中（几乎）一切都被处理为文件一样，所有这些文件都有一组不同的权限。在上一章中探索文件系统时，我们限制了自己只能查看所有人或当前登录用户可见的文件。然而，有许多文件只能被`root`用户查看或写入：通常，这些是一些敏感文件，比如`/etc/shadow`（其中包含所有用户的*哈希*密码），或者在启动系统时使用的文件，比如`/etc/fstab`（确定哪些文件系统在启动时被挂载）。如果每个人都可以编辑这些文件，很快就会导致系统无法启动！

# RWX

Linux 下的文件权限由三个属性处理：**读取**，**写入**和**执行**，或者 RWX。虽然还有其他权限（我们将在本章后面讨论一些），但大多数权限交互将由这三个属性处理。尽管这些名称似乎不言自明，但它们在（普通）文件和目录方面的行为是不同的。以下表格应该说明这一点：

允许用户使用任何支持此操作的命令查看文件的内容，比如`vim`，`nano`，`less`，`cat`等。

| **权限** | **对普通文件** | **对目录** |
| --- | --- | --- |
| 读取 | 允许用户使用`ls`命令列出目录的内容。这甚至会列出用户没有其他权限的目录中的文件！ |
| 写入 | 允许用户对文件进行更改。 | 允许用户替换或删除目录中的文件，即使用户对该文件没有直接权限。但这不包括对目录中所有文件的读取权限！ |
| 执行 | 允许用户执行文件。当文件是应该被执行的东西，比如二进制文件或脚本时，这是相关的；否则，这个属性什么也不做。 | 允许用户使用 `cd` 进入目录。这是与内容列表不同的权限，但它们几乎总是一起使用；能够列出而不能进入（反之亦然）大多数情况下是无效的配置。 |

这个概述应该为这三种不同的权限提供了一个基础。请仔细看一看，看看你是否完全理解了那里呈现的内容。

现在，情况将变得更加复杂。虽然文件和目录上的这些权限显示了用户可以做什么和不能做什么，但 Linux 如何处理多个用户？Linux 如何跟踪文件的*所有权*，文件如何被多个用户共享？

# 用户、组和其他

在 Linux 下，每个文件都*由一个用户和一个组拥有*。每个用户都有一个标识号，**用户 ID**（**UID**）。组也是一样：它由一个**组 ID**（**GID**）解析。每个用户有一个 UID 和一个*主要*GID；然而，用户可以是多个组的成员。在这种情况下，用户将有一个或多个附加 GID。您可以通过在 Ubuntu 机器上运行`id`命令来自己看到这一点：

```
reader@ubuntu:~$ id
uid=1000(reader) gid=1004(reader) groups=1004(reader),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd),1000(lpadmin),1001(sambashare),1002(debian-tor),1003(libvirtd)
reader@ubuntu:~$
```

在前面的输出中，我们可以看到以下内容：

+   `reader`用户的`uid`是`1000`；Linux 通常从`1000`开始对普通用户进行编号

+   `gid`是`1004`，对应于`reader`组；默认情况下，Linux 会创建一个与用户同名的组（除非明确告知不要创建）

+   其他组包括 `adm`、`sudo` 和其他

这意味着什么？当前登录的用户具有`uid`为`1000`，主要`gid`为`1004`，以及一些附加组，这确保它具有其他特权。例如，在 Ubuntu 下，`cdrom`组允许用户访问光驱。`sudo`组允许用户执行管理命令，`adm`组允许用户读取管理文件。

虽然我们通常按名称引用用户和组，但这只是 Linux 为我们提供的 UID 和 GID 的表示。在系统级别上，只有 UID 和 GID 对权限很重要。这使得例如，有两个具有相同用户名但不同 UID 的用户成为可能：这些用户的权限将不同。反之亦然也是可能的：两个不同用户名但相同 UID 的用户，这将导致两个用户的权限在 UID 级别上至少是相同的。然而，这两种情况都非常令人困惑，不应该使用！正如我们将在后面看到的那样，使用组来共享权限是共享文件和目录的最佳解决方案。

另一件需要记住的事情是，UID 和 GID 是*本地的*。所以，如果我在机器 A 上有一个名为 bob 的用户，UID 为 1000，在机器 B 上 UID 为 1000 映射到用户 alice，将 bob 的文件从机器 A 传输到机器 B 将导致文件在系统 B 上被 alice 拥有！

之前解释的 RWX 权限与我们现在讨论的用户和组有关。实质上，每个文件（或目录，这只是一种不同类型的文件）都有以下属性：

+   文件由一个*用户*拥有，该用户具有（部分）RWX 权限

+   文件也由一个*组*拥有，再次，有（部分）RWX 权限

+   文件最终对*其他人*具有 RWX 权限，这意味着所有不共享该组的不同用户

要确定用户是否可以读取、写入或执行文件或目录，我们需要查看以下属性（不一定按照这个顺序）：

+   用户是否是文件的所有者？所有者有什么 RWX 权限？

+   用户是否是拥有文件的组的一部分？为组设置了什么 RWX 权限？

+   文件在*其他*属性上有足够的权限吗？

在变得太抽象之前，让我们看一些简单的例子。在您的虚拟机上，按照以下命令操作：

```
reader@ubuntu:~$ pwd
/home/reader
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
reader@ubuntu:~$ touch testfile
reader@ubuntu:~$
```

首先，我们确保我们在`reader`用户的`home`目录中。如果不是，我们可以使用`cd /home/reader`命令或者只输入`cd`（没有参数时，`cd`默认为用户的`home`目录！）。我们继续使用`ls -l`以长格式列出目录的内容，其中显示了一个文件：`nanofile.txt`，来自第二章，*设置您的本地环境*（如果您没有跟随那里并且没有该文件，不用担心；我们稍后将创建和操作文件）。我们使用一个新命令`touch`来创建一个空文件。我们指定给`touch`的参数被解释为文件名，当我们再次列出文件时可以看到。

```
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
-rw-rw-r-- 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$
```

您将看到权限后面跟着两个名称：用户名和组名（按顺序！）。对于我们的`testfile`，用户`reader`和`reader`组的成员都可以读取和写入文件，但不能执行（在`x`的位置上，有一个`-`，表示没有该权限）。所有其他用户，例如既不是*读者*也不是*读者*组的成员（在这种情况下，实际上是所有其他用户），由于其他人的权限，只能读取文件。这也在下表中描述：

| **文件类型** **（第一个字符）** | **用户权限（第 2 到 4 个字符）** | **组权限（第 5 到 7 个字符）** | **其他权限（第 8 到 10 个字符）** | **用户所有权** | **组所有权** |
| --- | --- | --- | --- | --- | --- |
| -（普通文件） | `rw-`，读和写，无执行 | `rw-`，读和写，无执行 | `r--`，只读 | 读者 | 读者 |

如果一个**文件**对每个人都有完全权限，它会是这样的：`-rwxrwxrwx`。对于所有者和组都有所有权限，但其他人没有任何权限的文件，它将是`-rwxrwx---`。对于用户和组都有所有权限，但其他人没有任何权限的**目录**，表示为`drwxrwx---`。

让我们看另一个例子：

```
reader@ubuntu:~$ ls -l /
<SNIPPED>
dr-xr-xr-x 98 root root          0 Aug  4 10:49 proc
drwx------  3 root root       4096 Jul  1 09:40 root 
```

```

drwxr-xr-x 25 root root        900 Aug  4 10:51 run
<SNIPPED>
reader@ubuntu:~$
```

系统超级用户的`home`目录是`/root/`。我们可以从行的第一个字符看到它是一个`d`，表示*目录*。它对所有者`root`具有 RWX（最后一次：读取、写入、执行）权限，对组（也是`root`）没有权限，对其他人（由`---`表示）也没有权限。这些权限只能意味着一件事：**只有用户** **root** **可以进入或操作此目录！**让我们看看我们的假设是否正确。请记住，*进入*目录需要`x`权限，而*列出*目录内容需要`r`权限。我们既不是`root`用户也不在 root 组，因此我们既不能做任何一件事。在这种情况下，将应用其他人的权限，即`---`：

```
reader@ubuntu:~$ cd /root/
-bash: cd: /root/: Permission denied
reader@ubuntu:~$ ls /root/
ls: cannot open directory '/root/': Permission denied
reader@ubuntu:~$
```

# 操作文件权限和所有权

阅读本章的第一部分后，您应该对 Linux 文件权限有了相当好的理解，以及如何在用户、组和其他级别上使用读取、写入和执行来确保文件的暴露正好符合要求。然而，直到这一点，我们一直在处理静态权限。在管理 Linux 系统时，您很可能会花费大量时间调整和解决权限问题。在本书的这一部分中，我们将探讨可以用来操作文件权限的命令。

# chmod，umask

让我们回到我们的`testfile`。它具有以下权限：`-rw-rw----`。用户和组可读/写，其他人可读。虽然这些权限对大多数文件来说可能是合适的，但对于所有文件来说肯定不是一个很好的选择。私人文件呢？您可能不希望这些文件被所有人读取，甚至可能不希望被组成员读取。

在 Linux 中，更改文件或目录的权限的命令是`chmod`，我们喜欢将其读作**ch**ange file **mod**e。`chmod`有两种操作模式：符号模式和数字/八进制模式。我们将首先解释符号模式（更容易理解），然后再转到八进制模式（更快速使用）。

我们还没有介绍的是查看命令手册的命令。该命令就是`man`，后面跟着你想要查看手册的命令。在这种情况下，`man chmod`会将我们放入`chmod`手册分页器中，它使用与你学习 Vim 相同的导航控件。记住，退出是通过输入`:q`来完成的。在这种情况下，只需输入`q`就足够了。现在看一下`chmod`手册，至少读一下**description**标题；这将使接下来的解释更清晰。

符号模式使用了我们之前看到的 UGOA 字母的 RWX 构造。这可能看起来很新，但实际上并不是！**U**sers、**G**roups、**O**thers 和**A**ll 用于表示我们正在更改的权限。

要添加权限，我们告诉`chmod`我们（用户、组、其他或所有）为谁做这个操作，然后是我们想要添加的权限。例如，`chmod u+x <filename>`将为用户添加执行权限。类似地，使用`chmod`删除权限的方法如下：`chmod g-rwx <filename>`。请注意，我们使用`+`号来添加权限，使用`-`号来删除权限。如果我们没有指定用户、组、其他或所有，**所有**将默认使用。让我们在我们的 Ubuntu 机器上试一下：

```
reader@ubuntu:~$ cd
reader@ubuntu:~$ pwd
/home/reader
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
-rw-rw-r-- 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$ chmod u+x testfile 
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
-rwxrw-r-- 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$ chmod g-rwx testfile 
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
-rwx---r-- 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$ chmod -r testfile 
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
--wx------ 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$
```

首先，我们为`testfile`添加了用户的执行权限。接下来，我们从组中移除了读取、写入和执行权限，结果是`-rwx---r--`。在这种情况下，组成员仍然可以读取文件，然而，*因为每个人仍然可以读取文件*。可以说这不是隐私的完美权限。最后，我们在`-r`之前没有指定任何内容，这实际上移除了用户、组和其他的读取权限，导致文件最终变成了`--wx------`。

能够写和执行一个你无法阅读的文件有点奇怪。让我们来修复它，看看八进制权限是如何工作的！我们可以使用`chmod`的**verbose**选项，通过使用`-v`标志来打印更多信息：

```
reader@ubuntu:~$ chmod -v u+rwx testfile
mode of 'testfile' changed from 0300 (-wx------) to 0700 (rwx------)
reader@ubuntu:~$
```

正如你所看到的，我们现在从`chmod`得到了输出！具体来说，我们可以看到八进制模式。在我们改变文件之前，模式是`0300`，在为用户添加读取权限后，它跳到了`0700`。这些数字代表什么？

这一切都与权限的二进制实现有关。对于所有三个级别（用户、组、其他），在结合读取、写入和执行时，有 8 种不同的可能权限，如下所示：

| **符号** | **八进制** |
| --- | --- |
| `---` | 0 |
| `--x` | 1 |
| `-w-` | 2 |
| `-wx` | 3 |
| `r--` | 4 |
| `r-x` | 5 |
| `rw-` | 6 |
| `rwx` | 7 |

基本上，八进制值在 0 和 7 之间，总共有 8 个值。这就是为什么它被称为八进制：来自于拉丁语/希腊语中 8 的表示，**octo**。读取权限被赋予值 4，写入权限被赋予值 2，执行权限被赋予值 1。

通过使用这个系统，0 到 7 的值总是可以唯一地与 RWX 值相关联。RWX 是*4+2+1 = 7*，RX 是*4+1 = 5*，依此类推。

现在我们知道了八进制表示是如何工作的，我们可以使用它们来使用`chmod`修改文件权限。让我们在一个命令中为用户、组和其他给予测试文件完全权限（RWX 或 7）： 

```
reader@ubuntu:~$ chmod -v 0777 testfile 
mode of 'testfile' changed from 0700 (rwx------) to 0777 (rwxrwxrwx)
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
-rwxrwxrwx 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$
```

在这种情况下，`chmod`接受四个数字作为参数。第一个数字涉及到一种特殊类型的权限，称为粘滞位；我们不会讨论这个，但我们已经在*Further reading*部分中包含了相关材料，供感兴趣的人参考。在这些示例中，它总是设置为`0`，因此没有设置特殊位。第二个数字映射到用户权限，第三个映射到组权限，第四个，不出所料，映射到其他权限。

如果我们想要使用符号表示法来做到这一点，我们可以使用`chmod a+rwx`命令。那么，为什么八进制比我们之前说的更快呢？让我们看看如果我们想要为每个级别设置不同的权限会发生什么，例如`-rwxr-xr--`。如果我们想要用符号表示法来做到这一点，我们需要使用三个命令或一个链接的命令（`chmod`的另一个功能）：

```
reader@ubuntu:~$ chmod 0000 testfile 
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
---------- 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$ chmod u+rwx,g+rx,o+r testfile 
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
-rwxr-xr-- 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$
```

从`chmod u+rwx,g+rx,o+r testfile`命令中可以看出，事情变得有点复杂。然而，使用八进制表示法，命令要简单得多：

```
reader@ubuntu:~$ chmod 0000 testfile 
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
---------- 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$ chmod 0754 testfile 
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
-rwxr-xr-- 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$
```

基本上，主要区别在于使用*命令式*表示法（添加或删除权限）与*声明式*表示法（将其设置为这些值）。根据我们的经验，声明式几乎总是更好/更安全的选择。使用命令式，我们需要首先检查当前的权限并对其进行变异；而使用声明式，我们可以在单个命令中指定我们想要的内容。

现在可能很明显了，但我们更喜欢使用八进制表示法。除了从更短、更简单的命令中受益，这些命令是以声明方式处理的，另一个好处是大多数您在网上找到的示例也使用八进制表示法。要完全理解这些示例，您至少需要了解八进制。而且，如果无论如何都需要理解它们，那么在日常生活中使用它们是最好的！

早些时候，当我们使用`touch`命令时，我们最终得到了一个文件，该文件可以被用户和组读取和写入，并且对其他人是可读的。这些似乎是默认权限，但它们是从哪里来的？我们如何操纵它们？让我们来认识一下`umask`：

```
reader@ubuntu:~$ umask
0002
reader@ubuntu:~$
```

`umask`会话用于确定新创建的文件和目录的文件权限。对于文件，执行以下操作：取文件的最大八进制值`0666`，然后减去`umask`（在本例中为`0002`），这给我们`0664`。这意味着新创建的文件是`-rw-rwr--`，这正是我们在`testfile`中看到的。你可能会问，为什么我们使用`0666`而不是`0777`？这是 Linux 提供的一种保护措施；如果我们使用`0777`，大多数文件将被创建为可执行文件。可执行文件可能是危险的，因此设计决策是文件只有在明确设置为可执行时才能执行。因此，根据当前的实现，没有*意外*创建可执行文件这样的事情。对于目录，使用正常的八进制值`0777`，这意味着目录以`0775`，`-rwxrwxr-x`权限创建。我们可以通过使用`mkdir`命令创建一个新目录来验证这一点：

```
reader@ubuntu:~$ ls -l
total 4
-rw-rw-r-- 1 reader reader 69 Jul 14 13:18 nanofile.txt
-rwxr-xr-- 1 reader reader  0 Aug  4 13:44 testfile
reader@ubuntu:~$ umask
0002
reader@ubuntu:~$ mkdir testdir
reader@ubuntu:~$ ls -l
total 8
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 testfile
reader@ubuntu:~$
```

因为目录的执行权限要少得多（记住，它用于确定您是否可以进入目录），所以这个实现与文件不同。

关于`umask`，我们还有一个技巧要展示。在特定情况下，我们想要自己确定文件和目录的默认值。我们也可以使用`umask`命令来做到这一点：

```
reader@ubuntu:~$ umask
0002
reader@ubuntu:~$ umask 0007
reader@ubuntu:~$ umask
0007
reader@ubuntu:~$ touch umaskfile
reader@ubuntu:~$ mkdir umaskdir
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 testfile
drwxrwx--- 2 reader reader 4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader reader    0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

在上面的例子中，您可以看到运行`umask`命令而不带参数会打印当前的 umask。如果以有效的 umask 值作为参数运行它，将会改变 umask 为该值，然后在创建新文件和目录时使用。将上述输出中的`umaskfile`和`umaskdir`与之前的`testfile`和`testdir`进行比较。如果我们想要默认创建私有文件，这将非常有用！

# sudo、chown 和 chgrp

到目前为止，我们已经看到了如何操纵文件和目录的（基本）权限。然而，我们还没有处理更改文件的所有者或组。总是必须按照创建时的用户和组来工作有点不切实际。对于 Linux，我们可以使用两个工具来更改所有者和组：**ch**ange **own**er（`chown`）和**ch**ange **gr**ou**p**（`chgrp`）。然而，有一件非常重要的事情要注意：这些命令只能由具有 root 权限的人执行（通常是`root`用户）。因此，在我们向你介绍`chown`和`chgrp`之前，让我们先看看`sudo`！

# sudo

`sudo`命令最初是为**su**peruser **do**命名的，正如其名字所暗示的，它给了你一个机会以 root 超级用户的身份执行操作。`sudo`命令使用`/etc/sudoers`文件来确定用户是否被允许提升到超级用户权限。让我们看看它是如何工作的！

```
reader@ubuntu:~$ cat /etc/sudoers
cat: /etc/sudoers: Permission denied
reader@ubuntu:~$ ls -l /etc/sudoers
-r--r----- 1 root root 755 Jan 18  2018 /etc/sudoers
reader@ubuntu:~$ sudo cat /etc/sudoers
[sudo] password for reader: 
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults    env_reset
Defaults    mail_badpass
Defaults  secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
<SNIPPED>
# User privilege specification
root    ALL=(ALL:ALL) ALL

# Members of the admin group may gain root privileges
%admin ALL=(ALL) ALL

# Allow members of group sudo to execute any command
%sudo    ALL=(ALL:ALL) ALL
<SNIPPED>
reader@ubuntu:~$
```

我们首先尝试以普通用户的身份查看`/etc/sudoers`的内容。当这给我们一个`Permission denied`错误时，我们查看文件的权限。从`-r--r----- 1 root root`这一行，很明显只有`root`用户或`root`组的成员才能读取该文件。为了提升到 root 权限，我们使用`sudo`命令*在*我们想要运行的命令前面，即`cat /etc/sudoers`。为了验证，Linux 会始终要求用户输入密码。默认情况下，这个密码会被保存在内存中大约 5 分钟，所以如果你最近输入过密码，你就不必每次都输入密码。

输入密码后，`/etc/sudoers`文件就会被打印出来！看来`sudo`确实给了我们超级用户权限。这是如何工作的也可以通过`/etc/sudoers`文件来解释。`# Allow members of group sudo to execute any command`这一行是一个注释（因为它以`#`开头；稍后会详细介绍），告诉我们下面的行给了`sudo`组的所有用户任何命令的权限。在 Ubuntu 上，默认创建的用户被认为是管理员，并且是这个组的成员。使用`id`命令来验证这一点：

```
reader@ubuntu:~$ id
uid=1000(reader) gid=1004(reader) groups=1004(reader),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd),1000(lpadmin),1001(sambashare),1002(debian-tor),1003(libvirtd)
reader@ubuntu:~$
```

`sudo`命令还有另一个很好的用途：切换到`root`用户！为此，使用`--login`标志，或者它的简写，`-i`：

```
reader@ubuntu:~$ sudo -i
[sudo] password for reader: 
root@ubuntu:~#
```

在提示符中，你会看到用户名已经从`reader`变成了`root`。此外，你的提示符中的最后一个字符现在是`#`而不是`$`。这也用于表示当前的提升权限。你可以使用内置的`exit` shell 退出这个提升的位置：

```
root@ubuntu:~# exit
logout
reader@ubuntu:~$
```

记住，`root`用户是系统的超级用户，可以做任何事情。而且，我们真的是指任何事情！与其他操作系统不同，如果你告诉 Linux 删除根文件系统和其下的一切，它会乐意遵从（直到它破坏了太多以至于无法正常工作为止）。也不要指望有`Are you sure?`的提示。对于`sudo`命令或者 root 提示中的任何东西都要非常小心。

# chown，chgrp

经过一小段`sudo`的绕道之后，我们可以回到文件权限：我们如何改变文件的所有权？让我们从使用`chgrp`来改变组开始。语法如下：`chgrp <groupname> <filename>`：

```
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 testfile
drwxrwx--- 2 reader reader 4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader reader    0 Aug  4 16:18 umaskfile
reader@ubuntu:~$ chgrp games umaskfile 
chgrp: changing group of 'umaskfile': Operation not permitted
reader@ubuntu:~$ sudo chgrp games umaskfile 
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 testfile
drwxrwx--- 2 reader reader 4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader games     0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

首先，我们使用`ls`列出内容。接下来，我们尝试使用`chgrp`将`umaskfile`文件的组更改为 games。然而，由于这是一个特权操作，我们没有以`sudo`开头启动命令，所以它失败了，显示`Operation not permitted`错误消息。接下来，我们使用正确的`sudo chgrp games umaskfile`命令，这通常是 Linux 中的一个好迹象。我们再次列出文件，确保情况是这样，我们可以看到`umaskfile`的组已经更改为`games`！

让我们做同样的事情，但现在是为了用户，使用`chown`命令。语法与`chgrp`相同：`chown <username> <filename>`：

```
reader@ubuntu:~$ sudo chown pollinate umaskfile 
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader    reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader    reader 4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader    reader    0 Aug  4 13:44 testfile
drwxrwx--- 2 reader    reader 4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 pollinate games     0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

正如我们所看到的，我们现在已经将文件所有权从`reader:reader`更改为`pollinate:games`。然而，有一个小技巧非常方便，我们想立刻向您展示！您实际上可以使用`chown`通过以下语法更改用户和组：`chown <username>:<groupname> <filename>`。让我们看看这是否可以将`umaskfile`恢复到其原始所有权：

```
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader    reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader    reader 4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader    reader    0 Aug  4 13:44 testfile
drwxrwx--- 2 reader    reader 4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 pollinate games     0 Aug  4 16:18 umaskfile
reader@ubuntu:~$ sudo chown reader:reader umaskfile 
reader@ubuntu:~$ ls -l
total 12
-rw-rw-r-- 1 reader reader   69 Jul 14 13:18 nanofile.txt
drwxrwxr-x 2 reader reader 4096 Aug  4 16:16 testdir
-rwxr-xr-- 1 reader reader    0 Aug  4 13:44 testfile
drwxrwx--- 2 reader reader 4096 Aug  4 16:18 umaskdir
-rw-rw---- 1 reader reader    0 Aug  4 16:18 umaskfile
reader@ubuntu:~$
```

在前面的示例中，我们使用了随机用户和组。如果要查看系统上存在哪些组，请检查`/etc/group`文件。对于用户，相同的信息可以在`/etc/passwd`中找到。

# 与多个用户一起工作

正如我们之前所说，Linux 本质上是一个多用户系统，特别是在 Linux 服务器的情况下，这些系统通常不是由单个用户，而是经常是（大型）团队管理。服务器上的每个用户都有自己的权限集。例如，想象一下，一个服务器需要开发、运营和安全三个部门。开发和运营都有自己的东西，但也需要共享其他一些东西。安全部门需要能够查看所有内容，以确保符合安全准则和规定。我们如何安排这样的结构？让我们实现它！

首先，我们需要创建一些用户。对于每个部门，我们将创建一个单一用户，但由于我们将确保组级别的权限，因此对于每个部门中的 5、10 或 100 个用户来说，这也同样有效。我们可以使用`useradd`命令创建用户。在其基本形式中，我们可以只使用`useradd <username>`，Linux 将通过默认值处理其余部分。显然，与 Linux 中的几乎所有内容一样，这是高度可定制的；有关更多信息，请查看 man 页面（`man useradd`）。

与`chown`和`chgrp`一样，`useradd`（以及后来的`usermod`）是一个特权命令，我们将使用`sudo`来执行：

```
reader@ubuntu:~$ useradd dev-user1
useradd: Permission denied.
useradd: cannot lock /etc/passwd; try again later.
reader@ubuntu:~$ sudo useradd dev-user1
[sudo] password for reader: 
reader@ubuntu:~$ sudo useradd ops-user1
reader@ubuntu:~$ sudo useradd sec-user1
reader@ubuntu:~$ id dev-user1
uid=1001(dev-user1) gid=1005(dev-user1) groups=1005(dev-user1)
reader@ubuntu:~$ id ops-user1 
uid=1002(ops-user1) gid=1006(ops-user1) groups=1006(ops-user1)
reader@ubuntu:~$ id sec-user1 
uid=1003(sec-user1) gid=1007(sec-user1) groups=1007(sec-user1)
reader@ubuntu:~$
```

最后提醒一下，我们向您展示了当您忘记`sudo`时会发生什么。虽然错误消息在技术上是完全正确的（您需要 root 权限才能编辑`/etc/passwd`，其中存储了用户信息），但可能不太明显命令失败的原因，特别是因为误导性的`稍后重试！`错误。

然而，使用`sudo`，我们可以添加三个用户：`dev-user1`、`ops-user1`和`sec-user1`。当我们按顺序检查这些用户时，我们可以看到他们的`uid`每次增加一个。我们还可以看到与用户同名的组被创建，并且这是用户的唯一组成员。组也有它们的`gid`，每次下一个用户都会增加一个。

因此，现在我们已经有了用户，但我们需要共享组。为此，我们有一个类似的命令（在名称和操作上都相同）：`groupadd`。查看`groupadd`的 man 页面，并添加三个对应于我们部门的组：

```
reader@ubuntu:~$ sudo groupadd development
reader@ubuntu:~$ sudo groupadd operations
reader@ubuntu:~$ sudo groupadd security
reader@ubuntu:~$
```

要查看系统上已有哪些组，可以查看`/etc/group`文件（例如使用`less`或`cat`）。一旦满意，我们现在已经有了用户和组。但是我们如何使用户成为组的成员？输入`usermod`（表示**user** **mod**ify）。设置用户的主要组的语法如下：`usermod -g <groupname> <username>`：

```
reader@ubuntu:~$ sudo usermod -g development dev-user1 
reader@ubuntu:~$ sudo usermod -g operations ops-user1 
reader@ubuntu:~$ sudo usermod -g security sec-user1 
reader@ubuntu:~$ id dev-user1 
uid=1001(dev-user1) gid=1008(development) groups=1008(development)
reader@ubuntu:~$ id ops-user1 
uid=1002(ops-user1) gid=1009(operations) groups=1009(operations)
reader@ubuntu:~$ id sec-user1 
uid=1003(sec-user1) gid=1010(security) groups=1010(security)
reader@ubuntu:~$
```

我们现在已经实现的更接近我们的目标，但我们还没有到达那里。到目前为止，我们只确保多个开发人员可以通过所有在开发组中的文件共享文件。但是开发和运营之间的共享文件夹呢？安全性如何监视所有内容？让我们创建一些具有正确组的目录（使用`mkdir`，表示**m**a**k**e **dir**ectory），看看我们能走多远：

```
reader@ubuntu:~$ sudo mkdir /data
[sudo] password for reader:
reader@ubuntu:~$ cd /data
reader@ubuntu:/data$ sudo mkdir dev-files
reader@ubuntu:/data$ sudo mkdir ops-files
reader@ubuntu:/data$ sudo mkdir devops-files
reader@ubuntu:/data$ ls -l
total 12
drwxr-xr-x 2 root root 4096 Aug 11 10:03 dev-files
drwxr-xr-x 2 root root 4096 Aug 11 10:04 devops-files
drwxr-xr-x 2 root root 4096 Aug 11 10:04 ops-files
reader@ubuntu:/data$ sudo chgrp development dev-files/
reader@ubuntu:/data$ sudo chgrp operations ops-files/
reader@ubuntu:/data$ sudo chmod 0770 dev-files/
reader@ubuntu:/data$ sudo chmod 0770 ops-files/
reader@ubuntu:/data$ ls -l
total 12
drwxrwx--- 2 root development 4096 Aug 11 10:03 dev-files
drwxr-xr-x 2 root root        4096 Aug 11 10:04 devops-files
drwxrwx--- 2 root operations  4096 Aug 11 10:04 ops-files
reader@ubuntu:/data
```

我们现在有以下结构：一个`/data/`顶级目录，其中包含`dev-files`和`ops-files`目录，分别由`development`和`operations`组拥有。现在，让我们满足安全部门可以进入这两个目录并管理文件的要求！除了使用`usermod`来更改主要组，我们还可以将用户追加到额外的组中。在这种情况下，语法是`usermod -a -G <groupnames> <username>`。让我们将`sec-user1`添加到`development`和`operations`组中：

```
reader@ubuntu:/data$ id sec-user1
uid=1003(sec-user1) gid=1010(security) groups=1010(security)
reader@ubuntu:/data$ sudo usermod -a -G development,operations sec-user1 
reader@ubuntu:/data$ id sec-user1
uid=1003(sec-user1) gid=1010(security) groups=1010(security),1008(development),1009(operations)
reader@ubuntu:/data$
```

安全部门的用户现在是所有新组的成员：安全、开发和运维。由于`/data/dev-files/`和`/data/ops-files/`都没有*其他人*的权限，我们当前的用户不应该能够进入其中任何一个，但`sec-user1`应该可以。让我们看看这是否正确：

```
reader@ubuntu:/data$ sudo su - sec-user1
No directory, logging in with HOME=/
$ cd /data/
$ ls -l
total 12
drwxrwx--- 2 root development 4096 Aug 11 10:03 dev-files
drwxr-xr-x 2 root root        4096 Aug 11 10:04 devops-files
drwxrwx--- 2 root operations  4096 Aug 11 10:04 ops-files
$ cd dev-files
$ pwd
/data/dev-files
$ touch security-file
$ ls -l
total 0
-rw-r--r-- 1 sec-user1 security 0 Aug 11 10:16 security-file
$ exit
reader@ubuntu:/data$
```

如果您跟着这个例子，您应该会发现我们引入了一个新命令：`su`。它是**s**witch **u**ser 的缩写，它允许我们在用户之间切换。如果您在前面加上`sudo`，您可以切换到一个用户，而不需要该用户的密码，只要您有这些权限。否则，您将需要输入密码（在这种情况下很难，因为我们还没有为用户设置密码）。您可能已经注意到，新用户的 shell 是不同的。这是因为我们还没有加载任何配置（这是为默认用户自动完成的）。不过，不用担心——它仍然是一个完全功能的 shell！我们的测试成功了：我们能够进入`dev-files`目录，即使我们不是开发人员。我们甚至能够创建一个文件。如果您愿意，可以验证在`ops-files`目录中也是可能的。

最后，让我们创建一个新组`devops`，我们将使用它来在开发人员和运维之间共享文件。创建组后，我们将像将`sec-user1`添加到`development`和`operations`组一样，将`dev-user1`和`ops-user1`添加到这个组中：

```
reader@ubuntu:/data$ sudo groupadd devops
reader@ubuntu:/data$ sudo usermod -a -G devops dev-user1 
reader@ubuntu:/data$ sudo usermod -a -G devops ops-user1 
reader@ubuntu:/data$ id dev-user1 
uid=1001(dev-user1) gid=1008(development) groups=1008(development),1011(devops)
reader@ubuntu:/data$ id ops-user1 
uid=1002(ops-user1) gid=1009(operations) groups=1009(operations),1011(devops)
reader@ubuntu:/data$ ls -l
total 12
drwxrwx--- 2 root development 4096 Aug 11 10:16 dev-files
drwxr-xr-x 2 root root        4096 Aug 11 10:04 devops-files
drwxrwx--- 2 root operations  4096 Aug 11 10:04 ops-files
reader@ubuntu:/data$ sudo chown root:devops devops-files/
reader@ubuntu:/data$ sudo chmod 0770 devops-files/
reader@ubuntu:/data$ ls -l
total 12
drwxrwx---  2 root development 4096 Aug 11 10:16 dev-files/
drwxrwx---  2 root devops      4096 Aug 11 10:04 devops-files/
drwxrwx---  2 root operations  4096 Aug 11 10:04 ops-files/
reader@ubuntu:/data$
```

我们现在有一个共享目录，`/data/devops-files/`，`dev-user1`和`ops-user1`都可以进入并创建文件。

作为练习，可以执行以下任何一项：

+   将`sec-user1`添加到`devops`组，以便它也可以审计共享文件

+   验证`dev-user1`和`ops-user1`是否可以在共享目录中写入文件

+   了解为什么`dev-user1`和`ops-user1`只能读取`devops`目录中的对方文件，但不能编辑它们（提示：本章的下一节*高级权限*将告诉您如何使用 SGID 解决这个问题）

# 高级权限

这涵盖了 Linux 的基本权限。然而，还有一些高级主题，我们想指出，但我们不会详细讨论它们。有关这些主题的更多信息，请查看本章末尾的*进一步阅读*部分。我们已经包括了文件属性、特殊文件权限和访问控制列表的参考。

# 文件属性

文件也可以具有以不同于我们目前所见的权限表达的属性。一个例子是使文件不可变（一个花哨的词，意思是它不能被更改）。不可变文件仍然具有正常的所有权和组以及 RWX 权限，但它不允许用户更改它，即使它包含可写权限。另一个特点是该文件不能被重命名。

其他文件属性包括*不可删除*、*仅追加*和*压缩*。有关文件属性的更多信息，请查看`lsattr`和`chattr`命令的 man 页面（`man lsattr`和`man chattr`）。

# 特殊文件权限

正如您可能已经在八进制表示部分注意到的那样，我们总是以零开头（0775，0640 等）。如果我们不使用它，为什么要包括零？该位置保留用于特殊文件权限：SUID、SGID 和粘滞位。它们具有类似的八进制表示法（其中 SUID 为 4，SGID 为 2，粘滞位为 1），并且以以下方式使用：

|  | **文件** | **目录** |
| --- | --- | --- |
| **SUID** | 文件以所有者的权限执行，无论哪个用户执行它。 | 什么也不做。 |
| **SGID** | 文件以组的权限执行，无论哪个用户执行它。 | 在此目录中创建的文件获得与目录相同的组。 |
| **粘滞位** | 什么也不做。 | 用户只能删除他们在这个目录中的文件。查看`/tmp/`目录以了解其最著名的用途。 |

# 访问控制列表（ACL）

ACL 是增加 UGO/RWX 系统灵活性的一种方式。使用`setfacl`（**set** **f**ile **acl**）和`getfacl`（**get** **f**ile **acl**），您可以为文件和目录设置额外的权限。因此，例如，使用 ACL，您可以说，虽然`/root/`目录通常只能由`root`用户访问，但也可以被`reader`用户读取。另一种实现这一点的方法是将`reader`用户添加到`root`组中，这也给了`reader`用户系统上的许多其他特权（任何对 root 组有权限的东西都已经授予了 reader 用户！）。尽管根据我们的经验，ACL 在实践中并不经常使用，但对于边缘情况，它们可能是复杂解决方案和简单解决方案之间的区别。

# 总结

在本章中，我们已经了解了 Linux 权限方案。我们已经学到了权限安排的两个主要轴：文件权限和文件所有权。对于文件权限，每个文件都有对*读*、*写*和*执行*权限的允许（或不允许）。这些权限的工作方式对文件和目录有所不同。权限是通过所有权应用的：文件始终由用户和组拥有。除了*用户*和*组*之外，还有其他人的文件权限，称为*其他*所有权。如果用户是文件的所有者或文件组的成员，那么这些权限对用户是可用的。否则，其他人需要有权限才能与文件交互。

接下来，我们学习了如何操纵文件权限和所有权。通过使用`chmod`和`umask`，我们能够以所需的方式获取文件权限。使用`sudo`，`chown`和`chgrp`，我们操纵了文件的所有者和组。对于`sudo`和`root`用户的使用给出了警告，因为两者都可以在很少的努力下使 Linux 系统无法操作。

我们继续以一个与多个用户一起工作的示例。我们使用`useradd`添加了三个额外的用户到系统，并使用`usermod`为他们分配了正确的组。我们看到这些用户可以成为相同组的成员，并以这种方式共享对文件的访问。

最后，我们简要介绍了 Linux 下高级权限的一些基础知识。*进一步阅读*部分包含了这些主题的更多信息。

本章介绍了以下命令：`id`、`touch`、`chmod`、`umask`、`chown`、`chgrp`、`sudo`、`useradd`、`groupadd`、`usermod`、`mkdir`和`su`。 

# 问题

1.  Linux 文件使用了哪三种权限？

1.  为 Linux 文件定义了哪三种所有权类型？

1.  哪个命令用于更改文件的权限？

1.  什么机制控制了新创建文件的默认权限？

1.  以下符号权限如何用八进制描述：`rwxrw-r--`？

1.  以下八进制权限如何用符号描述：`0644`？

1.  哪个命令允许我们获得超级用户权限？

1.  我们可以使用哪些命令来更改文件的所有权？

1.  我们如何安排多个用户共享文件访问？

1.  Linux 有哪些类型的高级权限？

# 进一步阅读

如果您想深入了解本章主题，以下资源可能会很有趣：

+   **Linux 基础** 作者 *Oliver Pelz*，Packt 出版社：[`www.packtpub.com/networking-and-servers/fundamentals-linux`](https://www.packtpub.com/networking-and-servers/fundamentals-linux)

+   **文件属性**：[`linoxide.com/how-tos/howto-show-file-attributes-in-linux/`](https://linoxide.com/how-tos/howto-show-file-attributes-in-linux/)

+   **特殊文件权限**：[`thegeeksalive.com/linux-special-permissions/`](https://thegeeksalive.com/linux-special-permissions/)

+   **访问控制列表**：[`www.tecmint.com/secure-files-using-acls-in-linux/`](https://www.tecmint.com/secure-files-using-acls-in-linux/)
