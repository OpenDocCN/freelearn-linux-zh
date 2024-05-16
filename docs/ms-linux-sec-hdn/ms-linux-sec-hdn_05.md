# 第五章：掌握自主访问控制

自主访问控制，DAC，实际上只意味着每个用户都有能力控制谁可以进入他或她的东西。如果我想打开我的家目录，以便系统上的每个其他用户都可以进入，我可以这样做。这样做后，我可以控制谁可以访问每个特定的文件。在下一章中，我们将使用我们的 DAC 技能来管理共享目录，其中组的成员可能需要对其中的文件有不同级别的访问权限。

到您的 Linux 职业的这一点上，您可能已经了解了通过设置文件和目录权限来控制访问的基础知识。在本章中，我们将回顾基础知识，然后我们将看一些更高级的概念。我们将涵盖的主题包括：

+   使用`chown`更改文件和目录的所有权

+   使用`chmod`在文件和目录上设置权限

+   SUID 和 SGID 设置对我们常规文件有什么作用

+   在不需要它们的文件上设置 SUID 和 SGID 权限的安全影响

+   如何使用扩展文件属性来保护敏感文件

# 使用 chown 更改文件和目录的所有权

真正控制对文件和目录的访问其实归结为确保适当的用户拥有文件和目录，并且每个文件和目录都以这样的方式设置了权限，以便只有授权用户才能访问它们。`chown`实用程序涵盖了这个方程式的第一部分。

关于`chown`的一个独特之处是，即使您在自己的目录中处理自己的文件，您也必须具有 sudo 特权才能使用它。您可以使用它来更改文件或目录的用户，与文件或目录关联的组，或同时更改两者。

首先，假设您拥有`perm_demo.txt`文件，并且希望将用户和组关联更改为另一个用户。在这种情况下，我将把文件所有权从我更改为 Maggie：

```
[donnie@localhost ~]$ ls -l perm_demo.txt
-rw-rw-r--. 1 donnie donnie 0 Nov  5 20:02 perm_demo.txt

[donnie@localhost ~]$ sudo chown maggie:maggie perm_demo.txt

[donnie@localhost ~]$ ls -l perm_demo.txt
-rw-rw-r--. 1 maggie maggie 0 Nov  5 20:02 perm_demo.txt
[donnie@localhost ~]$
```

`maggie:maggie`中的第一个`maggie`是您想要授予所有权的用户。冒号后面的第二个`maggie`表示您希望文件关联的组。由于我要同时更改用户和组为`maggie`，我可以省略第二个`maggie`，只需在冒号后面跟着第一个`maggie`，我将获得相同的结果：

```
sudo chown maggie: perm_demo.txt
```

只需更改组关联而不更改用户，只需列出组名称，前面加上冒号：

```
[donnie@localhost ~]$ sudo chown :accounting perm_demo.txt

[donnie@localhost ~]$ ls -l perm_demo.txt
-rw-rw-r--. 1 maggie accounting 0 Nov  5 20:02 perm_demo.txt
[donnie@localhost ~]$
```

最后，只需更改用户而不更改组，列出用户名而不带冒号：

```
[donnie@localhost ~]$ sudo chown donnie perm_demo.txt

[donnie@localhost ~]$ ls -l perm_demo.txt
-rw-rw-r--. 1 donnie accounting 0 Nov  5 20:02 perm_demo.txt
[donnie@localhost ~]$
```

这些命令在目录上的工作方式与在文件上的工作方式相同。但是，如果您还想更改目录的内容的所有权和/或组关联，同时也在目录本身上进行更改，请使用`-R`选项，表示*递归*。在这种情况下，我只想将`perm_demo_dir`目录的组更改为`会计`：

```
[donnie@localhost ~]$ ls -ld perm_demo_dir
drwxrwxr-x. 2 donnie donnie 74 Nov  5 20:17 perm_demo_dir

[donnie@localhost ~]$ ls -l perm_demo_dir
total 0
-rw-rw-r--. 1 donnie donnie 0 Nov  5 20:17 file1.txt
-rw-rw-r--. 1 donnie donnie 0 Nov  5 20:17 file2.txt
-rw-rw-r--. 1 donnie donnie 0 Nov  5 20:17 file3.txt
-rw-rw-r--. 1 donnie donnie 0 Nov  5 20:17 file4.txt

[donnie@localhost ~]$ sudo chown -R :accounting perm_demo_dir

[donnie@localhost ~]$ ls -ld perm_demo_dir
drwxrwxr-x. 2 donnie accounting 74 Nov  5 20:17 perm_demo_dir

[donnie@localhost ~]$ ls -l perm_demo_dir
total 0
-rw-rw-r--. 1 donnie accounting 0 Nov  5 20:17 file1.txt
-rw-rw-r--. 1 donnie accounting 0 Nov  5 20:17 file2.txt
-rw-rw-r--. 1 donnie accounting 0 Nov  5 20:17 file3.txt
-rw-rw-r--. 1 donnie accounting 0 Nov  5 20:17 file4.txt
[donnie@localhost ~]$

```

这就是`chown`的全部内容。

# 使用 chmod 在文件和目录上设置权限值

在 Unix 和 Linux 系统上，您将使用`chmod`实用程序在文件和目录上设置权限值。您可以为文件或目录的用户、与文件或目录关联的组以及其他人设置权限。三个基本权限是：

+   `r`：这表示读取权限。

+   `w`：这是写入权限。

+   `x`：这是可执行权限。您可以将其应用于任何类型的程序文件，或者目录。如果您将可执行权限应用于目录，授权的用户将能够`cd`进入它。

对文件和目录进行`ls -l`，您会看到类似以下内容：

```
-rw-rw-r--. 1 donnie donnie     804692 Oct 28 18:44 yum_list.txt
```

这一行的第一个字符表示文件的类型。在这种情况下，我们看到一个破折号，表示一个普通文件。（普通文件基本上是普通用户在日常例行工作中能够访问的各种类型的文件。）接下来的三个字符`rw-`表示该文件对用户（即文件所有者）具有读取和写入权限。然后我们看到组的`rw-`权限，以及其他人的`r--`权限。程序文件还会设置可执行权限：

```
-rwxr-xr-x. 1 root root     62288 Nov 20  2015 xargs
```

在这里，我们看到`xargs`程序文件为所有人设置了可执行权限。

有两种方法可以使用`chmod`来更改权限设置：

+   符号方法

+   数字方法

# 使用符号方法设置权限

每当您以普通用户身份创建文件时，默认情况下，它将对用户和组具有读取和写入权限，并对其他人具有只读权限。如果您创建一个程序文件，您必须自己添加可执行权限。使用符号方法，您可以使用以下命令之一来执行此操作：

```
chmod u+x donnie_script.sh
chmod g+x donnie_script.sh
chmod o+x donnie_script.sh
chmod u+x,g+x donnie_script.sh
chmod a+x donnie_script.sh
```

前三个命令为用户、组和其他人添加可执行权限。第四个命令为用户和组添加可执行权限，最后一个命令为所有人（`a`代表所有人）添加可执行权限。您也可以通过用`-`替换`+`来删除可执行权限。并且，您也可以根据需要添加或删除读取或写入权限。

虽然这种方法有时很方便，但它也有一点缺陷。也就是说，它只能添加已有权限，或者删除已有权限。如果您需要确保特定文件的所有权限都设置为特定值，符号方法可能会变得有些难以掌握。而对于 shell 脚本来说，更是如此。在 shell 脚本中，您需要添加各种额外的代码来确定已设置了哪些权限。数字方法可以极大地简化我们的工作。

# 使用数字方法设置权限

使用数字方法，您将使用八进制值来表示文件或目录的权限设置。对于`r`、`w`和`x`权限，分别分配数字值`4`、`2`和`1`。对用户、组和其他人的位置进行此操作，然后将它们相加以获得文件或目录的权限值：

| **用户** | **组** | **其他人** |
| --- | --- | --- |
| `rwx` | `rwx` | `rwx` |
| `421` | `421` | `421` |
| `7` | `7` | `7` |

因此，如果您为所有人设置了所有权限，文件或目录的值将为`777`。如果我要创建一个 shell 脚本文件，默认情况下，它将具有标准的`664`权限，即用户和组具有读取和写入权限，其他人只有只读权限：

```
-rw-rw-r--. 1 donnie donnie 0 Nov  6 19:18 donnie_script.sh
```

如果您以 root 权限创建文件，无论是使用 sudo 还是从 root 用户命令提示符，您会发现默认权限设置更为严格，为`644`。

假设我想要使这个脚本文件可执行，但我希望是全世界唯一可以对其进行任何操作的人。我可以这样做：

```
[donnie@localhost ~]$ chmod 700 donnie_script.sh

[donnie@localhost ~]$ ls -l donnie_script.sh
-rwx------. 1 donnie donnie 0 Nov  6 19:18 donnie_script.sh
[donnie@localhost ~]$
```

通过这个简单的命令，我已经从组和其他人那里删除了所有权限，并为自己设置了可执行权限。这就是数字方法在编写 shell 脚本时如此方便的地方。

一旦您使用数字方法一段时间后，查看文件并确定其数字权限值将变得轻车熟路。与此同时，您可以使用带有`-c %a`选项的`stat`来显示这些值。例如：

```
[donnie@localhost ~]$ stat -c %a yum_list.txt
664
[donnie@localhost ~]$

[donnie@localhost ~]$ stat -c %a donnie_script.sh
700
[donnie@localhost ~]$

[donnie@localhost ~]$ stat -c %a /etc/fstab
644
[donnie@localhost ~]$
```

# 使用 SUID 和 SGID 设置普通文件的权限

当普通文件设置了 SUID 权限时，访问该文件的人将具有与文件所有者相同的权限。当普通文件设置了 SGID 权限时，访问该文件的人将具有与文件关联的组相同的权限。这在程序文件上特别有用。

为了演示这一点，让我们假设 Maggie，一个普通的、没有特权的用户，想要更改自己的密码。因为这是她自己的密码，她只需使用单词命令`passwd`，而不使用`sudo`：

```
[maggie@localhost ~]$ passwd
Changing password for user maggie.
Changing password for maggie.
(current) UNIX password:
New password:
Retype new password:
passwd: all authentication tokens updated successfully.
[maggie@localhost ~]$
```

要更改密码，一个人必须对`/etc/shadow`文件进行更改。在我的 CentOS 机器上，shadow 文件的权限看起来像这样：

```
[donnie@localhost etc]$ ls -l shadow
----------. 1 root root 840 Nov  6 19:37 shadow
[donnie@localhost etc]$
```

在 Ubuntu 机器上，它们看起来像这样：

```
donnie@ubuntu:/etc$ ls -l shadow
-rw-r----- 1 root shadow 1316 Nov  4 18:38 shadow
donnie@ubuntu:/etc$
```

无论如何，权限设置不允许 Maggie 修改 shadow 文件。然而，通过更改她的密码，她可以修改 shadow 文件。那么，到底发生了什么？为了回答这个问题，让我们进入`/usr/bin`目录，查看`passwd`可执行文件的权限设置：

```
[donnie@localhost etc]$ cd /usr/bin

[donnie@localhost bin]$ ls -l passwd
-rwsr-xr-x. 1 root root 27832 Jun 10 2014 passwd
[donnie@localhost bin]$
```

对于用户权限，您会看到`rws`而不是`rwx`。`s`表示该文件设置了 SUID 权限。由于该文件属于 root 用户，访问该文件的任何人都具有与 root 用户相同的权限。我们看到小写`s`意味着该文件也为 root 用户设置了可执行权限。由于 root 用户被允许修改 shadow 文件，使用`passwd`实用程序来更改自己的密码的人也可以修改 shadow 文件。

设置了 SGID 权限的文件在组的可执行位置上有一个`s`：

```
[donnie@localhost bin]$ ls -l write
-rwxr-sr-x. 1 root tty 19536 Aug  4 07:18 write
[donnie@localhost bin]$
```

与`tty`组关联的`write`实用程序允许用户通过他们的命令行控制台向其他用户发送消息。拥有`tty`组权限允许用户这样做。

# SUID 和 SGID 权限的安全影响

尽管在可执行文件上设置 SUID 或 SGID 权限可能很有用，但我们应该将其视为必要的恶。虽然在某些操作系统文件上设置 SUID 或 SGID 对于 Linux 系统的正常运行至关重要，但当用户在其他文件上设置 SUID 或 SGID 时，它就成为了安全风险。问题在于，如果入侵者找到了属于 root 用户并设置了 SUID 位的可执行文件，他们可以利用它来攻击系统。在离开之前，他们可能会留下自己的属于 root 的文件，并设置 SUID 位，这将使他们很容易地在下次进入系统。如果找不到入侵者的 SUID 文件，即使原始问题得到解决，入侵者仍将有访问权限。

SUID 的数字值为`4000`，SGID 的数字值为`2000`。要在文件上设置 SUID，您只需将`4000`添加到您否则设置的权限值中。例如，如果您有一个权限值为`755`的文件，您可以通过将权限值更改为`4755`来设置 SUID。（这将为用户提供读/写/执行权限，为组提供读/执行权限，为其他用户提供读/执行权限，并添加 SUID 位。）

# 查找虚假的 SUID 或 SGID 文件

一个快速的安全技巧是运行`find`命令来清点系统上的 SUID 和 SGID 文件。您可以将输出保存到文本文件中，以便在下次运行命令时验证是否添加了任何内容。您的命令看起来会像这样：

```
sudo find / -type f \( -perm -4000 -o -perm 2000 \) > suid_sgid_files.txt
```

这是分解：

+   `/`：我们正在搜索整个文件系统。由于某些目录只能由具有 root 权限的用户访问，我们需要使用`sudo`。

+   `-type f`：这意味着我们正在搜索常规文件，这将包括可执行程序文件和 shell 脚本。

+   `-perm 4000`：我们正在搜索具有`4000`或 SUID 权限位设置的文件。

+   `-o`：或操作符。

+   `-perm 2000`：我们正在搜索具有`2000`或 SGID 权限位设置的文件。

+   `>`：当然，我们正在使用`>`操作符将输出重定向到`suid_sgid_files.txt`文本文件中。

请注意，两个`-perm`项需要合并为一个用一对括号括起来的术语。为了防止 Bash shell 错误解释括号字符，我们需要用反斜杠对每个括号进行转义。我们还需要在第一个括号字符和第一个`-perm`之间以及`2000`和最后一个反斜杠之间放置一个空格。此外，`-type f`和`-perm`术语之间的 and 运算符被理解为存在，即使没有插入`-a`。您创建的文本文件应该看起来像这样：

```
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/chage
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/su
/usr/bin/umount
/usr/bin/sudo
/usr/bin/pkexec
/usr/bin/crontab
/usr/bin/passwd
/usr/sbin/pam_timestamp_check
/usr/sbin/unix_chkpwd
/usr/sbin/usernetctl
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/lib64/dbus-1/dbus-daemon-launch-helper
```

如果您想查看哪些文件是 SUID 和哪些是 SGID 的详细信息，可以添加`-ls`选项：

```
sudo find / -type f \( -perm -4000 -o -perm 2000 \) -ls > suid_sgid_files.txt
```

现在，假设 Maggie 出于任何原因，决定在她的主目录中的一个 shell 脚本文件上设置 SUID 位：

```
[maggie@localhost ~]$ chmod 4755 bad_script.sh

[maggie@localhost ~]$ ls -l
total 0
-rwsr-xr-x. 1 maggie maggie 0 Nov  7 13:06 bad_script.sh
[maggie@localhost ~]$

```

再次运行`find`命令，将输出保存到另一个文本文件中。然后，对两个文件进行`diff`操作，查看发生了什么变化：

```
[donnie@localhost ~]$ diff suid_sgid_files.txt suid_sgid_files2.txt
17a18
> /home/maggie/bad_script.sh
[donnie@localhost ~]$
```

唯一的区别是添加了 Maggie 的 shell 脚本文件。

# 动手实验-搜索 SUID 和 SGID 文件

您可以在您的任一虚拟机上进行此实验。您将把`find`命令的输出保存到一个文本文件中：

1.  在整个文件系统中搜索所有具有 SUID 或 SGID 设置的文件，将输出保存到一个文本文件中：

```
 sudo find / -type f \( -perm -4000 -o -perm 2000 \) -ls > 
 suid_sgid_files.txt
```

1.  登录到系统上的任何其他用户帐户，并创建一个虚拟 shell 脚本文件。然后，在该文件上设置 SUID 权限，并注销到您自己的用户帐户：

```
        su - desired_user_account
 touch some_shell_script.sh
 chmod 4755 some_shell_script.sh
 ls -l some_shell_script.sh
 exit
```

1.  再次运行`find`命令，将输出保存到另一个文本文件中：

```
        sudo find / -type f \( -perm -4000 -o -perm 2000 \) -ls > 
 suid_sgid_files_2.txt
```

1.  查看两个文件之间的区别：

```
 diff suid_sgid_files.txt suid_sgid_files_2.txt
```

1.  实验结束。

# 防止在分区上使用 SUID 和 SGID

正如我们之前所说，您不希望用户对他们创建的文件分配 SUID 和 SGID，因为这会带来安全风险。您可以通过使用`nosuid`选项来阻止在分区上使用 SUID 和 SGID。因此，我在上一章中创建的`luks`分区的`/etc/fstab`文件条目将如下所示：

```
/dev/mapper/luks-6cbdce17-48d4-41a1-8f8e-793c0fa7c389 /secrets          xfs     nosuid  0 0
```

不同的 Linux 发行版在操作系统安装期间设置默认分区方案的方式不同。大多数情况下，业务的默认方式是将除`/boot`目录之外的所有目录都放在`/`分区下。如果您要设置自定义分区方案，可以将`/home`目录放在自己的分区上，并在那里设置`nosuid`选项。请记住，您不希望为`/`分区设置`nosuid`，否则您将得到一个无法正常运行的操作系统。

# 使用扩展文件属性保护敏感文件

扩展文件属性是帮助您保护敏感文件的另一个工具。它们不会阻止入侵者访问您的文件，但它们可以帮助您防止敏感文件被更改或删除。有很多扩展属性，但我们只需要查看与文件安全有关的属性。

首先，让我们执行`lsattr`命令，查看您已经设置的扩展属性。在 CentOS 机器上，您的输出看起来会像这样：

```
[donnie@localhost ~]$ lsattr
---------------- ./yum_list.txt
---------------- ./perm_demo.txt
---------------- ./perm_demo_dir
---------------- ./donnie_script.sh
---------------- ./suid_sgid_files.txt
---------------- ./suid_sgid_files2.txt
[donnie@localhost ~]$
```

到目前为止，我的任何文件上都没有设置扩展属性。

在 Ubuntu 机器上，输出看起来会更像这样：

```
donnie@ubuntu:~$ lsattr
-------------e-- ./file2.txt
-------------e-- ./secret_stuff_dir
-------------e-- ./secret_stuff_for_frank.txt.gpg
-------------e-- ./good_stuff
-------------e-- ./secret_stuff
-------------e-- ./not_secret_for_frank.txt.gpg
-------------e-- ./file4.txt
-------------e-- ./good_stuff_dir
donnie@ubuntu:~$
```

我们不用担心`e`属性，因为那只意味着分区是用 ext4 文件系统格式化的。CentOS 没有设置该属性，因为其分区是用 XFS 文件系统格式化的。

我们将查看的两个属性是：

+   `a`：您可以将文本附加到具有此属性的文件的末尾，但不能覆盖它。只有具有适当 sudo 特权的人才能设置或删除此属性。

+   `i`：这使文件不可变，只有具有适当 sudo 特权的人才能设置或删除它。具有此属性的文件无法被删除或以任何方式更改。也不可能创建具有此属性的文件的硬链接。

要设置或删除属性，您将使用`chattr`命令。您可以在文件上设置多个属性，但只有在有意义的情况下才能这样做。例如，您不会在同一个文件上同时设置`a`和`i`属性，因为`i`会覆盖`a`。

让我们首先创建带有这个文本的`perm_demo.txt`文件：

```
This is Donnie's sensitive file that he doesn't want to have overwritten.
```

# 设置`a`属性

现在我将设置`a`属性：

```
[donnie@localhost ~]$ sudo chattr +a perm_demo.txt
[sudo] password for donnie:
[donnie@localhost ~]$
```

您将使用`+`来添加属性，使用`-`来删除属性。此外，文件属于我，位于我的家目录中也无关紧要。我仍然需要 sudo 权限来添加或删除此属性。

现在，让我们看看当我尝试覆盖这个文件时会发生什么：

```
[donnie@localhost ~]$ echo "I want to overwrite this file." > perm_demo.txt
-bash: perm_demo.txt: Operation not permitted

[donnie@localhost ~]$ sudo echo "I want to overwrite this file." > perm_demo.txt
-bash: perm_demo.txt: Operation not permitted
[donnie@localhost ~]$
```

无论是否具有`sudo`权限，我都无法覆盖它。那么，如果我尝试向其追加一些内容呢？

```
[donnie@localhost ~]$ echo "I want to append this to the end of the file." >> perm_demo.txt
[donnie@localhost ~]$
```

这次没有错误消息。让我们看看文件中现在有什么：

```
This is Donnie's sensitive file that he doesn't want to have overwritten.
I want to append this to the end of the file.
```

除了无法覆盖文件之外，我也无法删除它：

```
[donnie@localhost ~]$ rm perm_demo.txt
rm: cannot remove ‘perm_demo.txt’: Operation not permitted

[donnie@localhost ~]$ sudo rm perm_demo.txt
[sudo] password for donnie:
rm: cannot remove ‘perm_demo.txt’: Operation not permitted
[donnie@localhost ~]$
```

所以，`a`有效。但是，我决定不再设置这个属性，所以我会将其删除：

```
[donnie@localhost ~]$ sudo chattr -a perm_demo.txt
[donnie@localhost ~]$ lsattr perm_demo.txt
---------------- perm_demo.txt
[donnie@localhost ~]$
```

# 设置`i`属性

当文件设置了`i`属性时，您唯一能做的事情就是查看其内容。您不能更改它，移动它，删除它，重命名它，或者创建硬链接到它。让我们用`perm_demo.txt`文件来测试一下：

```
[donnie@localhost ~]$ sudo chattr +i perm_demo.txt
[donnie@localhost ~]$ lsattr perm_demo.txt
----i----------- perm_demo.txt
[donnie@localhost ~]$
```

现在，到有趣的部分：

```
[donnie@localhost ~]$ sudo echo "I want to overwrite this file." > perm_demo.txt
-bash: perm_demo.txt: Permission denied
[donnie@localhost ~]$ echo "I want to append this to the end of the file." >> perm_demo.txt
-bash: perm_demo.txt: Permission denied
[donnie@localhost ~]$ sudo echo "I want to append this to the end of the file." >> perm_demo.txt
-bash: perm_demo.txt: Permission denied
[donnie@localhost ~]$ rm -f perm_demo.txt
rm: cannot remove ‘perm_demo.txt’: Operation not permitted
[donnie@localhost ~]$ sudo rm -f perm_demo.txt
rm: cannot remove ‘perm_demo.txt’: Operation not permitted
[donnie@localhost ~]$ sudo rm -f perm_demo.txt
```

还有一些命令我可以尝试，但您已经明白了。要删除`i`属性，我会执行：

```
[donnie@localhost ~]$ sudo chattr -i perm_demo.txt
[donnie@localhost ~]$ lsattr perm_demo.txt
---------------- perm_demo.txt
[donnie@localhost ~]$
```

# 动手实验 - 设置与安全相关的扩展文件属性

对于这个实验，您将创建一个带有您自己选择的文本的`perm_demo.txt`文件。您将设置`i`和`a`属性，并查看结果：

1.  使用您喜欢的文本编辑器，创建带有一行文本的`perm_demo.txt`文件。

1.  查看文件的扩展属性：

```
 lsattr perm_demo.txt
```

1.  添加`a`属性：

```
        sudo chattr +a perm_demo.txt
 lsattr perm_demo.txt
```

1.  尝试覆盖和删除文件：

```
 echo "I want to overwrite this file." > perm_demo.txt
 sudo echo "I want to overwrite this file." > perm_demo.txt
 rm perm_demo.txt
 sudo rm perm_demo.txt
```

1.  现在，向文件追加一些内容：

```
        echo "I want to append this line to the end of the file." >> 
 perm_demo.txt
```

1.  删除`a`属性，并添加`i`属性：

```
 sudo chattr -a perm_demo.txt
 lsattr perm_demo.txt
 sudo chattr +i perm_demo.txt
 lsattr perm_demo.txt
```

1.  重复第 4 步。

1.  另外，尝试更改文件名并创建文件的硬链接：

```
 mv perm_demo.txt some_file.txt
 sudo mv perm_demo.txt some_file.txt
 ln ~/perm_demo.txt ~/some_file.txt
 sudo ln ~/perm_demo.txt ~/some_file.txt
```

1.  现在，尝试创建文件的符号链接：

```
        ln -s ~/perm_demo.txt ~/some_file.txt
```

请注意，`i`属性不允许您创建文件的硬链接，但它将允许您创建符号链接。

1.  实验结束。

# 总结

在本章中，我们回顾了为文件和目录设置所有权和权限的基础知识。然后，我们介绍了当正确使用时 SUID 和 SGID 可以为我们做什么，以及在我们自己的可执行文件上设置它们的风险。最后，我们通过查看处理文件安全的两个扩展文件属性来完成了这一系列。

在下一章中，我们将把我们在这里学到的知识扩展到更高级的文件和目录访问技术。我会在那里见到你。
