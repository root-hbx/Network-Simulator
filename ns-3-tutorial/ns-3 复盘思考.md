>做了几个月的计算机网络实验，感觉之前对ns-3的理解并不到位，于是我花了点时间从源理解整个ns-3包的构建&运行原理
## 前置技能
### Git
Complex software systems need some way to manage the organization and changes to the underlying code and documentation. There are many ways to perform this feat, and you may have heard of some of the systems that are currently used to do this. Until recently, the _ns-3_ project used Mercurial as its source code management system, but in December 2018, switch to using Git. Although you do not need to know much about Git in order to complete this tutorial, we recommend becoming familiar with Git and using it to access the source code.
### CMake工作原理
Once you have source code downloaded to your local system, you will need to compile that source to produce usable programs. Just as in the case of source code management, there are many tools available to perform this function. Probably the most well known of these tools is `make`. Along with being the most well known, `make` is probably the most difficult to use in a very large and highly configurable system. Because of this, many alternatives have been developed.

The build system CMake is used on the _ns-3_ project.  
### C++
As mentioned above, scripting in _ns-3_ is done in C++ or Python. Most of the _ns-3_ API is available in Python, but the models are written in C++ in either case. A working knowledge of C++ and object-oriented concepts is assumed in this document.
### Socket Programming
We will assume a basic facility with the Berkeley Sockets API in the examples used in this tutorial. If you are new to sockets, we recommend reviewing the API and some common usage cases. For a good overview of programming TCP/IP sockets we recommend [TCP/IP Sockets in C, Donahoo and Calvert](https://www.elsevier.com/books/tcp-ip-sockets-in-c/donahoo/978-0-12-374540-8).

You can get an online version on Google Library at [TCP / IP Sockets in C](https://books.google.com.hk/books/about/TCP_IP_Sockets_in_C.html?id=dmt_mERzxV4C&redir_esc=y)
## ns-3的运行原理
_ns-3_ is built as a system of software libraries that work together. User programs can be written that links with (or imports from) these libraries. User programs are written in either the C++ or Python programming languages.

_ns-3_ is distributed as source code, meaning that the target system needs to have a software development environment to build the libraries first, then build the user program. _ns-3_ could in principle be distributed as pre-built libraries for selected systems, and in the future it may be distributed that way.
### 构建命令
To build a specific target such as `test-runner` we use the following ns3 command:

```bash
./ns3 build
```

corresponding to :
```bash
$ cd /ns-3-dev/cmake-cache/
$ cmake --build . -j 16  # This command builds all the targets with the underlying build system
```
### 运行命令
```bash
./ns3 run test-runner
```

corresponding to :
```bash
$ cd /ns-3-dev/cmake-cache/
$ cmake --build . -j 16 --target test-runner # This command builds the test-runner target calling the underlying build system
$ export PATH=$PATH:/ns-3-dev/build/:/ns-3-dev/build/lib:/ns-3-dev/build/bindings/python # export library paths
$ export LD_LIBRARY_PATH=/ns-3-dev/build/:/ns-3-dev/build/lib:/ns-3-dev/build/bindings/python
$ export PYTHON_PATH=/ns-3-dev/build/:/ns-3-dev/build/lib:/ns-3-dev/build/bindings/python
$ /ns-3-dev/build/utils/ns3-dev-test-runner-optimized # call the executable with the real path
```

Note: 
>the command above would fail if `./ns3 build` was not executed first, since the examples won’t be built by the test-runner target.
### 运行脚本
We typically run scripts under the control of ns3. This allows the build system to ensure that the shared library paths are set correctly and that the libraries are available at run time. To run a program, simply use the `--run` option in ns3. Let’s run the _ns-3_ equivalent of the ubiquitous hello world program by typing the following:
### 提供运行参数
To feed command line arguments to an _ns-3_ program use this pattern:
```bash
$ ./ns3 run '<ns3-program> --arg1=value1 --arg2=value2 ...'
```
### Debug
To run _ns-3_ programs under the control of another utility, such as a debugger (_e.g._ `gdb`) or memory checker (_e.g._ `valgrind`), you use a similar `--command-template="..."` form.

For example, to run your _ns-3_ program `hello-simulator` with the arguments `<args>` under the `gdb` debugger:
```bash
$ ./ns3 run hello-simulator --command-template="gdb %s --args <args>"
```
## 设计理念
In this section, we’ll review some terms that are commonly used in networking, but have a specific meaning in _ns-3_.
### Node
In Internet jargon, a computing device that connects to a network is called a _host_ or sometimes an _end system_. Because _ns-3_ is a _network_ simulator, not specifically an _Internet_ simulator, we intentionally do not use the term host since it is closely associated with the Internet and its protocols. Instead, we use a more generic term also used by other simulators that originates in Graph Theory — the _node_.

In _ns-3_ the basic computing device abstraction is called the node. This abstraction is represented in C++ by the class `Node`. The `Node` class provides methods for managing the representations of computing devices in simulations.

You should think of a `Node` as a computer to which you will add functionality. One adds things like applications, protocol stacks and peripheral cards with their associated drivers to enable the computer to do useful work. We use the same basic model in _ns-3_.
### Application
Typically, computer software is divided into two broad classes. _System Software_ organizes various computer resources such as memory, processor cycles, disk, network, etc., according to some computing model. System software usually does not use those resources to complete tasks that directly benefit a user. A user would typically run an _application_ that acquires and uses the resources controlled by the system software to accomplish some goal.

Often, the line of separation between system and application software is made at the privilege level change that happens in operating system traps. In _ns-3_ there is no real concept of operating system and especially no concept of privilege levels or system calls. We do, however, have the idea of an application. Just as software applications run on computers to perform tasks in the “real world,” _ns-3_ applications run on _ns-3_ `Nodes` to drive simulations in the simulated world.

In _ns-3_ the basic abstraction for a user program that generates some activity to be simulated is the application. This abstraction is represented in C++ by the class `Application`. The `Application` class provides methods for managing the representations of our version of user-level applications in simulations. Developers are expected to specialize the `Application` class in the object-oriented programming sense to create new applications. In this tutorial, we will use specializations of class `Application` called `UdpEchoClientApplication` and `UdpEchoServerApplication`. As you might expect, these applications compose a client/server application set used to generate and echo simulated network packets
### Channel
In the real world, one can connect a computer to a network. Often the media over which data flows in these networks are called _channels_. When you connect your Ethernet cable to the plug in the wall, you are connecting your computer to an Ethernet communication channel. In the simulated world of _ns-3_, one connects a `Node` to an object representing a communication channel. Here the basic communication subnetwork abstraction is called the channel and is represented in C++ by the class `Channel`.

The `Channel` class provides methods for managing communication subnetwork objects and connecting nodes to them. `Channels` may also be specialized by developers in the object oriented programming sense. A `Channel` specialization may model something as simple as a wire. The specialized `Channel` can also model things as complicated as a large Ethernet switch, or three-dimensional space full of obstructions in the case of wireless networks.

We will use specialized versions of the `Channel` called `CsmaChannel`, `PointToPointChannel` and `WifiChannel` in this tutorial. The `CsmaChannel`, for example, models a version of a communication subnetwork that implements a _carrier sense multiple access_ communication medium. This gives us Ethernet-like functionality.
### Net Device
It used to be the case that if you wanted to connect a computer to a network, you had to buy a specific kind of network cable and a hardware device called (in PC terminology) a _peripheral card_ that needed to be installed in your computer. If the peripheral card implemented some networking function, they were called Network Interface Cards, or _NICs_. Today most computers come with the network interface hardware built in and users don’t see these building blocks.

A NIC will not work without a software driver to control the hardware. In Unix (or Linux), a piece of peripheral hardware is classified as a _device_. Devices are controlled using _device drivers_, and network devices (NICs) are controlled using _network device drivers_ collectively known as _net devices_. In Unix and Linux you refer to these net devices by names such as _eth0_.

In _ns-3_ the _net device_ abstraction covers both the software driver and the simulated hardware. **A net device is “installed” in a `Node` in order to enable the `Node` to communicate with other `Nodes` in the simulation via `Channels`**. Just as in a real computer, a `Node` may be connected to more than one `Channel` via multiple `NetDevices`.

The net device abstraction is represented in C++ by the class `NetDevice`. **The `NetDevice` class provides methods for managing connections to `Node` and `Channel` objects**; and may be specialized by developers in the object-oriented programming sense. We will use the several specialized versions of the `NetDevice` called `CsmaNetDevice`, `PointToPointNetDevice`, and `WifiNetDevice` in this tutorial. Just as an Ethernet NIC is designed to work with an Ethernet network, the `CsmaNetDevice` is designed to work with a `CsmaChannel`; the `PointToPointNetDevice` is designed to work with a `PointToPointChannel` and a `WifiNetDevice` is designed to work with a `WifiChannel`.
### Topology Helpers
In a real network, you will find host computers with added (or built-in) NICs. In _ns-3_ we would say that you will find `Nodes` with attached `NetDevices`. In a large simulated network you will need to arrange many connections between `Nodes`, `NetDevices` and `Channels`.

Since **connecting `NetDevices` to `Nodes`, `NetDevices` to `Channels`**, assigning IP addresses, etc., are such common tasks in _ns-3_, we provide what we call _topology helpers_ to make this as easy as possible. For example, it may take many distinct _ns-3_ core operations to create a NetDevice, add a MAC address, install that net device on a `Node`, configure the node’s protocol stack, and then connect the `NetDevice` to a `Channel`. Even more operations would be required to connect multiple devices onto multipoint channels and then to connect individual networks together into internetworks. We provide topology helper objects that combine those many distinct operations into an easy to use model for your convenience.
## 实例分析
- Here we take first.cc as an example.
- __Pay attention to the differences between Helper and Container !__

```c++
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
#include "ns3/internet-module.h"
#include "ns3/network-module.h"
#include "ns3/point-to-point-module.h"

// Default Network Topology
//
//       10.1.1.0
// n0 -------------- n1
//    point-to-point
//

using namespace ns3;

NS_LOG_COMPONENT_DEFINE("FirstScriptExample"); // 声明了一个名为FirstScriptExample的日志记录组件，它允许我引用名称来启用或禁用日志记录

int
main(int argc, char* argv[])
{
    CommandLine cmd(__FILE__);
    cmd.Parse(argc, argv);

    // 设置时间分辨率为ns，分辨率表示“可以表示的最小时间值”（以及两个时间值之间的最小可表示差异），如不设置，则默认是1ns
    Time::SetResolution(Time::NS); 

    /*
    在Echo客户端和服务器应用程序中，我们将使用日志记录功能来记录应用程序的活动。
        - 我们可以通过调用LogComponentEnable函数来启用日志记录。
        - Arg1: 日志记录组件的名称; Arg2: 日志记录级别
    日志记录级别是一个枚举类型，它定义了日志记录的详细程度。
        - 在这里，我们将日志记录级别设置为LOG_LEVEL_INFO，这意味着我们将记录所有信息性消息和更高级别的消息 => 应用程序在模拟期间发送和接收数据包时打印出信息
        - 我们可以使用其他级别，如LOG_LEVEL_DEBUG、LOG_LEVEL_WARN、LOG_LEVEL_ERROR、LOG_LEVEL_FUNCTION等。
    */
    LogComponentEnable("UdpEchoClientApplication", LOG_LEVEL_INFO);
    LogComponentEnable("UdpEchoServerApplication", LOG_LEVEL_INFO);

//  1. 物理背景建模

    // Container
    // NodeContainer 是一个“节点拓扑创建助手”，它本质上是一个“节点容器”，创造多少节点应在下一步骤中按需
    NodeContainer nodes;
    nodes.Create(2); // 容器创建了两个节点，容器调用ns-3系统本身来创建2节点，并在内部存储指向这些节点的指针
    // 然而此时仅仅是创建了2节点，下一步应将节点连接在一起形成网络
    
    // Helper
    // PointToPointHelper 是一个“点对点创建助手”，这里Point-Point包含：1. NetDevice || 2. Channel
    PointToPointHelper pointToPoint;
    // point-point 配置节点的网络设备，DataRate是数据传输速率
    pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
    // point-point 配置connection链路，Delay是数据传输延迟
    pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));

    // Container
    // NetDeviceContainer 是一个“网络设备容器”，需保存的网络设备早在上一步骤中就已确立
    NetDeviceContainer devices;
    /*
    Install方法采用NodeContainer作为参数，它将在节点上安装网络设备并负责连接链路
    - 它在内部自动创建了一个NetDeviceContainer
    - 对于NodeContainer中的每个节点，它将创建一个NetDevice附加到该节点上，并将其保存在NetDeviceContainer中
    */
    devices = pointToPoint.Install(nodes);
    /*
    现在：我们有2个节点，每个节点都安装了一个点对点网络设备，并且它们之间有一个点对点通道
    两个设备都配置为：传输延迟2ns，数据传输速率5Mbps
    */
    
    // Helper
    // InternetStackHelper 是一个“Internet协议栈创建助手”，它负责配置容器内每个节点的网络协议栈
    InternetStackHelper stack;
    // Install方法采用NodeContainer作为参数，它将在NodeContainer的每个Node上安装Internet协议栈(TCP/UDP/IP...)
    stack.Install(nodes);

    // 现在我们已经把整个网络的“物理意义”建模完毕了，下面将实现网络的“功能”

//  2. 网络功能建模
    
    // 2.1 创建 a list of IP Addresses
    // Helper
    // Ipv4AddressHelper 是一个“IPv4地址创建助手”，来管理Ip地址的分配 (还没具体分配到每个节点上 !)
    Ipv4AddressHelper address;
    address.SetBase("10.1.1.0", "255.255.255.0");
    // 声明了一个地址创建助手，从网络10.1.1.0开始分配IP，使用mask=255.255.255.0来定义可分配位
    // 此时：分配的地址将从1开始单调递增：10.1.1.1, 10.1.1.2, 10.1.1.3 ......

    // 2.2 将上述 IP Addresses 分配到每个节点上
    Ipv4InterfaceContainer interfaces = address.Assign(devices);
    // 使用IPv4Interface对象在 IP地址 和 设备(NetDevice) 之间建立关联

//  3. 创建应用层
    // 3.1 Server = Node 0
    // UDP Echo Server Helper是一个针对”建立服务器“的“应用程序创建帮手” 
    UdpEchoServerHelper echoServer(9); // 需要端口号作为构造函数的Arg，这里选择的端口号是9 => 服务器开启的service端口是9

    // ApplicationContainer 是一个“应用程序容器”，它负责管理应用程序的安装和启动
    ApplicationContainer serverApps = echoServer.Install(nodes.Get(1)); // "helper" -> nodes.Get(1) -> defined as "Server"
    serverApps.Start(Seconds(1.0)); // 模拟事件从1s开始
    serverApps.Stop(Seconds(10.0)); // 模拟事件到10s结束

    // 3.2 Client = Node 1
    // UDP Echo Client Helper是一个针对”建立客户端“的“应用程序创建帮手”
    // 3.2.1 远程连接
    UdpEchoClientHelper echoClient(interfaces.GetAddress(1), 9);  // 传递用于远程连接的IP地址(Node 1) 和 端口号(Port 9)
    // 3.2.2 配置客户端属性（参数...）
    echoClient.SetAttribute("MaxPackets", UintegerValue(1));      // 设置在模拟期间发送的最大数据包数量
    echoClient.SetAttribute("Interval", TimeValue(Seconds(1.0))); // 设置发送数据包之间的时间间隔
    echoClient.SetAttribute("PacketSize", UintegerValue(1024));   // 数据包有效负载是1024bytes

    // ApplicationContainer 是一个“应用程序容器”，它负责管理应用程序的安装和启动
    ApplicationContainer clientApps = echoClient.Install(nodes.Get(0)); // "helper" -> nodes.Get(1) -> defined as "Client"
    clientApps.Start(Seconds(2.0)); // 客户端从2s开始工作
    clientApps.Stop(Seconds(10.0)); // 客户端到10s结束工作

    Simulator::Run();
    Simulator::Destroy();
    return 0;
}
```
