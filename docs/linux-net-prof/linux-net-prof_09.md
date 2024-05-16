# 第六章：*第六章*：Linux 上的 DNS 服务

**域名系统**（**DNS**）是当今信息社会的重要基础。技术社区中使用的一句谚语（以俳句格式表达）如下所示：

*这不是 DNS*

*绝对不可能是 DNS*

*这是 DNS*

这描述了比你想象的更多的技术问题，甚至涉及到广泛的互联网或云服务中断。这也很好地描述了问题是如何解决的，答案是：“根本问题总是 DNS。”这很好地说明了这项服务对当今几乎每个企业网络和公共互联网的几乎每个方面都是多么重要。

在本章中，我们将涵盖几个涉及 DNS 基础知识的主题，然后构建和最终排除 DNS 服务。我们将关注以下领域：

+   什么是 DNS？

+   两种主要的 DNS 服务器实现

+   常见的 DNS 实现

+   DNS 故障排除和侦察

然后，在介绍了 DNS 基础知识之后，我们将讨论以下两种全新的 DNS 实现，这两种实现正在迅速被采用：

+   DNS over **HyperText Transfer Protocol Secure** (**HTTPS**), known as **DoH**

+   **传输层安全**（**TLS**）上的 DNS，称为**DoT**

我们还将讨论**DNS 安全扩展**（**DNSSEC**）实现，该实现对 DNS 响应进行加密签名，以证明它们已经经过验证并且没有被篡改。

# 技术要求

在本章的示例中，您应该能够继续使用您现有的 Linux 主机或**虚拟机**（**VM**）。没有额外的要求。

# 什么是 DNS？

DNS 基本上是人们所需的东西和网络所需的东西之间的翻译者。大多数情况下，人们理解主机和服务的文本名称，例如`google.com`或`paypal.com`。然而，这些名称对底层网络并没有意义。DNS 所做的就是将那些“完全合格的主机名”（有人可能在应用程序中输入，比如他们在**开放系统互连**（**OSI**）第 7 层的浏览器中输入的）翻译成**Internet Protocol**（**IP**）地址，然后可以用来在 OSI 第 3 和第 4 层路由应用程序请求。

在相反的方向上，DNS 也可以将 IP 地址翻译成**完全合格的域名**（**FQDN**），使用所谓的**指针**（**PTR**）请求（用于 DNS PTR 记录）或“反向查找”。这对技术人员来说可能很重要，但这些请求并不像常见的人们运行他们的浏览器和其他应用程序那样经常见到。

# 两种主要的 DNS 服务器实现

DNS 在互联网上有一个庞大而复杂的基础设施（我们将在本节中涉及）。这由 13 个根名称服务器（每个都是可靠的服务器集群）、一组常用的名称服务器（例如我们在谷歌或 Cloudflare 使用的服务器）以及一系列注册商组成，他们将为您注册 DNS 域名，例如您的组织域名，收取一定费用。

然而，大部分情况下，大多数管理员都在处理他们组织的需求——与面向内部人员的内部 DNS 名称服务器或者面向互联网的外部 DNS 名称服务器进行工作。这两种用例将是本章重点讨论的内容。当我们构建这些示例时，您将看到谷歌或 Cloudflare DNS 基础设施，甚至根 DNS 服务器并没有那么不同。

## 组织的“内部”DNS 服务器（以及 DNS 概述）

组织部署的最常见的 DNS 服务是供其员工使用的**内部 DNS 服务器**。该服务器可能具有用于内部 DNS 解析的 DNS 记录的区域文件。该文件可以通过手动编辑区域文件或使用客户端的自动注册或**动态主机配置协议**（**DHCP**）租约自动填充。通常，这三种方法都会结合使用。

基本的请求流程很简单。客户端发出 DNS 请求。如果该请求是针对组织内部的主机，并且请求是发送到内部 DNS 服务器，则由于该本地 DNS 服务器上有该请求，DNS 响应将立即提供。

如果是针对外部主机的请求，那么事情会变得更加复杂 - 例如，让我们查询`www.example.com`。在开始之前，请注意以下图表显示了*最坏情况*，但几乎每一步都有缓存过程，通常允许跳过一步或多步：

![图 6.1 - 单个 DNS 请求可以变得多么复杂的令人眼花缭乱的概述](img/B16336_06_001.jpg)

图 6.1 - 单个 DNS 请求可以变得多么复杂的令人眼花缭乱的概述

这个过程看起来很复杂，但您会发现它进行得非常快，实际上在许多情况下有许多*逃生舱*可以让协议跳过许多这些步骤。让我们详细看看整个*最坏情况*的过程，如下所示：

1.  如果内部 DNS 服务器的 DNS 缓存中有条目，并且该条目的**生存时间**（**TTL**）尚未过期，则立即向客户端提供响应。同样，如果客户端请求的条目托管在区域文件中的服务器上，则立即向客户端提供答案。

1.  如果内部 DNS 服务器的缓存中没有条目，或者如果条目在缓存中但其 TTL 已过期，则内部服务器将请求转发给其上游提供者（通常称为**转发器**）以刷新该条目。

如果查询在转发器的缓存中，它将简单地返回答案。如果该服务器具有该域的权威名称服务器，它将简单地查询该主机（在过程中跳过到*步骤 5*）。

1.  如果转发器在缓存中没有该请求，它将向上游请求。在这种情况下，它可能会查询根名称服务器。这样做的目的是找到具有该域的实际条目（在区域文件中）的“权威名称服务器”。在这种情况下，查询是针对`.com`的根名称服务器进行的。

1.  根名称服务器不会返回实际答案，而是返回`.com`的权威名称服务器。

1.  转发器收到此响应后，更新其缓存以包含该名称服务器条目，然后对该服务器进行实际查询。

1.  `.com`的权威服务器返回`example.com`的权威 DNS 服务器。

1.  转发服务器然后向最终的权威名称服务器发出请求。

1.  `example.com`的权威名称服务器将实际查询的“答案”返回给转发器服务器。

1.  转发器名称服务器缓存该答案，然后将答复发送回您的内部名称服务器。

1.  您的内部 DNS 服务器还会缓存该答案，然后将其转发回客户端。

客户端将请求缓存在其本地缓存中，然后将所请求的信息（DNS 响应）传递给请求它的应用程序（也许是您的网络浏览器）。

同样，这个过程展示了最坏情况下的简单 DNS 请求和接收答案的过程。实际上，一旦服务器运行了一段时间，缓存会大大缩短这个过程。一旦进入稳定状态，大多数组织的内部 DNS 服务器将会缓存大部分请求，因此该过程会直接从*步骤 1*跳到*步骤 10*。此外，您的转发 DNS 服务器也会缓存，特别是它几乎不会查询根名称服务器；通常它也会缓存顶级域服务器（在这种情况下，`.com`的服务器）。

在这个描述中，我们还提到了“根名称服务器”的概念。这些是根或`.`区域的权威服务器。为了冗余，有 13 个根服务器，每个实际上都是一个可靠的服务器集群。

我们的内部 DNS 服务器需要启用哪些关键功能才能使所有这些工作？我们需要启用以下功能：

+   **DNS 递归**：这种模式依赖于 DNS 递归，即每个服务器依次向上级服务器发出客户端的 DNS 请求。如果内部服务器上未定义所请求的 DNS 条目，它需要获得转发这些请求的权限。

+   **转发器条目**：如果所请求的 DNS 条目不托管在内部服务器上，**内部 DNS 服务**（**iDNS**）请求将被转发到这些配置的 IP 地址，这些应该是两个或更多可靠的上游 DNS 服务器。这些上游服务器将依次缓存 DNS 条目，并在其 TTL 计时器到期时使其过期。在过去，人们会使用他们的**互联网服务提供商**（**ISP**）的 DNS 服务器作为转发器。在现代，更大的 DNS 提供商比您的 ISP 更可靠并提供更多功能。下面列出了一些常用的用作转发器的 DNS 服务（最常用的地址以粗体显示）！[](img/Table_011.jpg)

+   **缓存**：在一个大型组织中，通过增加内存可以大大提高 DNS 服务器的性能，这样可以进行更多的缓存，这意味着更多的请求可以直接从服务器的内存中提供服务。

+   **动态注册**：虽然服务器通常具有静态 IP 地址和静态 DNS 条目，但工作站通常会通过 DHCP 分配地址，当然也希望将这些工作站注册到 DNS 中。DNS 通常配置为允许动态注册这些主机，可以通过在分配地址时从 DHCP 中填充 DNS 或允许主机自行在 DNS 中注册（如**请求评论**（**RFC**）*2136*中所述）。

微软在他们的动态更新过程中实现了身份验证机制，这是最常见的地方。然而，在 Linux DNS（**伯克利互联网名称域**，或**BIND**）中也有这个选项。

+   **主机冗余**：几乎所有核心服务都受益于冗余。对于 DNS 来说，通常会有第二个 DNS 服务器。数据库通常是单向复制（从主服务器到辅助服务器），并使用区域文件中的序列号来确定何时进行复制，使用一种称为**区域传输**的复制过程。冗余对于应对各种系统故障至关重要，但同样重要的是它可以允许系统维护而不会中断服务。

有了内部 DNS 服务器，我们需要在配置中做哪些改变才能使 DNS 服务器为公共互联网提供区域服务？

## 面向互联网的 DNS 服务器

在面向互联网的 DNS 服务器的情况下，您很可能正在为一个或多个 DNS 区域实现权威 DNS 服务器。例如，在我们的参考图表（*图 6.1*）中，`example.com`的权威 DNS 服务器就是一个很好的例子。

在这个实现中，重点从内部服务器的性能和转发转移到了限制访问以实现最大安全性。这些是我们想要实现的限制：

+   限制递归：在我们概述的 DNS 模型中，这个服务器是“终点”，它直接回答它托管的区域的 DNS 请求。这个服务器永远不应该向上游查找以服务 DNS 请求。

+   缓存不太重要：如果您是一个组织，并且正在托管自己的公共 DNS 区域，那么您只需要足够的内存来缓存自己的区域。

+   主机冗余：再次，如果您正在托管自己的区域文件，添加第二台主机对您来说可能比添加缓存更重要。这为您的 DNS 服务提供了一些硬件冗余，这样您就可以在不中断服务的情况下对一台服务器进行维护。

+   限制区域传输：这是您想要实施的关键限制——您希望在收到单独的 DNS 查询时进行回答。互联网上的 DNS 客户端请求组织的所有条目没有充分的理由。区域传输旨在在冗余服务器之间维护您的区域，以便在编辑区域时将更改复制到群集中的其他服务器。

+   速率限制：DNS 服务器具有一种称为响应速率限制（RRL）的功能，它限制任何一个源可以查询该服务器的频率。为什么要实施这样的功能？

DNS 经常用于“欺骗”攻击。由于它基于用户数据报协议（UDP），没有建立会话的“握手”；它是一个简单的请求/响应协议，因此，如果您想攻击已知地址，您可以简单地使用目标作为请求者进行 DNS 查询，未经请求的答案将发送到该 IP。

这似乎不像是一次攻击，但如果您然后添加一个“乘数”（换句话说，如果您正在进行小型 DNS 请求并获得更大的响应，例如文本（TXT）记录，并且正在使用多个 DNS 服务器作为“反射器”），那么您发送到目标的带宽可能会很快增加。

这使得速率限制变得重要——您希望限制任何一个 IP 地址每秒进行少量相同的查询。这是一个合理的做法；鉴于 DNS 缓存的依赖性，任何一个 IP 地址在任何 5 分钟内不应该进行超过一到两个相同的请求，因为 5 分钟是任何 DNS 区域的最小 TTL。

启用速率限制的另一个原因是限制攻击者在 DNS 中进行侦察的能力——为常见的 DNS 名称进行数十甚至数百个请求，并编制您有效主机的列表，以便随后对它们进行攻击。

+   限制动态注册：动态注册当然不建议在大多数面向互联网的 DNS 服务器上。唯一的例外是任何提供动态 DNS（DDNS）注册作为服务的组织。这类公司包括 Dynu、DynDNS、FreeDNS 和 No-IP 等几家公司。鉴于这些公司的专业性质，它们各自都有自己的方法来保护其 DDNS 更新（通常涉及自定义代理和某种形式的身份验证）。直接使用 RFC 2136 对于面向互联网的 DNS 服务器来说根本无法保护。

通过实施内部 DNS 服务器的基础知识并开始为它们的各种用例进行安全设置，我们有哪些 DNS 应用程序可用于构建 DNS 基础设施？让我们在下一节中了解这一点。

# 常见的 DNS 实现

BIND，也称为 named（用于名称守护程序），是 Linux 中最常实现的 DNS 工具，可以说是最灵活和完整的，同时也是最难配置和排除故障的。不管好坏，它是您最有可能看到和在大多数组织中实施的服务。主要的两种实现用例在接下来的两个部分中进行了概述。

**DNS 伪装**（**dnsmasq**）是一种竞争的 DNS 服务器实现。它通常出现在网络设备上，因为它的占用空间很小，但也可以作为较小组织的良好 DNS 服务器。Dnsmasq 的主要优势包括其内置的**图形用户界面**（**GUI**），可用于报告，以及其与 DHCP 的集成（我们将在下一章中讨论），允许直接从 DHCP 数据库进行 DNS 注册。此外，Dnsmasq 实现了一种友好的方式来实现 DNS 阻止列表，这在 Pi-hole 应用程序中非常好地打包起来。如果你的家庭网络在其外围防火墙或**无线接入点**（**WAP**）上有一个 DNS 服务器，那么该 DNS 服务器很可能是 Dnsmasq。

在本章中，我们将专注于常用的 BIND（或命名）DNS 服务器。让我们开始构建我们的内部 DNS 服务器使用该应用程序。

## 基本安装：用于内部使用的 BIND

正如你所期望的，安装`bind`，Linux 上最流行的 DNS 服务器，就是这么简单：

```
$ sudo apt-get install –y bind9
```

查看`/etc/bind/named.conf`文件。在旧版本中，应用程序配置都在这一个庞大的配置文件中，但在新版本中，它只是由三个`include`行组成，如下面的代码片段所示：

```
include "/etc/bind/named.conf.options";
include "/etc/bind/named.conf.local";
include "/etc/bind/named.conf.default-zones";
```

编辑`/etc/bind/named.conf.options`，并添加以下选项—确保使用`sudo`因为你需要管理员权限来更改`bind`的任何配置文件：

+   允许来自本地子网列表的查询。在这个例子中，我们允许所有*RFC 1918*中的子网，但你应该将其限制在你的环境中拥有的子网。请注意，我们使用无类别子网掩码来最小化这一部分的条目数量。

+   定义监听端口（默认情况下是正确的）。

+   启用递归查询。

+   定义递归工作的 DNS 转发器列表。在这个例子中，我们将添加谷歌和 Cloudflare 作为 DNS 转发。

完成后，我们的配置文件应该看起来像这样。请注意，这确实是一个几乎是“普通语言”的配置—对于这些部分的含义没有任何神秘之处：

```
options {
  directory "/var/cache/bind";
  listen-on port 53 { localhost; };
  allow-query { localhost; 192.168.0.0/16; 10.0.0.0/8; 172.16.0.0/12; };
  forwarders { 8.8.8.8; 8.8.4.4; 1.1.1.1; };
  recursion yes;
}
```

接下来，编辑`/etc/bind/named.conf.local`，并添加服务器类型、区域和区域文件名。此外，允许指定子网上的工作站使用`allow-update`参数向 DNS 服务器注册其 DNS 记录，如下面的代码片段所示：

```
zone "coherentsecurity.com" IN {
  type master;
  file "coherentsecurity.com.zone";
  allow-update { 192.168.0.0/16; 10.0.0.0/8;172.16.0.0/12 };
};
```

`zone`文件本身，其中存储了所有的 DNS 记录，不在与这前两个`config`文件相同的位置。要编辑`zone`文件，编辑`/var/cache/bind/<zone file name>`—所以，在这个例子中，是`/var/cache/bind/coherentsecurity.com.zone`。你需要`sudo`权限来编辑这个文件。做出以下更改：

+   根据需要添加记录。

+   使用你的区域和域名服务器的 FQDN 更新`SOA`行。

+   如果需要，在`SOA`记录的最后一行更新`TTL`值—默认值是`86400`秒（24 小时）。这通常是一个很好的折衷方案，因为它有利于在多个服务器上缓存记录。但是，如果你正在进行任何 DNS 维护，你可能希望在维护前一天（即维护前 24 小时或更长时间）编辑文件，并将其缩短到 5 或 10 分钟，以避免由于缓存而延迟你的更改。

+   更新`ns`记录，它标识了你域的 DNS 服务器。

+   根据需要添加`A`记录—这些标识每个主机的 IP 地址。请注意，对于`A`记录，我们只使用每个主机的**通用名称**（**CN**），而不是包括域的 FQDN 名称。

完成后，我们的 DNS 区域文件应该看起来像这样：

![图 6.2 - 一个 DNS 区域文件示例](img/B16336_06_002.jpg)

图 6.2 - 一个 DNS 区域文件示例

正如我们之前讨论的，在内部 DNS 区域中，通常希望客户端在 DNS 中注册自己。这允许管理员通过名称而不是确定其 IP 地址来访问客户端。这是对`named.conf`文件（或更可能是适用的包含的子文件）的简单编辑。请注意，这要求我们添加`192.168.122.0/24`（定义整个子网）可能更常见。通常也会看到定义整个公司的企业“超网”，例如`10.0.0.0/8`或`192.168.0.0/16`，但出于安全原因，这通常不建议；您可能实际上不需要设备在*每个*子网中自动注册。

在适用的区域中，添加以下代码行：

```
acl dhcp-clients { 192.168.122.128/25; };
acl static-clients { 192.168.122.64/26; };
zone "coherentsecurity.com" {
    allow-update { dhcp-clients; static-clients; };
};
```

有一些脚本可以检查您的工作——一个用于基本配置和包含的文件，另一个用于区域。如果没有错误，`named-checkconf`将不返回任何文本，而`named-checkzone`将给出一些`OK`状态消息，如下所示。如果您运行这些并且没有看到错误，那么至少应该足够开始服务。请注意，`named-checkzone`命令在以下代码示例中换行到下一行。`bind`配置文件中的错误很常见——例如缺少分号。这些脚本将非常具体地指出发现的问题，但如果它们出现错误并且您需要更多信息，则这些命令的日志文件（`bind`本身的`bind`）是标准的`/var/log/syslog`文件，因此请在那里查找：

```
$ named-checkconf
$ named-checkzone coherentsecurity.com /var/cache/bind/coherentsecurity.com.zone
zone coherentsecurity.com/IN: loaded serial 2021022401
OK
```

最后，通过运行以下命令启用`bind9`服务并启动它（或者如果您正在“推送”更新，则重新启动它）：

```
sudo systemctl enable bind9
sudo systemctl start bind9
```

我们现在能够使用本地主机上的 DNS 服务器解析我们区域中的主机名，方法如下：

```
$ dig @127.0.0.1 +short ns01.coherentsecurity.com A
192.168.122.157
$ dig @127.0.0.1 +short esx01.coherentsecurity.com A
192.168.122.51
```

由于递归和转发器已经就位，我们也可以解析公共互联网上的主机，就像这样：

```
$ dig @127.0.0.1 +short isc.sans.edu
45.60.31.34
45.60.103.34
```

完成并运行我们的内部 DNS 服务器后，让我们看看我们面向互联网的 DNS，这将允许人们从公共互联网解析我们公司的资源。

## BIND：面向互联网的实现细节

在我们开始之前，这种配置已经不像以前那样常见了。回到 20 世纪 90 年代或更早，如果您想让人们访问您的 Web 服务器，最常见的方法是建立自己的 DNS 服务器或使用由您的 ISP 提供的 DNS 服务器。在任何一种情况下，任何 DNS 更改都是手动文件编辑。

在更近的时代，更常见的是将 DNS 服务托管给 DNS 注册商。这种“云”方法将安全实施留给 DNS 提供商，并简化了维护，因为各种提供商通常会给您一个 Web 界面来维护您的区域文件。在这种模型中的关键安全考虑是，您将希望提供商为您提供启用**多因素身份验证**（**MFA**）的选项（例如，使用 Google Authenticator 或类似工具），以防范针对您的管理访问的**凭证填充**攻击。还值得研究您的注册商的帐户恢复程序——您不希望经过所有实施 MFA 的工作，然后让攻击者通过简单的求助电话来窃取它！

话虽如此，许多组织仍然有充分的理由实施自己的 DNS 服务器，因此让我们继续修改我们在上一节中使用的配置，以用作互联网 DNS 服务器，方法如下：

+   `etc/bind/named.conf.options`，我们将要添加某种速率限制——在 DNS 的情况下，这是 RRL 算法。

+   然而，请记住，这有可能拒绝合法查询的服务。让我们将`responses-per-second`值设置为`10`作为初步速率限制，但将其设置为`log-only`状态。让它在`log-only`模式下运行一段时间，并调整每秒速率，直到你确信你有一个足够低以防止激进攻击但又足够高以在合法操作期间不拒绝访问的值。在此过程中要监视的日志文件，如前面提到的，是`/var/log/syslog`。当你对你的值感到满意时，删除`log-only`行。一旦开始运行，请确保监视任何触发此设置的情况——这可以在你的日志记录或**安全信息和事件管理**（**SIEM**）解决方案中通过简单的关键字匹配轻松完成。代码如下所示：

```
        rate-limit {
             responses-per-second 10
             log-only yes;
        }
```

+   `/etc/bind/named.conf.options`。此外，完全删除 forwarders 行。代码如下所示：

```
        recursion no;
```

+   将`allow-query`行修改为如下内容：

```
allow-query { localhost; 0.0.0.0/0 }
```

既然我们既有内部用户的 DNS 服务器，又有互联网客户端的 DNS 服务器，我们可以使用哪些工具来排查这项服务？

# DNS 故障排除和侦察

在 Linux 中用于排查 DNS 服务的主要工具是`dig`，它几乎在所有 Linux 发行版中预装。如果你的发行版中没有`dig`，你可以用`apt-get install dnsutils`来安装它。这个工具的使用非常简单，可以在这里看到：

```
Dig <request value you are making> <the request type you are making>  +<additional request types>
```

因此，要查找公司（我们将检查`sans.org`）的名称服务器记录，我们将对`sans.org`进行`ns`查询，如下所示：

```
$ dig sans.org ns
; <<>> DiG 9.16.1-Ubuntu <<>> sans.org ns
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 27639
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 65494
;; QUESTION SECTION:
;sans.org.                      IN      NS
;; ANSWER SECTION:
sans.org.               86400   IN      NS      ns-1270.awsdns-30.org.
sans.org.               86400   IN      NS      ns-1746.awsdns-26.co.uk.
sans.org.               86400   IN      NS      ns-282.awsdns-35.com.
sans.org.               86400   IN      NS      ns-749.awsdns-29.net.
;; Query time: 360 msec
;; SERVER: 127.0.0.53#53(127.0.0.53)
;; WHEN: Fri Feb 12 12:02:26 PST 2021
;; MSG SIZE  rcvd: 174
```

这包含了很多注释信息——知道哪些 DNS 标志被设置，以及 DNS 问题和答案的确切操作，都是非常有价值的信息，而这些信息都在默认输出中。然而，通常也希望得到一个“只有事实”的输出——为了得到这个，我们将添加第二个参数`+short`，如下所示：

```
$ dig sans.org ns +short
ns-749.awsdns-29.net.
ns-282.awsdns-35.com.
ns-1746.awsdns-26.co.uk.
ns-1270.awsdns-30.org.
```

`dig`命令允许我们进行任何我们喜欢的 DNS 查询。然而，你一次只能查询一个目标，所以要获取**NS**信息（与**名称服务器**相关）和**邮件交换器**（**MX**）信息，你需要进行两次查询。MX 查询如下所示：

```
$ dig sans.org mx +short
0 sans-org.mail.protection.outlook.com.
```

我们可以使用哪些其他工具来进行故障排除，还有哪些其他 DNS 实现可能会涉及？

# DoH

**DoH**是一种较新的 DNS 协议；顾名思义，它是通过 HTTPS 传输的，实际上，DNS 查询和响应在形式上类似于**应用程序编程接口**（**API**）。这个新协议最初在许多浏览器中得到支持，而不是在主流操作系统中本地支持。然而，现在它已经在大多数主流操作系统上可用，只是默认情况下没有启用。

为了远程验证 DoH 服务器，`curl`（一个关于“*查看 url*”的双关语）工具可以很好地完成这项工作。在以下示例中，我们正在对 Cloudflare 的名称服务器进行查询：

```
$ curl -s -H 'accept: application/dns-json' 'https://1.1.1.1/dns-query?name=www.coherentsecurity.com&type=A'
{"Status":0,"TC":false,"RD":true,"RA":true,"AD":false,"CD":false,"Question":[{"name":"www.coherentsecurity.com","type":1}],"Answer":[{"name":"www.coherentsecurity.com","type":5,"TTL":1693,"data":"robvandenbrink.github.io."},{"name":"robvandenbrink.github.io","type":1,"TTL":3493,"data":"185.199.108.153"},{"name":"robvandenbrink.github.io","type":1,"TTL":3493,"data":"185.199.109.153"},
{"name":"robvandenbrink.github.io","type":1,"TTL":3493,"data":"185.199.110.153"},{"name":"robvandenbrink.github.io","type":1,"TTL":3493,"data":"185.199.111.153"}]}
```

请注意，查询只是一个形式如下的`https`请求：

```
https://<the dns server ip>/dns-query?name=<the dns query target>&type=<the dns request type>  
```

请求中的 HTTP 头是`accept: application/dns-json`。请注意，这个查询使用标准的 HTTPS，因此它监听在端口`tcp/443`上，而不是常规的`udp/53`和`tcp/53` DNS 端口。

我们可以通过`jq`将命令输出变得更易读。这个简单的查询显示了输出中的标志——DNS 问题、答案和授权部分。请注意在以下代码片段中，服务器设置了`RD`标志（代表`RA`标志（代表**递归可用**）：

```
curl -s -H 'accept: application/dns-json' 'https://1.1.1.1/dns-query?name=www.coherentsecurity.com&type=A' | jq
{
  "Status": 0,
  "TC": false,
  "RD": true,
  "RA": true,
  "AD": false,
  "CD": false,
  "Question": [
    {
      "name": "www.coherentsecurity.com",
      "type": 1
    }
  ],
  "Answer": [
    {
      "name": "www.coherentsecurity.com",
      "type": 5,
      "TTL": 1792,
      "data": "robvandenbrink.github.io."
    },
    ….  
    {
      "name": "robvandenbrink.github.io",
      "type": 1,
      "TTL": 3592,
      "data": "185.199.111.153"
    }
  ]
}
```

**网络映射器**（**Nmap**）也可以用来验证远程 DoH 服务器上的证书，如下面的代码片段所示：

```
nmap -p443 1.1.1.1 --script ssl-cert.nse
Starting Nmap 7.80 ( https://nmap.org ) at 2021-02-25 11:28 Eastern Standard Time
Nmap scan report for one.one.one.one (1.1.1.1)
Host is up (0.029s latency).
PORT    STATE SERVICE
443/tcp open  https
| ssl-cert: Subject: commonName=cloudflare-dns.com/organizationName=Cloudflare, Inc./stateOrProvinceName=California/countryName=US
| Subject Alternative Name: DNS:cloudflare-dns.com, DNS:*.cloudflare-dns.com, DNS:one.one.one.one, IP Address:1.1.1.1, IP Address:1.0.0.1, IP Address:162.159.36.1, IP Address:162.159.46.1, IP Address:2606:4700:4700:0:0:0:0:1111, IP Address:2606:4700:4700:0:0:0:0:1001, IP Address:2606:4700:4700:0:0:0:0:64, IP Address:2606:4700:4700:0:0:0:0:6400
| Issuer: commonName=DigiCert TLS Hybrid ECC SHA384 2020 CA1/organizationName=DigiCert Inc/countryName=US
| Public Key type: unknown
| Public Key bits: 256
| Signature Algorithm: ecdsa-with-SHA384
| Not valid before: 2021-01-11T00:00:00
| Not valid after:  2022-01-18T23:59:59
| MD5:   fef6 c18c 02d0 1a14 ab75 1275 dd6a bc29
|_SHA-1: f1b3 8143 b992 6454 97cf 452f 8c1a c842 4979 4282
Nmap done: 1 IP address (1 host up) scanned in 7.41 seconds
```

然而，Nmap 目前没有附带一个脚本，可以通过实际进行 DoH 查询来验证 DoH 本身。为了填补这个空白，你可以在这里下载这样一个脚本：https://github.com/robvandenbrink/dns-doh.nse。

该脚本通过 Lua `http.shortport`操作符验证端口是否正在服务 HTTP 请求，然后构造查询字符串，然后使用正确的标头进行 HTTPS 请求。此工具的完整说明可在此处找到：https://isc.sans.edu/forums/diary/Fun+with+NMAP+NSE+Scripts+and+DOH+DNS+over+HTTPS/27026/。

在彻底探索 DoH 之后，我们还有哪些其他协议可用于验证和加密我们的 DNS 请求和响应？

# DoT

`tcp/853`，这意味着它不会与 DNS（`udp/53`和`tcp/53`）或 DoH（`tcp/443`）发生冲突—如果 DNS 服务器应用程序支持所有三种服务，则这三种服务都可以在同一主机上运行。

大多数现代操作系统（作为客户端）支持 DoT 名称解析。它并不总是默认运行，但如果需要，可以启用它。

远程验证 DoT 服务器就像使用 Nmap 验证`tcp/853`是否在监听一样简单，如下面的代码片段所示：

```
$ nmap -p 853 8.8.8.8
Starting Nmap 7.80 ( https://nmap.org ) at 2021-02-21 13:33 PST
Nmap scan report for dns.google (8.8.8.8)
Host is up (0.023s latency).
PORT    STATE SERVICE
853/tcp open  domain-s
Doing a version scan gives us more good information, but the fingerprint (at the time of this book being published) is not in nmape:
$ nmap -p 853 -sV  8.8.8.8
Starting Nmap 7.80 ( https://nmap.org ) at 2021-02-21 13:33 PST
Nmap scan report for dns.google (8.8.8.8)
Host is up (0.020s latency).
PORT    STATE SERVICE    VERSION
853/tcp open  ssl/domain (generic dns response: NOTIMP)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port853-TCP:V=7.80%T=SSL%I=7%D=2/21%Time=6032D1B5%P=x86_64-pc-linux-gnu
SF:%r(DNSVersionBindReqTCP,20,"\0\x1e\0\x06\x81\x82\0\x01\0\0\0\0\0\0\x07v
SF:ersion\x04bind\0\0\x10\0\x03")%r(DNSStatusRequestTCP,E,"\0\x0c\0\0\x90\
SF:x04\0\0\0\0\0\0\0\0");
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.66 seconds
```

在前面的代码片段中显示的标记为`domain-s`（DNS over `-sV`）的开放端口`tcp/853`显示了响应中的`DNSStatusRequestTCP`字符串，这是一个很好的线索，表明该端口实际上正在运行 DoT。由于它是 DoT，我们还可以使用 Nmap 再次验证验证 DoT 服务的证书，如下所示：

```
nmap -p853 --script ssl-cert 8.8.8.8
Starting Nmap 7.80 ( https://nmap.org ) at 2021-02-21 16:35 Eastern Standard Time
Nmap scan report for dns.google (8.8.8.8)
Host is up (0.017s latency).
PORT    STATE SERVICE
853/tcp open  domain-s
| ssl-cert: Subject: commonName=dns.google/organizationName=Google LLC/stateOrProvinceName=California/countryName=US
| Subject Alternative Name: DNS:dns.google, DNS:*.dns.google.
com, DNS:8888.google, DNS:dns.google.com, DNS:dns64.dns.google, IP Address:2001:4860:4860:0:0:0:0:64, IP Address:2001:4860:4860:0:0:0:0:6464, IP Address:2001:4860:4860:0:0:0:0:8844, IP Address:2001:4860:4860:0:0:0:0:8888, IP Address:8.8.4.4, IP Address:8.8.8.8
| Issuer: commonName=GTS CA 1O1/organizationName=Google Trust Services/countryName=US
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2021-01-26T08:54:07
| Not valid after:  2021-04-20T08:54:06
| MD5:   9edd 82e5 5661 89c0 13a5 cced e040 c76d
|_SHA-1: 2e80 c54b 0c55 f8ad 3d61 f9ae af43 e70c 1e67 fafd
Nmap done: 1 IP address (1 host up) scanned in 7.68 seconds
```

这就是我们到目前为止讨论的工具所能达到的程度。`dig`工具（目前）不支持进行 DoT 查询。但是，`knot-dnsutils`软件包为我们提供了一个“几乎与`dig`相同”的命令行工具—`kdig`。让我们使用这个工具来更深入地探索 DoT。

## knot-dnsutils

`knot-dnsutils`是一个包含`kdig`工具的 Linux 软件包。`kdig`复制了`dig`工具的功能，但还添加了其他功能，包括对 DoT 查询的支持。要开始使用这个工具，我们首先必须安装`knot-dnsutils`软件包，如下所示：

```
sudo apt-get install  knot-dnsutils
```

安装完成后，`kdig`实用程序与`dig`命令非常相似，只是增加了一些额外的命令行参数—让我们进行 DoT 查询以说明这一点，如下所示：

```
kdig -d +short @8.8.8.8 www.cisco.com A  +tls-ca +tls-hostname=dns.google # +tls-sni=dns.google
;; DEBUG: Querying for owner(www.cisco.com.), class(1), type(1), server(8.8.8.8), port(853), protocol(TCP)
;; DEBUG: TLS, imported 129 system certificates
;; DEBUG: TLS, received certificate hierarchy:
;; DEBUG:  #1, C=US,ST=California,L=Mountain View,O=Google LLC,CN=dns.google
;; DEBUG:      SHA-256 PIN: 0r0ZP20iM96B8DOUpVSlh5sYx9GT1NBVp181TmVKQ1Q=
;; DEBUG:  #2, C=US,O=Google Trust Services,CN=GTS CA 1O1
;; DEBUG:      SHA-256 PIN: YZPgTZ+woNCCCIW3LH2CxQeLzB/1m42QcCTBSdgayjs=
;; DEBUG: TLS, skipping certificate PIN check
;; DEBUG: TLS, The certificate is trusted.
www.cisco.com.akadns.net.
wwwds.cisco.com.edgekey.net.
wwwds.cisco.com.edgekey.net.globalredir.akadns.net.
e2867.dsca.akamaiedge.net.
23.66.161.25
```

我们使用了哪些新参数？

`debug`参数（`-d`）给出了包括`DEBUG`字符串的所有前面的行。鉴于大多数人可能会使用`kdig`因为它支持 TLS，这些`DEBUG`行为我们提供了一些在测试新服务时可能经常需要的优秀信息。如果没有`debug`参数，我们的输出将更像是`dig`，如下面的代码片段所示：

```
kdig  +short @8.8.8.8 www.cisco.com A  +tls-ca +tls-hostname=dns.google +tls-sni=dns.google
www.cisco.com.akadns.net.
wwwds.cisco.com.edgekey.net.
wwwds.cisco.com.edgekey.net.globalredir.akadns.net.
e2867.dsca.akamaiedge.net.
23.66.161.25
```

`+short`参数将输出缩短为“只有事实”显示，就像`dig`一样。如果没有这个参数，输出将包括所有部分（不仅仅是“答案”部分），如下面的代码片段所示：

```
kdig @8.8.8.8 www.cisco.com A  +tls-ca +tls-hostname=dns.google +tls-sni=dns.google
;; TLS session (TLS1.3)-(ECDHE-X25519)-(RSA-PSS-RSAE-SHA256)-(AES-256-GCM)
;; ->>HEADER<<- opcode: QUERY; status: NOERROR; id: 57771
;; Flags: qr rd ra; QUERY: 1; ANSWER: 5; AUTHORITY: 0; ADDITIONAL: 1
;; EDNS PSEUDOSECTION:
;; Version: 0; flags: ; UDP size: 512 B; ext-rcode: NOERROR
;; PADDING: 240 B
;; QUESTION SECTION:
;; www.cisco.com.               IN      A
;; ANSWER SECTION:
www.cisco.com.          3571    IN      CNAME   www.cisco.com.akadns.net.
www.cisco.com.akadns.net.       120     IN      CNAME   wwwds.cisco.com.edgekey.net.
wwwds.cisco.com.edgekey.net.    13980   IN      CNAME   wwwds.cisco.com.edgekey.net.globalredir.akadns.net.
wwwds.cisco.com.edgekey.net.globalredir.akadns.net. 2490        IN      CNAME  e2867.dsca.akamaiedge.net.
e2867.dsca.akamaiedge.net.      19      IN      A       23.66.161.25
;; Received 468 B
;; Time 2021-02-21 13:50:33 PST
;; From 8.8.8.8@853(TCP) in 121.4 ms
```

我们使用的新参数列在此处：

+   `+tls-ca`参数强制执行 TLS 验证—换句话说，它验证证书。默认情况下，系统使用**证书颁发机构**（**CA**）列表进行此操作。

+   添加`+tls-hostname`允许您指定 TLS 协商的主机名。默认情况下，使用 DNS 服务器名称，但在我们的情况下，服务器名称是`8.8.8.8`—您需要一个出现在 TLS 正确协商的**CN**或**主题备用名称**（**SAN**）列表中的有效主机名。因此，此参数允许您独立于服务器名称字段中使用的内容指定该名称。

+   添加`+tls-sni`在请求中添加了**服务器名称指示**（**SNI**）字段，这是许多 DoT 服务器所必需的。这可能看起来有点奇怪，因为 SNI 字段是为了允许 HTTPS 服务器呈现多个证书（每个用于不同的 HTTPS 站点）。

如果您不使用这些参数，只是像使用`dig`一样使用`kdig`会发生什么？默认情况下，`kdig`不会强制验证证书与您指定的 FQDN 是否匹配，因此它通常会正常工作，如下面的代码片段所示：

```
$ kdig +short @8.8.8.8 www.cisco.com A
www.cisco.com.akadns.net.
wwwds.cisco.com.edgekey.net.
wwwds.cisco.com.edgekey.net.globalredir.akadns.net.
e2867.dsca.akamaiedge.net.
23.4.0.216
```

然而，最好按照 TLS 的原意使用它，进行验证——毕竟，重点是在 DNS 结果中添加另一层信任。如果您不验证服务器，那么您所做的就是加密查询和响应。如果不在服务器名称字段或 TLS 主机名字段中指定正确的主机名，您就无法进行验证（此值需要与证书参数匹配）。强制证书验证很重要，因为这样可以确保 DNS 服务器是您真正想要查询的服务器（即，您的流量没有被拦截），并且响应在返回客户端的过程中没有被篡改。

既然我们了解了 DoT 的工作原理，那么我们如何进行故障排除或查明 DNS 主机是否已实施 DoT 呢？

## 在 Nmap 中实施 DoT

与 DoH Nmap 示例类似，实施 DoT 在 Nmap 中允许您以更大的规模进行 DoT 发现和查询，而不是一次一个。考虑到在 Nmap 中进行 HTTPS 调用的复杂性，一个简单的方法就是在 Nmap 脚本中直接调用`kdig`，使用 Lua 中的`os.execute`函数来完成这个任务。

另一个关键区别是，我们不是测试`http`功能的目标端口（使用`shortport.http`测试），而是使用`shortport.ssl`测试来验证 SSL/TLS 功能的任何开放端口；因为如果它不能提供有效的 TLS 请求服务，那么它就不能是 DoT，对吧？

`dns.dot`工具可在此处下载：

https://github.com/robvandenbrink/dns-dot

您可以在此处查看完整的说明：

https://isc.sans.edu/diary/Fun+with+DNS+over+TLS+%28DoT%29/27150

我们可以在 DNS 协议本身上实施哪些其他安全机制？让我们来看看 DNSSEC，这是验证 DNS 响应的原始机制。

## DNSSEC

`udp/53`和`tcp/53`，因为它不加密任何内容——它只是添加字段来验证使用签名的标准 DNS 操作。

您可以使用`dig`中的`DNSKEY`参数查看任何 DNS 区域的公钥。在以下代码示例中，我们添加了`short`参数：

```
$ dig DNSKEY @dns.google example.com +short
256 3 8 AwEAAa79LdJaZfIxVzyjq4H7yB4VqT/rIreB+N0jija+4bWHzNrwhSiu D/SOtgvX+gXEgwAR6tHGn9q9t65o85RfdHJrueORb0usa3x6LHM7qy6A r22P78UUn/rxa9jbi6yS4cVOzLnJ+OKO0w1Scly5XLDmmWPbIM2LvayR 2U4UAqZZ
257 3 8 AwEAAZ0aqu1rJ6orJynrRfNpPmayJZoAx9Ic2/Rl9VQWLMHyjxxem3VU SoNUIFXERQbj0A9Ogp0zDM9YIccKLRd6LmWiDCt7UJQxVdD+heb5Ec4q lqGmyX9MDabkvX2NvMwsUecbYBq8oXeTT9LRmCUt9KUt/WOi6DKECxoG /bWTykrXyBR8elD+SQY43OAVjlWrVltHxgp4/rhBCvRbmdflunaPIgu2 7eE2U4myDSLT8a4A0rB5uHG4PkOa9dIRs9y00M2mWf4lyPee7vi5few2 dbayHXmieGcaAHrx76NGAABeY393xjlmDNcUkF1gpNWUla4fWZbbaYQz A93mLdrng+M=
257 3 8 AwEAAbOFAxl+Lkt0UMglZizKEC1AxUu8zlj65KYatR5wBWMrh18TYzK/ ig6Y1t5YTWCO68bynorpNu9fqNFALX7bVl9/gybA0v0EhF+dgXmoUfRX 7ksMGgBvtfa2/Y9a3klXNLqkTszIQ4PEMVCjtryl19Be9/PkFeC9ITjg MRQsQhmB39eyMYnal+f3bUxKk4fq7cuEU0dbRpue4H/N6jPucXWOwiMA kTJhghqgy+o9FfIp+tR/emKao94/wpVXDcPf5B18j7xz2SvTTxiuqCzC MtsxnikZHcoh1j4g+Y1B8zIMIvrEM+pZGhh/Yuf4RwCBgaYCi9hpiMWV vS4WBzx0/lU=
```

要查看`DS`参数，如下面的代码片段所示：

```
$ dig +short DS @dns.google example.com
31589 8 1 3490A6806D47F17A34C29E2CE80E8A999FFBE4BE
31589 8 2 CDE0D742D6998AA554A92D890F8184C698CFAC8A26FA59875A990C03 
E576343C
43547 8 1 B6225AB2CC613E0DCA7962BDC2342EA4F1B56083
43547 8 2 615A64233543F66F44D68933625B17497C89A70E858ED76A2145997E DF96A918
31406 8 1 189968811E6EBA862DD6C209F75623D8D9ED9142
31406 8 2 F78CF3344F72137235098ECBBD08947C2C9001C7F6A085A17F518B5D 8F6B916D
```

如果我们添加`-d`（调试）参数并过滤以仅查看`DEBUG`数据，我们将在输出中看到以下行，该行指示我们正在使用与常规 DNS 查询相同的端口和协议：

```
dig -d DNSKEY @dns.google example.com  | grep DEBUG
;; DEBUG: Querying for owner(example.com.), class(1), type(48), server(dns.google), port(53), protocol(UDP)
```

要进行 DNSSEC 查询，只需在`dig`命令行中添加`+dnssec`，如下所示：

```
$ dig +dnssec +short @dns.google www.example.com A
93.184.216.34
A 8 3 86400 20210316085034 20210223165712 45150 example.com. UyyNiGG0WDAsberOUza21vYos8vDc6aLq8FV9lvJT4YRBn6V8CTd3cdo ljXV5uETcD54tuv1kLZWg7YZxSQDGFeNC3luZFkbrWAqPbHXy4D7Tdey LBK0R3xywGxgZIEfp9HMjpZpikFQuKC/iFvd14uJhoquMqFPFvTfJB/s XJ8=
```

DNSSEC 主要是关于在客户端和服务器之间，以及在中继请求时服务器之间对 DNS 请求进行认证。正如我们所看到的，它是由任何特定区域的所有者实施的，以允许请求者验证他们得到的 DNS“答案”是否正确。然而，由于其复杂性和对证书的依赖，它并没有像 DoT 和 DoH 那样被广泛采用。

正如我们所看到的，DoT 和 DoH 专注于个人隐私，加密个人进行业务时所做的每个 DNS 请求。虽然这种加密使得这些 DNS 请求更难以捕获，但这些请求仍然记录在 DNS 服务器上。此外，如果攻击者能够收集个人的 DNS 请求，他们也能够简单地记录他们访问的站点（按 IP 地址）。

话虽如此，我们不会深入研究 DNSSEC 的深度，主要是因为作为一个行业，我们已经做出了同样的决定，并且（在大多数情况下）选择不实施它。然而，您确实会不时地看到它，特别是在解决涉及 DNS 的问题时，因此了解它的外观以及为什么可能会实施它是很重要的。

# 总结

随着我们对 DNS 的讨论接近尾声，您现在应该有了构建基本内部 DNS 服务器和面向互联网的标准 DNS 服务器的工具。您还应该具备开始通过编辑 Linux `bind`或命名服务的各种配置文件来保护这些服务的基本工具。

此外，您应该熟悉使用`dig`、`kdig`、`curl`和`nmap`等工具来排除各种 DNS 服务的故障。

在下一章中，我们将继续讨论 DHCP，正如我们在本章中所看到的，它确实是分开的，但仍然与 DNS 有关。

# 问题

最后，这里是一些问题列表，供您测试对本章材料的了解。您将在*附录*的*评估*部分找到答案。

1.  DNSSEC 与 DoT 有何不同？

1.  DoH 与“常规”DNS 有何不同？

1.  您会在内部 DNS 服务器上实现哪些功能，而不是在外部 DNS 服务器上实现？

# 进一步阅读

要了解更多关于这个主题的信息：

+   **权威 DNS 参考**

基本 DNS 实际上有数十个定义服务以及实施最佳实践的 RFC。这里可以找到这些 RFC 的一个好列表：https://en.wikipedia.org/wiki/Domain_Name_System#RFC_documents。

但是，如果您需要更多关于 DNS 的详细信息，并且正在寻找比 RFC 更易读的协议和实施细节的指南（重点是“易读”），许多人认为 Cricket Liu 的书是一个很好的下一步：

*Cricket Liu 和 Paul Albitz 的 DNS 和 BIND*：

https://www.amazon.ca/DNS-BIND-Help-System-Administrators-ebook/dp/B0026OR2QS/ref=sr_1_1?dchild=1&keywords=dns+and+bind+cricket+liu&qid=1614217706&s=books&sr=1-1

*Cricket Liu 的 IPv6 上的 DNS 和 BIND*：

https://www.amazon.ca/DNS-BIND-IPv6-Next-Generation-Internet-ebook/dp/B0054RCT4O/ref=sr_1_3?dchild=1&keywords=dns+and+bind+cricket+liu&qid=1614217706&s=books&sr=1-3

+   **DNS 更新（自动注册）**

*RFC 2136*：*域名系统（DNS UPDATE）中的动态更新*：

https://tools.ietf.org/html/rfc2136

+   **Active Directory（AD）中的经过身份验证的 DNS 注册**

*RFC 3645*：*用于 DNS 的秘密密钥事务认证的通用安全服务算法（GSS-TSIG）*：

https://tools.ietf.org/html/rfc3645

+   **DoH**

*使用 NMAP NSE 脚本和 DOH（HTTPS 上的 DNS）玩乐*：https://isc.sans.edu/forums/diary/Fun+with+NMAP+NSE+Scripts+and+DOH+DNS+over+HTTPS/27026/

DoH Nmap 脚本：https://github.com/robvandenbrink/dns-doh.nse

*RFC 8484*：*HTTPS 上的 DNS 查询（DoH）*：https://tools.ietf.org/html/rfc8484

+   `dns-dot` Nmap 脚本：https://isc.sans.edu/diary/Fun+with+DNS+over+TLS+%28DoT%29/27150

*RFC 7858*：*传输层安全性（TLS）上的 DNS 规范*：https://tools.ietf.org/html/rfc7858

+   **DNSSEC**

*域名系统安全扩展（DNSSEC）*：https://www.internetsociety.org/issues/dnssec/

*RFC 4033*：*DNS 安全介绍和要求*：https://tools.ietf.org/html/rfc4033

*RFC 4034*：*DNS 安全扩展的资源记录*：https://tools.ietf.org/html/rfc4034

*RFC 4035*：*DNS 安全扩展的协议修改*：https://tools.ietf.org/html/rfc4035

*RFC 4470*：*最小覆盖 NSEC 记录和 DNSSEC 在线签名*：https://tools.ietf.org/html/rfc4470

*RFC 4641*：*DNSSEC 操作实践*：https://tools.ietf.org/html/rfc4641

*RFC 5155*：*DNS 安全（DNSSEC）哈希认证否定存在*：https://tools.ietf.org/html/rfc5155

*RFC 6014*：*DNSSEC 的加密算法标识符分配*：https://tools.ietf.org/html/rfc6014

*RFC 4398*：*在域名系统（DNS）中存储证书*：https://tools.ietf.org/html/rfc4398
