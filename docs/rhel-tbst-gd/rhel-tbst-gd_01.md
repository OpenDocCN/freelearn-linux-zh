# 第一章：故障排除最佳实践

这一章，也就是第一章，可能是最重要但技术含量最低的。本书中的大多数章节涵盖了特定问题和解决这些问题所需的命令。然而，这一章将涵盖一些可以应用于任何问题的故障排除最佳实践。

你可以把这一章看作是应用实践背后的原则。

# 故障排除风格

在介绍故障排除的最佳实践之前，了解不同的故障排除风格是很重要的。根据我的经验，我发现人们倾向于使用以下三种故障排除风格：

+   数据收集者

+   受过教育的猜测者

+   适应者

每种风格都有其优点和缺点。让我们来看看这些风格的特点。

## 数据收集者

我喜欢称第一种故障排除风格为**数据收集者**。数据收集者通常是利用系统化方法解决问题的人。系统化的故障排除方法通常具有以下特点：

+   向报告问题的相关方提出具体问题，期望得到详细答案

+   运行命令以识别大多数问题的系统性能

+   在采取行动之前运行预定义的一系列故障排除步骤

这种风格的优势在于，无论是什么级别的工程师或管理员使用它都是有效的。通过系统地处理问题，收集每个数据点，并在执行任何解决方案之前了解结果，数据收集者能够解决他们可能不熟悉的问题。

这种风格的弱点在于数据收集通常不是解决问题的最快方法。根据问题的不同，收集数据可能需要很长时间，而且其中一些数据可能并不是找到解决方案所必需的。

## 受过教育的猜测者

我喜欢称第二种故障排除风格为**受过教育的猜测者**。受过教育的猜测者通常是利用直觉方法解决问题的人。直觉方法通常具有以下特点：

+   用最少的信息确定问题的原因

+   在解决问题之前运行几个命令

+   利用以前的经验来确定根本原因

这种故障排除风格的优势在于它能让你更快地找到解决方案。面对问题时，这种类型的故障排除者倾向于从经验中汲取，并且需要最少的信息来找到解决方案。

这种风格的弱点在于它严重依赖经验，因此在有效之前需要时间。在专注于解决问题时，这种故障排除者可能还会尝试多种行动来解决问题，这可能会使人觉得受过教育的猜测者并没有完全理解手头的问题。

## 适应者

还有一种第三种经常被忽视的故障排除风格；这种风格同时利用了系统化和直觉化的风格。我喜欢称这种风格为**适应者**。适应者有一种个性，使其能够在系统化和直觉化的故障排除风格之间切换。这种结合的风格通常比数据收集者风格更快，比受过教育的猜测者风格更注重细节。这是因为他们能够应用适合手头任务的故障排除风格。

## 选择适当的风格

虽然很容易说一个方法比另一个更好，但事实是选择适当的故障排除风格在很大程度上取决于个人。了解哪种故障排除风格最适合你自己的个性是很重要的。通过了解哪种风格更适合你，你可以学习和使用适合该风格的技术。你也可以学习和采用其他风格的技术，以应用你通常会忽略的故障排除步骤。

本书将展示数据收集者和有经验的猜测者两种故障排除风格，并定期强调哪种个性风格最适合这些步骤。

# 故障排除步骤

故障排除是一个既严格又灵活的过程。故障排除过程的严格性基于需要遵循基本步骤的事实。在这方面，我喜欢将故障排除过程等同于科学方法，科学方法有一系列必须遵循的特定步骤。

故障排除过程的灵活性在于可以按任何有意义的顺序遵循这些步骤。与科学方法不同，故障排除过程通常旨在快速解决问题。有时，为了快速解决问题，您可能需要跳过一步或按顺序执行它们。例如，在故障排除过程中，您可能需要解决即时问题，然后确定该问题的根本原因。

以下列表包括构成故障排除过程的五个步骤。每个步骤可能还包括几个子任务，这些子任务可能与问题相关也可能不相关。重要的是要以一颗谨慎的心态遵循这些步骤，因为并非每个问题都可以归入同一类别。以下步骤旨在作为最佳实践使用，但与所有事物一样，应根据手头的问题进行调整：

1.  理解问题陈述。

1.  建立假设。

1.  试错。

1.  寻求帮助。

1.  文档。

## 理解问题陈述

使用科学方法，第一步是建立问题陈述，这另一种说法是：确定并理解实验的目标。使用故障排除过程，第一步是理解报告的问题。我们越了解问题，解决问题就越容易。

我们可以执行一些任务来更好地理解问题。这第一步是数据收集者个性的显著特点。数据收集者天生倾向于在进入下一步之前收集尽可能多的数据，而有经验的猜测者通常倾向于迅速完成这一步，然后转移到下一步，这有时可能导致错过关键信息。

适应者倾向于了解哪些数据收集步骤是必要的，哪些是不必要的。这使他们能够像数据收集者一样收集数据，但不会花费时间收集对问题没有价值的数据。

这个故障排除步骤中的子任务是*提出正确的问题*。

### 提出问题

无论是通过人工还是自动化流程（如工单系统），报告问题的人通常是信息的重要来源。

#### 工单

当他们收到一个工单时，有经验的猜测者个性通常会阅读工单的标题，假设问题并转移到理解问题的下一个阶段。数据收集者个性通常会打开工单并阅读工单的所有细节。

虽然这取决于工单和监控系统，但通常工单中可能包含有用的信息。除非问题是常见问题，并且您能够从标题中理解所有信息，通常最好阅读工单描述。即使是少量信息也可能有助于解决特别棘手的问题。

#### 人类

然而，从人类那里收集额外信息可能是不一致的。这在很大程度上取决于所支持的环境。在某些环境中，报告问题的人可以提供解决问题所需的所有细节。在其他环境中，他们可能不理解问题，只是解释症状。

无论哪种排除故障风格最适合你的个性，能够从报告问题的人那里获得重要信息是一项重要的技能。直觉性问题解决者，如受过教育的猜测者或适应者，往往比数据收集者个性更容易找到这个过程，不是因为这些个性一定更擅长从人们那里获取细节，而是因为他们能够在较少的信息中识别模式。然而，数据收集者可以通过准备好提出故障排除问题来从报告问题的人那里获得他们需要的信息。

### 注意

**不要害怕问显而易见的问题**

我的第一份技术工作是在一个网络托管技术支持呼叫中心。在那里，我经常接到用户的电话，他们不想执行基本的故障排除步骤，只是希望问题升级。这些用户只是觉得他们已经自己执行了所有的故障排除步骤，并且已经发现了超出一级支持范围的问题。

虽然有时这是真的，但更多的时候，问题是一些他们忽视的基本问题。在那个角色中，我很快就学会了，即使用户不愿意回答基本或显而易见的问题，但在一天结束时，他们只是希望他们的问题得到解决。如果这意味着经历重复的步骤，那也没关系，只要问题得到解决。

即使在今天，作为高级工程师的升级点，我发现很多时候工程师（即使在他们的故障排除经验丰富的情况下）也会忽视简单的基本步骤。

有时问一些看似基本的简单问题是一个很好的时间节省者；所以不要害怕问。

### 尝试复制问题

收集信息和理解问题的最佳方法之一是亲身体验。当问题被报告时，最好是复制问题。

虽然用户可能是很多信息的来源，但他们并不总是最可靠的；用户经常会遇到错误并忽视它，或者在报告问题时简单地忘记传达错误。

通常，我会问用户如何重新创建问题。如果用户能够提供这些信息，我就能看到任何错误，并经常更快地确定问题的解决方案。

### 注意

**有时无法复制问题**

尽管复制问题通常是最好的，但并非总是可能的。每天，我与许多团队合作；有时，这些团队在公司内部，但很多时候它们是外部供应商。在关键问题时，我偶尔会看到有人发表类似于“如果我们无法复制它，我们就无法排除故障”的笼统声明。

尽管复制问题有时是找到根本原因的唯一方法，但我经常听到这种说法被滥用。复制问题应该被视为一种工具；它只是你排除故障工具箱中的众多工具之一。如果它不可用，那么你只能用其他工具。

无法找到解决方案和由于无法复制问题而不尝试找到解决方案之间存在显著的区别。后者不仅没有帮助，而且也不专业。

### 运行调查命令

很可能，你正在阅读这本书是为了学习排除故障红帽企业 Linux 系统的技术和命令。理解问题陈述的第三个子任务就是这样——运行调查命令以确定问题的原因。然而，在执行调查命令之前，重要的是要知道之前的步骤是有逻辑顺序的。

首先询问报告问题的用户一些基本问题的细节是最佳实践，然后在获得足够的信息后，复制问题。一旦问题被复制，下一个逻辑步骤就是运行必要的命令来排除故障和调查问题的原因。

在故障排除过程中，经常会发现自己需要返回到之前的步骤。在你确定了一些关键错误之后，你可能会发现自己必须向最初的报告者询问额外的信息。在故障排除过程中，不要害怕向后退几步，以便更清楚地了解手头的问题。

## 建立假设

使用科学方法，一旦问题陈述被制定出来，就是建立假设的时候了。在故障排除过程中，当你确定了问题，收集了关于问题的信息，比如错误、系统当前状态等，也是建立你认为引起或正在引起问题的原因的时候。

然而，有些问题可能不需要太多的假设。通常日志文件中的错误或系统当前状态可能会解释问题发生的原因。在这种情况下，你可以简单地解决问题，然后继续进行*文档*步骤。

对于那些不是非常明显的问题，你需要提出一个根本原因的假设。这是必要的，因为形成假设之后的下一步是尝试解决问题。如果你至少没有根本原因的理论，那么解决问题就会很困难。

以下是一些可以用来帮助形成假设的技术。

### 整理模式

在进行前面步骤的数据收集时，你可能会开始看到一些模式。模式可以是一些简单的东西，比如多个服务中相似的日志条目，发生的故障类型（比如，多个服务下线），甚至是系统资源利用率的反复波动。

这些模式可以用来制定问题的理论。为了强调这一点，让我们通过一个真实的情景来看一下。

你正在管理一个既运行 Web 应用程序又接收电子邮件的服务器。你有一个监控系统检测到 Web 服务出现错误并创建了一个工单。在调查工单时，你还接到一个电子邮件用户的电话，称他们收到了电子邮件反弹回来的消息。

当你要求用户向你读出错误时，他们提到“设备上没有剩余空间”。

让我们来分析一下这个情景：

+   我们的监控解决方案的工单告诉我们 Apache 已经停止了

+   我们还收到了来自电子邮件用户的报告，其中的错误表明文件系统已满

这一切可能意味着 Apache 已经停止，因为文件系统已满吗？可能。我们应该调查一下吗？当然！

### 这是我以前遇到过的事情吗？

上面的分析引出了形成假设的下一个技术。这可能听起来很简单，但经常被忘记。“我以前见过类似的情况吗？”

在先前的情景中，电子邮件反弹回来的错误报告通常表明文件系统已满。我们怎么知道的？很简单，我们以前见过。也许我们以前在电子邮件反弹回来时见过同样的错误，或者我们在其他服务中见过这个错误。关键是，这个错误是熟悉的，而且这个错误通常意味着一件事。

记住常见错误对于直觉类型的人（如有经验的猜测者和适应者）来说可能非常有用；这是他们自然而然会做的事情。对于数据收集者来说，一个方便的技巧是保留一张常见错误的参考表。

### 提示

根据我的经验，大多数数据收集者倾向于保留一组包含常见命令或程序步骤的笔记。添加常见错误和这些错误背后的含义是系统思维者（如数据收集者）更快地建立假设的好方法。

总的来说，建立假设对所有类型的故障排除者都很重要。这是直觉思维者（如有经验的猜测者和适应者）擅长的领域。通常情况下，这些类型的故障排除者会更早地提出假设，即使有时这些假设并不总是正确的。

## 试错法

在科学方法中，一旦形成假设，下一个阶段就是实验。在故障排除中，这相当于尝试解决问题。

有些问题很简单，可以使用标准程序或经验步骤解决。然而，其他问题并不那么简单。有时，假设结果是错误的，或者问题最终比最初想象的更复杂。

在这种情况下，可能需要多次尝试解决问题。我个人喜欢把这看作是类似于试错。一般来说，你可能对问题出了什么问题（假设）有一个想法，以及如何解决它的想法。你试图解决它（试验），如果不起作用（错误），你就会转向下一个可能的解决方案。

### 首先创建一个备份

对于那些担任新角色的 Linux 系统管理员，如果我只能给出一个建议，那就是大多数人都是通过艰难的方式学到的：*在进行更改之前备份所有内容*。

作为系统管理员，我们经常发现自己需要更改配置文件或删除一些不需要的文件以解决问题。不幸的是，我们可能认为自己知道需要删除或更改的内容，但并不总是正确。

如果已经进行了备份，那么更改可以简单地恢复到先前的状态，但是没有备份。因此，撤销更改并不那么容易。

备份可以包括许多内容，可以是使用诸如`rdiff-backup`之类的完整系统备份，VM 快照，或者简单地创建文件副本。

### 提示

对于那些有兴趣看到这个提示在实践中的程度的人，只需在任何有四名以上系统管理员并且已经存在数年的服务器上运行以下命令：

```
$ find /etc –name "*.bak"

```

## 寻求帮助

在许多情况下，问题在这一点上已经解决了，但就像故障排除过程中的每一步一样，这取决于手头的问题。虽然寻求帮助并不完全是故障排除的步骤，但如果你无法自己解决问题，这往往是下一个逻辑步骤。

在寻求帮助时，通常有六种资源可用：

+   书籍

+   团队维基或运行手册

+   谷歌

+   手册

+   红帽内核文档

+   人们

### 书籍

书籍（比如这本书）对于参考特定类型问题的命令或故障排除步骤是很好的。其他专门针对特定技术的书籍对于参考该技术的工作原理也是很好的。在以前的几年里，看到一位高级管理员手边放着一整排技术书籍是很常见的。

在今天的世界中，由于书籍更频繁地以数字格式出现，它们甚至更容易用作参考。数字格式使它们可以被搜索，并允许读者比传统的印刷版本更快地找到特定部分。

### 团队维基或运行手册

在**团队维基**变得普遍之前，许多运营团队都有名为**运行手册**的实体书籍。这些书是运营团队每天使用的流程和程序列表，用于保持生产环境正常运行。有时，这些运行手册会包含有关配置新服务器的信息，有时它们会专门用于故障排除。

在今天的世界中，这些运行手册大多被团队维基所取代，这些维基通常具有相同的内容，但是在线的。它们也往往是可搜索的，更容易保持最新，这意味着它们通常比传统的印刷运行手册更相关。

团队维基和运行手册的好处在于它们不仅可以解决特定于您环境的问题，而且还可以解决这些问题。有许多配置服务（如 Apache）的方法，外部系统对这些服务创建依赖的方式更多。

在某些环境中，当出现问题时，您可能只需简单地重新启动 Apache，但在其他情况下，您可能实际上需要经历几个先决步骤。如果在重新启动服务之前需要遵循特定的流程，最佳做法是将流程记录在团队 Wiki 或 Runbook 中。

### 谷歌

谷歌是系统管理员如此常用的工具，以至于曾经有特定的搜索门户网站可用，如`google.com/linux`，`google.com/microsoft`，`google.com/mac`和`google.com/bsd`。

谷歌已经停用了这些搜索门户，但这并不意味着系统管理员使用谷歌或任何其他搜索引擎进行故障排除的次数减少了。

事实上，在今天的世界中，在技术面试中听到“我会谷歌一下”这样的话并不罕见。

对于那些刚开始使用谷歌进行系统管理任务的人，一些建议是：

+   如果您复制并粘贴完整的错误消息（删除服务器特定的文本），您可能会找到更相关的结果：

例如，搜索*kdumpctl: No memory reserved for crash kernel*返回 600 个结果，而搜索*memory reserved for crash kernel*返回 449,000 个结果。

+   您可以通过搜索`man`然后是一个命令，比如`man netstat`，找到任何 man 页面的在线版本。

+   您可以用双引号包裹错误以细化搜索结果，使其包含相同的错误。

+   通常以问题的形式询问您要找的内容通常会得到教程。例如，*如何在 RHEL 7 上重新启动 Apache？*

虽然谷歌可能是一个很好的资源，但结果应该始终持保留态度。在谷歌上搜索错误时，您可能会找到一个建议的命令，它提供了很少的解释，只是简单地说“运行这个命令就会修复它”。在运行这些命令时要非常谨慎，重要的是您在执行系统上的任何命令之前都应该熟悉该命令。您应该在执行之前了解命令的作用。

### Man 页面

当谷歌不可用，甚至有时候它可用时，关于命令或 Linux 的最佳信息来源通常是**man 页面**。man 页面是核心 Linux 手册文档，可以通过`man`命令访问。

例如，要查找`netstat`命令的文档，只需运行以下命令：

```
$ man netstat
NETSTAT(8)
Linux System Administrator's Manual
NETSTAT(8)

NAME
       netstat - Print network connections, routing tables, interface statistics, masquerade connections, and multicast memberships
```

正如您所看到的，这个命令不仅输出了`netstat`命令的信息，还包含了使用信息的快速概要，如下所示：

```
SYNOPSIS
      netstat  [address_family_options]  [--tcp|-t]  [--udp|-u] [--udplite|-U] [--raw|-w] [--listening|-l] [--all|-a] [--numeric|-n] [--numeric-hosts]
       [--numeric-ports] [--numeric-users] [--symbolic|-N] [--extend|-e[--extend|-e]]  [--timers|-o]  [--program|-p] [--verbose|-v]  [--continuous|-c]
       [--wide|-W] [delay]
```

此外，它提供了每个标志的详细描述及其作用：

```
   --route , -r
       Display the kernel routing tables. See the description in route(8) for details.  netstat -r and route -e produce the same output.

   --groups , -g
       Display multicast group membership information for IPv4 and IPv6.

   --interfaces=iface , -I=iface , -i
       Display a table of all network interfaces, or the specified iface.
```

一般来说，核心系统和库的基本手册页面是通过`man-pages`软件包分发的。特定命令的 man 页面，如`top`，`netstat`或`ps`，是作为该命令的安装软件包的一部分分发的。这是因为个别命令和组件的文档留给了软件包维护者。

这意味着有些命令的文档水平可能不如其他命令。然而，总的来说，man 页面是非常有用的信息来源，可以回答大多数日常问题。

#### 阅读 man 页面

在前面的例子中，我们可以看到`netstat`的 man 页面包含了一些信息部分。一般来说，man 页面有一个一致的布局，其中包含一些常见的部分，这些部分可以在大多数 man 页面中找到。以下是一些常见部分的简单列表：

+   名称

+   概要

+   描述

+   示例

##### 名称

**名称**部分通常包含命令的名称和对命令的非常简要的描述。以下是`ps`命令的 man 页面中的名称部分：

```
NAME
       ps - report a snapshot of the current processes.
```

##### 概要

命令的 man 页面的**概要**部分通常会列出命令，后面是可能的命令标志或选项。这一部分的一个很好的例子可以在`netstat`命令的概要中看到：

```
SYNOPSIS
       netstat  [address_family_options]  [--tcp|-t]  [--udp|-u] [--raw|-w]  [--listening|-l]  [--all|-a] [--numeric|-n] [--numeric-hosts] [--numeric-ports]
       [--numeric-users] [--symbolic|-N] [--extend|-e[--extend|- e]] [--timers|-o] [--program|-p] [--verbose|-v] [--continuous|-c]
```

```
cat command's man page:
```

```
DESCRIPTION
       Concatenate FILE(s), or standard input, to standard output.

       -A, --show-all
              equivalent to -vET

       -b, --number-nonblank
              number nonempty output lines, overrides -n
```

描述部分非常有用，因为它不仅仅是查找选项。这部分通常是你会找到关于命令细微差别的文档的地方。

##### 示例

通常 man 页面还会包括使用命令的示例：

```
EXAMPLES
       cat f - g
             Output f's contents, then standard input, then g's infocontents.
```

```
cat command's man page. We can see, in this example, how to use cat to read from files and standard input in one command.
```

这部分通常是我发现如何使用我以前多次使用的命令的新方法的地方。

##### 其他部分

除了前面的部分，你可能还会看到诸如**另请参阅**、**文件**、**作者**和**历史**等部分。这些部分也可能包含有用的信息；但并非每个 man 页面都会有这些部分。

#### Info 文档

除了 man 页面，Linux 系统通常还包含**info 文档**，这些文档旨在包含超出 man 页面范围的额外文档。与 man 页面类似，info 文档包含在一个命令包中，文档的质量/数量可以因包而异。

调用 info 文档的方法类似于 man 页面，只需执行 `info` 命令，然后跟上你想查看的主题：

```
$ info gzip
GNU Gzip: General file (de)compression
**************************************

This manual is for GNU Gzip (version 1.5, 10 June 2014), and documents commands for compressing and decompressing data.

   Copyright (C) 1998-1999, 2001-2002, 2006-2007, 2009-2012 Free
Software Foundation, Inc.
```

#### 引用更多命令

除了使用 man 页面和 info 文档查找命令之外；这些工具也可以用来查看系统调用或配置文件等其他项目的文档。

举例来说，如果你使用 `man` 来搜索术语 `signal`，你会看到以下内容：

```
$ man signal
SIGNAL(2)
Linux Programmer's Manual
SIGNAL(2)

NAME
 signal - ANSI C signal handling

SYNOPSIS
 #include <signal.h>

 typedef void (*sighandler_t)(int);

 sighandler_t signal(int signum, sighandler_t handler);

DESCRIPTION
 The  behavior of signal() varies across UNIX versions, and has also varied historically across different versions of Linux. Avoid its use: use sigaction(2) instead.  See Portability below.

signal() sets the disposition of the signal signum to handler, which is either SIG_IGN, SIG_DFL, or the address of a programmer-defined  function  (a "signal handler").

```

`Signal` 是一个非常重要的系统调用和 Linux 的核心概念。知道可以使用 `man` 和 `info` 命令来查找核心 Linux 概念和行为在故障排除过程中非常有用。

#### 安装 man 页面

基于 Red Hat Enterprise Linux 的发行版通常包括 `man-pages` 包；如果你的系统没有安装 `man-pages` 包，你可以使用 `yum` 命令安装它：

```
# yum install man-pages

```

### Red Hat 内核文档

除了 man 页面，Red Hat 发行版还有一个叫做**kernel-doc**的包。这个包包含了关于系统内部工作原理的大量信息。

内核文档是一组文本文件，放置在 `/usr/share/doc/kernel-doc-<kernel-version>/` 中，并按其涵盖的主题进行分类。这个资源对于更深入的故障排除非常有用，比如调整内核可调整参数或理解 `ext4` 文件系统如何利用日志。

默认情况下，`kernel-doc` 包未安装，但可以使用 `yum` 命令轻松安装：

```
# yum install kernel-doc

```

### 人员

无论是朋友还是团队领导，在向他人寻求帮助时都有一定的礼仪。以下是人们在被要求解决问题时通常期望的事情列表。当我被要求帮助时，我希望你能：

+   **尝试自己解决问题**：在升级问题时，最好至少尝试遵循故障排除过程中的*理解问题*和*形成假设*步骤。

+   **记录你尝试过的内容**：文档是升级问题或寻求帮助的关键。你记录的步骤和发现的错误越详细，其他人就越快地能够识别和解决问题。

+   **解释你认为问题是什么以及报告了什么**：当你升级问题时，首先要指出的是你的假设。通常这可以帮助下一个人迅速找到可能的解决方案，而不必进行数据收集活动。

+   **提及最近这个系统是否发生了其他事情**：问题通常是成对出现的，突出系统或受影响系统上发生的所有因素是很重要的。

上述列表虽然不全面，但每个关键信息都可以帮助下一个人有效地解决问题。

#### 跟进

在升级问题时，最好跟进其他人，了解他们做了什么以及如何做的。这很重要，因为它会向你询问的人表明你愿意学习更多，这往往会导致他们花时间解释他们是如何解决和识别问题的。

这样的互动将使你获得更多知识，并帮助建立你的系统管理技能和经验。

## 文档

文档编制是故障排除过程中的关键步骤。在整个过程中，关键是要注意并记录正在执行的操作。为什么记录很重要？主要有三个原因：

+   在升级问题时，你记录的信息越多，就能传递更多信息给其他人

+   如果问题是一个再次发生的问题，文档可以用于更新团队 Wiki 或运行手册

+   如果在你的环境中进行**根本原因分析**（**RCA**），所有这些信息都将需要进行 RCA

根据环境的不同，文档可以是从保存在本地系统文本文件中的简单笔记到票务系统所需的笔记。每个工作环境都不同，但一个普遍规则是*没有太多的文档*。

对于数据收集者来说，这一步相当自然。因为大多数数据收集者的个性通常会为自己保留相当多的笔记。对于受过教育的猜测者来说，这一步可能看起来是不必要的。然而，对于任何再次发生或需要升级的问题，文档都是至关重要的。

应该记录哪些信息？以下列表是一个很好的起点，但与大多数故障排除中的事情一样，它取决于环境和问题：

+   问题陈述，你所理解的

+   导致问题的假设。

+   信息收集步骤中收集的数据：

+   找到的具体错误

+   相关系统指标（例如，CPU、内存和磁盘利用率）

+   信息收集步骤中执行的命令（在合理范围内，不需要包括每个执行的`cd`或`ls`命令）

+   尝试解决问题时采取的步骤，包括执行的具体命令

如果前面的项目有很好的记录，如果问题再次发生，将文档移至团队 Wiki 相对简单。这样做的好处是，其他需要在问题再次发生时解决相同问题的团队成员可以使用 Wiki 文章。

文档编制的三个原因之一是在根本原因分析期间使用文档，这将引出我们下一个话题——建立根本原因分析。

# 根本原因分析

根本原因分析是在事件发生后进行的过程。RCA 过程的目标是确定事件的根本原因，并确定可能的纠正措施，以防止同样的事件再次发生。这些纠正措施可能是简单的，比如建立用户培训，重新配置所有 Web 服务器上的 Apache。

RCA 过程并不局限于技术领域，在航空和职业安全等领域也是一种广泛实践的过程。在这些领域，事件往往不仅仅是几台计算机离线。这些事件可能会危及人的生命。

## 一个良好 RCA 的解剖结构

不同的工作环境可能以不同的方式实施 RCA 过程，但归根结底，每个良好的 RCA 都有一些关键要素：

+   问题的报告情况

+   问题的实际根本原因

+   事件和采取的行动的时间表

+   任何关键数据点

+   防止事件再次发生的行动计划

### 问题的报告情况

故障排除过程中的第一步之一是确定问题；这些信息对 RCA 非常重要。根据问题的原因，其重要性可能有所不同。有时，这些信息将显示问题是否被正确识别。大多数时候，它可以作为问题影响的估计。

了解问题的影响可能非常重要，对于一些公司和问题，这可能意味着收入损失；对于其他公司，这可能意味着损害其品牌，或者根据问题的严重程度，可能什么都不意味着。

### 问题的实际根本原因

根本原因分析的这一要素在其重要性上相当不言而喻。然而，有时可能无法确定根本原因。在本章和第十二章中，我将讨论如何处理无法获得完整根本原因的问题。

## 事件和采取的行动的时间表

如果我们以航空事件为例，很容易看出事件时间表的重要性，比如飞机何时起飞，何时乘客登机，以及维护人员何时完成评估，这些都可能很有用。技术事件的时间表也可能非常有用，因为它可以用来确定影响的持续时间以及采取关键行动的时间。

一个良好的时间表应该包括事件的时间和主要事件。以下是技术事件的时间表示例：

+   08:00，Joe B.打电话给 NOC 热线，报告 Tempe 的电子邮件服务器中断

+   08:15，John C.登录到 Tempe 的电子邮件服务器，注意到它们的可用内存不足。

+   08:17，根据 Runbook，John C.开始逐个重新启动 Tempe 的电子邮件服务器

### 验证根本原因的任何关键数据点

除了事件时间表之外，RCA 还应包括关键数据点。再次以航空事故为例，关键数据点可能是事故期间的天气条件，参与人员的工作时间，或飞机的状况。

我们的时间表示例包括一些关键数据点，其中包括：

+   事故时间：08:00

+   电子邮件服务器的状况：可用内存不足

+   受影响的服务：电子邮件

无论数据点是独立的还是在时间表内，都很重要确保这些数据点在 RCA 中得到充分记录。

## 防止事故再次发生的行动计划

执行根本原因分析的整个目的是确定为什么发生了事故以及防止再次发生的行动计划。

不幸的是，我看到许多 RCA 忽视了这一点。RCA 流程在实施良好时可能很有用；然而，当实施不当时，它们可能会变成浪费时间和资源。

通常，对于实施不良的情况，您会发现需要对每个大或小的事件进行 RCA。这样做的问题在于，它会导致 RCA 的质量降低。只有在事件造成重大影响时才应执行 RCA。例如，硬件故障是无法预防的，您可以使用诸如`smartd`之类的工具主动识别硬盘故障，但除了更换它们之外，您并不能总是防止它们发生故障。要求对每次硬件故障和更换进行 RCA 是 RCA 流程实施不当的一个例子。

当工程师需要确定像硬件故障这样常见的问题的根本原因时，他们忽视了根本原因的过程。当工程师忽视某种类型的事件的 RCA 过程时，它可能会扩散到其他类型的事件，导致 RCA 的质量下降。

根本原因分析应该只针对具有重大影响的事件。轻微事件或例行事件不应该有根本原因分析的要求；但是，它们应该被跟踪。通过跟踪已更换的硬盘数量以及这些硬盘的品牌和型号，可以识别硬件质量问题。对于重置用户密码等例行事件也是如此。通过跟踪这些类型的事件，可以识别可能的改进领域。

## 建立根本原因

为了更好地理解根本原因分析过程，让我们使用在生产环境中看到的一个假设问题。

### 注意

在写入文件时，一个 Web 应用程序崩溃了

登录系统后，你发现应用程序崩溃是因为应用程序尝试写入的文件系统已满。

### 注意

**根本原因并不总是显而易见的原因**

问题的根本原因是文件系统满了吗？不是。虽然文件系统满可能导致应用程序崩溃，但这被称为一个促成因素。促成因素，如文件系统满，可以纠正，但这不会防止问题再次发生。

在这一点上，重要的是要确定文件系统为什么会满。进一步调查后，你发现是因为一位同事禁用了一个删除旧应用程序文件的**定时任务**。在禁用了定时任务后，文件系统上的可用空间逐渐减少。最终，文件系统被 100%利用。

在这种情况下，问题的根本原因是禁用的定时任务。

### 有时你必须牺牲根本原因分析

让我们看另一个假设情况，一个问题导致了停机。由于问题造成了重大影响，它绝对需要进行根本原因分析。问题是，为了解决问题，你需要执行一项活动，这将消除进行准确根本原因分析的可能性。

这些情况有时需要判断，是选择忍受停机更长时间，还是解决停机并牺牲进行根本原因分析的机会。不幸的是，对于这些情况没有单一答案，正确答案取决于问题和受影响的环境。

### 提示

在处理财务系统时，我经常发现自己不得不做出这个决定。对于关键任务系统，答案几乎总是恢复服务优先于进行根本原因分析。然而，只要可能，总是首选首先捕获数据，即使这些数据不能立即审查。

# 了解你的环境

本章的最后一节是我能提出的最重要的最佳实践之一。最后一节涵盖了了解你的环境的重要性。

有些人认为系统管理员的工作止步于系统上安装的应用程序，系统管理员只应关注操作系统及其组件，如网络或文件系统。

我不赞同这种观点。事实上，通常情况下，系统管理员会开始比创建它的开发团队更好地了解应用程序在生产中的工作方式。

根据我的经验，为了真正支持服务器，你必须了解在该服务器内运行的服务和应用程序。例如，在许多企业环境中，系统管理员被期望处理 Web 服务器的配置和管理（例如，Apache 和 Nginx）。然而，同一系统管理员不被期望管理 Apache 后面的应用程序（例如，Java 和 C）。

Apache 与 Java 应用程序有何不同？答案实际上是没有什么不同；归根结底，它们都是在服务器上运行的应用程序。我看到许多管理员一旦问题与应用程序有关，就会简单地放手不管。然而，如果问题与 Apache 有关，他们就会迅速采取行动。

最后，如果这些管理团队与开发团队合作，问题可以更快地得到解决。管理员有责任了解并帮助解决系统上加载的任何软件的问题，无论是操作系统分发的软件还是后来由应用团队安装的软件。

# 总结

在本章中，您了解到故障排除有两种主要风格，直觉式（有经验的猜测者）和系统化（数据收集者）。我们介绍了哪些故障排除步骤对这两种风格最有效，以及一些人（适应者）可以利用这两种风格的故障排除。

在本书的后续章节中，当我们排除现实生活中的场景时，我将利用本章讨论的过程中突出的直觉和系统化的故障排除步骤。

本章没有涉及技术细节；下一章将充满技术细节，我们将介绍和探讨用于故障排除的常见 Linux 命令。