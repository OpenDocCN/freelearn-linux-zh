# 第五章。纠缠的网络？一点也不！

在本章中，我们将涵盖：

+   从网页下载

+   将网页下载为格式化的纯文本

+   cURL 入门

+   从命令行访问未读的 Gmail 邮件

+   从网站解析数据

+   创建图像爬虫和下载器

+   创建网页相册生成器

+   构建 Twitter 命令行客户端

+   定义具有 Web 后端的实用程序

+   在网站中查找损坏的链接

+   跟踪网站的更改

+   发布到网页并读取响应

# 介绍

网络正在成为技术的面孔。这是数据处理的中心访问点。虽然 shell 脚本不能像 PHP 等语言在 Web 上做的那样，但仍然有许多任务适合使用 shell 脚本。在本章中，我们将探讨一些可以用于解析网站内容、下载和获取数据、发送数据到表单以及自动执行网站使用任务和类似活动的方法。我们可以用几行脚本自动执行许多我们通过浏览器交互执行的活动。通过命令行实用程序访问 HTTP 协议提供的功能，我们可以编写适合解决大多数 Web 自动化实用程序的脚本。在阅读本章的食谱时玩得开心。

# 从网页下载

从给定 URL 下载文件或网页很简单。有几个命令行下载实用程序可用于执行此任务。

## 准备就绪

`wget`是一个文件下载命令行实用程序。它非常灵活，可以配置许多选项。

## 如何做...

可以使用`wget`下载网页或远程文件，如下所示：

```
$ wget URL

```

例如：

```
$ wget http://slynux.org 
--2010-08-01 07:51:20--  http://slynux.org/ 
Resolving slynux.org... 174.37.207.60 
Connecting to slynux.org|174.37.207.60|:80... connected. 
HTTP request sent, awaiting response... 200 OK 
Length: 15280 (15K) [text/html] 
Saving to: "index.html" 

100%[======================================>] 15,280      75.3K/s   in 0.2s 

2010-08-01 07:51:21 (75.3 KB/s) - "index.html" saved [15280/15280]

```

也可以指定多个下载 URL，如下所示：

```
$ wget URL1 URL2 URL3 ..

```

可以使用 URL 下载文件，如下所示：

```
$ wget ftp://example_domain.com/somefile.img

```

通常，文件以与 URL 相同的文件名下载，并且下载日志信息或进度被写入`stdout`。

您可以使用`-O`选项指定输出文件名。如果指定的文件名已经存在，它将首先被截断，然后下载的文件将被写入指定的文件。

您还可以使用`-o`选项指定不同的日志文件路径，而不是将日志打印到`stdout`，如下所示：

```
$ wget ftp://example_domain.com/somefile.img -O dloaded_file.img -o log

```

通过使用上述命令，屏幕上将不会打印任何内容。日志或进度将被写入`log`，输出文件将是`dloaded_file.img`。

由于不稳定的互联网连接可能会导致下载中断，因此我们可以使用尝试次数作为参数，以便一旦中断，实用程序将在放弃之前重试下载那么多次。

为了指定尝试次数，请使用`-t`标志，如下所示：

```
$ wget -t 5 URL

```

## 还有更多...

`wget`实用程序有几个额外的选项，可以在不同的问题领域下使用。让我们来看看其中的一些。

### 限速下载

当我们有限的互联网下行带宽和许多应用程序共享互联网连接时，如果给定一个大文件进行下载，它将吸取所有带宽，可能导致其他进程因带宽不足而挨饿。`wget`命令带有一个内置选项，可以指定下载作业可以拥有的最大带宽限制。因此，所有应用程序可以同时平稳运行。

我们可以使用`--limit-rate`参数限制`wget`的速度，如下所示：

```
$ wget  --limit-rate 20k http://example.com/file.iso

```

在这个命令中，`k`（千字节）和`m`（兆字节）指定了速度限制。

我们还可以指定下载的最大配额。当配额超过时，它将停止。在下载多个受总下载大小限制的文件时很有用。这对于防止下载意外使用太多磁盘空间很有用。

使用`--quota`或`-Q`如下：

```
$ wget -Q 100m http://example.com/file1 http://example.com/file2

```

### 恢复下载并继续

如果使用`wget`下载在完成之前中断，我们可以使用`-c`选项恢复下载，从我们离开的地方继续下载，如下所示：

```
$ wget -c URL

```

### 使用 cURL 进行下载

cURL 是另一个高级命令行实用程序。它比 wget 更强大。

cURL 可以用于下载如下：

```
$ curl http://slynux.org > index.html

```

与 wget 不同，curl 将下载的数据写入标准输出（stdout）而不是文件。因此，我们必须使用重定向运算符将数据从 stdout 重定向到文件。

### 复制完整的网站（镜像）

wget 有一个选项，可以通过递归收集网页中的所有 URL 链接并像爬虫一样下载所有网页，从而下载完整的网站。因此，我们可以完全下载网站的所有页面。

为了下载页面，请使用--mirror 选项如下：

```
$ wget --mirror exampledomain.com

```

或使用：

```
$ wget -r -N -l DEPTH URL

```

-l 指定网页的深度为级别。这意味着它只会遍历那么多级别。它与-r（递归）一起使用。-N 参数用于为文件启用时间戳。URL 是需要启动下载的网站的基本 URL。

### 使用 HTTP 或 FTP 身份验证访问页面

有些网页需要对 HTTP 或 FTP URL 进行身份验证。这可以通过使用--user 和--password 参数来提供：

```
$ wget –-user username –-password pass URL

```

还可以在不指定内联密码的情况下请求密码。为此，请使用--ask-password 而不是--password 参数。

# 将网页作为格式化纯文本下载

网页是包含一系列 HTML 标记以及其他元素（如 JavaScript、CSS 等）的 HTML 页面。但是 HTML 标记定义了网页的基础。在查找特定内容时，我们可能需要解析网页中的数据，这是 Bash 脚本可以帮助我们的地方。当我们下载一个网页时，我们会收到一个 HTML 文件。为了查看格式化的数据，它应该在 Web 浏览器中查看。然而，在大多数情况下，解析格式化的文本文档将比解析 HTML 数据更容易。因此，如果我们可以获得一个与在 Web 浏览器上看到的网页类似的格式化文本的文本文件，那将更有用，并且可以节省大量去除 HTML 标记所需的工作。Lynx 是一个有趣的命令行 Web 浏览器。我们实际上可以从 Lynx 获取网页作为纯文本格式化输出。让我们看看如何做到这一点。

## 如何做…

让我们使用 lynx 命令的--dump 标志将网页视图以 ASCII 字符表示形式下载到文本文件中：

```
$ lynx -dump URL > webpage_as_text.txt

```

该命令还将所有超链接（<a href="link">）单独列在文本输出的页脚下的**References**标题下。这将帮助我们避免使用正则表达式单独解析链接。

例如：

```
$ lynx -dump http://google.com > plain_text_page.txt

```

您可以使用 cat 命令查看文本的纯文本版本如下：

```
$ cat plain_text_page.txt

```

# cURL 入门

cURL 是一个强大的实用程序，支持包括 HTTP、HTTPS、FTP 等在内的许多协议。它支持许多功能，包括 POST、cookie、身份验证、从指定偏移量下载部分文件、引用、用户代理字符串、额外标头、限制速度、最大文件大小、进度条等。cURL 对于我们想要玩转自动化网页使用序列并检索数据时非常有用。这个配方是 cURL 最重要的功能列表。

## 准备工作

cURL 不会默认随主要 Linux 发行版一起提供，因此您可能需要使用软件包管理器安装它。默认情况下，大多数发行版都附带 wget。

cURL 通常将下载的文件转储到 stdout，并将进度信息转储到 stderr。为了避免显示进度信息，我们总是使用--silent 选项。

## 如何做…

curl 命令可用于执行不同的活动，如下载、发送不同的 HTTP 请求、指定 HTTP 标头等。让我们看看如何使用 cURL 执行不同的任务。

```
$ curl URL --silent

```

上述命令将下载的文件转储到终端（下载的数据写入 stdout）。

`--silent`选项用于防止`curl`命令显示进度信息。如果需要进度信息，请删除`--silent`。

```
$ curl URL –-silent -O

```

使用`-O`选项将下载的数据写入文件，文件名从 URL 中解析而不是写入标准输出。

例如：

```
$ curl http://slynux.org/index.html --silent -O

```

将创建`index.html`。

它将网页或文件写入与 URL 中的文件名相同的文件，而不是写入`stdout`。如果 URL 中没有文件名，将产生错误。因此，请确保 URL 是指向远程文件的 URL。`curl http://slynux.org -O --silent`将显示错误，因为无法从 URL 中解析文件名。

```
$ curl URL –-silent -o new_filename

```

`-o`选项用于下载文件并写入指定的文件名。

为了在下载时显示`#`进度条，使用`--progress`而不是`--silent`。

```
$ curl http://slynux.org -o index.html --progress
################################## 100.0% 

```

## 还有更多...

在前面的部分中，我们已经学习了如何下载文件并将 HTML 页面转储到终端。cURL 还有一些高级选项。让我们更深入地了解 cURL。

### 继续/恢复下载

cURL 具有高级的恢复下载功能，可以在给定的偏移量继续下载，而`wget`不具备这个功能。它可以通过指定偏移量来下载文件的部分。

```
$ curl URL/file -C offset

```

偏移是以字节为单位的整数值。

如果我们想要恢复下载文件，cURL 不需要我们知道确切的字节偏移量。如果要 cURL 找出正确的恢复点，请使用`-C -`选项，就像这样：

```
$ curl -C - URL

```

cURL 将自动找出重新启动指定文件的下载位置。

### 使用 cURL 设置引用字符串

引用者是 HTTP 头中的一个字符串，用于标识用户到达当前网页的页面。当用户从网页 A 点击链接到达网页 B 时，页面 B 中的引用头字符串将包含页面 A 的 URL。

一些动态页面在返回 HTML 数据之前会检查引用字符串。例如，当用户通过在 Google 上搜索导航到网站时，网页会显示一个附加了 Google 标志的页面，当他们通过手动输入 URL 导航到网页时，会显示不同的页面。

网页可以编写一个条件，如果引用者是[www.google.com](http://www.google.com)，则返回一个 Google 页面，否则返回一个不同的页面。

您可以使用`curl`命令的`--referer`选项指定引用字符串如下：

```
$ curl –-referer Referer_URL target_URL

```

例如：

```
$ curl –-referer http://google.com http://slynux.org

```

### 使用 cURL 的 cookies

使用`curl`我们可以指定并存储在 HTTP 操作期间遇到的 cookies。

为了指定 cookies，使用`--cookie "COOKIES"`选项。

Cookies 应该提供为`name=value`。多个 cookies 应该用分号“;”分隔。例如：

```
$ curl http://example.com –-cookie "user=slynux;pass=hack"

```

为了指定存储遇到的 cookies 的文件，使用`--cookie-jar`选项。例如：

```
$ curl URL –-cookie-jar cookie_file

```

### 使用 cURL 设置用户代理字符串

一些检查用户代理的网页如果没有指定用户代理就无法工作。您可能已经注意到，某些网站只在 Internet Explorer（IE）中运行良好。如果使用不同的浏览器，网站将显示一个消息，表示只能在 IE 上运行。这是因为网站检查用户代理。您可以使用`curl`将用户代理设置为 IE，并查看在这种情况下返回不同的网页。

使用 cURL 可以使用`--user-agent`或`-A`来设置如下：

```
$ curl URL –-user-agent "Mozilla/5.0"

```

可以使用 cURL 传递附加的标头。使用`-H "Header"`传递多个附加标头。例如：

```
$ curl -H "Host: www.slynux.org" -H "Accept-language: en" URL

```

### 在 cURL 上指定带宽限制

当可用带宽有限且多个用户共享互联网时，为了平稳地共享带宽，我们可以通过使用`--limit-rate`选项从`curl`限制下载速率到指定的限制。

```
$ curl URL --limit-rate 20k

```

在这个命令中，`k`（千字节）和`m`（兆字节）指定了下载速率限制。

### 指定最大下载大小

可以使用`--max-filesize`选项指定 cURL 的最大下载文件大小如下：

```
$ curl URL --max-filesize bytes

```

如果文件大小超过，则返回非零退出代码。如果成功，则返回零。

### 使用 cURL 进行身份验证

可以使用 cURL 和`-u`参数进行 HTTP 身份验证或 FTP 身份验证。

可以使用`-u username:password`指定用户名和密码。也可以不提供密码，这样在执行时会提示输入密码。

如果您希望提示输入密码，可以仅使用`-u username`。例如：

```
$ curl -u user:pass http://test_auth.com

```

为了提示输入密码，请使用：

```
$ curl -u user http://test_auth.com 

```

### 打印响应标头，不包括数据

仅打印响应标头非常有用，可以应用许多检查或统计。例如，要检查页面是否可访问，我们不需要下载整个页面内容。只需读取 HTTP 响应标头即可用于识别页面是否可用。

检查 HTTP 标头的一个示例用例是在下载之前检查文件大小。我们可以检查 HTTP 标头中的`Content-Length`参数以找出文件的长度。还可以从标头中检索到几个有用的参数。`Last-Modified`参数使我们能够知道远程文件的最后修改时间。

使用`curl`的`–I`或`–head`选项仅转储 HTTP 标头而不下载远程文件。例如：

```
$ curl -I http://slynux.org
HTTP/1.1 200 OK 
Date: Sun, 01 Aug 2010 05:08:09 GMT 
Server: Apache/1.3.42 (Unix) mod_gzip/1.3.26.1a mod_log_bytes/1.2 mod_bwlimited/1.4 mod_auth_passthrough/1.8 FrontPage/5.0.2.2635 mod_ssl/2.8.31 OpenSSL/0.9.7a 
Last-Modified: Thu, 19 Jul 2007 09:00:58 GMT 
ETag: "17787f3-3bb0-469f284a" 
Accept-Ranges: bytes 
Content-Length: 15280 
Connection: close 
Content-Type: text/html

```

## 参见

+   *发布到网页并读取响应*

# 从命令行访问 Gmail

Gmail 是谷歌提供的广泛使用的免费电子邮件服务[: http://mail.google.com/](http://: http://mail.google.com/)。 Gmail 允许您通过经过身份验证的 RSS 订阅来阅读邮件。我们可以解析 RSS 订阅，其中包括发件人的姓名和主题为电子邮件。这将有助于在不打开网络浏览器的情况下查看收件箱中的未读邮件。

## 如何做...

让我们通过 shell 脚本来解析 Gmail 的 RSS 订阅以显示未读邮件：

```
#!/bin/bash
Filename: fetch_gmail.sh
#Description: Fetch gmail tool

username="PUT_USERNAME_HERE"
password="PUT_PASSWORD_HERE"

SHOW_COUNT=5 # No of recent unread mails to be shown

echo

curl  -u $username:$password --silent "https://mail.google.com/mail/feed/atom" | \
tr -d '\n' | sed 's:</entry>:\n:g' |\
 sed 's/.*<title>\(.*\)<\/title.*<author><name>\([^<]*\)<\/name><email>\([^<]*\).*/Author: \2 [\3] \nSubject: \1\n/' | \
head -n $(( $SHOW_COUNT * 3 ))
```

输出将如下所示：

```
$ ./fetch_gmail.sh
Author: SLYNUX [ slynux@slynux.com ]
Subject: Book release - 2

Author: SLYNUX [ slynux@slynux.com ]
Subject: Book release - 1
.
… 5 entries

```

## 它是如何工作的...

该脚本使用 cURL 通过用户身份验证下载 RSS 订阅。用户身份验证由`-u username:password`参数提供。您可以使用`-u user`而不提供密码。然后在执行 cURL 时，它将交互式地要求输入密码。

在这里，我们可以将管道命令拆分为不同的块，以说明它们的工作原理。

`tr -d '\n'`删除换行符，以便我们使用`\n`作为分隔符重构每个邮件条目。`sed 's:</entry>:\n:g'`将每个`</entry>`替换为换行符，以便每个邮件条目都由换行符分隔，因此可以逐个解析邮件。查看[`mail.google.com/mail/feed/atom`](https://mail.google.com/mail/feed/atom)的源代码，了解 RSS 订阅中使用的 XML 标记。`<entry> TAGS </entry>`对应于单个邮件条目。

下一个脚本块如下：

```
 sed 's/.*<title>\(.*\)<\/title.*<author><name>\([^<]*\)<\/name><email>\([^<]*\).*/Author: \2 [\3] \nSubject: \1\n/'
```

此脚本使用`<title>\(.*\)<\/title`匹配子字符串标题，使用`<author><name>\([^<]*\)<\/name>`匹配发件人姓名，使用`<email>\([^<]*\)`匹配电子邮件。然后使用反向引用如下：

+   `Author: \2 [\3] \nSubject: \1\n`用于以易于阅读的格式替换邮件的条目。`\1`对应于第一个子字符串匹配，`\2`对应于第二个子字符串匹配，依此类推。

+   `SHOW_COUNT=5`变量用于在终端上打印未读邮件条目的数量。

+   `head`用于仅显示来自第一行的`SHOW_COUNT*3`行。 `SHOW_COUNT`被使用三次，以便显示输出的三行。

## 参见

+   *cURL 入门*，解释了 curl 命令

+   *基本的 sed 入门*第四章，解释了 sed 命令

# 从网站解析数据

通过消除不必要的细节，从网页中解析数据通常很有用。`sed`和`awk`是我们将用于此任务的主要工具。您可能已经在上一章的 grep 示例中看到了一个访问排名列表；它是通过解析网站页面[`www.johntorres.net/BoxOfficefemaleList.html`](http://www.johntorres.net/BoxOfficefemaleList.html)生成的。

让我们看看如何使用文本处理工具解析相同的数据。

## 如何做...

让我们通过用于解析女演员详情的命令序列：

```
$ lynx -dump http://www.johntorres.net/BoxOfficefemaleList.html  | \ grep -o "Rank-.*" | \
sed 's/Rank-//; s/\[[0-9]\+\]//' | \
sort -nk 1 |\
 awk ' 
{
 for(i=3;i<=NF;i++){ $2=$2" "$i } 
 printf "%-4s %s\n", $1,$2 ; 
}' > actresslist.txt

```

输出将如下所示：

```
# Only 3 entries shown. All others omitted due to space limits
1   Keira Knightley 
2   Natalie Portman 
3   Monica Bellucci 

```

## 它是如何工作的...

Lynx 是一个命令行网页浏览器；它可以转储网站的文本版本，就像我们在网页浏览器中看到的那样，而不是显示原始代码。因此，它避免了删除 HTML 标记的工作。我们使用`sed`解析以 Rank 开头的行，如下所示：

```
sed 's/Rank-//; s/\[[0-9]\+\]//'
```

然后可以根据排名对这些行进行排序。这里使用`awk`来保持排名和名称之间的间距，通过指定宽度来使其统一。`%-4s`指定四个字符的宽度。除了第一个字段之外的所有字段都被连接在一起形成一个单个字符串`$2`。

## 另请参阅

+   *第四章的基本 sed 入门*，解释了 sed 命令

+   *第四章的基本 awk 入门*，解释了 awk 命令

+   *以格式化纯文本形式下载网页*，解释了 lynx 命令

# 图像爬虫和下载器

当我们需要下载出现在网页中的所有图像时，图像爬虫非常有用。我们可以使用脚本来解析图像文件并自动下载，而不是查看 HTML 源并选择所有图像。让我们看看如何做到这一点。

## 如何做...

让我们编写一个 Bash 脚本来爬取并从网页下载图像，如下所示：

```
#!/bin/bash
#Description: Images downloader
#Filename: img_downloader.sh

if [ $# -ne 3 ];
then
  echo "Usage: $0 URL -d DIRECTORY"
  exit -1
fi

for i in {1..4}
do
  case $1 in
  -d) shift; directory=$1; shift ;;
   *) url=${url:-$1}; shift;;
esac
done

mkdir -p $directory;
baseurl=$(echo $url | egrep -o "https?://[a-z.]+")

curl –s $url | egrep -o "<img src=[^>]*>" | sed 's/<img src=\"\([^"]*\).*/\1/g' > /tmp/$$.list

sed -i "s|^/|$baseurl/|" /tmp/$$.list

cd $directory;

while read filename;
do
  curl –s -O "$filename" --silent

done < /tmp/$$.list
```

一个示例用法如下：

```
$ ./img_downloader.sh http://www.flickr.com/search/?q=linux -d images

```

## 它是如何工作的...

上述图像下载器脚本解析 HTML 页面，除了`<img>`之外剥离所有标记，然后从`<img>`标记中解析`src="img/URL"`并将其下载到指定目录。此脚本接受网页 URL 和目标目录路径作为命令行参数。脚本的第一部分是解析命令行参数的一种巧妙方法。`[ $# -ne 3 ]`语句检查脚本的参数总数是否为三，否则退出并返回一个使用示例。

如果有 3 个参数，那么解析 URL 和目标目录。为了做到这一点，使用了一个巧妙的技巧：

```
for i in {1..4}
do 
 case $1 in
 -d) shift; directory=$1; shift ;;
 *) url=${url:-$1}; shift;;
esac
done

```

`for`循环迭代了四次（数字四没有特殊意义，只是为了运行`case`语句几次）。

`case`语句将评估第一个参数（`$1`），并匹配`-d`或任何其他检查的字符串参数。我们可以在格式中的任何位置放置`-d`参数，如下所示：

```
$ ./img_downloader.sh -d DIR URL

```

或：

```
$ ./img_downloader.sh URL -d DIR

```

`shift`用于移动参数，这样当调用`shift`时，`$1`将被赋值为`$2`，再次调用时，`$1=$3`，依此类推，因为它将`$1`移动到下一个参数。因此，我们可以通过`$1`本身评估所有参数。

当匹配`-d`（`-d)`）时，很明显下一个参数是目标目录的值。`*）`对应默认匹配。它将匹配除`-d`之外的任何内容。因此，在迭代时，`$1=""`或`$1=URL`在默认匹配中，我们需要取`$1=URL`避免`""`覆盖。因此我们使用`url=${url:-$1}`技巧。如果已经不是`""`，它将返回一个 URL 值，否则它将分配`$1`。

`egrep -o "<img src=[^>]*>"`将仅打印匹配的字符串，即包括其属性的`<img>`标记。`[^>]*`用于匹配除了结束`>`之外的所有字符，即`<img src="img/image.jpg" …. >`。

`sed 's/<img src=\"\([^"]*\).*/\1/g'`解析`src="img/url"`，以便可以从已解析的`<img>`标记中解析所有图像 URL。

有两种类型的图像源路径：相对和绝对。绝对路径包含以`http://`或`https://`开头的完整 URL。相对 URL 以`/`或`image_name`本身开头。

绝对 URL 的示例是：[`example.com/image.jpg`](http://example.com/image.jpg)

相对 URL 的示例是：`/image.jpg`

对于相对 URL，起始的`/`应该被替换为基本 URL，以将其转换为[`example.com/image.jpg`](http://example.com/image.jpg)。

为了进行转换，我们首先通过解析找出`baseurl sed`。

然后用`sed -i "s|^/|$baseurl/|" /tmp/$$.list`将起始的`/`替换为`baseurl sed`。

然后使用`while`循环逐行迭代列表，并使用`curl`下载 URL。使用`--silent`参数与`curl`一起，以避免在屏幕上打印其他进度消息。

## 另请参阅

+   *cURL 入门*，解释了 curl 命令

+   *基本的 sed 入门* 第四章 ，解释 sed 命令

+   *使用 grep 在文件中搜索和挖掘“文本”* 第四章 ，解释 grep 命令

# Web 相册生成器

Web 开发人员通常为网站设计照片相册页面，该页面包含页面上的许多图像缩略图。单击缩略图时，将显示图片的大版本。但是，当需要许多图像时，每次复制`<img>`标签，调整图像以创建缩略图，将它们放在 thumbs 目录中，测试链接等都是真正的障碍。这需要很多时间并且重复相同的任务。通过编写一个简单的 Bash 脚本，可以轻松自动化。通过编写脚本，我们可以在几秒钟内自动创建缩略图，将它们放在确切的目录中，并自动生成`<img>`标签的代码片段。这个配方将教你如何做到这一点。

## 准备工作

我们可以使用`for`循环执行此任务，该循环遍历当前目录中的每个图像。通常使用 Bash 实用程序，如`cat`和`convert`（image magick）。这些将生成一个 HTML 相册，使用所有图像，放在`index.html`中。为了使用`convert`，请确保已安装 Imagemagick。

## 如何做...

让我们编写一个 Bash 脚本来生成 HTML 相册页面：

```
#!/bin/bash
#Filename: generate_album.sh
#Description: Create a photo album using images in current directory

echo "Creating album.."
mkdir -p thumbs
cat <<EOF > index.html
<html>
<head>
<style>

body 
{ 
  width:470px;
  margin:auto;
  border: 1px dashed grey;
  padding:10px; 
} 

img
{ 
  margin:5px;
  border: 1px solid black;

} 
</style>
</head>
<body>
<center><h1> #Album title </h1></center>
<p>
EOF

for img in *.jpg;
do 
  convert "$img" -resize "100x" "thumbs/$img"
  echo "<a href=\"$img\" ><img src=\"thumbs/$img\" title=\"$img\" /></a>" >> index.html
done

cat <<EOF >> index.html

</p>
</body>
</html>
EOF 

echo Album generated to index.html
```

按以下方式运行脚本：

```
$ ./generate_album.sh
Creating album..
Album generated to index.html

```

## 工作原理...

脚本的初始部分是编写 HTML 页面的标题部分。

以下脚本将所有内容重定向到 EOF（不包括）到`index.html`：

```
cat <<EOF > index.html
contents...
EOF
```

标题包括 HTML 和样式表。

`for img in *.jpg;`将遍历每个文件的名称并执行操作。

`convert "$img" -resize "100x" "thumbs/$img"`将创建宽度为 100px 的图像作为缩略图。

以下语句将生成所需的`<img>`标签并将其附加到`index.html`：

```
echo "<a href=\"$img\" ><img src=\"thumbs/$img\" title=\"$img\" /></a>" >> index.html
```

最后，使用`cat`附加页脚 HTML 标记。

## 另请参阅

+   *玩转文件描述符和重定向* 第一章 ，解释 EOF 和 stdin 重定向。

# Twitter 命令行客户端

Twitter 是最热门的微博平台，也是在线社交媒体的最新热点。发推文和阅读推文很有趣。如果我们可以从命令行做这两件事呢？编写命令行 Twitter 客户端非常简单。Twitter 有 RSS feeds，因此我们可以利用它们。让我们看看如何做到这一点。

## 准备工作

我们可以使用 cURL 进行身份验证并发送 twitter 更新，以及下载 RSS feed 页面以解析 tweets。只需四行代码就可以做到。让我们来做吧。

## 如何做...

让我们编写一个 Bash 脚本，使用`curl`命令来操作 twitter API：

```
#!/bin/bash
#Filename: tweets.sh
#Description: Basic twitter client

USERNAME="PUT_USERNAME_HERE"
PASSWORD="PUT_PASSWORD_HERE"
COUNT="PUT_NO_OF_TWEETS"

if [[ "$1" != "read" ]] && [[ "$1" != "tweet" ]];
then 
  echo -e "Usage: $0 send status_message\n   OR\n      $0 read\n"
  exit -1;
fi

if [[ "$1" = "read" ]];
then 
  curl --silent -u $USERNAME:$PASSWORD  http://twitter.com/statuses/friends_timeline.rss | \
grep title | \
tail -n +2 | \
head -n $COUNT | \
  sed 's:.*<title>\([^<]*\).*:\n\1:'

elif [[ "$1" = "tweet" ]];
then 
  status=$( echo $@ | tr -d '"' | sed 's/.*tweet //')
  curl --silent -u $USERNAME:$PASSWORD -d status="$status" http://twitter.com/statuses/update.xml > /dev/null
  echo 'Tweeted :)'
fi
```

运行以下脚本：

```
$ ./tweets.sh tweet Thinking of writing a X version of wall command "#bash"
Tweeted :)

$ ./tweets.sh read
bot: A tweet line
t3rm1n4l: Thinking of writing a X version of wall command #bash

```

## 工作原理...

让我们通过将上述脚本分成两部分来看看它的工作。第一部分是关于阅读推文的。要阅读推文，脚本会从[`twitter.com/statuses/friends_timeline.rss`](http://twitter.com/statuses/friends_timeline.rss)下载 RSS 信息，并解析包含`<title>`标签的行。然后，它使用`sed`剥离`<title>`和`</title>`标签，以形成所需的推文文本。然后使用`COUNT`变量来使用`head`命令除了最近推文的数量之外的所有其他文本。使用`tail -n +2`来删除不必要的标题文本“Twitter: Timeline of friends”。

在发送推文部分，`curl`的`-d`状态参数用于使用 Twitter 的 API 发布数据到 Twitter：[`twitter.com/statuses/update.xml`](http://twitter.com/statuses/update.xml)。

在发送推文的情况下，脚本的`$1`将是推文。然后，为了获取状态，我们使用`$@`（脚本的所有参数的列表）并从中删除单词“tweet”。

## 另请参阅

+   *cURL 入门*，解释了 curl 命令

+   *头和尾-打印最后或前 10 行* 第三章，解释了头和尾命令

# 具有 Web 后端的定义实用程序

谷歌通过使用搜索查询`define:WORD`为任何单词提供 Web 定义。我们需要一个 GUI 网页浏览器来获取定义。但是，我们可以通过使用脚本来自动化并解析所需的定义。让我们看看如何做到这一点。

## 准备工作

我们可以使用`lynx`，`sed`，`awk`和`grep`来编写定义实用程序。

## 如何做...

让我们看看从 Google 搜索中获取定义的定义实用程序脚本的核心部分：

```
#!/bin/bash
#Filename: define.sh
#Description: A Google define: frontend

limit=0
if  [ ! $# -ge 1 ];
then
  echo -e "Usage: $0 WORD [-n No_of_definitions]\n"
  exit -1;
fi

if [ "$2" = "-n" ];
then
  limit=$3;
  let limit++
fi

word=$1

lynx -dump http://www.google.co.in/search?q=define:$word | \
awk '/Defini/,/Find defini/' | head -n -1 | sed 's:*:\n*:; s:^[ ]*::' | \
grep -v "[[0-9]]" | \
awk '{
if ( substr($0,1,1) == "*" )
{ sub("*",++count".") } ;
print
} ' >  /tmp/$$.txt

echo

if [ $limit -ge 1 ];
then

cat /tmp/$$.txt | sed -n "/¹\./, /${limit}/p" | head -n -1

else

cat /tmp/$$.txt;

fi
```

按以下方式运行脚本：

```
$ ./define.sh hack -n 2
1\. chop: cut with a hacking tool
2\. one who works hard at boring tasks

```

## 它是如何工作的...

我们将研究定义解析器的核心部分。Lynx 用于获取网页的纯文本版本。[`www.google.co.in/search?q=define:$word`](http://www.google.co.in/search?q=define:$word)是网页定义网页的 URL。然后我们缩小“网页上的定义”和“查找定义”之间的文本。所有的定义都出现在这些文本行之间（`awk '/Defini/,/Find defini/'`）。

`'s:*:\n*:'`用于将*替换为*和换行符，以便在每个定义之间插入换行符，`s:^[ ]*::`用于删除行首的额外空格。在 lynx 输出中，超链接标记为[数字]。这些行通过`grep -v`（反向匹配行选项）被移除。然后使用`awk`将出现在行首的*替换为数字，以便为每个定义分配一个序号。如果我们在脚本中读取了一个`-n`计数，它必须根据计数输出一些定义。因此，使用`awk`打印序号 1 到计数的定义（这样做更容易，因为我们用序号替换了*）。

## 另请参阅

+   *基本的 sed 入门* 第四章，解释了 sed 命令

+   *基本的 awk 入门* 第四章，解释了 awk 命令

+   *使用 grep 在文件中搜索和挖掘“文本”* 第四章，解释了 grep 命令

+   *将网页下载为格式化的纯文本*，解释了 lynx 命令

# 在网站中查找损坏的链接

我看到人们手动检查网站上的每个页面以查找损坏的链接。这仅适用于页面非常少的网站。当页面数量变得很多时，这将变得不可能。如果我们可以自动查找损坏的链接，那将变得非常容易。我们可以使用 HTTP 操作工具来查找损坏的链接。让我们看看如何做到这一点。

## 准备工作

为了识别链接并从链接中找到损坏的链接，我们可以使用`lynx`和`curl`。它有一个`-traversal`选项，它将递归访问网站中的页面，并构建网站中所有超链接的列表。我们可以使用 cURL 来验证每个链接是否损坏。

## 如何做...

让我们通过`curl`命令编写一个 Bash 脚本来查找网页上的损坏链接：

```
#!/bin/bash 
#Filename: find_broken.sh
#Description: Find broken links in a website

if [ $# -eq 2 ]; 
then 
  echo -e "$Usage $0 URL\n" 
  exit -1; 
fi 

echo Broken links: 

mkdir /tmp/$$.lynx 

cd /tmp/$$.lynx 

lynx -traversal $1 > /dev/null 
count=0; 

sort -u reject.dat > links.txt 

while read link; 
do 
  output=`curl -I $link -s | grep "HTTP/.*OK"`; 
  if [[ -z $output ]]; 
  then 
    echo $link; 
    let count++ 
  fi 

done < links.txt 

[ $count -eq 0 ] && echo No broken links found.
```

## 工作原理...

`lynx -traversal URL`将在工作目录中生成多个文件。其中包括一个名为`reject.dat`的文件，其中包含网站中的所有链接。使用`sort -u`来避免重复构建列表。然后我们遍历每个链接，并使用`curl -I`检查标题响应。如果标题包含第一行`HTTP/1.0 200 OK`作为响应，这意味着目标不是损坏的。所有其他响应对应于损坏的链接，并打印到`stdout`。

## 另请参阅

+   *以格式化纯文本形式下载网页*，解释了 lynx 命令

+   *cURL 入门*，解释了 curl 命令

# 跟踪网站的变化

跟踪网站的变化对于网页开发人员和用户非常有帮助。在间隔时间内手动检查网站非常困难和不切实际。因此，我们可以编写一个在重复间隔时间内运行的变化跟踪器。当发生变化时，它可以播放声音或发送通知。让我们看看如何编写一个基本的网站变化跟踪器。

## 准备工作

在 Bash 脚本中跟踪网页变化意味着在不同时间获取网站并使用`diff`命令进行差异。我们可以使用`curl`和`diff`来做到这一点。

## 如何做...

让我们通过组合不同的命令来编写一个 Bash 脚本来跟踪网页中的变化：

```
#!/bin/bash
#Filename: change_track.sh
#Desc: Script to track changes to webpage

if [ $# -eq 2 ];
then 
  echo -e "$Usage $0 URL\n"
  exit -1;
fi

first_time=0
# Not first time

if [ ! -e "last.html" ];
then
  first_time=1
  # Set it is first time run
fi

curl --silent $1 -o recent.html

if [ $first_time -ne 1 ];
then
  changes=$(diff -u last.html recent.html)
  if [ -n "$changes" ];
  then
    echo -e "Changes:\n"
    echo "$changes"
  else
    echo -e "\nWebsite has no changes"
  fi
else
  echo "[First run] Archiving.."

fi

cp recent.html last.html
```

让我们看看`track_changes.sh`脚本在网页发生变化和网页未发生变化时的输出：

+   首次运行：

```
$ ./track_changes.sh http://web.sarathlakshman.info/test.html
[First run] Archiving..

```

+   第二次运行：

```
$ ./track_changes.sh http://web.sarathlakshman.info/test.html
Website has no changes 

```

+   对网页进行更改后的第三次运行：

```
$ ./test.sh http://web.sarathlakshman.info/test_change/test.html 
Changes: 

--- last.html	2010-08-01 07:29:15.000000000 +0200 
+++ recent.html	2010-08-01 07:29:43.000000000 +0200 
@@ -1,3 +1,4 @@ 
<html>
+added line :)
<p>data</p>
</html>

```

## 工作原理...

该脚本通过`[！-e`last.html`]`检查脚本是否是第一次运行。如果`last.html`不存在，这意味着这是第一次，因此必须下载网页并将其复制为`last.html`。

如果不是第一次，它应该下载新副本（`recent.html`）并使用`diff`实用程序检查差异。如果有变化，它应该打印出变化，最后应该将`recent.html`复制到`last.html`。

## 另请参阅

+   *cURL 入门*，解释了 curl 命令

# 向网页提交并读取响应

POST 和 GET 是 HTTP 中用于向网站发送信息或检索信息的两种请求类型。在 GET 请求中，我们通过网页 URL 本身发送参数（名称-值对）。在 POST 的情况下，它不会附加在 URL 上。当需要提交表单时使用 POST。例如，需要提交用户名、密码和检索登录页面。

在编写基于网页检索的脚本时，POST 到页面的使用频率很高。让我们看看如何使用 POST。通过发送 POST 数据和检索输出来自动执行 HTTP GET 和 POST 请求是我们在编写从网站解析数据的 shell 脚本时练习的非常重要的任务。

## 准备工作

cURL 和`wget`都可以通过参数处理 POST 请求。它们作为名称-值对传递。

## 如何做...

让我们看看如何使用`curl`从真实网站进行 POST 和读取 HTML 响应：

```
$ curl URL -d "postvar=postdata2&postvar2=postdata2"

```

我们有一个网站（[`book.sarathlakshman.com/lsc/mlogs/`](http://book.sarathlakshman.com/lsc/mlogs/)），用于提交当前用户信息，如主机名和用户名。假设在网站的主页上有两个字段 HOSTNAME 和 USER，以及一个 SUBMIT 按钮。当用户输入主机名、用户名并单击 SUBMIT 按钮时，详细信息将存储在网站中。可以使用一行`curl`命令自动化此过程，通过自动化 POST 请求。如果查看网站源代码（使用 Web 浏览器的查看源代码选项），可以看到类似于以下代码的 HTML 表单定义：

```
<form action="http://book.sarathlakshman.com/lsc/mlogs/submit.php" method="post" >

<input type="text" name="host" value="HOSTNAME" >
<input type="text" name="user" value="USER" >
<input type="submit" >
</form>
```

在这里，[`book.sarathlakshman.com/lsc/mlogs/submit.php`](http://book.sarathlakshman.com/lsc/mlogs/submit.php)是目标 URL。当用户输入详细信息并单击提交按钮时，主机和用户输入将作为 POST 请求发送到`submit.php`，并且响应页面将返回到浏览器。

我们可以按照以下方式自动化 POST 请求：

```
$ curl http://book.sarathlakshman.com/lsc/mlogs/submit.php -d "host=test-host&user=slynux"
<html>
You have entered :
<p>HOST : test-host</p>
<p>USER : slynux</p>
<html>

```

现在`curl`返回响应页面。

`-d`是用于发布的参数。 `-d`的字符串参数类似于 GET 请求语义。 `var=value`对应关系应该由`&`分隔。

### 注意

`-d`参数应该总是用引号括起来。如果不使用引号，`&`会被 shell 解释为表示这应该是一个后台进程。

## 还有更多

让我们看看如何使用 cURL 和`wget`执行 POST。

### 在 curl 中进行 POST

您可以使用`-d`或`–data`在`curl`中发送 POST 数据，如下所示：

```
$ curl –-data "name=value" URL -o output.html

```

如果要发送多个变量，请用`&`分隔它们。请注意，当使用`&`时，名称-值对应该用引号括起来，否则 shell 将把`&`视为后台进程的特殊字符。例如：

```
$ curl -d "name1=val1&name2=val2" URL -o output.html

```

### 使用 wget 发送 POST 数据

您可以使用`wget`通过使用`-–post-data "string"`来发送 POST 数据。例如：

```
$ wget URL –post-data "name=value" -O output.html

```

使用与 cURL 相同的格式进行名称-值对。

## 另请参阅

+   *关于 cURL 的入门*，解释了 curl 命令

+   *从网页下载*解释了 wget 命令
