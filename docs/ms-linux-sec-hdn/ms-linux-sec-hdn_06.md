# 第六章：访问控制列表和共享目录管理

在上一章中，我们回顾了自主访问控制的基础知识。在本章中，我们将进一步讨论 DAC，探讨一些更高级的技术，让您可以使用 DAC 完全按照您的意愿进行操作。

本章主题包括：

+   为用户或组创建访问控制列表（ACL）

+   为目录创建继承的 ACL

+   通过使用 ACL 掩码删除特定权限

+   使用`tar --acls`选项防止在备份期间丢失 ACL

+   创建用户组并向其中添加成员

+   为组创建共享目录，并对其进行适当的权限设置

+   在共享目录上设置 SGID 位和粘着位

+   使用 ACL 仅允许组的某些成员访问共享目录中的文件

# 为用户或组创建访问控制列表

正常的 Linux 文件和目录权限设置还可以，但它们不够精细。通过 ACL，我们可以只允许某个人访问文件或目录，或者我们可以允许多个人访问文件或目录，并为每个人设置不同的权限。如果我们有一个对所有人都开放的文件或目录，我们可以使用 ACL 来允许不同级别的访问，无论是对组还是个人。在本章末尾，我们将把我们学到的知识整合起来，以便为组管理共享目录。

您可以使用`getfacl`查看文件或目录的访问控制列表。（请注意，您不能一次查看目录中的所有文件。）首先，让我们使用`getfacl`来查看`acl_demo.txt`文件上是否已经设置了任何访问控制列表：

```
[donnie@localhost ~]$ touch acl_demo.txt

[donnie@localhost ~]$ getfacl acl_demo.txt
# file: acl_demo.txt
# owner: donnie
# group: donnie
user::rw-
group::rw-
other::r--

[donnie@localhost ~]$
```

这里我们只看到正常的权限设置，所以没有 ACL。

设置 ACL 的第一步是从除文件用户之外的所有人那里删除所有权限。这是因为默认权限设置允许组成员具有读/写访问权限，其他人具有读取权限。因此，在不删除这些权限的情况下设置 ACL 将是毫无意义的：

```
[donnie@localhost ~]$ chmod 600 acl_demo.txt

[donnie@localhost ~]$ ls -l acl_demo.txt
-rw-------. 1 donnie donnie 0 Nov  9 14:37 acl_demo.txt
[donnie@localhost ~]$
```

使用`setfacl`设置 ACL 时，可以允许用户或组具有任何组合的读取、写入或执行权限。在我们的情况下，假设我想让 Maggie 读取文件，并阻止她具有写入或执行权限：

```
[donnie@localhost ~]$ setfacl -m u:maggie:r acl_demo.txt

[donnie@localhost ~]$ getfacl acl_demo.txt
# file: acl_demo.txt
# owner: donnie
# group: donnie
user::rw-
user:maggie:r--
group::---
mask::r--
other::---

[donnie@localhost ~]$ ls -l acl_demo.txt
-rw-r-----+ 1 donnie donnie 0 Nov  9 14:37 acl_demo.txt
[donnie@localhost ~]$
```

`setfacl`的`-m`选项意味着我们将修改 ACL。（在这种情况下创建一个也可以，但没关系。）`u:`表示我们正在为用户设置 ACL。然后我们列出用户的名称，后跟另一个冒号，以及我们要授予该用户的权限列表。在这种情况下，我们只允许 Maggie 读取访问。我们通过列出要应用此 ACL 的文件来完成命令。`getfacl`输出显示 Maggie 确实具有读取权限。最后，我们在`ls -l`输出中看到组被列为具有读取权限，即使我们已经在此文件上设置了`600`权限设置。但是，还有一个`+`号，这告诉我们该文件具有 ACL。当我们设置 ACL 时，ACL 的权限显示为`ls -l`中的组权限。

更进一步，假设我想让 Frank 对此文件具有读/写访问权限：

```
[donnie@localhost ~]$ setfacl -m u:frank:rw acl_demo.txt

[donnie@localhost ~]$ getfacl acl_demo.txt
# file: acl_demo.txt
# owner: donnie
# group: donnie
user::rw-
user:maggie:r--
user:frank:rw-
group::---
mask::rw-
other::---

[donnie@localhost ~]$ ls -l acl_demo.txt
-rw-rw----+ 1 donnie donnie 0 Nov  9 14:37 acl_demo.txt
[donnie@localhost ~]$
```

因此，我们可以为同一个文件分配两个或更多不同的 ACL。在`ls -l`输出中，我们看到我们已经为组设置了`rw`权限，这实际上只是我们在两个 ACL 中设置的权限的总结。

我们可以通过将`u:`替换为`g:`来为组访问设置 ACL：

```
[donnie@localhost ~]$ getfacl new_file.txt
# file: new_file.txt
# owner: donnie
# group: donnie
user::rw-
group::rw-
other::r--

[donnie@localhost ~]$ chmod 600 new_file.txt

[donnie@localhost ~]$ setfacl -m g:accounting:r new_file.txt

[donnie@localhost ~]$ getfacl new_file.txt
# file: new_file.txt
# owner: donnie
# group: donnie
user::rw-
group::---
group:accounting:r--
mask::r--
other::---

[donnie@localhost ~]$ ls -l new_file.txt
-rw-r-----+ 1 donnie donnie 0 Nov  9 15:06 new_file.txt
[donnie@localhost ~]$
```

`accounting`组的成员现在可以读取此文件。

# 为目录创建继承的访问控制列表

有时您可能希望在共享目录中创建的所有文件都具有相同的访问控制列表。我们可以通过将继承的 ACL 应用于目录来实现这一点。尽管，要理解的是，即使这听起来像一个很酷的主意，以正常方式创建文件将导致文件对组设置读/写权限，并对其他用户设置读权限。因此，如果您为用户正常创建文件的目录设置了这一点，您最好希望创建一个 ACL，为某人添加写或执行权限。或者确保用户为他们创建的所有文件设置`600`权限设置，假设用户确实需要限制对他们的文件的访问。

另一方面，如果您正在创建一个在特定目录中创建文件的 shell 脚本，您可以包括`chmod`命令，以确保文件以必要的限制权限创建，以使您的 ACL 按预期工作。

为了演示，让我们创建`new_perm_dir`目录，并在其中设置继承的 ACL。我希望我的 shell 脚本在此目录中创建的文件具有读/写访问权限，并且 Frank 只能具有读取权限。我不希望其他人能够读取这些文件：

```
[donnie@localhost ~]$ setfacl -m d:u:frank:r new_perm_dir

[donnie@localhost ~]$ ls -ld new_perm_dir
drwxrwxr-x+ 2 donnie donnie 26 Nov 12 13:16 new_perm_dir
[donnie@localhost ~]$ getfacl new_perm_dir
# file: new_perm_dir
# owner: donnie
# group: donnie
user::rwx
group::rwx
other::r-x
default:user::rwx
default:user:frank:r--
default:group::rwx
default:mask::rwx
default:other::r-x

[donnie@localhost ~]$

```

我只需要做的就是在`u:frank`之前添加`d:`，使其成为继承的 ACL。我保留了目录上的默认权限设置，允许每个人对目录进行读取。接下来，我将创建`donnie_script.sh` shell 脚本，该脚本将在该目录中创建一个文件，并且仅为新文件的用户设置读/写权限：

```
#!/bin/bash
cd new_perm_dir
touch new_file.txt
chmod 600 new_file.txt
exit
```

使脚本可执行后，我将运行它并查看结果：

```
[donnie@localhost ~]$ ./donnie_script.sh

[donnie@localhost ~]$ cd new_perm_dir

[donnie@localhost new_perm_dir]$ ls -l
total 0
-rw-------+ 1 donnie donnie 0 Nov 12 13:16 new_file.txt
[donnie@localhost new_perm_dir]$ getfacl new_file.txt
# file: new_file.txt
# owner: donnie
# group: donnie
user::rw-
user:frank:r-- #effective:---
group::rwx #effective:---
mask::---
other::---

[donnie@localhost new_perm_dir]$
```

所以，`new_file.txt`以正确的权限设置创建，并且具有允许 Frank 读取的 ACL。（我知道这是一个非常简化的例子，但您明白我的意思。）

# 通过使用 ACL 掩码删除特定权限

您可以使用`-x`选项从文件或目录中删除 ACL。让我们回到我之前创建的`acl_demo.txt`文件，并删除 Maggie 的 ACL：

```
[donnie@localhost ~]$ setfacl -x u:maggie acl_demo.txt

[donnie@localhost ~]$ getfacl acl_demo.txt
# file: acl_demo.txt
# owner: donnie
# group: donnie
user::rw-
user:frank:rw-
group::---
mask::rw-
other::---

[donnie@localhost ~]$
```

所以，Maggie 的 ACL 消失了。但是，`-x`选项会删除整个 ACL，即使这并不是您真正想要的。如果您有一个设置了多个权限的 ACL，您可能只想删除一个权限，而保留其他权限。在这里，我们看到 Frank 仍然有他的 ACL，允许他读/写访问。现在，假设我们想要删除写权限，同时仍然允许他读权限。为此，我们需要应用一个掩码：

```
[donnie@localhost ~]$ setfacl -m m::r acl_demo.txt

[donnie@localhost ~]$ ls -l acl_demo.txt

-rw-r-----+ 1 donnie donnie 0 Nov  9 14:37 acl_demo.txt
[donnie@localhost ~]$ getfacl acl_demo.txt
# file: acl_demo.txt
# owner: donnie
# group: donnie
user::rw-
user:frank:rw-            #effective:r--
group::---
mask::r--
other::---

[donnie@localhost ~]$
```

`m::r`在 ACL 上设置了只读掩码。运行`getfacl`显示 Frank 仍然具有读/写 ACL，但旁边的注释显示他的有效权限是只读。因此，Frank 的文件写权限现在消失了。如果我们为其他用户设置了 ACL，这个掩码也会以相同的方式影响他们。

# 使用 tar --acls 选项来防止备份期间 ACLs 的丢失

如果您需要使用`tar`来创建文件或目录的备份，并且这些文件或目录有 ACLs 分配给它，您需要包括`--acls`选项开关。否则，ACLs 将会丢失。为了证明这一点，我将创建`perm_demo_dir`目录的备份，没有使用`--acls`选项。首先，请注意，我在此目录中的文件上有 ACLs，最后两个文件上的`+`符号表示了这一点：

```
[donnie@localhost ~]$ cd perm_demo_dir
[donnie@localhost perm_demo_dir]$ ls -l
total 0
-rw-rw-r--. 1 donnie accounting 0 Nov  5 20:17 file1.txt
-rw-rw-r--. 1 donnie accounting 0 Nov  5 20:17 file2.txt
-rw-rw-r--. 1 donnie accounting 0 Nov  5 20:17 file3.txt
-rw-rw-r--. 1 donnie accounting 0 Nov  5 20:17 file4.txt
-rw-rw----+ 1 donnie donnie     0 Nov  9 15:19 frank_file.txt
-rw-rw----+ 1 donnie donnie     0 Nov 12 12:29 new_file.txt
[donnie@localhost perm_demo_dir]$
```

现在，我将不使用`--acls`进行备份：

```
[donnie@localhost perm_demo_dir]$ cd
[donnie@localhost ~]$ tar cJvf perm_demo_dir_backup.tar.xz perm_demo_dir/
perm_demo_dir/
perm_demo_dir/file1.txt
perm_demo_dir/file2.txt
perm_demo_dir/file3.txt
perm_demo_dir/file4.txt
perm_demo_dir/frank_file.txt
perm_demo_dir/new_file.txt
[donnie@localhost ~]$
```

看起来不错，对吧？啊，但外表可能是具有欺骗性的。看看当我删除目录，然后从备份中恢复它时会发生什么：

```
[donnie@localhost ~]$ rm -rf perm_demo_dir/

[donnie@localhost ~]$ tar xJvf perm_demo_dir_backup.tar.xz
perm_demo_dir/
perm_demo_dir/file1.txt
perm_demo_dir/file2.txt
perm_demo_dir/file3.txt
perm_demo_dir/file4.txt
perm_demo_dir/frank_file.txt
perm_demo_dir/new_file.txt
[donnie@localhost ~]$ ls -l
total 812
. . .
drwxrwxr-x+ 2 donnie donnie 26 Nov 12 13:16 new_perm_dir
drwxrwx---. 2 donnie donnie 116 Nov 12 12:29 perm_demo_dir
-rw-rw-r--. 1 donnie donnie 284 Nov 13 13:45 perm_demo_dir_backup.tar.xz
. . .
[donnie@localhost ~]$ cd perm_demo_dir/

[donnie@localhost perm_demo_dir]$ ls -l
total 0
-rw-rw-r--. 1 donnie donnie 0 Nov 5 20:17 file1.txt
-rw-rw-r--. 1 donnie donnie 0 Nov 5 20:17 file2.txt
-rw-rw-r--. 1 donnie donnie 0 Nov 5 20:17 file3.txt
-rw-rw-r--. 1 donnie donnie 0 Nov 5 20:17 file4.txt
-rw-rw----. 1 donnie donnie 0 Nov 9 15:19 frank_file.txt
-rw-rw----. 1 donnie donnie 0 Nov 12 12:29 new_file.txt
[donnie@localhost perm_demo_dir]$
```

我甚至不需要使用`getfacl`来查看`perm_demo_dir`目录及其所有文件中的 ACL 已经消失，因为它们现在已经没有了`+`符号。现在，让我们看看当我包括`--acls`选项时会发生什么。首先，我将向您展示为此目录及其唯一文件设置了 ACL：

```
[donnie@localhost ~]$ ls -ld new_perm_dir
drwxrwxr-x+ 2 donnie donnie 26 Nov 13 14:01 new_perm_dir

[donnie@localhost ~]$ ls -l new_perm_dir
total 0
-rw-------+ 1 donnie donnie 0 Nov 13 14:01 new_file.txt
[donnie@localhost ~]$
```

现在，我将使用带有`--acls`的 tar：

```
[donnie@localhost ~]$ tar cJvf new_perm_dir_backup.tar.xz new_perm_dir/ --acls
new_perm_dir/
new_perm_dir/new_file.txt
[donnie@localhost ~]$
```

现在我将删除`new_perm_dir`目录，并从备份中恢复它。同样，我将使用`--acls`选项：

```
[donnie@localhost ~]$ rm -rf new_perm_dir/

[donnie@localhost ~]$ tar xJvf new_perm_dir_backup.tar.xz --acls
new_perm_dir/
new_perm_dir/new_file.txt

[donnie@localhost ~]$ ls -ld new_perm_dir
drwxrwxr-x+ 2 donnie donnie 26 Nov 13 14:01 new_perm_dir

[donnie@localhost ~]$ ls -l new_perm_dir
total 0
-rw-------+ 1 donnie donnie 0 Nov 13 14:01 new_file.txt
[donnie@localhost ~]$
```

`+`符号的存在表明 ACL 确实在备份和恢复过程中幸存下来了。这其中稍微棘手的部分是，你必须在备份和恢复时都使用`--acls`。如果你在任一次中省略了选项，你将丢失你的 ACL。

# 创建用户组并向其中添加成员

到目前为止，我一直在自己的家目录中进行所有演示，只是为了展示基本概念。但最终的目标是向你展示如何利用这些知识做一些更实际的事情，比如在共享组目录中控制文件访问。第一步是创建一个用户组并向其中添加成员。

假设我们想为市场部门的成员创建一个`marketing`组：

```
[donnie@localhost ~]$ sudo groupadd marketing
[sudo] password for donnie:
[donnie@localhost ~]$
```

现在让我们添加一些成员。你可以通过三种不同的方式来做到这一点：

+   在创建用户帐户时添加成员

+   使用`usermod`来添加已经有用户帐户的成员

+   编辑`/etc/group`文件

# 在创建用户帐户时添加成员

首先，我们可以在创建用户帐户时将成员添加到组中，使用`useradd`的`-G`选项。在 Red Hat 或 CentOS 上，命令看起来是这样的：

```
[donnie@localhost ~]$ sudo useradd -G marketing cleopatra
[sudo] password for donnie:

[donnie@localhost ~]$ groups cleopatra
cleopatra : cleopatra marketing
[donnie@localhost ~]$
```

在 Debian/Ubuntu 上，命令看起来是这样的：

```
donnie@ubuntu3:~$ sudo useradd -m -d /home/cleopatra -s /bin/bash -G marketing cleopatra

donnie@ubuntu3:~$ groups cleopatra
cleopatra : cleopatra marketing
donnie@ubuntu3:~$
```

当然，我还需要以正常方式为 Cleopatra 分配一个密码：

```
[donnie@localhost ~]$ sudo passwd cleopatra
```

# 使用 usermod 将现有用户添加到组

好消息是，在 Red Hat 或 CentOS 或 Debian/Ubuntu 上都是一样的：

```
[donnie@localhost ~]$ sudo usermod -a -G marketing maggie
[sudo] password for donnie:

[donnie@localhost ~]$ groups maggie
maggie : maggie marketing
[donnie@localhost ~]$
```

在这种情况下，`-a`是不必要的，因为 Maggie 不是任何其他次要组的成员。但是，如果她已经属于另一个组，那么`-a`将是必要的，以防止覆盖任何现有的组信息，从而将她从以前的组中删除。

这种方法在 Ubuntu 系统上特别方便，因为在创建加密的家目录时需要使用`adduser`。（正如我们在前一章中看到的，`adduser`不会给你在创建帐户时添加用户到组的机会。）

# 通过编辑/etc/group 文件将用户添加到组

这种最后的方法是一个好办法，可以加快将多个现有用户添加到组的过程。首先，只需在您喜欢的文本编辑器中打开`/etc/group`文件，并查找定义要添加成员的组的行：

```
. . .
marketing:x:1005:cleopatra,maggie
. . .
```

所以，我已经将 Cleopatra 和 Maggie 添加到了这个组。让我们编辑一下，再添加几个成员：

```
. . .
marketing:x:1005:cleopatra,maggie,vicky,charlie
. . .
```

完成后，保存文件并退出编辑器。

对于每个成员的`groups`命令将显示我们微小作弊的效果非常好：

```
[donnie@localhost etc]$ sudo vim group

[donnie@localhost etc]$ groups vicky
vicky : vicky marketing

[donnie@localhost etc]$ groups charlie
charlie : charlie marketing
[donnie@localhost etc]$
```

这种方法非常适用于需要同时向组中添加大量成员的情况。

# 创建共享目录

我们情节中的下一个行动涉及创建一个共享目录，所有我们市场部门的成员都可以使用。现在，这是另一个引起一些争议的领域。有些人喜欢将共享目录放在文件系统的根级目录，而其他人喜欢将共享目录放在`/home`目录中。还有一些人甚至有其他偏好。但实际上，这是个人偏好和/或公司政策的问题。除此之外，你把它们放在哪里并不重要。为了简化事情，我将在文件系统的根级目录中创建目录：

```
[donnie@localhost ~]$ cd /

[donnie@localhost /]$ sudo mkdir marketing
[sudo] password for donnie:

[donnie@localhost /]$ ls -ld marketing
drwxr-xr-x. 2 root root 6 Nov 13 15:32 marketing
[donnie@localhost /]$
```

新目录属于 root 用户。它的权限设置为`755`，允许每个人读取和执行访问，只允许 root 用户写入访问。我们真正想要的是只允许市场部门的成员访问这个目录。我们首先更改所有权和组关联，然后设置适当的权限：

```
[donnie@localhost /]$ sudo chown nobody:marketing marketing

[donnie@localhost /]$ sudo chmod 770 marketing

[donnie@localhost /]$ ls -ld marketing
drwxrwx---. 2 nobody marketing 6 Nov 13 15:32 marketing
[donnie@localhost /]$
```

在这种情况下，我们没有任何特定的用户想要拥有该目录，我们也不希望 root 用户拥有它。因此，将所有权分配给`nobody`伪用户帐户为我们提供了一种处理的方式。然后，我将`770`权限值分配给目录，这允许所有`marketing`组成员读/写/执行访问，同时让其他人离开。现在，让我们让我们的一个组成员登录，看看她是否可以在这个目录中创建一个文件：

```
[donnie@localhost /]$ su - vicky
Password:

[vicky@localhost ~]$ cd /marketing

[vicky@localhost marketing]$ touch vicky_file.txt

[vicky@localhost marketing]$ ls -l
total 0
-rw-rw-r--. 1 vicky vicky 0 Nov 13 15:41 vicky_file.txt
[vicky@localhost marketing]$
```

好的，它有效了，除了一个小问题。文件属于 Vicky，这是应该的。但是，它也与 Vicky 的个人组相关联。为了更好地控制这些共享文件的访问权限，我们需要它们与`marketing`组相关联。

# 在共享目录上设置 SGID 位和粘滞位

我之前告诉过你，在文件上设置 SUID 或 SGID 权限，特别是在可执行文件上，这是一种安全风险。但是，在共享目录上设置 SGID 是完全安全且非常有用的。

目录上的 SGID 行为与文件上的 SGID 行为完全不同。在目录上，SGID 将导致任何人创建的文件都与目录关联的相同组相关联。因此，牢记 SGID 权限值为`2000`，让我们在我们的`marketing`目录上设置 SGID：

```
[donnie@localhost /]$ sudo chmod 2770 marketing
[sudo] password for donnie:

[donnie@localhost /]$ ls -ld marketing
drwxrws---. 2 nobody marketing 28 Nov 13 15:41 marketing
[donnie@localhost /]$
```

在组的可执行位置上的`s`表示命令执行成功。现在让 Vicky 重新登录创建另一个文件：

```
[donnie@localhost /]$ su - vicky
Password:
Last login: Mon Nov 13 15:41:19 EST 2017 on pts/0

[vicky@localhost ~]$ cd /marketing

[vicky@localhost marketing]$ touch vicky_file_2.txt

[vicky@localhost marketing]$ ls -l
total 0
-rw-rw-r--. 1 vicky marketing 0 Nov 13 15:57 vicky_file_2.txt
-rw-rw-r--. 1 vicky vicky     0 Nov 13 15:41 vicky_file.txt
[vicky@localhost marketing]$
```

Vicky 的第二个文件与`marketing`组相关联，这正是我们想要的。只是为了好玩，让我们让 Charlie 做同样的事情：

```
[donnie@localhost /]$ su - charlie
Password:

[charlie@localhost ~]$ cd /marketing

[charlie@localhost marketing]$ touch charlie_file.txt

[charlie@localhost marketing]$ ls -l
total 0
-rw-rw-r--. 1 charlie marketing 0 Nov 13 15:59 charlie_file.txt
-rw-rw-r--. 1 vicky   marketing 0 Nov 13 15:57 vicky_file_2.txt
-rw-rw-r--. 1 vicky   vicky     0 Nov 13 15:41 vicky_file.txt
[charlie@localhost marketing]$
```

再次，Charlie 的文件与`marketing`组相关联。但是，由于没有人能理解的一些奇怪原因，Charlie 真的不喜欢 Vicky，并决定删除她的文件，纯粹是出于恶意：

```
[charlie@localhost marketing]$ rm vicky*
rm: remove write-protected regular empty file ‘vicky_file.txt’? y

[charlie@localhost marketing]$ ls -l
total 0
-rw-rw-r--. 1 charlie marketing 0 Nov 13 15:59 charlie_file.txt
[charlie@localhost marketing]$
```

系统抱怨 Vicky 的原始文件受到写保护，因为它仍然与她的个人组相关联。但是，系统仍然允许 Charlie 删除它，即使没有 sudo 权限。而且，由于 Charlie 对第二个文件有写访问权限，因为它与`marketing`组相关联，系统允许他毫不犹豫地删除它。

好的。所以，Vicky 抱怨了这个问题，并试图让 Charlie 被解雇。但是，我们勇敢的管理员有一个更好的主意。他将设置粘滞位，以防止这种情况再次发生。由于 SGID 位的值为`2000`，粘滞位的值为`1000`，我们可以将两者相加得到值为`3000`：

```
[donnie@localhost /]$ sudo chmod 3770 marketing
[sudo] password for donnie:

[donnie@localhost /]$ ls -ld marketing
drwxrws--T. 2 nobody marketing 30 Nov 13 16:03 marketing
[donnie@localhost /]$
```

在其他人的可执行位置上的`T`表示设置了粘滞位。由于`T`是大写的，我们知道其他人的可执行权限没有被设置。设置了粘滞位将阻止组成员删除其他人的文件。让 Vicky 演示当她试图对 Charlie 进行报复时会发生什么：

```
[donnie@localhost /]$ su - vicky
Password:
Last login: Mon Nov 13 15:57:41 EST 2017 on pts/0

[vicky@localhost ~]$ cd /marketing

[vicky@localhost marketing]$ ls -l
total 0
-rw-rw-r--. 1 charlie marketing 0 Nov 13 15:59 charlie_file.txt

[vicky@localhost marketing]$ rm charlie_file.txt
rm: cannot remove ‘charlie_file.txt’: Operation not permitted

[vicky@localhost marketing]$ rm -f charlie_file.txt
rm: cannot remove ‘charlie_file.txt’: Operation not permitted

[vicky@localhost marketing]$ ls -l
total 0
-rw-rw-r--. 1 charlie marketing 0 Nov 13 15:59 charlie_file.txt
[vicky@localhost marketing]$
```

即使使用`-f`选项，Vicky 仍然无法删除 Charlie 的文件。Vicky 在这个系统上没有`sudo`权限，所以她尝试是没有用的。

# 使用 ACL 访问共享目录中的文件

目前，`marketing`组的所有成员都可以读/写其他组成员的文件。将对文件的访问限制为仅特定组成员与我们已经讨论过的相同的两步过程。

# 设置权限并创建 ACL

首先，Vicky 设置了普通权限，只允许她访问她的文件。然后，她会设置 ACL：

```
[vicky@localhost marketing]$ echo "This file is only for my good friend, Cleopatra." > vicky_file.txt

[vicky@localhost marketing]$ chmod 600 vicky_file.txt

[vicky@localhost marketing]$ setfacl -m u:cleopatra:r vicky_file.txt

[vicky@localhost marketing]$ ls -l
total 4
-rw-rw-r--. 1 charlie marketing 0 Nov 13 15:59 charlie_file.txt
-rw-r-----+ 1 vicky marketing 49 Nov 13 16:24 vicky_file.txt

[vicky@localhost marketing]$ getfacl vicky_file.txt
# file: vicky_file.txt
# owner: vicky
# group: marketing
user::rw-
user:cleopatra:r--
group::---
mask::r--
other::---

[vicky@localhost marketing]$
```

这里没有什么是你之前没有见过的。Vicky 只是从组和其他人那里删除了所有权限，并设置了一个 ACL，只允许 Cleopatra 读取文件。让我们看看 Cleopatra 是否真的能读取它：

```
[donnie@localhost /]$ su - cleopatra
Password:

[cleopatra@localhost ~]$ cd /marketing

[cleopatra@localhost marketing]$ ls -l
total 4
-rw-rw-r--. 1 charlie marketing 0 Nov 13 15:59 charlie_file.txt
-rw-r-----+ 1 vicky marketing 49 Nov 13 16:24 vicky_file.txt

[cleopatra@localhost marketing]$ cat vicky_file.txt
This file is only for my good friend, Cleopatra.
[cleopatra@localhost marketing]$
```

到目前为止，一切都很好。但是，Cleopatra 能写入吗？

```
[cleopatra@localhost marketing]$ echo "You are my friend too, Vicky." >> vicky_file.txt
-bash: vicky_file.txt: Permission denied
[cleopatra@localhost marketing]$
```

好的，Cleopatra 不能这样做，因为 Vicky 只允许她在 ACL 中拥有读权限。

# Charlie 尝试使用为 Cleopatra 设置的 ACL 访问 Vicky 的文件

现在，不过，那个狡猾的 Charlie 呢，他想要窥探其他用户的文件呢？

```
[donnie@localhost /]$ su - charlie
Password:
Last login: Mon Nov 13 15:58:56 EST 2017 on pts/0

[charlie@localhost ~]$ cd /marketing

[charlie@localhost marketing]$ cat vicky_file.txt
cat: vicky_file.txt: Permission denied
[charlie@localhost marketing]$
```

所以，是的，只有 Cleopatra 才能访问 Vicky 的文件，即使是只读也是如此。

# 动手实验 - 创建共享组目录

在这个实验中，你将把本章学到的一切放在一起，为一个组创建一个共享目录。你可以在你的任一虚拟机上完成这个操作：

1.  在任一虚拟机上创建`sales`组：

```
 sudo groupadd sales
```

1.  创建用户 Mimi、Mr. Gray 和 Mommy，并在创建账户时将他们添加到 sales 组中。

在 CentOS 虚拟机上执行：

```
        sudo useradd -G sales mimi
 sudo useradd -G sales mrgray
 sudo useradd -G sales mommy
```

在 Ubuntu 虚拟机上执行：

```
 sudo useradd -m -d /home/mimi -s /bin/bash -G sales mimi
 sudo useradd -m -d /home/mrgray -s /bin/bash -G sales mrgray
 sudo useradd -m -d /home/mommy -s /bin/bash -G sales mommy
```

1.  为每个用户分配一个密码。

1.  在文件系统的根目录级别创建`sales`目录。设置适当的所有权和权限，包括 SGID 和粘性位：

```
        sudo mkdir /sales
 sudo chown nobody:sales /sales
 sudo chmod 3770 /sales
 ls -ld /sales
```

1.  以 Mimi 的身份登录，并让她创建一个文件：

```
 su - mimi
 cd /sales
 echo "This file belongs to Mimi." > mimi_file.txt
 ls -l
```

1.  让 Mimi 在她的文件上设置 ACL，只允许 Mr. Gray 读取。然后，让 Mimi 退出登录：

```
 chmod 600 mimi_file.txt
 setfacl -m u:mrgray:r mimi_file.txt
 getfacl mimi_file.txt
 ls -l
 exit
```

1.  让 Mr. Gray 登录，看看他能否对 Mimi 的文件做些什么。然后，让 Mr. Gray 创建自己的文件并退出登录：

```
 su - mrgray
 cd /sales
 cat mimi_file.txt
 echo "I want to add something to this file." >>
 mimi_file.txt
 echo "Mr. Gray will now create his own file." >
 mr_gray_file.txt
 ls -l
 exit
```

1.  Mommy 现在将登录并尝试通过窥探其他用户的文件和尝试删除它们来制造混乱：

```
 su - mommy
 cat mimi_file.txt
 cat mr_gray_file.txt
 rm -f mimi_file.txt
 rm -f mr_gray_file.txt
 exit
```

1.  实验结束。

# 总结

在本章中，我们看到了如何将自主访问控制提升到更高的水平。我们首先看到了如何创建和管理访问控制列表，以提供对文件和目录的更精细的访问控制。然后，我们看到了如何为特定目的创建用户组，以及如何向其中添加成员。然后，我们看到了如何使用 SGID 位、粘性位和访问控制列表来管理共享组目录。

但有时，自主访问控制可能不足以完成工作。对于这些时候，我们还有强制访问控制，我们将在下一章中介绍。到时候再见。
