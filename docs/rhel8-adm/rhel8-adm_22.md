# 第十八章：*第十八章*：**练习 1**

在这个练习中，我们将运行一系列步骤，检查您在整本书中所学到的知识。与之前的章节不同，不会指示所有步骤，因此您可以自行决定执行所需的步骤以完成您的目标。建议避免参考以前的章节进行指导。相反，尝试使用您的记忆或系统中可用的工具。如果正确执行此练习，将有效地为官方考试进行培训。

强烈建议在进行此练习时使用时钟来跟踪时间。

# **技术要求**

**本章中的所有练习都需要使用虚拟机**（**VM**），运行安装了基本安装的 Red Hat Enterprise Linux 8。此外，存储操作将需要新的虚拟驱动器。

对于练习，假设您拥有以下内容：

+   安装了基本操作系统**最小安装**软件选择的 Red Hat Enterprise Linux 8。

+   访问 Red Hat 客户门户，具有有效的订阅。

+   虚拟机必须是可扩展的。这是因为在练习期间对其执行的操作可能使其无法使用，并需要重新安装。

# 练习提示

这是任何测试的一般建议清单，大多数属于常识范畴，但在进行任何测试之前将它们牢记在心是非常重要的：

+   在开始官方考试或任何测试之前，请先阅读所有问题。

+   特定的词语具有特定的含义，可以提示关于要求或完成目标的方法。这就是为什么先阅读所有内容可能会给您多个完成测试的途径。

+   让自己感到舒适。安装您喜欢的编辑器，并运行`updatedb`以获得一个新的软件包和已安装文件的数据库，以备使用。定义您的键盘布局。安装`tmux`并学习如何使用它，这样您就可以打开新标签并命名它们，而无需额外的窗口。

+   查找请求之间的依赖关系，因为有些目标取决于其他目标的完成。找到这些依赖关系，看看如何在不必后来回来重新做一些步骤的情况下找到解决方案。

+   使用计时器。这对于了解哪些练习将花费更多时间来完成是很重要的，以便看到您需要改进的领域。

+   不要记住具体的命令行。学习如何使用系统中的文档，通过`man`、`/usr/share/docs`或像`--help`这样的参数来获取所需命令的帮助。

+   确保更改持久并在重新启动后仍然有效。有些更改可能在运行时是有效的，但必须持久。例如防火墙规则、启动时要启动的服务等。

+   记住使用`dnf whatprovides /COMMAND"`来查找提供您可能缺少的文件的软件包。

+   检查以下链接：[`www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam?=Objectives`](https://www.redhat.com/en/services/training/ex200-red-hat-certified-system-administrator-rhcsa-exam?=Objectives)。这将为您提供官方 EX200 考试目标。

# 练习 1

重要提示

以下练习是有意设计的，因此不会突出显示命令、软件包等。记住迄今为止学到的知识，以便检测关键字，看看需要做什么。

不要过早地进行实际操作。试着记住已经涵盖的内容。

## 练习

1.  将时区配置为 GMT。

1.  允许 root 用户使用 SSH 进行无密码登录。

1.  创建一个可以无密码连接到机器的用户（名为*user*）。

1.  用户`user`应该每周更改密码，提前 2 天警告，过期后使用 1 天。

1.  root 用户必须能够以*user*的身份通过 SSH 进行连接，而无需密码，以便没有人可以使用密码远程连接为 root。

1.  用户*user*应能够在无需密码的情况下成为 root 用户，并且还能够在无需密码的情况下执行命令。

1.  当用户尝试通过 SSH 登录时，显示有关不允许未经授权访问该系统的法律消息。

1.  SSH 必须在端口*22222*上监听，而不是默认的端口（*22*）。

1.  创建一个名为`devel`的组。

1.  将`user`添加为`devel`的成员。

1.  将用户成员身份存储在名为`userids`的文件中，位于*user*的主文件夹中。

1.  用户*user*和*root*用户应能够通过 SSH 连接到本地主机，而无需指定端口，并默认使用压缩进行连接。

1.  查找系统中所有的 man 页名称，并将名称放入名为*manpages.txt*的文件中。

1.  打印未允许登录到系统的用户的用户名。对于每个用户名，打印该用户的用户 ID 和组。

1.  每 5 分钟监视可用系统资源。不使用 cron。存储为*/root/resources.log*。

1.  添加一个每分钟的作业，报告可用磁盘空间的百分比，并将其存储在*/root/freespace.log*中，以便显示文件系统和可用空间。

1.  配置系统仅保留 3 天的日志。

1.  配置*/root/freespace.log*和*/root/resources.log*的日志轮换。

1.  使用快速同步配置时间同步针对*pool.ntp.org*。

1.  为子网*172.22.0.1/24*提供 NTP 服务器服务。

1.  配置系统统计信息，每分钟收集一次。

1.  将系统中用户的密码长度配置为 12 个字符。

1.  创建一个名为*privacy*的机器人用户，其文件默认情况下只对自己可见。

1.  创建一个在*shared*中可以被所有用户访问的文件夹，并将新文件和目录默认为仍然可以被*devel*组的用户访问。

1.  配置一个名为*mynic*的具有 IPv4 和 IPv6 地址的网络连接，使用以下数据：

```
Ip6: 2001:db8:0:1::c000:207/64 g
gateway 2001:db8:0:1::1 
Ipv4 192.0.1.3/24 
gateway 192.0.1.1 
```

1.  允许主机使用*google*主机名访问[www.google.com](https://www.google.com)，并使用*redhat*主机名访问[www.redhat.com](https://www.redhat.com)。

1.  报告从供应商分发的文件中修改的文件，并将它们存储在*/root/altered.txt*中。

1.  使我们的系统安装媒体包通过 HTTP 在*/mirror*路径下可供其他系统用作镜像，配置我们系统中的存储库。从该镜像中删除内核包，以便其他系统（甚至我们自己的系统）无法找到新的内核。防止从该存储库安装 glibc 包而不将其删除。

1.  在*user*身份下，将*/root*文件夹复制到*/home/user/root/*文件夹，并使其每天保持同步，同步添加和删除。

1.  检查我们的系统是否符合 PCI-DSS 标准。

1.  向系统添加第二个 30GB 的硬盘。但是，只使用 15GB 将镜像移动到其中，并使用压缩和去重功能使其在启动时可用。将其放置在*/mirror/mirror*下。

1.  由于我们计划基于相同数据镜像自定义软件包集，因此配置文件系统报告至少可供我们的镜像使用的 1,500GB。

1.  在*/mirror/mytailormirror*下创建镜像的第二个副本，删除所有以字母*k*开头的包。

1.  在添加的硬盘剩余空间（15GB）中创建一个新卷，并将其用于扩展根文件系统。

1.  创建一个启动项，允许您进入紧急模式，以更改根密码。

1.  创建一个自定义调整配置文件，定义第一个驱动器的预读为*4096*，第二个驱动器的预读为*1024*。此配置文件还应在发生 OOM 事件时使系统崩溃。

1.  禁用并删除已安装的 HTTP 包。然后，使用*registry.redhat.io/rhel8/httpd-24*镜像设置 HTTP 服务器。

对于这一部分，我们将复制目标列表中的每个项目，然后在其下方提供解释，使用适当的语法突出显示和解释。

# 练习 1 解决方案

## 1. 将时区配置为 GMT

我们可以通过执行`date`命令来检查当前系统日期。在随后打印的行的最后部分，将显示时区。为了配置它，我们可以使用`timedatectl`命令，或者修改`/etc/localtime`符号链接。

因此，为了实现这个目标，我们可以使用以下之一：

+   `timedatectl set-timezone GMT`

+   `rm –fv /etc/localtime; ln –s /usr/share/zoneinfo/GMT /etc/localtime`

现在`date`应该报告正确的时区。

## 2. 允许无密码登录到 root 用户使用 SSH

这将需要以下操作：

+   SSH 必须已安装并可用（这意味着已安装并已启动）。

+   root 用户应该生成一个 SSH 密钥并将其添加到授权密钥列表中。

首先，让我们通过 SSH 来解决这个问题，如下所示：

```
dnf –y install openssh-server; systemctl enable sshd; systemctl start sshd
```

现在，让我们通过按*Enter*来生成一个 SSH 密钥以接受所有默认值：

```
ssh-keygen
```

现在，让我们将生成的密钥（`/root/.ssh/id_rsa`）添加到授权密钥中：

```
cd; cd .ssh; cat id_rsa.pub >> authorized_keys; chmod 600 authorized_keys
```

为了验证这一点，我们可以执行`ssh localhost date`，之后我们将能够在不提供密码的情况下获取当前系统的日期和时间。

## 3. 创建一个名为'user'的用户，可以在没有密码的情况下连接到该机器

这需要创建一个用户和一个类似于 root 用户的 SSH 密钥。接下来的选项也将与用户相关，但为了演示目的，我们将它们作为单独的任务来解决：

```
useradd user
su – user
```

现在，让我们通过按*Enter*来生成一个 SSH 密钥以接受所有默认值：

```
ssh-keygen
```

现在，让我们将生成的密钥（`/root/.ssh/id_rsa`）添加到授权密钥中：

```
cd; cd .ssh; cat id_rsa.pub >> authorized_keys; chmod 600 authorized_keys
```

为了验证这一点，我们可以执行`ssh localhost date`，我们将能够在不提供密码的情况下获取当前系统日期和时间。

然后，使用`logout`返回到我们的`root`用户。

## 4. 用户'user'应该每周更改密码，提前 2 天警告，过期后使用 1 天

这要求我们调整用户限制，如下所示：

```
chage –W 2 user
chage –I 1 user
chage -M 7 user
```

## 5. root 用户必须能够以'user'的身份通过 SSH 登录，而无需密码，以便没有人可以使用密码远程连接为 root 用户

这需要两个步骤。第一步是使用 root 的授权密钥启用'user'，然后调整`sshd`守护程序，如下所示：

```
cat /root/id_rsa.pub >> ~user/.ssh/authorized_keys
```

编辑`/etc/sshd/sshd_config`文件，并添加或替换`PermitRootLogin`行，使其看起来像下面这样：

```
PermitRootLogin prohibit-password
```

保存，然后重新启动`sshd`守护程序：

```
systemctl restart sshd
```

## 6. 用户'user'应能够成为 root 并执行命令而无需密码

这意味着通过添加以下行来配置`/etc/sudoers`文件：

```
user ALL=(ALL) NOPASSWD:ALL
```

## 7. 当用户尝试通过 SSH 登录时，显示有关不允许未经授权访问此系统的法律消息

创建一个文件，例如`/etc/ssh/banner`，其中包含要显示的消息。例如，`"Get out of here"`。

修改`/etc/ssh/sshd_config`并将`banner`行设置为`/etc/ssh/banner`，然后使用`systemctl restart sshd`重新启动`sshd`守护程序。

## 8. SSH 必须在端口 22222 上监听，而不是默认端口

这是一个棘手的问题。第一步是修改`/etc/ssh/sshd_config`并定义端口`22222`。完成后，使用以下命令重新启动`sshd`：

```
systemctl restart sshd
```

这当然会失败...为什么？

必须配置防火墙：

```
firewall-cmd –-add-port=22222/tcp --permanent
firewall-cmd –-add-port=22222/tcp 
```

然后必须配置 SELinux：

```
semanage port -a -t ssh_port_t -p tcp 22222
```

现在，`sshd`守护程序可以重新启动：

```
systemctl restart sshd
```

## 9. 创建名为'devel'的组

使用以下命令：

```
groupadd devel
```

## 10. 使'user'成为'devel'的成员

使用以下命令：

```
usermod –G devel user
```

## 11. 将用户成员身份存储在名为'userids'的文件中，在'用户'的主文件夹中

使用以下命令：

```
id user > ~user/userids
```

## 12. 用户'user'和 root 用户应能够通过 SSH 连接到本地主机，而无需指定端口，并默认为连接进行压缩

我们修改了默认的 SSH 端口为`22222`。

为'user'和 root 创建一个名为`.ssh/config`的文件，内容如下：

```
Host localhost
Port 22222
    Compression yes
```

## 13. 查找系统中所有 man 页面名称，并将名称放入名为'manpages.txt'的文件中

手册页存储在 `/usr/share/man`。因此，使用以下命令：

```
find  /usr/share/man/ -type f > manpages.txt
```

## 14\. 打印没有登录的用户的用户名，以便他们可以被允许访问系统，并打印每个用户的用户 ID 和组

以下命令首先构建了一个使用 `nologin` shell 的系统用户列表：

```
for user in $(cat /etc/passwd| grep nologin|cut -d ":" -f 1)
do
echo "$user -- $(grep $user /etc/group|cut -d ":" -f 1|xargs)"
done
```

从列表中检查 `/etc/group` 文件中的成员资格，仅保留组名，并使用 `xargs` 将它们连接成一个要打印的字符串。

上面的示例使用了 `for` 循环和命令的内联执行，通过 `$()`。

## 15\. 每 5 分钟监视可用的系统资源，而不使用 cron，并将它们存储为 /root/resources.log

监视某些东西的理想方式是使用 cron，但由于我们被告知不要使用它，这只留下了我们使用 systemd 定时器。 （您可以通过以下链接检查经过测试的文件：[`github.com/PacktPublishing/Red-Hat-Enterprise-Linux-8-Administration/tree/main/chapter-18-exercise1.`](https://github.com/PacktPublishing/Red-Hat-Enterprise-Linux-8-Administration/tree/main/chapter-18-exercise1

）

创建 `/etc/systemd/system/monitorresources.service`，内容如下：

```
[Unit]
Description=Monitor system resources

[Service]
Type=oneshot
ExecStart=/root/myresources.sh
```

创建 `/etc/systemd/system/monitorresources.timer`，内容如下：

```
[Unit]
Description=Monitor system resources

[Timer]
OnCalendar=*-*-* *:0,5,10,15,20,25,30,35,40,45,50,55:00
Persistent=true

[Install]
WantedBy=timers.target
```

创建 `/root/myresources.sh`，内容如下：

```
#!/bin/bash
df > /root/resources.log
```

启用新的定时器，如下所示：

```
systemctl daemon-reload
systemctl enable  monitorresources.timer
```

它有效吗？如果不行，`journalctl –f` 将提供一些细节。SELinux 阻止我们执行 root 文件，所以让我们将其转换为二进制类型并标记为可执行，如下所示：

```
chcon –t bin_t /root/myresources.sh
chmod +x /root/myresources.sh
```

## 16\. 添加每分钟的作业，报告可用的空闲磁盘空间百分比，并将其存储在 /root/freespace.log 中，以显示文件系统和可用空间

`df` 报告已使用的磁盘空间和可用空间，所以我们需要进行一些数学运算。

这将报告挂载位置、大小、已用空间和可用空间，使用 `;` 作为分隔符。参考以下内容：

```
df|awk '{print $6";"$2";"$3";"$4}'
```

Bash 允许我们进行一些数学运算，但这些运算缺少小数部分。幸运的是，我们可以做一个小技巧：我们将循环执行它，如下所示：

```
for each in $(df|awk '{print $6";"$2";"$3";"$4}'|grep -v "Mounted")
do 
    FREE=$(echo $each|cut -d ";" -f 4) 
    TOTAL=$(echo $each|cut -d ";" -f 2) 
    echo "$each has $((FREE*100/TOTAL)) free"
done
```

`for` 循环将检查所有可用的数据，抓取一些特定字段，用 `;` 分隔它们，然后对每一行在 `$each` 变量中运行循环。

我们截取输出，然后获取第四个字段。这是可用空间。

我们截取输出，然后获取第二个字段。这是总块数。

由于 `bash` 可以进行整数除法，我们可以乘以 100，然后除以获取百分比，并将一个字符串添加到输出中。

或者（但不够说明性），我们可以通过 `df` 已经给出的使用百分比减去 100，并节省一些计算步骤。

我们还需要将输出存储在一个文件中。为此，我们可以将整个循环包装在重定向中，或者将其添加到 `echo` 行中，以便将其附加到一个文件中。

我们还需要通过 cron 来完成，因此完整的解决方案如下：

创建一个 `/root/myfreespace.sh` 脚本，内容如下：

```
for each in $(df|awk '{print $6";"$2";"$3";"$4}'|grep -v "Mounted")
do 
    FREE=$(echo $each|cut -d ";" -f 4) 
    TOTAL=$(echo $each|cut -d ";" -f 2) 
    echo "$each has $((FREE*100/TOTAL)) free"
done
```

然后，使用 `chmod 755 /root/myfreespace.sh` 使其可执行。

运行 `crontab -e` 来编辑 root 的 crontab，并添加以下行：

```
*/1 * * * * /root/myfreespace.sh >> /root/freespace.log
```

## 17\. 配置系统只保留 3 天的日志

这可以通过编辑 `/etc/logrorate.conf` 来完成，设置如下：

```
daily
rotate 3
```

删除其他的每周、每月等出现，只留下我们想要的一个。

## 18\. 为 /root/freespace.log 和 /root/resources.log 配置日志轮转

创建一个 `/etc/logrotate.d/rotateroot` 文件，内容如下：

```
/root/freespace.log {
    missingok
    notifempty
    sharedscripts
    copytruncate
}
/root/resources.log {
    missingok
    notifempty
    sharedscripts
    copytruncate
}
```

## 19\. 针对 pool.ntp.org 进行快速同步的时间同步配置

编辑 `/etc/chrony.conf`，添加以下行：

```
pool pool.ntp.org iburst
```

然后运行以下命令：

```
systemctl restart chronyd
```

## 20\. 为子网 172.22.0.1/24 提供 NTP 服务器服务

编辑 `/etc/chrony.conf`，添加以下行：

```
Allow 172.22.0.1/24
```

然后运行以下命令：

```
systemctl restart chronyd
```

## 21\. 每分钟配置系统统计收集

运行以下命令：

```
dnf –y install sysstat
```

现在我们需要修改`/usr/lib/systemd/system/sysstat-collect.timer`。让我们通过创建一个覆盖来做到这一点，如下所示：

```
cp /usr/lib/systemd/system/sysstat-collect.timer /etc/systemd/system/
```

编辑`/etc/systemd/system/sysstat-collect.timer`，将`OnCalendar`值替换为以下内容：

```
OnCalendar=*:00/1
```

然后，使用以下命令重新加载单元：

```
systemctl daemon-reload
```

## 22\. 配置系统中用户密码长度为 12 个字符

使用以下行编辑`/etc/login.defs`：

```
PASS_MIN_LEN 12
```

## 23\. 创建一个名为'privacy'的机器人用户，它默认情况下只能自己看到它的文件

要做到这一点，请运行以下命令：

```
adduser privacy
su – privacy
echo "umask 0077" >> .bashrc
```

此解决方案使用`umask`从所有新创建的文件中删除其他人的权限。

## 24\. 创建一个名为/shared 的文件夹，所有用户都可以访问，并将新文件和目录默认设置为仍然可以被‘devel’组的用户访问

要做到这一点，请运行以下命令：

```
mkdir /shared
chown root:devel /shared
chmod 777 /shared
chmod +s /shared
```

## 25\. 使用提供的数据 Ip6，配置一个名为'mynic'的具有 IPv4 和 IPv6 地址的网络连接，如下所示：2001:db8:0:1::c000:207/64 g gateway 2001:db8:0:1::1 IPv4 192.0.1.3/24 gateway 192.0.1.1

查看以下内容以了解如何完成此操作：

```
nmcli con add con-name mynic type ethernet ifname eth0 ipv6.address 2001:db8:0:1::c000:207/64 ipv6.gateway 2001:db8:0:1::1 ipv4.address 192.0.1.3/24 ipv4.gateway 192.0.1.1
```

## 26\. 允许主机使用 google 主机名访问 www.google.com，使用 redhat 主机名访问 www.redhat.com

运行并记录获取的 IP，如下所示：

```
ping www.google.com
ping www.redhat.com 
```

记下上面获取的 IP。

通过添加以下内容编辑`/etc/hosts`：

```
IPFORGOOGLE google
IPFORREDHAT redhat
```

然后，保存并退出。

## 27\. 报告从供应商分发的文件中修改的文件，并将它们存储在`/root/altered.txt`中

查看以下内容以了解如何完成此操作：

```
rpm  -Va > /root/altered.txt
```

## 28\. 通过 HTTP 在路径`/mirror`下使我们的系统安装媒体包对其他系统可用，并在我们的系统中配置存储库。从该镜像中删除内核软件包，以便其他系统（甚至我们自己）无法找到新的内核。忽略此存储库中的 glibc 软件包，以便安装而不删除它们

这是一个复杂的问题，所以让我们一步一步地来看看。

安装`http`并使用以下命令启用它：

```
dnf –y install httpd
firewall-cmd  --add-service=http --permanent
firewall-cmd  --add-service=http 
systemctl start httpd
systemctl enable httpd
```

在`/mirror`下创建一个文件夹，然后复制源媒体包并通过`http`使其可用：

```
mkdir /mirror /var/www/html/mirror
mount /dev/cdrom /mnt
rsync –avr –progress /mnt/ /mirror/
mount –o bind /mirror /var/www/html/mirror
chcon  -R -t httpd_sys_content_t /var/www/html/mirror/
```

删除内核软件包：

```
find /mirror -name kernel* -exec rm '{}' \;
```

使用以下命令创建存储库文件元数据：

```
dnf –y install createrepo
cd /mirror
createrepo .
```

使用我们创建的存储库文件，并在系统上设置它，忽略其中的`glibc*`软件包。

通过添加以下内容编辑`/etc/yum.repos.d/mymirror.repo`：

```
[mymirror]
name=My RHEL8 Mirror
baseurl=http://localhost/mirror/
enabled=1
gpgcheck=0
exclude=glibc*
```

## 29\. 作为‘user’，将/root 文件夹复制到/home/user/root/文件夹中，并每天保持同步，同步添加和删除

查看以下内容以了解如何完成此操作：

```
su – user
crontab –e 
```

编辑 crontab 并添加以下行：

```
@daily rsync  -avr –-progress –-delete root@localhost:/root/ /home/user/root/
```

## 30\. 检查我们的系统是否符合 PCI-DSS 标准

```
dnf –y install openscap  scap-security-guide openscap-utils 
oscap xccdf eval --report pci-dss-report.html --profile pci-dss /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml
```

## 31\. 向系统添加一个 30GB 的第二硬盘，但只使用 15GB 将镜像移动到其中，并使用压缩和去重功能在启动时使其可用，并在`/mirror/mirror`下可用

这句话中的压缩和去重意味着 VDO。我们需要将当前的镜像移动到 VDO，并使我们以前的镜像移到那里。

如果我们有安装媒体，我们可以选择复制它并重复内核删除或转移。为此，首先让我们在新的硬盘（`sdb`）的分区中创建 VDO 卷：

```
fdisk /dev/sdb
n <enter>
p <enter>
1 <enter>
<enter>
+15G <enter>
w <enter>
q <enter>
```

这将从开始创建一个 15GB 的分区。让我们使用以下命令在其上创建一个 VDO 卷：

```
dnf –y install vdo kmod-kvdo
vdo create –n myvdo –device /dev/sdb --force
pvcreate /dev/mapper/myvdo
vgcreate myvdo /dev/mapper/myvdo
lvcreate –L 15G –n myvol myvdo
mkfs.xfs /dev/myvdo/myvol
# Let's umount cdrom if it was still mounted
umount /mnt
# Mount vdo under /mnt and copy files over
mount /dev/myvdo/myvol /mnt
rsync –avr –progress /mirror/ /mnt/mirror/
# Delete the original mirror once copy has finished 
rm –Rfv /mirror
umount /mnt
mount /dev/myvdo/myvol /mirror
```

此时，旧的镜像已复制到 VDO 卷上的`mirror`文件夹中。这在`/mirror`下挂载，因此它在`/mirror/mirror`下有原始镜像，如要求的。我们可能需要执行以下操作：

+   将`/mirror`绑定到`/var/www/html/mirror/`以使文件可用。

+   恢复 SELinux 上下文以允许`httpd`守护程序访问`/var/www/html/mirror/`中的文件。

调整我们创建的 repofile 以指向新路径。

## 32\. 配置文件系统报告至少 1,500GB 的大小，供我们的镜像使用

查看以下命令：

```
vdo growLogical --name=myvdo --vdoLogicalSize=1500G 
```

## 33. 在/mirror/mytailormirror 下创建镜像的第二个副本，并删除所有以 k*开头的软件包

请参考以下内容如何完成此操作：

```
rsync –avr –progress /mirror/mirror/ /mirror/mytailormirror/
find /mirror/mytailormirror/ -name "k*" -type f –exec rm '{}' \;
cd /mirror/mytailormirror/
createrepo . 
```

## 34. 在硬盘的剩余空间（15 GB）中创建一个新的卷，并用它来扩展根文件系统

请参考以下内容如何完成此操作：

```
fdisk /dev/sdb
n <enter>
p <enter>
<enter>
<enter>
w <enter>
q <enter>
pvcreate /dev/sdb2
# run vgscan to find out the volume name to use (avoid myvdo as is the VDO from above)
vgextend $MYROOTVG /dev/sdb2
# run lvscan to find out the LV storing the root filesystem and pvscan to find the maximum available space
lvresize –L +15G /dev/rhel/root
```

## 35. 创建一个引导项，允许我们进入紧急模式以更改根密码

请参考以下内容如何完成此操作：

```
grubby --args="systemd.unit=emergency.target" --update-kernel=/boot/vmlinuz-$(uname –r)
```

## 36. 创建一个自定义调整配置文件，定义第一个驱动器的预读取为 4096，第二个驱动器的预读取为 1024 - 此配置文件还应在发生 OOM 事件时使系统崩溃

参考以下命令：

```
dnf –y install tuned
mkdir –p /etc/tuned/myprofile
```

编辑`/etc/tuned/myprofile/tuned.conf`文件，添加以下内容：

```
[main]
summary=My custom tuned profile
[sysctl]
vm.panic_on_oom=1
[main_disk]
type=disk
devices=sda
readahead=>4096
[data_disk]
type=disk
devices=!sda
readahead=>1024
```

## 37. 禁用并删除已安装的 httpd 软件包，并使用 registry.redhat.io/rhel8/httpd-24 镜像设置 httpd 服务器

请参考以下内容如何完成此操作：

```
rpm –e httpd
dnf –y install podman
podman login registry.redhat.io # provide RHN credentials
podman pull registry.redhat.io/rhel8/httpd-24 
podman run -d --name httpd –p 80:8080 -v /var/www:/var/www:Z registry.redhat.io/rhel8/httpd-24
```
