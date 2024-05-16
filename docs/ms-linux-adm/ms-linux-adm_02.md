# 第三章：基本系统管理任务

## 用户和组管理

在 Linux 中，用户和组管理是系统管理的重要部分。它涉及在系统上创建、修改和删除用户和组，分配权限和特权，并管理系统上的用户帐户。以下是 Linux 中用于用户和组管理的一些常用命令和工具：

+   useradd：此命令用于创建新用户帐户。例如，要创建名为“john”的用户，可以运行 useradd john。

+   passwd：passwd 命令允许您设置或更改用户的密码。通过运行 passwd john，您可以为用户“john”设置密码。

+   userdel：要删除用户帐户，可以使用 userdel 命令。例如，userdel john 将从系统中删除用户“john”。

+   groupadd：此命令用于创建新组。例如，groupadd developers 将创建一个名为“developers”的组。

+   groupdel：要删除组，可以使用 groupdel 命令。例如，groupdel developers 将删除组“developers”。

+   usermod：usermod 命令允许您修改用户帐户的各种属性，例如用户名、主目录或组。例如，usermod -l newname john 将把用户“john”重命名为“newname”。

+   usermod -aG：此命令用于将用户添加到一个或多个组。例如，usermod -aG developers john 将用户“john”添加到“developers”组。

+   id：id 命令显示指定用户的用户和组信息。运行 id john 将显示用户“john”的用户和组成员资格。

+   chown：chown 命令用于更改文件和目录的所有权。例如，chown john:developers myfile.txt 将把文件“myfile.txt”的所有者更改为“john”，组更改为“developers”。

+   chgrp：chgrp 命令更改文件和目录的组所有权。运行 chgrp developers myfile.txt 将把文件“myfile.txt”的组所有权更改为“developers”。

这些只是 Linux 中可用于用户和组管理的命令和工具的几个示例。根据您使用的具体 Linux 发行版，还有其他选项和配置可用。始终建议查阅文档或手册页面（man 命令）以获取有关每个命令的更详细信息。

## 权限和访问控制

Linux 权限和访问控制是 Linux 操作系统的关键方面，有助于保护文件、目录和系统资源。它们确保只有授权用户和进程才能访问或修改某些文件和目录。Linux 使用权限和所有权的组合来控制对文件和目录的访问。

### Linux 权限：

Linux 权限由一组三个字符或字符组表示：读取（r）、写入（w）和执行（x）。这些权限可以分配给三个不同的用户组：文件的所有者、与文件关联的组和所有其他用户。

+   读取（r）：允许用户查看文件的内容或列出目录中的文件。

+   写入（w）：允许用户修改或删除文件，或在目录中创建、修改或删除文件。

+   执行（x）：允许用户将文件作为程序执行或访问目录。

### 访问控制列表（ACL）：

除了标准权限外，Linux 还支持访问控制列表（ACL）。ACL 提供了更精细的访问控制级别，允许附加权限并指定特定用户或组的访问。

### 文件所有权：

Linux 中的每个文件和目录都有一个关联的所有者。所有者具有特殊权限，允许他们控制对文件的访问，包括更改权限和修改所有权。所有者还可以为文件分配特定的组。

### Linux 权限表示的示例：

文件或目录的权限通常表示为十个字符的序列：

+   第一个字符表示文件类型：“-”表示常规文件，“d”表示目录，“l”表示符号链接，依此类推。

+   下一个三个字符表示所有者的权限。

+   以下三个字符表示组的权限。

+   最后三个字符表示其他用户的权限。

例如，权限表示“drwxr-xr--”表示文件是一个目录（d），所有者具有读（r），写（w）和执行（x）权限，组具有读（r）和执行（x）权限，其他用户只有读（r）权限。

### 更改权限：

要更改文件或目录的权限，可以在 Linux 中使用“chmod”命令。它允许您修改所有者，组和其他用户的读取，写入和执行权限。

访问控制是 Linux 安全的关键方面，了解并正确配置权限和访问控制可以帮助保护敏感数据并维护系统的完整性。

## 进程管理和监控

进程管理和监控是管理基于 Linux 的系统的重要方面，确保资源利用效率和维护系统稳定性。以下是 Linux 进程管理和监控的概述：

### 进程管理

+   ps：ps 命令显示有关活动进程的信息。常见选项包括：

+   ps aux：显示系统上运行的所有进程的详细列表。

+   ps -ef：类似于 ps aux，但使用 BSD 风格的语法。

+   ps -e：显示所有进程的简单列表。

+   top：top 命令提供动态的，实时的系统进程和资源使用情况视图。它不断更新信息，显示 CPU，内存和其他统计信息。按“q”退出。

+   htop：top 的改进版本，具有更加用户友好的界面和额外功能，如滚动和进程树视图。

+   kill：kill 命令通过向进程发送信号来终止进程。例如：

+   kill PID：向具有指定 PID（进程 ID）的进程发送 SIGTERM 信号。

+   kill -9 PID：发送 SIGKILL 信号，强制进程立即终止。

+   nice 和 renice：nice 命令以指定的优先级级别（niceness）启动新进程。renice 允许更改现有进程的优先级。

+   bg 和 fg：bg 命令将停止或挂起的进程移至后台，而 fg 将后台进程带回前台。

+   nohup：nohup 命令允许运行即使终端会话结束后仍继续的进程。

### 进程监控

+   使用 top 和 htop 进行系统监控：top 和 htop 都提供进程，CPU，内存，负载平均值和其他系统统计信息的实时监控。

+   sar：sar 命令用于系统活动报告，提供 CPU，内存，I/O 和网络使用情况的历史性能数据。

+   vmstat：vmstat 命令报告虚拟内存统计信息，包括内存，分页和 CPU 活动。

+   iostat：iostat 命令显示设备和分区的 I/O 统计信息。

+   ps 和 watch：将 ps 与 watch 结合使用可以连续监视特定进程。例如：

watch -n 1 'ps aux | grep process_name'

每秒监视名为“process_name”的进程。

+   Systemd：对于使用 systemd 的系统，systemctl 提供对系统服务的控制和监控。例如：

systemctl status service_name

显示特定服务的状态。

+   监控工具：各种监控工具如 Nagios，Zabbix 和 Prometheus 为更大的环境提供了更广泛的监控和警报功能。

定期监视进程和系统资源对于识别性能问题，资源瓶颈和潜在问题至关重要。适当的进程管理确保资源的有效利用，并有助于在基于 Linux 的环境中维护系统的稳定性。

## 管理服务和守护程序

在 Linux 中，服务和守护进程是持续运行以执行各种任务的后台进程。管理服务和守护进程涉及根据需要启动、停止、重新启动、启用和禁用它们。管理服务和守护进程的具体方法可能因您使用的 Linux 发行版而异。但是，我将为您提供一些通用指南，这些指南通常适用。

### 服务与守护进程：

服务是在后台运行并通常提供与网络相关功能的程序，例如 Web 服务器或数据库服务器。守护进程是执行特定任务或功能的任何后台进程的通用术语。

### 服务管理命令：

+   service：此命令通常用于管理使用 System V init 系统的发行版中的服务，例如 Debian 或 Ubuntu。例如：

+   启动服务：service <service-name> start

+   停止服务：service <service-name> stop

+   重新启动服务：service <service-name> restart

+   检查服务状态：service <service-name> status

### Systemctl 命令（systemd）：

许多现代 Linux 发行版使用 systemd init 系统，其中包括用于管理服务和守护进程的 systemctl 命令。

+   启动服务：systemctl start <service-name>

+   停止服务：systemctl stop <service-name>

+   重新启动服务：systemctl restart <service-name>

+   检查服务状态：systemctl status <service-name>

+   启用服务在启动时自动启动：systemctl enable <service-name>

+   禁止服务在启动时自动启动：systemctl disable <service-name>

### 配置文件：

服务和守护进程配置文件通常存储在/etc 目录中。

这些配置文件的确切位置和格式可能会因发行版和您正在管理的服务或守护进程而异。

常见的配置文件包括 System V init 的/etc/init.d/<service-name>和 systemd 的/etc/systemd/system/<service-name>.service。

重要的是要注意，这些是一般指南，实际命令和程序可能因 Linux 发行版而异。建议查阅文档或特定指南，以获取管理服务和守护进程的准确指令。
