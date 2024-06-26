# 第十章：Linux 负载均衡器服务

在本章中，我们将讨论适用于 Linux 的负载均衡器服务，具体来说是 HAProxy。负载均衡器允许客户端工作负载分布到多个后端服务器。这允许单个 IP 扩展到比单个服务器更大，并且在服务器故障或维护窗口的情况下也允许冗余。

完成这些示例后，您应该具备通过几种不同的方法在自己的环境中部署基于 Linux 的负载均衡器服务的技能。

特别是，我们将涵盖以下主题：

+   负载均衡简介

+   负载均衡算法

+   服务器和服务健康检查

+   数据中心负载均衡器设计考虑

+   构建 HAProxy NAT/代理负载均衡器

+   关于负载均衡器安全性的最后说明

由于设置此部分的基础设施的复杂性，您可以在示例配置方面做出一些选择。

# 技术要求

在本章中，我们将探讨负载均衡器功能。当我们在本书的后面示例中工作时，您可以跟着进行，并在当前 Ubuntu 主机或虚拟机中实施我们的示例配置。但是，要看到我们的负载均衡示例的实际效果，您需要一些东西：

+   至少有两个目标主机来平衡负载

+   当前 Linux 主机中的另一个网络适配器

+   另一个子网来托管目标主机和这个新的网络适配器

此配置有一个匹配的图表，*图 10.2*，将在本章后面显示，说明了当我们完成时所有这些将如何连接在一起。

这给我们的实验室环境的配置增加了一整个层次的复杂性。当我们到达实验室部分时，我们将提供一些替代方案（下载预构建的虚拟机是其中之一），但您也可以选择跟着阅读。如果是这种情况，我认为您仍然会对这个主题有一个很好的介绍，以及对现代数据中心中各种负载均衡器配置的设计、实施和安全影响有一个扎实的背景。

# 负载均衡简介

在其最简单的形式中，负载均衡就是将客户端负载分布到多个服务器上。这些服务器可以在一个或多个位置，以及分配负载的方法可以有很大的不同。事实上，您在均匀分配负载方面的成功程度也会有很大的不同（主要取决于所选择的方法）。让我们探讨一些更常见的负载均衡方法。

## 循环 DNS（RRDNS）

您可以只使用 DNS 服务器进行简单的负载均衡，即所谓的`a.example.com`主机名，DNS 服务器将返回服务器 1 的 IP；然后，当下一个客户端请求时，它将返回服务器 2 的 IP，依此类推。这是最简单的负载均衡方法，对于共同放置的服务器和不同位置的服务器同样有效。它也可以在基础设施上不做任何更改-没有新组件，也没有配置更改：

![图 10.1-循环 DNS 的简单负载均衡](img/B16336_10_001.jpg)

图 10.1-循环 DNS 的简单负载均衡

配置 RRDNS 很简单-在 BIND 中，只需为目标主机名配置多个`A`记录，其中包含多个 IP。连续的 DNS 请求将按顺序返回每个`A`记录。将域的`A`记录缩短是个好主意，以便按顺序（顺序返回匹配的记录）、随机或固定（始终以相同顺序返回匹配的记录）。更改返回顺序的语法如下（`cyclic`，默认设置，如下所示）：

```
options { 
    rrset-order { 
        class IN type A name "mytargetserver.example.com" order cyclic; 
    }; 
}; 
```

这种配置存在一些问题：

+   在这种模型中，没有好的方法来整合任何类型的健康检查-所有服务器是否正常运行？服务是否正常？主机是否正常？

+   没有办法看到任何 DNS 请求是否实际上会跟随连接到服务。有各种原因可能会发出 DNS 请求，并且交互可能就此结束，没有后续连接。

+   也没有办法监视会话何时结束，这意味着没有办法将下一个请求发送到最少使用的服务器 - 它只是在所有服务器之间稳定地轮换。因此，在任何工作日的开始，这可能看起来像一个好模型，但随着一天的进展，总会有持续时间更长的会话和极短的会话（或根本没有发生的会话），因此很常见看到服务器负载在一天进展过程中变得“不平衡”。如果没有明确的一天开始或结束来有效地“清零”，这种情况可能会变得更加明显。

+   出于同样的原因，如果集群中的一个服务器因维护或非计划中断而下线，没有好的方法将其恢复到与会话计数相同的状态。

+   通过一些 DNS 侦察，攻击者可以收集所有集群成员的真实 IP，然后分别评估它们或对它们进行攻击。如果其中任何一个特别脆弱或具有额外的 DNS 条目标识它为备用主机，这将使攻击者的工作变得更加容易。

+   将任何目标服务器下线可能会成为一个问题 - DNS 服务器将继续按请求的顺序提供该地址。即使记录被编辑，任何下游客户端和 DNS 服务器都将缓存其解析的 IP，并继续尝试连接到已关闭的主机。

+   下游 DNS 服务器（即互联网上的服务器）将在区域的 TTL 周期内缓存它们获取的任何记录。因此，任何这些 DNS 服务器的客户端都将被发送到同一个目标服务器。

因此，RRDNS 可以在紧急情况下简单地完成工作，但通常不应将其实施为长期的生产解决方案。也就是说，**全局服务器负载均衡器**（**GSLB**）产品实际上是基于这种方法的，具有不同的负载均衡选项和健康检查。负载均衡器与目标服务器之间的断开在 GSLB 中仍然存在，因此许多相同的缺点也适用于这种解决方案。

在数据中心中，我们更经常看到基于代理（第 7 层）或基于 NAT（第 4 层）的负载均衡。让我们探讨这两个选项。

## 入站代理 - 第 7 层负载均衡

在这种架构中，客户端的会话在代理服务器上终止，并在代理的内部接口和真实服务器 IP 之间启动新的会话。

这也提出了许多在许多负载均衡解决方案中常见的架构术语。在下图中，我们可以看到**前端**的概念，面向客户端，以及**后端**，面向服务器。我们还应该在这一点上讨论 IP 地址。前端呈现了所有目标服务器共享的**虚拟 IP**（**VIP**），客户端根本看不到服务器的**真实 IP**（**RIPs**）：

![图 10.2 - 使用反向代理进行负载均衡](img/B16336_10_002.jpg)

图 10.2 - 使用反向代理进行负载均衡

这种方法有一些缺点：

+   在本章讨论的所有方法中，它对负载均衡器的 CPU 负载最高，并且在极端情况下可能会对客户端产生性能影响。

+   此外，由于目标服务器上的客户端流量都来自代理服务器（或服务器），如果没有一些特殊处理，那么在目标/应用程序服务器上看到的客户端 IP 将始终是负载均衡器的后端 IP。这使得在应用程序中记录直接客户端交互成为问题。要从一个会话中解析出流量并将其与客户端的实际地址相关联，我们必须将负载均衡器的客户端会话（它看到客户端 IP 地址但看不到用户身份）与应用程序/网络服务器日志（它看到用户身份但看不到客户端 IP 地址）进行匹配。在这些日志之间匹配会话可能是一个真正的问题；它们之间的共同元素是负载均衡器上的时间戳和源端口，而源端口通常不在 Web 服务器上。

+   这可以通过具有一些应用程序意识来减轻。例如，常见的是为 Citrix ICA 服务器或 Microsoft RDP 服务器后端设置 TLS 前端。在这些情况下，代理服务器对协议有一些出色的“钩子”，允许客户端 IP 地址一直传递到服务器，并且负载均衡器检测到的身份也被检测到。

然而，使用代理架构允许我们完全检查流量是否受到攻击，如果工具设置好的话。实际上，由于代理架构，负载均衡器和目标服务器之间的最后一个“跳跃”是一个全新的会话 - 这意味着无效的协议攻击大部分被过滤掉，而无需进行任何特殊配置。

我们可以通过将负载均衡器作为入站**网络地址转换**（**NAT**）配置来减轻代理方法的一些复杂性。当不需要解密时，NAT 方法通常是常见的，并内置于大多数环境中。

## 入站 NAT - 第 4 层负载平衡

这是最常见的解决方案，也是我们在示例中将要开始使用的解决方案。在许多方面，这种架构看起来与代理解决方案相似，但有一些关键的区别。请注意，在下图中，前端和后端的 TCP 会话现在匹配 - 这是因为负载均衡器不再是代理；它已被配置为入站 NAT 服务。所有客户端仍然连接到单个 VIP，并由负载均衡器重定向到各种服务器 RIP：

![图 10.3 - 使用入站 NAT 进行负载平衡](img/B16336_10_003.jpg)

图 10.3 - 使用入站 NAT 进行负载平衡

在许多情况下，这是首选架构的几个原因：

+   服务器看到客户端的真实 IP，并且服务器日志正确地反映了这一点。

+   负载均衡器在内存中维护 NAT 表，负载均衡器日志反映了各种 NAT 操作，但无法“看到”会话。例如，如果服务器正在运行 HTTPS 会话，如果这是一个简单的第 4 层 NAT，则负载均衡器可以看到 TCP 会话，但无法解密流量。

+   我们可以选择在前端终止 HTTPS 会话，然后在此架构中在后端运行加密或明文。然而，由于我们维护了两个会话（前端和后端），这开始看起来更像是代理配置。

+   由于负载均衡器看到整个 TCP 会话（直到第 4 层），现在可以使用多种负载平衡算法（有关更多信息，请参见负载平衡算法的下一节）。

+   这种架构允许我们在负载均衡器上放置**Web 应用程序防火墙**（**WAF**）功能，这可以掩盖目标服务器 Web 应用程序上的一些漏洞。例如，WAF 是对跨站脚本或缓冲区溢出攻击的常见防御，或者任何可能依赖输入验证中断的攻击。对于这些类型的攻击，WAF 识别任何给定字段或 URI 的可接受输入，然后丢弃任何不匹配的输入。但是，WAF 并不局限于这些攻击。将 WAF 功能视为 Web 特定的 IPS（见*第十四章*，*Linux 上的蜜罐服务*）。

+   这种架构非常适合使会话持久或“粘性” - 我们的意思是一旦客户端会话“附加”到服务器，随后的请求将被定向到同一台服务器。这非常适合具有后端数据库的页面，例如，如果您不保持相同的后端服务器，您的活动（例如，电子商务网站上的购物车）可能会丢失。动态或参数化网站 - 在这些网站上，页面在您导航时实时生成（例如，大多数具有产品目录或库存的网站） - 通常也需要会话持久性。

+   您还可以独立地负载均衡每个连续的请求，因此，例如，当客户端浏览网站时，他们的会话可能会在每个页面由不同的服务器终止。这种方法非常适合静态网站。

+   您可以在这种架构的基础上叠加其他功能。例如，这些通常与防火墙并行部署，甚至与公共互联网上的本机接口并行部署。因此，您经常会看到负载均衡器供应商配备 VPN 客户端以配合其负载均衡器。

+   如前图所示，入站 NAT 和代理负载均衡器具有非常相似的拓扑结构 - 连接看起来都非常相似。这一点延续到了实现中，可以看到一些东西通过代理和一些东西通过同一负载均衡器上的 NAT 过程。

然而，即使这种配置的 CPU 影响远低于代理解决方案，每个工作负载数据包仍必须通过负载均衡器在两个方向上进行路由。我们可以使用**直接服务器返回**（**DSR**）架构大大减少这种影响。

## DSR 负载平衡

在 DSR 中，所有传入的流量仍然从负载均衡器上的 VIP 负载均衡到各个服务器的 RIP。然而，返回流量直接从服务器返回到客户端，绕过负载均衡器。

这怎么可能？这是怎么回事：

+   在进入时，负载均衡器会重写每个数据包的 MAC 地址，将它们负载均衡到目标服务器的 MAC 地址上。

+   每台服务器都有一个环回地址，这是一个配置的地址，与 VIP 地址匹配。这是返回所有流量的接口（因为客户端期望从 VIP 地址返回流量）。但是，它必须配置为不回复 ARP 请求（否则，负载均衡器将在入站路径上被绕过）。

这可能看起来很复杂，但以下图表应该使事情变得更清晰一些。请注意，此图表中只有一个目标主机，以使流量流动更容易看到：

![图 10.4 - DSR 负载平衡](img/B16336_10_004.jpg)

图 10.4 - DSR 负载平衡

构建这个的要求非常严格：

+   负载均衡器和所有目标服务器都需要在同一个子网上。

+   这种机制需要在默认网关上进行一些设置，因为在进入时，它必须将所有客户端流量定向到负载均衡器上的 VIP，但它还必须接受来自具有相同地址但不同 MAC 地址的多个目标服务器的回复。这个必须有一个 ARP 条目，对于每个目标服务器，都有相同的 IP 地址。在许多架构中，这是通过多个静态 ARP 条目来完成的。例如，在 Cisco 路由器上，我们会这样做：

```
arp 192.168.124.21 000c.2933.2d05 arpa
arp 192.168.124.22 000c.29ca.fbea arpa
```

请注意，在这个例子中，`192.168.124.21`和`22`是被负载均衡的目标主机。此外，MAC 地址具有 OUI，表明它们都是 VMware 虚拟主机，在大多数数据中心都是典型的。

为什么要经历所有这些麻烦和不寻常的网络配置？

+   DSR 配置的优势在于大大减少了通过负载均衡器的流量。例如，在 Web 应用程序中，通常会看到返回流量超过传入流量的 10 倍以上。这意味着对于这种流量模型，DSR 实现将看到负载均衡器将看到的流量的 90%或更少。

+   不需要“后端”子网；负载均衡器和目标服务器都在同一个子网中 - 实际上，这是一个要求。这也有一些缺点，正如我们已经讨论过的那样。我们将在*DSR 的特定服务器设置*部分详细介绍这一点。

然而，也有一些缺点：

+   集群中的相对负载，或者任何一个服务器上的个体负载，最多只能由负载均衡器推断出来。如果一个会话正常结束，负载均衡器将捕捉到足够的“会话结束”握手来判断会话已经结束，但如果一个会话没有正常结束，它完全依赖超时来结束会话。

+   所有主机必须配置相同的 IP（原始目标），以便返回流量不会来自意外的地址。这通常是通过环回接口完成的，并且通常需要对主机进行一些额外的配置。

+   上游路由器（或者如果它是子网的网关，则是第 3 层交换机）需要配置为允许目标 IP 地址的所有可能的 MAC 地址。这是一个手动过程，如果可能看到 MAC 地址意外更改，这可能是一个问题。

+   如果任何需要代理或完全可见会话的功能（如 NAT 实现中）无法工作，负载均衡器只能看到会话的一半。这意味着任何 HTTP 头解析、cookie 操作（例如会话持久性）或 SYN cookie 都无法实现。

此外，因为（就路由器而言）所有目标主机具有不同的 MAC 地址但相同的 IP 地址，而目标主机不能回复任何 ARP 请求（否则，它们将绕过负载均衡器），因此需要在目标主机上进行大量的工作。

### DSR 的特定服务器设置

对于 Linux 客户端，必须对“VIP”寻址接口（无论是环回还是逻辑以太网）进行 ARP 抑制。可以使用`sudo ip link set <interface name> arp off`或（使用较旧的`ifconfig`语法）`sudo ifconfig <interface name> -arp`来完成。

您还需要在目标服务器上实现“强主机”和“弱主机”设置。如果服务器接口不是路由器，并且不能发送或接收来自接口的数据包，除非数据包中的源或目的 IP 与接口 IP 匹配，则将其配置为“强主机”。如果接口已配置为“弱主机”，则不适用此限制-它可以代表其他接口接收或发送数据包。

Linux 和 BSD Unix 默认在所有接口上启用了`weak host`（`sysctl net.ip.ip.check_interface = 0`）。Windows 2003 及更早版本也启用了这个功能。但是，Windows Server 2008 及更高版本为所有接口采用了`strong host`模型。要更改新版本 Windows 中的 DSR，执行以下代码：

```
netsh interface ipv4 set interface "Local Area Connection" weakhostreceive=enabled
netsh interface ipv4 set interface "Loopback" weakhostreceive=enabled
netsh interface ipv4 set interface "Loopback" weakhostsend=enabled 
```

您还需要在目标服务器上禁用任何 IP 校验和 TCP 校验和卸载功能。在 Windows 主机上，这两个设置位于`网络适配器/高级`设置中。在 Linux 主机上，`ethtool`命令可以操作这些设置，但这些基于硬件的卸载功能在 Linux 中默认情况下是禁用的，因此通常不需要调整它们。

在描述的各种架构中，我们仍然需要确定如何精确地分配客户端负载到我们的目标服务器组。

# 负载平衡算法

到目前为止，我们已经涉及了一些负载平衡算法，让我们更详细地探讨一下更常见的方法（请注意，这个列表不是详尽无遗的；这里只提供了最常见的方法）：

![](img/B16336_10_Table_01.jpg)

**最少连接**，正如你可能期望的那样，是最常分配的算法。我们将在本章后面的配置示例中使用这种方法。

既然我们已经看到了如何平衡工作负载的一些选项，那么我们如何确保那些后端服务器正常工作呢？

# 服务器和服务健康检查

我们在 DNS 负载平衡部分讨论的问题之一是健康检查。一旦开始负载平衡，通常希望有一种方法来知道哪些服务器（和服务）正在正确运行。检查任何连接的*健康*的方法包括以下内容：

1.  定期使用 ICMP 有效地“ping”目标服务器。如果没有 ICMP 回显回复，则认为它们宕机，并且不会接收任何新的客户端。现有客户端将分布在其他服务器上。

1.  使用 TCP 握手并检查开放端口（例如`80/tcp`和`443/tcp`用于 Web 服务）。同样，如果握手未完成，则主机被视为宕机。

1.  在 UDP 中，您通常会发出应用程序请求。例如，如果您正在负载均衡 DNS 服务器，负载均衡器将进行简单的 DNS 查询-如果收到 DNS 响应，则认为服务器正常运行。

1.  最后，在平衡 Web 应用程序时，您可以进行实际的 Web 请求。通常，您会请求索引页面（或任何已知页面）并查找该页面上的已知文本。如果该文本不出现，则该主机和服务组合被视为宕机。在更复杂的环境中，您检查的测试页面可能会对后端数据库进行已知调用以进行验证。

测试实际应用程序（如前两点所述）当然是验证应用程序是否正常工作的最可靠方法。

我们将在示例配置中展示一些这些健康检查。在我们开始之前，让我们深入了解一下您可能会在典型数据中心中看到负载均衡器的实现方式-无论是在“传统”配置中还是在更现代的实现中。

# 数据中心负载均衡器设计考虑

负载平衡已经成为较大架构的一部分几十年了，这意味着我们经历了几种常见的设计。

我们经常看到的“传统”设计是一个单一对（或集群）物理负载均衡器，为数据中心中的所有负载平衡工作负载提供服务。通常，相同的负载均衡器集群用于内部和外部工作负载，但有时，您会看到一个内部负载均衡器对内部网络进行服务，另一个对只服务 DMZ 工作负载（即对外部客户端）的负载均衡器对外部工作负载进行服务。

这种模型在我们拥有物理服务器且负载均衡器是昂贵的硬件的时代是一个很好的方法。

然而，在虚拟化环境中，工作负载 VM 绑定到物理负载均衡器，这使得网络配置复杂化，限制了灾难恢复选项，并且通常导致流量在（物理）负载均衡器和虚拟环境之间进行多次“循环”：

![图 10.5 - 传统负载均衡架构](img/B16336_10_005.jpg)

图 10.5 - 传统负载均衡架构

随着虚拟化的出现，一切都发生了变化。现在使用物理负载均衡器几乎没有意义 - 最好是为每个工作负载使用专用的小型虚拟机，如下所示：

![图 10.6 - 现代负载均衡架构](img/B16336_10_006.jpg)

图 10.6 - 现代负载均衡架构

这种方法有几个优点：

+   **成本**是一个优势，因为这些小型虚拟负载均衡器如果获得许可要便宜得多，或者如果使用诸如 HAProxy（或任何其他免费/开源解决方案）的解决方案则是免费的。这可能是影响最小的优势，但毫不奇怪通常是改变意见的因素。

+   **配置要简单得多**，更容易维护，因为每个负载均衡器只服务一个工作负载。如果进行了更改并且可能需要后续调试，从较小的配置中“挑出”某些东西要简单得多。

+   在发生故障或更可能的是配置错误时，**散射区**或**影响范围**要小得多。如果将每个负载均衡器绑定到单个工作负载，任何错误或故障更可能只影响该工作负载。

+   此外，从运营的角度来看，**使用编排平台或 API 来扩展工作负载要简单得多**（根据需求增加或删除后端服务器到集群）。这种方法使得构建这些 playbook 要简单得多 - 主要是因为配置更简单，在 playbook 出错时影响范围更小。

+   **开发人员更快的部署**。由于您保持了这种简单的配置，在开发环境中，您可以在开发或修改应用程序时向开发人员提供这种配置。这意味着应用程序是针对负载均衡器编写的。此外，大部分测试是在开发周期内完成的，而不是在开发结束时在单个更改窗口中进行配置和测试。即使负载均衡器是有许可的产品，大多数供应商也为这种情况提供了免费（低带宽）许可的产品。

+   向开发人员或部署提供**安全配置的模板**要简单得多。

+   **在开发或 DevOps 周期中进行安全测试**包括负载均衡器，而不仅仅是应用程序和托管服务器。

+   **培训和测试要简单得多**。由于负载均衡产品是免费的，设置培训或测试环境是快速简单的。

+   **工作负载优化**是一个重要的优势，因为在虚拟化环境中，通常可以将一组服务器“绑定”在一起。例如，在 VMware vSphere 环境中，这被称为**vApp**。这个结构允许您将所有 vApp 成员一起保持在一起，例如，如果您将它们 vMotion 到另一个 hypervisor 服务器。您可能需要进行这样的操作进行维护，或者这可能会自动发生，使用**动态资源调度**（**DRS**），它可以在多个服务器之间平衡 CPU 或内存负载。或者，迁移可能是灾难恢复工作流的一部分，您可以将 vApp 迁移到另一个数据中心，使用 vMotion 或者简单地激活一组 VM 的副本。

+   **云部署更适合这种分布式模型**。在较大的云服务提供商中，这一点被推到了极致，负载平衡只是一个您订阅的服务，而不是一个离散的实例或虚拟机。其中包括 AWS 弹性负载均衡服务、Azure 负载均衡器和 Google 的云负载均衡服务。

负载平衡带来了一些管理挑战，其中大部分源于一个问题 - 如果所有目标主机都有负载平衡器的默认网关，我们如何监视和管理这些主机？

## 数据中心网络和管理考虑

如果使用 NAT 方法对工作负载进行负载平衡，路由就成了一个问题。潜在应用程序客户端的路由必须指向负载平衡器。如果这些目标是基于互联网的，这将使管理单个服务器成为一个问题 - 您不希望服务器管理流量被负载平衡。您也不希望不必要的流量（例如备份或大容量文件复制）通过负载平衡器路由 - 您希望它路由应用程序流量，而不是所有流量！

这通常通过添加静态路由和可能的管理 VLAN 来处理。

现在是一个好时机提出，管理 VLAN 应该从一开始就存在 - 我对管理 VLAN 的“赢得一点”的短语是“您的会计组（或接待员或制造组）需要访问您的 SAN 或超级管理员登录吗？”如果您可以得到一个答案，让您朝着保护内部攻击的敏感接口的方向，那么管理 VLAN 就很容易实现。

无论如何，在这种模型中，默认网关仍然指向负载平衡器（为了服务互联网客户端），但是特定路由被添加到服务器以指向内部或服务资源。在大多数情况下，这些资源的列表仍然很小，因此即使内部客户端计划使用相同的负载平衡应用程序，这仍然可以工作：

![图 10.7 - 路由非应用流量（高级）](img/B16336_10_007.jpg)

图 10.7 - 路由非应用流量（高级）

如果出于某种原因这种模型无法工作，那么您可能需要考虑添加**基于策略的路由**（**PBR**）。

在这种情况下，例如，您的服务器正在负载平衡 HTTP 和 HTTPS - 分别是`80/tcp`和`443/tcp`。您的策略可能如下所示：

+   将所有流量`80/tcp`和`443/tcp`路由到负载平衡器（换句话说，从应用程序的回复流量）。

+   将所有其他流量通过子网路由器路由。

这个策略路由可以放在服务器子网的路由器上，如下所示：

![图 10.8 - 路由非应用流量 - 在上游路由器上的策略路由](img/B16336_10_008.jpg)

图 10.8 - 路由非应用流量 - 在上游路由器上的策略路由

在上图中，服务器都有基于路由器接口的默认网关（在本例中为`10.10.10.1`）：

```
! this ACL matches reply traffic from the host to the client stations
ip access-list ACL-LB-PATH
   permit tcp any eq 443 any
   permit tcp any eq 90 any
! regular default gateway, does not use the load balancer, set a default gateway for that
ip route 0.0.0.0 0.0.0.0 10.10.x.1
! this sets the policy for the load balanced reply traffic
route-map RM-LB-PATH permit 10
   match ip address ACL-LB-BYPASS
   set next-hop 10.10.10.5
! this applies the policy to the L3 interface.
! note that we have a "is that thing even up" check before we forward the traffic
int vlan x
ip policy route-map RM-LB-PATH
 set ip next-hop verify-availability 10.10.10.5 1 track 1
 set ip next-hop 10.10.10.5
! track 1 is defined here
track 1 rtr 1 reachability
rtr 1
type echo protocol ipIcmpEcho 10.10.10.5
rtr schedule 1 life forever start-time now
```

这样做的好处是简单，但是这个子网默认网关设备必须有足够的性能来满足所有回复流量的需求，而不会影响其其他工作负载的性能。幸运的是，许多现代的 10G 交换机确实有这样的性能。然而，这也有一个缺点，即您的回复流量现在离开了超级管理员，到达了默认网关路由器，然后很可能再次进入虚拟基础设施以到达负载平衡器。在某些环境中，这在性能上仍然可以工作，但如果不行，考虑将策略路由移动到服务器本身。

要在 Linux 主机上实现相同的策略路由，按照以下步骤进行：

1.  首先，将路由添加到`表 5`：

```
ip route add table 5 0.0.0.0/0 via 10.10.10.5
```

1.  定义与负载平衡器匹配的流量（源`10.10.10.0/24`，源端口`443`）：

```
iptables -t mangle -A PREROUTING -i eth0 -p tcp -m tcp --sport 443 -s 10.10.10.0/24 -j MARK --set-mark 2
iptables -t mangle -A PREROUTING -i eth0 -p tcp -m tcp --sport 80 -s 10.10.10.0/24 -j MARK --set-mark 2
```

1.  添加查找，如下所示：

```
ip rule add fwmark 2 lookup 5
```

这种方法比大多数人想要的更复杂，CPU 开销也更大。另外，对于“网络路由问题”，支持人员更有可能在未来的故障排除中首先查看路由器和交换机，而不是主机配置。出于这些原因，我们经常看到将策略路由放在路由器或三层交换机上被实施。

使用管理接口更加优雅地解决了这个问题。另外，如果管理接口在组织中尚未广泛使用，这种方法可以很好地将其引入环境中。在这种方法中，我们保持目标主机配置为默认网关指向负载均衡器。然后，我们为每个主机添加一个管理 VLAN 接口，可能直接在该 VLAN 中提供一些管理服务。此外，根据需要，我们仍然可以添加到诸如 SNMP 服务器、日志服务器或其他内部或互联网目的地的特定路由：

![图 10.9 – 添加管理 VLAN](img/B16336_10_009.jpg)

图 10.9 – 添加管理 VLAN

不用说，这是常见的实施方式。这不仅是最简单的方法，而且还为架构添加了一个非常需要的管理 VLAN。

在大部分理论已经涵盖的情况下，让我们开始构建几种不同的负载均衡场景。

# 构建 HAProxy NAT/代理负载均衡器

首先，我们可能不想使用我们的示例主机，因此我们必须添加一个新的网络适配器来演示 NAT/代理（L4/L7）负载均衡器。

如果您的示例主机是虚拟机，构建一个新的应该很快。或者更好的是，克隆您现有的虚拟机并使用它。或者，您可以下载一个`haproxy –v`。

或者，如果您选择不使用我们的示例配置进行“构建”，您仍然可以“跟随”。虽然为负载均衡器构建管道可能需要一些工作，但实际配置非常简单，我们的目标是介绍您到该配置。您完全可以在不构建支持虚拟或物理基础设施的情况下实现这一目标。

如果您正在新的 Linux 主机上安装此软件，请确保您有两个网络适配器（一个面向客户端，一个面向服务器）。与往常一样，我们将从安装目标应用程序开始：

```
$ sudo apt-get install haproxy
```

*<如果您正在使用基于 OVA 的安装，请从这里开始：>*

您可以通过使用`haproxy`应用程序本身来检查版本号来验证安装是否成功：

```
$ haproxy –v
HA-Proxy version 2.0.13-2ubuntu0.1 2020/09/08 - https://haproxy.org/
```

请注意，任何新版本都应该可以正常工作。

安装了软件包后，让我们来看看我们的示例网络构建。

## 在开始配置之前 – 网卡、寻址和路由

您可以使用任何您选择的 IP 地址，但在我们的示例中，前端为`192.168.122.21/24`（请注意，这与主机的接口 IP 不同），而负载均衡器的后端地址将为`192.168.124.1/24` – 这将是目标主机的默认网关。我们的目标 Web 服务器将为`192.168.124.10`和`192.168.124.20`。

我们的最终构建将如下所示：

![图 10.10 – 负载均衡器示例构建](img/B16336_10_010.jpg)

图 10.10 – 负载均衡器示例构建

在我们开始构建负载均衡器之前，现在是调整 Linux 中一些设置的最佳时机（其中一些需要重新加载系统）。

## 在开始配置之前 – 性能调优

一个基本的“开箱即用”的 Linux 安装必须对各种设置做出一些假设，尽管其中许多会导致性能或安全方面的妥协。对于负载均衡器，有几个 Linux 设置需要解决。幸运的是，HAProxy 安装为我们做了很多这方面的工作（如果我们安装了许可版本）。安装完成后，编辑`/etc/sysctl.d/30-hapee-2.2.conf`文件，并取消注释以下代码中的行（在我们的情况下，我们正在安装社区版，因此创建此文件并取消注释这些行）。与所有基本系统设置一样，测试这些设置时，逐个或逻辑分组进行更改。此外，正如预期的那样，这可能是一个迭代过程，您可能需要在一个设置和另一个设置之间来回。正如文件注释中所指出的，并非所有这些值在所有情况下甚至在大多数情况下都是推荐的。

这些设置及其描述都可以在[`www.haproxy.com/documentation/hapee/2-2r1/getting-started/system-tuning/`](https://www.haproxy.com/documentation/hapee/2-2r1/getting-started/system-tuning/)找到。

限制每个套接字的默认接收/发送缓冲区，以限制在大量并发连接时的内存使用。这些值以字节表示，分别代表最小值、默认值和最大值。默认值是`4096`、`87380`和`4194304`：

```
    # net.ipv4.tcp_rmem            = 4096 16060 262144
    # net.ipv4.tcp_wmem            = 4096 16384 262144
```

允许对传出连接早期重用相同的源端口。如果每秒有几百个连接，这是必需的。默认值如下：

```
    # net.ipv4.tcp_tw_reuse        = 1
```

扩展传出 TCP 连接的源端口范围。这限制了早期端口重用，并使用了`64000`个源端口。默认值为`32768`和`61000`：

```
    # net.ipv4.ip_local_port_range = 1024 65023
```

增加 TCP SYN 积压大小。这通常需要支持非常高的连接速率，以及抵抗 SYN 洪水攻击。然而，设置得太高会延迟 SYN cookie 的使用。默认值是`1024`：

```
    # net.ipv4.tcp_max_syn_backlog = 60000
```

设置`tcp_fin_wait`状态的超时时间（以秒为单位）。降低它可以加快释放死连接，尽管它会在 25-30 秒以下引起问题。如果可能的话最好不要更改它。默认值为`60`：

```
    # net.ipv4.tcp_fin_timeout     = 30
```

限制传出 SYN-ACK 重试次数。这个值是 SYN 洪水的直接放大因子，所以保持它相当低是很重要的。然而，将它设置得太低会阻止丢包网络上的客户端连接。

使用`3`作为默认值可以得到良好的结果（总共 4 个 SYN-ACK），而在 SYN 洪水攻击下将其降低到`1`可以节省大量带宽。默认值为`5`：

```
    # net.ipv4.tcp_synack_retries  = 3
```

将其设置为`1`以允许本地进程绑定到系统上不存在的 IP。这通常发生在共享 VRRP 地址的情况下，您希望主备两者都启动，即使 IP 不存在。始终将其保留为`1`。默认值为`0`：

```
    # net.ipv4.ip_nonlocal_bind    = 1
```

以下作为系统所有 SYN 积压的上限。将它至少设置为`tcp_max_syn_backlog`一样高；否则，客户端可能在高速率或 SYN 攻击下连接时遇到困难。默认值是`128`：

```
     # net.core.somaxconn           = 60000
```

再次注意，如果您进行了任何这些更改，您可能会在以后回到这个文件来撤消或调整您的设置。完成所有这些（至少现在是这样），让我们配置我们的负载均衡器，使其与我们的两个目标 Web 服务器配合工作。

## 负载均衡 TCP 服务 - Web 服务

负载均衡服务的配置非常简单。让我们从在两个 Web 服务器主机之间进行负载均衡开始。

让我们编辑`/etc/haproxy/haproxy.cfg`文件。我们将创建一个`frontend`部分，定义面向客户端的服务，以及一个`backend`部分，定义两个下游 Web 服务器： 

```
frontend http_front
   bind *:80
   stats uri /haproxy?stats
   default_backend http_back
backend  http_back
   balance roundrobin
   server WEBSRV01 192.168.124.20:80 check fall 3 rise 2
   server WEBSRV02 192.168.124.21:80 check fall 3 rise 2
```

请注意以下内容：

+   前端部分中有一行`default backend`，告诉它将哪些服务绑定到该前端。

+   前端有一个`bind`语句，允许负载在该接口上的所有 IP 之间平衡。因此，在这种情况下，如果我们只使用一个 VIP 进行负载平衡，我们可以在负载均衡器的物理 IP 上执行此操作。

+   后端使用`roundrobin`作为负载平衡算法。这意味着当用户连接时，他们将被引导到 server1，然后是 server2，然后是 server1，依此类推。

+   `check`参数告诉服务检查目标服务器以确保其正常运行。当负载平衡 TCP 服务时，这要简单得多，因为简单的 TCP“连接”就可以解决问题，至少可以验证主机和服务是否正在运行。

+   `fall 3`在连续三次失败的检查后将服务标记为离线，而`rise 2`在两次成功的检查后将其标记为在线。这些 rise/fall 关键字可以在使用任何检查类型时使用。

我们还希望在此文件中有一个全局部分，以便我们可以设置一些服务器参数和默认值：

```
global
    maxconn 20000
    log /dev/log local0
    user haproxy
    group haproxy
    stats socket /run/haproxy/admin.sock user haproxy group haproxy mode 660 level admin
    nbproc 2
    nbthread 4
    timeout http-request <timeout>
    timeout http-keep-alive <timeout>
    timeout queue <timeout>
    timeout client-fin <timeout>
    timeout server-fin <timeout>
    ssl-default-bind-ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
```

请注意，我们在此部分定义了用户和组。回溯到*第三章*，*使用 Linux 和 Linux 工具进行网络诊断*，我们提到，如果端口号小于`1024`，您需要具有 root 权限才能启动侦听端口。对于 HAProxy 来说，这意味着它需要 root 权限来启动服务。全局部分中的用户和组指令允许服务“降级”其权限。这很重要，因为如果服务被攻击，拥有较低权限会给攻击者提供更少的选项，可能增加攻击所需的时间，并希望增加他们被抓获的可能性。

`log`行非常直接 - 它告诉`haproxy`将日志发送到哪里。如果您有任何需要解决的负载平衡问题，这是一个很好的起点，接着是目标服务的日志。

`stats`指令告诉`haproxy`存储其各种性能统计信息的位置。

`nbproc`和`nbpthread`指令告诉 HAProxy 服务可用于使用的处理器和线程数量。这些数字应该至少比可用的进程少一个，以便在拒绝服务攻击发生时，整个负载平衡器平台不会瘫痪。

各种超时参数用于防止协议级拒绝服务攻击。在这些情况下，攻击者发送初始请求，但随后从不继续会话 - 他们只是不断发送请求，“耗尽”负载均衡器资源，直到内存完全耗尽。这些超时限制了负载均衡器将保持任何一个会话活动的时间。下表概述了我们在这里讨论的每个保持活动参数的简要描述：

![](img/B16336_10_Table_02.jpg)

此外，SSL 指令相当容易理解：

+   `ssl-default-bind-ciphers`列出了在任何 TLS 会话中允许的密码，如果负载均衡器正在终止或启动会话（即，如果您的会话处于代理或第 7 层模式）。

+   `ssl-default-bind-options`用于设置支持的 TLS 版本的下限。在撰写本文时，所有 SSL 版本以及 TLS 版本 1.0 都不再推荐使用。特别是 SSL 容易受到多种攻击。由于所有现代浏览器都能够协商 TLS 版本 3，大多数环境选择支持 TLS 版本 1.2 或更高版本（如示例中所示）。

现在，从客户端机器，您可以浏览到 HAProxy 主机，您会看到您将连接到其中一个后端。如果您尝试从不同的浏览器再次连接，您应该连接到第二个。

让我们扩展一下，为 HTTPS（在`443/tcp`上）添加支持。我们将在前端接口上添加一个 IP 并绑定到该 IP。我们将把平衡算法更改为最少连接。最后，我们将更改前端和后端的名称，以包括端口号。这使我们能够为`443/tcp`添加额外的配置部分。如果我们只监视第 4 层 TCP 会话，这些流量将得到很好的负载平衡；不需要解密：

```
frontend http_front-80
   bind 192.168.122.21:80
   stats uri /haproxy?stats
   default_backend http_back-80
frontend http_front-443
   bind 192.168.122.21:443
   stats uri /haproxy?stats
   default_backend http_back-443
backend  http_back-80
   balance leastconn
   server WEBSRV01 192.168.124.20:80 check fall 3 rise 2
   server WEBSRV02 192.168.124.21:80 check fall 3 rise 2
backend  http_back-443
   balance leastconn
   server WEBSRV01 192.168.124.20:443 check fall 3 rise 2
   server WEBSRV02 192.168.124.21:443 check fall 3 rise 2
```

请注意，我们仍然只是检查 TCP 端口是否对“服务器健康”检查打开。这通常被称为第 3 层健康检查。我们将端口`80`和`443`放入两个部分 - 这些可以合并到前端段的一个部分中，但通常最好将它们分开以便可以分别跟踪它们。这样做的副作用是两个后端部分的计数不会相互影响，但通常这不是一个问题，因为如今整个 HTTP 站点通常只是重定向到 HTTPS 站点。

另一种表达方式是在`listen`段上，而不是在前端和后端段上。这种方法将前端和后端部分合并到一个段中，并添加一个“健康检查”：

```
listen webserver 192.168.122.21:80
    mode http
    option httpchk HEAD / HTTP/1.0
    server websrv01 192.168.124.20:443 check fall 3 rise 2
    server websrv02 192.168.124.21:443 check fall 3 rise 2
```

这个默认的 HTTP 健康检查只是打开默认页面，并通过检查标题中的短语`HTTP/1.0`来确保有内容返回。如果在返回的页面中没有看到这个短语，就算作是一次失败的检查。您可以通过检查站点上的任何 URI 并查找该页面上的任意文本字符串来扩展此功能。这通常被称为“第 7 层”健康检查，因为它正在检查应用程序。但是请确保您的检查简单 - 如果应用程序即使稍微更改，页面返回的文本可能会发生足够的变化，导致您的健康检查失败，并意外地标记整个集群为离线！

## 建立持久（粘性）连接

让我们通过使用服务器名称的变体将 cookie 注入到 HTTP 会话中。我们还将对 HTTP 服务进行基本检查，而不仅仅是开放端口。我们将回到我们的“前端/后端”配置文件方法：

```
backend  http_back-80
   mode http
   balance leastconn
   cookie SERVERUSED insert indirect nocache
   option httpchk HEAD /
   server WEBSRV01 192.168.124.20:80 cookie WS01 check fall 3 rise 2
   server WEBSRV02 192.168.124.21:80 cookie WS02 check fall 3 rise 2
```

确保您不要使用服务器的 IP 地址或真实名称作为 cookie 值。如果使用真实服务器名称，攻击者可能会通过在 DNS 中查找该服务器名称或在具有历史 DNS 条目数据库的站点（例如`dnsdumpster.com`）来访问该服务器。服务器名称也可以用来从证书透明日志中获取有关目标的信息（正如我们在[*第八章*]（B16336_08_Final_NM_ePub.xhtml#_idTextAnchor133）中讨论的那样，*Linux 上的证书服务*）。最后，如果服务器 IP 地址用于 cookie 值，该信息将使攻击者对您的内部网络架构有所了解，如果披露的网络是公共可路由的，可能会成为他们的下一个目标！

## 实施说明

现在我们已经介绍了基本配置，一个非常常见的步骤是在每台服务器上都有一个“占位符”网站，每个网站都被标识为与服务器匹配。使用“1-2-3”，“a-b-c”或“red-green-blue”都是常见的方法，足以区分每个服务器会话。现在，使用不同的浏览器或不同的工作站，多次浏览共享地址，以确保您被定向到正确的后端服务器，如您的规则集所定义的那样。

当然，这是一个逐步构建配置的好方法，以确保事情正常运行，但它也是一个很好的故障排除机制，可以帮助您决定一些简单的事情，比如“更新后这还有效吗？”或“我知道帮助台票说了什么，但是真的有问题要解决吗？”甚至几个月甚至几年后。像这样的测试页面是一个很好的长期保留的东西，用于未来的测试或故障排除。

## HTTPS 前端

过去，服务器架构师乐意设置负载均衡器来卸载 HTTPS 处理，将加密/解密处理从服务器转移到负载均衡器。这样可以节省服务器 CPU，并且将实施和维护证书的责任转移到管理负载均衡器的人。然而，出于几个原因，这些原因现在大多数已经不再有效：

+   如果服务器和负载均衡器都是虚拟的（在大多数情况下是推荐的），这只是在不同虚拟机之间移动处理 - 没有净增益。

+   现代处理器在执行加密和解密方面效率更高 - 算法是针对 CPU 性能编写的。事实上，根据算法的不同，加密/解密操作可能是 CPU 的本地操作，这是一个巨大的性能提升。

+   使用通配符证书可以使整个“证书管理”过程变得更简单。

然而，我们仍然使用负载均衡器进行 HTTPS 前端处理，通常是为了使用 cookie 实现可靠的会话持久性 - 除非你能读取和写入数据流，否则无法在 HTTPS 响应中添加 cookie（或在下一个请求中读取），这意味着在某个时刻它已经被解密。

请记住，根据我们之前的讨论，在这个配置中，每个 TLS 会话将在前端终止，使用有效的证书。由于现在这是一个代理设置（第 7 层负载平衡），后端会话是一个单独的 HTTP 或 HTTPS 会话。在过去，后端通常会是 HTTP（主要是为了节省 CPU 资源），但在现代，这将被视为安全风险，特别是如果你在金融、医疗保健或政府部门（或任何承载敏感信息的部门）。因此，在现代构建中，后端几乎总是会是 HTTPS，通常使用目标 Web 服务器上相同的证书。

再次强调这种设置的缺点是，由于目标 Web 服务器的实际客户端是负载均衡器，`X-Forwarded-*` HTTPS 头将丢失，并且实际客户端的 IP 地址将不可用于 Web 服务器（或其日志）。

我们如何配置这个设置？首先，我们必须获取站点证书和私钥，无论是“命名证书”还是通配符。现在，将它们合并成一个文件（不是作为`pfx`文件，而是作为一个链），只需使用`cat`命令简单地将它们连接在一起：

```
cat sitename.com.crt sitename.com.key | sudo tee /etc/ssl/sitename.com/sitename.com.pem
```

请注意，在命令的后半部分我们使用了`sudo`，以赋予命令对`/etc/ssl/sitename.com`目录的权限。还要注意`tee`命令，它会将命令的输出显示在屏幕上，并将输出定向到所需的位置。

现在，我们可以将证书绑定到前端文件段中的地址：

```
frontend http front-443
    bind 192.168.122.21:443 ssl crt /etc/ssl/sitename.com/sitename.com.pem
    redirect scheme https if !{ ssl_fc }
    mode http
    default_backend back-443
backend back-443
    mode http
    balance leastconn
    option forwardfor
    option httpchk HEAD / HTTP/1.1\r\nHost:localhost
    server web01 192.168.124.20:443 cookie WS01 check fall 3 rise 2
    server web02 192.168.124.21:443 cookie WS02 check fall 3 rise 2
    http-request add-header X-Forwarded-Proto https 
```

在这个配置中，请注意以下内容：

+   现在我们可以在后端部分使用 cookie 来实现会话持久性，这通常是这个配置中的主要目标。

+   我们在前端使用`redirect scheme`行指示代理在后端使用 SSL/TLS。

+   `forwardfor`关键字将实际客户端 IP 添加到后端请求的`X-Forwarded-For` HTTP 头字段中。请注意，需要由 Web 服务器解析这些内容并适当地记录，以便以后使用。

根据应用程序和浏览器的不同，你还可以在`X-Client-IP`头字段中添加客户端 IP 到后端 HTTP 请求中：

```
http-request set-header X-Client-IP %[req.hdr_ip(X-Forwarded-For)]
```

注意

这种方法效果参差不齐。

然而，请注意，无论你在 HTTP 头中添加或更改什么，目标服务器“看到”的实际客户端 IP 地址仍然是负载均衡器的后端地址 - 这些更改或添加的头值只是 HTTPS 请求中的字段。如果你打算使用这些头值进行日志记录、故障排除或监控，就需要 Web 服务器来解析它们并适当地记录。

这涵盖了我们的示例配置 - 我们涵盖了基于 NAT 和基于代理的负载平衡，以及 HTTP 和 HTTPS 流量的会话持久性。在所有理论之后，实际配置负载均衡器很简单 - 工作都在设计和设置支持网络基础设施中。在结束本章之前，让我们简要讨论一下安全性。

# 关于负载均衡器安全性的最后一点说明

到目前为止，我们已经讨论了攻击者如何能够获得有关内部网络的见解或访问权限，如果他们可以获得服务器名称或 IP 地址。我们讨论了恶意行为者如何使用本地负载均衡器配置中披露的信息来获取这些信息以进行持久设置。攻击者还可以以其他方式获取有关我们的目标服务器（这些服务器位于负载均衡器后面并且应该被隐藏）的信息吗？

证书透明信息是获取当前或旧服务器名称的另一种常用方法，正如我们在*第八章*中讨论的那样，*Linux 上的证书服务*。即使旧的服务器名称不再使用，其过去证书的记录是永恒的。

互联网档案馆网站[`archive.org`](https://archive.org)定期对网站进行“快照”，并允许搜索和查看它们，使人们可以“回到过去”并查看您基础设施的旧版本。如果旧服务器在您的旧 DNS 或 Web 服务器的旧代码中披露，它们很可能可以在此网站上找到。

DNS 存档网站，如`dnsdumpster`，使用被动方法（如数据包分析）收集 DNS 信息，并通过 Web 或 API 界面呈现。这使攻击者可以找到旧的 IP 地址和旧（或当前）主机名，组织有时可以通过 IP 仍然访问这些服务，即使 DNS 条目被删除。或者，他们可以通过主机名单独访问它们，即使它们在负载均衡器后面。

*Google Dorks*是获得此类信息的另一种方法 - 这些术语用于在搜索引擎（不仅仅是 Google）中查找特定信息。通常，像`inurl:targetdomain.com`这样的搜索词将找到目标组织宁愿保持隐藏的主机名。一些特定于`haproxy`的 Google Dorks 包括以下内容：

```
intitle:"Statistics Report for HAProxy" + "statistics report for pid" site:www.targetdomain.com 
inurl:haproxy-status site:target.domain.com
```

请注意，在我们说`site:`时，您也可以指定`inurl:`。在这种情况下，您还可以将搜索词缩短为域而不是完整的站点名称。

诸如`shodan.io`之类的网站还将索引您服务器的历史版本，重点关注服务器 IP 地址，主机名，开放端口以及在这些端口上运行的服务。 Shodan 在识别开放端口上运行的服务方面非常独特。当然，他们在这方面并不百分之百成功（将其视为他人的 NMAP 结果），但是当他们识别服务时，会附上“证据”，因此，如果您使用 Shodan 进行侦察，您可以使用它来验证该确定可能有多准确。Shodan 既有 Web 界面又有全面的 API。通过这项服务，您通常可以按组织或地理区域找到未经适当保护的负载均衡器。

最后对搜索引擎的评论：如果 Google（或任何搜索引擎）可以直接访问您的真实服务器，那么该内容将被索引，使其易于搜索。如果网站可能存在身份验证绕过问题，则“受身份验证保护”的内容也将被索引，并可供使用该引擎的任何人使用。

也就是说，始终使用我们刚刚讨论过的工具定期查找外围基础设施上的问题是一个好主意。

另一个重要的安全问题是管理访问。重要的是要限制对负载均衡器的管理界面（即 SSH）的访问，将其限制在所有接口上的允许主机和子网。请记住，如果您的负载均衡器与防火墙平行，整个互联网都可以访问它，即使不是这样，您内部网络上的每个人也可以访问它。您需要将访问权限缩减到可信任的管理主机和子网。如果您需要参考，记住我们在*第四章*中涵盖了这一点，*Linux 防火墙*，以及*第五章*，*具有实际示例的 Linux 安全标准*。

# 总结

希望本章对负载均衡器的介绍、部署以及您可能选择围绕它们做出各种设计和实施决策的原因有所帮助。

如果您在本章节中使用新的虚拟机来跟随示例，那么在接下来的章节中我们将不再需要它们，但是如果您需要以后参考示例，您可能希望保留 HAProxy 虚拟机。如果您只是通过阅读本章中的示例来跟随，那么本章中的示例仍然对您可用。无论哪种方式，当您阅读本章时，我希望您能够在脑海中思考负载均衡器如何适应您组织的内部或边界架构。

完成本章后，您应该具备在任何组织中构建负载均衡器所需的技能。这些技能是在（免费）版本的 HAProxy 的背景下讨论的，但设计和实施考虑几乎都可以直接在任何供应商的平台上使用，唯一的变化是配置选项或菜单中的措辞和语法。在下一章中，我们将看一下基于 Linux 平台的企业路由实现。

# 问题

最后，这里有一些问题供您测试对本章材料的了解。您将在*附录*的*评估*部分中找到答案：

1.  何时您会选择使用**直接服务器返回**（**DSR**）负载均衡器？

1.  为什么您会选择使用基于代理的负载均衡器，而不是纯 NAT-based 解决方案的负载均衡器？

# 进一步阅读

查看以下链接，了解本章涵盖的主题的更多信息：

+   HAProxy 文档：[`www.haproxy.org/#docs`](http://www.haproxy.org/#docs)

+   HAProxy 文档（商业版本）：[`www.haproxy.com/documentation/hapee/2-2r1/getting-started/`](https://www.haproxy.com/documentation/hapee/2-2r1/getting-started/)

+   HAProxy GitHub：[`github.com/haproxytech`](https://github.com/haproxytech)

+   HAProxy GitHub，OVA 虚拟机下载：[`github.com/haproxytech/vmware-haproxy#download`](https://github.com/haproxytech/vmware-haproxy#download)

+   HAProxy 社区与企业版本的区别：[`www.haproxy.com/products/community-vs-enterprise-edition/`](https://www.haproxy.com/products/community-vs-enterprise-edition/)

+   有关负载均衡算法的更多信息：[`cbonte.github.io/haproxy-dconv/2.4/intro.html#3.3.5`](http://cbonte.github.io/haproxy-dconv/2.4/intro.html#3.3.5)
