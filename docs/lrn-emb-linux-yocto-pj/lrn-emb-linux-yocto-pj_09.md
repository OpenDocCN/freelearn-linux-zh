# 第九章：Wic 和其他工具

在本章中，将简要介绍一些解决各种问题并以巧妙方式解决它们的工具。这一章可以被认为是为你准备的开胃菜。如果这里介绍的任何工具似乎引起了你的兴趣，我鼓励你满足你的好奇心，尝试找到更多关于那个特定工具的信息。当然，这条建议适用于本书中提供的任何信息。然而，这条建议特别适用于本章，因为我选择了对我介绍的工具进行更一般的描述。我这样做是因为我假设你们中的一些人可能对冗长的描述不感兴趣，而只想把兴趣集中在开发过程中，而不是其他领域。对于其他对了解更多其他关键领域感兴趣的人，请随意浏览本章中提供的信息扩展。

在本章中，将提供对 Swabber、Wic 和 LAVA 等组件的更详细解释。这些工具不是嵌入式开发人员在日常工作中会遇到的工具，但与这些工具的交互可能会让生活变得更轻松一些。我应该首先提到这些工具的一件事是它们彼此之间没有任何共同之处，它们之间非常不同，并且解决了不同的问题。如果 Swabber，这里介绍的第一个工具，用于在主机开发机器上进行访问检测，那么第二个工具代表了 BitBake 在复杂打包选项方面的限制的解决方案。在这里，我指的是 wic 工具。本章介绍的最后一个元素是名为 LAVA 的自动化测试框架。这是来自 Linaro 的一个倡议，我认为这个项目非常有趣。它还与 Jenkins 等持续集成工具结合在一起，这可能对每个人都是一个致命的组合。

# 拖把

Swabber 是一个项目，虽然它在 Yocto Project 的官方页面上展示，但据说它还在进行中；自 2011 年 9 月 18 日以来没有任何活动。它没有维护者文件，您无法在其中找到更多关于其创建者的信息。然而，对于任何对这个项目感兴趣的人来说，提交者列表应该足够了解更多。

本章选择介绍这个工具，因为它构成了 Yocto Project 生态系统的另一个视角。当然，对主机系统进行访问检测的机制并不是一个坏主意，对于检测可能对系统有问题的访问非常有用，但在开发软件时并不是首选的工具。当你有可能重新构建并手动检查主机生态系统时，你往往会忽视工具也可以用于这个任务，并且它们可以让你的生活更轻松。

与 Swabber 交互，需要首先克隆存储库。可以使用以下命令来实现这一目的：

```
git clone http://git.yoctoproject.org/git/swabber

```

源代码在主机上可用后，存储库的内容应如下所示：

```
tree swabber/
swabber/
├── BUGS
├── canonicalize.c
├── canonicalize.h
├── COPYING
├── detect_distro
├── distros
│   ├── Fedora
│   │   └── whitelist
│   ├── generic
│   │   ├── blacklist
│   │   ├── filters
│   │   └── whitelist
│   ├── Ubuntu
│   │   ├── blacklist
│   │   ├── filters
│   │   └── whitelist
│   └── Windriver
│       └── whitelist
├── dump_blob.c
├── lists.c
├── lists.h
├── load_distro.c
├── Makefile
├── packages.h
├── README
├── swabber.c
├── swabber.h
├── swabprof.c
├── swabprof.in
├── swab_testf.c
├── update_distro
├── wandering.c
└── wandering.h

5 directories, 28 files

```

正如你所看到的，这个项目并不是一个重大项目，而是由一些热情的人提供的一些工具。其中包括来自**Windriver**的两个人：Alex deVries 和 David Borman。他们独自开发了之前介绍的工具，并将其提供给开源社区使用。Swabber 是用 C 语言编写的，这与 Yocto Project 社区提供的通常的 Python/Bash 工具和其他项目有很大的不同。每个工具都有自己的目的，相似之处在于所有工具都是使用相同的 Makefile 构建的。当然，这不仅限于使用二进制文件；还有两个 bash 脚本可用于分发检测和更新。

### 注

有关该工具的更多信息可以从其创建者那里获得。他们的电子邮件地址，可在项目的提交中找到，分别是`<alex.devries@windriver.com>`和`<david.borman@windriver.com>`。但请注意，这些是工作场所的电子邮件地址，而曾经参与 Swabber 工作的人现在可能没有相同的电子邮件地址。

与 Swabber 工具的交互在`README`文件中有很好的描述。在这里，关于 Swabber 的设置和运行的信息是可用的，不过，为了你的方便，这也将在接下来的几行中呈现，以便你能更快地理解和更容易地掌握。

第一个必要的步骤是编译源代码。这是通过调用`make`命令来完成的。在源代码构建并可执行文件可用后，可以使用`update_distro`命令对主机分发进行配置，然后是分发目录的位置。我们选择的名称是`Ubuntu-distro-test`，它是特定于执行工具的主机分发。这个生成过程一开始可能需要一些时间，但之后，对主机系统的任何更改都将被检测到，并且过程所需的时间将更少。在配置过程结束时，`Ubuntu-distro-test`目录的内容如下：

```
Ubuntu-distro-test/
├── distro
├── distro.blob
├── md5
└── packages

```

主机分发配置文件后，可以基于创建的配置文件生成一个 Swabber 报告。此外，在创建报告之前，还可以创建一个配置文件日志，以备报告过程中使用。为了生成报告，我们将创建一个具有特定日志信息的日志文件位置。日志可用后，就可以生成报告了：

```
strace -o logs/Ubuntu-distro-test-logs.log -e trace=open,execve -f pwd
./swabber -v -v -c all -l logs/ -o required.txt -r extra.txt -d Ubuntu-distro-test/ ~ /tmp/

```

工具需要这些信息，如其帮助信息所示：

```
Usage: swabber [-v] [-v] [-a] [-e]
 -l <logpath> ] -o <outputfile> <filter dir 1> <filter dir 2> ...

 Options:
 -v: verbose, use -v -v for more detail
 -a: print progress (not implemented)
 -l <logfile>: strace logfile or directory of log files to read
 -d <distro_dir>: distro directory
 -n <distro_name>: force the name of the distribution
 -r <report filename>: where to dump extra data (leave empty for stdout)
 -t <global_tag>: use one tag for all packages
 -o <outputfile>: file to write output to
 -p <project_dir>: directory were the build is being done
 -f <filter_dir>: directory where to find filters for whitelist,
 blacklist, filters
 -c <task1>,<task2>...: perform various tasks, choose from:
 error_codes: show report of files whose access returned an error
 whitelist: remove packages that are in the whitelist
 blacklist: highlight packages that are in the blacklist as
 being dangerous
 file_detail: add file-level detail when listing packages
 not_in_distro: list host files that are not in the package
 database
 wandering: check for the case where the build searches for a
 file on the host, then finds it in the project.
 all: all the above

```

从前面代码中附加的帮助信息中，可以调查测试命令所选参数的作用。此外，由于 C 文件中不超过 1550 行，最大的文件是`swabber.c`文件，因此建议检查工具的源代码。

`required.txt`文件包含有关使用的软件包和特定文件的信息。有关配置的更多信息也可以在`extra.txt`文件中找到。这些信息包括可以访问的文件和软件包，各种警告以及主机数据库中不可用的文件，以及各种错误和被视为危险的文件。

对于跟踪的命令，输出信息并不多。这只是一个示例；我鼓励你尝试各种场景，并熟悉这个工具。这可能对你以后有所帮助。

# Wic

Wic 是一个命令行工具，也可以看作是 BitBake 构建系统的扩展。它是由于需要有一个分区机制和描述语言而开发的。很容易得出结论，BitBake 在这些方面存在不足，尽管已经采取了一些措施，以确保这样的功能在 BitBake 构建系统内可用，但这只能在一定程度上实现；对于更复杂的任务，Wic 可以是一个替代解决方案。

在接下来的几行中，我将尝试描述与 BitBake 功能不足相关的问题，以及 Wic 如何以简单的方式解决这个问题。我还将向你展示这个工具是如何诞生的，以及灵感来源是什么。

在使用 BitBake 构建图像时，工作是在继承`image.bbclass`的图像配方中完成的，以描述其功能。在这个类中，`do_rootfs()`任务是负责创建后续将包含在最终软件包中的根文件系统目录的 OS。该目录包含了在各种板上引导 Linux 图像所需的所有源。完成`do_rootfs()`任务后，会查询一系列命令，为每种图像定义类型生成输出。图像类型的定义是通过`IMAGE_FSTYPE`变量完成的，对于每种图像输出类型，都有一个`IMAGE_CMD_type`变量被定义为从外部层继承的额外类型，或者是在`image_types.bbclass`文件中描述的基本类型。

实际上，每种类型背后的命令都是针对特定的根文件系统格式的 shell 命令。其中最好的例子就是`ext3`格式。为此，定义了`IMAGE_CMD_ext3`变量，并调用了这些命令，如下所示：

```
genext2fs -b $ROOTFS_SIZE ... ${IMAGE_NAME}.rootfs.ext3
tune2fs -j ${DEPLOY_DIR_IMAGE}/${IMAGE_NAME}.rootfs.ext3

```

在调用命令后，输出以`image-*.ext3`文件的形式呈现。这是根据定义的`FSTYPES`变量值新创建的 EXT3 文件系统，并包含了根文件系统内容。这个例子展示了一个非常常见和基本的文件系统创建命令。当然，在工业环境中可能需要更复杂的选项，这些选项不仅包括根文件系统，还可能包括额外的内核或甚至是引导加载程序。对于这些复杂的选项，需要广泛的机制或工具。

Yocto 项目中可见的可用机制在`image_types.bbclass`文件中通过`IMAGE_CMD_type`变量可见，并具有以下形式：

```
image_types_foo.bbclass:
  IMAGE_CMD_bar = "some shell commands"
  IMAGE_CMD_baz = "some more shell commands"
```

要使用新定义的图像格式，需要相应地更新机器配置，使用以下命令：

```
foo-default-settings.inc
  IMAGE_CLASSES += "image_types_foo"
```

通过在`image.bbclass`文件中使用`inherit ${IMAGE_CLASSES}`命令，新定义的`image_types_foo.bbclass`文件的功能可见并准备好被使用，并添加到`IMAGE_FSTYPE`变量中。

前面的实现意味着对于每个实现的文件系统，都会调用一系列命令。这对于非常简单的文件系统格式是一个很好的简单方法。然而，对于更复杂的文件系统，需要一种语言来定义格式、状态以及图像格式的属性。Poky 中提供了各种其他复杂的图像格式选项，如**vmdk**、**live**和**directdisk**文件类型，它们都定义了一个多阶段的图像格式化过程。

要使用`vmdk`图像格式，需要在`IMAGE_FSTYPE`变量中定义一个`vmdk`值。然而，为了生成和识别这种图像格式，应该可用并继承`image-vmdk.bbclass`文件的功能。有了这些功能，可以发生三件事：

+   在`do_rootfs()`任务中创建了对 EXT3 图像格式的依赖，以确保首先生成`ext3`图像格式。`vmdk`图像格式依赖于此。

+   `ROOTFS`变量被设置为`boot-directdisk`功能。

+   继承了`boot-directdisk.bbclass`。

此功能提供了生成可以复制到硬盘上的映像的可能性。在其基础上，可以生成 `syslinux` 配置文件，并且启动过程还需要两个分区。最终结果包括 MBR 和分区表部分，后跟一个包含引导文件、SYSLINUX 和 Linux 内核的 FAT16 分区，以及用于根文件系统位置的 EXT3 分区。此图像格式还负责将 Linux 内核、`syslinux.cfg` 和 `ldlinux.sys` 配置移动到第一个分区，并使用 `dd` 命令将 EXT3 图像格式复制到第二个分区。在此过程结束时，使用 `tune2fs` 命令为根目录保留空间。

从历史上看，`directdisk` 在其最初版本中是硬编码的。对于每个图像配方，都有一个类似的实现，它镜像了基本实现，并在 `image.bbclass` 功能的配方中硬编码了遗产。对于 `vmdk` 图像格式，添加了 `inherit boot-directdisk` 行。

关于自定义定义的图像文件系统类型，一个示例可以在 `meta-fsl-arm` 层中找到；此示例可在 `imx23evk.conf` 机器定义中找到。此机器添加了下面两种图像文件系统类型：`uboot.mxsboot-sdcard` 和 `sdcard`。

```
meta-fsl-arm/imx23evk.conf
  include conf/machine/include/mxs-base.inc
  SDCARD_ROOTFS ?= "${DEPLOY_DIR_IMAGE}/${IMAGE_NAME}.rootfs.ext3"
  IMAGE_FSTYPES ?= "tar.bz2 ext3 uboot.mxsboot-sdcard sdcard"
```

在前面的行中包含的 `mxs-base.inc` 文件又包含了 `conf/machine/include/fsl-default-settings.inc` 文件，后者又添加了 `IMAGE_CLASSES +="image_types_fsl"` 行，如一般情况所示。使用前面的行提供了首先为 `uboot.mxsboot-sdcard` 格式可用的命令执行 `IMAGE_CMD` 命令的可能性，然后是 `sdcard IMAGE_CMD` 命令特定的图像格式。

`image_types_fsl.bbclass` 文件定义了 `IMAGE_CMD` 命令，如下所示：

```
inherit image_types
  IMAGE_CMD_uboot.mxsboot-sdcard = "mxsboot sd ${DEPLOY_DIR_IMAGE}/u-boot-${MACHINE}.${UBOOT_SUFFIX} \
${DEPLOY_DIR_IMAGE}/${IMAGE_NAME}.rootfs.uboot.mxsboot-sdcard"
```

在执行过程结束时，使用 `mxsboot` 命令调用 `uboot.mxsboot-sdcard` 命令。执行此命令后，将调用 `IMAGE_CMD_sdcard` 特定命令来计算 SD 卡的大小和对齐方式，初始化部署空间，并将适当的分区类型设置为 `0x53` 值，并将根文件系统复制到其中。在此过程结束时，将可用多个分区，并且它们具有相应的 twiddles，用于打包可引导的映像。

有多种方法可以创建各种文件系统，它们分布在大量现有的 Yocto 层中，并且一些文档可供一般公众使用。甚至有许多脚本用于为开发人员的需求创建合适的文件系统。其中一个示例是 `scripts/contrib/mkefidisk.sh` 脚本。它用于从另一种图像格式（即 `live.hddimg`）创建一个 EFI 可引导的直接磁盘映像。然而，一个主要的想法仍然存在：这种类型的活动应该在没有在中间阶段生成的中间图像文件系统的情况下进行，并且应该使用无法处理复杂场景的分区语言。

牢记这些信息，似乎在前面的示例中，我们应该使用另一个脚本。考虑到可以在构建系统内部和外部构建映像的可能性，开始寻找适合我们需求的一些工具。这个搜索结果是 Fedora Kickstart 项目。尽管它的语法也适用于涉及部署工作的领域，但它通常被认为对开发人员最有帮助。

### 注意

有关 Fedora Kickstart 项目的更多信息，请访问 [`fedoraproject.org/wiki/Anaconda/Kickstart`](http://fedoraproject.org/wiki/Anaconda/Kickstart)。

从这个项目中，最常用和有趣的组件是`clearpart`，`part`和`bootloader`，这些对我们的目的也很有用。当您查看 Yocto 项目的 Wic 工具时，它也可以在配置文件中找到。如果 Wic 的配置文件在 Fedora kickstart 项目中定义为`.wks`，则配置文件使用`.yks`扩展名。一个这样的配置文件定义如下：

```
def pre():
    free-form python or named 'plugin' commands

  clearpart commands
  part commands
  bootloader commands
  named 'plugin' commands

  def post():
    free-form python or named 'plugin' commands  
```

前面脚本背后的想法非常简单：`clearpart`组件用于清除磁盘上的任何分区，而`part`组件用于相反的操作，即用于创建和安装文件系统的组件。定义的第三个工具是`bootloader`组件，用于安装引导加载程序，并处理从`part`组件接收到的相应信息。它还确保引导过程按照配置文件中的描述进行。定义为`pre()`和`post()`的函数用于创建图像、阶段图像工件或其他复杂任务的预和后计算。

如前述描述所示，与 Fedora kickstarter 项目的交互非常富有成效和有趣，但源代码是在 Wic 项目内使用 Python 编写的。这是因为搜索了一个类似工具的 Python 实现，并在`pykickstarted`库的形式下找到了。这并不是 Meego 项目在其**Meego Image Creator**（**MIC**）工具中使用的前述库的全部用途。该工具用于 Meego 特定的图像创建过程。后来，该项目被 Tizen 项目继承。

### 注意

有关 MIC 的更多信息，请参阅[`github.com/01org/mic`](https://github.com/01org/mic)。

Wic，我承诺在本节中介绍的工具，源自 MIC 项目，它们两者都使用 kickstarter 项目，因此所有三者都基于定义了创建各种图像格式过程行为的插件。在 Wic 的第一个实现中，它主要是 MIC 项目的功能。在这里，我指的是它定义的 Python 类，几乎完全复制到了 Poky 中。然而，随着时间的推移，该项目开始拥有自己的实现，也有了自己的个性。从 Poky 存储库的 1.7 版本开始，不再直接引用 MIC Python 定义的类，使 Wic 成为一个独立的项目，具有自己定义的插件和实现。以下是您可以检查 Wic 中可访问的各种格式配置的方法：

```
tree scripts/lib/image/canned-wks/
scripts/lib/image/canned-wks/
├── directdisk.wks
├── mkefidisk.wks
├── mkgummidisk.wks
└── sdimage-bootpart.wks
```

Wic 中定义了配置。然而，考虑到这个工具近年来的兴趣增加，我们只能希望支持的配置数量会增加。

我之前提到 MIC 和 Fedora kickstarter 项目的依赖关系已经被移除，但在 Poky `scripts/lib/wic`目录中快速搜索会发现情况并非如此。这是因为 Wic 和 MIC 都有相同的基础，即`pykickstarted`库。尽管 Wic 现在在很大程度上基于 MIC，并且两者都有相同的父级，即 kickstarter 项目，但它们的实现、功能和各种配置使它们成为不同的实体，尽管相关，但它们已经走上了不同的发展道路。

# LAVA

**LAVA**（**Linaro 自动化和验证架构**）是一个连续集成系统，专注于物理目标或虚拟硬件部署，其中执行一系列测试。执行的测试种类繁多，从只需要启动目标的最简单测试到需要外部硬件交互的非常复杂的场景。

LAVA 代表一系列用于自动验证的组件。LAVA 堆栈的主要思想是创建一个适用于各种规模项目的质量受控测试和自动化环境。要更仔细地查看 LAVA 实例，读者可以检查已经创建的实例，由 Linaro 在剑桥托管的官方生产实例。您可以在[`validation.linaro.org/`](https://validation.linaro.org/)访问它。希望您喜欢使用它。

LAVA 框架支持以下功能：

+   它支持在各种硬件包上对多个软件包进行定期自动测试

+   确保设备崩溃后系统会自动重新启动

+   它进行回归测试

+   它进行持续集成测试

+   它进行平台启用测试

+   它支持本地和云解决方案

+   它提供了结果捆绑支持

+   它提供性能和功耗的测量

LAVA 主要使用 Python 编写，这与 Yocto 项目提供的内容没有什么不同。正如在 Toaster 项目中看到的那样，LAVA 还使用 Django 框架进行 Web 界面，项目使用 Git 版本控制系统进行托管。这并不奇怪，因为我们正在谈论 Linaro，这是一个致力于自由开源项目的非营利组织。因此，应用于项目的所有更改应返回到上游项目，使项目更容易维护。但是，它也更健壮，性能更好。

### 注意

对于那些对如何使用该项目的更多细节感兴趣的人，请参阅[`validation.linaro.org/static/docs/overview.html`](https://validation.linaro.org/static/docs/overview.html)。

使用 LAVA 框架进行测试，第一步是了解其架构。了解这一点不仅有助于测试定义，还有助于扩展测试，以及整个项目的开发。该项目的主要组件如下：

```
               +-------------+
               |web interface|
               +-------------+
                      |
                      v
                  +--------+
            +---->|database|
            |     +--------+
            |
+-----------+------[worker]-------------+
|           |                           |
|  +----------------+     +----------+  |
|  |scheduler daemon|---→ |dispatcher|  |
|  +----------------+     +----------+  |
|                              |        |
+------------------------------+--------+
                               |
                               V
                     +-------------------+
                     | device under test |
                     +-------------------+
```

第一个组件**Web 界面**负责用户交互。它用于存储数据和使用 RDBMS 提交作业，并负责显示结果、设备导航，或者通过 XMLRPC API 进行作业提交接收活动。另一个重要组件是**调度程序守护程序**，负责分配作业。它的活动非常简单。它负责从数据库中汇集数据，并为由调度程序提供给它们的作业保留设备，调度程序是另一个重要组件。**调度程序**是负责在设备上运行实际作业的组件。它还管理与设备的通信，下载图像并收集结果。

有时只能使用调度程序的情况；这些情况涉及使用本地测试或测试功能开发。还有一些情况，所有组件都在同一台机器上运行，比如单个部署服务器。当然，理想的情况是组件解耦，服务器在一台机器上，数据库在另一台机器上，调度程序守护程序和调度程序在另一台机器上。

对于使用 LAVA 进行开发过程，推荐的主机是 Debian 和 Ubuntu。与 LAVA 合作的 Linaro 开发团队更喜欢 Debian 发行版，但它也可以在 Ubuntu 机器上很好地运行。有一些需要提到的事情：对于 Ubuntu 机器，请确保宇宙存储库可供包管理器使用并可见。

必需的第一个软件包是`lava-dev`；它还有脚本指示必要的软件包依赖项，以确保 LAVA 工作环境。以下是执行此操作所需的必要命令：

```
sudo apt-get install lava-dev
git clone http://git.linaro.org/git/lava/lava-server.git
cd lava-server
/usr/share/lava-server/debian-dev-build.sh lava-server

git clone http://git.linaro.org/git/lava/lava-dispatcher.git
cd lava-dispatcher
/usr/share/lava-server/debian-dev-build.sh lava-dispatcher

```

考虑到更改的位置，需要采取各种行动。例如，对于“模板”目录中的 HTML 内容的更改，刷新浏览器就足够了，但在`*_app`目录的 Python 实现中进行的任何更改都需要重新启动`apache2ctl`HTTP 服务器。此外，`*_daemon`目录中的 Python 源代码的任何更改都需要完全重新启动`lava-server`。

### 注意

对于所有对获取有关 LAVA 开发的更多信息感兴趣的人，开发指南构成了一份良好的文档资源，可在[`validation.linaro.org/static/docs/#developer-guides`](https://validation.linaro.org/static/docs/#developer-guides)找到。

要在 64 位 Ubuntu 14.04 机器上安装 LAVA 或任何与 LAVA 相关的软件包，除了启用通用存储库`deb http://people.linaro.org/~neil.williams/lava jessie main`之外，还需要新的软件包依赖项，以及之前为 Debian 发行版描述的安装过程。我必须提到，当安装`lava-dev`软件包时，用户将被提示进入一个菜单，指示`nullmailer mailname`。我选择让默认值保持不变，实际上这是运行`nullmailer`服务的计算机的主机名。我还保持了默认为`smarthost`定义的相同配置，并且安装过程已经继续。以下是在 Ubuntu 14.04 机器上安装 LAVA 所需的命令：

```
sudo add-apt-repository "deb http://archive.ubuntu.com/ubuntu $(lsb_release -sc) universe"
sudo apt-get update
sudo add-apt-repository "deb http://people.linaro.org/~neil.williams/lava jessie main"
sudo apt-get update

sudo apt-get install postgresql
sudo apt-get install lava
sudo a2dissite 000-default
sudo a2ensite lava-server.conf
sudo service apache2 restart

```

### 注意

有关 LAVA 安装过程的信息可在[`validation.linaro.org/static/docs/installing_on_debian.html#`](https://validation.linaro.org/static/docs/installing_on_debian.html#)找到。在这里，您还可以找到 Debian 和 Ubuntu 发行版的安装过程。

# 总结

在本章中，您被介绍了一组新的工具。我必须诚实地承认，这些工具并不是在嵌入式环境中最常用的工具，但它们被引入是为了为嵌入式开发环境提供另一个视角。本章试图向开发人员解释，嵌入式世界不仅仅是开发和帮助这些任务的工具。在大多数情况下，相邻的组件可能是对开发过程影响最大的组件。

在下一章中，将简要介绍 Linux 实时要求和解决方案。我们将强调在这一领域与 Linux 一起工作的各种功能。将提供 meta-realtime 层的简要介绍，并讨论 Preempt-RT 和 NOHZ 等功能。话不多说，让我们继续下一章。希望您会喜欢它的内容。
