# 第八章：*第八章*：Linux 上的证书服务

在本章中，我们将涵盖几个涉及在 Linux 中使用证书来保护或加密流量，并特别是配置和使用各种**证书颁发机构**（**CA**）服务器的主题。

我们将介绍这些证书的基本用途，然后继续构建证书服务器。最后，我们将讨论围绕证书服务的安全考虑，无论是在保护 CA 基础设施还是使用**证书透明度**（**CT**）来强制信任模型，以及在组织内进行清单/审计或侦察。

在本章中，我们将涵盖以下主题：

+   证书是什么？

+   获取证书

+   使用证书-Web 服务器示例

+   构建私有证书颁发机构

+   保护您的证书颁发机构基础设施

+   证书透明度

+   证书自动化和**自动证书管理环境**（**ACME**）协议

+   `OpenSSL`速查表

当我们完成本章时，您将在 Linux 主机上拥有一个可用的私有 CA，并且对证书的发放方式以及如何管理和保护您的 CA 都有一个很好的理解，无论您是在实验室还是生产环境中使用它。您还将对标准证书握手的工作原理有很好的理解。

让我们开始吧！

# 技术要求

在本章中，我们可以继续使用同一台 Ubuntu **虚拟机**（**VM**）或工作站，因为这是一个学习练习。即使在我们既是 CA 又是证书申请人的部分，本节中的示例也都可以在这一台主机上完成。

鉴于我们正在构建证书服务器，如果您正在使用此指南来帮助构建生产主机，强烈建议您在单独的主机或虚拟机上构建。虚拟机是生产服务的首选-请阅读“保护您的 CA 基础设施”部分，了解更多建议。

# 证书是什么？

证书本质上是*真相的证明* - 换句话说，证书是一份声明，“相信我，这是真的”。这听起来很简单，某种程度上确实如此。但在其他方面，证书的各种用途以及安全地部署 CA 基础设施是一个重大挑战-例如，我们在最近几年看到了一些公共 CA 的惊人失败：那些唯一业务是保护证书流程的公司在受到审查时却无法做到。我们将在本章后面的*保护您的 CA 基础设施*和*CT*部分更详细地介绍保护 CA 的挑战和解决方案。

从根本上讲，工作站和服务器都信任一系列 CA。这种信任是通过使用加密签名的文档来传递的，这些文档是每个 CA 的公共证书，存储在 Linux 或 Windows 主机的特定位置上。

例如，当您浏览到一个 Web 服务器时，本地的*证书存储*会被引用，以查看我们是否应该信任 Web 服务器的证书。这是通过查看该 Web 服务器的公共证书，并查看它是否由您信任的 CA 之一（或其下属）签名而完成的。实际签名使用*子*或*下属* CA 是常见的-每个公共 CA 都希望尽可能保护其*根* CA，因此创建了*下属 CA*或*颁发 CA*，这些是公共互联网所见的。

组织可以创建自己的 CA，用于验证其用户、服务器、工作站和网络基础设施之间的身份和授权。这使得信任保持在“家庭”内，完全受到组织的控制。这也意味着组织可以使用内部和免费的证书服务，而不必为数百或数千个工作站或用户证书付费。

现在我们知道了证书是什么，让我们看看它们是如何颁发的。

# 获取证书

在下图中，一个应用程序 - 例如，一个 Web 服务器 - 需要一个证书。这个图看起来复杂，但我们将把它分解成简单的步骤：

![图 8.1 - 证书签名请求（CSR）和颁发证书](img/B16336_08_001.jpg)

图 8.1 - 证书签名请求（CSR）和颁发证书

让我们逐步了解创建证书涉及的步骤，从最初的请求到准备在目标应用程序中安装证书（*步骤 1-6*），如下所示：

1.  该过程从创建 CSR 开始。这只是一个简短的文本文件，用于标识请求证书的服务器/服务和组织。这个文件在加密时被“混淆” - 虽然字段是标准化的，只是文本，但最终结果不是人类可读的。然而，诸如 OpenSSL 之类的工具可以读取 CSR 文件和证书本身（如果需要示例，请参见本章末尾的*OpenSSL 备忘单*部分）。CSR 的文本信息包括这些标准字段的一些或全部：![](img/B16336_08_Table_01.jpg)

上述列表并非 CSR 中可以使用的字段的详尽列表，但这些是最常见的字段。

我们需要所有这些信息的原因是，当客户端连接到使用证书的服务时（例如，使用**超文本传输安全协议**（**HTTPS**）和**传输层安全**（**TLS**）的 Web 服务器），客户端可以验证连接到的服务器名称是否与 CN 字段或其中一个 SAN 条目匹配。

这使得 CA 操作员验证这些信息变得很重要。对于面向公众的证书，操作员/供应商通过验证公司名称、电子邮件等信息来完成这一过程。自动化解决方案通过验证您对域或主机具有管理控制权来实现这一点。

1.  仍然遵循*图 8.1*，接下来将这些文本信息与申请人的公钥加密组合，形成`CSR`文件。

1.  现在完成的 CSR 被发送到 CA。当 CA 是公共 CA 时，通常通过网站完成。自动化的公共 CA（如**Let's Encrypt**）通常使用 ACME **应用程序编程接口**（**API**）在申请人和 CA 之间进行通信。在高风险的实施中，*步骤 3*和*6*可能使用安全媒体，通过正式的*保管链*程序在受信任的各方之间物理交接。重要的是申请人和 CA 之间的通信使用一些安全的方法。虽然可能存在较不安全的方法，如电子邮件，但不建议使用。

1.  在 CA 处，身份信息（我们仍然遵循*图 8.1*中的信息流）得到验证。这可能是一个自动化或手动的过程，取决于几个因素。例如，如果这是一个公共 CA，您可能已经有一个帐户，这将使半自动化检查更有可能。如果您没有帐户，这个检查很可能是手动的。对于私人 CA，这个过程可能是完全自动化的。

1.  一旦验证，验证的 CSR 将与 CA 的私钥加密组合，创建最终的证书。

1.  然后将此证书发送回申请人，并准备安装到将使用该证书的应用程序中。

请注意，在此交易中，申请人的私钥从未被使用 - 我们将在 TLS 密钥交换中看到它在哪里使用（在本章的下一节）。

现在我们了解了证书是如何创建或发布的，应用程序如何使用证书来信任服务或加密会话流量呢？让我们看看浏览器和受 TLS 保护的网站之间的交互，以了解这是如何工作的。

# 使用证书 - Web 服务器示例

当被问及时，大多数人会说证书最常见的用途是使用 HTTPS 协议保护网站。虽然这可能不是当今互联网上证书最常见的用途，但它确实仍然是最显眼的。让我们讨论一下 Web 服务器的证书如何用于在服务器中提供信任并帮助建立加密的 HTTPS 会话。

如果你还记得我们 CSR 示例中的*申请人*，在这个例子中，申请人是[www.example.com](http://www.example.com)这个网站，可能驻留在 Web 服务器上。我们将从上一个会话结束的地方开始我们的例子——证书已经颁发并安装在 Web 服务器上，准备好接受客户端连接。

**步骤 1**：客户端向 Web 服务器发出初始的 HTTPS 请求，称为**客户端 HELLO**（*图 8.2*）。

在这个初始的*Hello*交换中，客户端向服务器发送以下内容：

+   它支持的 TLS 版本

+   它支持的加密密码

这个过程在下面的图表中有所说明：

![图 8.2 – TLS 通信从客户端 hello 开始](img/B16336_08_002.jpg)

图 8.2 – TLS 通信从客户端 hello 开始

Web 服务器通过发送其证书进行回复。如果你还记得，证书包含几个信息。

**步骤 2**：Web 服务器通过发送其证书（*图 8.3*）进行回复。如果你还记得，证书包含以下几个信息：

+   陈述服务器身份的文本信息

+   Web 服务器/服务的公钥

+   CA 的身份

服务器还发送以下内容：

+   支持的 TLS 版本

+   它在密码中的第一个提议（通常是服务器支持的客户端列表中最高强度的密码）

这个过程在下面的图表中有所说明：

![图 8.3 – TLS 交换：服务器 hello 被发送并由客户端验证](img/B16336_08_003.jpg)

图 8.3 – TLS 交换：服务器 hello 被发送并由客户端验证

**步骤 3**：客户端接收此证书和其他信息（称为服务器 hello），然后（如*图 8.4*中所示）验证一些信息，如下所示：

+   我刚刚收到的证书中是否包含我请求的服务器的身份（通常会在 CN 字段或 SAN 字段中）？

+   今天的日期/时间是否在证书的*之后*和*之前*日期之间（也就是说，证书是否已过期）？

+   我信任 CA 吗？它将通过查看其证书存储来验证这一点，其中通常包含几个 CA 的公共证书（几个公共 CA，通常还有一个或多个在组织内部使用的私有 CA）。

+   客户端还有机会通过向**在线证书状态协议**（**OCSP**）服务器发送请求来检查证书是否已被吊销。检查**证书吊销列表**（**CRL**）的旧方法仍然受到支持，但不再经常使用——这个列表被证明在成千上万的吊销证书中不太适用。在现代实现中，CRL 通常由已吊销的公共 CA 证书组成，而不是常规服务器证书。

+   *信任*和*吊销*检查非常重要。这些检查验证服务器是否是其所声称的。如果这些检查没有进行，那么任何人都可以建立一个声称是你的银行的服务器，你的浏览器就会让你登录到这些恶意服务器上。现代网络钓鱼活动经常试图通过*相似域*和其他方法来*欺骗系统*，让你做这样的事情。

**步骤 4**：如果证书在客户端通过了所有检查，客户端将生成一个伪随机对称密钥（称为预主密钥）。这个密钥使用服务器的公钥加密并发送给服务器（如*图 8.4*所示）。这个密钥将用于加密实际的 TLS 会话。

在这一点上，客户端被允许修改密码。最终密码是客户端和服务器之间的协商-请记住这一点，因为当我们讨论攻击和防御时，我们将深入探讨这一点。长话短说-客户端通常不会更改密码，因为服务器已经选择了来自客户端列表的密码。

该过程在以下图表中说明：

![图 8.4 - 客户端密钥交换，服务器有最后一次机会更改密码](img/B16336_08_004.jpg)

图 8.4 - 客户端密钥交换，服务器有最后一次机会更改密码

**步骤 5**：在这一步之后，服务器也有最后一次更改密码的机会（仍在*图 8.4*中）。这一步通常不会发生，密码协商通常已经完成。预主密钥现在已经最终确定，并称为主密钥。

**步骤 6**：现在证书验证已经完成，密码和对称密钥都已经达成一致，通信可以继续进行。加密是使用上一步的对称密钥进行的。

这在以下图表中说明：

![图 8.5 - 协商完成，通信使用主密钥（密钥）进行加密进行](img/B16336_08_005.jpg)

图 8.5 - 协商完成，通信使用主密钥（密钥）进行加密进行

在这种交换中有两个重要的事情暗示但尚未明确说明，如下：

+   一旦协商完成，证书将不再使用-加密将使用协商的主密钥进行。

+   在正常的协商过程中，不需要 CA。当我们开始讨论保护组织的 CA 基础设施时，这将成为一个重要的观点。

现在我们对证书的工作原理有了更好的理解（至少在这个用例中），让我们为我们的组织构建一个基于 Linux 的 CA。我们将以几种不同的方式进行此操作，以便为您的组织提供一些选项。我们还将在下一章[*第九章*]（B16336_09_Final_NM_ePub.xhtml#_idTextAnchor153），*Linux 的 RADIUS 服务*中使用 CA，因此这是一组重要的示例，需要密切关注。

# 构建私有证书颁发机构

构建私有 CA 始于我们面临的每个基础设施包的相同决定：*我们应该使用哪个 CA 包？*与许多服务器解决方案一样，有几种选择。以下概述了一些选项：

+   **OpenSSL**在技术上为我们提供了编写自己的脚本和维护**公钥基础设施**（**PKI**）位和片段的目录结构的所有工具。您可以创建根和从属 CA，制作 CSR，然后签署这些证书以制作真正的证书。实际上，虽然这种方法得到了普遍支持，但对大多数人来说，它最终变得有点太过于手动化。

+   **证书管理器**是与 Red Hat Linux 和相关发行版捆绑在一起的 CA。

+   **openSUSE**和相关发行版可以使用本机**另一种设置工具**（**YaST**）配置和管理工具作为 CA。

+   **Easy-RSA**是一组脚本，本质上是对相同的 OpenSSL 命令的包装。

+   **Smallstep**实现更多自动化-它可以配置为私有 ACME 服务器，并且可以轻松允许您的客户请求和履行其自己的证书。

+   `LetsEncrypt` GitHub 页面并用 Go 编写。

正如您所看到的，有相当多的 CA 包可供选择。大多数较旧的包都是对各种 OpenSSL 命令的包装。较新的包具有额外的自动化功能，特别是围绕 ACME 协议，这是由`LetsEncrypt`首创的。先前提到的每个包的文档链接都在本章的*进一步阅读*列表中。作为最广泛部署的 Linux CA，我们将使用 OpenSSL 构建我们的示例 CA 服务器。

## 使用 OpenSSL 构建 CA

因为我们只使用几乎每个 Linux 发行版都包含的命令，所以在开始使用此方法构建我们的 CA 之前，无需安装任何内容。

让我们按照以下步骤开始这个过程：

1.  首先，我们将为 CA 创建一个位置。`/etc/ssl`目录应该已经存在于您的主机文件结构中，我们将通过运行以下代码向其中添加两个新目录：

```
$ sudo mkdir /etc/ssl/CA
$ sudo mkdir /etc/ssl/newcerts
```

1.  接下来，请记住，随着证书的发放，CA 需要跟踪序列号（通常是顺序的），以及关于每个证书的一些详细信息。让我们在`serial`文件中开始序列号，从`1`开始，并创建一个空的`index`文件来进一步跟踪证书，如下所示：

```
sudo syntax when creating a serial file. This is needed because if you just use sudo against the echo command, you don't have rights under the /etc directory. What this syntax does is start a sh temporary shell and pass the character string in quotes to execute using the -c parameter. This is equivalent to running sudo sh or su, executing the command, and then exiting back to the regular user context. However, using sudo sh –c is far preferable to these other methods, as it removes the temptation to stay in the root context. Staying in the root context brings with it all kinds of opportunities to mistakenly and permanently change things on the system that you didn't intend—anything from accidentally deleting a critical file (which only root has access to), right up to—and including—mistakenly installing malware, or allowing ransomware or other malware to run as root.
```

1.  接下来，我们将编辑现有的`/etc/ssl/openssl.cnf`配置文件，并导航到`[CA_default]`部分。默认文件中的此部分如下所示：

```
private_key line, but be sure to double-check it for correctness while you are in the file.
```

1.  接下来，我们将创建一个自签名的根证书。这对于私有 CA 的根是正常的。（在公共 CA 中，您将创建一个新的 CSR 并让另一个 CA 对其进行签名，以提供对受信任根的*链*。）

由于这是一个组织的内部 CA，我们通常会选择一个很长的寿命，这样我们就不必每一两年重建整个 CA 基础设施。让我们选择 10 年（3,650 天）。请注意，此命令要求输入密码（不要丢失！）以及其他将标识证书的信息。请注意在以下代码片段中，`openssl`命令一步创建了 CA（`cakey.pem`）和根证书（`cacert.pem`）的私钥。在提示时，请使用您自己的主机和公司信息填写请求的值：

```
$ openssl req -new -x509 -extensions v3_ca -keyout cakey.pem -out cacert.pem -days 3650
Generating a RSA private key
...............+++++
.................................................+++++
writing new private key to 'cakey.pem'
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CA
State or Province Name (full name) [Some-State]:ON
Locality Name (eg, city) []:MyCity
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Coherent Security
Organizational Unit Name (eg, section) []:IT
Common Name (e.g. server FQDN or YOUR name) []:ca01.coherentsecurity.com
Email Address []:
```

1.  在最后一步中，我们将密钥和根证书移动到正确的位置。请注意，您需要再次拥有`sudo`权限才能执行此操作。

```
mv command. In security engagements, it's common to find certificates and keys stored in all sorts of temporary or archive locations—needless to say, if an attacker is able to obtain the root certificate and private key for your certificate server, all sorts of shenanigans can result!
```

您的 CA 现在已经开业！让我们继续创建 CSR 并对其进行签名。

## 请求和签署 CSR

让我们创建一个测试 CSR——您可以在我们一直在使用的相同示例主机上执行此操作。首先，为此证书创建一个私钥，如下所示：

```
$ openssl genrsa -des3 -out server.key 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
...............................................+++++
........................+++++
e is 65537 (0x010001)
Enter pass phrase for server.key:
Verifying - Enter pass phrase for server.key:
```

请记住该密码，因为在安装证书时将需要它！还要注意，该密钥具有`2048`位模数——这是您应该期望在此目的上看到或使用的最小值。

证书密钥的密码非常重要且非常敏感，您应该将它们存储在安全的地方——例如，如果您计划在证书到期时（或者希望在此之前）更新该证书，您将需要该密码来完成该过程。我建议不要将其保存在纯文本文件中，而是建议使用密码保险库或密码管理器来存储这些重要的密码。

请注意，许多守护程序样式的服务将需要没有密码的密钥和证书（例如 Apache Web 服务器、Postfix 和许多其他服务），以便在没有干预的情况下自动启动。如果您为这样的服务创建密钥，我们将去除密码以创建一个*不安全的密钥*，如下所示：

```
$ openssl rsa -in server.key -out server.key.insecure
Enter pass phrase for server.key:
writing RSA key
```

现在，让我们重命名密钥——`server.key`的*安全*密钥变为`server.key.secure`，而`server.key.insecure`的*不安全*密钥变为`server.key`，如下面的代码片段所示：

```
$ mv server.key server.key.secure
$ mv server.key.insecure server.key
```

无论我们创建哪种类型的密钥（带有或不带有密码），最终文件都是`server.key`。使用此密钥，我们现在可以创建 CSR。此步骤需要另一个密码，该密码将用于签署 CSR，如下面的代码片段所示：

```
~$ openssl req -new -key server.key -out server.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CA
State or Province Name (full name) [Some-State]:ON
Locality Name (eg, city) []:MyCity
Organization Name (eg, company) [Internet Widgits Pty Ltd]:Coherent Security
Organizational Unit Name (eg, section) []:IT
Common Name (e.g. server FQDN or YOUR name) []:www.coherentsecurity.com
Email Address []:
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:passphrase
An optional company name []:
```

现在我们在`server.csr`文件中有了 CSR，它已经准备好被签名。在证书服务器上（对我们来说恰好是同一台主机，但这不是典型的情况），使用以下命令对`CSR`文件进行签名：

```
$ sudo openssl ca -in server.csr -config /etc/ssl/openssl.cnf
```

这将生成几页输出（未显示）并要求确认几次。其中一个确认将是我们在之前创建 CSR 时提供的密码。当一切都说完了，你会看到实际的证书在输出的最后一部分滚动显示。你还会注意到，由于我们没有指定任何日期，证书从现在开始有效，并且设置在一年后过期。

我们刚刚签署的证书存储在`/etc/ssl/newcerts/01.pem`中，如下面的代码片段所示，并且应该准备好供请求服务使用：

```
$ ls /etc/ssl/newcerts/
01.pem
```

随着我们的进展，颁发的证书将递增到`02.pem`，`03.pem`等等。

请注意在下面的代码片段中，`index`文件已经更新了证书的详细信息，`序列号`文件已经递增，准备好下一个签名请求：

```
$ cat /etc/ssl/CA/index.txt
V       220415165738Z           01      unknown /C=CA/ST=ON/O=Coherent Security/OU=IT/CN=www.coherentsecurity.com
$ cat /etc/ssl/CA/serial
02
```

完成了一个 CA 示例并且使用了一个测试证书，让我们看看如何保护你的 CA 基础设施。

# 保护你的证书颁发机构基础设施

通常建议采取一些最佳实践来保护你的 CA。一些“传统”的建议是针对个别 CA 的，但随着虚拟化在大多数数据中心变得普遍，这带来了额外的机会来简化和保护 CA 基础设施。

## 传统的经过验证的建议

传统的建议是为了保护组织的证书基础设施，利用它只在颁发证书时使用的事实。如果你能很好地掌握新证书需求的时间，那么在不需要时可以关闭 CA 服务器。

如果你需要更灵活性，可以创建一个分级证书基础设施。为你的组织创建一个根 CA，它的唯一工作是签署用于创建下级 CA（或可能是多个下级 CA）的证书。然后使用这些下级 CA 来创建所有客户端和服务器证书。根 CA 可以在不需要时关闭或以其他方式下线，除了打补丁。

如果一个组织特别关心保护他们的 CA，可以使用专用硬件，比如硬件安全模块（HSM）来存储他们的 CA 的私钥和证书，通常是在保险箱或其他离线、安全的地方。HSM 的商业示例包括 Nitrokey HSM 或 YubiHSM。NetHSM 是开源 HSM 的一个很好的例子。

## 现代建议

前面的建议仍然完全有效。我们在现代基础设施中看到有助于保护我们的 CA 的新要素是服务器虚拟化。在大多数环境中，这意味着每台服务器都有一个或多个镜像备份存储在本地磁盘上，因为 VM 是如何备份的。因此，如果主机受到无法修复的损坏，无论是来自恶意软件（通常是勒索软件）还是一些严重的配置错误，只需要大约 5 分钟的时间就可以将整个服务器回滚到前一天的镜像，或者在最坏的情况下，回滚到两天前的镜像。

在这种恢复中丢失的一切将是在那个*丢失*间隔中颁发的任何证书的服务器数据，如果我们再次回顾一下会话是如何协商的，那么服务器数据实际上从未在建立会话时使用。这意味着服务器为恢复所花费的这段*时光旅行*不会影响任何使用颁发的证书进行加密协商的客户端或服务器（或者认证，当我们到达*第九章*，*Linux 的 RADIUS 服务*时我们会看到）。

在较小的环境中，根据情况，你可以只使用一个 CA 服务器轻松地保护你的基础设施——只需保留镜像备份，这样如果需要恢复，那个逐字节的镜像是可用的，并且可以在几分钟内回滚。

在更大的环境中，为您的 CA 基础设施建立一个分层模型仍然是有意义的——例如，这可以使合并和收购变得更加容易。分层模型有助于将基础设施保持为一个单一组织，同时使得更容易将多个业务单元的 CA 连接到一个主服务器下。然后，您可以使用**操作系统**（**OS**）的安全性来限制在某个部门发生恶意软件事件时的*扩散区域*；或者在日常模型中，如果需要，您可以使用相同的操作系统安全性来限制业务单元之间的证书管理访问。

依赖镜像备份来保护您的 CA 基础设施的主要风险在于 CA 服务器的传统用法——在某些环境中，可能只偶尔需要证书。例如，如果您在本地保留了一周的服务器镜像备份，但需要一个月（或几个月）才意识到您应用的脚本或补丁已经使您的 CA 服务器崩溃，那么从备份中恢复可能会变得棘手。这可以通过更广泛地使用证书（例如，在对无线客户端进行身份验证以连接到无线网络时）以及自动证书颁发解决方案（例如 Certbot 和 ACME 协议（由 Let's Encrypt 平台开创））来解决。这些事情，特别是结合起来，意味着 CA 的使用频率越来越高，以至于如果 CA 服务器无法正常运行，情况现在可能会在几小时或几天内升级，而不是几周或几个月内。

## 现代基础设施中的 CA 特定风险

*证书颁发机构*或*CA*不是在派对上随意谈论的术语，甚至在工作的休息室里也不会出现。这意味着，如果您给您的 CA 服务器命名为`ORGNAME-CA01`，虽然名称中的`CA01`部分显然对您很重要，但不要指望主机名中的`CA`对其他人来说也很重要。例如，对于您的经理、程序员、在您度假时替您工作的人，或者因某种原因拥有超级用户密码的暑期学生来说，这很可能不会引起注意。如果您是顾问，可能没有人实际在组织中知道 CA 的作用。

这意味着，特别是在虚拟化基础设施中，我们经常会看到 CA 虚拟机被（某种程度上）意外删除。这种情况发生的频率足够高，以至于当我构建一个新的 CA 虚拟机时，我通常会将其命名为`ORGNAME-CA01 – 不要删除，联系 RV`，其中`RV`代表拥有该服务器的管理员的缩写（在这种情况下，是我）。

当任何服务器虚拟机被删除时，设立警报可能是个明智的选择，通知管理团队的任何人——这将为您提供另一层防御，或者至少及时通知，以便您可以快速恢复。

最后，在您的虚拟化基础设施上实施**基于角色的访问控制**（**RBAC**）是每个人的最佳实践清单上的事项。任何特定服务器的直接管理员应该能够删除、重新配置或更改该服务器的电源状态。这种控制级别在现代虚拟化器中很容易配置（例如，VMware 的 vSphere）。这至少使意外删除虚拟机变得更加困难。

现在我们已经制定了一些安全实践来保护我们的 CA，让我们从攻击者和基础设施防御者的角度来看看 CT。

# 证书透明性

回顾本章的开头段落，回想一下 CA 的主要*工作*之一是*信任*。无论是公共 CA 还是私人 CA，您都必须信任 CA 来验证请求证书的人是否是他们所说的那个人。如果这个检查失败，那么任何想要代表[yourbank.com](http://yourbank.com)的人都可以请求该证书，并假装是你的银行！在当今以网络为中心的经济中，这将是灾难性的。

当这种信任失败时，各种 CA、浏览器团队（尤其是 Mozilla、Chrome 和 Microsoft）以及操作系统供应商（主要是 Linux 和 Microsoft）将简单地从各种操作系统和浏览器证书存储中删除违规的 CA。这基本上将由该 CA 签发的所有证书移至*不受信任*类别，迫使所有这些服务从其他地方获取证书。这在最近的过去发生过几次。

DigiNotar 在遭到破坏后被删除，攻击者控制了其一些关键基础设施。一个欺诈的`*.`[google.com](http://google.com)——请注意，`*`是使这个证书成为通配符，可以用来保护或冒充该域中的任何主机。不仅是那个欺诈的通配符被签发了，它还被用来拦截真实的流量。不用说，每个人对此都持负面看法。

在 2009 年至 2015 年期间，赛门铁克 CA 签发了许多**测试证书**，包括属于谷歌和 Opera（另一个浏览器）的域。当这一事件曝光后，赛门铁克受到了越来越严格的限制。最终，赛门铁克的工作人员反复跳过了验证重要证书的步骤，该 CA 最终在 2018 年被删除。

为了帮助检测这种类型的事件，公共 CA 现在参与**证书透明度**（**CT**），如**请求评论**（**RFC**）*6962*中所述。这意味着当证书被签发时，该 CA 会将有关证书的信息发布到其 CT 服务中。这个过程对于所有用于**安全套接字层**（**SSL**）/TLS 的证书是强制性的。这个程序意味着任何组织都可以检查（或更正式地说，审计）它购买的证书的注册表。更重要的是，它可以检查/审计它*没有*购买的证书的注册表。让我们看看这在实践中是如何运作的。

## 使用 CT 进行库存或侦察

正如我们讨论过的，CT 服务存在的主要原因是通过允许任何人验证或正式审计已签发的证书来确保对公共 CA 的信任。

然而，除此之外，组织可以查询 CT 服务，看看是否有为他们公司购买的合法证书，而这些证书是由不应该从事服务器业务的人购买的。例如，市场团队建立了一个与云服务提供商合作的服务器，绕过了可能已经讨论过的所有安全和成本控制，如果**信息技术**（**IT**）组为他们代建服务器的话。这种情况通常被称为*影子 IT*，即非 IT 部门决定用他们的信用卡去做一些并行的、通常安全性较差的服务器，而*真正的*IT 组通常直到为时已晚才发现。

或者，在安全评估或渗透测试的情境中，找到客户的所有资产是谜题的关键部分——你只能评估你能找到的东西。使用 CT 服务将找到为公司颁发的所有 SSL/TLS 证书，包括测试、开发和质量保证（QA）服务器的任何证书。测试和开发服务器通常是最不安全的，而且这些服务器通常为渗透测试人员提供了一个开放的入口。很多时候，这些开发服务器包含了生产数据库的最新副本，因此在许多情况下，入侵开发环境就等于完全入侵。不用说，真正的攻击者也使用这些方法来找到这些同样脆弱的资产。这也意味着在这种情况下的蓝队（IT 组中的防御者）应该经常检查诸如 CT 服务器之类的东西。

话虽如此，您究竟如何检查 CT 呢？让我们使用[`crt.sh`](https://crt.sh)上的服务器，并搜索颁发给`example.com`的证书。要做到这一点，请浏览[`crt.sh/?q=example.com`](https://crt.sh/?q=example.com)（如果您感兴趣，也可以使用您的公司域名）。

请注意，因为这是一个完整的审计跟踪，这些证书通常会回溯到 2013-2014 年 CT 仍处于实验阶段的时候！这可以成为一个很好的侦察工具，可以帮助您找到已过期证书或现在受到通配符证书保护的主机。旧的`*.example.com`（或`*.yourorganisation.com`）。这些证书旨在保护指定父域下的任何主机（由`*`指示）。使用通配符的风险在于，如果适当的材料被盗，可能来自一个脆弱的服务器，域中的任何或所有主机都可以被冒充——这当然是灾难性的！另一方面，购买了三到五个单独的证书之后，将它们全部合并为一个通配符证书变得具有成本效益，而且更重要的是，只有一个到期日期需要跟踪。一个附带的好处是使用通配符证书意味着使用 CT 进行侦察对攻击者来说变得不那么有效。然而，防御者仍然可以看到欺诈证书，或者其他部门购买并正在使用的证书。

在本章中，我们涵盖了很多内容。现在我们对现代基础设施中证书的位置有了牢固的掌握，让我们探讨如何使用现代应用程序和协议来自动化整个证书过程。

# 证书自动化和 ACME 协议

近年来，CA 的自动化得到了一些严重的推广。特别是 Let's Encrypt 通过提供免费的公共证书服务推动了这一变化。他们通过使用自动化，特别是使用 ACME 协议（RFC 8737/RFC 8555）和 Certbot 服务来验证 CSR 信息，以及颁发和交付证书，降低了这项服务的成本。在很大程度上，这项服务和协议侧重于为 Web 服务器提供自动化证书，但正在扩展到其他用例。

Smallstep 等实现使用 ACME 协议来自动化和颁发证书请求，已将这一概念扩展到包括以下内容：

+   使用身份令牌进行身份验证的开放授权（OAuth）/OpenID Connect（OIDC）配置，允许 G Suite、Okta、Azure Active Directory（Azure AD）和任何其他 OAuth 提供商进行单点登录（SSO）集成

+   使用来自 Amazon Web Services（AWS）、Google Cloud Platform（GCP）或 Azure 的 API 进行 API 配置

+   **JavaScript 对象表示法（JSON）Web 密钥**（**JWK**）和**JSON Web 令牌**（**JWT**）集成，允许一次性令牌用于身份验证或利用后续证书颁发

由于使用 ACME 协议颁发的证书通常是免费的，它们也是恶意行为者的主要目标。例如，恶意软件经常利用 Let's Encrypt 提供的免费证书来加密**命令和控制**（**C2**）操作或数据外泄。即使对于 Smallstep 等内部 ACME 服务器，对细节的疏忽也可能意味着恶意行为者能够破坏组织中的所有加密。因此，基于 ACME 的服务器通常只颁发短期证书，并且自动化将通过完全消除增加的管理开销来“弥补不足”。Let's Encrypt 是使用 ACME 的最知名的公共 CA，其证书有效期为 90 天。Smallstep 则采取极端措施，默认证书有效期为 24 小时。请注意，24 小时的到期时间是极端的，这可能会严重影响可能每天不在内部网络上的移动工作站，因此通常会设置更长的间隔。

在 ACME 之前，**简单证书注册协议**（**SCEP**）用于自动化，特别是用于提供机器证书。SCEP 仍然广泛用于**移动设备管理**（**MDM**）产品，以向移动电话和其他移动设备提供企业证书。SCEP 在 Microsoft 的**网络设备注册服务**（**NDES**）组件中仍然被广泛使用，在其基于**Active Directory**（**AD**）的证书服务中也是如此。

说到微软，他们的免费证书服务会自动注册工作站和用户证书，都受到组策略控制。这意味着随着工作站和用户自动化身份验证要求的增加，微软 CA 服务的使用似乎也在增加。

基于 Linux 的 CA 服务的整体趋势是尽可能自动化证书的颁发。然而，底层的证书原则与本章讨论的完全相同。随着这一趋势中的*赢家*开始出现，您应该掌握工具，以了解在您的环境中任何 CA 应该如何工作，无论使用的是前端还是自动化方法。

随着自动化的完成，我们已经涵盖了您在现代基础设施中看到的主要证书操作和配置。然而，在结束这个话题之前，通常有一个简短的“食谱式”命令集是很有用的，用于证书操作。由于 OpenSSL 是我们的主要工具，我们已经整理了一份常见命令的列表，希望这些命令能够使这些复杂的操作更简单完成。

# OpenSSL 备忘单

要开始本节，让我说一下，这涵盖了本章中使用的命令，以及您可能在检查、请求和颁发证书时使用的许多命令。还演示了一些远程调试命令。OpenSSL 有数百个选项，因此像往常一样，man 页面是您更全面地探索其功能的朋友。在紧要关头，如果您搜索`OpenSSL` `cheat sheet`，您会发现数百页显示常见 OpenSSL 命令的页面。

以下是在证书创建中常见的一些步骤和命令：

+   要为新证书（申请人）创建私钥，请运行以下命令：

```
openssl genrsa -des3 -out private.key <bits>
```

+   要为新证书（申请人）创建 CSR，请运行以下命令：

```
openssl req -new -key private.key -out server.csr
```

+   要验证 CSR 签名，请运行以下命令：

```
openssl req -in example.csr -verify
```

+   要检查 CSR 内容，请运行以下命令：

```
openssl req -in server.csr -noout -text
```

+   要签署 CSR（在 CA 服务器上），请运行以下命令：

```
sudo openssl ca -in server.csr -config <path to configuration file>
```

+   创建自签名证书（通常不是最佳做法），运行以下命令：

```
openssl req -x509 -sha256 -nodes -days <days>  -newkey rsa:2048 -keyout privateKey.key -out certificate.crt
```

以下是在检查证书状态时使用的一些命令：

+   要检查标准的`x.509`证书文件，请运行以下命令：

```
openssl x509 -in certificate.crt -text –noout
```

+   要检查`PKCS#12`文件（这将证书和私钥合并为一个文件，通常带有`pfx`或`p12`后缀），运行以下命令：

```
openssl pkcs12 -info -in certpluskey.pfx
```

+   要检查私钥，请运行以下命令：

```
openssl rsa -check -in example.key
```

以下是远程调试证书中常用的一些命令：

+   要检查远程服务器上的证书，请运行以下命令：

```
openssl s_client -connect <servername_or_ip>:443
```

+   使用 OCSP 协议检查证书吊销状态（请注意，这是一个过程，因此我们已编号了步骤），请按以下步骤进行：

1.  首先，收集公共证书并去除`BEGIN`和`END`行，如下所示：

```
openssl s_client -connect example.com:443 2>&1 < /dev/null | sed -n '/-----BEGIN/,/-----END/p' > publiccert.pem
```

1.  接下来，检查证书中是否有 OCSP**统一资源标识符**（**URI**），如下所示：

```
openssl x509 -noout -ocsp_uri -in publiccert.pem http://ocsp.ca-ocspuri.com
```

1.  如果有，您可以在此时发出请求，如下所示：

```
http://ocsp.ca-ocspuri.com is the URI of the issuing CA's OCSP server (previously found).
```

1.  如果公共证书中没有 URI，我们需要获取证书链（即到发行者的链），然后获取发行者的根 CA，如下所示：

```
openssl s_client -connect example.com443 -showcerts 2>&1 < /dev/null
```

1.  这通常会产生大量输出-要提取证书链到文件（在本例中为`chain.pem`），请运行以下命令：

```
openssl ocsp -issuer chain.pem -cert publiccert.pem -text -url http://ocsp.ca-ocspuri.com
```

以下是一些 OpenSSL 命令，用于在文件格式之间进行转换：

+   要转换`-----BEGIN CERTIFICATE-----`：

```
openssl x509 -outform der -in certificate.pem -out certificate.der
```

+   要将 DER 文件（`.crt`，`.cer`或`.der`）转换为 PEM 文件，请运行以下命令：

```
openssl x509 -inform der -in certificate.cer -out certificate.pem
```

+   要转换包含私钥和证书的`PKCS#12`文件（`.pfx`，`.p12`）为 PEM 文件，请运行以下命令：

```
openssl pkcs12 -in keyStore.pfx -out keyStore.pem –nodes
```

+   OpenSLL 命令也用于将 PEM 证书文件和私钥转换为`PKCS#12`（`.pfx`，`.p12`）。

如果服务需要身份证书，但在安装过程中没有 CSR 提供私钥信息，则通常需要`PKCS#12`格式文件。在这种情况下，使用**个人交换格式**（**PFX**）文件或**公钥密码标准#12**（**P12**）文件提供所需的所有信息（私钥和公共证书）在一个文件中。示例命令如下：

```
openssl pkcs12 -export -out certificate.pfx -inkey privateKey.key -in certificate.crt -certfile CACert.crt
```

希望这本简短的“食谱”有助于揭秘证书操作，并简化阅读涉及您证书基础设施的各种文件。

# 总结

通过本讨论，您应该了解使用 OpenSSL 安装和配置证书服务器的基础知识。您还应该了解请求证书和签署证书所需的基本概念。不同 CA 实现中的基本概念和工具保持不变。您还应该了解用于检查证书材料或在远程服务器上调试证书的基本 OpenSSL 命令。

您还应该进一步了解保护您的证书基础设施所涉及的因素。这包括使用 CT 进行库存和侦察，无论是防御性还是进攻性。

在*第九章*，*Linux 的 RADIUS 服务*，我们将在此基础上添加 RADIUS 认证服务到我们的 Linux 主机。您将看到在更高级的配置中，RADIUS 可以使用您的证书基础设施来保护您的无线网络，证书将用于双向认证和加密。

# 问题

最后，这里是一些问题列表，供您测试对本章材料的了解。您将在*附录*的*评估*部分找到答案：

1.  证书在通信中发挥了哪两个功能？

1.  什么是`PKCS#12`格式，它可能在哪里使用？

1.  CT 为什么重要？

1.  为什么您的 CA 服务器跟踪已发行证书的详细信息很重要？

# 进一步阅读

要了解更多关于主题的信息，请参考以下材料：

+   Ubuntu 上的证书（特别是构建 CA）：[`ubuntu.com/server/docs/security-certificates`](https://ubuntu.com/server/docs/security-certificates)

+   OpenSSL 主页：[`www.openssl.org/`](https://www.openssl.org/)

+   *使用 OpenSSL 进行网络安全*: [`www.amazon.com/Network-Security-OpenSSL-John-Viega/dp/059600270X`](https://www.amazon.com/Network-Security-OpenSSL-John-Viega/dp/059600270X)

+   CT: [`certificate.transparency.dev`](https://certificate.transparency.dev)

+   在 OpenSUSE 上的 CA 操作（使用 YaST）：[`doc.opensuse.org/documentation/leap/archive/42.3/security/html/book.security/cha.security.yast_ca.html`](https://doc.opensuse.org/documentation/leap/archive/42.3/security/html/book.security/cha.security.yast_ca.html)

+   基于 Red Hat 的 CA 操作（使用证书管理器）：[`access.redhat.com/documentation/en-us/red_hat_certificate_system/9/html/planning_installation_and_deployment_guide/planning_how_to_deploy_rhcs`](https://access.redhat.com/documentation/en-us/red_hat_certificate_system/9/html/planning_installation_and_deployment_guide/planning_how_to_deploy_rhcs)

+   Easy-RSA：[`github.com/OpenVPN/easy-rsa`](https://github.com/OpenVPN/easy-rsa)

+   支持 ACME 的 CA：

Smallstep CA: [`smallstep.com/`](https://smallstep.com/)

Boulder CA: [`github.com/letsencrypt/boulder`](https://github.com/letsencrypt/boulder)
