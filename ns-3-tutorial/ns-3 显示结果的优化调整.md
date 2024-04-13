## 6.1 Using the Logging Module（日志模块）
在我们讨论 first.cc 脚本时，我们已经简要介绍了 ns-3 日志模块。现在我们将更详细地了解一下，看看日志子系统被设计用来覆盖哪些用例。
#### 1.1. Logging Overview
许多大型系统都支持==某种形式的消息记录==功能，而 ns-3 也不例外。在某些情况下，只有错误消息被记录到“操作员控制台”（通常是 Unix 系统中的 stderr）。在其他系统中，除了更详细的信息消息外，还可能输出警告消息。在某些情况下，记录设施用于输出调试消息，这可能迅速使输出变得模糊。

ns-3 认为所有这些详细级别都是有用的，我们提供了一种可选择的、多级别的消息记录方法。记录可以完全禁用，在组件级别启用，或全局启用；并提供可选择的详细级别。**ns-3 日志模块**（logging module）提供了一种简单、相对容易使用的方法，**可以从模拟中获取有用的信息**。

您应该了解，我们提供了一个通用的机制 —— tracing（跟踪） —— 用于从您的模型中获取数据，应优先用于模拟输出（有关我们的跟踪系统的更多详细信息，请参阅教程部分“使用跟踪系统”）。==日志记录应该优先用于调试信息、警告、错误消息==，或者任何时候您想轻松地从脚本或模型中获取快速消息。

当前系统中**定义了七个日志消息级**别，其**详细级别逐渐增加**。

>LOG_ERROR — 记录*错误消息*（关联的宏：NS_LOG_ERROR）；
LOG_WARN — 记录*警告消息*（关联的宏：NS_LOG_WARN）；
LOG_DEBUG — 记录相对罕见的、临时的*调试消息*（关联的宏：NS_LOG_DEBUG）；
LOG_INFO — 记录有关*程序进度*的信息消息（关联的宏：NS_LOG_INFO）；
LOG_FUNCTION — 记录描述每个*被调用函数的消息*（有两个关联的宏：NS_LOG_FUNCTION，用于成员函数，以及NS_LOG_FUNCTION_NOARGS，用于静态函数）；
LOG_LOGIC — 记录描述*函数内逻辑流*的消息（关联的宏：NS_LOG_LOGIC）；
LOG_ALL — 记录*上述所有内容*（没有关联的宏）。

对于每个 LOG_TYPE，还有 LOG_LEVEL_TYPE，如果使用，将启用其上面所有级别的记录，此外还包括它的级别。（由此带来的结果是，LOG_ERROR 和 LOG_LEVEL_ERROR，以及 LOG_ALL 和 LOG_LEVEL_ALL 在功能上是等效的。）例如，启用 LOG_INFO 将只启用由 NS_LOG_INFO 宏提供的消息，而启用 LOG_LEVEL_INFO 还将启用由 NS_LOG_DEBUG、NS_LOG_WARN 和 NS_LOG_ERROR 宏提供的消息。

我们还提供一个无条件记录的宏，始终显示，无论记录级别或组件选择如何。

NS_LOG_UNCOND — 无条件记录关联的消息（没有关联的日志级别）。
每个级别可以单独或累积请求；记录可以使用 shell 环境变量（NS_LOG）或通过日志系统函数调用设置。正如在教程中前面看到的，日志系统具有 Doxygen 文档，如果您还没有阅读，现在是研究日志模块文档的好时机。

既然您已经详细阅读了文档，让我们利用其中的一些知识，从您已经构建的 scratch/myfirst.cc 示例脚本中获取一些有趣的信息。

#### 1.2. 启用日志记录

让我们使用 NS_LOG 环境变量来增加一些日志记录，但首先，为了确保我们的方向正确，继续运行之前的脚本，就像之前一样：

```bash
$ ./ns3 run scratch/myfirst
```

你应该看到第一个 ns-3 示例程序的熟悉输出：

```
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
```

原来你在上面看到的 "Sent" 和 "Received" 消息实际上是来自 UdpEchoClientApplication 和 UdpEchoServerApplication 的日志消息✅。我们可以通过设置其日志级别来要求客户端应用程序打印更多信息，通过 NS_LOG 环境变量。

我将从这里开始假设你使用的是类似 sh 的 shell，使用 "VARIABLE=value" 语法。如果你使用的是类似 csh 的 shell，那么你将不得不将我的示例转换为这些 shell 所需的 "setenv VARIABLE value" 语法。

现在，UDP 回显客户端应用程序正在响应 scratch/myfirst.cc 中的以下代码行：

```cpp
LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
```

==此代码行启用了 LOG_LEVEL_INFO 级别的日志==记录。当我们**传递一个日志级别标志**时，==实际上是==**启用给定级别及以下所有级别**。

在这种情况下，我们已经启用了 NS_LOG_INFO、NS_LOG_DEBUG、NS_LOG_WARN 和 NS_LOG_ERROR。
==我们可以通过设置 NS_LOG 环境变量来增加日志级别==，以便**在不更改脚本和重新编译的情况下**获取更多信息，如下所示：

```bash
$ export NS_LOG=UdpEchoClientApplication=level_all
```

这将把 shell 环境变量 NS_LOG 设置为字符串：

```
UdpEchoClientApplication=level_all
```

赋值语句的左侧是我们要设置的日志组件的名称，右侧是我们要使用的标志。

在这种情况下，我们将打开应用程序的所有调试级别。如果以这种方式设置 NS_LOG 运行脚本，ns-3 日志系统将检测到更改，你应该看到类似以下的输出：

```
UdpEchoClientApplication:UdpEchoClient(0xef90d0)
UdpEchoClientApplication:SetDataSize(0xef90d0, 1024)
UdpEchoClientApplication:StartApplication(0xef90d0)
UdpEchoClientApplication:ScheduleTransmit(0xef90d0, +0ns)
UdpEchoClientApplication:Send(0xef90d0)
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
UdpEchoClientApplication:HandleRead(0xef90d0, 0xee7b20)
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
UdpEchoClientApplication:StopApplication(0xef90d0)
UdpEchoClientApplication:DoDispose(0xef90d0)
UdpEchoClientApplication:~UdpEchoClient(0xef90d0)
```

应用程序提供的附加调试信息来自 NS_LOG_FUNCTION 级别。这显示了在脚本执行期间每次调用应用程序中的函数的时间。通常，在成员函数中使用（至少）NS_LOG_FUNCTION(this)是首选的。仅在静态函数中使用 NS_LOG_FUNCTION_NOARGS()。然而，请注意，在 ns-3 系统中，没有必须支持任何特定日志功能的模型的要求。有关记录多少信息的决定留给了各个模型开发人员。在回声应用程序的情况下，提供了相当多的日志输出。

现在，你可以看到应用程序的所有函数调用的日志。==如果你仔细观察，你会注意到字符串 "UdpEchoClientApplication" 和你可能期望看到 C++ 作用域运算符 (::) 的方法名之间有一个冒号而不是两个冒号。这是有意的。==

**该名称实际上不是类名，而是日志组件名**。当源文件和类之间存在一对一的对应关系时，通常这将是类名，但你应该理解它实际上不是类名，而且在这里使用单冒号而不是双冒号是为了以相对微妙的方式提醒你将日志组件名从类名中概念上分离开来。

事实证明，**在某些情况下，确定哪个方法实际上生成了日志消息可能很难**。
如果你查看上面的文本，你可能会想知道字符串 "Received 1024 bytes from 10.1.1.2" 来自何处。
你可以通过将 prefix_func 级别 OR 到 NS_LOG 环境变量中来解决这个问题。尝试执行以下操作：

```bash
$ export 'NS_LOG=UdpEchoClientApplication=level_all|prefix_func' // 显式声明所有Client发出的消息
```

请注意，引号是必需的，因为我们用来表示 OR 操作的垂直线也是 Unix 管道连接器。

现在，如果你运行脚本，你将看到日志系统确保==来自给定日志组件的每条消息都以组件名为前缀。==

```
UdpEchoClientApplication:UdpEchoClient(0xea8e50)
UdpEchoClientApplication:SetDataSize(0xea8e50, 1024)
UdpEchoClientApplication:StartApplication(0xea8e50)
UdpEchoClientApplication:ScheduleTransmit(0xea8e50, +0ns)
UdpEchoClientApplication:Send(0xea8e50)
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
UdpEchoClientApplication:HandleRead(0xea8e50, 0xea5b20)
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
UdpEchoClientApplication:StopApplication(0xea8e50)
UdpEchoClientApplication:DoDispose(0xea8e50)
UdpEchoClientApplication:~UdpEchoClient(0xea8e50)
```

现在，UDP 回显客户端应用程序生成的所有消息都**被明确标识为此类消息**。字符串 "Received 1024 bytes from 10.1.1.2" 现在清楚地标识为来自回显客户端应用程序的消息。

此外，在大多数日志语句中，你将看到打印出十六进制值，如 0xea8e50；这是因为大多数语句会打印出 C++ this 指针的值，以便可以区分对象。剩下的消息必须来自 UDP 回显服务器应用程序。

我们可以通过在 NS_LOG 环境变量中输入一个以冒号分隔的组件列表来启用该组件。

```bash
$ export 'NS_LOG=UdpEchoClientApplication=level_all|prefix_func:      // 显式声明所有Client发出的消息
               UdpEchoServerApplication=level_all|prefix_func'        // 显式声明所有Server发出的消息
```
警告：你需要删除上述示例文本中冒号后的换行符，这只是为了文档格式而存在的。

现在，如果你运行脚本，你将看到来自回声客户端和服务器应用程序的所有日志消息。你可能会发现这在调试问题时非常有用。

```
UdpEchoServerApplication:UdpEchoServer(0x2101590)
UdpEchoClientApplication:UdpEchoClient(0x2101820)
UdpEchoClientApplication:SetDataSize(0x2101820, 1024)
UdpEchoServerApplication:StartApplication(0x2101590)
UdpEchoClientApplication:StartApplication(0x2101820)
UdpEchoClientApplication:ScheduleTransmit(0x2101820, +0ns)
UdpEchoClientApplication:Send(0x2101820)
UdpEchoClientApplication:Send(): At time +2s client sent 1024 bytes to 10.1.1.2 port 9
UdpEchoServerApplication:HandleRead(0x2101590, 0x2106240)
UdpEchoServerApplication:HandleRead(): At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
UdpEchoServerApplication:HandleRead(): Echoing packet
UdpEchoServerApplication:HandleRead(): At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
UdpEchoClientApplication:HandleRead(0x2101820, 0x21134b0)
UdpEchoClientApplication:HandleRead(): At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
UdpEchoClientApplication:StopApplication(0x2101820)
UdpEchoServerApplication:StopApplication(0x2101590)
UdpEchoClientApplication:DoDispose(0x2101820)
UdpEchoServerApplication:DoDispose(0x2101590)
UdpEchoClientApplication:~UdpEchoClient(0x2101820)
UdpEchoServerApplication:~UdpEchoServer(0x2101590)
```

有时候，能够看到生成日志消息的模拟时间也是很有用的。你可以通过 OR 运算符添加 prefix_time 位来实现这一点。

```bash
$ export 'NS_LOG=UdpEchoClientApplication=level_all|prefix_func|prefix_time: // 显式声明所有Client发出的消息and生成消息的时间
               UdpEchoServerApplication=level_all|prefix_func|prefix_time'   // 显式声明所有Server发出的消息and生成消息的时间
```

您现在需要移除上面的换行符。如果现在运行脚本，您应该会看到以下输出：

```plaintext
+0.000000000s UdpEchoServerApplication:UdpEchoServer(0x8edfc0)
+0.000000000s UdpEchoClientApplication:UdpEchoClient(0x8ee210)
+0.000000000s UdpEchoClientApplication:SetDataSize(0x8ee210, 1024)
+1.000000000s UdpEchoServerApplication:StartApplication(0x8edfc0)
+2.000000000s UdpEchoClientApplication:StartApplication(0x8ee210)
+2.000000000s UdpEchoClientApplication:ScheduleTransmit(0x8ee210, +0ns)
+2.000000000s UdpEchoClientApplication:Send(0x8ee210)
+2.000000000s UdpEchoClientApplication:Send(): At time +2s client sent 1024 bytes to 10.1.1.2 port 9
+2.003686400s UdpEchoServerApplication:HandleRead(0x8edfc0, 0x936770)
+2.003686400s UdpEchoServerApplication:HandleRead(): At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
+2.003686400s UdpEchoServerApplication:HandleRead(): Echoing packet
+2.003686400s UdpEchoServerApplication:HandleRead(): At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
+2.007372800s UdpEchoClientApplication:HandleRead(0x8ee210, 0x8f3140)
+2.007372800s UdpEchoClientApplication:HandleRead(): At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
+10.000000000s UdpEchoClientApplication:StopApplication(0x8ee210)
+10.000000000s UdpEchoServerApplication:StopApplication(0x8edfc0)
UdpEchoClientApplication:DoDispose(0x8ee210)
UdpEchoServerApplication:DoDispose(0x8edfc0)
UdpEchoClientApplication:~UdpEchoClient(0x8ee210)
UdpEchoServerApplication:~UdpEchoServer(0x8edfc0)
```
（看行首！可以清晰地得出时间轴！）
您可以看到，UdpEchoServer 的构造函数在模拟时间为 0 秒时被调用。实际上，这是在模拟开始之前发生的，但时间显示为零秒。对于 UdpEchoClient 构造函数的消息也是一样的。

回想一下，scratch/myfirst.cc 脚本在模拟开始后的一秒钟启动了回显服务器应用程序。现在您可以看到服务器的 StartApplication 方法实际上在一秒钟时被调用。您还可以看到我们在脚本中请求的时刻，即两秒时启动了回显客户端应用程序。

您现在可以通过在客户端的 ScheduleTransmit 调用到回显服务器应用程序的 HandleRead 回调中跟踪模拟的进度。注意，将数据包发送到点对点链路的经过时间为 3.69 毫秒。您会看到回显服务器记录一条消息，告诉您它已经回显了数据包，然后，在另一次信道延迟后，您会看到回显客户端在其 HandleRead 方法中接收到了回显的数据包。

在此模拟中，发生了很多您看不到的事情。您可以通过在系统中打开所有日志记录组件来轻松跟踪整个过程。尝试将 NS_LOG 变量设置为以下内容：

```bash
$ export 'NS_LOG=*=level_all|prefix_func|prefix_time'
```

上面的**星号是日志组件通配符。这将在模拟中使用的所有组件中打开所有日志记录**。我不会在此处复制输出（截至撰写本文时，对于单个数据包的回显，它产生**数千行输出**），但如果您愿意，您可以将此信息重定向到文件中，并在您喜欢的编辑器中查看，
##### ==recommend 调试命令==
```bash
$ ./ns3 run scratch/myfirst > log.out 2>&1
```

当我遇到问题而又不知道哪里出了问题时，我个人会使用这种非常详细的日志记录。我可以很容易地跟踪代码的进展，而无需设置断点并在调试器中逐步执行代码。我只需在我喜欢的编辑器中编辑输出，搜索我期望的事物，并查看我不期望发生的事物。当我对问题的发生有一个大致的想法时，我会切换到调试器，进行对问题的精细检查。当脚本执行某些完全出乎意料的操作时，此类输出尤其有用。如果您在调试器中逐步执行，您可能会完全错过意外的执行。记录下这一过程使其迅速可见。

#### 1.3. 手动将日志加入代码
您可以通过使用几个宏调用来向您的模拟添加新的日志记录。让我们在我们在 scratch 目录中的 myfirst.cc 脚本中进行此操作。

回想一下，我们在该脚本中定义了一个日志组件：

```cpp
NS_LOG_COMPONENT_DEFINE("FirstScriptExample");
```

您现在知道，通过将 NS_LOG 环境变量设置为各个级别，可以启用此组件的所有日志记录。让我们继续在脚本中添加一些日志记录。==用于添加信息级别日志消息的宏是 `NS_LOG_INFO`==。在创建节点之前的代码段中添加一条消息，告诉您脚本正在“创建拓扑”。代码如下：

打开您喜欢的编辑器并在 `scratch/myfirst.cc` 中添加以下行：

```cpp
NS_LOG_INFO("Creating Topology"); // 你可以直接类比于“cout”！
```

添加在以下行之前，

```cpp
NodeContainer nodes;
nodes.Create(2);
```

现在使用 ns3 构建脚本，并清除之前启用的大量日志记录的 NS_LOG 变量：

```bash
$ ./ns3
$ export NS_LOG=""
```

现在，如果运行脚本，

```bash
$ ./ns3 run scratch/myfirst
```

您将看不到新消息，因为其关联的日志组件（`FirstScriptExample`）尚未启用。

为了看到消息，您必须通过设置级别大于或等于 `NS_LOG_INFO` 的 `FirstScriptExample` 日志组件。如果您只想看到此特定级别的日志记录，可以通过以下方式启用它：

```bash
$ export NS_LOG=FirstScriptExample=info
```

如果现在运行脚本，您将看到您的新的“Creating Topology”日志消息：

```plaintext
Creating Topology
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
```

## 6.2. Using Command Line Arguments
#### 6.2.1. Overriding Default Attributes
您可以==通过命令行参数更改 ns-3 脚本的行为==，而无需编辑和构建。我们提供了一种解析命令行参数并根据这些参数自动设置本地和全局变量的机制。

使用命令行参数系统的第一步是声明命令行解析器。这可以通过在主程序中执行以下代码来简单实现：

```cpp
int main(int argc, char *argv[])
{
  ...

  CommandLine cmd;
  cmd.Parse(argc, argv);

  ...
}
```

这简单的两行代码片段本身实际上非常有用。它为 ns-3 全局变量和属性系统打开了大门。请在 `scratch/myfirst.cc` 脚本的 main 函数开始处添加这两行代码。构建脚本并运行它，但通过以下方式向脚本请求帮助：

```bash
$ ./ns3 run "scratch/myfirst --PrintHelp"
```

这将要求 ns3 运行 `scratch/myfirst` 脚本并将命令行参数 `--PrintHelp` 传递给脚本。**引号是必需的，以解决哪个程序获得哪个参数的问题**。命令行解析器现在将看到 `--PrintHelp` 参数并回复：

```plaintext
myfirst [General Arguments]
General Arguments:
  --PrintGlobals:              Print the list of globals.
  --PrintGroups:               Print the list of groups.
  --PrintGroup=[group]:        Print all TypeIds of group.
  --PrintTypeIds:              Print all TypeIds.
  --PrintAttributes=[typeid]:  Print all attributes of typeid.
  --PrintVersion:              Print the ns-3 version.
  --PrintHelp:                 Print this help message.
```

让我们专注于 `--PrintAttributes` 选项。我们在遍历 `first.cc` 脚本时已经提到了 ns-3 属性系统。我们看过以下代码行：

```cpp
PointToPointHelper pointToPoint;
pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));
```

并提到 `DataRate` 实际上是 `PointToPointNetDevice` 的一个属性。让==我们使用命令行参数解析器查看 `PointToPointNetDevice` 的属性==。帮助列表说我们应该提供==一个 `TypeId`。这对应于属性属于的类的类名==。在这种情况下，它将是 `ns3::PointToPointNetDevice`。让我们继续键入：

```bash
$ ./ns3 run "scratch/myfirst --PrintAttributes=ns3::PointToPointNetDevice"
```

系统将打印出此类网络设备的所有属性。在列出的属性中，您将看到：

```plaintext
--ns3::PointToPointNetDevice::DataRate=[32768bps]:
  The default data rate for point to point links
```

这是在系统中创建 `PointToPointNetDevice` 时将使用的默认值。我们通过上面 `PointToPointHelper` 中的属性设置覆盖了这个默认值。

让我们使用点对点设备和通道的默认值，删除 `myfirst.cc` 中的 `SetDeviceAttribute` 调用和 `SetChannelAttribute` 调用。

现在，您的脚本应该只声明 `PointToPointHelper`，并像以下示例一样不执行任何设置操作：

```cpp
...

NodeContainer nodes;
nodes.Create(2);

PointToPointHelper pointToPoint;

NetDeviceContainer devices;
devices = pointToPoint.Install(nodes);

...
```

使用 ns3 构建新脚本 (`./ns3`)，然后让我们返回并从 UDP 回显服务器应用程序启用一些日志记录，并打开时间前缀。

```bash
$ export 'NS_LOG=UdpEchoServerApplication=level_all|prefix_time'
```

如果运行脚本，您现在应该看到以下输出：

```plaintext
+0.000000000s UdpEchoServerApplication:UdpEchoServer(0x20d0d10)
+1.000000000s UdpEchoServerApplication:StartApplication(0x20d0d10)
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
+2.257324218s UdpEchoServerApplication:HandleRead(0x20d0d10, 0x20900b0)
+2.257324218s At time +2.25732s server received 1024 bytes from 10.1.1.1 port 49153
+2.257324218s Echoing packet
+2.257324218s At time +2.25732s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.51465s client received 1024 bytes from 10.1.1.2 port 9
+10.000000000s UdpEchoServerApplication:StopApplication(0x20d0d10)
UdpEchoServerApplication:DoDispose(0x20d0d10)
UdpEchoServerApplication:~UdpEchoServer(0x20d0d10)
```

回顾一下，我们上次查看回声服务器接收数据包的模拟时间时，是在 2.0073728 秒。

```plaintext
+2.007372800s UdpEchoServerApplication:HandleRead(): Received 1024 bytes from 10.1.1.1
```

现在它在 2.25732 秒接收到数据包。这是因为我们将 PointToPointNetDevice 的数据速率从*5兆每秒*降低到其默认值 *32768 每秒*。

如果我们通过命令行提供一个新的 DataRate，我们可以再次加速模拟。我们可以按照帮助项隐含的公式以下列方式执行：

```bash
$ ./ns3 run "scratch/myfirst --ns3::PointToPointNetDevice::DataRate=5Mbps"
```

这将将 DataRate 属性的默认值设置回每秒五兆位。您对结果感到惊讶吗？事实证明，为了恢复脚本的原始行为，我们将不得不再次设置通道的光速延迟。我们可以要求命令行系统打印出通道的属性，就像我们为网络设备做的那样：

```bash
$ ./ns3 run "scratch/myfirst --PrintAttributes=ns3::PointToPointChannel"
```

我们发现通道的延迟属性设置如下：

```plaintext
--ns3::PointToPointChannel::Delay=[0ns]:
  Transmission delay through the channel
```

然后，我们可以通过命令行系统设置这些默认值：

```bash
$ ./ns3 run "scratch/myfirst
  --ns3::PointToPointNetDevice::DataRate=5Mbps
  --ns3::PointToPointChannel::Delay=2ms"
```

在这种情况下，我们恢复了在脚本中显式设置 DataRate 和 Delay 时的时间：

```plaintext
+0.000000000s UdpEchoServerApplication:UdpEchoServer(0x1df20f0)
+1.000000000s UdpEchoServerApplication:StartApplication(0x1df20f0)
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
+2.003686400s UdpEchoServerApplication:HandleRead(0x1df20f0, 0x1de0250)
+2.003686400s At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
+2.003686400s Echoing packet
+2.003686400s At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
+10.000000000s UdpEchoServerApplication:StopApplication(0x1df20f0)
UdpEchoServerApplication:DoDispose(0x1df20f0)
UdpEchoServerApplication:~UdpEchoServer(0x1df20f0)
```

请注意，数据包再次在 2.00369 秒被服务器接收。

实际上，我们可以用这种方式设置脚本中使用的任何属性！
特别是，我们可以设置 UdpEchoClient 的 MaxPackets 属性为除 1 之外的其他值。

您会如何操作呢？试一试。请记住，您必须注释掉我们在脚本中覆盖默认属性并显式设置 MaxPackets 的地方。然后，您必须重新构建脚本。您还必须找到使用命令行帮助设施实际设置新默认属性值的语法。一旦弄清楚了这一点，您就可以从命令行控制回显的数据包数量。由于我们是好人，我们会告诉您，您的命令行应最终看起来像：

```bash
$ ./ns3 run "scratch/myfirst
  --ns3::PointToPointNetDevice::DataRate=5Mbps
  --ns3::PointToPointChannel::Delay=2ms
  --ns3::UdpEchoClient::MaxPackets=2"
```

此时一个自然的问题是如何了解所有这些属性的存在。同样，命令行帮助设施有一个功能可以做到这一点。如果我们要求命令行帮助，我们应该看到：

```bash
$ ./ns3 run "scratch/myfirst --PrintHelp"
myfirst [General Arguments]

General Arguments:
  --PrintGlobals:              Print the list of globals.
  --PrintGroups:               Print the list of groups.
  --PrintGroup=[group]:        Print all TypeIds of group.
  --PrintTypeIds:              Print all TypeIds.
  --PrintAttributes=[typeid]:  Print all attributes of typeid.
  --PrintVersion:              Print the ns-3 version.
  --PrintHelp:                 Print this help message.
```

如果选择 "PrintGroups" 参数，您应该会看到所有注册的 TypeId 组的列表。组名

与源目录中的模块名称对齐（尽管首字母大写）。一次性打印所有信息会太多，因此可以进一步过滤以按组打印信息。因此，再次关注点对点模块：

```bash
./ns3 run "scratch/myfirst --PrintGroup=PointToPoint"
```

PointToPoint 组中的 TypeId：
```plaintext
TypeIds in group PointToPoint:
  ns3::PointToPointChannel
  ns3::PointToPointNetDevice
  ns3::PppHeader
```

从这里，可以找到要搜索属性的可能 TypeId 名称，例如上面示例中的 `--PrintAttributes=ns3::PointToPointChannel`。了解属性的另一种方法是通过 ns-3 Doxygen；有一页列出了模拟器中的所有注册属性。

了解属性的另一种方法是通过 ns-3 Doxygen；有一个页面列出了模拟器中的所有注册属性。

#### 6.2.2. Hooking Your Own Values
您还可以通过使用`AddValue`方法将自己的hook（钩子）添加到命令行系统中。这可以通过在命令行解析器中使用`AddValue`方法来完成。

让我们使用这个功能==以完全不同的方式指定要回显的数据包数==。让我们添加一个名为 `nPackets` 的局部变量到 `main` 函数中。我们将其初始化为1，以匹配先前的默认行为。==为了允许命令行解析器更改此值，我们需要将该值连接到解析器中==。我们==通过添加对 `AddValue` 的调用来实现这一点==。请继续修改 `scratch/myfirst.cc` 脚本，以便从以下代码开始：

```cpp
int
main(int argc, char *argv[])
{
  uint32_t nPackets = 1;

  CommandLine cmd;
  cmd.AddValue("nPackets", "Number of packets to echo", nPackets);
  cmd.Parse(argc, argv);

  ...
```

在脚本中找到设置 `MaxPackets` 属性的位置，并更改为将其设置为变量 `nPackets` 而不是常数1，如下所示：

```cpp
echoClient.SetAttribute("MaxPackets", UintegerValue(nPackets));
```

现在，如果运行脚本并提供 `--PrintHelp` 参数，您应该在帮助显示中看到新的用户参数。

尝试一下：

```bash
$ ./ns3 build
$ ./ns3 run "scratch/myfirst --PrintHelp"
```

如果要指定要回显的数据包数，现在可以通过在命令行中设置 `--nPackets` 参数来执行。

```bash
$ ./ns3 run "scratch/myfirst --nPackets=2"
```

您现在应该看到已经回显了两个数据包。相当简单，不是吗？

可以看出，如果您是 ns-3 ==用户==，您==可以使用命令行参数系统来控制全局值和属性==。如果您是==模型作者==，您可以==向对象添加新属性==，它们将==自动可通过命令行系统设置给您的用户==。如果您是脚本作者，您可以向脚本添加新变量并将其轻松连接到命令行系统。

## 6.3. Using the Tracing System
模拟的整个目的是生成输出以供进一步研究，而 ns-3 跟踪系统是实现此目的的主要机制之一。由于 ns-3 是一个 C++ 程序，因此可以使用从 C++ 程序生成输出的标准工具：

```cpp
#include <iostream>
// ...
int main()
{
  // ...
  std::cout << "The value of x is " << x << std::endl;
  // ...
}
```

您甚至可以使用日志模块为解决方案添加一些结构。由于这种方法产生了许多众所周知的问题，因此我们提供了一个通用的事件跟踪子系统来解决我们认为重要的问题。

ns-3 跟踪系统的基本目标是：

- 对于基本任务，跟踪系统应允许用户为流行的跟踪源生成标准跟踪，并自定义生成跟踪的对象；
- 中级用户必须能够扩展跟踪系统以修改生成的输出格式，或插入新的跟踪源，而无需修改模拟器的核心；
- 高级用户可以修改模拟器核心以添加新的跟踪源和接收器。

ns-3 跟踪系统建立在独立跟踪源和跟踪接收器的概念以及将源连接到接收器的统一机制上。跟踪源是可以发出模拟中发生的事件并提供对底层数据的有趣访问的实体。例如，跟踪源可以指示数据包何时被网络设备接收，并为感兴趣的跟踪接收器提供对数据包内容的访问。

跟踪源本身并不有用，它们必须与实际使用由接收器提供的信息的代码的其他部分“连接”起来。跟踪接收器是跟踪源提供的事件和数据的消费者。例如，可以创建一个跟踪接收器，当连接到前面示例的跟踪源时，它会打印出接收到的数据包的有趣部分。

这种明确的划分的理由是允许用户将新类型的接收器附加到现有的跟踪源，而无需编辑和重新编译模拟器的核心。因此，在上述示例中，用户可以仅通过编辑用户脚本来定义新的跟踪接收器，并将其连接到模拟器中定义的现有跟踪源。

在本教程中，我们将介绍一些预定义的源和接收器，并展示如何通过少量用户工作自定义它们。有关高级跟踪配置的信息，请参阅 ns-3 手册或 how-to 部分，其中包括扩展跟踪命名空间和创建新跟踪源的内容。
#### 6.3.1. ASCII Tracing（解析 ASCII 跟踪）
ns-3提供了==包装低级跟踪系统的辅助功能==，以帮助您处理配置一些易于理解的数据包跟踪的详细信息。
==如果启用此功能，您将在ASCII文件中看到输出==，因此得名。对于熟悉ns-2输出的人来说，这种类型的跟踪类似于许多脚本生成的 out.tr。

让我们直接开始，向我们的 scratch/myfirst.cc 脚本的 Simulator::Run() 调用之前添加以下代码：

```cpp
AsciiTraceHelper ascii;
pointToPoint.EnableAsciiAll(ascii.CreateFileStream("myfirst.tr"));
```

- 与许多其他ns-3习语一样，此==代码使用辅助对象来帮助创建ASCII跟踪==。
- 第二行包含两个嵌套的方法调用。"内部"方法 CreateFileStream() 使用未命名对象惯用法在堆栈上创建一个文件流对象（没有对象名称）并将其传递给被调用的方法。我们将在将来详细介绍这一点，但在此时，您只需要知道==您正在创建一个代表名为 "myfirst.tr" 的文件的对象==，==并将其传递给ns-3==。您正在告诉ns-3处理已创建对象的生命周期问题，并处理与C++ ofstream对象的复制构造函数相关的一个小众（故意的）限制引起的问题。
- 外部调用 EnableAsciiAll() ==告诉助手您要在模拟中的所有点对点设备上启用ASCII跟踪==；并==且您希望（提供的）跟踪接收器写出有关数据包移动的信息，以ASCII格式==。

对于熟悉ns-2的人，被跟踪的事件等同于记录“+”、“-”、“d”和“r”事件的流行跟踪点。

现在，您可以构建脚本并从命令行运行它：

```bash
$ ./ns3 run scratch/myfirst
```

与之前看到的许多次一样，您将看到来自ns3的一些消息，然后是带有一些来自运行程序的消息的 "'build' finished successfully"。

当==运行时，程序将创建一个名为 myfirst.tr 的文件==。由于==ns3的工作方式，该文件不会在本地目录中创建==，而是==默认情况下在存储库的顶层目录中创建==。**如果要控制跟踪保存的位置，可以使用ns3的 --cwd 选项来指定**。我们没有这样做，因此我们需要切换到存储库的顶级目录并在您喜欢的编辑器中查看ASCII跟踪文件 myfirst.tr。
##### 6.3.1.1. Parsing Ascii Traces
这个文件中有很多以相当密集的形式呈现的信息，但首先要注意的是文件中有许多不同的行。除非您大幅度调整窗口的宽度，否则可能很难清楚地看到这一点。

文件中的==每一行对应一个跟踪事件==。

在这种情况下，我们正在跟踪模拟中每个点对点网络设备上的传输队列上发生的事件。传输队列是每个发往点对点通道的数据包都必须经过的队列。请注意，跟踪文件中的每一行都以单个字符（其后有一个空格）开头。此字符将具有以下含义：

- `+`：设备队列上发生了入队操作；
- `-`：设备队列上发生了出队操作；
- `d`：数据包被丢弃，通常是因为队列已满；（discard）
- `r`：数据包被网络设备接收。（receive）

让我们更详细地查看跟踪文件中的第一行。我将其分解为各个部分（为清晰起见缩进），左侧有一个参考编号：

```plaintext
+
2
/NodeList/0/DeviceList/0/$ns3::PointToPointNetDevice/TxQueue/Enqueue
ns3::PppHeader (
  Point-to-Point Protocol: IP (0x0021))
  ns3::Ipv4Header (
    tos 0x0 ttl 64 id 0 protocol 17 offset 0 flags [none]
    length: 1052 10.1.1.1 > 10.1.1.2)
    ns3::UdpHeader (
      length: 1032 49153 > 9)
      Payload (size=1024)
```

1. 这个扩展的跟踪事件的第一部分（参考编号0）是操作。我们==有一个 `+` 字符==，因此这对应于==在传输队列上进行的入队==操作。
2. 第二部分（参考1）是==以秒为单位表示的模拟时间==。您可能还记得我们要求 UdpEchoClientApplication 在两秒时开始发送数据包。在这里，我们看到确实发生了这种情况。
3. 下一部分（参考2）告诉我们==哪个跟踪源产生了此事件==（以跟踪命名空间表示）。您可以将跟踪命名空间看作是文件系统命名空间。命名空间的根是 ==NodeList。这对应于在 ns-3 核心代码中管理的容器==，其中包含在脚本中创建的所有节点。就像文件系统在根下有目录一样，NodeList 中可能有节点编号。因此，字符串==`/NodeList/0` 是指 NodeList 中的零节点==，我们通常认为是“节点 0”。在==每个节点中，有一个已安装设备的列表==。此列表==随后出现在命名空间中==。您可以看到此跟踪事件来自 DeviceList/0，这是节点中安装的第零个设备。
4. 接下来的==字符串 `$ns3::PointToPointNetDevice`== 告诉您==节点零的设备列表中第零位置的设备是什么类型的设备==。回顾一下，参考0处的 `+` 操作意味着在设备的传输队列上发生了入队操作。这反映在“跟踪路径”的最终部分中，即 TxQueue/Enqueue。
5. 跟踪中的其余部分应该相当直观。参考3-4指示数据包封装在点对点协议中。
6. 参考5-7显示数据包具有 IP 版本四 (IPv4协议) 头部，并源自 IP 地址 10.1.1.1，目标是 10.1.1.2。
7. 参考8-9显示此数据包具有 UDP 头部，最后，参考10显示有效载荷为预期的 1024 字节。

跟踪文件中的下一行显示相同的数据包从相同节点的传输队列中出队。
跟踪文件中的第三行显示了该数据包被回显服务器节点上的网络设备接收。我在下面重新复制了该事件。

```plaintext
r
2.25732
/NodeList/1/DeviceList/0/$ns3::PointToPointNetDevice/MacRx
  ns3::Ipv4Header (
    tos 0x0 ttl 64 id 0 protocol 17 offset 0 flags [none]
    length: 1052 10.1.1.1 > 10.1.1.2)
    ns3::UdpHeader (
      length: 1032 49153 > 9)
      Payload (size=1024)
```

请注意，跟踪操作现在是 `r`，模拟时间已增加到 2.25732 秒。如果您一直密切关注教程步骤，这意味着您已将网络设备和信道的 DataRate 保持为其默认值。这个时间应该是熟悉的，因为您在前面的部分中已经看到过它。

跟踪源命名空间条目（参考02）已更改，以反映此事件来自节点 1（/NodeList/1）和数据包接收跟踪源（/MacRx）。通过查看文件中的其余跟踪，您应该很容易追踪数据包通过拓扑的进展。
#### 6.3.2 PCAP Tracing
##### ps：PCAP 简介
>PCAP Tracing是指使用==`.pcap`（packet capture）文件格式==进行跟踪的一种技术。PCAP文件是==网络数据包捕获文件的标准格式==，通常用于存储网络通信的详细信息。这种跟踪技术的**目的是***记录网络中发送和接收的每个数据包*，以便后续分析和调试。
主要特征和用途包括：
>- **捕获网络流量：** PCAP Tracing通过捕获网络接口上的数据包来记录通信。这对于了解网络中发生的事务以及调查问题非常有用。
>- **协议分析：** PCAP文件可以通过各种网络分析工具进行解析，以深入了解网络通信中所使用的协议和数据。
>-  **网络故障排除：** 当网络发生问题时，PCAP Tracing允许工程师在捕获的数据包中查找问题的根本原因。通过分析通信模式和检查数据包的内容，可以更容易地诊断和解决网络故障。
>-  **安全审计：** 安全专业人员可以使用PCAP Tracing来监视网络流量，检测潜在的安全威胁和攻击。这对于进行入侵检测和网络安全审计非常重要。
>-  **网络性能优化：** 通过分析PCAP文件，网络管理员可以评估网络性能、识别瓶颈并进行优化。这对于确保网络高效运行至关重要。
>
在ns-3（网络仿真工具）中，PCAP Tracing通常用于记录仿真中的数据包传输，允许用户在仿真运行后分析模拟的网络通信。通过启用PCAP Tracing，用户可以生成以`.pcap`格式保存的跟踪文件，然后使用各种工具进行分析。

ns-3设备助手还==可用于创建.pcap格式的跟踪文件==。 pcap（通常以小写字母写入）的首字母缩写代表数据包捕获，实际上是包含.pcap文件格式定义的API。最流行的可以读取和显示此格式的程序是Wireshark（以前称为Ethereal）。然而，有许多使用此数据包格式的流量跟踪分析器。我们鼓励用户利用许多用于分析pcap跟踪的工具。在==本教程中，我们集中在使用tcpdump查看pcap跟踪==。

启用pcap跟踪的代码只有一行。

```cpp
pointToPoint.EnablePcapAll("myfirst");
```

在刚刚添加到scratch/myfirst.cc的ASCII跟踪代码之后，插入这行代码。**请注意，我们只传递了字符串“myfirst”，而没有传递类似于“myfirst.pcap”之类的东西**。这是==因为该参数是一个前缀，而不是一个完整的文件名==。助手实际上将为模拟中的每个点对点设备创建一个跟踪文件。文件名将使用前缀，节点编号，设备编号和“.pcap”后缀构建。

在我们的示例脚本中，我们最终将看到名为“myfirst-0-0.pcap”和“myfirst-1-0.pcap”的文件，分别是节点0-设备0和节点1-设备0的pcap跟踪。

添加**启用pcap跟踪的代码行后**，您可以以通常的方式运行脚本：

```bash
$ ./ns3 run scratch/myfirst
```

如果查看分发目录的顶层目录，您现在应该看到三个日志文件：myfirst.tr是我们先前检查过的ASCII跟踪文件。myfirst-0-0.pcap和myfirst-1-0.pcap是我们刚刚生成的新pcap文件。
##### 6.3.2.1. 使用tcpdump阅读输出

此时最容易做的事情就是==使用tcpdump查看pcap文件==。
###### 1. 键入下式，查看myfirst-0-0.pcap
```bash
$ tcpdump -nn -tt -r myfirst-0-0.pcap
```

从文件myfirst-0-0.pcap读取，==链路类型为PPP==（PPP）
```plaintext
2.000000 IP 10.1.1.1.49153 > 10.1.1.2.9: UDP, length 1024
2.514648 IP 10.1.1.2.9 > 10.1.1.1.49153: UDP, length 1024
```

比如，在我的电脑上显示：
```shell
huluobo@ubuntu:/Users/huluobo/Desktop/workspace/ns-3-allinone/ns-3.37$ tcpdump -nn -tt -r myfirst-0-0.pcap
reading from file myfirst-0-0.pcap, link-type PPP (PPP), snapshot length 65535
2.000000 IP 10.1.1.1.49153 > 10.1.1.2.9: UDP, length 1024
2.514648 IP 10.1.1.2.9 > 10.1.1.1.49153: UDP, length 1024
huluobo@ubuntu:/Users/huluobo/Desktop/workspace/ns-3-allinone/ns-3.37$ 
```
###### 2. 键入下式，查看myfirst-1-0.pcap
```bash
$ tcpdump -nn -tt -r myfirst-1-0.pcap
```

从文件myfirst-1-0.pcap读取，链路类型为PPP（PPP）
```plaintext
2.257324 IP 10.1.1.1.49153 > 10.1.1.2.9: UDP, length 1024
2.257324 IP 10.1.1.2.9 > 10.1.1.1.49153: UDP, length 1024
```

比如，在我的电脑上显示：
```shell
huluobo@ubuntu:/Users/huluobo/Desktop/workspace/ns-3-allinone/ns-3.37$ tcpdump -nn -tt -r myfirst-1-0.pcap
reading from file myfirst-1-0.pcap, link-type PPP (PPP), snapshot length 65535
2.257324 IP 10.1.1.1.49153 > 10.1.1.2.9: UDP, length 1024
2.257324 IP 10.1.1.2.9 > 10.1.1.1.49153: UDP, length 1024
huluobo@ubuntu:/Users/huluobo/Desktop/workspace/ns-3-allinone/ns-3.37$
```

>Corollary:
您可以在myfirst-0-0.pcap的转储中看到（客户端设备）在模拟中的2秒时发送的回显数据包。(Client在t=2时发送数据包)
如果查看第二个转储（myfirst-1-0.pcap），您将看到该数据包在2.257324秒时被接收。（Server在t=2.257324时接收到该数据包）
您可以在第二个转储中看到在2.257324秒时回显该数据包。（Server在t=2.257324时回传该数据包）
最后，在第一个转储中在2.514648秒时再次接收该数据包。（Client在t=2.514648时收到Server回传到数据包）
##### 6.3.2.2. 使用Wireshark阅读输出

如果您不熟悉Wireshark，可以从以下网站下载程序和文档：http://www.wireshark.org/。

Wireshark是一个图形用户界面，可用于显示这些跟踪文件。如果您有Wireshark，可以打开每个跟踪文件，并像使用数据包嗅探器捕获数据包一样显示其内容。
>我个人目前不准备直接上图形化界面，后面再看吧