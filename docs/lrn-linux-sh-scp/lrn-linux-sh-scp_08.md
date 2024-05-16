# 第八章：变量和用户输入

在本章中，我们将首先描述变量是什么，以及我们为什么需要它们。我们将解释变量和常量之间的区别。接下来，我们将提供一些关于变量命名的可能性，并介绍一些关于命名约定的最佳实践。最后，我们将讨论用户输入以及如何正确处理它：无论是使用位置参数还是交互式脚本。我们将以介绍`if-then`结构和退出代码结束本章，我们将使用它们来结合位置参数和交互提示。

本章将介绍以下命令：`read`，`test`和`if`。

本章将涵盖以下主题：

+   什么是变量？

+   变量命名

+   处理用户输入

+   交互式与非交互式脚本

# 技术要求

除了具有来自前几章的文件的 Ubuntu 虚拟机外，不需要其他资源。

本章的所有脚本都可以在 GitHub 上找到：[`github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter08`](https://github.com/PacktPublishing/Learn-Linux-Shell-Scripting-Fundamentals-of-Bash-4.4/tree/master/Chapter08)。对于 `name-improved.sh` 脚本，只能在网上找到最终版本。在执行脚本之前，请务必验证头部中的脚本版本。

# 什么是变量？

变量是许多（如果不是所有）编程和脚本语言中使用的标准构建块。变量允许我们存储信息，以便稍后可以引用和使用它，通常是多次。例如，我们可以使用`textvariable`变量来存储句子`This text is contained in the variable`。在这种情况下，`textvariable`的变量名称被称为键，变量的内容（文本）被称为值，构成了变量的键值对。

在我们的程序中，当我们需要文本时，我们总是引用`textvariable`变量。现在可能有点抽象，但我们相信在本章的其余部分看到示例之后，变量的用处将变得清晰起来。

实际上，我们已经看到了 Bash 变量的使用。还记得在第四章 *Linux 文件系统*中，我们看过`BASH_VERSION`和`PATH`变量。让我们看看如何在 shell 脚本中使用变量。我们将使用我们的`hello-world-improved.sh`脚本，而不是直接使用`Hello world`文本，我们将首先将其放入一个变量中并引用它：

```
reader@ubuntu:~/scripts/chapter_08$ cp ../chapter_07/hello-world-improved.sh hello-world-variable.sh
reader@ubuntu:~/scripts/chapter_08$ ls -l
total 4
-rwxrwxr-x 1 reader reader 277 Sep  1 10:35 hello-world-variable.sh
reader@ubuntu:~/scripts/chapter_08$ vim hello-world-variable.sh
```

首先，我们将`hello-world-improved.sh`脚本从`chapter_07`目录复制到新创建的`chapter_08`目录中，并命名为`hello-world-variable.sh`。然后，我们使用`vim`进行编辑。给它以下内容：

```
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-01
# Description: Our first script using variables!
# Usage: ./hello-world-variable.sh
#####################################

hello_text="Hello World!"

# Print the text to the terminal.
echo ${hello_text}

reader@ubuntu:~/scripts/chapter_08$ ./hello-world-variable.sh 
Hello World!
reader@ubuntu:~/scripts/chapter_08$
```

恭喜，您刚刚在脚本中使用了您的第一个变量！正如您所看到的，您可以通过在`${...}`语法中包装其名称来使用变量的内容。从技术上讲，只需在名称前面放置`$`就足够了（例如，`echo $hello_text`）。但是，在那种情况下，很难区分变量名称何时结束以及程序的其余部分开始——例如，如果您在句子中间使用变量（或者更好的是，在单词中间！）。如果使用`${..}`，那么变量名称在`}`处结束是清晰的。

在运行时，我们定义的变量将被实际内容替换，而不是变量名称：这个过程称为*变量插值*，并且在所有脚本/编程语言中都会使用。我们永远不会在脚本中看到或直接使用变量的值，因为在大多数情况下，值取决于运行时配置。

您还将看到我们编辑了头部中的信息。虽然很容易忘记，但如果头部不包含正确的信息，就会降低可读性。请务必确保您的头部是最新的！

如果我们进一步解剖这个脚本，你会看到`hello_text`变量是标题之后的第一行功能性代码。我们称这个为**给变量赋值**。在一些编程/脚本语言中，你首先必须在*分配*之前*声明*一个变量（大多数情况下，这些语言有简写形式，你可以一次性声明和分配）。

声明的需要来自于一些语言是*静态类型*的事实（变量类型——例如字符串或整数——应该在分配值之前声明，并且编译器将检查你是否正确地进行了赋值——例如不将字符串赋值给整数类型的变量），而其他语言是*动态类型*的。对于动态类型的语言，语言只是假定变量的类型是从分配给它的内容中得到的。如果它被分配了一个数字，它将是一个整数；如果它被分配了文本，它将是一个字符串，依此类推。

基本上，变量可以被**赋值**一个值，**声明**或**初始化**。尽管从技术上讲，这些是不同的事情，但你经常会看到这些术语被互换使用。不要太过纠结于此；最重要的是记住你正在*创建变量及其内容*！

Bash 并没有真正遵循任何一种方法。Bash 的简单变量（不包括数组，我们稍后会解释）始终被视为字符串，除非操作明确指定我们应该进行算术运算。看一下下面的脚本和结果（我们为了简洁起见省略了标题）：

```
reader@ubuntu:~/scripts/chapter_08$ vim hello-int.sh 
reader@ubuntu:~/scripts/chapter_08$ cat hello-int.sh 
#/bin/bash

# Assign a number to the variable.
hello_int=1

echo ${hello_int} + 1
reader@ubuntu:~/scripts/chapter_08$ bash hello-int.sh 
1 + 1
```

你可能期望我们打印出数字 2。然而，正如所述，Bash 认为一切都是字符串；它只是打印出变量的值，然后是空格、加号、另一个空格和数字 1。如果我们想要进行实际的算术运算，我们需要一种专门的语法，以便 Bash 知道它正在处理数字：

```
reader@ubuntu:~/scripts/chapter_08$ vim hello-int.sh 
reader@ubuntu:~/scripts/chapter_08$ cat hello-int.sh 
#/bin/bash

# Assign a number to the variable.
hello_int=1

echo $(( ${hello_int} + 1 ))

reader@ubuntu:~/scripts/chapter_08$ bash hello-int.sh 
2
```

通过在`$((...))`中包含`variable + 1`，我们告诉 Bash 将其作为算术表达式进行评估。

# 我们为什么需要变量？

希望你现在明白了如何使用变量。然而，你可能还没有完全理解为什么我们会*想要*或*需要*使用变量。这可能只是为了小小的回报而额外工作，对吧？考虑下一个例子：

```
reader@ubuntu:~/scripts/chapter_08$ vim name.sh 
reader@ubuntu:~/scripts/chapter_08$ cat name.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-01
# Description: Script to show why we need variables.
# Usage: ./name.sh
#####################################

# Assign the name to a variable.
name="Sebastiaan"

# Print the story.
echo "There once was a guy named ${name}. ${name} enjoyed Linux and Bash so much that he wrote a book about it! ${name} really hopes everyone enjoys his book."

reader@ubuntu:~/scripts/chapter_08$ bash name.sh 
There once was a guy named Sebastiaan. Sebastiaan enjoyed Linux and Bash so much that he wrote a book about it! Sebastiaan really hopes everyone enjoys his book.
reader@ubuntu:~/scripts/chapter_08$
```

正如你所看到的，我们不止一次使用了`name`变量，而是三次。如果我们没有这个变量，而我们需要编辑这个名字，我们就需要在文本中搜索每个使用了这个名字的地方。

此外，如果我们在某个地方拼写错误，写成*Sebastian*而不是*Sebastiaan*（如果你感兴趣，这种情况*经常*发生），那么阅读文本和编辑文本都需要更多的努力。此外，这只是一个简单的例子：通常，变量会被多次使用（至少比三次多得多）。

此外，变量通常用于存储程序的*状态*。对于 Bash 脚本，你可以想象创建一个临时目录，在其中执行一些操作。我们可以将这个临时目录的位置存储在一个变量中，任何需要在临时目录中进行的操作都将使用这个变量来找到位置。程序完成后，临时目录应该被清理，变量也将不再需要。对于每次运行程序，临时目录的名称将不同，因此变量的内容也将不同，或者*可变*。

变量的另一个优点是它们有一个名称。因此，如果我们创建一个描述性的名称，我们可以使应用程序更容易阅读和使用。我们已经确定可读性对于 shell 脚本来说总是必不可少的，而使用适当命名的变量可以帮助我们实现这一点。

# 变量还是常量？

到目前为止的例子中，我们实际上使用的是**常量**作为变量。变量这个术语意味着它可以改变，而我们的例子总是在脚本开始时分配一个变量，并在整个过程中使用它。虽然这有其优点（如前面所述，为了一致性或更容易编辑），但它还没有充分利用变量的全部功能。

常量是变量，但是一种特殊类型。简单来说，常量是*在脚本开始时定义的变量，不受用户输入的影响，在执行过程中不改变值*。

在本章后面，当我们讨论处理用户输入时，我们将看到真正的变量。在那里，变量的内容由脚本的调用者提供，这意味着脚本的输出每次调用时都会不同，或者*多样化*。在本书后面，当我们描述条件测试时，我们甚至会根据脚本本身的逻辑在脚本执行过程中改变变量的值。

# 变量命名

接下来是命名的问题。你可能已经注意到到目前为止我们看到的变量有些什么：Bash 变量`PATH`和`BASH_VERSION`都是完全大写的，但在我们的例子中，我们使用小写，用下划线分隔单词（`hello_text`）。考虑以下例子：

```
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-08
# Description: Showing off different styles of variable naming.
# Usage: ./variable-naming.sh
#####################################

# Assign the variables.
name="Sebastiaan"
home_type="house"
LOCATION="Utrecht"
_partner_name="Sanne"
animalTypes="gecko and hamster"

# Print the story.
echo "${name} lives in a ${home_type} in ${LOCATION}, together with ${_partner_name} and their two pets: a ${animalTypes}."
```

如果我们运行这个，我们会得到一个不错的小故事：

```
reader@ubuntu:~/scripts/chapter_08$ bash variable-naming.sh 
Sebastiaan lives in a house in Utrecht, together with Sanne and their two pets: a gecko and hamster.
```

所以，我们的变量运行得很好！从技术上讲，我们在这个例子中所做的一切都是正确的。然而，它们看起来很混乱。我们使用了四种不同的命名约定：用下划线分隔的小写、大写、_ 小写，最后是驼峰命名法。虽然这些在技术上是有效的，但要记住可读性很重要：最好选择一种命名变量的方式，并坚持下去。

正如你所期望的，对此有很多不同的意见（可能和制表符与空格的辩论一样多！）。显然，我们也有自己的意见，我们想要分享：对于普通变量，使用**用下划线分隔的小写**，对于常量使用**大写**。从现在开始，你将在所有后续脚本中看到这种做法。

前面的例子会是这样的：

```
reader@ubuntu:~/scripts/chapter_08$ cp variable-naming.sh variable-naming-proper.sh
reader@ubuntu:~/scripts/chapter_08$ vim variable-naming-proper.sh
vim variable-naming-proper.sh
reader@ubuntu:~/scripts/chapter_08$ cat variable-naming-proper.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-08
# Description: Showing off uniform variable name styling.
# Usage: ./variable-naming-proper.sh
#####################################

NAME="Sebastiaan"
HOME_TYPE="house"
LOCATION="Utrecht"
PARTNER_NAME="Sanne"
ANIMAL_TYPES="gecko and hamster"

# Print the story.
echo "${NAME} lives in a ${HOME_TYPE} in ${LOCATION}, together with ${PARTNER_NAME} and their two pets: a ${ANIMAL_TYPES}."
```

我们希望你同意这看起来*好多了*。在本章后面，当我们介绍用户输入时，我们将使用普通变量，而不是到目前为止一直在使用的常量。

无论你在命名变量时决定了什么，最终只有一件事情真正重要：一致性。无论你喜欢小写、驼峰命名法还是大写，它对脚本本身没有影响（除了某些可读性的利弊，如前所述）。然而，同时使用多种命名约定会极大地混淆事情。一定要确保明智地选择一个约定，然后**坚持下去！**

为了保持清洁，我们通常避免使用大写变量，除了常量。这样做的主要原因是（几乎）Bash 中的所有*环境变量*都是用大写字母写的。如果你在脚本中使用大写变量，有一件重要的事情要记住：**确保你选择的名称不会与预先存在的 Bash 变量发生冲突**。这些包括`PATH`、`USER`、`LANG`、`SHELL`、`HOME`等等。如果你在脚本中使用相同的名称，可能会得到一些意想不到的行为。

最好避免这些冲突，并为你的变量选择唯一的名称。例如，你可以选择`SCRIPT_PATH`变量，而不是`PATH`。

# 处理用户输入

到目前为止，我们一直在处理非常静态的脚本。虽然为每个人准备一个可打印的故事很有趣，但它几乎不能算作一个功能性的 shell 脚本。至少，你不会经常使用它！因此，我们想要介绍 shell 脚本中非常重要的一个概念：**用户输入**。

# 基本输入

在非常基本的层面上，调用脚本后在命令行上输入的所有内容都可以作为输入使用。然而，这取决于脚本如何使用它！例如，考虑以下情况：

```
reader@ubuntu:~/scripts/chapter_08$ ls
hello-int.sh hello-world-variable.sh name.sh variable-naming-proper.sh variable-naming.sh
reader@ubuntu:~/scripts/chapter_08$ bash name.sh 
There once was a guy named Sebastiaan. Sebastiaan enjoyed Linux and Bash so much that he wrote a book about it! Sebastiaan really hopes everyone enjoys his book.
reader@ubuntu:~/scripts/chapter_08$ bash name.sh Sanne
There once was a guy named Sebastiaan. Sebastiaan enjoyed Linux and Bash so much that he wrote a book about it! Sebastiaan really hopes everyone enjoys his book
```

当我们第一次调用`name.sh`时，我们使用了最初预期的功能。第二次调用时，我们提供了额外的参数：`Sanne`。然而，因为脚本根本不解析用户输入，我们看到的输出完全相同。

让我们修改`name.sh`脚本，以便在调用脚本时实际使用我们指定的额外输入：

```
reader@ubuntu:~/scripts/chapter_08$ cp name.sh name-improved.sh
reader@ubuntu:~/scripts/chapter_08$ vim name-improved.sh
reader@ubuntu:~/scripts/chapter_08$ cat name-improved.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-08
# Description: Script to show why we need variables; now with user input!
# Usage: ./name-improved.sh <name>
#####################################

# Assign the name to a variable.
name=${1}

# Print the story.
echo "There once was a guy named ${name}. ${name} enjoyed Linux and Bash so much that he wrote a book about it! ${name} really hopes everyone enjoys his book."

reader@ubuntu:~/scripts/chapter_08$ bash name-improved.sh Sanne
There once was a guy named Sanne. Sanne enjoyed Linux and Bash so much that he wrote a book about it! Sanne really hopes everyone enjoys his book.
```

现在看起来好多了！脚本现在接受用户输入；具体来说，是人的名字。它通过使用`$1`构造来实现这一点：这是*第一个位置参数*。我们称这些参数为位置参数，因为位置很重要：第一个参数将始终被写入`$1`，第二个参数将被写入`$2`，依此类推。我们无法交换它们。只有当我们开始考虑使我们的脚本与标志兼容时，我们才会获得更多的灵活性。如果我们向脚本提供更多的参数，我们可以使用`$3`、`$4`等来获取它们。

你可以提供的参数数量是有限制的。然而，这个限制足够高，以至于你永远不必真正担心它。如果你达到了这一点，你的脚本将变得非常笨重，以至于没有人会使用它！

你可能想将一个句子作为**一个**参数传递给一个 Bash 脚本。在这种情况下，如果你希望将其解释为*单个位置参数*，你需要用单引号或双引号将整个句子括起来。如果不这样做，Bash 将认为句子中的每个空格是参数之间的分隔符；传递句子**This Is Cool**将导致脚本有三个参数：This、Is 和 Cool。

请注意，我们再次更新了标题，包括*Usage*下的新输入。然而，从功能上讲，脚本并不是那么好；我们用男性代词来指代一个女性名字！让我们快速修复一下，看看如果我们现在*省略用户输入*会发生什么：

```
reader@ubuntu:~/scripts/chapter_08$ vim name-improved.sh 
reader@ubuntu:~/scripts/chapter_08$ tail name-improved.sh 
# Date: 2018-09-08
# Description: Script to show why we need variables; now with user input!
# Usage: ./name-improved.sh
#####################################

# Assign the name to a variable.
name=${1}

# Print the story.
echo "There once was a person named ${name}. ${name} enjoyed Linux and Bash so much that he/she wrote a book about it! ${name} really hopes everyone enjoys his/her book."

reader@ubuntu:~/scripts/chapter_08$ bash name-improved.sh 
There once was a person named .  enjoyed Linux and Bash so much that he/she wrote a book about it!  really hopes everyone enjoys his/her book.
```

因此，我们已经使文本更加中性化。然而，当我们在没有提供名字作为参数的情况下调用脚本时，我们搞砸了输出。在下一章中，我们将更深入地讨论错误检查和输入验证，但现在请记住，如果变量缺失/为空，Bash**不会提供错误**；你完全有责任处理这个问题。我们将在下一章中进一步讨论这个问题，因为这是 Shell 脚本中的另一个非常重要的主题。

# 参数和参数

我们需要退一步，讨论一些术语——参数和参数。这并不是非常复杂，但可能有点令人困惑，有时会被错误使用。

基本上，参数是你传递给脚本的东西。在脚本中定义的内容被视为参数。看看下面的例子，看看它是如何工作的：

```
reader@ubuntu:~/scripts/chapter_08$ vim arguments-parameters.sh
reader@ubuntu:~/scripts/chapter_08$ cat arguments-parameters.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-08
# Description: Explaining the difference between argument and parameter.
# Usage: ./arguments-parameters.sh <argument1> <argument2>
#####################################

parameter_1=${1}
parameter_2=${2}

# Print the passed arguments:
echo "This is the first parameter, passed as an argument: ${parameter_1}"
echo "This is the second parameter, also passed as an argument: ${parameter_2}"

reader@ubuntu:~/scripts/chapter_08$ bash arguments-parameters.sh 'first-arg' 'second-argument'
This is the first parameter, passed as an argument: first-arg
This is the second parameter, also passed as an argument: second-argument
```

我们在脚本中以这种方式使用的变量称为参数，但在传递给脚本时被称为参数。在我们的`name-improved.sh`脚本中，参数是`name`变量。这是静态的，与脚本版本绑定。然而，参数每次脚本运行时都不同：可以是`Sebastiaan`，也可以是`Sanne`，或者其他任何名字。

记住，当我们谈论参数时，你可以将其视为*运行时参数*；每次运行都可能不同的东西。如果我们谈论脚本的参数，我们指的是脚本期望的静态信息（通常由运行时参数提供，或者脚本中的一些逻辑提供）。

# 交互式与非交互式脚本

到目前为止，我们创建的脚本使用了用户输入，但实际上并不能称之为交互式。一旦脚本启动，无论是否有参数传递给参数，脚本都会运行并完成。

但是，如果我们不想使用一长串参数，而是提示用户提供所需的信息呢？

输入`read`命令。`read`的基本用法是查看来自命令行的输入，并将其存储在`REPLY`变量中。自己试一试：

```
reader@ubuntu:~$ read
This is a random sentence!
reader@ubuntu:~$ echo $REPLY
This is a random sentence!
reader@ubuntu:~$
```

在启动`read`命令后，您的终端将换行并允许您输入任何内容。一旦您按下*Enter*（或者实际上，直到 Bash 遇到*换行*键），输入将保存到`REPLY`变量中。然后，您可以 echo 此变量以验证它是否实际存储了您的文本。

`read`有一些有趣的标志，使其在 shell 脚本中更易用。我们可以使用`-p`标志和一个参数（用引号括起来的要显示的文本）来向用户显示提示，并且我们可以将要存储响应的变量的名称作为最后一个参数提供：

```
reader@ubuntu:~$ read -p "What day is it? "
What day is it? Sunday
reader@ubuntu:~$ echo ${REPLY}
Sunday
reader@ubuntu:~$ read -p "What day is it? " day_of_week
What day is it? Sunday
reader@ubuntu:~$ echo ${day_of_week}
Sunday
```

在上一个示例中，我们首先使用了`read -p`，而没有指定要保存响应的变量。在这种情况下，`read`的默认行为将其放在`REPLY`变量中。一行后，我们用`day_of_week`结束了`read`命令。在这种情况下，完整的响应保存在一个名为此名称的变量中，如紧随其后的`echo ${day_of_week}`中所示。

现在让我们在实际脚本中使用`read`。我们将首先使用`read`创建脚本，然后使用到目前为止使用的位置参数：

```
reader@ubuntu:~/scripts/chapter_08$ vim interactive.sh
reader@ubuntu:~/scripts/chapter_08$ cat interactive.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-09
# Description: Show of the capabilities of an interactive script.
# Usage: ./interactive.sh
#####################################

# Prompt the user for information.
read -p "Name a fictional character: " character_name
read -p "Name an actual location: " location
read -p "What's your favorite food? " food

# Compose the story.
echo "Recently, ${character_name} was seen in ${location} eating ${food}!

reader@ubuntu:~/scripts/chapter_08$ bash interactive.sh
Name a fictional character: Donald Duck
Name an actual location: London
What's your favorite food? pizza
Recently, Donald Duck was seen in London eating pizza!
```

这样做得相当不错。用户只需调用脚本，而无需查看如何使用它，并且进一步提示提供信息。现在，让我们复制和编辑此脚本，并使用位置参数提供信息：

```
reader@ubuntu:~/scripts/chapter_08$ cp interactive.sh interactive-arguments.sh
reader@ubuntu:~/scripts/chapter_08$ vim interactive-arguments.sh 
reader@ubuntu:~/scripts/chapter_08$ cat interactive-arguments.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-09
# Description: Show of the capabilities of an interactive script, 
# using positional arguments.
# Usage: ./interactive-arguments.sh <fictional character name> 
# <actual location name> <your favorite food>
#####################################

# Initialize the variables from passed arguments.
character_name=${1}
location=${2}
food=${3}

# Compose the story.
echo "Recently, ${character_name} was seen in ${location} eating ${food}!"

reader@ubuntu:~/scripts/chapter_08$ bash interactive-arguments.sh "Mickey Mouse" "Paris" "a hamburger"
Recently, Mickey Mouse was seen in Paris eating a hamburger!
```

首先，我们将`interactive.sh`脚本复制到`interactive-arguments.sh`。我们编辑了此脚本，不再使用`read`，而是从传递给脚本的参数中获取值。我们编辑了标题，使用*新名称和新用法*，并通过提供另一组参数来运行它。再次，我们看到了一个不错的小故事。

因此，您可能会想知道，何时应该使用哪种方法？两种方法最终都得到了相同的结果。但就我们而言，这两个脚本都不够可读或简单易用。请查看以下表格，了解每种方法的优缺点：

|  | **优点** | **缺点** |
| --- | --- | --- |
| 读取 |

+   用户无需了解要提供的参数；他们只需运行脚本，并提示提供所需的任何信息

+   不可能忘记提供信息

|

+   如果要多次重复运行脚本，则需要每次输入响应

+   无法以非交互方式运行；例如，在计划任务中

|

| 参数 |
| --- |

+   可以轻松重复

+   也可以以非交互方式运行

|

+   用户需要在尝试运行脚本**之前**了解要提供的参数

+   很容易忘记提供所需的部分信息

|

基本上，一种方法的优点是另一种方法的缺点，反之亦然。似乎我们无法通过使用任一方法来取胜。那么，我们如何创建一个健壮的交互式脚本，也可以以非交互方式运行呢？

# 结合位置参数和 read

通过结合两种方法，当然可以！在我们开始执行脚本的实际功能之前，我们需要验证是否已提供了所有必要的信息。如果没有，我们可以提示用户提供缺失的信息。

我们将稍微提前查看第十一章，*条件测试和脚本循环*，并解释`if-then`逻辑的基本用法。我们将结合`test`命令，该命令可用于检查变量是否包含值或为空。*如果*是这种情况，*那么*我们可以使用`read`提示用户提供缺失的信息。

在本质上，`if-then`逻辑只不过是说`if <某事>，then 做 <某事>`。在我们的例子中，`if`角色名的变量为空，`then`使用`read`提示输入这个信息。我们将在我们的脚本中为所有三个参数执行此操作。

因为我们提供的参数是位置参数，我们不能只提供第一个和第三个参数；脚本会将其解释为第一个和第二个参数，第三个参数缺失。根据我们目前的知识，我们受到了这个限制。在第十五章中，*使用 getopts 解析 Bash 脚本参数*，我们将探讨如何使用标志提供信息。在这种情况下，我们可以分别提供所有信息，而不必担心顺序。然而，现在我们只能接受这种限制！

在我们解释`test`命令之前，我们需要回顾一下**退出代码**。基本上，每个运行并退出的程序都会返回一个代码给最初启动它的父进程。通常，如果一个进程完成并且执行成功，它会以**代码 0**退出。如果程序的执行不成功，它会以*任何其他代码*退出；然而，这通常是**代码 1**。虽然有关于退出代码的约定，通常你会遇到 0 表示良好退出，1 表示不良退出。

当我们使用`test`命令时，它也会生成符合指南的退出代码：如果测试成功，我们会看到退出代码 0。如果不成功，我们会看到另一个代码（可能是 1）。你可以使用`echo $?`命令查看上一个命令的退出代码。

让我们来看一个例子：

```
reader@ubuntu:~/scripts/chapter_08$ cd
reader@ubuntu:~$ ls -l
total 8
-rw-rw-r-- 1 reader reader    0 Aug 19 11:54 emptyfile
drwxrwxr-x 4 reader reader 4096 Sep  1 09:51 scripts
-rwxrwxr-x 1 reader reader   23 Aug 19 11:54 textfile.txt
reader@ubuntu:~$ mkdir scripts
mkdir: cannot create directory ‘scripts’: File exists
reader@ubuntu:~$ echo $?
1
reader@ubuntu:~$ mkdir testdir
reader@ubuntu:~$ echo $?
0
reader@ubuntu:~$ rmdir testdir/
reader@ubuntu:~$ echo $?
0
reader@ubuntu:~$ rmdir scripts/
rmdir: failed to remove 'scripts/': Directory not empty
reader@ubuntu:~$ echo $?
1
```

在上一个例子中发生了很多事情。首先，我们试图创建一个已经存在的目录。由于在同一位置不能有两个同名目录，所以`mkdir`命令失败了。当我们使用`$?`打印退出代码时，返回了`1`。

接下来，我们成功创建了一个新目录`testdir`。在执行该命令后，我们打印了退出代码，看到了成功的数字：`0`。成功删除空的`testdir`后，我们再次看到了退出代码`0`。当我们尝试使用`rmdir`删除非空的`scripts`目录（这是不允许的）时，我们收到了一个错误消息，并看到退出代码再次是`1`。

让我们回到`test`。我们需要做的是验证一个变量是否为空。如果是，我们希望启动一个`read`提示，让用户输入。首先我们将在`${PATH}`变量上尝试这个（它永远不会为空），然后在`empty_variable`上尝试（它确实为空）。要测试一个变量是否为空，我们使用`test -z <变量名>`：

```
reader@ubuntu:~$ test -z ${PATH}
reader@ubuntu:~$ echo $?
1
reader@ubuntu:~$ test -z ${empty_variable}
reader@ubuntu:~$ echo $?
0
```

虽然这乍看起来似乎是错误的，但想一想。我们正在测试一个变量是否**为空**。由于`$PATH`不为空，测试失败并产生了退出代码 1。对于`${empty_variable}`（我们从未创建过），我们确信它确实为空，退出代码 0 证实了这一点。

如果我们想要将 Bash 的`if`与`test`结合起来，我们需要知道`if`期望一个以退出代码 0 结束的测试。因此，如果测试成功，我们可以做一些事情。这与我们的例子完全吻合，因为我们正在测试空变量。如果你想测试另一种情况，你需要测试一个非零长度的变量，这是`test`的`-n`标志。

让我们先看一下`if`语法。实质上，它看起来像这样：`if <退出代码 0>; then <做某事>; fi`。你可以选择将其放在多行上，但在一行上使用;也会终止它。让我们看看我们是否可以为我们的需求进行操作：

```
reader@ubuntu:~$ if test -z ${PATH}; then read -p "Type something: " PATH; fi
reader@ubuntu:~$ if test -z ${empty_variable}; then read -p "Type something: " empty_variable; fi
Type something: Yay!
reader@ubuntu:~$ echo ${empty_variable} 
Yay!
reader@ubuntu:~$ if test -z ${empty_variable}; then read -p "Type something: " empty_variable; fi
reader@ubuntu:~
```

首先，我们在`PATH`变量上使用了我们构建的`if-then`子句。由于它不是空的，我们不希望出现提示：幸好我们没有得到！我们使用了相同的结构，但现在是使用`empty_variable`。看哪，由于`test -z`返回了退出码 0，所以`if-then`子句的`then`部分被执行，并提示我们输入一个值。在输入值之后，我们可以将其输出。再次运行`if-then`子句不会给我们`read`提示，因为此时变量`empty_variable`不再为空！

最后，让我们将这种`if-then`逻辑融入到我们的`new interactive-ultimate.sh`脚本中：

```
reader@ubuntu:~/scripts/chapter_08$ cp interactive.sh interactive-ultimate.sh
reader@ubuntu:~/scripts/chapter_08$ vim interactive-ultimate.sh 
reader@ubuntu:~/scripts/chapter_08$ cat interactive-ultimate.sh 
#!/bin/bash

#####################################
# Author: Sebastiaan Tammer
# Version: v1.0.0
# Date: 2018-09-09
# Description: Show the best of both worlds!
# Usage: ./interactive-ultimate.sh [fictional-character-name] [actual-
# location] [favorite-food]
#####################################

# Grab arguments.
character_name=$1
location=$2
food=$3

# Prompt the user for information, if it was not passed as arguments.
if test -z ${character_name}; then read -p "Name a fictional character: " character_name; fi
if test -z ${location}; then read -p "Name an actual location: " location; fi
if test -z ${food}; then read -p "What's your favorite food? " food; fi

# Compose the story.
echo "Recently, ${character_name} was seen in ${location} eating ${food}!"

reader@ubuntu:~/scripts/chapter_08$ bash interactive-ultimate.sh 
"Goofy"

Name an actual location: Barcelona
What's your favorite food? a hotdog
Recently, Goofy was seen in Barcelona eating a hotdog!
```

成功！我们被提示输入`location`和`food`，但`character_name`成功地从我们传递的参数中解析出来。我们创建了一个脚本，可以完全交互使用，而无需提供参数，但也可以使用参数进行非交互操作。

虽然这个脚本很有信息量，但效率并不是很高。最好是将`test`直接与传递的参数（`$1`，`$2`，`$3`）结合起来，这样我们只需要一行。在本书的后面，我们将开始使用这样的优化，但现在更重要的是将事情写得详细一些，这样您就可以更容易地理解它们！

# 总结

在本章开始时，我们解释了什么是变量：它是一个标准的构建块，允许我们存储信息，以便以后引用。我们更喜欢使用变量有很多原因：我们可以存储一个值一次并多次引用它，如果需要更改值，我们只需更改一次，新值将在所有地方使用。

我们解释了常量是一种特殊类型的变量：它只在脚本开始时定义一次，不受用户输入的影响，在脚本执行过程中不会改变。

我们继续讨论了一些关于变量命名的注意事项。我们演示了 Bash 在变量命名方面非常灵活：它允许许多不同风格的变量命名。但是，我们解释了如果在同一个脚本或多个脚本之间使用多种不同的命名约定，可读性会受到影响。最好的方法是选择一种变量命名方式，并坚持下去。我们建议使用大写字母表示常量，使用小写字母和下划线分隔其他变量。这将减少本地变量和环境变量之间冲突的机会。

接下来，我们探讨了用户输入以及如何处理它。我们赋予我们脚本的用户改变脚本结果的能力，这几乎是大多数现实生活中功能脚本的必备功能。我们描述了两种不同的用户交互方法：使用位置参数的基本输入，以及使用`read`构造的交互式输入。

我们在本章结束时简要介绍了 if-then 逻辑和`test`命令。我们使用这些概念创建了一种处理用户输入的强大方式，将位置参数与`read`提示结合起来处理缺少的信息，同时介绍了单独使用每种方法的利弊。这样创建了一个脚本，可以根据使用情况进行交互和非交互操作。

本章介绍了以下命令：`read`、`test`和`if`。

# 问题

1.  什么是变量？

1.  我们为什么需要变量？

1.  什么是常量？

1.  为什么变量的命名约定特别重要？

1.  什么是位置参数？

1.  参数和参数之间有什么区别？

1.  我们如何使脚本交互式？

1.  我们如何创建一个既可以进行非交互操作又可以进行交互操作的脚本？

# 进一步阅读

如果您想更深入地了解本章主题，以下资源可能会很有趣：

+   **Bash 变量**：[`ryanstutorials.net/bash-scripting-tutorial/bash-variables.php`](https://ryanstutorials.net/bash-scripting-tutorial/bash-variables.php)

+   谷歌 Shell 风格指南：[`google.github.io/styleguide/shell.xml`](https://google.github.io/styleguide/shell.xml)
