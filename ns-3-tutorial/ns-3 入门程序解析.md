### （0）A First ns-3 Script
如果您按照上面的建议下载了ns-3系统，那么您的主目录下将有一个名为"workspace"的目录，里面包含ns-3的一个发布版本。切换到该发布目录，您应该会找到以下类似的目录结构：

```shell
AUTHORS        CMakeLists.txt   examples   RELEASE_NOTES.md  testpy.supp
bindings       contrib          LICENSE    scratch           utils
build-support  CONTRIBUTING.md  ns3        src               utils.py
CHANGES.md     doc              README.md  test.py           VERSION
```

切换到`examples/tutorial`目录。您应该会看到一个名为`first.cc`的文件位于其中。这是一个脚本，将在两个节点之间创建一个简单的点对点链接，并在这两个节点之间回显单个数据包。让我们逐行查看这个脚本，打开您喜欢的编辑器并编辑`first.cc`。
### （1）Copyright
ns-3模拟器采用GNU通用公共许可证第2版进行许可。在ns-3分发包的每个文件开头，你都会看到相应的GNU法律术语。通常，你会在GPL文本上方看到涉及ns-3项目的机构之一的版权声明，并在下方列出作者。

```cpp
/*
 * 本程序是自由软件；您可以根据GNU通用公共许可证第2版的条款
 * 进行再发布或修改。
 *
 * 本程序是基于希望它是有用的出版的；
 * 但没有任何担保；甚至没有对适销性或特定用途的隐含担保。
 * 有关更多详细信息，请参阅GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的一份副本；
 * 如果没有，请写信给Free Software Foundation，Inc.，
 * 地址：59 Temple Place，Suite 330，Boston，MA  02111-1307 USA
 */
```
### （2）Module Includes
代码正文以一系列的包含语句开始：

```cpp
#include "ns3/core-module.h"
#include "ns3/network-module.h"
#include "ns3/internet-module.h"
#include "ns3/point-to-point-module.h"
#include "ns3/applications-module.h"
```

为了帮助我们的高级脚本用户处理系统中存在的大量包含文件，我们根据相对较大的模块对包含进行分组。我们提供了一个单一的包含文件，它将递归加载每个模块中使用的所有包含文件。与必须==查找确切需要的头文件==并可能需要正确获取多个依赖项的方式不同，我们为您提供了以较大的颗粒度加载一组文件的能力。这并不是最高效的方法，但它确实使编写脚本变得更容易。

在构建过程中，将每个ns-3包含文件放置在一个名为ns3的目录中（在构建目录下），以避免包含文件名冲突。ns3/core-module.h文件对应于在下载的发布分发版本的src/core目录中找到的ns-3模块。如果列出此目录，将找到大量的头文件。进行构建时，ns3将在适当的build/debug或build/optimized目录下的ns3目录中放置公共头文件。CMake还会自动生成一个模块包含文件，以加载所有公共头文件。

由于你当然是在如宗教一般地遵循本教程，你已经从顶级目录运行了以下命令：

```bash
$ ./ns3 configure -d debug --enable-examples --enable-tests
```

以配置项目以执行包含示例和测试的调试构建。你还将调用

```bash
$ ./ns3 build
```

来构建项目。现在，如果你查看目录`../../build/include/ns3`，你将找到上述四个模块包含文件（还有许多其他头文件）。你可以查看这些文件的内容，并发现它们确实包含了各自模块中的所有公共包含文件。
### （3）Ns3 Namespace
在`first.cc`脚本中的下一行是一个命名空间声明：

```cpp
using namespace ns3;
```

ns-3项目是在一个名为`ns3`的C++命名空间中实现的。这将所有与ns-3相关的声明组合到全局命名空间之外的一个作用域中，我们希望这有助于与其他代码的集成。C++的`using`语句将ns-3命名空间引入当前（全局）声明区域。这是一种简洁的说法，即在此声明之后，你将无需在使用ns-3代码之前键入`ns3::`范围解析运算符。如果你对命名空间不熟悉，请参考几乎任何C++教程，并将这里的`ns3`命名空间和使用与讨论`cout`和流时经常遇到的`std`命名空间以及`using namespace std;`语句进行比较。
### （4）Logging
脚本的下一行是以下内容：

```cpp
NS_LOG_COMPONENT_DEFINE("FirstScriptExample");
```

我们将使用这个语句作为一个便利的地方来讨论我们的Doxygen文档系统。如果你查看项目网站，ns-3项目，你会在导航栏中找到一个名为“Documentation”的链接。如果选择此链接，将会跳转到我们的文档页面。有一个指向“Latest Release”的链接，该链接将带你到ns-3最新稳定版本的文档。如果选择“API Documentation”链接，将跳转到ns-3 API文档页面。

在左侧，你会找到文档结构的图形表示。一个好的开始地方是ns-3导航树中的NS-3模块“book”。如果展开“Modules”，你将看到ns-3模块文档的列表。这里的模块概念直接与上面讨论的模块包含文件相关。ns-3日志子系统在“Using the Logging Module”部分讨论，所以我们稍后在本教程中会涉及到它，但你可以通过查看“Core module”，然后展开“Debugging tools book”，再选择“Logging page”来了解上述语句。点击Logging。

你现在应该看到Logging模块的Doxygen文档。在页面顶部的宏列表中，你将看到NS_LOG_COMPONENT_DEFINE的条目。在深入研究之前，最好找一下日志模块的“Detailed Description”以了解整体操作感觉。你可以向下滚动，或者选择协作图下的“More…”链接来做到这一点。

一旦你对正在发生的事情有一个大致的了解，那么可以继续查看具体的NS_LOG_COMPONENT_DEFINE文档。我不会在这里重复文档内容，但总结一下，这行代码声明了一个名为FirstScriptExample的日志组件，允许你通过引用该名称启用和禁用控制台消息日志。
### （5）Main Function
脚本的下面几行是：

```cpp
int main(int argc, char *argv[])
{
```

这只是你程序（脚本）主函数的声明。与任何C++程序一样，你需要定义一个main函数，它将是首个运行的函数。这里没有任何特殊之处。你的ns-3脚本只是一个C++程序。

下一行将时间分辨率设置为一纳秒，这恰好是默认值：

```cpp
Time::SetResolution(Time::NS);
```

分辨率是可以表示的最小时间值（以及两个时间值之间的最小可表示差异）。你只能更改分辨率一次。这种灵活性的机制在某种程度上对内存消耗较大，因此一旦分辨率已经被显式设置，我们释放内存，防止进一步更新。（如果你不显式设置分辨率，它将默认为一纳秒，并且内存将在模拟开始时被释放。）

脚本的下两行用于启用内置于Echo Client和Echo Server应用程序中的两个日志组件：

```cpp
LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO);
```

如果你阅读了Logging组件文档，你将了解到每个组件上可以启用多个日志详细程度/冗余度级别。这两行代码启用了INFO级别的调试日志，用于echo客户端和服务器。这将导致应用程序在模拟过程中打印出发送和接收数据包的消息。

现在我们将直接着手创建拓扑并运行模拟。我们使用拓扑帮助对象使这项工作尽可能简单。
### （6）Topology Helpers
#### 1. NodeContainer
代码脚本中的下两行实际上将**创建在模拟中表示计算机的ns-3 Node对象**。

```cpp
NodeContainer nodes; // 声明一个NodeContainer类的对象
nodes.Create(2);     // 调用该对象的create方法 => 产生2个节点
```

在继续之前，让我们找到NodeContainer类的文档。另一种查看给定类文档的方法是通过Doxygen页面中的Classes选项卡。如果你仍然可以访问Doxygen，请向页面顶部滚动并选择Classes选项卡。你应该会看到一组新的选项卡出现，其中一个是Class List。在该选项卡下，你将看到所有ns-3类的列表。向下滚动，查找`ns3::NodeContainer`。当找到该类时，请选择它以查看该类的文档。

你可能还记得我们的一个关键抽象是Node。这代表我们将要向其添加协议栈、应用程序和外围设备卡的计算机。NodeContainer拓扑助手提供了一种方便的方式来创建、管理和访问我们为运行模拟而创建的任何Node对象。

**上面的第一行只是声明了一个名为`nodes`的NodeContainer**。**第二行在`nodes`对象上调用了`Create`方法，并要求容器创建两个节点**。如Doxygen中所述，该容器向ns-3系统适当地调用以创建两个Node对象，并在内部存储对这些对象的指针。

在脚本中，==这些节点目前并没有做任何事情==。构建拓扑的==下一步是将我们的节点连接到一个网络中==。我们支持的最简单形式的网络是两个节点之间的单点对点链接。我们将在这里构建其中一个链接。

#### 2. PointToPointHelper
我们正在构建一条点对点链接，而按照你将会变得非常熟悉的模式，我们使用拓扑助手对象来执行组装链接所需的低级工作。回顾一下，我们的两个关键抽象是NetDevice和Channel。在现实世界中，这些术语大致对应于外围设备卡和网络电缆。通常这两者是密切相关的，不能期望例如以太网设备和无线信道之间可以互换。

我们的拓扑助手遵循这种密切的耦合，因此在这个脚本中，你将使用一个PointToPointHelper来**配置和连接**ns-3的*PointToPointNetDevice*和*PointToPointChannel*对象。

脚本中的下面三行是：

```cpp
PointToPointHelper pointToPoint;
pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));
```

第一行，

```cpp
PointToPointHelper pointToPoint;
```

在堆栈上实例化了一个PointToPointHelper对象。从高层次的角度来看，下一行，

```cpp
pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
```

告诉PointToPointHelper对象，在创建PointToPointNetDevice对象时，使用值“5Mbps”（每秒五兆位）作为“DataRate”。

从更详细的角度来看，字符串“DataRate”对应于我们称为PointToPointNetDevice的属性。如果查看类ns3::PointToPointNetDevice的Doxygen并找到GetTypeId方法的文档，你将找到为该设备定义的属性列表。其中之一是“DataRate”属性。大多数用户可见的ns-3对象都有类似的属性列表。正如你将在接下来的部分中看到的那样，我们使用这种机制可以轻松地配置模拟而无需重新编译。

与PointToPointNetDevice上的“DataRate”类似，你将在PointToPointChannel上找到与之关联的“Delay”属性。最后一行，

```cpp
pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));
```

告诉PointToPointHelper在随后创建的每个点对点信道中使用值“2ms”（两毫秒）作为传播延迟的值。
#### 3. NetDeviceContainer
截至目前：
1. 在脚本的这一点上，我们有一个包含两个节点的NodeContainer✅。
2. 我们有一个PointToPointHelper，它已经准备好创建PointToPointNetDevices并在它们之间连接PointToPointChannel对象✅。

就像我们使用NodeContainer拓扑助手对象为模拟创建节点一样，我们将==要求PointToPointHelper执行涉及创建、配置和安装设备的工作==。我们需要有一个包含所有创建的NetDevice对象的列表，因此我们使用NetDeviceContainer来保存它们，就像我们使用NodeContainer来保存我们创建的节点一样。下面的两行代码，

```cpp
NetDeviceContainer devices;
devices = pointToPoint.Install(nodes);
```

将完成设备和信道的配置。
第一行声明了上面提到的设备容器，而第二行则完成了繁重的工作。
PointToPointHelper的Install方法以NodeContainer作为参数。在内部，将创建一个NetDeviceContainer。对于NodeContainer中的每个节点（对于点对点链接，必须恰好有两个节点），将创建一个PointToPointNetDevice并将其保存在设备容器中。将创建一个PointToPointChannel，并连接两个PointToPointNetDevice。当由PointToPointHelper创建对象时，帮助器中先前设置的属性将用于初始化所创建对象中的相应属性。

执行pointToPoint.Install(nodes)调用后，我们将有两个节点，每个节点都安装有一个点对点网络设备，它们之间有一个单独的点对点信道。这两个设备都将配置为在通道上以每秒五兆位的速度传输数据，该通道具有两毫秒的传输延迟。
#### 4. InternetStackHelper
现在我们已经配置了节点和设备，但是我们的节点上还没有安装任何协议栈。接下来的两行代码将处理这个问题。

```cpp
InternetStackHelper stack;
stack.Install(nodes);
```

InternetStackHelper是一个拓扑助手，对于Internet协议栈而言，它的作用类似于PointToPointHelper对于点对点网络设备。Install方法以NodeContainer作为参数。当执行时，它==将在节点容器中的每个节点上==安装一个Internet协议栈（TCP、UDP、IP等）。
#### 5. Ipv4AddressHelper
接下来，我们需要将节点上的设备与IP地址关联起来。我们提供了一个拓扑助手来管理IP地址的分配。用户可见的唯一API是在执行实际地址分配时设置要使用的基本IP地址和网络掩码（这是在助手内部的较低级别上完成的）。

在我们的示例脚本first.cc中的下面两行代码，

```cpp
Ipv4AddressHelper address;
address.SetBase("10.1.1.0", "255.255.255.0");
```

声明一个地址助手对象，并告诉它应该从网络10.1.1.0开始使用掩码255.255.255.0来定义可分配的位。

**默认情况下，分配的地址将从1开始单调增加**，因此从此基础分配的第一个地址将是10.1.1.1，然后是10.1.1.2等。*ns-3的底层系统实际上会记住所有分配的IP地址，并且如果你不小心导致生成相同的地址两次，它将生成致命错误*（这是一个非常难以调试的错误，顺便说一下）。

下一行代码，

```cpp
Ipv4InterfaceContainer interfaces = address.Assign(devices);
```

执行实际的地址分配。在ns-3中，我们使用Ipv4Interface对象将IP地址与设备关联起来。就像我们有时需要一个由助手创建的未来参考的网卡设备列表一样，有时我们需要一个Ipv4Interface对象列表。Ipv4InterfaceContainer提供了这个功能。

现在我们已经建立了一个点对点网络，安装了协议栈并分配了IP地址。在这一点上，我们需要的是产生流量的应用程序。
### （7）Application
ns-3系统的另一个核心抽象是Application。在这个脚本中，我们使用了ns-3核心类Application的两个专业化类，分别是UdpEcho==Server==Application和UdpEcho==Client==Application。就像我们在之前的解释中所做的那样，我们使用辅助对象来帮助配置和管理底层对象。在这里，我们使用UdpEchoServerHelper和UdpEchoClientHelper对象来使我们的生活更加轻松。
#### 1. UdpEchoServerHelper
在我们的示例脚本first.cc中，以下代码用于**在我们之前创建的节点之一上设置UDP回显服务器应用程序**。

```cpp
UdpEchoServerHelper echoServer(9);

ApplicationContainer serverApps = echoServer.Install(nodes.Get(1));
serverApps.Start(Seconds(1.0));
serverApps.Stop(Seconds(10.0));
```

上面代码片段中的第一行声明了UdpEchoServerHelper。和往常一样，这不是应用程序本身，而是一个帮助我们创建实际应用程序的对象。我们的一个惯例是将所需的属性放在助手构造函数中。在这种情况下，==除非提供了客户端也知道的端口号，否则助手无法执行任何有用的操作==。我们要求==在构造函数的参数中提供端口号，而构造函数则简单地使用传递的值进行SetAttribute==。如果愿意，可以稍后使用SetAttribute将“Port”属性设置为另一个值。

与许多其他帮助对象类似，UdpEchoServerHelper对象具有Install方法。实际上，正是这个方法的执行导致底层的回显服务器应用程序被实例化并附加到一个节点。有趣的是，Install方法接受一个NodeContainer作为参数，就像我们看到的其他Install方法一样。尽管在这种情况下看起来不是这样，但实际上是将nodes.Get(1)的结果（返回一个指向节点对象的智能指针-Ptr\<Node>）进行了C++的隐式转换，然后使用该结果在传递给Install的无名NodeContainer的构造函数中。如果你在C++代码中找不到特定方法签名而它却能够编译和运行正常，可以寻找这种隐式转换。

我们现在看到，echoServer.Install==将在我们用于管理节点的NodeContainer中找到的索引号为1的节点上==安装一个UdpEchoServerApplication。Install将返回一个容器，其中包含由助手创建的所有应用程序的指针（在本例中为一个，因为我们传递了包含一个节点的NodeContainer）。

==应用程序需要一个时间来“开始”生成流量，并且可能需要一个可选的时间来“停止”==。我们都提供了。这些时间是使用ApplicationContainer的Start和Stop方法设置的。这些方法采用Time参数。在这种情况下，我们使用一个==显式的C++转换序列，将C++的double 1.0转换为ns-3的Time对象，使用Seconds转换==。请注意，转换规则可能由模型作者控制，并且C++有自己的规则，因此你不能总是假设参数会愉快地为你转换。这两行代码，

```cpp
serverApps.Start(Seconds(1.0));
serverApps.Stop(Seconds(10.0));
```

将导致回显服务器应用程序在模拟进行到一秒时启动（启用自身），并在模拟进行到十秒时停止（禁用自身）。由于我们声明了一个模拟事件（应用程序停止事件）将在十秒时执行，因此模拟将至少持续十秒。
#### 2. UdpEchoClientHelper
回显客户端应用程序的设置方法与服务器的方法基本相似。有一个由UdpEchoClientHelper管理的底层UdpEchoClientApplication。

```cpp
UdpEchoClientHelper echoClient(interfaces.GetAddress(1), 9);
echoClient.SetAttribute("MaxPackets", UintegerValue(1));
echoClient.SetAttribute("Interval", TimeValue(Seconds(1.0)));
echoClient.SetAttribute("PacketSize", UintegerValue(1024));

ApplicationContainer clientApps = echoClient.Install(nodes.Get(0));
clientApps.Start(Seconds(2.0));
clientApps.Stop(Seconds(10.0));
```

然而，对于回显客户端，我们需要设置五个不同的属性。前两个属性在UdpEchoClientHelper的构造过程中设置。我们传递参数（在助手内部使用）以根据我们的约定将所需属性设置为助手构造函数的参数。

回想一下，我们使用Ipv4InterfaceContainer来跟踪我们分配给设备的IP地址。接口容器中的零接口将对应于节点容器中的零节点分配的IP地址。接口容器中的第一个接口对应于节点容器中第一个节点的IP地址。
因此，在上述==代码的第一行==中，我们创建了助手，并==告诉它将客户端的远程地址设置为服务器所在节点分配的IP地址==。我们==还告诉它安排发送到端口九的数据包==。
**“MaxPackets”属性**告诉客户端我们**允许它在模拟期间发送的最大数据包数**。
**“Interval”属性**告诉客户端在数据包之间**等待多长时间**。
**“PacketSize”属性**告诉客户端其**数据包负载应该有多大** 。 通过这组特定的属性组合，我们告诉客户端发送一个大小为1024字节的数据包。

就像在回显服务器的情况下一样，我们告诉回显客户端启动和停止，但在这里我们在服务器启用后的一秒钟再启动客户端（在模拟中的两秒时）。
### （8）Simulator
在这一点上，我们需要实际运行模拟。这是通过全局函数`Simulator::Run`完成的。

```cpp
Simulator::Run();
```

当我们之前调用了方法：

```cpp
serverApps.Start(Seconds(1.0));
serverApps.Stop(Seconds(10.0));
// ...
clientApps.Start(Seconds(2.0));
clientApps.Stop(Seconds(10.0));
```

实际上，我们在模拟中安排了事件，分别在1.0秒、2.0秒和两个事件在10.0秒。
当调用`Simulator::Run`时，系统将开始查看已安排事件的列表并执行它们。首先，它将运行在1.0秒时安排的事件，这将启用回显服务器应用程序（此事件可能反过来安排许多其他事件）。然后，它将运行在t=2.0秒时安排的事件，这将启动回显客户端应用程序。同样，这个事件可能安排了许多更多的事件。回显客户端应用程序中的启动事件实现将通过向服务器发送数据包开始模拟的数据传输阶段。

将数据包发送到服务器的行为==将触发==一系列幕后**自动安排**的事件，这些事件将根据我们在脚本中设置的各种时间参数执行数据包回显的机制。

最终，由于我们只发送一个数据包（回想一下，MaxPackets属性被设置为一个），由单个客户端回显请求触发的事件链将逐渐减少，模拟将进入空闲状态。一旦这发生，剩下的事件将是服务器和客户端的停止事件。当执行这些事件时，没有进一步的事件要处理，`Simulator::Run`将返回。然后模拟就完成了。

现在剩下的就是清理工作。这是通过调用全局函数`Simulator::Destroy`完成的。作为辅助函数（或底层ns-3代码）执行时，它们安排了在模拟器中插入挂钩以销毁所有创建的对象。您不必自己跟踪这些对象 - 您只需要调用`Simulator::Destroy`并退出。ns-3系统为您处理了困难的部分。我们第一个ns-3脚本`first.cc`的剩余行就是做这个：

```cpp
  Simulator::Destroy();
  return 0;
}
```
#### 1. When the simulator will stop?
ns-3是一个==离散事件（DE）模拟器==。在这样的模拟器中，==每个事件都与其执行时间关联==，模拟通过按照模拟时间的时间顺序执行事件进行。事件可能会导致安排将来的事件（例如，定时器可能会重新安排自己以在下一个间隔到期）。

##### rule1: 初始事件通常由每个对象触发，例如IPv6将安排路由器通告、邻居请求等，一个应用程序安排第一个发送数据包的事件等。

当处理事件时，它可能生成零个、一个或多个事件。随着模拟的执行，事件被消耗，但可能随之（或可能不）生成更多事件。当事件队列中没有更多事件时，或者找到特殊的Stop事件时，模拟将自动停止。通过Simulator::Stop(stopTime);函数创建Stop事件。

Simulator::Stop在必须停止模拟的情况下是绝对必要的一种典型情况：==当存在自维持事件时。自维持（或周期性）事件是总是重新安排自己的事件。因此，它们始终保持事件队列非空。==

有许多包含周期性事件的协议和模块，例如：

- FlowMonitor - 定期检查丢失的数据包
- RIPng - 定期广播路由表更新
- 等等

在==这些情况下，Simulator::Stop优雅地停止模拟是必要的！==
##### rule2: 此外，当ns-3处于仿真模式时，RealtimeSimulator用于保持模拟时钟与机器时钟对齐，而Simulator::Stop对于停止进程是必要的。

教程中的许多模拟程序实际上不显式调用Simulator::Stop，因为事件队列将自动用尽。但是，这些程序也将接受对Simulator::Stop的调用。例如，以下在第一个示例程序中的其他语句将在11秒时安排一个显式的停止：

```cpp
+  Simulator::Stop(Seconds(11.0));
   Simulator::Run();
   Simulator::Destroy();
   return 0;
}
```

以上实际上不会改变此程序的行为，因为这个特定的模拟在10秒后自然结束。但是，如果你将上述语句中的停止时间从11秒更改为1秒，你会注意到模拟在屏幕上打印任何输出之前就停止了（因为输出发生在模拟时间的约2秒处）。

在调用Simulator::Run之前调用Simulator::Stop非常重要；否则，Simulator::Run可能永远不会将控制返回到主程序以执行停止！
### （9） Building Your Script
我们已经使得构建简单脚本变得非常容易。您只需将脚本放入scratch目录，如果运行ns3，它将自动构建。让我们试一试。在返回到顶级目录之后，将examples/tutorial/first.cc复制到scratch目录中。

```bash
$ cd ../..
$ cp examples/tutorial/first.cc scratch/myfirst.cc
```

现在使用ns3构建您的第一个示例脚本：

```bash
$ ./ns3 build
```

您应该会看到消息报告您的myfirst示例已成功构建。

扫描 scratch_myfirst 目标的依赖关系
[  0%] 正在构建 CXX 对象 scratch/CMakeFiles/scratch_myfirst.dir/myfirst.cc.o
[  0%] 正在链接 CXX 可执行文件 ../../build/scratch/ns3.36.1-myfirst-debug
已完成执行以下命令：
cd cmake-cache; cmake --build . -j 7 ; cd ..

现在可以运行该示例（请注意，==如果在scratch中构建程序==，则必须在scratch目录中==运行==它 ==[即：scratch的“外面”！]==）：

```bash
$ ./ns3 run scratch/myfirst
```

您应该会看到一些输出：

```
At time +2s client sent 1024 bytes to 10.1.1.2 port 9
At time +2.00369s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.00369s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.00737s client received 1024 bytes from 10.1.1.2 port 9
// 在时间 +2s 时，客户端将 1024 字节发送到 10.1.1.2 端口 9
// 在时间 +2.00369s 时，服务器从 10.1.1.1 端口 49153 接收到 1024 字节
// 在时间 +2.00369s 时，服务器将 1024 字节发送到 10.1.1.1 端口 49153
// 在时间 +2.00737s 时，客户端从 10.1.1.2 端口 9 接收到 1024 字节
```

在这里，您看到了Echo客户端的日志组件表示它已经向10.1.1.2的Echo服务器发送了一个1024字节的数据包。您还看到Echo服务器的日志组件说它已经从10.1.1.1接收到了1024字节。Echo服务器默默地回显数据包，您会看到Echo客户端记录它已经从服务器收到了自己的数据包。
### （10）Ns-3 Source Code
现在您已经使用了一些ns-3助手，可能想要查看一些实现该功能的源代码。

我们的示例脚本位于examples目录中。如果切换到examples目录，您将看到一个子目录列表。tutorial子目录中的一个文件是first.cc。如果点击first.cc，您将找到刚刚浏览的代码。

源代码主要位于src目录中。模拟器的核心位于src/core/model子目录中。在那里，您会找到第一个文件（截至撰写本文时）是abort.h。如果打开该文件，可以查看在检测到异常条件时退出脚本的宏。

我们在本章中使用的助手的源代码可以在src/applications/helper目录中找到。请随意在目录树中浏览，了解那里有什么以及ns-3程序的风格。
### （11）回顾整理
#### 示例源码：
```cpp
linux> huluobo@ubuntu:/Users/huluobo/Desktop/workspace/ns-3-allinone/ns-3.37/scratch$ cat myfirst.cc

/*
 * This program is free software; you can redistribute it and/or modify
 * it under the terms of the GNU General Public License version 2 as
 * published by the Free Software Foundation;
 *
 * This program is distributed in the hope that it will be useful,
 * but WITHOUT ANY WARRANTY; without even the implied warranty of
 * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
 * GNU General Public License for more details.
 *
 * You should have received a copy of the GNU General Public License
 * along with this program; if not, write to the Free Software
 * Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA
 */ 
 //这一段是废话！
 
#include "ns3/applications-module.h"
#include "ns3/core-module.h"
#include "ns3/internet-module.h"
#include "ns3/network-module.h"
#include "ns3/point-to-point-module.h" 
//引入头文件而已，也是废话！

// Default Network Topology
//
//       10.1.1.0
// n0 -------------- n1
//    point-to-point
//
//形象化展示本段代码最终达成的效果，直观化，看个乐呵就行

using namespace ns3;

NS_LOG_COMPONENT_DEFINE("FirstScriptExample"); // 没啥用

int
main(int argc, char* argv[])
{
    CommandLine cmd(__FILE__);
    cmd.Parse(argc, argv);

    Time::SetResolution(Time::NS);
    LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
    LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO);
    // 上述这些是“形式化语言”，记住格式就行，目前不需要在意！
    
    //（1）建立节点：
    NodeContainer nodes; // 建立一个节点容器，called：nodes
    nodes.Create(2);     // 建立的这个节点容器内部放置两个节点



    //（2）将节点进行连接，设置“后面会用到的”节点属性：
    PointToPointHelper pointToPoint;                                   // 建立一个点对点的连接器，called：pointToPoint
    pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps")); // 使用值“5Mbps”（每秒五兆位）作为“DataRate”
    pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));     // 使用值“2ms”作为“延迟时间”



    //（3）建立网络设备，让节点具有网络意义：
    NetDeviceContainer devices;            // 建立一个网络设备容器 called：devices
    devices = pointToPoint.Install(nodes); 
/*
第二行则完成了繁重的工作：
1. 【自动完成】对于前面设置完的“节点容器”内的每个节点（对于点对点链接，必须恰好有两个节点），将创建一个PointToPointNetDevice并将其保存在网络设备容器中
2. 【自动完成】将创建一个PointToPointChannel，并连接两个PointToPointNetDevice
3. 【自动按照前置的配置】当由PointToPointHelper创建对象时，帮助器“先前设置的属性”将用于初始化所创建对象中的相应属性

即：执行pointToPoint.Install(nodes)调用后，我们将有两个节点，每个节点都安装有一个点对点网络设备，它们之间有一个单独的点对点信道。这两个设备都将配置为在通道上以每秒五兆位的速度传输数据，该通道具有两毫秒的传输延迟
*/



    //（4）网络设备基础已经完毕，现在该考虑吧交互了！那么首先要安排协议栈：
    InternetStackHelper stack; // 对于Internet协议栈而言，它的作用类似于PointToPointHelper对于点对点网络设备
    stack.Install(nodes);      // 将在节点容器内的每个节点上都安装一个Internet协议栈（TCP、UDP、IP等）



    //（5）需要将节点上的设备与IP地址关联，通信当然须知晓IP地址：
    Ipv4AddressHelper address;                    // 声明一个地址助手
    address.SetBase("10.1.1.0", "255.255.255.0"); // 告诉它应该从网络10.1.1.0开始使用掩码255.255.255.0来定义可分配的位

    Ipv4InterfaceContainer interfaces = address.Assign(devices); // 使用Ipv4Interface对象将IP地址与网络设备关联起来



    //（6）现在你会发现所有的硬件全部搭建好了！还差交流通货————流量：【传递在server与client之间】
    
    //【1】先配置服务器的信息（server）
    
    // step1: 建立助手
    UdpEchoServerHelper echoServer(9); // 1）建立一个创建应用程序的助手，设计它的端口号为9
    
    /*和之前的一样，这不是应用程序本身，而是一个帮助我们创建实际应用程序的对象！在这种情况下，除非提供了客户端也知道的端口号，否则助手无法执行任何有用的操作。我们要求在构造函数的参数中提供端口号，而构造函数则简单地使用传递的值进行SetAttribute*/
    
    // step2: 配置基本属性【这里服务器没有】

    // step3:安装到实体（节点容器中的对应节点上）
    ApplicationContainer serverApps = echoServer.Install(nodes.Get(1)); // 2）将在我们用于“节点容器”中找到索引号为1的节点，在它身上安装一个UdpEchoServerApplication

    // step4：设置开始与结束
    serverApps.Start(Seconds(1.0)); // 3）应用程序需要一个时间来“开始”生成流量，并且可能需要一个可选的时间来“停止”
    serverApps.Stop(Seconds(10.0)); // 这里的seconds时间就是App的“开/关”时刻【很显然duration是10-1】



    //【2】再配置客户端的信息（client）
    
    // step1: 建立助手
    UdpEchoClientHelper echoClient(interfaces.GetAddress(1), 9); // 1）建立一个创建客户端的助手，设计客户端的远程地址为服务器所在节点分配的IP地址（大地点确定）。我们设置将它安排发送到端口9的数据包（细化的小地点也确定）。

    // step2: 配置基本属性【这里服务器没有】
    echoClient.SetAttribute("MaxPackets", UintegerValue(1)); // 2）MaxPackets属性：告诉客户端，我们允许它在模拟期间发送的最大数据包数是UintegerValue(1)
    echoClient.SetAttribute("Interval", TimeValue(Seconds(1.0))); // 3）Interval”属性：告诉客户端在数据包之间等待多长时间
    echoClient.SetAttribute("PacketSize", UintegerValue(1024)); // 4）PacketSize”属性：告诉客户端，其数据包负载应该有多大

    // step3:安装到实体（节点容器中的对应节点上）
    ApplicationContainer clientApps = echoClient.Install(nodes.Get(0)); // 5）将在我们用于“节点容器”中找到索引号为0的节点，在它身上安装一个UdpEchoClientApplication

    // step4：设置开始与结束
    clientApps.Start(Seconds(2.0)); // 6）客户端需要一个时间来“开始”生成流量，并且可能需要一个可选的时间来“停止”
    clientApps.Stop(Seconds(10.0)); // 这里的seconds时间就是客户端的“开/关”时刻【很显然duration是10-2】



    //（7）所有东西已经全方位配置完毕了！现在可以开始模拟网络运行！
    Simulator::Run();     //开始运行！
    Simulator::Destroy(); //销毁所有创建的对象，类比c++内的delete / ...[]
    return 0;
}
// 仔细体会！这段代码体现的“类思想“非常浓厚，因为我们的一个惯例是：将所需的属性放在助手构造函数中！

linux> huluobo@ubuntu:/Users/huluobo/Desktop/workspace/ns-3-allinone/ns-3.37/scratch$
```

- 注意我之前配置的环境是在linux虚拟机上，因此记得进zsh后先orb一下，进入linux虚拟机
- 基本上，我们操作的“大本营”文件都是在ns-3.37文件夹内，在我的mac上绝对路径是：`/Users/huluobo/Desktop/workspace/ns-3-allinone/ns-3.37/`
- 示例程序是在`/Users/huluobo/Desktop/workspace/ns-3-allinone/ns-3.37/examples/tutorial`中
- 要是想调用：先将上述参考程序cp到scratch文件夹内，即：`/Users/huluobo/Desktop/workspace/ns-3-allinone/ns-3.37/scratch`