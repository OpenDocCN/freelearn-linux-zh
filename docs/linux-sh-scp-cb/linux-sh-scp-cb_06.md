# 第六章：备份计划

在本章中，我们将涵盖：

+   使用 tar 进行存档

+   使用 cpio 进行存档

+   使用 gunzip（gzip）进行压缩

+   使用 bunzip（bzip）进行压缩

+   使用 lzma 进行压缩

+   使用 zip 进行存档和压缩

+   重压缩 squashfs 文件系统

+   使用标准算法对文件和文件夹进行加密

+   使用 rsync 备份快照

+   使用 git 进行版本控制备份

+   使用 dd 进行克隆磁盘

# 介绍

数据的快照和备份是我们经常遇到的常规任务。当涉及到服务器或大型数据存储系统时，定期备份非常重要。可以通过 shell 脚本自动化备份。存档和压缩似乎在系统管理员或普通用户的日常生活中找到了用途。有各种压缩格式可以以各种方式使用，以便获得最佳结果。加密是另一个经常使用的任务，用于保护数据。为了减小加密数据的大小，通常在加密之前对文件进行存档和压缩。有许多标准的加密算法可供使用，并且可以使用 shell 实用程序进行处理。本章将介绍使用 shell 创建和维护文件或文件夹存档、压缩格式和加密技术的不同方法。让我们来看看这些方法。

# 使用 tar 进行存档

`tar`命令可用于存档文件。它最初是为了在磁带存档上存储数据而设计的。它允许您将多个文件和目录存储为单个文件。它可以保留所有文件属性，如所有者、权限等。`tar`命令创建的文件通常被称为 tarball。

## 准备就绪

`tar`命令默认随所有类 UNIX 操作系统一起提供。它具有简单的语法，并且是一种可移植的文件格式。让我们看看如何做到这一点。

`tar`有一系列参数：`A`、`c`、`d`、`r`、`t`、`u`、`x`、`f`和`v`。每个字母都可以独立使用，用于相应的不同目的。

## 如何做到…

要使用 tar 存档文件，请使用以下语法：

```
$ tar -cf output.tar [SOURCES]

```

例如：

```
$ tar -cf output.tar file1 file2 file3 folder1 ..

```

在这个命令中，`-c`代表“创建文件”，`–f`代表“指定文件名”。

我们可以将文件夹和文件名指定为`SOURCES`。我们可以使用文件名列表或通配符，如`*.txt`来指定源。

它将源文件存档到名为`output.tar`的文件中。

文件名必须立即出现在`–f`之后，并且应该是参数组中的最后一个选项（例如，`-cvvf filename.tar`和`-tvvf filename.tar`）。

我们不能将数百个文件或文件夹作为命令行参数传递，因为有限制。因此，如果要存档许多文件，最好使用附加选项。

## 还有更多…

让我们看看`tar`命令提供的其他功能。

### 将文件附加到存档中

有时我们可能需要将文件添加到已经存在的存档中（一个示例用法是当需要存档数千个文件时，无法将它们作为命令行参数在一行中指定）。

附加选项：`-r`

为了将文件附加到已经存在的存档中使用：

```
$ tar -rvf original.tar new_file

```

按如下方式列出存档中的文件：

```
$ tar -tf archive.tar
yy/lib64/
yy/lib64/libfakeroot/
yy/sbin/

```

为了在存档或列表时打印更多细节，使用`-v`或`–vv`标志。这些标志称为详细（`v`），它将在终端上打印更多细节。例如，通过使用详细，您可以打印更多细节，如文件权限、所有者组、修改日期等。

例如：

```
$ tar -tvvf archive.tar
drwxr-xr-x slynux/slynux     0 2010-08-06 09:31 yy/
drwxr-xr-x slynux/slynux     0 2010-08-06 09:39 yy/usr/
drwxr-xr-x slynux/slynux     0 2010-08-06 09:31 yy/usr/lib64/

```

### 从存档中提取文件和文件夹

以下命令将存档的内容提取到当前目录：

```
$ tar -xf archive.tar

```

`-x`选项代表提取。

当使用`–x`时，`tar`命令将存档的内容提取到当前目录。我们还可以使用`–C`标志指定需要提取文件的目录，如下所示：

```
$ tar -xf archive.tar -C /path/to/extraction_directory

```

该命令将归档的内容提取到指定目录中。它提取归档的全部内容。我们也可以通过将它们指定为命令参数来仅提取一些文件：

```
$ tar -xvf file.tar file1 file4

```

上面的命令仅提取`file1`和`file4`，并忽略归档中的其他文件。

### 使用 tar 的 stdin 和 stdout

在归档时，我们可以指定`stdout`作为输出文件，以便通过管道出现的另一个命令可以将其读取为`stdin`，然后执行一些处理或提取归档。

这对于通过安全外壳（SSH）连接传输数据很有帮助（在网络上）。例如：

```
$ mkdir ~/destination
$ tar -cf - file1 file2 file3 | tar -xvf -  -C ~/destination

```

在上面的例子中，`file1`，`file2`和`file3`被合并成一个 tarball，然后提取到`~/destination`。在这个命令中：

+   `-f`指定`stdout`作为归档文件（当使用`-c`选项时）

+   `-f`指定`stdin`作为提取文件（当使用`-x`选项时）

### 连接两个归档

我们可以使用`-A`选项轻松合并多个 tar 文件。

假设我们有两个 tarballs：`file1.tar`和`file2.tar`。我们可以将`file2.tar`的内容合并到`file1.tar`中，如下所示：

```
$ tar -Af file1.tar file2.tar

```

通过列出内容来验证它：

```
$ tar -tvf file1.tar

```

### 使用时间戳检查更新归档中的文件

追加选项将任何给定的文件追加到归档中。如果在归档中有相同的文件要追加，它将追加该文件，并且归档将包含重复文件。我们可以使用更新选项`-u`来指定仅追加比归档中具有相同名称的文件更新的文件。

```
$ tar -tf archive.tar
filea
fileb
filec

```

该命令列出归档中的文件。

为了仅在`filea`的修改时间比`archive.tar`中的`filea`更新时追加`filea`，使用：

```
$ tar -uvvf archive.tar filea

```

如果归档外的`filea`的版本和`archive.tar`中的`filea`具有相同的时间戳，则不会发生任何事情。

使用`touch`命令修改文件时间戳，然后再次尝试`tar`命令：

```
$ tar -uvvf archive.tar filea
-rw-r--r-- slynux/slynux     0 2010-08-14 17:53 filea

```

由于其时间戳比归档中的时间戳更新，因此文件被追加。

### 比较归档和文件系统中的文件

有时候知道归档中的文件和文件系统中具有相同文件名的文件是否相同或包含任何差异是有用的。`–d`标志可用于打印差异：

```
$ tar -df archive.tar filename1 filename2 ...

```

例如：

```
$ tar -df archive.tar afile bfile
afile: Mod time differs
afile: Size differs

```

### 从归档中删除文件

我们可以使用`–delete`选项从给定的归档中删除文件。例如：

```
$ tar -f archive.tar --delete file1 file2 ..

```

让我们看另一个例子：

```
$ tar -tf archive.tar
filea
fileb
filec

```

或者，我们也可以使用以下语法：

```
$ tar --delete --file archive.tar [FILE LIST]

```

例如：

```
$ tar --delete --file archive.tar filea
$ tar -tf archive.tar
fileb
filec

```

### 使用 tar 归档进行压缩

`tar`命令只对文件进行归档，不对其进行压缩。因此，大多数人在处理 tarballs 时通常会添加某种形式的压缩。这显着减小了文件的大小。Tarballs 通常被压缩成以下格式之一：

+   `file.tar.gz`

+   `file.tar.bz2`

+   `file.tar.lzma`

+   `file.tar.lzo`

不同的`tar`标志用于指定不同的压缩格式。

+   `-j`用于 bunzip2

+   `-z`用于 gzip

+   `--lzma`用于 lzma

它们在以下特定于压缩的配方中有解释。

可以使用压缩格式而不明确指定特殊选项。`tar`可以通过查看输出或输入文件名的给定扩展名来进行压缩。为了使`tar`通过查看扩展名自动支持压缩，请使用`-a`或`--auto-compress`与`tar`。

### 从归档中排除一组文件

可以通过指定模式来排除一组文件不进行归档。使用`--exclude [PATTERN]`来排除与通配符模式匹配的文件。

例如，要排除所有`.txt`文件不进行归档，请使用：

```
$ tar -cf arch.tar * --exclude "*.txt"

```

### 提示

请注意，模式应该用双引号括起来。

还可以使用`-X`标志排除列表文件中提供的文件列表，如下所示：

```
$ cat list
filea
fileb

$ tar -cf arch.tar * -X list

```

现在排除了`filea`和`fileb`不进行归档。

### 排除版本控制目录

我们通常使用 tarballs 来分发源代码。大多数源代码是使用诸如 subversion、Git、mercurial、cvs 等版本控制系统进行维护的。版本控制下的代码目录将包含用于管理版本的特殊目录，如`.svn`或`.git`。但是，这些目录对代码本身并不需要，因此应该从源代码的 tarball 中删除。

为了在归档时排除与版本控制相关的文件和目录，请使用`tar`的`--exclude-vcs`选项。例如：

```
$ tar --exclude-vcs -czvvf source_code.tar.gz eye_of_gnome_svn

```

### 打印总字节数

如果我们可以打印复制到归档中的总字节数，有时会很有用。通过使用`--totals`选项在归档后打印复制的总字节数，如下所示：

```
$ tar -cf arc.tar * --exclude "*.txt" --totals
Total bytes written: 20480 (20KiB, 12MiB/s)

```

## 另请参阅

+   *使用 gunzip（gzip）进行压缩*，解释了 gzip 命令

+   *使用 bunzip（bzip2）进行压缩*，解释了 bzip2 命令

+   *使用 lzma 进行压缩*，解释了 lzma 命令

# 使用 cpio 进行归档

**cpio**是另一种类似于`tar`的归档格式。它用于将文件和目录存储在具有权限、所有权等属性的文件中。但是它并不像`tar`那样常用。然而，`cpio`似乎被用于 RPM 软件包归档、Linux 内核的 initramfs 文件等。本文将提供`cpio`的最小使用示例。

## 如何做...

`cpio`通过`stdin`接受输入文件名，并将归档写入`stdout`。我们必须将`stdout`重定向到文件以接收输出的`cpio`文件，如下所示：

创建测试文件：

```
$ touch file1 file2 file3

```

我们可以将测试文件归档如下：

```
$ echo file1 file2 file3 | cpio -ov > archive.cpio

```

在此命令中：

+   `-o`指定输出

+   `-v`用于打印已归档文件的列表

### 注意

通过使用`cpio`，我们还可以使用文件的绝对路径进行归档。`/usr/somedir`是一个绝对路径，因为它包含了从根目录(`/`)开始的完整路径。

相对路径不会以`/`开头，但它从当前目录开始。例如，`test/file`表示有一个名为`test`的目录，`file`在`test`目录内。

在提取时，`cpio`提取到绝对路径本身。但是在`tar`中，它会删除绝对路径中的`/`，并将其转换为相对路径。

为了列出`cpio`归档中的文件，请使用以下命令：

```
$ cpio -it < archive.cpio

```

此命令将列出给定`cpio`归档中的所有文件。它从`stdin`读取文件。在此命令中：

+   `-i`用于指定输入

+   `-t`用于列出

为了从`cpio`归档中提取文件，请使用：

```
$ cpio -id < archive.cpio

```

这里，`-d`用于提取。

它会在不提示的情况下覆盖文件。如果归档中存在绝对路径文件，它将替换该路径下的文件。它不会像`tar`那样在当前目录中提取文件。

# 使用 gunzip（gzip）进行压缩

**gzip**是 GNU/Linux 平台上常用的压缩格式。可用的实用程序包括`gzip`、`gunzip`和`zcat`，用于处理 gzip 压缩文件类型。`gzip`只能应用于文件。它不能归档目录和多个文件。因此我们使用`tar`归档并用`gzip`压缩。当多个文件作为输入时，它将产生多个单独压缩（`.gz`）文件。让我们看看如何使用`gzip`。

## 如何做...

为了使用`gzip`压缩文件，请使用以下命令：

```
$ gzip filename

$ ls
filename.gz

```

然后它将删除该文件并生成名为`filename.gz`的压缩文件。

提取`gzip`压缩文件如下：

```
$ gunzip filename.gz

```

它将删除`filename.gz`并生成`filename.gz`的未压缩版本。

为了列出压缩文件的属性，请使用：

```
$ gzip -l test.txt.gz
compressed        uncompressed  ratio uncompressed_name
 35                   6        -33.3% test.txt

```

`gzip`命令可以从`stdin`读取文件，并将压缩文件写入`stdout`。

从`stdin`读取并输出到`stdout`如下：

```
$ cat file | gzip -c > file.gz

```

`-c`选项用于指定输出到`stdout`。

我们可以为`gzip`指定压缩级别。使用`--fast`或`--best`选项分别提供低和高的压缩比。

## 还有更多...

`gzip`命令通常与其他命令一起使用。它还有高级选项来指定压缩比。让我们看看如何使用这些功能。

### 使用 tarball 的 gzip

我们通常使用`gzip`与 tarballs。可以通过在归档和提取时传递`-z`选项给`tar`命令来压缩 tarball。

您可以使用以下方法创建 gzipped tarballs：

+   **方法-1**

```
$ tar -czvvf archive.tar.gz [FILES]

```

或：

```
$ tar -cavvf archive.tar.gz [FILES]

```

`-a`选项指定应自动从扩展名检测压缩格式。

+   **方法-2**

首先，创建一个 tarball：

```
$ tar -cvvf archive.tar [FILES]

```

在打包后进行压缩如下：

```
$ gzip archive.tar

```

如果有许多文件（几百个）需要在一个 tarball 中进行归档并进行压缩，我们使用方法-2 并进行少量更改。使用`tar`将许多文件作为命令参数的问题是它只能从命令行接受有限数量的文件。为了解决这个问题，我们可以使用循环逐个添加文件并使用附加选项（`-r`）创建一个`tar`文件，如下所示：

```
FILE_LIST="file1  file2  file3  file4  file5"

for f in $FILE_LIST;
do
tar -rvf archive.tar $f 
done

gzip archive.tar
```

为了提取一个 gzipped tarball，使用以下命令：

+   -x 用于提取

+   `-z`用于 gzip 规范

或：

```
$ tar -xavvf archive.tar.gz -C extract_directory

```

在上述命令中，使用`-a`选项自动检测压缩格式。

### zcat-读取 gzipped 文件而不解压

`zcat`是一个命令，可用于从`.gz`文件中转储提取的文件到`stdout`，而无需手动提取它。`.gz`文件仍然与以前一样，但它将提取的文件转储到`stdout`，如下所示：

```
$ ls
test.gz

$ zcat test.gz
A test file
# file test contains a line "A test file"

$ ls
test.gz

```

### 压缩比

我们可以指定压缩比，可在 1 到 9 的范围内使用，其中：

+   1 是最低的，但速度最快

+   9 是最好的，但速度最慢

您还可以在之间指定比率，如下所示：

```
$ gzip -9 test.img

```

这将将文件压缩到最大。

## 另请参阅

+   *使用 tar 进行归档*，解释了 tar 命令

# 使用 bunzip（bzip）进行压缩

**bunzip2**是另一种与`gzip`非常相似的压缩技术。`bzip2`通常比`gzip`产生更小（更压缩）的文件。它随所有 Linux 发行版一起提供。让我们看看如何使用`bzip2`。

## 如何做...

为了使用`bzip2`进行压缩，使用：

```
$ bzip2 filename
$ ls
filename.bz2

```

然后它将删除文件并生成一个名为`filename.bzip2`的压缩文件。

提取一个 bzipped 文件如下：

```
$ bunzip2 filename.bz2

```

它将删除`filename.bz2`并产生一个未压缩版本的`filename`。

`bzip2`可以从`stdin`读取文件，并将压缩文件写入`stdout`。

为了从`stdin`读取并作为`stdout`读取，请使用：

```
$ cat file | bzip2 -c > file.tar.bz2

```

`-c`用于指定输出到`stdout`。

我们通常使用`bzip2`与 tarballs。可以通过在归档和提取时传递`-j`选项给`tar`命令来压缩 tarball。

可以通过以下方法创建一个 bzipped tarball：

+   **方法-1**

```
$ tar -cjvvf archive.tar.bz2 [FILES]

```

或：

```
$ tar -cavvf archive.tar.bz2 [FILES]

```

`-a`选项指定自动从扩展名检测压缩格式。

+   **方法-2**

首先创建 tarball：

```
$ tar -cvvf archive.tar [FILES]

```

在 tarball 后进行压缩：

```
$ bzip2 archive.tar

```

如果我们需要将数百个文件添加到存档中，则上述命令可能会失败。要解决此问题，请使用循环逐个使用`-r`选项将文件附加到存档中。请参阅食谱中的类似部分，*使用 gunzip（gzip）进行压缩*。

提取一个 bzipped tarball 如下：

```
$ tar -xjvvf archive.tar.bz2 -C extract_directory

```

在这个命令中：

+   `-x`用于提取

+   `-j`是用于`bzip2`规范

+   `-C`用于指定要提取文件的目录

或者，您可以使用以下命令：

```
$ tar -xavvf archive.tar.bz2 -C extract_directory

```

`-a`将自动检测压缩格式。

## 还有更多...

bunzip 有几个附加选项来执行不同的功能。让我们浏览其中的一些。

### 保留输入文件而不删除它们

在使用`bzip2`或`bunzip2`时，它将删除输入文件并生成一个压缩的输出文件。但是我们可以使用`-k`选项防止它删除输入文件。

例如：

```
$ bunzip2 test.bz2 -k
$ ls
test test.bz2

```

### 压缩比

我们可以指定压缩比，可在 1 到 9 的范围内使用（其中 1 是最少压缩，但速度快，9 是最高可能的压缩，但要慢得多）。

例如：

```
$ bzip2 -9 test.img

```

此命令提供最大压缩。

## 另请参阅

+   *使用 tar 进行归档*，解释了 tar 命令

# 使用 lzma 进行压缩

**lzma**与`gzip`或`bzip2`相比相对较新。`lzma`的压缩率比`gzip`或`bzip2`更高。由于大多数 Linux 发行版上没有预安装`lzma`，您可能需要使用软件包管理器进行安装。

## 如何做到...

为了使用`lzma`进行压缩，请使用以下命令：

```
$ lzma filename
$ ls
filename.lzma

```

这将删除文件并生成名为`filename.lzma`的压缩文件。

要提取`lzma`文件，请使用：

```
$ unlzma filename.lzma

```

这将删除`filename.lzma`并生成文件的未压缩版本。

`lzma`命令还可以从`stdin`读取文件并将压缩文件写入`stdout`。

为了从`stdin`读取并作为`stdout`读取，请使用：

```
$ cat file | lzma -c > file.lzma

```

`-c`用于指定输出到`stdout`。

我们通常使用`lzma`与 tarballs。可以通过在归档和提取时传递`--lzma`选项给`tar`命令来压缩 tarball。

有两种方法可以创建`lzma` tarball：

+   **方法-1**

```
$ tar -cvvf --lzma archive.tar.lzma [FILES]

```

或者：

```
$ tar -cavvf archive.tar.lzma [FILES]

```

`-a`选项指定自动从扩展名中检测压缩格式。

+   **方法-2**

首先，创建 tarball：

```
$ tar -cvvf archive.tar [FILES]

```

在 tarball 后进行压缩：

```
$ lzma archive.tar

```

如果我们需要将数百个文件添加到存档中，则上述命令可能会失败。为了解决这个问题，使用循环使用`-r`选项逐个将文件附加到存档中。请参阅配方中的类似部分，*使用 gunzip（gzip）进行压缩*。

## 还有更多...

让我们看看与`lzma`实用程序相关的其他选项

### 提取 lzma tarball

为了将使用`lzma`压缩的 tarball 提取到指定目录，请使用：

```
$ tar -xvvf --lzma archive.tar.lzma -C extract_directory

```

在这个命令中，`-x`用于提取。`--lzma`指定使用`lzma`来解压缩生成的文件。

或者，我们也可以使用：

```
$ tar -xavvf archive.tar.lzma -C extract_directory

```

`-a`选项指定自动从扩展名中检测压缩格式。

### 保留输入文件而不删除它们

在使用`lzma`或`unlzma`时，它将删除输入文件并生成输出文件。但是我们可以使用`-k`选项防止删除输入文件并保留它们。例如：

```
$ lzma test.bz2 -k
$ ls
test.bz2.lzma

```

### 压缩比

我们可以指定压缩比，可在 1 到 9 的范围内选择（1 表示最小压缩，但速度快，9 表示最大可能的压缩，但速度慢）。

您还可以按照以下方式指定比率：

```
$ lzma -9 test.img

```

此命令将文件压缩到最大。

## 另请参阅

+   *使用 tar 进行归档*，解释了 tar 命令

# 使用 zip 进行归档和压缩

ZIP 是许多平台上常用的压缩格式。在 Linux 平台上，它不像`gzip`或`bzip2`那样常用，但是互联网上的文件通常以这种格式保存。

## 如何做到...

为了使用 ZIP 进行归档，使用以下语法：

```
$ zip archive_name.zip [SOURCE FILES/DIRS]

```

例如：

```
$ zip file.zip file

```

在这里，将生成`file.zip`文件。

递归存档目录和文件如下：

```
$ zip -r archive.zip folder1 file2

```

在这个命令中，`-r`用于指定递归。

与`lzma`，`gzip`或`bzip2`不同，`zip`在归档后不会删除源文件。`zip`在这方面类似于`tar`，但`zip`可以压缩`tar`无法压缩的文件。但是，`zip`也添加了压缩。

为了提取 ZIP 文件中的文件和文件夹，请使用：

```
$ unzip file.zip

```

它将提取文件而不删除`filename.zip`（不像`unlzma`或`gunzip`）。

为了使用文件系统中的新文件更新存档中的文件，请使用`-u`标志：

```
$ zip file.zip -u newfile

```

通过使用`-d`来从压缩存档中删除文件，如下所示：

```
$ zip -d arc.zip file.txt

```

为了列出存档中的文件，请使用：

```
$ unzip -l archive.zip

```

# squashfs-重压缩文件系统

**squashfs**是一种基于重压缩的只读文件系统，能够将 2 到 3GB 的数据压缩到 700MB 的文件中。您是否曾经想过 Linux Live CD 是如何工作的？当启动 Live CD 时，它加载完整的 Linux 环境。Linux Live CD 使用名为 squashfs 的只读压缩文件系统。它将根文件系统保留在一个压缩的文件系统文件中。它可以进行回环挂载并访问文件。因此，当进程需要一些文件时，它们会被解压缩并加载到 RAM 中并被使用。当构建自定义的实时操作系统或需要保持文件高度压缩并且无需完全提取文件时，了解 squashfs 可能是有用的。对于提取大型压缩文件，需要很长时间。但是，如果文件进行回环挂载，它将非常快，因为只有在出现文件请求时才会解压缩压缩文件的所需部分。在常规解压缩中，首先解压缩所有数据。让我们看看如何使用 squashfs。

## 准备工作

如果您有 Ubuntu CD，只需在`CDRom ROOT/casper/filesystem.squashfs`中找到`.squashfs`文件。`squashfs`内部使用压缩算法，如`gzip`和`lzma`。`squashfs`支持在所有最新的 Linux 发行版中都可用。但是，为了创建`squashfs`文件，需要从软件包管理器安装额外的软件包**squashfs-tools**。

## 如何做...

为了通过添加源目录和文件创建`squashfs`文件，请使用：

```
$ mksquashfs SOURCES compressedfs.squashfs

```

源可以是通配符、文件或文件夹路径。

例如：

```
$ sudo mksquashfs /etc test.squashfs
Parallel mksquashfs: Using 2 processors
Creating 4.0 filesystem on test.squashfs, block size 131072.
[=======================================] 1867/1867 100%

More details will be printed on terminal. They are limited to save space

```

为了将`squashfs`文件挂载到挂载点，使用回环挂载如下：

```
# mkdir /mnt/squash
# mount -o loop compressedfs.squashfs /mnt/squash

```

您可以通过访问`/mnt/squashfs`来复制内容。

## 还有更多...

可以通过指定附加参数来创建`squashfs`文件系统。让我们看看附加选项。

### 在创建 squashfs 文件时排除文件

在创建`squashfs`文件时，可以使用通配符指定要排除的文件或文件模式的列表。

通过使用`-e`选项，可以排除作为命令行参数指定的文件列表。例如：

```
$ sudo mksquashfs /etc test.squashfs -e /etc/passwd /etc/shadow

```

`-e`选项用于排除`passwd`和`shadow`文件。

也可以使用`-ef`指定要排除的文件列表。

```
$ cat excludelist
/etc/passwd
/etc/shadow

$ sudo mksquashfs /etc test.squashfs -ef excludelist

```

如果我们想在排除列表中支持通配符，请使用`-wildcard`作为参数。

# 加密工具和哈希

加密技术主要用于保护数据免受未经授权的访问。有许多可用的算法，我们使用一组常见的标准算法。在 Linux 环境中有一些可用的工具用于执行加密和解密。有时我们使用加密算法哈希来验证数据完整性。本节将介绍一些常用的加密工具和这些工具可以处理的一般算法。

## 如何做...

让我们看看如何使用诸如 crypt、gpg、base64、md5sum、sha1sum 和 openssl 等工具：

+   **crypt**

`crypt`命令是一个简单的加密实用程序，它从`stdin`获取文件和密码作为输入，并将加密数据输出到`stdout`。

```
$ crypt <input_file> output_file
Enter passphrase:

```

它将交互式地要求输入密码。我们也可以通过命令行参数提供密码。

```
$ crypt PASSPHRASE < input_file > encrypted_file

```

要解密文件，请使用：

```
$ crypt PASSPHRASE -d < encrypted_file > output_file

```

+   **gpg（GNU 隐私保护）**

gpg（GNU 隐私保护）是一种广泛使用的加密方案，用于通过密钥签名技术保护文件，使得只有认证目标才能访问数据。gpg 签名非常有名。gpg 的详细信息超出了本书的范围。在这里，我们可以学习如何加密和解密文件。

要使用`gpg`加密文件，请使用：

```
$ gpg -c filename

```

此命令交互式地读取密码并生成`filename.gpg`。

要解密`gpg`文件，请使用：

```
$ gpg filename.gpg

```

此命令读取密码并解密文件。

+   **Base64**

Base64 是一组类似的编码方案，通过将二进制数据转换为基 64 表示形式的 ASCII 字符串格式来表示二进制数据。`base64`命令可用于编码和解码 Base64 字符串。

为了将二进制文件编码为 Base64 格式，请使用：

```
$ base64 filename > outputfile

```

或：

```
$ cat file | base64 > outputfile

```

它可以从`stdin`读取。

按照以下方式解码 Base64 数据：

```
$ base64 -d file > outputfile

```

或：

```
$ cat base64_file | base64 -d > outputfile

```

+   **md5sum** 和 **sha1sum**

**md5sum** 和 **sha1sum** 是单向哈希算法，无法逆转为原始数据。通常用于验证数据的完整性或从给定数据生成唯一密钥。对于每个文件，它通过分析其内容生成一个唯一密钥。

```
$ md5sum file
8503063d5488c3080d4800ff50850dc9  file

$ sha1sum file
1ba02b66e2e557fede8f61b7df282cd0a27b816b  file

```

这些类型的哈希适合于存储密码。密码以其哈希形式存储。当用户要进行身份验证时，密码被读取并转换为哈希。然后将哈希与已存储的哈希进行比较。如果它们相同，密码得到验证并提供访问权限，否则拒绝。存储原始密码字符串是有风险的，并会暴露密码的安全风险。

+   **类似影子的哈希（盐哈希）**

让我们看看如何生成类似影子的盐哈希密码。

Linux 中的用户密码以其哈希形式存储在`/etc/shadow`文件中。`/etc/shadow`中的典型行看起来像这样：

```
test:$6$fG4eWdUi$ohTKOlEUzNk77.4S8MrYe07NTRV4M3LrJnZP9p.qc1bR5c.EcOruzPXfEu1uloBFUa18ENRH7F70zhodas3cR.:14790:0:99999:7:::
```

在这行中`$6$fG4eWdUi$ohTKOlEUzNk77.4S8MrYe07NTRV4M3LrJnZP9p.qc1bR5c.EcOruzPXfEu1uloBFUa18ENRH7F70zhodas3cR`是与其密码对应的影子哈希。

在某些情况下，我们可能需要编写关键的管理脚本，可能需要使用 shell 脚本手动编辑密码或添加用户。在这种情况下，我们必须生成一个影子密码字符串，并写一个类似上面的行到影子文件中。让我们看看如何使用`openssl`生成影子密码。

影子密码通常是盐密码。`SALT`是一个额外的字符串，用于混淆和加强加密。盐由随机位组成，用作生成密码的盐哈希的输入之一。

有关盐的更多细节，请参阅维基百科页面[`en.wikipedia.org/wiki/Salt_(cryptography)`](http://en.wikipedia.org/wiki/Salt_(cryptography))。

```
$ openssl passwd -1 -salt SALT_STRING PASSWORD
$1$SALT_STRING$323VkWkSLHuhbt1zkSsUG.

```

用随机字符串替换`SALT_STRING`，用要使用的密码替换`PASSWORD`。

# 使用 rsync 备份快照

备份数据是大多数系统管理员需要定期执行的操作。我们可能需要在 Web 服务器或远程位置备份数据。`rsync`是一个可以用于将文件和目录从一个位置同步到另一个位置的命令，同时最小化数据传输，使用文件差异计算和压缩。`rsync`相对于`cp`命令的优势在于`rsync`使用强大的差异算法。此外，它支持网络数据传输。在复制文件时，它会比较原始位置和目标位置的文件，并只复制更新的文件。它还支持压缩、加密等等。让我们看看如何使用`rsync`。

## 如何做到...

为了将源目录复制到目的地（创建镜像），使用以下命令：

```
$ rsync -av source_path destination_path

```

在这个命令中：

+   `-a`代表存档

+   `-v`（详细）在`stdout`上打印详细信息或进度

上述命令将递归地将所有文件从源路径复制到目标路径。我们可以指定远程或本地主机路径。

它可以是格式为`/home/slynux/data`，[slynux@192.168.0.6:/home/backups/data](http://slynux@192.168.0.6:/home/backups/data)，等等。

`/home/slynux/data`指定了在执行`rsync`命令的机器上的绝对路径。`slynux@192.168.0.6:/home/backups/data`指定了 IP 地址为`192.168.0.6`的机器上的路径为`/home/backups/data`，并以用户`slynux`登录。

为了将数据备份到远程服务器或主机，请使用：

```
$ rsync -av source_dir username@host:PATH

```

为了在目的地保持镜像，以规律的间隔运行相同的`rsync`命令。它只会将更改的文件复制到目的地。

从远程主机恢复数据到`localhost`如下：

```
$ rsync -av username@host:PATH destination

```

`rsync`命令使用 SSH 连接到另一台远程计算机。在格式[user@host](http://user@host)中提供远程计算机地址，其中 user 是用户名，host 是附加到远程计算机的 IP 地址或域名。`PATH`是需要复制数据的绝对路径地址。`rsync`将像往常一样要求用户密码以进行 SSH 逻辑。这可以通过使用 SSH 密钥来自动化（避免用户密码探测）。

确保远程计算机上安装并运行 OpenSSH。

通过网络传输时压缩数据可以显著优化传输速度。我们可以使用`rsync`选项`-z`来指定在通过网络传输时压缩数据。例如：

```
$ rsync -avz source destination

```

### 注意

对于 PATH 格式，如果我们在源路径的末尾使用`/`，`rsync`将把指定在`source_path`中的末尾目录的内容复制到目的地。

如果源路径末尾没有`/`，`rsync`将复制该末尾目录本身到目的地。

例如，以下命令复制了`test`目录的内容：

```
$ rsync -av /home/test/ /home/backups

```

以下命令将`test`目录复制到目的地：

```
$ rsync -av /home/test /home/backups

```

### 注意

如果`destination_path`末尾有`/`，`rsync`将把源复制到目的地目录。

如果目的地路径末尾没有使用`/`，`rsync`将在目的地路径的末尾创建一个文件夹，该文件夹的名称与源目录类似，并将源复制到该目录中。

例如：

```
$ rsync -av /home/test /home/backups/

```

此命令将源（`/home/test`）复制到名为`backups`的现有文件夹。

```
$ rsync -av /home/test /home/backups

```

此命令通过创建目录将源（`/home/test`）复制到名为`backups`的目录中。

## 还有更多...

`rsync`命令具有几个额外的功能，可以使用其命令行选项来指定。让我们逐个了解。

### 在使用 rsync 进行归档时排除文件

有些文件在归档到远程位置时不需要更新。可以告诉`rsync`从当前操作中排除某些文件。文件可以通过两个选项来排除：

```
--exclude PATTERN
```

我们可以指定要排除的文件的通配符模式。例如：

```
$ rsync -avz /home/code/some_code /mnt/disk/backup/code --exclude "*.txt"

```

此命令排除了`.txt`文件的备份。

或者，我们可以通过提供一个文件列表来指定要排除的文件。

使用`--exclude-from FILEPATH`。

### 在更新 rsync 备份时删除不存在的文件

我们将文件存档为 tarball 并将 tarball 传输到远程备份位置。当我们需要更新备份数据时，我们再次创建一个 TAR 文件并将文件传输到备份位置。默认情况下，如果目的地不存在源文件，`rsync`不会从目的地删除文件。为了从目的地删除在源文件中不存在的文件，请使用`rsync --delete`选项：

```
$ rsync -avz SOURCE DESTINATION --delete

```

### 在间隔时间安排备份

您可以创建一个 cron 作业以规律的间隔安排备份。

示例如下：

```
$ crontab -e

```

添加以下行：

```
0 */10 * * * rsync -avz /home/code user@IP_ADDRESS:/home/backups

```

上述`crontab`条目安排每 10 小时执行一次`rsync`。

`*/10`是`crontab`语法的小时位置。`/10`指定每 10 小时执行一次备份。如果`*/10`写在分钟位置，它将每 10 分钟执行一次。

查看第九章中的*Scheduling with cron*配方，了解如何配置`crontab`。

# 基于 Git 的版本控制备份

人们在备份数据时使用不同的策略。差异备份比将整个源目录的副本复制到目标备份目录更有效，使用日期或当天时间的版本号。这会造成空间的浪费。我们只需要复制从第二次备份发生以来发生的文件更改。这称为增量备份。我们可以使用`rsync`等工具手动创建增量备份。但是恢复这种备份可能会很困难。保持和恢复更改的最佳方法是使用版本控制系统。它们在软件开发和代码维护中被广泛使用，因为编码经常发生变化。Git（GNU it）是一个非常著名且最有效的版本控制系统。让我们在非编程环境中使用 Git 备份常规文件。Git 可以通过您的发行版软件包管理器安装。它是由 Linus Torvalds 编写的。

## 准备工作

这是问题陈述：

我们有一个包含多个文件和子目录的目录。我们需要跟踪目录内容发生的更改并对其进行备份。如果数据损坏或丢失，我们必须能够恢复该数据的以前副本。我们需要定期将数据备份到远程机器。我们还需要在同一台机器（本地主机）的不同位置进行备份。让我们看看如何使用 Git 来实现它。

## 如何做…

在要备份的目录中使用：

```
$ cd /home/data/source

```

让源目录被跟踪。

设置并初始化远程备份目录。在远程机器上，创建备份目标目录：

```
$ mkdir -p /home/backups/backup.git

$ cd /home/backups/backup.git

$ git init --bare

```

以下步骤需要在源主机机器上执行：

1.  在源主机机器上将用户详细信息添加到 Git 中：

```
$ git config --global user.name  "Sarath Lakshman"
#Set user name to "Sarath Lakshman"

$ git config --global user.email slynux@slynux.com
# Set email to slynux@slynux.com

```

从主机机器初始化要备份的源目录。在要备份其文件的主机机器中的源目录中，执行以下命令：

```
$ git init
Initialized empty Git repository in /home/backups/backup.git/
# Initialize git repository

$ git commit --allow-empty -am "Init"
[master (root-commit) b595488] Init

```

1.  在源目录中，执行以下命令以添加远程 git 目录并同步备份：

```
$ git remote add origin user@remotehost:/home/backups/backup.git

$ git push origin master
Counting objects: 2, done.
Writing objects: 100% (2/2), 153 bytes, done.
Total 2 (delta 0), reused 0 (delta 0)
To user@remotehost:/home/backups/backup.git
 * [new branch]      master -> master

```

1.  添加或删除 Git 跟踪的文件。

以下命令将当前目录中的所有文件和文件夹添加到备份列表中：

```
$ git add *

```

我们可以有条件地将某些文件添加到备份列表中，方法如下：

```
$ git add *.txt
$ git add *.py

```

通过使用以下命令，我们可以删除不需要跟踪的文件和文件夹：

```
$ git rm file

```

它可以是一个文件夹，甚至是一个通配符，如下所示：

```
$ git rm *.txt

```

1.  检查点或标记备份点。

我们可以使用以下命令为备份标记检查点并附上消息：

```
$ git commit -m "Commit Message"

```

我们需要定期更新远程位置的备份。因此，设置一个 cron 作业（例如，每五个小时备份一次）。

创建包含以下行的 crontab 条目文件：

```
0 */5 * * *  /home/data/backup.sh

```

创建脚本`/home/data/backup.sh`如下：

```
#!/bin/ bash
cd /home/data/source
git add .
git commit -am "Commit - @ $(date)"
git push
```

现在我们已经设置好了备份系统。

1.  使用 Git 恢复数据。

为了查看所有备份版本，请使用：

```
$ git log

```

通过忽略任何最近的更改，将当前目录更新到上次备份。

+   要恢复到任何以前的状态或版本，请查看提交 ID，这是一个 32 个字符的十六进制字符串。使用提交 ID 和`git checkout`。

+   对于提交 ID 3131f9661ec1739f72c213ec5769bc0abefa85a9，它将是：

```
$ git checkout 3131f9661ec1739f72c213ec5769bc0abefa85a9
$ git commit -am "Restore @ $(date) commit ID: 3131f9661ec1739f72c213ec5769bc0abefa85a9"
$ git push

```

+   为了再次查看版本的详细信息，请使用：

```
$ git log

```

如果工作目录由于某些问题而损坏，我们需要使用远程位置的备份来修复目录。

然后我们可以按以下方式从远程位置重新创建内容：

```
$ git clone user@remotehost:/home/backups/backup.git

```

这将创建一个包含所有内容的目录备份。

# 使用 dd 克隆硬盘和磁盘

在处理硬盘和分区时，我们可能需要创建副本或备份完整分区，而不是复制所有内容（不仅是硬盘分区，还包括复制整个硬盘而不丢失任何信息，如引导记录、分区表等）。在这种情况下，我们可以使用`dd`命令。它可以用于克隆任何类型的磁盘，如硬盘、闪存驱动器、CD、DVD、软盘等。

## 准备工作

`dd`命令扩展到数据定义。由于不正确的使用会导致数据丢失，因此它被昵称为“数据销毁者”。在使用参数顺序时要小心。错误的参数可能导致整个数据丢失或变得无用。`dd`基本上是一个比特流复制器，它将整个比特流从磁盘复制到文件或从文件复制到磁盘。让我们看看如何使用`dd`。

## 如何做...

`dd`的语法如下：

```
$ dd if=SOURCE of=TARGET bs=BLOCK_SIZE count=COUNT

```

在这个命令中：

+   `if`代表输入文件或输入设备路径

+   `of`代表目标文件或目标设备路径

+   `bs`代表块大小（通常以 2 的幂给出，例如 512、1024、2048 等）。`COUNT`是要复制的块数（一个整数）。

总字节数=块大小*计数

`bs`和`count`是可选的。

通过指定`COUNT`，我们可以限制从输入文件复制到目标的字节数。如果未指定`COUNT`，`dd`将从输入文件复制，直到达到文件的结尾（EOF）标记。

为了将分区复制到文件中使用：

```
# dd if=/dev/sda1 of=sda1_partition.img

```

这里`/dev/sda1`是分区的设备路径。

使用备份还原分区如下：

```
# dd if=sda1_partition.img of=/dev/sda1

```

您应该小心处理参数`if`和`of`。不正确的使用可能会导致数据丢失。

通过将设备路径`/dev/sda1`更改为适当的设备路径，可以复制或还原任何磁盘。

为了永久删除分区中的所有数据，我们可以使用以下命令使`dd`将零写入分区：

```
# dd if=/dev/zero of=/dev/sda1

```

`/dev/zero`是一个字符设备。它总是返回无限的零'\0'字符。

将一个硬盘克隆到另一个相同大小的硬盘如下：

```
# dd if=/dev/sda  of=/dev/sdb

```

这里`/dev/sdb`是第二个硬盘。

为了使用 CD ROM（ISO 文件）的映像文件使用：

```
# dd if=/dev/cdrom of=cdrom.iso

```

## 还有更多...

当在使用`dd`生成的文件中创建文件系统时，我们可以将其挂载到挂载点。让我们看看如何处理它。

### 挂载映像文件

使用`dd`创建的任何文件映像都可以使用回环方法挂载。使用`mount`命令和`-o loop`。

```
# mkdir /mnt/mount_point
# mount -o loop file.img /mnt/mount_point

```

现在我们可以通过位置`/mnt/mount_point`访问映像文件的内容。

## 参见

+   *创建 ISO 文件，混合 ISO* 第三章解释了如何使用 dd 从 CD 创建 ISO 文件
