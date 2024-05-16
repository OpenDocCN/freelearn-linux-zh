# 第九章：*第九章*：Linux 的 RADIUS 服务

在本章中，我们将介绍远程身份验证拨入用户服务（RADIUS），这是在网络上验证服务的主要方法之一。我们将在服务器上实现 FreeRADIUS，将其链接到后端轻量级目录访问协议（LDAP）/安全 LDAP（LDAPS）目录，并使用它来验证对网络上各种服务的访问。

特别是，我们将涵盖以下主题：

+   RADIUS 基础知识-什么是 RADIUS，它是如何工作的？

+   使用本地 Linux 认证实现 RADIUS

+   具有 LDAP/LDAPS 后端认证的 RADIUS

+   Unlang-非语言

+   RADIUS 使用案例场景

+   使用 Google Authenticator 进行 RADIUS 的多因素认证（MFA）

# 技术要求

为了跟随本节中的示例，我们将使用我们现有的 Ubuntu 主机或虚拟机（VM）。在本章中，我们将涉及一些无线主题，因此，如果您的主机或虚拟机中没有无线网卡，您将需要一个无线适配器来完成这些示例。

当我们逐步进行各种示例时，我们将编辑多个配置文件。如果没有特别提到，`freeradius`的配置文件都存储在`/etc/freeradius/3.0/`目录中。

对于 Ubuntu 默认未包含的我们正在安装的软件包，请确保您有可用的互联网连接，以便您可以使用`apt`命令进行安装。

# RADIUS 基础知识-什么是 RADIUS，它是如何工作的？

在我们开始之前，让我们回顾一个关键概念-AAA。AAA 是一个常见的行业术语，代表认证、授权和计费-这是控制资源访问的三个关键概念。

认证是证明您身份所需的一切。在许多情况下，这只涉及用户标识符（ID）和密码，但在本章中，我们还将探讨使用 MFA 的更复杂的方法。

授权通常发生在认证之后。一旦您证明了您的身份，各种系统将使用该身份信息来确定您可以访问什么。这可能意味着您可以访问哪些子网、主机和服务，或者可能涉及您可以访问哪些文件或目录。在常规语言中，认证和授权经常可以互换使用，但在讨论 RADIUS 和系统访问时，它们是完全不同的。

计费有点像回到拨号上网的日子。当人们使用拨号调制解调器访问公司系统或互联网时，他们在会话期间占用了宝贵的资源（即接收调制解调器和电路），因此 RADIUS 用于跟踪他们的会话时间和持续时间，以便进行月度发票。在现代，RADIUS 计费仍然用于跟踪会话时间和持续时间，但这些信息现在更多用于故障排除，有时也用于取证目的。

RADIUS 这些天的主要用途是用于认证，通常也配置了计费。授权通常由其他后端系统完成，尽管 RADIUS 可以用于为每个认证会话分配基于网络的访问控制列表（ACL），这是一种授权形式。

在掌握了这些背景知识之后，让我们更详细地讨论 RADIUS。RADIUS 认证协议非常简单，这使得它对许多不同的用例都很有吸引力，因此几乎所有可能需要认证的设备和服务都支持它。让我们通过一个配置以及一个典型的认证交换（在高层次上）来进行讨论。

首先，让我们讨论一个需要认证的设备，在这种情况下称为**网络访问服务器**（**NAS**）。NAS 可以是**虚拟私人网络**（**VPN**）设备，无线控制器或接入点，或交换机 - 实际上，任何用户可能需要认证的设备。NAS 通常由 RADIUS 服务器定义，通常具有关联的“共享密钥”以允许对设备进行身份验证。

接下来，配置设备以使用 RADIUS 进行身份验证。如果这是用于管理访问，通常会保留本地身份验证作为备用方法 - 因此，如果 RADIUS 不可用，本地身份验证仍将起作用。

这就是设备（NAS）的配置。当客户端尝试连接到 NAS 时，NAS 收集登录信息并将其转发到 RADIUS 服务器进行验证（请参阅*图 9.1*，其中显示了在 Wireshark 中捕获的典型 RADIUS 请求数据包）。数据包中需要注意的内容包括以下内容：

+   用于 RADIUS 请求的端口是`1812/udp`。RADIUS 会计的匹配端口是`1813/udp` - 会计跟踪连接时间等，通常用于计费。许多 RADIUS 服务器仍然完全支持一组较旧的端口（`1645`和`1646/udp`）。

+   `Code`字段用于标识数据包类型 - 在本例中，我们将涵盖`Access-Request`（代码`1`），`Accept`（代码`2`）和`Reject`（代码`3`）。RADIUS 代码的完整列表包括以下内容：

![表 9.1 - RADIUS 代码](img/B16336_09_Table_01.jpg)

表 9.1 - RADIUS 代码

+   `Packet ID`字段用于将请求和响应数据包联系在一起。由于 RADIUS 是**用户数据报协议**（**UDP**）协议，协议级别上没有会话的概念 - 这必须在数据包的有效负载中。

+   `Authenticator`字段对于每个数据包是唯一的，应该是随机生成的。

+   数据包的其余部分由数据包中的`AVP`组成。这使得协议具有可扩展性；NAS 和 RADIUS 服务器都可以根据情况添加 AV 对。所有实现通常都支持几个 AV 对，以及几个特定于供应商的 AV 对，通常与 NAS 供应商和特定情况相关联 - 例如，区分对设备的管理访问和对 VPN 或无线**服务集 ID**（**SSID**）的用户访问。我们将在本章后面探讨更多用例时更深入地讨论这一点。

在以下简单示例中，我们的两个属性是`User-Name` AV 对，它是明文的，以及`User-Password` AV 对，它被标记为`Encrypted`，但实际上是 MD5 哈希值（其中`Request Authenticator`值。**请求评论**（**RFC**）（*RFC 2865* - 请参阅*进一步阅读*部分）对此的计算有详细的解释，如果您对此感兴趣：

![图 9.1 - 简单的 RADIUS 请求](img/B16336_09_001.jpg)

图 9.1 - 简单的 RADIUS 请求

响应通常要简单得多，如下所述：

+   通常要么是代码 2 `Accept`（*图 9.2*），要么是代码 3 `Reject`（*图 9.3*）的响应。

+   数据包 ID 与请求中的相同。

+   响应认证器是根据响应数据包代码（在本例中为 2），响应的长度（在本例中为 20 字节），数据包 ID（2），请求认证器和共享密钥计算出来的。回复中的其他 AV 对也将用于计算此值。该字段的关键是 NAS 将使用它来验证响应是否来自它期望的 RADIUS 服务器。这个第一个数据包示例显示了一个`Access-Accept`响应，其中访问请求被授予：

![图 9.2 - 简单的 RADIUS 响应（Access-Accept）](img/B16336_09_002.jpg)

图 9.2 - 简单的 RADIUS 响应（Access-Accept）

这个第二个响应数据包示例显示了一个`Access-Reject`数据包。所有字段都保持不变，只是访问请求被拒绝了。如果没有配置错误，通常会在用户名或密码值不正确时看到这个结果：

![图 9.3-简单的 RADIUS 响应（Access-Reject）](img/B16336_09_003.jpg)

图 9.3-简单的 RADIUS 响应（Access-Reject）

现在我们知道了简单的 RADIUS 请求是如何工作的，让我们开始构建我们的 RADIUS 服务器。

# 使用本地 Linux 身份验证实现 RADIUS

这个示例显示了最简单的 RADIUS 配置，其中`UserID`和`Password`值都在配置文件中定义。由于几个原因，这不建议用于任何生产环境，详细如下：

+   密码以明文字符串的形式存储，因此在发生妥协时，所有 RADIUS 密码都可以被恶意行为者收集。

+   密码是由管理员输入而不是用户。这意味着“不可否认”的关键安全概念丢失了-如果事件与这样的帐户相关联，受影响的用户总是可以说“管理员也知道我的密码-一定是他们”。

+   与管理员输入密码相关的是-用户无法更改他们的密码，这也意味着在大多数情况下，这个 RADIUS 密码将与用户使用的其他密码不同，这使得记住它更困难。

然而，这是一个方便的方法，在我们用后端身份验证存储和更复杂的 RADIUS 交换之前测试初始 RADIUS 配置。

首先，我们将安装`freeradius`，如下所示：

```
sudo apt-get install freeradius
```

接下来，让我们编辑`client`配置，定义我们的各种 NAS 设备，人们将向其发出身份验证请求。为此，使用`sudo`编辑`/etc/freeradius/3.0/clients.conf`文件。正如您所期望的那样，RADIUS 配置文件不能使用普通权限进行编辑或查看，因此对这些文件的所有访问都必须使用`sudo`。

在这个文件的底部，我们将为每个 RADIUS 客户端设备添加一个段，其中包含其名称、IP 地址和该设备的共享密钥。请注意，最好使用一个长的、随机的字符串，对于每个设备都是唯一的。您可以很容易地编写一个快速脚本来为您生成这个-有关更多详细信息，请参见[`isc.sans.edu/forums/diary/How+do+you+spell+PSK/16643`](https://isc.sans.edu/forums/diary/How+do+you+spell+PSK/16643)。

在以下代码示例中，我们添加了三个交换机（每个交换机的名称都以`sw`开头）和一个无线控制器（`VWLC01`，一个虚拟无线控制器）。这里的一个关键概念是一致地命名设备。您可能需要为不同的设备类型制定不同的规则或策略；按设备类型给它们一致的名称是一个方便的概念，可以简化这一点。此外，如果设备名称标准是已知和一致的，那么简单的排序列表也会变得更简单：

```
client sw-core01 {
   ipaddr=192.168.122.9
   nastype = cisco
   secret = 7HdRRTP8qE9T3Mte
}
client sw-office01 {
   ipaddr=192.168.122.5
   nastype = cisco
   secret = SzMjFGX956VF85Mf
}
client sw-floor0 {
   ipaddr = 192.168.122.6
   nastype = cisco
   secret = Rb3x5QW9W6ge6nsR
}
client vwlc01 {
   ipaddr = 192.168.122.8
   nastype = cisco
   secret = uKFJjaBbk2uBytmD
}
```

请注意，在某些情况下，您可能需要配置整个子网-在这种情况下，客户端行可能会读取类似于这样的内容：

```
Client 192.168.0.0/16 {
```

这通常不建议，因为它会使 RADIUS 服务器对该子网上的任何内容都开放。如果可能的话，请使用固定的 IP 地址。然而，在某些情况下，您可能被迫使用子网-例如，如果您有**无线接入点**（**WAPs**）直接对 RADIUS 进行无线客户端认证，使用**动态主机配置协议**（**DHCP**）动态分配 IP。

还要注意`nastype`行-这将设备与包含该供应商的常见 AV 对的定义的`dictionary`文件联系起来。

接下来，让我们创建一个测试用户-使用`sudo`编辑`/etc/freeradius/3.0/users`文件，并添加一个测试帐户，就像这样：

```
testaccount  Cleartext-Password := "Test123"
```

最后，使用以下命令重新启动您的服务：

```
sudo service freeradius restart
```

现在，一些故障排除-要测试配置文件的语法，请使用以下命令：

```
sudo freeradius –CX
```

要测试身份验证操作，请验证您的 RADIUS 服务器信息是否定义为 RADIUS 客户端（默认情况下是这样），然后使用如下所示的`radclient`命令：

```
$ echo "User-Name=testaccount,User-Password=Test123" | radclient localhost:1812 auth testing123
Sent Access-Request Id 31 from 0.0.0.0:34027 to 127.0.0.1:1812 length 44
Received Access-Accept Id 31 from 127.0.0.1:1812 to 127.0.0.1:34027 length 20
```

完成这些测试后，建议删除本地定义的用户——这不是您应该忘记的事情，因为这可能会使攻击者稍后可以使用。现在让我们将我们的配置扩展到更典型的企业配置——我们将添加一个基于 LDAP 的后端目录。

# 具有 LDAP/LDAPS 后端身份验证的 RADIUS

使用诸如**LDAP**之类的后端身份验证存储对许多原因都很有用。由于这通常使用与常规登录相同的身份验证存储，这给我们带来了几个优势，如下所述：

+   LDAP 中的组成员资格可用于控制对关键访问的访问权限（例如管理访问）。

+   RADIUS 访问的密码与标准登录的密码相同，更容易记住。

+   密码和密码更改由用户控制。

+   在用户更改组时，凭证维护位于一个中央位置。特别是，如果用户离开组织，他们的帐户在 LDAP 中被禁用后，RADIUS 也会立即被禁用。

这种方法的缺点很简单：用户很难选择好的密码。这就是为什么，特别是对于面向公共互联网的任何接口，建议使用 MFA（我们稍后将在本章中介绍）。

利用这一点，如果访问仅由简单的用户/密码交换控制，攻击者有几种很好的选择来获取访问权限，如下所述：

+   **使用凭证填充**：使用此方法，攻击者从其他泄漏中收集密码（这些是免费提供的），以及您可能期望在本地或公司内部看到的密码（例如本地体育队或公司产品名称），或者可能对目标帐户有重要意义的单词（例如孩子或配偶的姓名，汽车型号，街道名称或电话号码信息）。然后他们尝试所有这些针对他们的目标，通常是从企业网站或社交媒体网站（LinkedIn 是其中的一个最爱）收集的。这非常成功，因为人们往往有可预测的密码，或者在多个网站上使用相同的密码，或两者兼而有之。在任何规模的组织中，攻击者通常在这种攻击中取得成功，通常需要几分钟到一天的时间。这是如此成功，以至于它在几种恶意软件中被自动化，最著名的是从 2017 年开始的*Mirai*（它攻击了常见的**物联网**（**IoT**）设备的管理访问），然后扩展到包括使用常见单词列表进行猜测密码的任意数量的衍生品系。

+   凭证的强制破解：与凭证填充相同，但是使用整个密码列表对所有帐户进行攻击，以及在用尽这些单词后尝试所有字符组合。实际上，这与凭证填充相同，但在初始攻击之后“继续进行”。这显示了攻击者和防御者之间的不平衡——继续攻击对于攻击者来说基本上是免费的（或者与计算时间和带宽一样便宜），那么他们为什么不继续尝试呢？

为 LDAP 身份验证存储配置 RADIUS 很容易。虽然我们将介绍标准的 LDAP 配置，但重要的是要记住这个协议是明文的，因此是攻击者的一个很好的目标——**LDAPS**（**LDAP over Transport Layer Security (TLS)**）始终是首选。通常，标准的 LDAP 配置应该仅用于测试，然后再使用 LDAPS 添加加密方面。

首先，让我们使用 LDAP 作为传输协议在 RADIUS 中配置我们的后端目录。在此示例中，我们的 LDAP 目录是微软的**Active Directory**（**AD**），但在仅 Linux 环境中，通常会有一个 Linux LDAP 目录（例如使用 OpenLDAP）。

首先，安装`freeradius-ldap`软件包，如下所示：

```
$ sudo apt-get install freeradius-ldap
```

在我们继续实施 LDAPS 之前，您需要收集 LDAPS 服务器使用的 CA 服务器的公共证书。将此文件收集到`/usr/share/ca-certificates/extra`目录中（您需要创建此目录），如下所示：

```
$ sudo mkdir /usr/share/ca-certificates/extra
```

将证书复制或移动到新目录中，如下所示：

```
$ sudo cp publiccert.crt /usr/share/ca-certifiates/extra
```

告诉 Ubuntu 将此目录添加到`certs listgroups`，如下所示：

```
$ sudo dpkg-reconfigure ca-certificates
```

您将被提示添加任何新证书，因此请务必选择刚刚添加的证书。如果列表中有任何您不希望看到的证书，请取消此操作并在继续之前验证这些证书是否不恶意。

接下来，我们将编辑`/etc/freeradius/3.0/mods-enabled/ldap`文件。这个文件不会在这里-如果需要，您可以参考`/etc/freeradius/3.0/mods-available/ldap`文件作为示例，或直接链接到该文件。

下面显示的配置中的`server`行意味着您的 RADIUS 服务器必须能够使用**域名系统**（**DNS**）解析该服务器名称。

我们将使用以下行配置 LDAPS：

```
ldap {
        server = 'dc01.coherentsecurity.com'
        port = 636
        # Login credentials for a special user for FreeRADIUS which has the required permissions
        identity = ldapuser@coherentsecurity.com
        password = <password>
        base_dn = 'DC=coherentsecurity,DC=com'
        user {
        # Comment out the default filter which uses uid and replace that with samaccountname
                #filter = "(uid=%{%{Stripped-User-Name}:-%{User-Name}})"
                filter = "(samaccountname=%{%{Stripped-User-Name}:-%{User-Name}})"
        }
        tls {
                ca_file = /usr/share/ca-certificates/extra/publiccert.crt
        }
}
```

如果您被迫配置 LDAP 而不是 LDAPS，则端口更改为`389`，当然也没有证书，因此可以删除或注释掉`ldap`配置文件中的`tls`部分。

我们通常使用的`ldapuser`示例用户不需要任何特殊访问权限。但是，请确保为此帐户使用一个长度（> 16 个字符）的随机密码，因为在大多数环境中，这个密码不太可能经常更改。

接下来，我们将指导`/etc/freeradius/3.0/sites-enabled/default`文件的`authenticate / pap`部分（请注意，这是指向`/etc/freeradius/3.0/sites-available`中主文件的链接），如下所示：

```
        pap
        if (noop && User-Password) {
                update control {
                        Auth-Type := LDAP
                }
        }
```

此外，请确保取消注释该部分中的`ldap`行，如下所示：

```
       ldap
```

现在我们可以在前台运行`freeradius`。这将允许我们在发生时查看消息处理-特别是显示的任何错误。这意味着在进行这一系列初始测试期间，我们不必寻找错误日志。以下是您需要的代码：

```
$ sudo freeradius -cx
```

如果您需要进一步调试，可以使用以下代码将`freeradius`服务器作为前台应用程序运行，以实时显示默认日志记录：

```
$ sudo freeradius –X
```

最后，当一切正常工作时，通过运行以下命令重新启动您的 RADIUS 服务器以收集配置更改：

```
$ sudo service freeradius restart
```

再次，要从本地计算机测试用户登录，请执行以下代码：

```
$ echo "User-Name=test,User-Password=P@ssw0rd!" | radclient localhost:1812 auth testing123
```

最后，我们将希望启用 LDAP 启用的组支持-我们将在后面的部分（* RADIUS 使用案例场景*）中看到，我们将希望在各种策略中使用组成员资格。为此，我们将重新访问`ldap`文件并添加一个`group`部分，如下所示：

```
        group {
            base_dn = "${..base_dn}"
            filter = '(objectClass=Group)'
            name_attribute = cn
            membership_filter = "(|(member=%{control:${..user_dn}})(memberUid=%{%{Stripped-User-Name}:-%{User-Name}}))"
             membership_attribute = 'memberOf'
             cacheable_name = 'no'
             cacheable_dn = 'no'
        }
```

完成这些操作后，我们应该意识到的一件事是，LDAP 并不是用于身份验证，而是用于授权-这是一个检查组成员资格的好方法，例如。实际上，如果您在构建过程中注意到，这在配置文件中是明确指出的。

让我们解决这种情况，并使用**NT LAN Manager**（**NTLM**）作为身份验证的底层 AD 协议。

## NTLM 身份验证（AD）-引入 CHAP

将 RADIUS 与 AD 链接以获取帐户信息和组成员资格，这是我们在大多数组织中看到的最常见的配置。虽然微软**网络策略服务器**（**NPS**）是免费的，并且可以轻松安装在域成员 Windows 服务器上，但它没有一个简单的配置来将其链接到**双因素身份验证**（**2FA**）服务，比如 Google Authenticator。这使得基于 Linux 的 RADIUS 服务器与 AD 集成成为组织需要 MFA 并在建立访问权限时利用 AD 组成员资格的吸引人选择。

这种方法的身份验证是什么样的？让我们来看看标准的**挑战-握手认证协议**（**CHAP**），**Microsoft CHAP**（**MS-CHAP**）或 MS-CHAPv2，它为 RADIUS 交换添加了更改密码的功能。基本的 CHAP 交换如下：

![图 9.4 - 基本 CHAP 交换](img/B16336_09_004.jpg)

图 9.4 - 基本 CHAP 交换

按顺序进行前述交换，我们可以注意到以下内容：

+   首先，客户端发送初始的**Hello**，其中包括**USERID**（但不包括密码）。

+   **CHAP Challenge**从 NAS 发送。这是随机数和 RADIUS 秘钥的结果，然后使用 MD5 进行哈希处理。

+   客户端（**Supplicant**）使用该值对密码进行哈希处理，然后将该值发送到响应中。

+   NAS 将该随机数和响应值发送到 RADIUS 服务器，RADIUS 服务器执行自己的计算。

+   如果两个值匹配，则会话会收到**RADIUS Access-Accept**响应；如果不匹配，则会收到**RADIUS Access-Reject**响应。

**受保护的可扩展认证协议**（**PEAP**）为此交换增加了一个额外的复杂性 - 客户端和 RADIUS 服务器之间存在 TLS 交换，这允许客户端验证服务器的身份，并使用标准 TLS 加密数据交换。为此，RADIUS 服务器需要一个证书，并且客户端需要在其受信任的 CA 存储中拥有发行 CA。

要为 FreeRADIUS 配置 AD 集成（使用 PEAP MS-CHAPv2），我们将为身份验证配置`ntlm_auth`，并将 LDAP 原样移动到配置的`authorize`部分。

要开始使用`ntlm_auth`，我们需要安装`samba`（这是**SMB**的玩笑，代表**服务器消息块**）。首先，确保它尚未安装，如下所示：

```
$ sudo apt list --installed | grep samba
WARNING: apt does not have a stable CLI interface. Use with caution in scripts.
samba-libs/focal-security,now 2:4.11.6+dfsg-0ubuntu1.6 amd64 [installed,upgradable to: 2:4.11.6+dfsg-0ubuntu1.8]
```

从此列表中，我们看到它没有安装在我们的 VM 中，所以让我们使用以下命令将其添加到我们的配置中：

```
 sudo apt-get install samba
```

还要安装以下内容：

```
winbind with sudo apt-get install winbind.
```

编辑`/etc/samba/smb.conf`，并根据您的域更新以下代码段中显示的行（我们的测试域已显示）。在编辑时确保使用`sudo` - 您需要 root 权限来修改此文件（请注意，默认情况下`[homes]`行可能已被注释掉）：

```
[global]
   workgroup = COHERENTSEC
    security = ADS
    realm = COHERENTSECURITY.COM
    winbind refresh tickets = Yes
    winbind use default domain = yes
    vfs objects = acl_xattr
    map acl inherit = Yes
    store dos attributes = Yes 
    dedicated keytab file = /etc/krb5.keytab
    kerberos method = secrets and keytab
[homes]
    comment = Home Directories
    browseable = no
    writeable=yes
```

接下来，我们将编辑`krb5.conf`文件。示例文件位于`/usr/share/samba/setup`中 - 将该文件复制到`/etc`并编辑该副本。请注意，默认情况下`EXAMPLE.COM`条目是存在的，在大多数安装中，这些条目应该被删除（`example.com`是用于示例和文档的保留域）。代码如下所示：

```
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log
[libdefaults]
 default_realm = COHERENTSECURITY.COM
 dns_lookup_realm = false
 dns_lookup_kdc = false
[realms]
 COHERENTSECURITY.COM = {
  kdc = dc01.coherentsecurity.com:88
  admin_server = dc01.coherentsecurity.com:749
  kpaswordserver = dc01.coherentsecurity.com
  default_domain = COHERENTSECURITY.COM
 }
[domain_realm]
 .coherentsecurity.com = coherentsecurity.com
[kdc]
  profile = /var/kerberos/krb5kdc/kdc.conf
[appdefaults]
 pam = {
  debug = false
  ticket_lifetime = 36000
  renew_lifetime = 36000
  forwardable = true
  krb4_convert = false
 }
```

编辑`/etc/nsswitch.conf`文件，并添加`winbind`关键字，如下代码段所示。请注意，在 Ubuntu 20 中，默认情况下可能没有`automount`行，因此您可能希望添加它：

```
passwd:         files systemd winbind
group:          files systemd winbind
shadow:         files winbind
protocols:      db files winbind
services:       db files winbind
netgroup:       nis winbind
automount:      files winbind
```

现在应该为您部分配置了 - 重新启动 Linux 主机，然后验证以下两个服务是否正在运行：

+   `smbd`提供文件和打印共享服务。

+   `nmbd`提供 NetBIOS 到 IP 地址名称服务。

此时，您可以将 Linux 主机加入 AD 域（将提示您输入密码），如下所示：

```
# net ads join –U Administrator
```

重新启动`smbd`和`windbind`守护程序，如下所示：

```
# systemctl restart smbd windbind
```

您可以使用以下代码检查状态：

```
$ sudo ps –e | grep smbd
$ sudo ps –e | grep nmbd
```

或者，要获取更多详细信息，您可以运行以下代码：

```
$ sudo service smbd status
$ sudo service nmbd status
```

现在，您应该能够列出 Windows 域中的用户和组，如下面的代码片段所示：

```
$ wbinfo -u
COHERENTSEC\administrator
COHERENTSEC\guest
COHERENTSEC\ldapuser
COHERENTSEC\test
….
$ wbinfo -g
COHERENTSEC\domain computers
COHERENTSEC\domain controllers
COHERENTSEC\schema admins
COHERENTSEC\enterprise admins
COHERENTSEC\cert publishers
COHERENTSEC\domain admins
…
```

如果这行不通，那么寻找答案的第一个地方很可能是 DNS。请记住这句古老的谚语，这里以俳句的形式表达：

*不是 DNS*

*绝对不是 DNS*

*是 DNS 的问题*

这太有趣了，因为这是真的。如果 DNS 配置不完美，那么各种其他事情都不会按预期工作。为了使所有这些工作正常，您的 Linux 站点将需要在 Windows DNS 服务器上解析记录。使这成为现实的最简单方法是将您站点的 DNS 服务器设置指向该 IP（如果您需要刷新`nmcli`命令，请参考*第二章*，*基本 Linux 网络配置和操作 – 使用本地接口*）。或者，您可以在 Linux DNS 服务器上设置有条件的转发器，或者在 Linux 主机上添加 AD DNS 的辅助区域—根据您需要在您的情况下“主要”的服务，有几种可用的替代方案。

要测试 DNS 解析，请尝试按名称 ping 您的域控制器。如果成功，请尝试查找一些**service**（**SRV**）记录（这些记录是 AD 的基础组成部分）—例如，您可以查看这个：

```
dig +short _ldap._tcp.coherentsecurity.com SRV
0 100 389 dc01.coherentsecurity.com.
```

接下来，验证您是否可以使用`wbinfo`进行 AD 身份验证，然后再次使用 RADIUS 使用的`ntlm_auth`命令，如下所示：

```
wbinfo -a administrator%Passw0rd!
plaintext password authentication failed
# ntlm_auth –-request-nt-key –-domain=coherentsecurity.com --username=Administrator
Password:
NT_STATUS_OK: The operation completed successfully. (0x0)
```

请注意，纯文本密码在`wbinfo`登录尝试中失败了，这（当然）是期望的情况。

通过与域的连接正常工作，我们现在可以开始配置 RADIUS 了。

我们的第一步是更新`/etc/freeradius/3.0/mods-available/mschap`文件，以配置一个设置来修复挑战/响应握手中的问题。您的`mschap`文件需要包含以下代码：

```
chap {
    with_ntdomain_hack = yes
}
```

此外，如果您在文件中向下滚动，您会看到以`ntlm_auth =“`开头的一行。您希望该行读起来像这样：

```
ntlm_auth = "/usr/bin/ntlm_auth --request-nt-key --username=%{%{Stripped-User-Name}:-%{%{User-Name}:-None}} --challenge=%{%{mschap:Challenge}:-00} --nt-response=%{%{mschap:NT-Response}:-00} --domain=%{mschap:NT-Domain}"
```

如果您正在进行机器身份验证，您可能需要将`username`参数更改为以下内容：

```
--username=%{%{mschap:User-Name}:-00}
```

最后，要启用 PEAP，我们转到`mods-available/eap`文件并更新`default_eap_type`行，并将该方法从`md5`更改为`peap`。然后，在`tls-config tls-common`部分，将`random_file`行从`${certdir}/random`的默认值更新为现在显示为`random_file = /dev/urandom`。

完成后，您希望对`eap`文件的更改如下所示：

```
eap {
        default_eap_type = peap
}
tls-config tls-common {
        random_file = /dev/urandom
}
```

这完成了 PEAP 身份验证的典型服务器端配置。

在客户端（请求者）端，我们只需启用 CHAP 或 PEAP 身份验证。在这种配置中，站点发送用户 ID 或机器名称作为认证帐户，以及用户或工作站密码的哈希版本。在服务器端，此哈希与其自己的计算进行比较。密码永远不会以明文形式传输；然而，服务器发送的“挑战”作为额外步骤发送。

在 NAS 设备上（例如，VPN 网关或无线系统），我们启用`MS-CHAP`身份验证，或者`MS-CHAPv2`（它增加了通过 RADIUS 进行密码更改的功能）。

现在，我们将看到事情变得更加复杂；如果您想要使用 RADIUS 来控制多个事物，例如同时控制 VPN 访问和对该 VPN 服务器的管理员访问，使用相同的 RADIUS 服务器？让我们探讨如何使用*U**nlang*语言设置规则来实现这一点。

# Unlang – 无语言

FreeRADIUS 支持一种称为**Unlang**（**无语言**的缩写）的简单处理语言。这使我们能够制定规则，为 RADIUS 身份验证流程和最终决策添加额外的控制。

Unlang 语法通常可以在虚拟服务器文件中找到，例如在我们的情况下，可能是`/etc/freeradius/3.0/sites-enabled/default`，并且可以在标题为`authorize`、`authenticate`、`post-auth`、`preacct`、`accounting`、`pre-proxy`、`post-proxy`和`session`的部分中找到。

在大多数常见的部署中，我们可能会寻找一个传入的 RADIUS 变量或 AV 对，例如`Service-Type`，可能是`Administrative`或`Authenticate-Only`，在 Unlang 代码中，将其与组成员资格进行匹配，例如网络管理员、VPN 用户或无线用户。

对于两个防火墙登录要求的简单情况（`仅 VPN`或`管理`访问），您可能会有这样的规则：

```
if(&NAS-IP-Address == "192.168.122.20") {
    if(Service-Type == Administrative && LDAP-Group == "Network Admins") {
            update reply {
                Cisco-AVPair = "shell:priv-lvl=15"
            } 
            accept
    } 
    elsif (Service-Type == "Authenticate-Only" && LDAP-Group == "VPN Users" ) {
        accept
    }
    elsif {
        reject
    }
}
```

您可以进一步添加到这个示例中，如果用户正在 VPN 连接，`Called-Station-ID`将是防火墙的外部 IP 地址，而管理登录请求将是内部 IP 或管理 IP（取决于您的配置）。

如果有大量设备在运行，`switch/case`结构可以很方便地简化永无止境的`if/else-if`语句列表。您还可以使用`all switches`与（例如）`NAS-Identifier =~ /SW*/`。

如果要进行无线访问的身份验证，`NAS-Port-Type`设置将是`Wireless-802.11`，对于 802.1x 有线访问请求，`NAS-Port-Type`设置将是`Ethernet`。

您还可以根据不同的无线 SSID 包含不同的身份验证标准，因为 SSID 通常在`Called-Station-SSID`变量中，格式为`<AP 的 MAC 地址>:SSID 名称`，用`-`字符分隔`58-97-bd-bc-3e-c0:WLCORP`。因此，要返回 MAC 地址，您将匹配最后六个字符，例如`.\.WLCORP$`。

在典型的企业环境中，我们可能会有两到三个不同访问级别的 SSID，对不同网络设备类型的管理用户，具有 VPN 访问权限或访问特定 SSID 的用户。您可以看到这种编码练习如何迅速变得非常复杂。建议首先在小型测试环境中测试您的`unlang`语法的更改（也许使用虚拟网络设备），然后在计划的停机/测试维护窗口期间进行部署和生产测试。

现在我们已经构建好了所有的部分，让我们为各种身份验证需求配置一些真实的设备。

# RADIUS 使用案例场景

在本节中，我们将看看几种设备类型以及这些设备可能具有的各种身份验证选项和要求，并探讨如何使用 RADIUS 来解决它们。让我们从 VPN 网关开始，使用标准的用户 ID 和密码身份验证（不用担心，我们不会就这样留下它）。

## 使用用户 ID 和密码进行 VPN 身份验证

VPN 服务（或者在此之前，拨号服务）的身份验证是大多数组织首先使用 RADIUS 的原因。然而，随着时间的推移，单因素用户 ID 和密码登录对于任何面向公众的服务来说已经不再是安全选项。我们将在本节讨论这一点，但当我们在 MFA 部分时，我们将更新为更现代的方法。

首先，将您的 VPN 网关（通常是防火墙）添加为 RADIUS 的客户端-将其添加到您的`/etc/freeradius/3.0/clients.conf`文件中，如下所示：

```
client hqfw01 {
  ipaddr = 192.168.122.1
  vendor = cisco
  secret = pzg64yr43njm5eu
}
```

接下来，配置您的防火墙指向 RADIUS 进行 VPN 用户身份验证。例如，对于 Cisco 自适应安全设备（ASA）防火墙，您可以进行以下更改：

```
! create a AAA Group called "RADIUS" that uses the protocol RADIUS
aaa-server RADIUS protocol radius
! next, create servers that are members of this group
aaa-server RADIUS (inside) host <RADIUS Server IP 01>
 key <some key 01>
 radius-common-pw <some key 01>
 no mschapv2-capable
 acl-netmask-convert auto-detect
aaa-server RADIUS (inside) host <RADIUS Server IP 02>
 key <some key 02>
 radius-common-pw <some key 02>
 no mschapv2-capable
 acl-netmask-convert auto-detect
```

接下来，更新隧道组以使用`RADIUS`服务器组进行身份验证，如下所示：

```
tunnel-group VPNTUNNELNAME general-attributes
 authentication-server-group RADIUS
 default-group-policy VPNPOLICY
```

现在这个已经可以工作了，让我们将`RADIUS`作为对这个相同设备的管理访问的身份验证方法。

## 对网络设备的管理访问

接下来，我们将要添加的是对同一防火墙的管理访问。我们如何为管理员做到这一点，但又防止常规 VPN 用户访问管理功能？很简单 - 我们将利用一些额外的 AV 对（记得我们在本章前面讨论过吗？）。

我们将首先添加一个新的网络策略，具有以下凭据：

+   对于 VPN 用户，我们将添加一个`服务类型`的 AV 对，值为`仅认证`。

+   对于管理用户，我们将添加一个`服务类型`的 AV 对，值为`管理`。

在 RADIUS 端，策略将要求每个策略的组成员资格，因此我们将在后端身份验证存储中创建名为`VPN 用户`和`网络管理员`的组，并适当填充它们。请注意，当所有这些放在一起时，管理员将具有 VPN 访问权限和管理访问权限，但具有常规 VPN 帐户的人只能具有 VPN 访问权限。

要获取此规则的实际语法，我们将回到 Unlang 的上一节，并使用那个例子，它恰好满足我们的需求。如果您正在请求管理访问权限，您需要在`网络管理员`组中，如果您需要 VPN 访问权限，您需要在`VPN 用户`组中。如果访问和组成员资格不符，则将拒绝访问。

现在 RADIUS 已经设置好了，让我们将对**图形用户界面**（**GUI**）和**安全外壳**（**SSH**）接口的管理访问指向 RADIUS 进行身份验证。在防火墙上，将以下更改添加到我们在 VPN 示例中讨论过的 ASA 防火墙配置中：

```
aaa authentication enable console RADIUS LOCAL
aaa authentication http console RADIUS LOCAL
aaa authentication ssh console RADIUS LOCAL
aaa accounting enable console RADIUS
aaa accounting ssh console RADIUS
aaa authentication login-history
```

请注意，每种登录方法都有一个“身份验证列表”。我们首先使用 RADIUS，但如果失败（例如，如果 RADIUS 服务器宕机或无法访问），则对本地帐户的身份验证将失败。还要注意，我们在`enable`模式的列表中有 RADIUS。这意味着我们不再需要一个所有管理员必须使用的单一共享启用密码。最后，`aaa authentication log-history`命令意味着当您进入`enable`模式时，防火墙将将您的用户名注入 RADIUS 请求，因此您只需要在输入`enable`模式时输入密码。

如果我们没有设置`unlang`规则，那么仅仅前面的配置将允许常规访问 VPN 用户请求和获取管理访问权限。一旦 RADIUS 控制了一个设备上的多个访问权限，就必须编写规则来保持它们的清晰。

配置好我们的防火墙后，让我们来看看对路由器和交换机的管理访问。

### 对路由器和交换机的管理访问

我们将从思科路由器或交换机配置开始。这个配置在不同平台或** Internetwork Operating System **（** IOS **）版本之间会有轻微差异，但应该看起来非常类似于这样：

```
radius server RADIUS01
    address ipv4 <radius server ip 01> auth-port 1812 acct-port 1813
    key <some key>
radius server RADIUS02
    address ipv4 <radius server ip 02> auth-port 1812 acct-port 1813
    key <some key>
aaa group server radius RADIUSGROUP
    server name RADIUS01
    server name RADIUS02
ip radius source-interface <Layer 3 interface name>
aaa new-model
aaa authentication login RADIUSGROUP group radius local
aaa authorization exec RADIUSGROUP group radius local
aaa authorization network RADIUSGROUP group radius local
line vty 0 97
 ! restricts access to a set of trusted workstations or subnets
 access-class ACL-MGT in
 login authentication RADIUSG1
 transport input ssh
```

**惠普**（**HP**）ProCurve 等效配置将如下所示：

```
radius-server host <server ip> key <some key 01>
aaa server-group radius "RADIUSG1" host <server ip 01>
! optional RADIUS and AAA parameters
radius-server dead-time 5
radius-server timeout 3
radius-server retransmit 2
aaa authentication num-attempts 3
aaa authentication ssh login radius server-group "RADIUSG1" local
aaa authentication ssh enable radius server-group "RADIUSG1" local
```

请注意，当进入`enable`模式时，HP 交换机将需要第二次进行完整身份验证（用户 ID 和密码），而不仅仅是密码，这可能出乎您的意料。

在 RADIUS 服务器上，来自思科和惠普交换机的管理访问请求将包括我们在防火墙管理访问中看到的相同 AV 对：`服务类型：管理`。您可能会将此与 RADIUS 中的组成员资格要求配对，就像我们为防火墙所做的那样。

既然我们已经让 RADIUS 控制我们的交换机的管理访问权限，让我们将 RADIUS 控制扩展到包括更安全的身份验证方法。让我们从探索 EAP-TLS（其中** EAP **代表**可扩展身份验证协议**）开始，它使用证书进行客户端和 RADIUS 服务器之间的相互身份验证交换。

## EAP-TLS 身份验证的 RADIUS 配置

要开始本节，让我们讨论一下 EAP-TLS 到底是什么。**EAP**是一种扩展 RADIUS 传统用户 ID/密码交换的方法。我们在*第八章*中熟悉了 TLS，*Linux 上的证书服务*。因此，简单地说，EAP-TLS 是在 RADIUS 内使用证书来证明身份和提供认证服务。

在大多数“常规公司”使用情况下，EAP-TLS 与一个名为 802.1x 的第二协议配对使用，该协议用于控制对网络的访问，例如对无线 SSID 或有线以太网端口的访问。我们将花一些时间来了解这一点，但让我们开始看看 EAP-TLS 的具体细节，然后加入网络访问。

那么，从协议的角度来看，这是什么样子的呢？如果您回顾我们在*第八章*中讨论的*使用证书–Web 服务器*示例，它看起来与那个例子完全一样，但是在双向上。绘制出来（在*图 9.5*中），我们看到与 Web 服务器示例中相同的信息交换，但在双向上，如下所述：

+   客户端（或 supplicant）使用其用户或设备证书向 RADIUS 发送其身份信息，而不是使用用户 ID 和密码——RADIUS 服务器使用这些信息来验证 supplicant 的身份，并根据该信息（和 RADIUS 内的相关规则）允许或拒绝访问。

+   同时，supplicant 以相同的方式验证 RADIUS 服务器的身份——验证服务器名称是否与证书中的**通用名称**（**CN**）匹配，并且证书是否受信任。这可以防范恶意部署的 RADIUS 服务器（例如，在“恶意双胞胎”无线攻击中）。

+   一旦完成了这种相互认证，网络连接就在 supplicant 和网络设备（NAS）之间完成了——通常，该设备是交换机或 WAP（或无线控制器）。

您可以在以下图表中看到这一点的说明：

![图 9.5 – 802.1x/EAP-TLS 会话的认证流程](img/B16336_09_005.jpg)

图 9.5 – 802.1x/EAP-TLS 会话的认证流程

以下是一些需要注意的事项：

+   所有这些都要求提前分发所有必需的证书。这意味着 RADIUS 服务器需要安装其证书，而 supplicants 需要安装其设备证书和/或用户证书。

+   作为其中的一部分，CA 必须得到设备、用户和 RADIUS 服务器的信任。虽然所有这些都可以通过公共 CA 完成，但通常由私有 CA 完成。

+   在认证过程中，supplicant 和 RADIUS 服务器（当然）都不与 CA 通信。

既然我们理解了 EAP-TLS 的工作原理，那么在无线控制器上，EAP-TLS 配置是什么样子的呢？

## 使用 802.1x/EAP-TLS 进行无线网络认证

对于许多公司来说，EAP-TLS 用于 802.1x 认证作为其无线客户端认证机制，主要是因为无线的其他任何认证方法都容易受到一种或多种简单攻击的影响。EAP-TLS 实际上是唯一安全的无线认证方法。

也就是说，在 NAS 上（在这种情况下是无线控制器）的配置非常简单——准备和配置的大部分工作都在 RADIUS 服务器和客户端站上。对于思科无线控制器，配置通常主要通过 GUI 完成，当然，也有命令行。

在 GUI 中，EAP-TLS 认证非常简单——我们只是为客户端建立一个直接向 RADIUS 服务器进行认证的通道（反之亦然）。步骤如下：

1.  首先，为身份验证定义一个 RADIUS 服务器。几乎相同的配置也适用于相同服务器的 RADIUS 计费，使用端口`1813`。您可以在以下截图中看到一个示例配置：![图 9.6 – 无线控制器配置的 RADIUS 服务器](img/B16336_09_006.jpg)

图 9.6 – 无线控制器配置的 RADIUS 服务器

1.  接下来，在 SSID 定义下，我们将设置 802.1x 身份验证，如以下截图所示：![图 9.7 – 配置 SSID 使用 802.1x 身份验证](img/B16336_09_007.jpg)

图 9.7 – 配置 SSID 使用 802.1x 身份验证

1.  最后，在 AAA 服务器下，我们将 RADIUS 服务器链接到 SSID，如以下截图所示：

![图 9.8 – 为 802.1x 身份验证和计费分配 RADIUS 服务器](img/B16336_09_008.jpg)

图 9.8 – 为 802.1x 身份验证和计费分配 RADIUS 服务器

为了使所有这些工作正常运行，客户端和 RADIUS 服务器都需要适当的证书，并且需要配置 EAP-TLS 身份验证。建议提前分发证书，特别是如果您正在使用自动化发放证书，您需要给客户端足够的时间，以便它们都连接并触发证书的发放和安装。

现在使用 EAP-TLS 安全认证的无线网络，典型的工作站交换机上的类似配置是什么样的？

## 使用 802.1x/EAP-TLS 的有线网络身份验证

在这个例子中，我们将展示网络设备的 802.1x 身份验证的交换机端配置（思科）。在这种配置中，工作站使用 EAP-TLS 进行身份验证，我们告诉交换机“信任”电话。虽然这是一种常见的配置，但很容易被规避——攻击者可以告诉他们的笔记本电脑将其数据包“标记”（例如使用`nmcli`命令）为虚拟局域网（VLAN）105（语音 VLAN）。只要交换机信任设备设置自己的 VLAN，这种攻击就不那么困难，尽管从那里继续攻击需要一些努力来使所有参数“完美”。因此，最好是让 PC 和电话都进行身份验证，但这需要额外的设置——电话需要设备证书才能完成这种推荐的配置。

让我们继续我们的示例交换机配置。首先，我们定义 RADIUS 服务器和组（这应该看起来很熟悉，来自管理访问部分）。

允许 802.1x 的交换机配置包括一些全局命令，设置 RADIUS 服务器和 RADIUS 组，并将 802.1x 身份验证链接回 RADIUS 配置。这些命令在以下代码片段中说明：

```
radius server RADIUS01
    address ipv4 <radius server ip 01> auth-port 1812 acct-port 1813
    key <some key>
radius server RADIUS02
    address ipv4 <radius server ip 02> auth-port 1812 acct-port 1813
    key <some key>
aaa group server radius RADIUSGROUP
    server name RADIUS01
    server name RADIUS02
! enable dot1x authentication for all ports by default
dot1x system-auth-control
! set up RADIUS Authentication and Accounting for Network Access
aaa authentication dot1x default group RADIUSGROUP
aaa accounting dot1x default start-stop group RADIUSGROUP
```

接下来，我们配置交换机端口。典型的交换机端口，使用 VLAN 101 上的工作站的 802.1x 身份验证，使用工作站和/或用户证书（之前发放），并且对语音 IP 电话（在 VLAN 105 上）不进行身份验证。请注意，正如我们讨论的那样，身份验证是相互的——工作站在 RADIUS 服务器认证有效的同时，RADIUS 服务器也认证工作站。

![表 9.2 – 交换机 802.1x/EAP-TLS 配置的接口配置](img/B16336_09_Table_02.jpg)

表 9.2 – 交换机 802.1x/EAP-TLS 配置的接口配置

强制 VOIP 电话也使用 802.1x 和证书进行身份验证，删除`trust device cisco-phone`行。这种改变存在一定的政治风险——如果一个人的个人电脑无法进行身份验证，他们又无法打电话给帮助台，那么整个故障排除和解决过程的“温度”立即升高，即使他们可以使用手机打电话给帮助台。

接下来，让我们稍微回顾一下，并添加 Google Authenticator 的多因素认证。当用户 ID 和密码可能是传统解决方案时，通常会使用这种方式。例如，这是保护 VPN 认证免受诸如密码填充攻击之类的问题的绝佳解决方案。

# 使用 Google Authenticator 进行 RADIUS 的多因素认证

正如讨论的那样，对于访问公共服务，特别是面向公共互联网的任何服务，2FA 认证方案是最佳选择，而在过去，您可能已经配置了简单的用户 ID 和密码进行认证。随着持续发生的**短信服务**（**SMS**）泄露事件，我们看到了新闻报道中为什么短信消息不适合作为 2FA 的例子，幸运的是，像 Google Authenticator 这样的工具可以免费配置用于这种情况。

首先，我们将安装一个新的软件包，允许对 Google Authenticator 进行认证，如下所示：

```
$ sudo apt-get install libpam-google-authenticator -y
```

在`users`文件中，我们将更改用户认证以使用**可插拔认证模块**（**PAMs**），如下所示：

```
# Instruct FreeRADIUS to use PAM to authenticate users
DEFAULT Auth-Type := PAM
$ sudo vi /etc/freeradius/3.0/sites-enabled/default
```

取消注释`pam`行，如下所示：

```
#  Pluggable Authentication Modules.
        pam
```

接下来，我们需要编辑`/etc/pam.d/radiusd`文件。注释掉默认的`include`文件，如下面的代码片段所示，并添加 Google Authenticator 的行。请注意，`freeraduser`是一个本地 Linux 用户 ID，将成为该模块的进程所有者：

```
#@include common-auth
#@include common-account
#@include common-password
#@include common-session
auth requisite pam_google_authenticator.so forward_pass secret=/etc/freeradius/${USER}/.google_authenticator user=<freeraduser>
auth required pam_unix.so use_first_pass
```

如果您的 Google Authenticator 服务正常工作，那么与之相关的 RADIUS 链接现在也应该正常工作了！

接下来，生成 Google Authenticator 的秘钥并提供**快速响应**（**QR**）码、账户恢复信息和其他账户信息给客户（在大多数环境中，这可能是一个自助实现）。

现在，当用户对 RADIUS 进行认证（对于 VPN、管理访问或其他任何情况），他们使用常规密码和他们的 Google 秘钥。在大多数情况下，您不希望为无线认证增加这种开销。证书往往是最适合的解决方案，甚至可以说，如果您的无线网络没有使用 EAP-TLS 进行认证，那么它就容易受到一种或多种常见攻击。

# 总结

这结束了我们对使用 RADIUS 对各种服务器进行认证的旅程。与我们在本书中探讨过的许多 Linux 服务一样，本章只是对 RADIUS 可以用来解决的常见配置、用例和组合进行了初步探讨。

在这一点上，您应该具备理解 RADIUS 工作原理并能够为 VPN 服务和管理访问配置安全的 RADIUS 认证，以及无线和有线网络访问的基础知识。您应该具备理解 PAP、CHAP、LDAP、EAP-TLS 和 802.1x 认证协议的基础知识。特别是 EAP-TLS 的使用案例应该说明为什么拥有内部 CA 可以真正帮助您保护网络基础设施。

最后，我们提到了将 Google Authenticator 与 RADIUS 集成以实现多因素认证。尽管我们没有详细介绍 Google Authenticator 服务的配置，但是这似乎最近变化如此频繁，以至于该服务的 Google 文档是最好的参考资料。

在下一章中，我们将讨论如何将 Linux 用作负载均衡器。负载均衡器已经存在多年了，但近年来，它们在物理和虚拟数据中心中的部署频率和方式都有了很大的变化，敬请关注！

# 问题

最后，这里有一些问题供您测试对本章材料的了解。您将在*附录*的*评估*部分找到答案：

1.  对于您打算对其进行管理访问和 VPN 访问认证的防火墙，您如何允许普通用户进行 VPN 访问，但不允许进行管理访问？

1.  为什么 EAP-TLS 是无线网络的一个很好的认证机制？

1.  如果 EAP-TLS 如此出色，为什么 MFA 优先于具有证书的 EAP-TLS 进行 VPN 访问认证？

# 进一步阅读

本章引用的基本 RFC 列在这里：

+   RFC 2865: *RADIUS* ([`tools.ietf.org/html/rfc2865`](https://tools.ietf.org/html/rfc2865))

+   RFC 3579: *EAP 的 RADIUS 支持* ([`tools.ietf.org/html/rfc3579`](https://tools.ietf.org/html/rfc3579))

+   RFC 3580: *IEEE 802.1X RADIUS 使用指南* ([`tools.ietf.org/html/rfc3580`](https://tools.ietf.org/html/rfc3580))

然而，DNS 的完整 RFC 列表很长。以下列表仅显示当前的 RFC - 已废弃和实验性的 RFC 已被删除。当然，这些都可以在[`tools.ietf.org`](https://tools.ietf.org)以及[`www.rfc-editor.org:`](https://www.rfc-editor.org:)找到。

RFC 2548: *Microsoft 特定供应商的 RADIUS 属性*

RFC 2607: *漫游中的代理链接和策略实施*

RFC 2809: *通过 RADIUS 实现 L2TP 强制隧道*

RFC 2865: *远程认证拨号用户服务（RADIUS）*

RFC 2866: *RADIUS 会计*

RFC 2867: *用于隧道协议支持的 RADIUS 会计修改*

RFC 2868: *用于隧道协议支持的 RADIUS 属性*

RFC 2869: *RADIUS 扩展*

RFC 2882: *网络访问服务器要求：扩展的 RADIUS 实践*

RFC 3162: *RADIUS 和 IPv6*

RFC 3575: *RADIUS 的 IANA 考虑事项*

RFC 3579: *EAP 的 RADIUS 支持*

RFC 3580: *IEEE 802.1X RADIUS 使用指南*

RFC 4014: *DHCP 中继代理信息选项的 RADIUS 属性子选项*

RFC 4372: *可计费用户身份*

RFC 4668: *IPv6 的 RADIUS 认证客户端 MIB*

RFC 4669: *IPv6 的 RADIUS 认证服务器 MIB*

RFC 4670: *IPv6 的 RADIUS 会计客户端 MIB*

RFC 4671: *IPv6 的 RADIUS 会计服务器 MIB*

RFC 4675: *虚拟局域网和优先级支持的 RADIUS 属性*

RFC 4679: *DSL 论坛特定供应商的 RADIUS 属性*

RFC 4818: *RADIUS 委派 IPv6 前缀属性*

RFC 4849: *RADIUS 过滤规则属性*

RFC 5080: *常见的 RADIUS 实施问题和建议的修复*

RFC 5090: *摘要认证的 RADIUS 扩展*

RFC 5176: *RADIUS 的动态授权扩展*

RFC 5607: *NAS 管理的 RADIUS 授权*

RFC 5997: *RADIUS 协议中状态服务器数据包的使用*

RFC 6158: *RADIUS 设计指南*

RFC 6218: *Cisco 特定供应商的 RADIUS 属性用于密钥材料的传递*

RFC 6421: *远程认证拨号用户服务（RADIUS）的密码敏捷要求*

RFC 6911: *IPv6 访问网络的 RADIUS 属性*

RFC 6929: *远程认证拨号用户服务（RADIUS）协议扩展*

RFC 8044: *RADIUS 中的数据类型*

+   AD/SMB 集成:

[`wiki.freeradius.org/guide/freeradius-active-directory-integration-howto`](https://wiki.freeradius.org/guide/freeradius-active-directory-integration-howto)

[`web.mit.edu/rhel-doc/5/RHEL-5-manual/Deployment_Guide-en-US/s1-samba-security-modes.html`](https://web.mit.edu/rhel-doc/5/RHEL-5-manual/Deployment_Guide-en-US/s1-samba-security-modes.html)

[`wiki.samba.org/index.php/Setting_up_Samba_as_a_Domain_Member`](https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Domain_Member)

+   802.1x: [`isc.sans.edu/diary/The+Other+Side+of+Critical +Control+1%3A+802.1x+Wired+Network+Access+Controls/25146`](https://isc.sans.edu/diary/The+Other+Side+of+Critical+Control+1%3A+802.1x+Wired+Network+Access+Controls/25146)

+   Unlang 参考:

[`networkradius.com/doc/3.0.10/unlang/home.html`](https://networkradius.com/doc/3.0.10/unlang/home.html)

[`freeradius.org/radiusd/man/unlang.txt`](https://freeradius.org/radiusd/man/unlang.txt)
