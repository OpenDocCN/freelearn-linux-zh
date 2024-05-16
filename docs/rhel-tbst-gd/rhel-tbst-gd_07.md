# 第七章：文件系统错误和恢复

在第五章*网络故障排除*和第六章*诊断和纠正防火墙问题*中，我们使用了许多工具来排除由于错误配置的路由和防火墙导致的网络连接问题。网络相关问题非常常见，这两个示例问题也是常见的情况。在本章中，我们将专注于与硬件相关的问题，并从排除文件系统错误开始。

就像其他章节一样，我们将从发现的错误开始，排除问题直到找到原因和解决方案。在这个过程中，我们将发现许多用于排除文件系统问题的不同命令和日志。

# 诊断文件系统错误

与之前的章节不同，那时最终用户向我们报告了问题，这一次我们自己发现了问题。在数据库服务器上执行一些日常任务时，我们尝试创建数据库备份，并收到以下错误：

```
[db]# mysqldump wordpress > /data/backups/wordpress.sql
-bash: /data/backups/wordpress.sql: Read-only file system

```

这个错误很有趣，因为它不一定来自`mysqldump`命令，而是来自写入`/data/backups/wordpress.sql`文件的 bash 重定向。

如果我们看一下错误，它非常具体，我们试图将备份写入的文件系统是`只读`的。`只读`是什么意思？

## 只读文件系统

在 Linux 上定义和挂载文件系统时，你有很多选项，但有两个选项最能定义文件系统的可访问性。这两个选项分别是`rw`表示读写，**ro**表示只读。当文件系统以读写选项挂载时，这意味着文件系统的内容可以被读取，并且具有适当权限的用户可以向文件系统写入新文件/目录。

当文件系统以只读模式挂载时，这意味着用户可以读取文件系统，但新的写入请求将被拒绝。

## 使用`mount`命令列出已挂载的文件系统

由于我们收到的错误明确指出文件系统是只读的，我们下一个逻辑步骤是查看服务器上已挂载的文件系统。为此，我们将使用`mount`命令：

```
[db]# mount
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime,seclabel)
devtmpfs on /dev type devtmpfs (rw,nosuid,seclabel,size=228500k,nr_inodes=57125,mode=755)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,seclabel)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,seclabel,gid=5,mode=620,ptmxmode=000)
tmpfs on /run type tmpfs (rw,nosuid,nodev,seclabel,mode=755)
tmpfs on /sys/fs/cgroup type tmpfs (rw,nosuid,nodev,noexec,seclabel,mode=755)
selinuxfs on /sys/fs/selinux type selinuxfs (rw,relatime)
systemd-1 on /proc/sys/fs/binfmt_misc type autofs (rw,relatime,fd=33,pgrp=1,timeout=300,minproto=5,maxproto=5,direct)
mqueue on /dev/mqueue type mqueue (rw,relatime,seclabel)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,seclabel)
debugfs on /sys/kernel/debug type debugfs (rw,relatime)
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw,relatime)
nfsd on /proc/fs/nfsd type nfsd (rw,relatime)
/dev/sda1 on /boot type xfs (rw,relatime,seclabel,attr2,inode64,noquota)
192.168.33.13:/nfs on /data type nfs4 (rw,relatime,vers=4.0,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.33.12,local_lock=none,addr=192.168.33.13)

```

`mount`命令在处理文件系统时非常有用。它不仅可以用于显示已挂载的文件系统（如前面的命令所示），还可以用于附加（或挂载）和卸载文件系统。

### 已挂载的文件系统

称文件系统为已挂载的文件系统是一种常见的说法，表示文件系统已*连接*到服务器。对于文件系统，它们通常有两种状态，要么是已连接（已挂载），内容对用户可访问，要么是未连接（未挂载），对用户不可访问。在本章的后面，我们将使用`mount`命令来介绍挂载和卸载文件系统。

`mount`命令不是查看已挂载或未挂载文件系统的唯一方法。另一种方法是简单地读取`/proc/mounts`文件：

```
[db]# cat /proc/mounts 
rootfs / rootfs rw 0 0
proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
sysfs /sys sysfs rw,seclabel,nosuid,nodev,noexec,relatime 0 0
devtmpfs /dev devtmpfs rw,seclabel,nosuid,size=228500k,nr_inodes=57125,mode=755 0 0
securityfs /sys/kernel/security securityfs rw,nosuid,nodev,noexec,relatime 0 0
tmpfs /dev/shm tmpfs rw,seclabel,nosuid,nodev 0 0
devpts /dev/pts devpts rw,seclabel,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=000 0 0
tmpfs /run tmpfs rw,seclabel,nosuid,nodev,mode=755 0 0
tmpfs /sys/fs/cgroup tmpfs rw,seclabel,nosuid,nodev,noexec,mode=755 0 0
selinuxfs /sys/fs/selinux selinuxfs rw,relatime 0 0
systemd-1 /proc/sys/fs/binfmt_misc autofs rw,relatime,fd=33,pgrp=1,timeout=300,minproto=5,maxproto=5,direct 0 0
mqueue /dev/mqueue mqueue rw,seclabel,relatime 0 0
hugetlbfs /dev/hugepages hugetlbfs rw,seclabel,relatime 0 0
debugfs /sys/kernel/debug debugfs rw,relatime 0 0
sunrpc /var/lib/nfs/rpc_pipefs rpc_pipefs rw,relatime 0 0
nfsd /proc/fs/nfsd nfsd rw,relatime 0 0
/dev/sda1 /boot xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0
192.168.33.13:/nfs /data nfs4 rw,relatime,vers=4.0,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.33.12,local_lock=none,addr=192.168.33.13 0 0

```

实际上，`/proc/mounts`文件的内容与`mount`命令的输出非常接近，主要区别在于每行末尾的两个数字列。为了更好地理解这个文件和`mount`命令的输出，让我们更仔细地看一下`/proc/mounts`中`/boot`文件系统的条目：

```
/dev/sda1 /boot xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0

```

`/proc/mounts`文件有六列数据——**设备**、**挂载点**、**文件系统类型**、**选项**，以及两个未使用的列，用于向后兼容。为了更好地理解这些值，让我们更好地理解这些列。

第一列设备指定了用于文件系统的设备。在前面的例子中，/boot 文件系统所在的设备是/dev/sda1。

从设备的名称（sda1）可以识别出一个关键信息。这个设备是另一个设备的分区，我们可以通过设备名称末尾的数字来识别。

这个设备，从名称上看似乎是一个物理驱动器（假设是硬盘），名为/dev/sda；这个驱动器至少有一个分区，其设备名称为/dev/sda1。每当一个驱动器上有分区时，分区会被创建为自己的设备，每个设备都被分配一个编号；在这种情况下是 1，这意味着它是第一个分区。

### 使用 fdisk 列出可用分区

我们可以通过使用 fdisk 命令来验证这一点：

```
[db]# fdisk -l /dev/sda

Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0009c844

 Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048    83886079    41430016   8e  Linux LVM

```

fdisk 命令可能很熟悉，因为它是一个用于创建磁盘分区的跨平台命令。但它也可以用来列出分区。

在前面的命令中，我们使用了-l（列出）标志来列出分区，然后是我们想要查看的设备/dev/sda。然而，fdisk 命令显示的不仅仅是这个驱动器上可用的分区。它还显示了磁盘的大小：

```
Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors

```

我们可以从 fdisk 命令打印的第一行中看到这一点，根据这一行，我们的设备/dev/sda 的大小为 42.9GB。如果我们看输出的底部，还可以看到在这个磁盘上创建的分区：

```
 Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     1026047      512000   83  Linux
/dev/sda2         1026048    83886079    41430016   8e  Linux LVM

```

从前面的列表中，看起来/dev/sda 有两个分区，/dev/sda1 和/dev/sda2。使用 fdisk，我们已经能够识别出关于这个文件系统物理设备的许多细节。如果我们继续查看/proc/mounts 的详细信息，我们应该能够识别出一些其他非常有用的信息，如下所示：

```
/dev/sda1 /boot xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0

```

前一行中的第二列*挂载点*标注了这个文件系统挂载到的路径。在这种情况下，路径是/boot；/boot 本身只是根文件系统上的一个目录。然而，一旦存在于设备/dev/sda1 上的文件系统被挂载，/boot 现在就是它自己的文件系统。

为了更好地理解这个概念，我们将使用 mount 和 umount 命令来挂载和卸载/boot 文件系统：

```
[db]# ls /boot/
config-3.10.0-123.el7.x86_64
grub
grub2
initramfs-0-rescue-dee83c8c69394b688b9c2a55de9e29e4.img
initramfs-3.10.0-123.el7.x86_64.img
initramfs-3.10.0-123.el7.x86_64kdump.img
initrd-plymouth.img
symvers-3.10.0-123.el7.x86_64.gz
System.map-3.10.0-123.el7.x86_64
vmlinuz-0-rescue-dee83c8c69394b688b9c2a55de9e29e4
vmlinuz-3.10.0-123.el7.x86_64

```

如果我们在/boot 路径上执行一个简单的 ls 命令，我们可以看到这个目录中有很多文件。从/proc/mounts 文件和 mount 命令中，我们知道/boot 上有一个文件系统挂载：

```
[db]# mount | grep /boot
/dev/sda1 on /boot type xfs (rw,relatime,seclabel,attr2,inode64,noquota)

```

为了卸载这个文件系统，我们可以使用 umount 命令：

```
[db]# umount /boot
[db]# mount | grep /boot

```

umount 命令的任务非常简单，它卸载已挂载的文件系统。

### 提示

前面的命令是卸载文件系统可能是危险的示例。一般来说，您应该首先验证文件系统在卸载之前是否正在被访问。

/boot 文件系统现在已经卸载，当我们执行 ls 命令时会发生什么？

```
# ls /boot

```

/boot 路径仍然有效。但现在它只是一个空目录。这是因为/dev/sda1 上的文件系统没有挂载；因此，该文件系统上存在的任何文件目前在这个系统上都无法访问。

如果我们使用 mount 命令重新挂载文件系统，我们将看到文件重新出现：

```
[db]# mount /boot
[db]# ls /boot
config-3.10.0-123.el7.x86_64
grub
grub2
initramfs-0-rescue-dee83c8c69394b688b9c2a55de9e29e4.img
initramfs-3.10.0-123.el7.x86_64.img
initramfs-3.10.0-123.el7.x86_64kdump.img
initrd-plymouth.img
symvers-3.10.0-123.el7.x86_64.gz
System.map-3.10.0-123.el7.x86_64
vmlinuz-0-rescue-dee83c8c69394b688b9c2a55de9e29e4
vmlinuz-3.10.0-123.el7.x86_64

```

正如我们所看到的，当 mount 命令给出路径参数时，该命令将尝试挂载该文件系统。然而，当没有给出参数时，mount 命令将简单地显示当前挂载的文件系统。

在本章的后面，我们将探讨使用 mount 以及它如何理解文件系统应该在何处以及如何挂载；现在，让我们来看一下/proc/mounts 输出中的下一列：

```
/dev/sda1 /boot xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0

```

第三列文件系统类型表示正在使用的文件系统类型。在许多操作系统中，特别是 Linux，通常可以使用多种类型的文件系统。在上面的情况下，我们的引导文件系统设置为`xfs`，这是 Red Hat Enterprise Linux 7 的新默认文件系统。

在使用`xfs`之前，旧版本的 Red Hat 默认使用`ext3`或`ext4`文件系统。Red Hat 仍然支持`ext3/4`文件系统和其他文件系统，因此`/proc/mounts`文件中可能列出了许多不同的文件系统类型。

对于`/boot`文件系统，了解文件系统类型并不立即有用；然而，在我们深入研究这个问题时，可能需要知道如何查找底层文件系统的类型：

```
/dev/sda1 /boot xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0

```

第四列选项显示了文件系统挂载的选项。

当文件系统被挂载时，可以为该文件系统指定特定选项，以改变文件系统的默认行为。在上面的例子中，提供了相当多的选项；让我们分解这个列表，以更好地理解指定了什么：

+   `**inode64**`**：这使文件系统能够创建大于 32 位长度的索引节点号**

+   **noquota**：这禁用了对该文件系统的磁盘配额和强制执行**

**从描述中可以看出，这些选项可以极大地改变文件系统的行为。在排除任何文件系统问题时，查看这些选项也非常重要：**

```
**/dev/sda1 /boot xfs rw,seclabel,relatime,attr2,inode64,noquota 0 0** 
```

`/proc/mounts`输出的最后两列，表示为`0 0`，实际上在`/proc/mounts`中没有使用。这些列实际上只是为了与`/etc/mtab`向后兼容而添加的，`/etc/mtab`是一个类似的文件，但不像`/proc/mounts`那样被认为是最新的。

**这两个文件之间的区别在于它们的用途。`/etc/mtab`文件是为用户或应用程序设计的，用于读取和利用，而`/proc/mounts`文件是由内核本身使用的。因此，`/proc/mounts`文件被认为是最权威的版本。**

### **回到故障排除**

**如果我们回到手头的问题，我们在向`/data/backups`目录写入备份时收到了错误。使用`mount`命令，我们可以确定该目录存在于哪个文件系统上：**

```
**# mount | grep "data"**
**192.168.33.13:/nfs on /data type nfs4 (rw,relatime,vers=4.0,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.33.12,local_lock=none,addr=192.168.33.13)** 
```

**现在我们更好地理解了`mount`命令的格式，我们可以从上面的命令行中识别出一些关键信息。我们可以看到，此文件系统的设备设置为（`192.168.33.13:/nfs`），`mount`点（要附加的路径）设置为（`/data`），文件系统类型为（`nfs4`），并且文件系统设置了相当多的选项。**

**# NFS - 网络文件系统

查看`/data`文件系统，我们可以看到文件系统类型设置为`nfs4`。这种文件系统类型意味着文件系统是一个**网络文件系统**（**NFS**）。

NFS 是一种允许服务器与其他远程服务器共享导出目录的服务。`nfs4`文件系统类型是一种特殊的文件系统，允许远程服务器访问此服务，就像它是一个标准文件系统一样。

文件系统类型中的`4`表示要使用的版本，这意味着远程服务器要使用 NFS 协议的第 4 版。

### 提示

目前，NFS 最流行的版本是版本 3 和版本 4，版本 4 是 Red Hat Enterprise Linux 6 和 7 的默认版本。版本 3 和版本 4 之间有相当多的区别；然而，这些区别都不足以影响我们的故障排除方法。如果您在使用 NFS 版本 3 时遇到问题，那么您很可能可以按照我们将在本章中遵循的相同类型的步骤进行操作。

现在我们已经确定了文件系统是 NFS 文件系统，让我们看看它挂载的选项：

```
192.168.33.13:/nfs on /data type nfs4 (rw,relatime,vers=4.0,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.33.12,local_lock=none,addr=192.168.33.13)

```

从我们收到的错误来看，文件系统似乎是“只读”的，但如果我们查看列出的选项，第一个选项是`rw`。这意味着 NFS 文件系统本身已被挂载为“读写”，这应该允许对此文件系统进行写操作。

为了测试问题是与路径`/data/backups`还是挂载的文件系统`/data`有关，我们可以使用`touch`命令来测试在此文件系统中创建文件：

```
# touch /data/file.txt
touch: cannot touch '/data/file.txt': Read-only file system

```

甚至`touch`命令也无法在此文件系统上创建新文件。这清楚地表明文件系统存在问题；唯一的问题是是什么导致了这个问题。

如果我们查看此文件系统挂载的选项，没有任何导致文件系统为“只读”的原因；这意味着问题很可能不在于文件系统的挂载方式，而是其他地方。

由于问题似乎与 NFS 文件系统的挂载方式无关，而且这个文件系统是基于网络的，下一个有效的步骤将是验证与 NFS 服务器的网络连接。

## NFS 和网络连接

就像网络故障排除一样，我们的第一个测试将是 ping NFS 服务器，看看是否有响应；但问题是：*我们应该 ping 哪个服务器？*

答案在文件系统挂载的设备名称中（`192.168.33.13:/nfs`）。挂载 NFS 文件系统时，设备的格式为`<nfs 服务器>:<共享目录>`。在我们的示例中，这意味着我们的`/data`文件系统正在从服务器`192.168.33.13`挂载`/nfs`目录。为了测试连接性，我们可以简单地`ping` IP `192.168.33.13`：

```
[db]# ping 192.168.33.13
PING 192.168.33.13 (192.168.33.13) 56(84) bytes of data.
64 bytes from 192.168.33.13: icmp_seq=1 ttl=64 time=0.495 ms
64 bytes from 192.168.33.13: icmp_seq=2 ttl=64 time=0.372 ms
64 bytes from 192.168.33.13: icmp_seq=3 ttl=64 time=0.364 ms
64 bytes from 192.168.33.13: icmp_seq=4 ttl=64 time=0.337 ms
^C
--- 192.168.33.13 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
rtt min/avg/max/mdev = 0.337/0.392/0.495/0.060 ms

```

从`ping`结果来看，NFS 服务器似乎是正常的；但 NFS 服务呢？我们可以通过使用`curl`命令`telnet`到 NFS 端口来验证与 NFS 服务的连接。但首先，我们需要确定应连接到哪个端口。

在早期章节中排除数据库连接问题时，我们主要使用了众所周知的端口；由于 NFS 使用了几个不太常见的端口，我们需要确定要连接的端口：

这样做的最简单方法是在`/etc/services`文件中搜索端口：

```
[db]# grep nfs /etc/services 
nfs             2049/tcp        nfsd shilp      # Network File System
nfs             2049/udp        nfsd shilp      # Network File System
nfs             2049/sctp       nfsd shilp      # Network File System
netconfsoaphttp 832/tcp                 # NETCONF for SOAP over HTTPS
netconfsoaphttp 832/udp                 # NETCONF for SOAP over HTTPS
netconfsoapbeep 833/tcp                 # NETCONF for SOAP over BEEP
netconfsoapbeep 833/udp                 # NETCONF for SOAP over BEEP
nfsd-keepalive  1110/udp                # Client status info
picknfs         1598/tcp                # picknfs
picknfs         1598/udp                # picknfs
shiva_confsrvr  1651/tcp   shiva-confsrvr   # shiva_confsrvr
shiva_confsrvr  1651/udp   shiva-confsrvr   # shiva_confsrvr
3d-nfsd         2323/tcp                # 3d-nfsd
3d-nfsd         2323/udp                # 3d-nfsd
mediacntrlnfsd  2363/tcp                # Media Central NFSD
mediacntrlnfsd  2363/udp                # Media Central NFSD
winfs           5009/tcp                # Microsoft Windows Filesystem
winfs           5009/udp                # Microsoft Windows Filesystem
enfs            5233/tcp                # Etinnae Network File Service
nfsrdma         20049/tcp               # Network File System (NFS) over RDMA
nfsrdma         20049/udp               # Network File System (NFS) over RDMA
nfsrdma         20049/sctp              # Network File System (NFS) over RDMA

```

`/etc/services`文件是许多 Linux 发行版中包含的静态文件。它用作查找表，将网络端口映射到简单易读的名称。从前面的输出中，我们可以看到`nfs`名称映射到 TCP 端口`2049`；这是 NFS 服务的默认端口。我们可以利用这个端口来测试连接性，如下所示：

```
[db]# curl -vk telnet://192.168.33.13:2049
* About to connect() to 192.168.33.13 port 2049 (#0)
*   Trying 192.168.33.13...
* Connected to 192.168.33.13 (192.168.33.13) port 2049 (#0)

```

我们的`telnet`似乎成功了；我们可以进一步验证它，使用`netstat`命令：

```
[db]# netstat -na | grep 192.168.33.13
tcp        0      0 192.168.33.12:756       192.168.33.13:2049      ESTABLISHED

```

看起来连接性不是问题，如果我们的问题与连接性无关，也许是 NFS 共享的配置有问题。

我们实际上可以使用一个命令验证 NFS 共享的设置和网络连接性——`showmount`。

## 使用`showmount`命令

`showmount`命令可用于显示通过`-e`（显示导出）标志导出的目录。此命令通过查询指定主机上的 NFS 服务来工作。

对于我们的问题，我们将查询`192.168.33.13`上的 NFS 服务：

```
[db]# showmount -e 192.168.33.13
Export list for 192.168.33.13:
/nfs 192.168.33.0/24

```

`showmount`命令的格式使用两列。第一列是共享的目录。第二个是共享该目录的网络或主机名。

在前面的示例中，我们可以看到从此主机共享的目录是`/nfs`目录。这与设备名称`192.168.33.13:/nfs`中列出的目录相匹配。

`/nfs`目录正在共享的网络是`192.166.33.0/24`网络，正如我们在网络章节中学到的那样，它是`192.168.33.0`到`192.168.33.255`的缩写。我们已经知道从以前的故障排除中，我们所在的数据库服务器位于该网络中。

我们还可以看到自从之前执行`netstat`命令以来，这并没有改变：

```
[db]# netstat -na | grep 192.168.33.13
tcp        0      0 192.168.33.12:756       192.168.33.13:2049      ESTABLISHED

```

`netstat`命令的第四列显示了在`已建立`的 TCP 连接中使用的本地 IP 地址。根据前面的输出，我们可以看到`192.168.33.12`地址是我们的数据库服务器的 IP（在前几章中已经看到）。

到目前为止，关于这个 NFS 共享的一切看起来都是正确的，从这里开始，我们需要登录到 NFS 服务器继续故障排除。

## NFS 服务器配置

一旦登录到 NFS 服务器，我们应该首先检查 NFS 服务是否正在运行：

```
[db]# systemctl status nfs
nfs-server.service - NFS server and services
 Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled)
 Active: active (exited) since Sat 2015-04-25 14:01:13 MST; 17h ago
 Process: 2226 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
 Process: 2225 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 2226 (code=exited, status=0/SUCCESS)
 CGroup: /system.slice/nfs-server.service

```

使用`systemctl`，我们可以简单地查看服务状态；从前面的输出来看，这是正常的。这是可以预料的，因为我们能够`telnet`到 NFS 服务并使用`showmount`命令来查询它。

### 探索`/etc/exports`

由于 NFS 服务正在运行且正常，下一步是检查定义了哪些目录被导出以及它们如何被导出的配置；`/etc/exports`文件：

```
[nfs]# ls -la /etc/exports
-rw-r--r--. 1 root root 40 Apr 26 08:28 /etc/exports
[nfs]# cat /etc/exports
/nfs  192.168.33.0/24(rw,no_root_squash)

```

这个文件的格式实际上与`showmount`命令的输出类似。

第一列是要共享的目录，第二列是要与之共享的网络。然而，在这个文件中，在网络定义之后还有额外的信息。

网络/子网列后面跟着一组括号，里面包含各种`NFS`选项。这些选项与我们在`/proc/mounts`文件中看到的挂载选项非常相似。

这些选项可能是我们`只读`文件系统的根本原因吗？很可能。让我们分解这两个选项以更好地理解：

+   `rw`：这允许在共享目录上进行读取和写入

+   `no_root_squash`：这禁用了`root_squash`；`root_squash`是一个将 root 用户映射到匿名用户的系统

不幸的是，这两个选项都不能强制文件系统处于`只读`模式。事实上，根据这些选项的描述，它们似乎表明这个 NFS 共享应该处于`读写`模式。

在对`/etc/exports`文件执行`ls`时，出现了一个有趣的事实：

```
[nfs]# ls -la /etc/exports
-rw-r--r--. 1 root root 40 Apr 26 08:28 /etc/exports

```

`/etc/exports`文件最近已经被修改。我们的共享文件系统实际上是以`只读`方式共享的，但是最近有人改变了`/etc/exports`文件，将文件系统导出为`读写`方式。

这种情况是完全可能的，实际上，这是 NFS 的一个常见问题。NFS 服务并不会不断地读取`/etc/exports`文件以寻找更改。事实上，只有在服务启动时才会读取这个文件。

对`/etc/exports`文件的任何更改都不会生效，直到重新加载服务或使用`exportfs`命令刷新导出的文件系统为止。

### 识别当前的导出

一个非常常见的情况是，有人对这个文件进行了更改，然后忘记运行命令来刷新导出的文件系统。我们可以使用`exportfs`命令来确定是否是这种情况：

```
[nfs]# exportfs -s
/nfs  192.168.33.0/24(rw,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)

```

当给出`-s`（显示当前导出）标志时，`exportfs`命令将简单地列出现有的共享目录，包括目录共享的选项。

从前面的输出可以看出，这个文件系统与许多未在`/etc/exports`中列出的选项共享。这是因为通过 NFS 共享的所有目录都有一个默认的选项列表，用于管理目录的共享方式。在`/etc/exports`中指定的选项实际上是用来覆盖默认设置的。

为了更好地理解这些选项，让我们分解它们：

+   `rw`：这允许在共享目录上进行读取和写入。

+   `wdelay`：这会导致 NFS 在怀疑另一个客户端正在进行写入时暂停写入请求。这旨在减少多个客户端连接时的写入冲突。

+   `no_root_squash`：这禁用了`root_squash`，它是一个将 root 用户映射到匿名用户的系统。

+   `no_subtree_check`：这禁用了`subtree`检查；子树检查实质上确保对导出子目录的目录的请求将遵守子目录更严格的策略。

+   `sec=sys`：这告诉 NFS 使用用户 ID 和组 ID 值来控制文件访问的权限和授权。

+   `secure`：这确保 NFS 只接受客户端端口低于 1024 的请求，实质上要求它来自特权 NFS 挂载。

+   `no_all_squash`：这禁用了`all_squash`，用于强制将所有权限映射到匿名用户和组。

似乎这些选项也没有解释“只读”文件系统。这似乎是一个非常棘手的故障排除问题，特别是当 NFS 服务似乎配置正确时。

### 从另一个客户端测试 NFS

由于 NFS 服务器的配置似乎正确，客户端（数据库服务器）也似乎正确，我们需要缩小问题是在客户端还是服务器端。

我们可以通过在另一个客户端上挂载文件系统并尝试相同的写入请求来做到这一点。根据配置，似乎我们只需要另一个服务器在`192.168.33.0/24`网络中执行此测试。也许我们之前章节中的博客服务器是一个好的客户端选择？

### 提示

在某些环境中，对这个问题的答案可能是否定的，因为 Web 服务器通常被认为比数据库服务器不太安全。但是，由于这只是本书的一个测试环境，所以可以接受。

一旦我们登录到博客服务器，我们可以测试是否可以使用`showmount`命令看到挂载：

```
[blog]# showmount -e 192.168.33.13
Export list for 192.168.33.13:
/nfs 192.168.33.0/24

```

这回答了两个问题。第一个是 NFS 客户端软件是否安装；由于`showmount`命令存在，答案很可能是“是”。

第二个问题是 NFS 服务是否可以从博客服务器访问，这也似乎是肯定的。

为了测试挂载，我们将简单地使用`mount`命令：

```
[blog]# mount -t nfs 192.168.33.13:/nfs /mnt

```

要使用`mount`命令挂载文件系统，语法是：`mount -t <文件系统类型> <设备> <挂载点>`。在上面的示例中，我们只是将`192.168.33.13:/nfs`设备挂载到了`/mnt`目录，文件系统类型为`nfs`。

在运行命令时，我们没有收到任何错误，但为了确保文件系统被正确挂载，我们可以使用`mount`命令，就像我们之前做的那样：

```
[blog]# mount | grep /mnt
192.168.33.13:/nfs on /mnt type nfs4 (rw,relatime,vers=4.0,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.33.11,local_lock=none,addr=192.168.33.13)

```

从`mount`命令的输出中，似乎`mount`请求成功，并且处于“读写”模式，这意味着`mount`选项类似于数据库服务器上使用的选项。

现在我们可以尝试使用`touch`命令在文件系统中创建文件来测试文件系统：

```
# touch /mnt/testfile.txt 
touch: cannot touch '/mnt/testfile.txt': Read-only file system

```

看起来问题不在客户端的配置上，因为即使我们的新客户端也无法写入这个文件系统。

### 提示

作为提示，在前面的示例中，我将`/nfs`共享挂载到了`/mnt`。`/mnt`目录被用作通用挂载点，通常被认为是可以使用的。但是，最好的做法是在挂载到`/mnt`之前确保没有其他东西挂载到`/mnt`。

# 使挂载永久化

当前，即使我们使用`mount`命令挂载了 NFS 共享，这个挂载的文件系统并不被认为是持久的。下次系统重新启动时，NFS 挂载将不会重新挂载。

这是因为在系统启动时，启动过程的一部分是读取`/etc/fstab`文件并`mount`其中定义的任何文件系统。

为了更好地理解这是如何工作的，让我们看一下数据库服务器上的`/etc/fstab`文件：

```
[db]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Mon Jul 21 23:35:56 2014
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/os-root /                       xfs     defaults        1 1
UUID=be76ec1d-686d-44a0-9411-b36931ee239b /boot                   xfs     defaults        1 2
/dev/mapper/os-swap swap                    swap    defaults        0 0
192.168.33.13:/nfs  /data      nfs  defaults  0 0

```

`/etc/fstab`文件的内容实际上与`/proc/mounts`文件的内容非常相似。`/etc/fstab`文件中的第一列用于指定要挂载的设备，第二列是要挂载到的路径或挂载点，第三列只是文件系统类型，第四列是`mount`文件系统的选项。

然而，这些文件在`/etc/fstab`文件中的最后两列是不同的。这最后两列实际上是有意义的。在`fstab`文件中，第五列由`dump`命令使用。

`dump`命令是一个简单的备份实用程序，它读取`/etc/fstab`以确定要备份的文件系统。当执行 dump 实用程序时，任何值设置为`0`的文件系统都不会被备份。

尽管这个实用程序在今天并不经常使用，但`/etc/fstab`文件中的这一列是为了向后兼容而保留的。

`/etc/fstab`文件中的第六列对今天的系统非常重要。此列用于表示在引导过程中执行文件系统检查或`fsck`的顺序（通常在故障后）。

文件系统检查或`fsck`是一个定期运行的过程，检查文件系统中的错误并尝试纠正它们。这是我们将在本章稍后介绍的一个过程。

## 卸载/mnt 文件系统

由于我们不希望 NFS 共享的文件系统保持挂载在博客服务器的`/mnt`路径上，我们需要卸载文件系统。

我们可以像之前对`/boot`文件系统所做的那样，使用`umount`命令来执行此操作：

```
[blog]# umount /mnt
[blog]# mount | grep /mnt

```

从博客服务器上，我们只需使用`umount`，然后是客户端的`/mnt`挂载点来`卸载`NFS`挂载`。现在我们已经这样做了，我们可以回到 NFS 服务器继续排除故障。

# 再次排除 NFS 服务器故障

由于我们确定即使新客户端也无法写入`/nfs`共享，我们现在已经缩小了问题很可能是在服务器端而不是客户端。

早些时候，在排除 NFS 服务器故障时，我们几乎检查了关于 NFS 的所有内容。我们验证了服务实际上正在运行，可以被客户端访问，`/etc/exports`中的数据是正确的，并且当前导出的目录与`/etc/exports`中的内容匹配。此时，只剩下一个地方需要检查：`日志`文件。

默认情况下，NFS 服务没有像 Apache 或 MariaDB 那样拥有自己的日志文件。相反，RHEL 系统上的此服务利用`syslog`设施；这意味着我们的日志将在`/var/log/messages`中。

`messages`日志是基于 Red Hat Enterprise Linux 的 Linux 发行版中非常常用的日志文件。实际上，默认情况下，除了 cron 作业和身份验证之外，RHEL 系统上的每条高于 info 日志级别的 syslog 消息都会发送到`/var/log/messages`。

由于 NFS 服务将其日志消息发送到本地`syslog`服务，因此其消息也包含在`messages`日志中。

## 查找 NFS 日志消息

如果我们不知道 NFS 日志被发送到`/var/log/messages`日志文件中怎么办？有一个非常简单的技巧来确定哪个日志文件包含 NFS 日志消息。

通常，在 Linux 系统上，所有系统服务的日志文件都位于`/var/log`中。由于我们知道系统上大多数日志的默认位置，我们可以简单地浏览这些文件，以确定哪些文件可能包含 NFS 日志消息：

```
[nfs]# cd /var/log
[nfs]# grep -rc nfs ./*
./anaconda/anaconda.log:14
./anaconda/syslog:44
./anaconda/anaconda.xlog:0
./anaconda/anaconda.program.log:7
./anaconda/anaconda.packaging.log:16
./anaconda/anaconda.storage.log:56
./anaconda/anaconda.ifcfg.log:0
./anaconda/ks-script-Sr69bV.log:0
./anaconda/ks-script-lfU6U2.log:0
./audit/audit.log:60
./boot.log:4
./btmp:0
./cron:470
./cron-20150420:662
./dmesg:26
./dmesg.old:26
./grubby:0
./lastlog:0
./maillog:112386
./maillog-20150420:17
./messages:3253
./messages-20150420:11804
./sa/sa15:1
./sa/sar15:1
./sa/sa16:1
./sa/sar16:1
./sa/sa17:1
./sa/sa19:1
./sa/sar19:1
./sa/sa20:1
./sa/sa25:1
./sa/sa26:1
./secure:14
./secure-20150420:63
./spooler:0
./tallylog:0
./tuned/tuned.log:0
./wtmp:0
./yum.log:0

```

`grep`命令递归（`-r`）搜索每个文件中的字符串"`nfs`"，并输出包含字符串的行数的文件名及计数（`-c`）。

在前面的输出中，有两个日志文件包含了最多数量的字符串"`nfs`"。第一个是`maillog`，这是用于电子邮件消息的系统日志；这不太可能与 NFS 服务相关。

第二个是`messages`日志文件，正如我们所知，这是系统默认的日志文件。

即使没有关于特定系统日志记录方法的先验知识，如果您对 Linux 有一般了解，并且熟悉前面的示例中的技巧，通常可以找到包含所需数据的日志。

既然我们知道要查找的日志文件，让我们浏览一下`/var/log/messages`日志。

## 阅读`/var/log/messages`

由于这个`log`文件可能相当大，我们将使用`tail`命令和`-100`标志，这会导致`tail`只显示指定文件的最后`100`行。通过将输出限制为`100`行，我们应该只看到最相关的数据：

```
[nfs]# tail -100 /var/log/messages
Apr 26 10:25:44 nfs kernel: md/raid1:md127: Disk failure on sdb1, disabling device.
md/raid1:md127: Operation continuing on 1 devices.
Apr 26 10:25:55 nfs kernel: md: unbind<sdb1>
Apr 26 10:25:55 nfs kernel: md: export_rdev(sdb1)
Apr 26 10:27:20 nfs kernel: md: bind<sdb1>
Apr 26 10:27:20 nfs kernel: md: recovery of RAID array md127
Apr 26 10:27:20 nfs kernel: md: minimum _guaranteed_  speed: 1000 KB/sec/disk.
Apr 26 10:27:20 nfs kernel: md: using maximum available idle IO bandwidth (but not more than 200000 KB/sec) for recovery.
Apr 26 10:27:20 nfs kernel: md: using 128k window, over a total of 511936k.
Apr 26 10:27:20 nfs kernel: md: md127: recovery done.
Apr 26 10:27:41 nfs nfsdcltrack[4373]: sqlite_remove_client: unexpected return code from delete: 8
Apr 26 10:27:59 nfs nfsdcltrack[4375]: sqlite_remove_client: unexpected return code from delete: 8
Apr 26 10:55:06 nfs dhclient[3528]: can't create /var/lib/NetworkManager/dhclient-05be239d-0ec7-4f2e-a68d-b64eec03fcb2-enp0s3.lease: Read-only file system
Apr 26 11:03:43 nfs chronyd[744]: Could not open temporary driftfile /var/lib/chrony/drift.tmp for writing
Apr 26 11:55:03 nfs rpc.mountd[4552]: could not open /var/lib/nfs/.xtab.lock for locking: errno 30 (Read-only file system)
Apr 26 11:55:03 nfs rpc.mountd[4552]: can't lock /var/lib/nfs/xtab for writing

```

即使`100`行也可能相当繁琐，我已将输出截断为只包含相关行。这显示了相当多带有字符串"`nfs`"的消息；然而，并非所有这些消息都来自 NFS 服务。由于我们的 NFS 服务器主机名设置为`nfs`，因此来自该系统的每个日志条目都包含字符串"`nfs`"。

然而，即使如此，我们仍然看到了一些与`NFS`服务相关的消息，特别是以下行：

```
Apr 26 10:27:41 nfs nfsdcltrack[4373]: sqlite_remove_client: unexpected return code from delete: 8
Apr 26 10:27:59 nfs nfsdcltrack[4375]: sqlite_remove_client: unexpected return code from delete: 8
Apr 26 11:55:03 nfs rpc.mountd[4552]: could not open /var/lib/nfs/.xtab.lock for locking: errno 30 (Read-only file system)
Apr 26 11:55:03 nfs rpc.mountd[4552]: can't lock /var/lib/nfs/xtab for writing

```

这些日志条目的有趣之处在于其中一个明确指出服务`rpc.mountd`由于文件系统为`只读`而无法打开文件。然而，它试图打开的文件`/var/lib/nfs/.xtab.lock`并不是我们 NFS 共享的一部分。

由于这个文件系统不是我们 NFS 的一部分，让我们快速查看一下这台服务器上挂载的文件系统。我们可以再次使用`mount`命令来做到这一点：

```
[nfs]# mount
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
sysfs on /sys type sysfs (rw,nosuid,nodev,noexec,relatime,seclabel)
devtmpfs on /dev type devtmpfs (rw,nosuid,seclabel,size=241112k,nr_inodes=60278,mode=755)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
selinuxfs on /sys/fs/selinux type selinuxfs (rw,relatime)
systemd-1 on /proc/sys/fs/binfmt_misc type autofs (rw,relatime,fd=33,pgrp=1,timeout=300,minproto=5,maxproto=5,direct)
mqueue on /dev/mqueue type mqueue (rw,relatime,seclabel)
debugfs on /sys/kernel/debug type debugfs (rw,relatime)
hugetlbfs on /dev/hugepages type hugetlbfs (rw,relatime,seclabel)
sunrpc on /var/lib/nfs/rpc_pipefs type rpc_pipefs (rw,relatime)
nfsd on /proc/fs/nfsd type nfsd (rw,relatime)
/dev/mapper/md0-root on / type xfs (ro,relatime,seclabel,attr2,inode64,noquota)
/dev/md127 on /boot type xfs (ro,relatime,seclabel,attr2,inode64,noquota)
/dev/mapper/md0-nfs on /nfs type xfs (ro,relatime,seclabel,attr2,inode64,noquota)

```

与另一台服务器一样，有相当多的挂载文件系统，但我们并不对所有这些感兴趣；只对其中的一小部分感兴趣。

```
/dev/mapper/md0-root on / type xfs (ro,relatime,seclabel,attr2,inode64,noquota)
/dev/md127 on /boot type xfs (ro,relatime,seclabel,attr2,inode64,noquota)
/dev/mapper/md0-nfs on /nfs type xfs (ro,relatime,seclabel,attr2,inode64,noquota)

```

前面的三行是我们应该感兴趣的行。这三个挂载的文件系统是我们系统定义的持久文件系统。如果我们查看这三个持久文件系统，我们可以找到一些有趣的信息。

`/`或根文件系统存在于设备`/dev/mapper/md0-root`上。这个文件系统对我们的系统非常重要，因为看起来这台服务器配置为在根文件系统(`/`)下安装整个操作系统，这是一种相当常见的设置。这个文件系统包括了问题文件`/var/lib/nfs/.xtab.lock`。

`/boot`文件系统存在于设备`/dev/md127`上，根据名称判断，这很可能是使用 Linux 软件 RAID 系统的阵列设备。`/boot`文件系统和根文件系统一样重要，因为`/boot`包含了服务器启动所需的所有文件。没有`/boot`文件系统，这个系统很可能无法重新启动，并且在下一次系统重启时会发生内核崩溃。

最后一个文件系统`/nfs`使用了`/dev/mapper/md0-nfs`设备。根据我们之前的故障排除，我们确定了这个文件系统是通过 NFS 服务导出的文件系统。

## 只读文件系统

如果我们回顾错误和`mount`的输出，我们将开始在这个系统上识别一些有趣的错误：

```
Apr 26 11:55:03 nfs rpc.mountd[4552]: could not open /var/lib/nfs/.xtab.lock for locking: errno 30 (Read-only file system)

```

错误报告称，`.xtab.lock`文件所在的文件系统是`只读`的：

```
/dev/mapper/md0-root on / type xfs (ro,relatime,seclabel,attr2,inode64,noquota)

```

从`mount`命令中，我们可以看到问题的文件系统是`/`文件系统。在查看`/`或根文件系统的选项后，我们可以看到这个文件系统实际上是使用`ro`选项挂载的。

实际上，如果我们查看这三个文件系统的选项，我们会发现`/`、`/boot`和`/nfs`都是使用`ro`选项挂载的。`rw`挂载文件系统为`读写`，`ro`选项挂载文件系统为`只读`。这意味着目前这些文件系统不能被任何用户写入。

所有三个定义的文件系统都以`只读`模式挂载是相当不寻常的配置。为了确定这是否是期望的配置，我们可以检查`/etc/fstab`文件，这是之前用来识别持久文件系统的同一个文件：

```
[nfs]# cat /etc/fstab
#
# /etc/fstab
# Created by anaconda on Wed Apr 15 09:39:23 2015
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/md0-root    /                       xfs     defaults        0 0
UUID=7873e886-78d5-46cc-b4d9-0c385995d915 /boot                   xfs     defaults        0 0
/dev/mapper/md0-nfs     /nfs                    xfs     defaults        0 0
/dev/mapper/md0-swap    swap                    swap    defaults        0 0

```

从`/etc/fstab`文件的内容来看，这些文件系统并没有配置为以`只读`模式挂载。相反，这些文件系统是以“默认”选项挂载的。

在 Linux 上，`xfs`文件系统的“默认”选项将文件系统挂载为“读写”模式，而不是“只读”模式。如果我们查看数据库服务器上的`/etc/fstab`文件，我们可以验证这种行为：

```
[db]# cat /etc/fstab 
#
# /etc/fstab
# Created by anaconda on Mon Jul 21 23:35:56 2014
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/os-root /                       xfs     defaults        1 1
UUID=be76ec1d-686d-44a0-9411-b36931ee239b /boot                   xfs     defaults        1 2
/dev/mapper/os-swap swap                    swap    defaults        0 0
192.168.33.13:/nfs  /data      nfs  defaults  0 0

```

在数据库服务器上，我们可以看到`/`或根文件系统的文件系统选项也设置为“默认”。然而，当我们使用`mount`命令查看文件系统选项时，我们可以看到`rw`选项以及一些其他默认选项被应用：

```
[db]# mount | grep root
/dev/mapper/os-root on / type xfs (rw,relatime,seclabel,attr2,inode64,noquota)

```

这证实了三个持久文件系统的“只读”状态不是期望的配置。

### 识别磁盘问题

如果`/etc/fstab`文件系统被特别配置为以“读写”方式挂载文件系统，并且`mount`命令显示文件系统以“只读”模式挂载。这清楚地表明所涉及的文件系统可能在引导过程的一部分挂载后被重新挂载。

正如我们之前讨论的，当 Linux 系统引导时，它会读取`/etc/fstab`文件并挂载所有定义的文件系统。但是，挂载文件系统的过程就此停止。默认情况下，没有持续监视`/etc/fstab`文件进行更改并挂载或卸载修改后的文件系统的过程。

实际上，看到新创建的文件系统未挂载但在`/etc/fstab`文件中指定是很常见的，因为有人在编辑`/etc/fstab`文件后忘记使用`mount`命令将其挂载。

然而，很少见到文件系统被挂载为“只读”，但之后`fstab`被更改。

实际上，对于我们的情况来说，这并不容易实现，因为`/etc/fstab`是不可访问的，因为`/`文件系统是“只读”的：

```
[nfs]# touch /etc/fstab
touch: cannot touch '/etc/fstab': Read-only file system

```

这意味着我们的文件系统处于“只读”模式，是在这些文件系统最初被挂载后执行的。

实际上，导致这种状态的罪魁祸首实际上是我们之前浏览的日志消息：

```
Apr 26 10:25:44 nfs kernel: md/raid1:md127: Disk failure on sdb1, disabling device.
md/raid1:md127: Operation continuing on 1 devices.
Apr 26 10:25:55 nfs kernel: md: unbind<sdb1>
Apr 26 10:25:55 nfs kernel: md: export_rdev(sdb1)
Apr 26 10:27:20 nfs kernel: md: bind<sdb1>
Apr 26 10:27:20 nfs kernel: md: recovery of RAID array md127
Apr 26 10:27:20 nfs kernel: md: minimum _guaranteed_  speed: 1000 KB/sec/disk.
Apr 26 10:27:20 nfs kernel: md: using maximum available idle IO bandwidth (but not more than 200000 KB/sec) for recovery.
Apr 26 10:27:20 nfs kernel: md: using 128k window, over a total of 511936k.
Apr 26 10:27:20 nfs kernel: md: md127: recovery done.

```

从`/var/log/messages`日志文件中，我们实际上可以看到在某个时候，软件 RAID（`md`）存在问题，标记磁盘`/dev/sdb1`为失败。

在 Linux 中，默认情况下，如果物理磁盘驱动器失败或以其他方式对内核不可用，Linux 内核将以“只读”模式重新挂载驻留在该物理磁盘上的文件系统。正如前面的错误消息中所述，`sdb1`物理磁盘和`md127` RAID 设备的故障似乎是文件系统变为“只读”的根本原因。

由于软件 RAID 和硬件问题是下一章的主题，我们将推迟故障排除 RAID 和磁盘问题至第八章，“硬件故障排除”。

# 恢复文件系统

现在我们知道文件系统为何处于“只读”模式，我们可以解决它。将文件系统从“只读”模式强制转换为“读写”模式实际上非常容易。但是，由于我们不知道导致文件系统进入“只读”模式的故障的所有情况，我们必须小心谨慎。

从文件系统错误中恢复可能非常棘手；如果操作不当，我们很容易陷入破坏文件系统或以其他方式导致部分甚至完全数据丢失的情况。

由于我们有多个文件系统处于“只读”模式，我们将首先从`/boot`文件系统开始。我们之所以从`/boot`文件系统开始，是因为这从技术上讲是最好的文件系统来体验数据丢失。由于`/boot`文件系统仅在服务器引导过程中使用，我们可以确保在`/boot`文件系统恢复之前不重新启动此服务器。

在可能的情况下，最好在采取任何行动之前备份数据。在接下来的步骤中，我们将假设`/boot`文件系统定期备份。

## 卸载文件系统

为了恢复这个文件系统，我们将执行三个步骤。在第一步中，我们将卸载`/boot`文件系统。在采取任何其他步骤之前卸载文件系统，我们将确保文件系统不会被主动写入。这一步将大大减少在恢复过程中文件系统损坏的机会。

但是，在卸载文件系统之前，我们需要确保没有应用程序或服务正在尝试写入我们正在尝试恢复的文件系统。

为了确保这一点，我们可以使用`lsof`命令。 `lsof`命令用于列出打开的文件；我们可以浏览此列表，以确定`/boot`文件系统中是否有任何文件是打开的。

如果我们只是运行没有选项的`lsof`，它将打印所有当前打开的文件：

```
[nfs]# lsof
COMMAND    PID TID           USER   FD      TYPE             DEVICE SIZE/OFF       NODE NAME
systemd      1               root  cwd       DIR              253,1 4096        128 /

```

通过向`lsof`添加“-r”（重复）标志，我们告诉它以重复模式运行。然后我们可以将此输出传输到`grep`命令，其中我们可以过滤在`/boot`文件系统上打开的文件的输出：

```
[nfs]# lsof -r | grep /boot

```

如果前面的命令一段时间内没有产生任何输出，可以安全地继续卸载文件系统。如果命令打印出任何打开的文件，最好找到适当的进程读取/写入文件系统并在卸载文件系统之前停止它们。

由于我们的例子在`/boot`文件系统上没有打开的文件，我们可以继续卸载`/boot`文件系统。为此，我们将使用`umount`命令：

```
[nfs]# umount /boot

```

幸运的是，`umount`命令没有出现错误。如果文件正在被写入，我们在卸载时可能会收到错误。通常，此错误包括一条消息，指出**设备正忙**。为了验证文件系统已成功卸载，我们可以再次使用`mount`命令：

```
[nfs]# mount | grep /boot

```

现在`/boot`文件系统已经卸载，我们可以执行我们恢复过程的第二步。我们现在可以检查和修复文件系统。

## 文件系统检查与 fsck

Linux 有一个非常有用的文件系统检查命令，可以用来检查和修复文件系统。这个命令叫做`fsck`。

然而，`fsck`命令实际上并不只是一个命令。每种文件系统类型都有其自己的检查一致性和修复问题的方法。 `fsck`命令只是一个调用相应文件系统的适当命令的包装器。

例如，当对`ext4`文件系统运行`fsck`命令时，实际执行的命令是`e2fsck`。 `e2fsck`命令用于`ext2`到`ext4`文件系统类型。

我们可以以两种方式调用`e2fsck`，直接或间接通过`fsck`命令。在这个例子中，我们将使用`fsck`方法，因为这可以用于 Linux 支持的几乎所有文件系统。

要使用`fsck`命令简单地检查文件系统的一致性，我们可以不带标志运行它，并指定要检查的磁盘设备：

```
[nfs]# fsck /dev/sda1
fsck from util-linux 2.20.1
e2fsck 1.42.9 (4-Feb-2014)
cloudimg-rootfs: clean, 85858/2621440 files, 1976768/10485504 blocks

```

在前面的例子中，我们可以看到文件系统没有发现任何错误。如果有的话，我们会被问及是否希望`e2fsck`实用程序来纠正这些错误。

如果我们愿意，我们可以通过传递“-y”（是）标志使`fsck`自动修复发现的问题：

```
[nfs]# fsck -y /dev/sda1
fsck from util-linux 2.20.1
e2fsck 1.42 (29-Nov-2011)
/dev/sda1 contains a file system with errors, check forced.
Pass 1: Checking inodes, blocks, and sizes
Inode 2051351 is a unknown file type with mode 0137642 but it looks 
like it is really a directory.
Fix? yes

Pass 2: Checking directory structure
Entry 'test' in / (2) has deleted/unused inode 49159\.  Clear? yes

Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information

/dev/sda1: ***** FILE SYSTEM WAS MODIFIED *****
/dev/sda1: 96/2240224 files (7.3% non-contiguous), 3793508/4476416 blocks

```

此时，`e2fsck`命令将尝试纠正它发现的任何错误。幸运的是，从我们的例子中，错误能够被纠正；然而，也有时候情况并非如此。

### fsck 和 xfs 文件系统

当对`xfs`文件系统运行`fsck`命令时，结果实际上是完全不同的：

```
[nfs]# fsck /dev/md127 
fsck from util-linux 2.23.2
If you wish to check the consistency of an XFS filesystem or
repair a damaged filesystem, see xfs_repair(8).

```

`xfs`文件系统不同于`ext2/3/4`文件系统系列，因为每次挂载文件系统时都会执行一致性检查。这并不意味着您不能手动检查和修复文件系统。要检查`xfs`文件系统，我们可以使用`xfs_repair`实用程序：

```
[nfs]# xfs_repair -n /dev/md127
Phase 1 - find and verify superblock...
Phase 2 - using internal log
 - scan filesystem freespace and inode maps...
 - found root inode chunk
Phase 3 - for each AG...
 - scan (but don't clear) agi unlinked lists...
 - process known inodes and perform inode discovery...
 - agno = 0
 - agno = 1
 - agno = 2
 - agno = 3
 - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
 - setting up duplicate extent list...
 - check for inodes claiming duplicate blocks...
 - agno = 0
 - agno = 1
 - agno = 2
 - agno = 3
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
 - traversing filesystem ...
 - traversal finished ...
 - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.

```

使用`-n`（不修改）标志后跟要检查的设备执行`xfs_repair`实用程序时，它只会验证文件系统的一致性。在这种模式下运行时，它根本不会尝试修复文件系统。

要以修复文件系统的模式运行`xfs_repair`，只需省略`-n`标志，如下所示：

```
[nfs]# xfs_repair /dev/md127
Phase 1 - find and verify superblock...
Phase 2 - using internal log
 - zero log...
 - scan filesystem freespace and inode maps...
 - found root inode chunk
Phase 3 - for each AG...
 - scan and clear agi unlinked lists...
 - process known inodes and perform inode discovery...
 - agno = 0
 - agno = 1
 - agno = 2
 - agno = 3
 - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
 - setting up duplicate extent list...
 - check for inodes claiming duplicate blocks...
 - agno = 0
 - agno = 1
 - agno = 2
 - agno = 3
Phase 5 - rebuild AG headers and trees...
 - reset superblock...
Phase 6 - check inode connectivity...
 - resetting contents of realtime bitmap and summary inodes
 - traversing filesystem ...
 - traversal finished ...
 - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...
Done

```

从前面的`xfs_repair`命令的输出来看，我们的`/boot`文件系统似乎不需要任何修复过程。

### 这些工具是如何修复文件系统的？

你可能会认为使用`fsck`和`xfs_repair`等工具修复这个文件系统非常容易。原因很简单，这是因为`xfs`和`ext2/3/4`等文件系统的设计。`xfs`和`ext2/3/4`家族都是日志文件系统；这意味着这些类型的文件系统会记录对文件系统对象（如文件、目录等）所做的更改。 

这些更改将保存在日志中，直到更改提交到主文件系统。`xfs_repair`实用程序只是查看这个日志，并重放未提交到主文件系统的最后更改。这些文件系统日志使文件系统在意外断电或系统重新启动等情况下非常有韧性。

不幸的是，有时文件系统的日志和诸如`xfs_repair`之类的工具并不足以纠正情况。

在这种情况下，还有一些更多的选项，比如以强制模式运行修复。然而，这些选项应该总是保留作为最后的努力，因为它们有时会导致文件系统损坏。

如果你发现自己有一个损坏且无法修复的文件系统，最好的办法可能就是重新创建文件系统并恢复备份，如果有备份的话...

## 挂载文件系统

现在`/boot`文件系统已经经过检查和修复，我们可以简单地重新挂载它以验证数据是否正确。为此，我们可以简单地运行`mount`命令，然后跟上`/boot`：

```
[nfs]# mount /boot
[nfs]# mount | grep /boot
/dev/md127 on /boot type xfs (rw,relatime,seclabel,attr2,inode64,noquota)

```

当文件系统在`/etc/fstab`文件中定义时，可以只使用`mount`点调用`mount`和`umount`命令。这将导致这两个命令根据`/etc/fstab`文件中的定义来`mount`或`unmount`文件系统。

从`mount`的输出来看，我们的`/boot`文件系统现在是`读写`而不是`只读`。如果我们执行`ls`命令，我们也应该仍然看到我们的原始数据：

```
[nfs]# ls /boot
config-3.10.0-229.1.2.el7.x86_64                         initrd-plymouth.img
config-3.10.0-229.el7.x86_64                             symvers-3.10.0-229.1.2.el7.x86_64.gz
grub                                                     symvers-3.10.0-229.el7.x86_64.gz
grub2                                                    System.map-3.10.0-229.1.2.el7.x86_64
initramfs-0-rescue-3f370097c831473a8cfec737ff1d6c55.img  System.map-3.10.0-229.el7.x86_64
initramfs-3.10.0-229.1.2.el7.x86_64.img                  vmlinuz-0-rescue-3f370097c831473a8cfec737ff1d6c55
initramfs-3.10.0-229.1.2.el7.x86_64kdump.img             vmlinuz-3.10.0-229.1.2.el7.x86_64
initramfs-3.10.0-229.el7.x86_64.img                      vmlinuz-3.10.0-229.el7.x86_64
initramfs-3.10.0-229.el7.x86_64kdump.img

```

看来我们的恢复步骤取得了成功！现在我们已经用`/boot`文件系统测试过它们，我们可以开始修复`/nfs`文件系统了。

## 修复其他文件系统

修复`/nfs`文件系统的步骤实际上与`/boot`文件系统的步骤基本相同，只有一个主要的区别，如下所示：

```
[nfs]# lsof -r | grep /nfs
rpc.statd 1075            rpcuser  cwd       DIR              253,1 40     592302 /var/lib/nfs/statd
rpc.mount 2282               root  cwd       DIR              253,1 4096    9125499 /var/lib/nfs
rpc.mount 2282               root    4u      REG                0,3 0 4026532125 /proc/2280/net/rpc/nfd.export/channel
rpc.mount 2282               root    5u      REG                0,3 0 4026532129 /proc/2280/net/rpc/nfd.fh/channel

```

使用`lsof`检查`/nfs`文件系统上的打开文件时，我们可能看不到 NFS 服务进程。然而，很有可能 NFS 服务在`lsof`命令停止后会尝试访问这个共享文件系统中的文件。为了防止这种情况，最好（如果可能的话）在对共享文件系统进行任何更改时停止 NFS 服务：

```
[nfs]# systemctl stop nfs

```

一旦 NFS 服务停止，其余步骤都是一样的：

```
[nfs]# umount /nfs
[nfs]# xfs_repair /dev/md0/nfs
Phase 1 - find and verify superblock...
Phase 2 - using internal log
 - zero log...
 - scan filesystem freespace and inode maps...
 - found root inode chunk
Phase 3 - for each AG...
 - scan and clear agi unlinked lists...
 - process known inodes and perform inode discovery...
 - agno = 0
 - agno = 1
 - agno = 2
 - agno = 3
 - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
 - setting up duplicate extent list...
 - check for inodes claiming duplicate blocks...
 - agno = 0
 - agno = 1
 - agno = 2
 - agno = 3
Phase 5 - rebuild AG headers and trees...
 - reset superblock...
Phase 6 - check inode connectivity...
 - resetting contents of realtime bitmap and summary inodes
 - traversing filesystem ...
 - traversal finished ...
 - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...
done

```

文件系统修复后，我们可以简单地按如下方式重新挂载它：

```
[nfs]# mount /nfs
[nfs]# mount | grep /nfs
nfsd on /proc/fs/nfsd type nfsd (rw,relatime)
/dev/mapper/md0-nfs on /nfs type xfs (rw,relatime,seclabel,attr2,inode64,noquota)

```

重新挂载`/nfs`文件系统后，我们可以看到选项显示为`rw`，这意味着它是`可读写`的。

### 恢复`/`（根）文件系统

`/`或`root`文件系统有点不同。它不同，因为它是包含大部分 Linux 软件包、二进制文件和命令的顶层文件系统。这意味着我们不能简单地卸载这个文件系统，否则就会丢失重新挂载它所需的工具。

因此，我们实际上将使用`mount`命令重新挂载`/`文件系统，而无需先卸载它：

```
[nfs]# mount -o remount /

```

为了告诉`mount`命令卸载然后重新挂载文件系统，我们只需要传递`-o`（选项）标志，后面跟着选项`remount`。`-o`标志允许您从命令行传递文件系统选项，如`rw`或`ro`。当我们重新挂载`/`文件系统时，我们只是传递重新挂载文件系统选项：

```
# mount | grep root
/dev/mapper/md0-root on / type xfs (rw,relatime,seclabel,attr2,inode64,noquota)

```

如果我们使用`mount`命令来显示已挂载的文件系统，我们可以验证`/`文件系统已重新挂载为`读写`访问。由于文件系统类型为`xfs`，重新挂载应该导致文件系统执行一致性检查和修复。如果我们对`/`文件系统的完整性有任何疑问，下一步应该是简单地重新启动 NFS 服务器。

如果服务器无法挂载`/`文件系统，`xfs_repair`实用程序将自动调用。

# 验证

目前，我们可以看到 NFS 服务器的文件系统问题已经恢复。我们现在应该验证我们的 NFS 客户端能否写入 NFS 共享。但在这之前，我们还应该先重新启动之前停止的 NFS 服务：

```
[nfs]# systemctl start nfs
[nfs]# systemctl status nfs
nfs-server.service - NFS server and services
 Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; enabled)
 Active: active (exited) since Mon 2015-04-27 22:20:46 MST; 6s ago
 Process: 2278 ExecStopPost=/usr/sbin/exportfs -f (code=exited, status=0/SUCCESS)
 Process: 3098 ExecStopPost=/usr/sbin/exportfs -au (code=exited, status=1/FAILURE)
 Process: 3095 ExecStop=/usr/sbin/rpc.nfsd 0 (code=exited, status=0/SUCCESS)
 Process: 3265 ExecStart=/usr/sbin/rpc.nfsd $RPCNFSDARGS (code=exited, status=0/SUCCESS)
 Process: 3264 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCESS)
 Main PID: 3265 (code=exited, status=0/SUCCESS)
 CGroup: /system.slice/nfs-server.service

```

一旦 NFS 服务启动，我们可以使用`touch`命令从客户端进行测试：

```
[db]# touch /data/testfile.txt
[db]# ls -la /data/testfile.txt 
-rw-r--r--. 1 root root 0 Apr 28 05:24 /data/testfile.txt

```

看起来我们已经成功解决了问题。

另外，如果我们注意到对 NFS 共享的请求花费了很长时间，可能需要在客户端上卸载并重新挂载 NFS 共享。如果 NFS 客户端没有意识到 NFS 服务器已重新启动，这是一个常见问题。

# 总结

在本章中，我们深入探讨了文件系统的挂载方式，NFS 的配置以及文件系统进入`只读`模式时应该采取的措施。我们甚至进一步手动修复了一个物理磁盘设备出现问题的文件系统。

在下一章中，我们将进一步解决硬件故障的问题。这意味着查看硬件消息日志，解决硬盘 RAID 集的故障以及许多其他与硬件相关的故障排除步骤。
