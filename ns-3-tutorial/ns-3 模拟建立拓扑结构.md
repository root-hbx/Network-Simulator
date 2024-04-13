### 7.1. Building a Bus Network Topology
在这一部分，我们将扩展对ns-3网络设备和通道的了解，介绍一种总线网络的示例。ns-3提供了一种我们称之为CSMA（Carrier Sense Multiple Access）的网络设备和通道。

ns-3 CSMA设备模拟了以太网的简单网络。==实际的以太网使用CSMA/CD（载波监听多路访问与冲突检测）方案==，通过指数增加的退避来争夺共享传输介质。而==ns-3 CSMA设备和通道仅模拟了这个方案的一部分==。

就像我们在构建点对点拓扑时看到的点对点拓扑帮助对象一样，在本节中，我们将看到等效的CSMA拓扑帮助对象（CSMA建立助手）。这些帮助器的外观和操作应该对你来说非常熟悉。

我们在examples/tutorial目录中提供了一个示例脚本。这个脚本基于first.cc脚本，向我们已经考虑过的点对点仿真中添加了CSMA网络。请打开你喜欢的编辑器并查看examples/tutorial/second.cc。你已经看到了足够的ns-3代码，以理解这个例子中发生的大部分事情，但我们将检查整个脚本并查看一些输出。

就像在first.cc示例中（以及所有ns-3示例中）一样，文件以emacs模式行和一些GPL样板开头。

实际的代码通过加载模块包含文件开始，就像在first.cc示例中所做的那样。

```cpp
#include "ns3/core-module.h"
#include "ns3/network-module.h"
#include "ns3/csma-module.h"
#include "ns3/internet-module.h"
#include "ns3/point-to-point-module.h"
#include "ns3/applications-module.h"
#include "ns3/ipv4-global-routing-helper.h"
```

令人惊讶但非常有用的一点是，有一小段ASCII艺术，显示了在示例中构建的网络拓扑的图形化示意（小卡通cartoon）。你会在我们的大多数示例中找到类似的“图”。

在这种情况下，你可以看到我们将扩展我们的==点对点示例==（下面的n0和n1之间的链接）通过==在右侧挂载一个总线网络==。请注意，这是默认的网络拓扑，因为你实际上可以改变在LAN上创建的节点的数量。如果将nCsma设置为1，则LAN（CSMA通道）上将有总共两个节点（CSMA设备）——一个必需的节点和一个“额外”的节点。默认情况下，有三个“额外”的节点，如下所示：

```plaintext
// 默认网络拓扑
//
//       10.1.1.0
// n0 -------------- n1   n2   n3   n4
//    点对点         |    |    |    |
//                    ================
//                      LAN 10.1.2.0
```

然后使用ns-3命名空间并定义一个日志记录组件。这与first.cc中的操作完全相同，因此还没有新东西。

```cpp
using namespace ns3;

NS_LOG_COMPONENT_DEFINE("SecondScriptExample");
```

主程序以稍微不同的方式开始。我们使用==verbose标志==来确定==是否启用UdpEchoClientApplication和UdpEchoServerApplication日志记录==组件。此标志默认为true（启用日志记录组件），但也允许我们在对该示例进行回归测试时关闭日志记录。

你将看到一些熟悉的代码，这将允许你==通过命令行参数更改CSMA网络上的设备数量。==我们在处理命令行参数的部分允许更改发送的数据包数量时已经做过类似的事情。最后一行确保你至少有一个“额外”的节点。

```cpp
bool verbose = true; 
uint32_t nCsma = 3;

CommandLine cmd;
cmd.AddValue("nCsma", "Number of \"extra\" CSMA nodes/devices", nCsma);
cmd.AddValue("verbose", "Tell echo applications to log if true", verbose);

cmd.Parse(argc, argv);

if (verbose)
{
  LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
  LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO);
}

nCsma = nCsma == 0 ? 1 : nCsma;
```

接下来的步骤是创建两个我们将通过点对点链接连接的节点。NodeContainer用于执行这个操作，就像在first.cc中所做的那样。

```cpp
NodeContainer p2pNodes;
p2pNodes.Create(2);
```

接下来，我们声明另一个NodeContainer来保存将成为总线（CSMA）网络一部分的节点。首先，我们只是实例化容器对象本身。

```cpp
NodeContainer csmaNodes;
csmaNodes.Add(p2pNodes.Get(1));
csmaNodes.Create(nCsma);
```

下一行代码从==点对点节点容器中获取第一个节点==（索引为1），并==将其添加到将获取CSMA设备的节点容器==。我们要讨论的节点最终将具有点对点设备和CSMA设备。然后，我们创建组成CSMA网络其余部分的“额外”节点的数量。**由于我们已经有一个节点在CSMA网络中**——即**将同时具有点对点和CSMA网络设备的节点**，==所以“额外”节点的数量表示你在CSMA部分中需要的节点数减去1==。

下一部分的代码到目前为止应该非常熟悉。我们==实例化了一个PointToPointHelper==，并设置了关联的默认属性，以便使用该助手创建的设备上有一个==每秒5兆比特的发送器==，并且由该助手==创建的通道上有2毫秒的延迟==。

```cpp
PointToPointHelper pointToPoint;
pointToPoint.SetDeviceAttribute("

DataRate", StringValue("5Mbps"));
pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));

NetDeviceContainer p2pDevices;
p2pDevices = pointToPoint.Install(p2pNodes);
```

然后，我们==实例化了一个NetDeviceContainer来跟踪点对点网络设备==，并在==点对点节点上安装设备==。

我们上面提到你将看到==CSMA设备和通道的帮助器==，接下来的代码引入了它们。CsmaHelper的工作方式与PointToPointHelper相同，但它创建并连接CSMA设备和通道。**在CSMA设备和==通道对==中，注意数据速率是通过通道属性指定而不是设备属性**。这是因为真实的CSMA网络不允许在给定通道上混合使用不同速率的设备，例如10Base-T和100Base-T设备。我们首先将==数据速率设置==为每秒100兆比特，然后将==通道的光速延迟==设置为6560纳秒（任意选择，假设每2000米段1纳秒），请注意，你可以使用其本机数据类型设置属性。

```cpp
CsmaHelper csma;
csma.SetChannelAttribute("DataRate", StringValue("100Mbps"));
csma.SetChannelAttribute("Delay", TimeValue(NanoSeconds(6560)));

NetDeviceContainer csmaDevices;
csmaDevices = csma.Install(csmaNodes);
```

就像我们创建了一个NetDeviceContainer来保存由PointToPointHelper创建的设备一样，==我们创建了一个NetDeviceContainer来保存由CsmaHelper创建的设备==。我们调用CsmaHelper的Install方法，==将设备安装到csmaNodes NodeContainer的节点中。==

现在我们**已经创建了节点、设备和通道，但我们没有协议栈**。就像在first.cc脚本中一样，我们将使用InternetStackHelper来安装这些栈。

```cpp
InternetStackHelper stack;
stack.Install(p2pNodes.Get(0));
stack.Install(csmaNodes);
```

*回想一下，我们从p2pNodes容器中取出了一个节点，并将其添加到csmaNodes容器中*。因此，==我们只需要在剩余的p2pNodes节点和csmaNodes容器中的所有节点上安装协议栈，以覆盖仿真中的所有节点==。⚠️

就像在first.cc示例脚本中一样，我们将==使用Ipv4AddressHelper为设备接口分配IP地址==。首先，我们使用网络10.1.1.0创建两个我们两个点对点设备所需的地址。

```cpp
Ipv4AddressHelper address;
address.SetBase("10.1.1.0", "255.255.255.0");
Ipv4InterfaceContainer p2pInterfaces;
p2pInterfaces = address.Assign(p2pDevices);
```

回想一下，我们将创建的接口保存在容器中，以便稍后轻松提取用于设置应用程序的寻址信息。

现在，我们==需要为我们的CSMA设备接口分配IP地址==。该操作的执行方式与点对点情况相同，只是现在我们在一个包含可变数量CSMA设备的容器上执行该操作——请记住，我们通过命令行参数使CSMA设备的数量可变。==在这种情况下，CSMA设备将与网络编号为10.1.2.0的IP地址相关联==，如下所示。

```cpp
address.SetBase("10.1.2.0", "255.255.255.0");
Ipv4InterfaceContainer csmaInterfaces;
csmaInterfaces = address.Assign(csmaDevices);
```

**现在我们建立了一个拓扑，但我们需要应用程序**。本节将基本上与first.cc的应用程序部分相似，但我们**将在具有CSMA设备的节点上实例化服务器，而在仅具有点对点设备的节点上实例化客户端**。

首先，我们设置回显服务器。我们创建了一个UdpEchoServerHelper，并在构造函数中为其提供了一个所需的属性值，即服务器端口号。请记住，稍后可以使用SetAttribute方法更改此端口（如果需要），但我们要求在构造函数中提供它。

```cpp
UdpEchoServerHelper echoServer(9);

ApplicationContainer serverApps = echoServer.Install(csmaNodes.Get(nCsma));
serverApps.Start(Seconds(1.0));
serverApps.Stop(Seconds(10.0));
```

csmaNodes NodeContainer包含为点对点网络创建的一个节点和nCsma个“额外”的节点。我们想要获取的是“额外”节点的最后一个。**csmaNodes容器的第零个条目将是点对点节点**。因此，==简单的思考方式是，如果我们创建一个“额外”的CSMA节点，那么它将位于csmaNodes容器的索引1处==。归纳起来，==如果我们创建nCsma个“额外”的节点，最后一个将位于索引nCsma处==。你可以在代码的第一行的Get中看到这一点。

客户端应用程序的设置与first.cc示例脚本中所做的完全相同。再次在构造函数中为==UdpEchoClientHelper提供所需的属性（在这种情况下是远程地址和端口）==。我们==告诉客户端将数据包发送到刚刚在“额外”的CSMA节点上安装的服务器==。我们将客户端安装在拓扑示意图中最左侧的点对点节点上。

```cpp
UdpEchoClientHelper echoClient(csmaInterfaces.GetAddress(nCsma), 9);
echoClient.SetAttribute("MaxPackets", UintegerValue(1));
echoClient.SetAttribute("Interval", TimeValue(Seconds(1.0)));
echoClient.SetAttribute("PacketSize", UintegerValue(1024));

ApplicationContainer clientApps = echoClient.Install(p2pNodes.Get(0));
clientApps.Start(Seconds(2.0));
clientApps.Stop(Seconds(10.0));
```

由于我们实际上在这里建立了一个互联网，我们需要某种形式的互联网路由。ns-3提供了我们称之为全局路由的帮助器。全局路由利用仿真中整个互联网可访问的事实，并通过仿真中创建的所有节点运行——它为你设置了路由，而无需配置路由器。

==基本上，每个节点的行为都好像它是一个OSPF路由器，与幕后的所有其他路由器进行即时和神奇的通信==。每个节点生成链接广播，并直接将它们通信到全局路由管理器，**该管理器使用此全局信息构造每个节点的路由表**。设置这种形式的路由是一个一行代码的操作：

```cpp
Ipv4GlobalRoutingHelper::PopulateRoutingTables();
```

接下来我们启用pcap跟踪。**启用点对点助手中的pcap跟踪**的第一行代码现在对你来说应该很熟悉。第二行**启用CSMA助手中的pcap跟踪**，并且还有一个你还没有遇到的额外参数。

```cpp
pointToPoint.EnablePcapAll("second");
csma.EnablePcap("second", csmaDevices.Get(1), true);
```

CSMA网络是一个多点到点网络。这意味着可以（并在这种情况下确实）在共享媒体上存在多个端点。每个端点都有与其关联的网络设备。从这种网络中获取跟踪信息的基本替代方法有两种。一种方法是**为每个网络设备创建一个跟踪文件，并仅存储该网络设备发出或消耗的数据包**。另一种方法是**选择其中一个设备并将其置于混杂模式**。然后，**该单个设备将对所有数据包进行“嗅探”并将它们存储在单个pcap文件中**。这就是==例如tcpdump的工作方式==。*最后一个参数*告诉CSMA助手*是否安排以混杂模式捕获数据包*。

在这个例子中，==我们将选择CSMA网络上的一个设备，并要求其以混杂模式对网络进行嗅探，从而模拟tcpdump的操作==。如果你在Linux机器上，你可能会做一些类似于tcpdump -i eth0的事情来获取跟踪。在这种情况下，==我们使用csmaDevices.Get(1)指定设备，选择容器中的第一个设备。将最后一个参数设置为true将启用混杂模式捕获==。

最后一部分的代码运行和清理仿真，就像first.cc示例一样。

```cpp
Simulator::Run();
Simulator::Destroy();
return 0;
}
```

为了运行这个例子，将second.cc示例脚本复制到scratch目录，并使用ns3构建脚本进行构建，就像在first.cc示例中一样。如果你在存储库的顶层目录，只需键入：

```bash
$ cp examples/tutorial/second.cc scratch/mysecond.cc
$ ./ns3 build
```

警告：我们将文件second.cc用作回归测试之一，以确保它确切地按照我们的期望工作，以使你的教程体验变得积极。这意味着项目中已经存在名为second的可执行文件。为了避免关于你正在执行的内容的任何混淆，请执行上面建议的将其重命名为mysecond.cc。

如果你正在密切关注教程，你仍然可能已经设置了NS_LOG变量，因此请清除该变量并运行程序。

```bash
$ export NS_LOG=""
$ ./ns3 run scratch/mysecond
```

由于我们已经设置了UDP回显应用程序记录，就像在first.cc中一样，当你运行脚本时，你将看到类似的输出。

```
At time +2s client sent 1024 bytes to 10.1.2.4 port 9
At time +2.0078s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.0078s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.01761s client received 1024 bytes from 10.1.2.4 port 9
```

回想一下，第一条消息“Sent 1024 bytes to 10.1.2.4”是UDP客户端将数据包发送到服务器。在这种情况下，服务器位于另一个网络（10.1.2.0）上。
第二条消息“Received 1024 bytes from 10.1.1.1”来自UDP服务器，在接收到回显数据包时生成。
最后一条消息“Received 1024 bytes from 10.1.2.4”来自客户端，表示它已经从服务器那里收到了回显。

如果你现在查看顶层目录，你将找到三个跟踪文件：

```
second-0-0.pcap  second-1-0.pcap  second-2-0.pcap
```

让我们花一点时间来看这些文件的命名。它们都具有相同的形式，\<name>-\<node>-\<device>.pcap。
例如，列表中的第一个文件是second-0-0.pcap，这是来自节点零、设备零的pcap跟踪。这是节点零上的点对点网络设备。
文件second-1-0.pcap是节点一上设备零的pcap跟踪，也是点对点网络设备；
而文件second-2-0.pcap是节点二上设备零的pcap跟踪。

```cpp
// Review: Default Network Topology
//
//       10.1.1.0
// n0 -------------- n1   n2   n3   n4
//    point-to-point  |    |    |    |
//                    ================
//                      LAN 10.1.2.0
```

如果你回顾一下本节开始时的拓扑示意图，你将看到节点零是点对点链路的最左边的节点，节点一是既有点对点设备又有CSMA设备的节点。你将看到节点二是CSMA网络上的第一个“额外”节点，设备零被选为以混杂模式捕获跟踪的设备。

现在，让我们跟踪回显数据包穿过互联网。首先，在左侧点对点节点（节点零）的跟踪文件上运行tcpdump。

```bash
$ tcpdump -nn -tt -r second-0-0.pcap
```

你应该会看到pcap文件的内容：

```
reading from file second-0-0.pcap, link-type PPP (PPP)
2.000000 IP 10.1.1.1.49153 > 10.1.2.4.9: UDP, length 1024
2.017607 IP 10.1.2.4.9 > 10.1.1.1.49153: UDP, length 1024
```

第一行指示==链接类型为PPP（点对点）==，这是我们期望的。然后，你会看到回显数据包通过与IP地址10.1.1.1关联的设备离开节点零，朝着IP地址10.1.2.4（最右边的CSMA节点）前进。这个数据包将经过点对点链路，然后由节点一上的点对点网络设备接收。让我们来看看：

```bash
$ tcpdump -nn -tt -r second-1-0.pcap
```

现在，你应该会看到点对点链路的另一端的pcap跟踪输出：

```
reading from file second-1-0.pcap, link-type PPP (PPP)
2.003686 IP 10.1.1.1.49153 > 10.1.2.4.9: UDP, length 1024
2.013921 IP 10.1.2.4.9 > 10.1.1.1.49153: UDP, length 1024
```

在这里，我们看到==链接类型也是PPP==，正如我们所期望的。你会看到从IP地址10.1.1.1（在2.000000秒时发送的）朝着IP地址10.1.2.4的数据包出现在此接口。现在，在此节点内部，数据包将被转发到CSMA接口，我们应该看到它从该设备弹出，朝着其最终目的地。

请记住，我们选择了节点2作为CSMA网络的混杂嗅探器节点，

因此让我们看看second-2-0.pcap，看看它是否存在。

```bash
$ tcpdump -nn -tt -r second-2-0.pcap
```

现在，你应该会看到节点二、设备零的混杂转储：

```
reading from file second-2-0.pcap, link-type EN10MB (Ethernet)
2.007698 ARP, Request who-has 10.1.2.4 (ff:ff:ff:ff:ff:ff) tell 10.1.2.1, length 50
2.007710 ARP, Reply 10.1.2.4 is-at 00:00:00:00:00:06, length 50
2.007803 IP 10.1.1.1.49153 > 10.1.2.4.9: UDP, length 1024
2.013815 ARP, Request who-has 10.1.2.1 (ff:ff:ff:ff:ff:ff) tell 10.1.2.4, length 50
2.013828 ARP, Reply 10.1.2.1 is-at 00:00:00:00:00:03, length 50
2.013921 IP 10.1.2.4.9 > 10.1.1.1.49153: UDP, length 1024
```

正如你所见，==链接类型现在是“以太网”==。然而，出现了一些新的东西:
总线网络需要ARP（地址解析协议）。节点一知道它需要将数据包发送到IP地址10.1.2.4，**但它不知道相应节点的MAC地址**。它在CSMA**网络上广播**（ff:ff:ff:ff:ff:ff）请求具有IP地址10.1.2.4的设备。
在这种情况下，**最右边的节点回复**说它的**MAC地址是00:00:00:00:00:06**。请注意，节点二在这个交换中没有直接参与，而是在嗅探网络并报告它看到的所有流量。

这个交换在以下几行中看到：

```
2.007698 ARP, Request who-has 10.1.2.4 (ff:ff:ff:ff:ff:ff) tell 10.1.2.1, length 50
2.007710 ARP, Reply 10.1.2.4 is-at 00:00:00:00:00:06, length 50
```

然后，节点一、设备一继续将回显数据包发送到IP地址10.1.2.4。

```
2.007803 IP 10.1.1.1.49153 > 10.1.2.4.9: UDP, length 1024
```

服务器接收回显请求，并尝试将数据包返回到源地址。服务器知道该地址在通过IP地址10.1.2.1到达的另一个网络上，这是因为我们初始化了全局路由，并且它已经为我们解决了所有这些问题。但是，回显服务器节点不知道第一个CSMA节点的MAC地址，因此它必须像第一个CSMA节点一样进行ARP。

```
2.013815 ARP, Request who-has 10.1.2.1 (ff:ff:ff:ff:ff:ff) tell 10.1.2.4, length 50
2.013828 ARP, Reply 10.1.2.1 is-at 00:00:00:00:00:03, length 50
```

然后，服务器将回显发送回转发节点。

```
2.013921 IP 10.1.2.4.9 > 10.1.1.1.49153: UDP, length 1024
```

回顾一下点对点链路的最右边的节点，

```bash
$ tcpdump -nn -tt -r second-1-0.pcap
```

你现在可以看到回显的数据包通过点对点链路回到的最后一行跟踪转储。

```
reading from file second-1-0.pcap, link-type PPP (PPP)
2.003686 IP 10.1.1.1.49153 > 10.1.2.4.9: UDP, length 1024
2.013921 IP 10.1.2.4.9 > 10.1.1.1.49153: UDP, length 1024
```

最后，你可以回顾发起回显的节点

```bash
$ tcpdump -nn -tt -r second-0-0.pcap
```

并看到回显的数据包在2.017607秒时到达源地址，

```
reading from file second-0-0.pcap, link-type PPP (PPP)
2.000000 IP 10.1.1.1.49153 > 10.1.2.4.9: UDP, length 1024
2.017607 IP 10.1.2.4.9 > 10.1.1.1.49153: UDP, length 1024
```

最后，请回想一下，我们==通过命令行参数添加了控制仿真中CSMA设备数量的功能==。你可以以与我们在first.cc示例中查看==更改回显的数据包数量相同的方式更改此参数==。尝试以“extra”设备设置为四个，而不是默认值为三个（extra nodes）运行程序：

```bash
$ ./ns3 run "scratch/mysecond --nCsma=4"
```

现在，你应该会看到，

```
At time +2s client sent 1024 bytes to 10.1.2.5 port 9
At time +2.0118s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.0118s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.02461s client received 1024 bytes from 10.1.2.5 port 9
```

请**注意，回显服务器现在已经被重新定位到CSMA节点的最后一个，即10.1.2.5，而不是默认情况下的10.1.2.4**。

**如果您不满意由CSMA网络中的旁观者生成的跟踪文件**，==而是希望从单个设备获取跟踪，而且对网络上的其他任何流量都不感兴趣==，您可以相对容易地实现这一点。

让我们查看`scratch/mysecond.cc`文件，并添加以下代码，以使我们能够更加具体。ns-3助手提供了一些方法，这些方法接受节点编号和设备编号作为参数。请继续替换下面的EnablePcap调用。

```cpp
pointToPoint.EnablePcap("second", p2pNodes.Get(0)->GetId(), 0);
csma.EnablePcap("second", csmaNodes.Get(nCsma)->GetId(), 0, false);
csma.EnablePcap("second", csmaNodes.Get(nCsma-1)->GetId(), 0, false);
```

我们知道我们要创建一个以“second”为基本名称的pcap文件，我们还知道在这两种情况下感兴趣的设备都将是零，所以这些参数实际上并不重要。

**为了获取节点编号，您有两种选择**：首先，==节点按照您创建它们的顺序以从零开始的单调递增的方式进行编号==。获取节点编号的一种方法是通过考虑节点创建的顺序来==“手动”确定这个数字==。如果您查看文件开头的网络拓扑图示，我们已经为您做了这个工作，您可以看到最后的CSMA节点将成为节点编号nCsma + 1。在较大的模拟中，这种方法可能变得非常困难。

另一种方法是==使用这里使用的方法，即认识到`NodeContainers`包含指向ns-3节点对象的指针==。`Node`对象有一个称为==`GetId`的方法，该方法将返回该节点的ID==，这就是我们要查找的节点编号。让我们去查看`Node`的Doxygen，并找到该方法，该方法在ns-3核心代码中比我们迄今为止看到的更深；但有时你必须辛勤搜索有用的东西。

转到您的版本的Doxygen文档（请记住，您可以在项目网站上找到它）。您可以通过查看“Classes”选项卡并在“Class List”中滚动查找ns3::Node来访问Node文档。选择ns3::Node，然后您将进入Node类的文档。如果现在滚动到`GetId`方法并选择它，您将进入该方法的详细文档。==在复杂的拓扑中，使用`GetId`方法可以更容易地确定节点编号==。

让我们清除顶级目录中的旧跟踪文件，以避免对正在进行的操作产生困惑：

```bash
$ rm *.pcap
```

在第110行，请注意以下命令，以在一个节点上启用跟踪（索引1对应于容器中的第二个CSMA节点）：

```cpp
csma.EnablePcap("second", csmaDevices.Get(1), true);
```

将索引更改为数量`nCsma`，对应于拓扑中的最后一个节点-包含回显服务器的节点：

```cpp
csma.EnablePcap("second", csmaDevices.Get(nCsma), true);
```

如果您构建新的脚本并运行将`nCsma`设置为100的模拟：

```bash
$ ./ns3 build
$ ./ns3 run "scratch/mysecond --nCsma=100"
```

您将看到以下输出：

```bash
At time +2s client sent 1024 bytes to 10.1.2.101 port 9
At time +2.0068s server received 1024 bytes from 10.1.1.1 port 49153
At time +2.0068s server sent 1024 bytes to 10.1.1.1 port 49153
At time +2.01761s client received 1024 bytes from 10.1.2.101 port 9
```

请注意，回显服务器现在位于`10.1.2.101`，这对应于拥有100个“额外”CSMA节点的节点上的回显服务器。如果列出顶级目录中的pcap文件，您将看到：

```bash
second-0-0.pcap  second-1-0.pcap  second-101-0.pcap
```

trace文件`second-0-0.pcap`是“最左侧”的点对点设备，是回显数据包的源。文件`second-101-0.pcap`对应于最右侧的CSMA设备，是回显服务器所在的位置。您可能已经注意到，在==启用回显服务器节点上的pcap跟踪时，调用的最后一个参数是true==。这==意味着在该节点上收集的跟踪处于混杂模式==。

为了说明混杂和非混杂跟踪之间的差异，让我们为倒数第二个节点添加一个非混杂的跟踪。在现有的PCAP跟踪行之前或之后添加以下行；false的最后一个参数表示您希望进行非混杂跟踪：

```cpp
csma.EnablePcap("second", csmaDevices.Get(nCsma - 1), false);
```

然后像之前一样构建和运行：

```bash
$ rm *.pcap
$ ./ns3 build
$ ./ns3 run "scratch/mysecond --nCsma=100"
```

这将生成一个新的PCAP文件，`second-100-0.pcap`。现在查看`second-100-0.pcap`的tcpdump。

```bash
$ tcpdump -nn -tt -r second-100-0.pcap
```

==现在您可以看到节点100实际上是回显交换中的旁观者。它接收到的唯一数据包是ARP请求，该请求广播到整个CSMA网络==。

```bash
reading from file second-100-0.pcap, link-type EN10MB (Ethernet)
2.006698 ARP, Request who-has 10.1.2.101 (ff:ff:ff:ff:ff:ff) tell 10.1.2.1, length 50
2.013815 ARP, Request who-has 10.1.2.1 (ff:ff:ff:ff:ff:ff) tell 10.1.2.101, length 50
```

==现在查看`second-101-0.pcap`的tcpdump==。

```bash
$ tcpdump -nn -tt -r second-101-0.pcap
```

==节点101实际上是回显交换的参与者==；无论是否在该PCAP语句中设置混杂模式，都将存在以下跟踪。

```bash
reading from file second-101-0.pcap, link-type EN10MB (Ethernet)
2.006698 ARP, Request who-has 10.1.2.101 (ff:ff:ff:ff:ff:ff) tell 10.1.2.1, length 50
2.006698 ARP, Reply 10.1.2.101 is-at 00:00:00:00:00:67, length 50
2.006803 IP 10.1.1.1.49153 > 10.1.2.101.9: UDP, length 1024
2.013803 ARP, Request who-has 10.1.2.1 (ff:ff:ff:ff:ff:ff) tell 10.1.2.101, length 50
2.013828 ARP, Reply 10.1.2.1 is-at 00:00:00:00:00:03, length 50
2.013828 IP 10.1.2.101.9 > 10.1.1.1.49153: UDP, length 1024
```
#### 整体初样代码review：
```cpp
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

#include "ns3/applications-module.h"
#include "ns3/core-module.h"
#include "ns3/csma-module.h"
#include "ns3/internet-module.h"
#include "ns3/ipv4-global-routing-helper.h"
#include "ns3/network-module.h"
#include "ns3/point-to-point-module.h"

// Default Network Topology
//
//       10.1.1.0
// n0 -------------- n1   n2   n3   n4
//    point-to-point  |    |    |    |
//                    ================
//                      LAN 10.1.2.0

using namespace ns3;

NS_LOG_COMPONENT_DEFINE("SecondScriptExample");

int
main(int argc, char* argv[])
{
    bool verbose = true;     // 确定是否启用UdpEchoClientApplication和UdpEchoServerApplication日志记录
    uint32_t nCsma = 3;      // 允许动态通过命令行参数更改CSMA网络上的设备数量

    CommandLine cmd(__FILE__);
    cmd.AddValue("nCsma", "Number of \"extra\" CSMA nodes/devices", nCsma);
    cmd.AddValue("verbose", "Tell echo applications to log if true", verbose);

    cmd.Parse(argc, argv);

    if (verbose)
    {
        LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
        LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO);
    }

    nCsma = nCsma == 0 ? 1 : nCsma; // 确保我们的拓扑中至少有一个“额外”的节点

    NodeContainer p2pNodes; // 建立2个我们将通过点对点链接连接的节点
    p2pNodes.Create(2);     // 节点容器called：p2pNodes，里面装有两个节点

    NodeContainer csmaNodes;        // 建立另一个NodeContainer来保存将成为总线（CSMA）网络一部分的节点
    csmaNodes.Add(p2pNodes.Get(1)); // 从点对点节点容器中获取第一个节点（索引为1）
                                    // 并将其添加到将获取CSMA设备的节点容器中
    csmaNodes.Create(nCsma);        // 创建合理点数的“CSMA容器”

    PointToPointHelper pointToPoint; // 建立一个“节点连接助手”，called：pointTopoint
    pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps")); // 该助手配置：每秒5兆比特的发送器
    pointToPoint.SetChannelAttribute("Delay", StringValue("2ms")); // 由该助手配置的通道上：有2毫秒的延迟

    NetDeviceContainer p2pDevices; // 建立一个“网络设备创建助手”
    p2pDevices = pointToPoint.Install(p2pNodes); // 在“点对点链接的节点”上安装设备

    CsmaHelper csma; // 建立一个“CSMA搭建助手”，called：csma
    csma.SetChannelAttribute("DataRate", StringValue("100Mbps")); // csma设备之间“通道对”的带宽是100Mbps
    csma.SetChannelAttribute("Delay", TimeValue(NanoSeconds(6560))); // 通道的光速延迟设置为6560ns

    NetDeviceContainer csmaDevices;        // 建立一个“网络设备创建助手”
    csmaDevices = csma.Install(csmaNodes); // 在“CSMA节点”上安装设备

    InternetStackHelper stack;      // 建立一个“网络栈搭建助手”
    stack.Install(p2pNodes.Get(0)); // 将“点对点连接的节点容器”内所有“剩余”的节点安排上协议栈
    stack.Install(csmaNodes);       // 将“CSMA节点容器”内所有的节点均安排上协议栈

    Ipv4AddressHelper address;      // 建立一个“IP地址分配助手”，旨在为设备接口分配IP地址
    address.SetBase("10.1.1.0", "255.255.255.0"); // 助手内含：使用网络10.1.1.0，创建“点对点设备”地址
    Ipv4InterfaceContainer p2pInterfaces;         // 建立一个“接口容器”，将创建的接口保存在容器中
    p2pInterfaces = address.Assign(p2pDevices);   // 将“点对点链接容器内的节点”接口赋上地址

    address.SetBase("10.1.2.0", "255.255.255.0"); // 需要为CSMA设备接口分配IP地址（使用网络10.1.2.0）
    Ipv4InterfaceContainer csmaInterfaces;        // 建立一个“接口容器”，将创建的接口保存在容器中
    csmaInterfaces = address.Assign(csmaDevices); // 将“CSMA容器内节点”接口赋上地址

    UdpEchoServerHelper echoServer(9); // 建立一个“服务器端搭建助手”

    ApplicationContainer serverApps = echoServer.Install(csmaNodes.Get(nCsma)); // 在CSMA容器内的最后一
                                                                                // 个节点上配置服务器
    serverApps.Start(Seconds(1.0)); // 服务器的开始时刻是t=1s
    serverApps.Stop(Seconds(10.0)); // 服务器的关闭时刻是t=10s

    UdpEchoClientHelper echoClient(csmaInterfaces.GetAddress(nCsma), 9); // 客户端的远程地址是“CSMA容器”
                                                                         // 配置好服务器的节点，即：
                                                                         // 上述index=nCsma的那个节点
    echoClient.SetAttribute("MaxPackets", UintegerValue(1));      // 规范：客户端发送数据包的最大数量
    echoClient.SetAttribute("Interval", TimeValue(Seconds(1.0))); // 规范：发送的时间间隔
    echoClient.SetAttribute("PacketSize", UintegerValue(1024));   // 规范：所发送数据包的最大容量

    ApplicationContainer clientApps = echoClient.Install(p2pNodes.Get(0)); // 建立一个“应用安装助手”
    clientApps.Start(Seconds(2.0)); // 客户端应用的开始时刻是t=2s
    clientApps.Stop(Seconds(10.0)); // 客户端应用的关闭时刻是t=10s

    Ipv4GlobalRoutingHelper::PopulateRoutingTables(); // 建立一个“设置全局路由表”的助手
                                                      // 该管理器使用此全局信息构造每个节点的路由表
    pointToPoint.EnablePcapAll("second");                 // 启用“点对点”助手中的pcap跟踪
    csma.EnablePcap("second", csmaDevices.Get(1), true);  // 启用CSMA助手中的pcap跟踪

    Simulator::Run();     // 代码运行
    Simulator::Destroy(); // 清理仿真
    return 0;
}
                                                                                       117,1         Bot
```
### 7.2. Models, Attributes and Reality
这是一个方便的地方，可以进行一个小小的远足并提出一个重要观点。可能对你来说很明显，也可能不是，但在使用模拟时，了解正在建模的内容以及未建模的内容是很重要的。例如，很容易将前一节中使用的CSMA设备和信道视为真实的以太网设备，并期望模拟结果直接反映在真实以太网中会发生的情况。但事实并非如此。

模型从本质上讲是对现实的抽象。模拟脚本作者最终有责任确定模拟整体以及其组成部分的所谓“准确范围”和“适用领域”，因此也就是其构成部分的准确性和适用性。

在某些情况下，比如CSMA，确定未建模的内容可能相当容易。通过阅读模型描述（csma.h），你可以了解到CSMA模型中没有冲突检测，并决定在模拟中使用它的适用性或者在结果中包含哪些注意事项。在其他情况下，配置可能与你无法购买的任何现实相符的行为可能会相当容易。花一些时间调查一些这样的实例以及你在模拟中如何轻松越出现实边界，这将是值得的。

正如你所看到的，ns-3提供了属性，用户可以轻松设置以更改模型行为。考虑CsmaNetDevice的两个属性：Mtu和EncapsulationMode。Mtu属性指示设备的最大传输单元。这是设备可以发送的最大协议数据单元（PDU）的大小。

在CsmaNetDevice中，MTU默认为1500字节。此默认值对应于RFC 894中找到的一个数字，“在以太网网络上传输IP数据报的标准”。该数字实际上是从10Base5（全规格以太网）网络的最大数据包大小（1518字节）派生而来的。如果从以太网数据包的DIX封装开销中减去18字节，则最终得到最大可能的数据大小（MTU）为1500字节。还可以发现IEEE 802.3网络的MTU为1492字节。这是因为LLC/SNAP封装在数据包中添加了额外的八字节开销。在这两种情况下，底层硬件只能发送1518字节，但数据大小是不同的。

为了设置封装模式，CsmaNetDevice提供了一个称为EncapsulationMode的属性，可以取Dix或Llc的值，分别对应以太网和LLC/SNAP封装。

如果将Mtu保持在1500字节，并将封装模式更改为Llc，结果将是一个网络，它使用LLC/SNAP封装将1500字节的PDU封装为包长为1526字节的数据包，在许多网络中这是非法的，因为它们最多可以传输1518字节每个数据包。这很可能会导致模拟结果微妙地不符合你可能期望的现实。

为了使情况变得更加复杂，存在着巨幅数据帧（1500 < MTU <= 9000字节）和超巨幅数据帧（MTU > 9000字节），虽然它们并没有得到IEEE的正式认可，但在一些高速（千兆位）网络和网络接口卡中是可用的。可以将封装模式保持为Dix，将CsmaNetDevice的Mtu属性设置为64000字节，即使关联的CsmaChannel DataRate设置为10兆位每秒。这本质上就模拟了一个由吸血鬼式1980年代风格的10Base5网络构建的以太网交换机，支持超巨幅数据报。尽管这从未被制造过，也不太可能被制造，但你很容易进行配置。

在前面的例子中，你使用命令行创建了一个具有100个Csma节点的模拟。你也同样轻松地创建了一个具有500个节点的模拟。如果你实际上在模拟一个10Base5吸血鬼式网络，那么完整规格以太网电缆的最大长度为500米，最小的挂接间距为2.5米。这意味着真实网络上只能有200个挂接点。你也可以很容易地以这种方式构建一个非法网络。这可能或可能不会导致一个有意义的模拟，这取决于你试图模拟什么。

在ns-3和任何模拟器中都可能发生类似的情况。例如，你可能能够以节点在相同的空间占据相同的位置，或者你可能能够配置违反基本物理定律的放大器或噪声水平。

ns-3通常倾向于灵活性，许多模型将允许自由设置属性，而不试图强制执行任何任意的一致性或特定的底层规范。

从中得到的要点是，ns-3将为你提供一个超级灵活的基础，让你进行实验。你有责任了解系统要求你做什么，并确保你创建的模拟与由你定义的现实有某种联系。
### 7.3. Building a Wireless Network Topology
在本节中，我们将进一步扩展我们对ns-3网络设备和信道的了解，==介绍一个无线网络的示例==。ns-3提供了一组802.11模型，试图提供802.11规范的准确MAC层实现和802.11a规范的“不那么慢”的PHY层模型。

就像我们在构建点对点拓扑时看到了点对点和CSMA拓扑助手对象一样，在本节中我们==将看到等效的Wifi拓扑助手==。这些助手的外观和操作对你来说应该会很熟悉。

我们在examples/tutorial目录中提供了一个示例脚本。这个脚本基于second.cc脚本，并添加了一个Wi-Fi网络。打开你喜欢的编辑器并查看examples/tutorial/third.cc。你已经看到了足够的ns-3代码，以理解这个例子中发生的大部分事情，但还有一些新的东西，所以我们将浏览整个脚本并检查一些输出。

就像在second.cc示例中一样（以及所有ns-3示例中一样），文件以emacs模式行和一些GPL模板开始。

看一下下面显示的ASCII艺术，显示了在示例中构建的默认网络拓扑。你可以看到我们==将通过左侧连接一个无线网络来进一步扩展我们的示例==。请注意，这是一个默认网络拓扑，因为你实际上可以改变在有线和无线网络上创建的节点数量。就像在second.cc脚本中一样，如果你更改==nCsma，它将给你一些“额外”的CSMA节点==。同样，你可以设置==nWifi来控制在模拟中创建多少STA（站点）节点==。无线网络上始终会有一个AP（接入点）节点。默认情况下，有**三个“额外”的CSMA节点**和三个**无线STA节点**。

代码开始时加载模块包含文件，就像在second.cc示例中所做的那样。有一些新的包含，对应于我们将在下面讨论的wifi模块和mobility模块。

```cpp
#include "ns3/core-module.h"
#include "ns3/point-to-point-module.h"
#include "ns3/network-module.h"
#include "ns3/applications-module.h"
#include "ns3/wifi-module.h"
#include "ns3/mobility-module.h"
#include "ns3/csma-module.h"
#include "ns3/internet-module.h"
```

网络拓扑图如下：

```plaintext
// 默认网络拓扑
//
//   Wifi 10.1.3.0
//                 AP
//  *    *    *    *
//  |    |    |    |    10.1.1.0
// n5   n6   n7   n0 -------------- n1   n2   n3   n4
//                   point-to-point  |    |    |    |
//                                   ================
//                                     LAN 10.1.2.0
```

你可以看到，我们正在为(成为无线网络接入点的)(左侧点对点链接)的**节点**添加一个新的网络设备。创建了一些无线STA节点，以填充图示左侧的新网络10.1.3.0。

在插图之后，使用了ns-3命名空间，并定义了一个日志组件。到目前为止，这应该都很熟悉了。

```cpp
using namespace ns3;

NS_LOG_COMPONENT_DEFINE("ThirdScriptExample");
```

主程序的开始方式与second.cc相似，通过添加一些命令行参数来启用或禁用日志组件，并更改创建的设备数量。

```cpp
bool verbose = true;
uint32_t nCsma = 3;
uint32_t nWifi = 3;

CommandLine cmd;
cmd.AddValue("nCsma", "Number of \"extra\" CSMA nodes/devices", nCsma);
cmd.AddValue("nWifi", "Number of wifi STA devices", nWifi);
cmd.AddValue("verbose", "Tell echo applications to log if true", verbose);

cmd.Parse(argc,argv);

if (verbose)
{
  LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
  LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO);
}
```

与之前的所有示例一样，下一步是创建两个将通过点对点链接连接的节点。

```cpp
NodeContainer p2pNodes;
p2pNodes.Create(2);
```

接下来，我们看到一个老朋友。我们实例化一个PointToPointHelper，并设置关联的默认属性，以便我们使用该助手创建的设备上创建每秒五兆位的发射机和由助手创建的信道上的两毫秒的延迟。然后我们将设备安装在节点和它们之间的通道上。

```cpp
PointToPointHelper pointToPoint;
pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));

NetDeviceContainer p2pDevices;
p2pDevices = pointToPoint.Install(p2pNodes);
```

接下来，我们声明另一个NodeContainer来容纳将成为总线（CSMA）网络一部分的节点。

```cpp
NodeContainer csmaNodes;
csmaNodes.Add(p2pNodes.Get(1));
csmaNodes.Create(nCsma);
```

下一行代码从点对点节点容器中获取第一个节点（索引为1），并将其添加到将获取CSMA设备的节点容器中。相关节点最终将具有点对点设备和CSMA设备。然后我们创建一些“额外”的节点，构成CSMA网络的其余部分。

然后，我们实例化了一个CsmaHelper，并像在前面的例子中一样设置其属性。我们创建了一个NetDeviceContainer来跟踪创建的CSMA网络设备，然后在选定的节点上安装CSMA设备。

```cpp
CsmaHelper csma;
csma.SetChannelAttribute("DataRate", StringValue("100Mbps"));
csma.SetChannelAttribute("Delay", TimeValue(NanoSeconds(6560)));

NetDeviceContainer csmaDevices;
csmaDevices = csma.Install(csmaNodes);
```

接下来，我们将创建将成为Wi-Fi网络一部分的节点。我们将按照命令行参数创建一些“站点”节点，并将点对点链接的“最左侧”节点作为接入点的节点。

```cpp
NodeContainer wifiStaNodes;
wifiStaNodes.Create(nWifi);
NodeContainer wifiApNode = p2pNodes.Get(0);
```

接下来的代码构建了Wi-Fi设备和这些Wi-Fi节点之间的互连通道。首先，我们配置PHY和通道助手：

```cpp
YansWifiChannelHelper channel = YansWifiChannelHelper::Default();
YansWifiPhyHelper phy = YansWifiPhyHelper::Default();
```

为简单起见，此代码使用==默认的PHY层配置和通道模型==，这些在YansWifiChannelHelper::Default和YansWifiPhyHelper::Default方法的API Doxygen文档中有详细说明。创建这些对象后，我们创建一个通道对象，并将其与我们的PHY层对象管理器关联，以确保由YansWifiPhyHelper创建的所有PHY层对象共享相同的基础通道，即它们共享相同的无线介质，可以通信和干扰：

```cpp
phy.SetChannel(channel.Create());
```

一旦PHY助手配置完成，我们可以专注于MAC层。WifiMacHelper对象用于设置MAC参数。下面的第二个语句创建一个将用于设置MAC层实现的“Ssid”属性值的802.11服务集标识符（SSID）对象。

```cpp
WifiMacHelper mac;
Ssid ssid = Ssid("ns-3-ssid");
WifiHelper wifi;
```

现在，我们==准备在节点上安装Wi-Fi模型，使用这四个助手对象（YansWifiChannelHelper、YansWifiPhyHelper、WifiMacHelper、WifiHelper）==和==上面创建的Ssid对象==。这些助手封装了许多默认配置，并且如果需要，可以使用额外的属性配置进行进一步定制。我们还将创建NetDevice容器来存储助手创建的WifiNetDevice对象的指针。

```cpp
NetDeviceContainer staDevices;
mac.SetType("ns3::StaWifiMac",
            "Ssid", SsidValue(ssid),
            "ActiveProbing", BooleanValue(false));
```

在上述代码中，==通过ns3::StaWifiMac类型的TypeId值指定了将由助手创建的具体类型的MAC层==。当标准至少为802.11n或更新版本时，WifiMacHelper对象的“QosSupported”属性默认设置为true。这两个配置的组合意味着接下来创建的MAC实例将是一个具有QoS意识的非AP站点（STA），位于基础结构BSS中（即具有AP的BSS）。最后，==“ActiveProbing”属性被设置为false。这意味着由该助手创建的MAC将不会发送探测请求，并且站点将监听AP的信标。==

一旦所有与站点相关的参数在MAC和PHY层都配置好，我们就可以调用我们现在熟悉的Install方法来创建这些站点的Wi-Fi设备：

```cpp
NetDeviceContainer staDevices;
staDevices = wifi.Install(phy, mac, wifiStaNodes);
```

我们已经为所有STA节点配置了Wi-Fi，现在我们需要配置AP（接入点）节点。我们通过更改WifiMacHelper的默认属性来反映AP的要求来开始这个过程。

```cpp
mac.SetType("ns3::ApWifiMac",
            "Ssid", SsidValue(ssid));
```

在这种情况下，==WifiMacHelper将创建“ns3::ApWifiMac”类型的MAC层，后者指定将创建一个配置为AP的MAC实例==。

接下来的行==创建单个AP，它与站点共享相同的PHY层属性（和信道）：==

```cpp
NetDeviceContainer apDevices;
apDevices = wifi.Install(phy, mac, wifiApNode);
```

现在，我们**将添加移动模型**。**我们希望STA节点是移动的**，在一个边界框内四处漫游，而我们希望使AP节点保持静止。我们使用MobilityHelper使这对我们来说变得容易。==首先，我们实例化一个MobilityHelper对象，并设置一些控制“位置分配器”功能的属性。==

```cpp
MobilityHelper mobility;

mobility.SetPositionAllocator("ns3::GridPositionAllocator",
                              "MinX", DoubleValue(0.0),
                              "MinY", DoubleValue(0.0),
                              "DeltaX", DoubleValue(5.0),
                              "DeltaY", DoubleValue(10.0),
                              "GridWidth", UintegerValue(3),
                              "LayoutType", StringValue("RowFirst"));
```

这段代码告诉移动助手使用二维网格来最初放置STA节点。随时查看ns3::GridPositionAllocator类的Doxygen，以了解确切在做什么。

我们已经在初始网格上安排了我们的节点，但==现在我们需要告诉它们如何移动。我们选择RandomWalk2dMobilityModel==，该模型使节点在一个边界框内以随机方向和随机速度移动。

```cpp
mobility.SetMobilityModel("ns3::RandomWalk2dMobilityModel",
                          "Bounds", RectangleValue(Rectangle(-50, 50, -50, 50)));
```

现在，我们==告诉MobilityHelper在STA节点上安装移动模型==。

```cpp
mobility.Install(wifiStaNodes);
```

我们**希望在仿真期间让接入点保持固定位置**。我们==通过将此节点的移动模型设置为ns3::ConstantPositionMobilityModel来实现==这一点：

```cpp
mobility.SetMobilityModel("ns3::ConstantPositionMobilityModel");
mobility.Install(wifiApNode);
```

现在，我们已经创建了我们的节点、设备和信道，并为Wi-Fi节点选择了移动模型，但我们还没有协议栈。就像我们以前做过很多次一样，我们将使用InternetStackHelper来安装这些栈。

```cpp
InternetStackHelper stack;
stack.Install(csmaNodes);
stack.Install(wifiApNode);
stack.Install(wifiStaNodes);
```

就像在second.cc示例脚本中一样，我们==将使用Ipv4AddressHelper为我们的设备接口分配IP地址==。首先，我们使用网络10.1.1.0创建两个点对点设备所需的地址。然后，我们使用网络10.1.2.0为CSMA网络分配地址，然后我们从网络10.1.3.0为无线网络上的STA设备和AP分配地址。

```cpp
Ipv4AddressHelper address;

address.SetBase("10.1.1.0", "255.255.255.0");
Ipv4InterfaceContainer p2pInterfaces;
p2pInterfaces = address.Assign(p2pDevices);

address.SetBase("10.1.2.0", "255.255.255.0");
Ipv4InterfaceContainer csmaInterfaces;
csmaInterfaces = address.Assign(csmaDevices);

address.SetBase("10.1.3.0", "255.255.255.0");
address.Assign(staDevices);
address.Assign(apDevices);
```

我们将回声服务器放在文件开头插图中的“最右侧”节点上。我们以前做过这个。

```cpp
UdpEchoServerHelper echoServer(9);

ApplicationContainer serverApps = echoServer.Install(csmaNodes.Get(nCsma));
serverApps.Start(Seconds(1.0));
serverApps.Stop(Seconds(10.0));
```

并将回声客户端放在我们创建的最后一个STA节点上，将其指向CSMA网络上的服务器。我们以前也进行过类似的操作。

```cpp
UdpEchoClientHelper echoClient(csmaInterfaces.GetAddress(nCsma), 9);
echoClient.SetAttribute("MaxPackets", UintegerValue(1));
echoClient.SetAttribute("Interval", TimeValue(Seconds(1.0)));
echoClient.SetAttribute("PacketSize", UintegerValue(1024));

ApplicationContainer clientApps =
    echoClient.Install(wifiStaNodes.Get(nWifi - 1));
clientApps.Start(Seconds(2.0));
clientApps.Stop(Seconds(10.0));
```

**由于我们在这里建立了一个互联网，我们需要启用互联网路由**，就像在second.cc示例脚本中所做的那样。

```cpp
Ipv4GlobalRoutingHelper::PopulateRoutingTables();
```

有一件事可能会让一些用户感到惊讶，即==我们刚刚创建的仿真将永远不会“自然”停止。这是因为我们要求无线接入点生成信标。它将永远生成信标，这将导致将来无限期地安排模拟器事件==，**因此我们必须告诉模拟器停止**，即使它可能已经安排了信标生成事件。以下代码行告诉模拟器停止，以防止我们无限模拟信标并进入本质上是无限循环的状态。

```cpp
Simulator::Stop(Seconds(10.0));
```

我们==创建了足够的跟踪来覆盖所有三个网络：==

```cpp
pointToPoint.EnablePcapAll("third");
phy.EnablePcap("third", apDevices.Get(0));
csma.EnablePcap("third", csmaDevices.Get(0), true);
```

这三行代码将在作为我们骨干的**两个点对点节点上启动**pcap跟踪，将在**Wi-Fi网络上启动**混杂（监视器）模式跟踪，并将在**CSMA网络上启动混杂跟踪。这样，我们就可以用最少数量的跟踪文件查看所有的流量。

最后，我们实际运行仿真，进行清理，然后退出程序。

```cpp
Simulator::Run();
Simulator::Destroy();
return 0;
}
```

为了运行这个例子，你必须将third.cc示例脚本复制到scratch目录中，并使用CMake进行构建，就像在second.cc示例中所做的那样。如果你在存储库的顶层目录中，输入以下命令：

```sh
$ cp examples/tutorial/third.cc scratch/mythird.cc
$ ./ns3 run 'scratch/mythird --tracing=1'
```

再次强调，由于我们设置了与second.cc脚本中相似的UDP回显应用程序，你将看到类似的输出。

在时间+2s时，客户端向10.1.2.4的端口9发送了1024字节
在时间+2.01624s时，服务器从10.1.3.3的端口49153接收到了1024字节
在时间+2.01624s时，服务器向10.1.3.3的端口49153发送了1024字节
在时间+2.02849s时，客户端从10.1.2.4的端口9接收到了1024字节
```shell
At time +2s client sent 1024 bytes to 10.1.2.4 port 9
At time +2.01828s server received 1024 bytes from 10.1.3.3 port 49153
At time +2.01828s server sent 1024 bytes to 10.1.3.3 port 49153
At time +2.02652s client received 1024 bytes from 10.1.2.4 port 9
```

回想一下，第一条消息“向10.1.2.4发送了1024字节”是UDP回显客户端向服务器发送数据包的消息。在这种情况下，客户端位于无线网络（10.1.3.0）上。第二条消息“从10.1.3.3接收到了1024字节”来自UDP回显服务器，表示服务器接收到了回显数据包。最后一条消息“从10.1.2.4接收到了1024字节”来自回显客户端，表示它已从服务器接收到了回显。

如果现在查看顶级目录，并按照上面建议的命令行启用了跟踪，你将在此仿真的四个跟踪文件中找到四个跟踪文件，两个来自节点零，两个来自节点一：

```sh
third-0-0.pcap  third-0-1.pcap  third-1-0.pcap  third-1-1.pcap
```

文件“third-0-0.pcap”对应于节点零上的点对点设备 - “骨干”左侧。
文件“third-1-0.pcap”对应于节点一上的点对点设备 - “骨干”右侧。
文件“third-0-1.pcap”将是Wi-Fi网络上的混杂（监视器模式）跟踪.
文件“third-1-1.pcap”将是CSMA网络上的混杂跟踪。你能通过检查代码验证这一点吗？

由于回显客户端位于Wi-Fi网络上，让我们从那里开始。让我们查看我们在该网络上捕获的混杂（监视器模式）跟踪。

```sh
$ tcpdump -nn -tt -r third-0-1.pcap
```

你应该会看到一些你以前没有看到的Wi-Fi内容：

```shell
reading from file third-0-1.pcap, link-type IEEE802_11_RADIO (802.11 plus radiotap header)

0.033119 33119us tsft 6.0 Mb/s 5210 MHz 11a Beacon (ns-3-ssid) [6.0* 9.0 12.0* 18.0 24.0* 36.0 48.0 54.0 Mbit] ESS
0.120504 120504us tsft 6.0 Mb/s 5210 MHz 11a -62dBm signal -94dBm noise Assoc Request (ns-3-ssid) [6.0 9.0 12.0 18.0 24.0 36.0 48.0 54.0 Mbit]
0.120520 120520us tsft 6.0 Mb/s 5210 MHz 11a Acknowledgment RA:00:00:00:00:00:08
0.120632 120632us tsft 6.0 Mb/s 5210 MHz 11a -62dBm signal -94dBm noise CF-End RA:ff:ff:ff:ff:ff:ff
0.120666 120666us tsft 6.0 Mb/s 5210 MHz 11a Assoc Response AID(1) :: Successful
...
```

你可以看到链接类型现在是802.11，这是你所期望的。你可能能够理解正在发生什么，并在这个跟踪中找到IP回显请求和响应数据包。我们将它作为一个练习来完全解析跟踪转储。

现在，查看==点对点链接的左侧的pcap==文件，

```sh
$ tcpdump -nn -tt -r third-0-0.pcap
```

同样，你应该看到一些熟悉的内容：

```shell
reading from file third-0-0.pcap, link-type PPP (PPP)
2.006440 IP 10.1.3.3.49153 > 10.1.2.4.9: UDP, length 1024
2.025048 IP 10.1.2.4.9 > 10.1.3.3.49153: UDP, length 1024
```

这是从左到右（从Wi-Fi到CSMA）的回显数据包，然后再次穿越点对点链接返回。

现在，查看==点对点链接的右侧的pcap==文件，

```sh
$ tcpdump -nn -tt -r third-1-0.pcap
```

同样，你应该看到一些熟悉的内容：

```shell
reading from file third-1-0.pcap, link-type PPP (PPP)
2.010126 IP 10.1.3.3.49153 > 10.1.2.4.9: UDP, length 1024
2.021361 IP 10.1.2.4.9 > 10.1.3.3.49153: UDP, length 1024
```

这也是从左到右（从Wi-Fi到CSMA）的回显数据包，但由于你可能期望的原因，时间稍微有所不同。

回显服务器位于CSMA网络上，让我们在那里查看混杂跟踪：

```sh
$ tcpdump -nn -tt -r third-1-1.pcap
```

你应该看到一些熟悉的内容：

```
reading from file third-1-1.pcap, link-type EN10MB (Ethernet)
2.016126 ARP, Request who-has 10.1.2.4 (ff:ff:ff:ff:ff:ff) tell 10.1.2.1, length 50
2.016151 ARP, Reply 10.1.2.4 is-at 00:00:00:00:00:06, length 50
2.016151 IP 10.1.3.3.49153 > 10.1.2.4.9: UDP, length 1024
2.021255 ARP, Request who-has 10.1.2.1 (ff:ff:ff:ff:ff:ff) tell 10.1.2.4, length 50
2.021255 ARP, Reply 10.1.2.1 is-at 00:00:00:00:00:03, length 50
2.021361 IP 10.1.2.4.9 > 10.1.3.3.49153: UDP, length 1024
```

这应该很容易理解。如果你忘记了，请返回并查看second.cc中的讨论。这是相同的顺序。

现在，我们花费了很多时间为无线网络设置移动模型，如果在不显示STA节点在仿真过程中移动的情况下结束，那将是一种遗憾。

让我们通过连接到MobilityModel课程(course)更改跟踪源来实现这一点。这只是详细跟踪部分的一个小窥探，但这似乎是一个很好的地方来展示一个例子。

如“调整ns-3”部分所述，==ns-3跟踪系统分为跟踪源和跟踪接收器，并且我们提供了将两者连接起来的函数==。我们将使==用MobilityModel预定义的课程(course)更改跟踪源来发出跟踪事件==。我们将需要编写一个跟踪接收器来连接到该源，以便为我们显示一些漂亮的信息。尽管它被认为是困难的，但它实际上非常简单。在scratch/mythird.cc脚本的主程序之前（即在NS_LOG_COMPONENT_DEFINE语句之后），添加以下函数：

```cpp
void
CourseChange(std::string context, Ptr<const MobilityModel> model)
{
  Vector position = model->GetPosition();
  NS_LOG_UNCOND(context <<
              " x = " << position.x << ", y = " << position.y);
}
```

这段代码==只是从移动模型中提取位置信息，并无条件地记录节点的x和y位置==。我们**将安排这个函数在具有回显客户端的无线节点改变其位置时被调用**。我们使用Config::Connect函数来做到这一点。在运行Simulator::Run调用之前，在脚本中添加以下代码行。

```cpp
std::ostringstream oss;
oss << "/NodeList/" << wifiStaNodes.Get(nWifi - 1)->GetId()
    << "/$ns3::MobilityModel/CourseChange";

Config::Connect(oss.str(), MakeCallback(&CourseChange));
```

在这里我们做的是==创建一个包含我们要连接的跟踪命名空间路径的字符串==。首先，我们必须通过**GetId方法找出我们想要的是哪个节点**。在CSMA和无线节点的默认数量的情况下，结果是节点七，移动模型的跟踪命名空间路径将如下所示：

```
/NodeList/7/$ns3::MobilityModel/CourseChange
```

基于跟踪部分的讨论，您可以推断此跟踪路径引用全局NodeList中的第七个节点。它指定了类型为`ns3::MobilityModel`的聚合对象。美元符号前缀意味着MobilityModel已聚合到节点七。路径的最后一个组件意味着我们正在连接到该模型的“CourseChange”事件。

我们==通过调用`Config::Connect`并传递此命名空间路径，在跟踪源和跟踪汇之间建立了连接==。*一旦完成此操作，每当节点七发生课程更改事件时，它都将连接到我们的跟踪汇，后者将依次打印出新的位置*。

如果您现在运行模拟，您将看到发生的课程更改。

```bash
$ ./ns3 build
$ ./ns3 run scratch/mythird
/NodeList/7/$ns3::MobilityModel/CourseChange x = 10, y = 0
/NodeList/7/$ns3::MobilityModel/CourseChange x = 9.36083, y = -0.769065
/NodeList/7/$ns3::MobilityModel/CourseChange x = 9.62346, y = 0.195831
/NodeList/7/$ns3::MobilityModel/CourseChange x = 9.42533, y = 1.17601
/NodeList/7/$ns3::MobilityModel/CourseChange x = 8.4854, y = 0.834616
/NodeList/7/$ns3::MobilityModel/CourseChange x = 7.79244, y = 1.55559
/NodeList/7/$ns3::MobilityModel/CourseChange x = 7.85546, y = 2.55361
At time +2s client sent 1024 bytes to 10.1.2.4 port 9
At time +2.01624s server received 1024 bytes from 10.1.3.3 port 49153
At time +2.01624s server sent 1024 bytes to 10.1.3.3 port 49153
At time +2.02849s client received 1024 bytes from 10.1.2.4 port 9
/NodeList/7/$ns3::MobilityModel/CourseChange x = 8.72774, y = 2.06461
/NodeList/7/$ns3::MobilityModel/CourseChange x = 9.52954, y = 2.6622
/NodeList/7/$ns3::MobilityModel/CourseChange x = 10.523, y = 2.77665
/NodeList/7/$ns3::MobilityModel/CourseChange x = 10.7054, y = 3.75987
/NodeList/7/$ns3::MobilityModel/CourseChange x = 10.143, y = 2.93301
/NodeList/7/$ns3::MobilityModel/CourseChange x = 10.2355, y = 1.9373
/NodeList/7/$ns3::MobilityModel/CourseChange x = 11.2152, y = 1.73647
/NodeList/7/$ns3::MobilityModel/CourseChange x = 10.2379, y = 1.94864
/NodeList/7/$ns3::MobilityModel/CourseChange x = 10.4491, y = 0.971199
/NodeList/7/$ns3::MobilityModel/CourseChange x = 9.56013, y = 1.42913
/NodeList/7/$ns3::MobilityModel/CourseChange x = 9.11607, y = 2.32513
/NodeList/7/$ns3::MobilityModel/CourseChange x = 8.22047, y = 1.88027
/NodeList/7/$ns3::MobilityModel/CourseChange x = 8.79149, y = 1.05934
/NodeList/7/$ns3::MobilityModel/CourseChange x = 9.41195, y = 0.275103
/NodeList/7/$ns3::MobilityModel/CourseChange x = 9.83369, y = -0.631617
/NodeList/7/$ns3::MobilityModel/CourseChange x = 9.15219, y = 0.100206
/NodeList/7/$ns3::MobilityModel/CourseChange x = 8.32714, y = 0.665266
/NodeList/7/$ns3::MobilityModel/CourseChange x = 7.46368, y = 0.160847
/NodeList/7/$ns3::MobilityModel/CourseChange x = 7.40394, y = -0.837367
/NodeList/7/$ns3::MobilityModel/CourseChange x = 6.96716, y = -1.73693
/NodeList/7/$ns3::MobilityModel/CourseChange x = 7.62062, y = -2.49388
/NodeList/7/$ns3::MobilityModel/CourseChange x = 7.99793, y = -1.56779
```
这些输出显示了节点七的课程更改。
##### review：源码
```cpp
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

#include "ns3/applications-module.h"
#include "ns3/core-module.h"
#include "ns3/csma-module.h"
#include "ns3/internet-module.h"
#include "ns3/mobility-module.h"
#include "ns3/network-module.h"
#include "ns3/point-to-point-module.h"
#include "ns3/ssid.h"
#include "ns3/yans-wifi-helper.h"

// Default Network Topology
//
//   Wifi 10.1.3.0
//                 AP
//  *    *    *    *
//  |    |    |    |    10.1.1.0
// n5   n6   n7   n0 -------------- n1   n2   n3   n4
//                   point-to-point  |    |    |    |
//                                   ================
//                                     LAN 10.1.2.0

using namespace ns3;

NS_LOG_COMPONENT_DEFINE("ThirdScriptExample");

int
main(int argc, char* argv[])
{
    bool verbose = true;
    uint32_t nCsma = 3;
    uint32_t nWifi = 3;
    bool tracing = false;

    CommandLine cmd(__FILE__);
    cmd.AddValue("nCsma", "Number of \"extra\" CSMA nodes/devices", nCsma);
    cmd.AddValue("nWifi", "Number of wifi STA devices", nWifi);
    cmd.AddValue("verbose", "Tell echo applications to log if true", verbose);
    cmd.AddValue("tracing", "Enable pcap tracing", tracing);

    cmd.Parse(argc, argv);

    // The underlying restriction of 18 is due to the grid position
    // allocator's configuration; the grid layout will exceed the
    // bounding box if more than 18 nodes are provided.
    if (nWifi > 18)
    {
        std::cout << "nWifi should be 18 or less; otherwise grid layout exceeds the bounding box"
                  << std::endl;
        return 1;
    }

    if (verbose)
    {
        LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
        LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO);
    }

    NodeContainer p2pNodes;
    p2pNodes.Create(2);

    PointToPointHelper pointToPoint;
    pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
    pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));

    NetDeviceContainer p2pDevices;
    p2pDevices = pointToPoint.Install(p2pNodes);

    NodeContainer csmaNodes;
    csmaNodes.Add(p2pNodes.Get(1));
    csmaNodes.Create(nCsma);

    CsmaHelper csma;
    csma.SetChannelAttribute("DataRate", StringValue("100Mbps"));
    csma.SetChannelAttribute("Delay", TimeValue(NanoSeconds(6560)));

    NetDeviceContainer csmaDevices;
    csmaDevices = csma.Install(csmaNodes);

    NodeContainer wifiStaNodes;
    wifiStaNodes.Create(nWifi);
    NodeContainer wifiApNode = p2pNodes.Get(0);

    // 构建了Wi-Fi设备和这些Wi-Fi节点之间的互连通道:
    YansWifiChannelHelper channel = YansWifiChannelHelper::Default();// 建立了默认的通道模型“搭建助手”
    YansWifiPhyHelper phy;             // 建立了默认的PHY模型“搭建助手”
    phy.SetChannel(channel.Create());
    // 建立一个“通道对象”，并将其与我们的PHY层对象管理器关联，以确保由YansWifiPhyHelper创建的所有PHY层对象共享相同的基础通道，即它们共享相同的无线介质，可以通信和干扰

    // PHY助手配置已经完成，现在我们可以专注于MAC层：
    WifiMacHelper mac;             // 建立一个WifiMacHelper对象，用于设置MAC参数
    Ssid ssid = Ssid("ns-3-ssid"); // 建立一个将用于设置MAC层实现的“Ssid”属性值的802.11SSID对象

    // 现在，我们准备在节点上安装Wi-Fi模型：（我们将依赖于四个helper）
    WifiHelper wifi;

    NetDeviceContainer staDevices;
    mac.SetType("ns3::StaWifiMac", "Ssid", SsidValue(ssid), "ActiveProbing", BooleanValue(false));
    // 这里四个参数，实现所有与站点相关的参数在MAC和PHY层都配置好
    // 第一个是TypeId值，指定了将由助手创建的具体类型的MAC层
    // 最后一个“ActiveProbing”属性被设置为false，这意味着由该助手创建的MAC将不会发送探测请求，并且站点将监听AP的信标。

    staDevices = wifi.Install(phy, mac, wifiStaNodes);// 调用熟悉的Install方法来创建这些站点的Wi-Fi设备

    // 我们已经为所有STA节点配置了Wi-Fi，现在我们需要配置AP（接入点）节点：
    NetDeviceContainer apDevices;
    mac.SetType("ns3::ApWifiMac", "Ssid", SsidValue(ssid)); // WifiMacHelper将创建“ns3::ApWifiMac”类型的MAC层，后者指定将创建一个配置为AP的MAC实例
    apDevices = wifi.Install(phy, mac, wifiApNode);// 创建单个AP，它与站点共享相同的PHY层属性（和信道）

    MobilityHelper mobility;// 建立一个MobilityHelper对象（移动搭建助手）

    mobility.SetPositionAllocator("ns3::GridPositionAllocator",
                                  "MinX",
                                  DoubleValue(0.0),
                                  "MinY",
                                  DoubleValue(0.0),
                                  "DeltaX",
                                  DoubleValue(5.0),
                                  "DeltaY",
                                  DoubleValue(10.0),
                                  "GridWidth",
                                  UintegerValue(3),
                                  "LayoutType",
                                  StringValue("RowFirst")); // 使用二维网格来最初放置STA节点

    mobility.SetMobilityModel("ns3::RandomWalk2dMobilityModel",
                              "Bounds",
                              RectangleValue(Rectangle(-50, 50, -50, 50)));
    // 我们选择RandomWalk2dMobilityModel告诉这些点如何移动
    mobility.Install(wifiStaNodes); // 告诉MobilityHelper在STA节点上安装移动模型

    mobility.SetMobilityModel("ns3::ConstantPositionMobilityModel");// 希望仿真期间让接入点AP保持固定位置
    mobility.Install(wifiApNode); //通过将此节点的移动模型设置为ns3::ConstantPositionMobilityModel来实现

    // 现在，我们已经创建了我们的节点、设备和信道，并为Wi-Fi节点选择了移动模型，但我们还没有协议栈：
    InternetStackHelper stack;
    stack.Install(csmaNodes);
    stack.Install(wifiApNode);
    stack.Install(wifiStaNodes);

    Ipv4AddressHelper address; // 建立一个IPv4地址分配助手

    // 使用Ipv4AddressHelper为我们的设备接口分配IP地址：
    address.SetBase("10.1.1.0", "255.255.255.0");
    Ipv4InterfaceContainer p2pInterfaces;
    p2pInterfaces = address.Assign(p2pDevices); // 使用网络10.1.1.0创建两个点对点设备所需的地址

    address.SetBase("10.1.2.0", "255.255.255.0");
    Ipv4InterfaceContainer csmaInterfaces;
    csmaInterfaces = address.Assign(csmaDevices); // 使用网络10.1.2.0为CSMA网络分配地址

    address.SetBase("10.1.3.0", "255.255.255.0");
    address.Assign(staDevices);
    address.Assign(apDevices);              // 然后我们从网络10.1.3.0为无线网络上的STA设备和AP分配地址

    UdpEchoServerHelper echoServer(9); // 将回声服务器放在文件开头插图中的“最右侧”节点上：

    ApplicationContainer serverApps = echoServer.Install(csmaNodes.Get(nCsma));
    serverApps.Start(Seconds(1.0));
    serverApps.Stop(Seconds(10.0));

    UdpEchoClientHelper echoClient(csmaInterfaces.GetAddress(nCsma), 9);
    echoClient.SetAttribute("MaxPackets", UintegerValue(1));
    echoClient.SetAttribute("Interval", TimeValue(Seconds(1.0)));
    echoClient.SetAttribute("PacketSize", UintegerValue(1024));
    // 将回声客户端放在我们创建的最后一个STA节点上，将其指向CSMA网络上的服务器。
    // 我们以前也进行过类似的操作。

    ApplicationContainer clientApps = echoClient.Install(wifiStaNodes.Get(nWifi - 1));
    clientApps.Start(Seconds(2.0));
    clientApps.Stop(Seconds(10.0));

    // 由于我们在这里建立了一个互联网，我们需要启用互联网路由：
    Ipv4GlobalRoutingHelper::PopulateRoutingTables();

    Simulator::Stop(Seconds(10.0)); //告诉模拟器停止，以防止我们无限模拟信标并进入本质上是无限循环的状态

    if (tracing)                    // 创建足够的跟踪来覆盖所有（三个）网络
    {
        phy.SetPcapDataLinkType(WifiPhyHelper::DLT_IEEE802_11_RADIO);
        pointToPoint.EnablePcapAll("third");
        phy.EnablePcap("third", apDevices.Get(0));
        csma.EnablePcap("third", csmaDevices.Get(0), true);
    }

    Simulator::Run();
    Simulator::Destroy();
    return 0;
}
```
### 7.4. Queues in ns-3
在ns-3中，==队列调度机制的选择对性能有很大影响==，用户了解默认安装的内容以及如何更改默认设置并观察性能是很重要的。

从架构上讲，ns-3将**设备层与互联网主机的IP层或流量控制层分开**。
在ns-3的最新版本中，==出站数据包在到达信道对象之前经过两个队列层==。
- 遇到的第一个队列层是在ns-3中称为“流量控制层”的地方；在这里，通过使用队列调度**以设备无关的方式进行主动队列管理**（RFC7567）和由于**服务质量（QoS）而进行**的优先级设置。
- 第二个队列层通常在NetDevice对象中找到。不同的设备（例如LTE、Wi-Fi）对这些队列有不同的实现。这两层的方法反映了实际情况（软件队列提供优先级设置，硬件队列特定于链接类型）。在实践中，情况可能比这更复杂。例如，地址解析协议具有较小的队列。Linux中的Wi-Fi具有四个队列层（[https://lwn.net/Articles/705884/）。](https://lwn.net/Articles/705884/%EF%BC%89%E3%80%82)

==流量控制层==仅当NetDevice面临==在设备队列已满时==通知它时才有效，**以便流量控制层可以停止将数据包发送到NetDevice**。否则，队列调度的积压始终为零，它们就无效。

目前，**以下**支持使用Queue对象（或Queue子类的对象）存储其数据包的NetDevice**支持流控制**（即：通知流量控制层的能力）：
- Point-To-Point
- Csma
- Wi-Fi
- SimpleNetDevice

队列调度的性能受到NetDevices使用的队列大小的极大影响。目前，在ns-3中，==默认情况下队列不会根据配置的链路属性（带宽、延迟）进行自动调整==，通常是最简单的变体（例如，具有先进先出调度和丢弃尾部行为的FIFO调度）。

然而，通过启用BQL（字节队列限制），可以动态调整队列的大小。这是Linux内核中实现的一种算法，用于根据链路属性调整设备队列的大小，以防止缓冲区过大而避免饥饿。

目前，支持流控制的NetDevice支持BQL。通过使用ns-3模拟和实际实验进行的关于设备队列大小对队列调度效果的影响的分析在以下文献中报告：P. Imputato 和 S. Avallone. An analysis of the impact of network device buffers on packet schedulers through experiments and simulations. Simulation Modelling Practice and Theory, 80(Supplement C):1–18, January 2018. DOI: 10.1016/j.simpat.2017.09.008
#### 7.4.1. Available queueing models in ns-3
在流量控制层，以下是一些选项：

- PFifoFastQueueDisc：默认最大大小为1000个数据包
- FifoQueueDisc：默认最大大小为1000个数据包
- RedQueueDisc：默认最大大小为25个数据包
- CoDelQueueDisc：默认最大大小为1500千字节
- FqCoDelQueueDisc：默认最大大小为10240个数据包
- PieQueueDisc：默认最大大小为25个数据包
- MqQueueDisc：此队列调度没有容量限制
- TbfQueueDisc：默认最大大小为1000个数据包

==默认情况下，当IPv4或IPv6地址分配给与NetDevice相关联的接口时，在NetDevice上安装了pfifo_fast队列调度==，除非NetDevice上已经安装了其他队列调度。

在设备层，有特定于设备的队列：

- PointToPointNetDevice：默认配置（由helper设置）是安装默认大小（100个数据包）的DropTail队列
- CsmaNetDevice：默认配置（由helper设置）是安装默认大小（100个数据包）的DropTail队列
- WiFiNetDevice：默认配置是为非QoS站点安装默认大小（100个数据包）的DropTail队列，并为QoS站点安装默认大小（100个数据包）的四个DropTail队列
- SimpleNetDevice：默认配置是安装默认大小（100个数据包）的DropTail队列
- LteNetDevice：队列处理发生在RLC层（RLC UM默认缓冲区为10 * 1024字节，RLC AM没有缓冲区限制）
- UanNetDevice：MAC层有一个默认的10个数据包队列

#### 7.4.2. Changing from the defaults

通过设备助手，可以通常修改NetDevice使用的队列类型：

```cpp
NodeContainer nodes;
nodes.Create(2);

PointToPointHelper p2p;
p2p.SetQueue("ns3::DropTailQueue", "MaxSize", StringValue("50p"));
// 将点对点链接的节点容器的队列管理设置为ns3::DropTailQueue

NetDeviceContainer devices = p2p.Install(nodes);
```

通过流量控制助手，可以修改安装在NetDevice上的队列调度器的类型：

```cpp
InternetStackHelper stack;
stack.Install(nodes);

TrafficControlHelper tch;
tch.SetRootQueueDisc("ns3::CoDelQueueDisc", "MaxSize", StringValue("1000p"));
// 将网络设备拥塞控制器的队列管理设置为ns3::DropTailQueue

tch.Install(devices);
```

如果设备支持，可以通过流量控制助手启用BQL：

```cpp
InternetStackHelper stack;
stack.Install(nodes);

TrafficControlHelper tch;
tch.SetRootQueueDisc("ns3::CoDelQueueDisc", "MaxSize", StringValue("1000p"));
tch.SetQueueLimits("ns3::DynamicQueueLimits", "HoldTime", StringValue("4ms"));
tch.Install(devices);
```