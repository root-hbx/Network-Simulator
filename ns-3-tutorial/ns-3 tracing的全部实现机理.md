### 8.1. Background
如使用跟踪系统中所述，运行ns-3模拟的整个目的是为了生成用于研究的输出。获取ns-3输出有两种基本策略：使用**通用预定义的批量输出机制**并==解析其内容以提取感兴趣的信息==；或者以**某种方式开发输出机制**，==传达确切（可能仅有的）所需信息==。

**使用预定义的批量输出机制**的优点是不需要对ns-3进行任何更改，但**可能需要编写脚本来解析和过滤感兴趣的数据**。通常，在模拟运行期间收集PCAP或NS_LOG输出消息，并通过使用grep、sed或awk的脚本分别运行这些消息，以解析和减少数据并将其转换为可管理的形式。必须编写程序来进行转换，因此这并不是免费的。NS_LOG输出不被视为ns-3 API的一部分，并且在发布之间可能会在不提前通知的情况下更改。此外，NS_LOG输出仅在调试构建中可用，因此依赖它会导致性能损失。当然，如果感兴趣的信息在任何预定义的输出机制中都不存在，这种方法将失败。

如果需要向预定义的批量机制添加一些信息片段，这当然是可以做的；如果使用ns-3机制之一，可能会将您的代码作为贡献添加进去。

ns-3==提供了另一种机制，称为跟踪（tracing）==，它避免了批量输出机制固有的一些问题。它具有几个重要的优势。
首先，您可以通过**仅跟踪您感兴趣的事件来减少您需要管理的数据量**（对于大型模拟，将所有内容转储到磁盘进行后期处理可能会创建I/O瓶颈）。
其次，如果使用此方法，您**可以直接控制输出的格式**，因此可以避免使用sed、awk、perl或python脚本的后处理步骤。如果需要，您的输出可以直接格式化为gnuplot接受的形式（也请参阅GnuplotHelper）。您可以在核心中添加钩子，然后其他用户可以访问这些钩子，但除非明确要求，否则它们将不会生成任何信息。
因此，我们认为ns-3跟踪系统(tracing system)是从模拟中获取信息的最佳方式之一，也因此是理解ns-3中最重要的机制之一。
#### 8.1.1. Blunt Instruments
>"Blunt Instruments" 的中文翻译可以是 "钝器"。在这个上下文中，"Blunt Instruments" 可能指的是一种==不够精确或直接的工具或方法，类似于使用钝器而非精细工具==。在文中，这可能是指使用不够精确或灵活的方法来获取模拟数据，例如通过大量输出和后处理脚本，而不是使用更直接、精确的跟踪系统。

有很多方法可以从程序中获取信息。最直接的方法是直接将信息打印到标准输出，如下所示：

```cpp
#include <iostream>
// ...
void SomeFunction()
{
  uint32_t x = SOME_INTERESTING_VALUE;
  // ...
  std::cout << "The value of x is " << x << std::endl;
  // ...
}
```

没有人会阻止你深入ns-3的核心并添加打印语句。这做起来非常简单，毕竟，你完全掌控自己的ns-3分支。然而，从长远来看，这可能并不是很令人满意的方法！

随着程序中打印语句的数量增加，处理大量输出的任务会变得越来越复杂。最终，您可能会感到有必要以某种方式控制正在打印的信息，例如通过打开和关闭某些类别的打印，或增加或减少所需信息的数量。如果您继续沿着这条道路走下去，可能会发现您已经重新实现了NS_LOG机制（请参阅使用Logging Module）。为了避免这种情况，您可能首先考虑使用NS_LOG本身。

我们==之前提到从ns-3获取信息的一种方法是解析现有的NS_LOG输出以获取有趣的信息==。如果您发现您需要的某些信息片段在现有的日志输出中不存在，您可以编辑ns-3的核心并简单地将您感兴趣的信息添加到输出流中。现在，这肯定比添加自己的打印语句更好，因为它遵循ns-3的编码约定，而且可能对其他人作为现有核心的补丁也有用。

让我们随机选一个例子。如果您想要在ns-3 TCP套接字（tcp-socket-base.cc）中添加更多的日志记录，您可以在实现中添加一个新的消息。请注意，在TcpSocketBase::ProcessEstablished()中，对于在已建立状态下接收到SYN+ACK的情况，没有日志消息。您可以简单地添加一个，修改代码。以下是原始代码：

```cpp
/* Received a packet upon ESTABLISHED state. This function is mimicking the
    role of tcp_rcv_established() in tcp_input.c in Linux kernel. */
void TcpSocketBase::ProcessEstablished(Ptr<Packet> packet, const TcpHeader& tcpHeader)
{
  NS_LOG_FUNCTION(this << tcpHeader);
  // ...

  else if (tcpflags == (TcpHeader::SYN | TcpHeader::ACK))
    { // No action for received SYN+ACK, it is probably a duplicated packet
    }
  // ...
}
```

要记录SYN+ACK情况，可以在if语句体中添加一个新的NS_LOG_LOGIC：

```cpp
/* Received a packet upon ESTABLISHED state. This function is mimicking the
    role of tcp_rcv_established() in tcp_input.c in Linux kernel. */
void TcpSocketBase::ProcessEstablished(Ptr<Packet> packet, const TcpHeader& tcpHeader)
{
  NS_LOG_FUNCTION(this << tcpHeader);
  // ...
  else if (tcpflags == (TcpHeader::SYN | TcpHeader::ACK))
    { // No action for received SYN+ACK, it is probably a duplicated packet
      NS_LOG_LOGIC("TcpSocketBase " << this << " ignoring SYN+ACK");
    }
  // ...
}
```

乍一看，这可能看起来相当简单和令人满意，但需要考虑的一点是，您将编写代码以添加NS_LOG语句，并且还必须编写代码（就像grep、sed或awk脚本一样）以解析日志输出，以便隔离您的信息。这是因为尽管您对日志系统输出有一些控制权，但您只能控制到日志组件级别，通常是整个源代码文件。

如果您正在向现有模块添加代码，您还必须接受每个其他开发人员都找到有趣的输出。您可能会发现，为了获取所需的少量信息，您可能必须浏览大量对您无关的多余消息。您可能被迫将巨大的日志文件保存到磁盘并在想要执行任何操作时将它们处理为几行。

由于==在ns-3中关于NS_LOG输出的稳定性没有保证==，您还可能发现您依赖的日志输出的某些部分在发布之间消失或更改。如果依赖输出的结构，您可能会发现其他消息正在添加或删除，这可能会影响您的解析代码。

最后，==NS_LOG输出仅在调试构建中可用，您无法从优化构建中获取日志输出==，后者运行大约两倍快。依赖NS_LOG会带来性能损耗。

出于这些原因，==我们认为向std::cout和NS_LOG消息添加打印语句是从ns-3中获取更多信息的一种快速而不够严谨的方法，不适用于严肃的工作。==

理想情况下，希望有一个稳定的设施，使用稳定的API，允许用户访问核心系统并仅获取所需的信息。最好的情况是，可以在项目更改或发生有趣事件时通知用户代码，使用户无需主动在系统中查找事物。

ns-3跟踪系统旨在沿这些方向工作，并与属性和配置子系统紧密集成，允许相对简单的使用场景。
### Overview
ns-3的追踪系统建立在独立追踪源和追踪接收器的概念上，同时提供了一个连接源和接收器的统一机制。

**追踪源**是能够**发出模拟中事件信号**并**提供对有趣底层数据的访问**的实体。例如，追踪源可以指示数据包何时被网络设备接收，并为感兴趣的追踪接收器提供对数据包内容的访问。追踪源还可能指示模型中发生有趣状态变化的时刻。==例如，TCP模型的拥塞窗口是追踪源的主要候选者。每当拥塞窗口发生变化时，连接的追踪接收器会得到旧值和新值的通知。==

追踪源**本身并不实用；它们必须连接到其他实际利用源提供信息的代码片段**。

消耗追踪信息的实体称为**追踪接收器**。==追踪源是数据的生成者，而追踪接收器是数据的消费者==。**这种明确的划分允许大量的追踪源分布在系统中，这些位置是模型作者认为可能有用的地方**。*插入追踪源会引入非常小的执行开销*。

对于由追踪源生成的追踪事件，可以有零个或多个消费者。可以将追踪源视为一种点对多点信息链路。寻找来自特定核心代码的追踪事件的代码可以与执行完全不同操作的其他代码和平共存。

除非用户将追踪接收器连接到这些源之一，否则将不会输出任何内容。通过使用追踪系统，您和其他连接到相同追踪源的人都可以从系统中获取确切想要的信息，而不会影响其他用户通过更改系统输出的信息。如果您恰好添加了一个追踪源，作为良好的开源公民，您的工作可能允许其他用户提供可能在整体上非常有用的新实用程序，而无需对ns-3核心进行任何更改。
### Simple Example
让我们花几分钟来简单地追踪一个示例。为了理解示例中发生的事情，我们需要先了解一些关于回调（Callbacks）的背景知识，因此我们首先需要稍微偏离一下主题。
#### Callbacks
在ns-3中，回调系统的目标是允许代码的一部分调用一个函数（或C++中的方法），而无需具体的模块间依赖关系。这最终意味着你需要一种间接的方式 - 你将调用的函数的地址视为变量。这个变量被称为指向函数的指针变量。函数和指向函数的指针之间的关系实际上与对象和指向对象的指针的关系没有太大区别。

在C中，指向函数的经典示例是指向返回整数的函数指针（PFI）。对于一个带有一个int参数的PFI，可以这样声明：

```c
int (*pfi)(int arg) = 0;
```

（但在编写这样的代码之前，请阅读C++-FAQ第33节！）从中得到的是一个名为pfi的变量，它被初始化为值0。

如果要将此指针初始化为有意义的内容，需要具有匹配签名的函数。在这种情况下，你可以提供一个看起来像这样的函数：

```c
int MyFunction(int arg) {}
```

如果有了这个目标，你可以将变量初始化为指向你的函数：

```c
pfi = MyFunction;
```

然后，你可以间接调用MyFunction，使用更具提示性的调用形式：

```c
int result = (*pfi)(1234);
```

这是具有提示性的，因为它看起来像你正在解引用函数指针，就像你解引用任何指针一样。然而，通常，人们会利用编译器知道发生了什么的事实，将使用一个更短的形式：

```c
int result = pfi(1234);
```

这==看起来像你正在调用一个名为pfi的函数，但编译器足够聪明，知道通过变量pfi间接调用函数MyFunction==。

从概念上讲，这几乎与跟踪系统的工作方式完全相同。基本上，跟踪池是一个回调。当跟踪池对接收跟踪事件表达兴趣时，它将自己作为回调添加到跟踪源内部持有的回调列表中。当发生有趣的事件时，跟踪源调用它的operator(...)并提供零个或多个参数。operator(...)最终漫游到系统中并执行与刚才看到的间接调用非常相似的操作，提供零个或多个参数，就像上面对目标函数MyFunction的调用传递了一个参数一样。

跟踪系统增加的重要区别是对于每个跟踪源，都有一个内部回调列表。跟踪源可能调用多个回调，而不仅仅是进行一次间接调用。当跟踪池表达对来自跟踪源的通知的兴趣时，它基本上只是安排将其自己的函数添加到回调列表中。

如果你对在ns-3中实际上是如何安排这些的更多细节感兴趣，请随时查阅ns-3手册中的回调部分。
#### Walkthrough: `fourth.cc`
我们提供了一些代码，实现了实际上是跟踪的最简单示例。你可以在教程目录中找到这个代码，文件名为fourth.cc。让我们逐步解释它：

```cpp
/* -*- Mode:C++; c-file-style:"gnu"; indent-tabs-mode:nil; -*- */
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

#include "ns3/object.h"
#include "ns3/uinteger.h"
#include "ns3/traced-value.h"
#include "ns3/trace-source-accessor.h"

#include <iostream>

using namespace ns3;
```

这段代码应该对你来说非常熟悉。如上所述，==跟踪系统大量使用Object和Attribute系统==，因此你需要包含它们。上面的前两个包含显式地引入了这些系统的声明。虽然你可以使用核心模块的头文件一次性获取所有内容，但我们在这里显式地包含以说明这一切实际上是多么简单。

文件`traced-value.h`引入了==遵循值语义的数据跟踪所需的声明==。通常，==值语义只是意味着你可以传递对象本身，而不是传递对象的地址==。这实际上意味着你将能够以一种非常简单的方式跟踪对`TracedValue` 所做的所有更改。

由于跟踪系统与Attributes集成，并且Attributes与Objects一起工作，必须有一个ns-3 Object来容纳跟踪源。下一段==代码片段声明并定义了一个我们可以使用的简单Object：==

```cpp
class MyObject : public Object
{
public:
  static TypeId GetTypeId()
  {
    static TypeId tid = TypeId("MyObject")
      .SetParent(Object::GetTypeId())
      .SetGroupName("MyGroup")
      .AddConstructor<MyObject>()
      .AddTraceSource("MyInteger",
                      "An integer value to trace.",
                      MakeTraceSourceAccessor(&MyObject::m_myInt),
                      "ns3::TracedValueCallback::Int32")
      ;
    return tid;
  }

  MyObject() {}
  TracedValue<int32_t> m_myInt; // 提供了驱动回调过程的基础设施
};
```

在跟踪的角度来看，上面的代码中的两行最重要的是`.AddTraceSource` 和`TracedValue` 声明 `m_myInt`。

`.AddTraceSource` 提供了==通过Config系统将跟踪源连接到外部世界的“挂钩”==。
第一个参数是**此跟踪源的名称，在Config系统中可见**。
第二个参数是帮助字符串。
第三个参数，实际上聚焦在第三个参数的参数上：`&MyObject::m_myInt`。这是要**添加到类**的`TracedValue`；它**始终是一个类数据成员**。 （最后一个参数是 `TracedValue` 类型的 `typedef` 的名称，作为字符串。这用于生成正确的Callback函数签名的文档，对于更一般类型的Callbacks尤其有用。）

`TracedValue<>` 声明==提供了驱动回调过程的基础设施==。每==当底层值更改时==，`TracedValue` ==机制将提供该变量的旧值和新值==，这种情况下是 `int32_t` 值。此 `TracedValue` 的trace sink函数 `traceSink` 需要以下签名：

```cpp
void (*traceSink)(int32_t oldValue, int32_t newValue);
```

所有连接到此跟踪源的trace sink都必须具有此签名。我们将在下面讨论如何确定其他情况下所需的回调签名。

当我们继续阅读 `fourth.cc` 时，我们看到：

```cpp
void
IntTrace(int32_t oldValue, int32_t newValue)
{
  std::cout << "Traced " << oldValue << " to " << newValue << std::endl;
}
```

这是一个==匹配的跟踪池（trace sink）的定义==。它直接对应于回调函数的签名。一旦连接，每当 `TracedValue` 发生变化时，此函数将被调用。

我们现在已经看到了跟踪源和跟踪池。剩下的是连接源和池的代码，这发生在 `main` 函数中：

```cpp
int
main(int argc, char *argv[])
{
  Ptr<MyObject> myObject = CreateObject<MyObject>();
  myObject->TraceConnectWithoutContext("MyInteger", MakeCallback(&IntTrace));

  myObject->m_myInt = 1234;
}
```

在这里，我们首先创建了包含跟踪源的 `MyObject` 实例。

下一步，`TraceConnectWithoutContext` 形成了跟踪源和跟踪池之间的连接。
- 第一个参数只是我们上面看到的==跟踪源名称“MyInteger”==。
- 注意 `MakeCallback` 模板函数。此函数执行所需的魔法，以创建底层的 ns-3 `Callback` 对象，并将其与函数 `IntTrace` 关联。
- `TraceConnect` 建立了你提供的函数与由“MyInteger”属性引用的跟踪变量中的 `overloaded operator()` 之间的关联。
- 建立此关联后，跟踪源将“触发”你提供的回调函数。

当然，使所有这些发生的代码是非常繁琐的，但其本质是你正在安排类似于上面的 `pfi()` 示例的东西，被跟踪源调用。在 `Object` 本身中 `TracedValue<int32_t> m_myInt;` 的声明执行了所需的魔法，提供了使用 `operator()` 调用回调的 `overloaded` 赋值运算符。`.AddTraceSource` 执行魔法，将 `Callback` 连接到 `Config` 系统，而 `TraceConnectWithoutContext` 执行魔法，将你的函数连接到由属性名称指定的跟踪源。

暂时让我们忽略有关上下文的部分。

最后，赋值给 `m_myInt` 的这一行：

```cpp
myObject->m_myInt = 1234;
```

应该被解释为在成员变量 `m_myInt` 上调用 `operator=` 并将整数 `1234` 作为参数传递。

由于 `m_myInt` 是一个 `TracedValue`，此运算符被定义为执行一个返回 `void` 并接受两个整数值作为参数的回调 - 有关所讨论整数的旧值和新值。这正是我们提供的回调函数 `IntTrace` 的函数签名。

总结一下，==跟踪源本质上是一个保存回调函数列表的变量==。==跟踪池是作为回调目标的函数==。Attribute和对象类型信息系统用于提供一种连接跟踪源和跟踪池的方式。“击中”跟踪源的行为是在跟踪源上执行运算符，这会触发回调函数。这导致已注册对源感兴趣的跟踪池回调被调用，调用参数由源提供。

如果你现在构建并运行这个例子：

```bash
$ ./ns3 run fourth
```

你将看到从 `IntTrace` 函数的输出，这会在跟踪源被触发时执行：

```
Traced 0 to 1234
```

当我们执行代码 `myObject->m_myInt = 1234;` 时，跟踪源被触发，并自动将前后的值提供给跟踪池。然后，`IntTrace` 函数将这些值打印到标准输出。
### Connect with Config
在简单示例中上面显示的 `TraceConnectWithoutContext` 调用实际上在系统中非常少见。

更典型的情况是使用 Config 子系统使用所谓的 Config 路径在系统中选择一个跟踪源。我们在前一节中看到了一个例子，当我们在 `third.cc` 中进行实验时，我们连接了“CourseChange”事件。

回想一下，我们定义了一个跟踪池，以打印模拟中移动模型的航向变化信息。现在，对于你来说，应该更清楚这个函数在做什么：

```cpp
void
CourseChange(std::string context, Ptr<const MobilityModel> model)
{
  Vector position = model->GetPosition();
  NS_LOG_UNCOND(context <<
    " x = " << position.x << ", y = " << position.y);
}
```

当我们将“CourseChange”跟踪源连接到上面的跟踪池时，我们使用了 Config 路径来指定源，当我们在预定义的跟踪源和新的跟踪池之间建立连接时：

```cpp
std::ostringstream oss;
oss << "/NodeList/"
    << wifiStaNodes.Get(nWifi - 1)->GetId()
    << "/$ns3::MobilityModel/CourseChange";  // 这就是 “Config 路径”

Config::Connect(oss.str(), MakeCallback(&CourseChange)); // 在预定义的跟踪源和新的跟踪池之间建立连接
```

让我们试着理解这段有时被认为相对神秘的代码。为了讨论，假设 `GetId()` 返回的节点编号是 “7”。在这种情况下，上述路径实际上是：

`"/NodeList/7/$ns3::MobilityModel/CourseChange"`

配置路径的最后一段必须是对象的属性。

事实上，如果你手头有具有“CourseChange”属性的对象的指针，你可以像在前面的示例中那样直接编写它。
你现在已经知道我们通常将指向节点的指针存储在 `NodeContainer` 中。
在 `third.cc` 示例中，感兴趣的节点存储在 `wifiStaNodes` 的 `NodeContainer` 中。

事实上，在组合路径时，我们使用了此容器获取了 `Ptr<Node>`，并用它调用了 `GetId()`。我们可以使用这个 `Ptr<Node>` 直接调用 `Connect` 方法：

```cpp
Ptr<Object> theObject = wifiStaNodes.Get(nWifi - 1);
theObject->TraceConnectWithoutContext("CourseChange", MakeCallback(&CourseChange));
```

在 `third.cc` 示例中，我们实际上希望随着回调参数一起传递附加的 “context”，以便我们可以使用以下等效代码：

```cpp
Ptr<Object> theObject = wifiStaNodes.Get(nWifi - 1);
theObject->TraceConnect("CourseChange", MakeCallback(&CourseChange));
```

事实上，`Config::ConnectWithoutContext` 和 `Config::Connect` 的内部代码实际上找到了 `Ptr<Object>`，并在最低级别调用了适当的 `TraceConnect` 方法。

**Config 函数**接受一个表示对象指针链的路径。路径的每个部分对应于一个对象属性。最后一部分是感兴趣的属性，而之前的部分必须被类型化为包含或查找对象。

Config 代码解析并“遍历”此路径，直到到达路径的最终部分。然后，它将最后一部分解释为在遍历路径时找到的最后一个对象上的属性。然后，Config 函数调用最终对象上的适当 TraceConnect 或 TraceConnectWithoutContext 方法。让我们稍微详细看一下上面路径在遍历时发生了什么。

路径中的前导“/”字符是所谓的命名空间。配置系统中的一个==预定义命名空间是 “NodeList”，它是模拟中所有节点的列表==。列表中的项通过索引引用，因此==“/NodeList/7”是指模拟期间创建的节点列表中的第八个节点==（请记住索引从0开始）。此引用实际上是 `Ptr<Node>`，因此是 `ns3::Object` 的子类。

正如在 ns-3 手册的对象模型部分中所述，我们广泛使用对象聚合（即：可变继承）。这使我们能够在不构建复杂的继承树或预先决定哪些对象将成为节点的一部分的情况下，在不同对象之间建立关联。聚合中的每个对象都可以从其他对象中访问。

在我们的例子中，正在遍历的下一个路径段以 “$” 字符开头。这告诉配置系统，该段是对象类型的名称，因此应该进行一个 GetObject 调用来查找该类型。事实证明，在 `third.cc` 中使用的 `MobilityHelper` 安排了将移动模型与每个无线节点关联，或聚合。当添加“$”时，您正在请求另一个先前已经聚合的对象。您可以将其视为将指针从由“/NodeList/7”指定的原始 `Ptr<Node>` 切换到其关联的移动模型，其类型为 `ns3::MobilityModel`。如果您熟悉 `GetObject`，我们要求系统执行以下操作：

```cpp
Ptr<MobilityModel> mobilityModel = node->GetObject<MobilityModel>()
```

现在我们已经到达路径中的最后一个对象，因此我们将注意力转向该对象的属性。`MobilityModel` 类定义了一个名为 “CourseChange”的属性。您可以通过查看 `src/mobility/model/mobility-model.cc` 中的源代码，并在您喜欢的编辑器中搜索 “CourseChange” 来查看这一点。您应该会找到：

```cpp
.AddTraceSource("CourseChange",
                "The value of the position and/or velocity vector changed",
                MakeTraceSourceAccessor(&MobilityModel::m_courseChangeTrace),
                "ns3::MobilityModel::CourseChangeCallback")
```

这在这一点上看起来应该非常熟悉。

如果查找 mobility-model.h 文件中底层跟踪变量的相应声明，您将找到：

```cpp
TracedCallback<Ptr<const MobilityModel>> m_courseChangeTrace;
```

类型声明 `TracedCallback` 将 `m_courseChangeTrace` 标识为一种特殊的回调列表，可以使用上述 Config 函数进行连接。回调函数签名的 typedef 也在头文件中定义：

```cpp
typedef void (* CourseChangeCallback)(Ptr<const MobilityModel> * model);
```

`MobilityModel` 类被设计为一个基类，为所有具体子类提供一个共同的接口。如果您向下搜索文件直到底部，您将看到一个名为 `NotifyCourseChange()` 的方法：

```cpp
void
MobilityModel::NotifyCourseChange() const
{
  m_courseChangeTrace(this);
}
```

派生类将在每次进行航向变更以支持跟踪时调用此方法。此方法调用底层的 `m_courseChangeTrace` 上的 `operator()`，这将依次调用所有注册的回调，调用所有通过调用 Config 函数注册了对跟踪源感兴趣的跟踪池。

因此，在我们查看的 `third.cc` 示例中，每当在安装的 `RandomWalk2dMobilityModel` 实例中进行航向变更时，将会有一个 `NotifyCourseChange()` 调用，该调用会调用到 `MobilityModel` 基类。
如上所示，这会调用 `m_courseChangeTrace` 上的 `operator()`，进而调用任何已注册的跟踪池。
在示例中，注册兴趣的唯一代码是提供 Config 路径的代码。因此，被连接的来自节点编号七的 `CourseChange` 函数将是唯一调用的回调。

谜题的最后一块是“上下文”。回想一下，我们在 `third.cc` 中看到了类似以下的输出：

```
/NodeList/7/$ns3::MobilityModel/CourseChange x = 7.27897, y =
2.22677
```

输出的第一部分是上下文。它只是配置代码通过它找到跟踪源的路径。在我们一直在看的这个例子中，系统中可能有任意数量的跟踪源，对应于任意数量的具有移动模型的节点。需要某种方式来标识实际触发回调的跟踪源。简单的方法是使用 `Config::Connect` 连接，而不是 `Config::ConnectWithoutContext`。
### Finding Sources
新用户使用跟踪系统时不可避免地会遇到的第一个问题是：“好的，我知道模拟核心中必须有跟踪源，但我==怎样才能找出对我可用的跟踪源有哪些？==”

第二个问题是：“好的，我找到了一个跟踪源，我==怎样才能弄清楚连接到它时要使用的 Config 路径？==”

第三个问题是：“好的，我找到了一个跟踪源和 Config 路径，我怎样==才能弄清楚我的回调函数的返回类型和形式参数需要是什么==？”

第四个问题是：“好的，我键入了所有这些并获得了这个令人难以置信的奇怪错误消息，这究竟是什么意思？”

我们将依次回答这些问题。
### Available Sources
好的，我知道模拟核心中必须有跟踪源，但我怎样才能找出对我可用的跟踪源有哪些？

第一个问题的答案可以在 ns-3 的 API 文档中找到。如果您转到项目网站 [ns-3 project](https://www.nsnam.org/)，您将在导航栏中找到一个指向“Documentation”的链接。如果选择此链接，您将被带到我们的文档页面。有一个指向“Latest Release”的链接，将带您到 ns-3 的最新稳定版本的文档。如果选择“API Documentation”链接，您将被带到 ns-3 API 文档页面。

在侧边栏中，您应该看到一个层次结构，以以下内容开头：

- ns-3
  - ns-3 Documentation
    - All TraceSources
    - All Attributes
    - All GlobalValues

我们在这里关心的列表是“All TraceSources”（所有跟踪源）。请单击该链接。您将看到可能并不令人惊讶的是，ns-3 中所有跟踪源的列表。

举个例子，向下滚动到 ns3::MobilityModel。您将找到一个条目：

- CourseChange: The value of the position and/or velocity vector changed

您应该认识到这是我们在 `third.cc` 示例中使用的跟踪源。浏览此列表将是有帮助的。
### Config Paths
好的，我找到了一个跟踪源，但我怎样找出连接到它时要使用的 Config 路径呢？

如果您知道您感兴趣的对象，该类的“Detailed Description”（详细说明）部分将列出所有可用的跟踪源。例如，从“All TraceSources”列表开始，点击 ns3::MobilityModel 链接，将带您到 MobilityModel 类的文档。在页面的顶部几乎是一个类的简要描述，最后一个链接是“More…”（更多…）。点击此链接以跳过 API 摘要并进入类的“Detailed Description”（详细说明）。在描述的末尾，将有（最多）三个列表：

- Config Paths（配置路径）：此类的典型配置路径列表。
- Attributes（属性）：此类提供的所有属性列表。
- TraceSources（跟踪源）：此类提供的所有跟踪源列表。

首先我们来讨论 Config 路径。

让我们假设您刚刚在“All TraceSources”列表中找到了“CourseChange”跟踪源，而且您想弄清楚如何连接到它。您知道您正在使用（同样是从 third.cc 示例中）一个 ns3::RandomWalk2dMobilityModel。因此，单击“所有跟踪源”列表中的类名，或在“类列表”中找到 ns3::RandomWalk2dMobilityModel。无论哪种方式，您现在应该看到“ns3::RandomWalk2dMobilityModel Class Reference”页面。

如果您现在向下滚动到“Detailed Description”部分，在类方法和属性的摘要列表之后（或者只需点击页面顶部的类简要描述结束处的“More…”链接），您将看到该类的整体文档。继续向下滚动，找到“Config Paths”列表：

```
Config Paths

ns3::RandomWalk2dMobilityModel is accessible through the following paths with Config::Set and Config::Connect:

“/NodeList/[i]/$ns3::MobilityModel/$ns3::RandomWalk2dMobilityModel”
```

文档告诉您如何获取 RandomWalk2dMobilityModel 对象。将上面的字符串与我们在示例代码中实际使用的字符串进行比较：

```
"/NodeList/7/$ns3::MobilityModel"
```

差异是由于在文档中找到的字符串中隐含了两个 GetObject 调用。第一个是对 $ns3::MobilityModel 的 GetObject 调用，将查询基类的聚合。第二个隐含的 GetObject 调用是对 $ns3::RandomWalk2dMobilityModel 的调用，用于将基类转换为具体的实现类。文档为您展示了这两个操作。实际上，您正在寻找的实际跟踪源位于基类中。

在“Detailed Description”部分进一步查找跟踪源列表。您将找到：

```
TraceSources defined in parent class ``ns3::MobilityModel``

CourseChange: The value of the position and/or velocity vector changed.

Callback signature: ns3::MobilityModel::CourseChangeCallback
```

这正是您需要知道的。感兴趣的跟踪源位于 ns3::MobilityModel 中（这一点您早就知道）。API 文档的有趣之处在于，它告诉您在配置路径中不需要那个额外的转换，以获得具体的类，因为跟踪源实际上在基类中。因此，不需要额外的 GetObject，您只需使用路径：

```
"/NodeList/[i]/$ns3::MobilityModel"
```

这与示例路径完全匹配：

```
"/NodeList/7/$ns3::MobilityModel"
```

另外，找到配置路径的另一种方法是在 ns-3 代码库中查找已经找到它的人。在开始编写自己的代码之前，您应该尽量复制他人的工作代码。尝试执行类似以下的命令：

```
$ find . -name '*.cc' | xargs grep CourseChange | grep Connect
```

您可能会找到您的答案以及可工作的代码。例如，在此情况下，src/mobility/examples/main-random-topology.cc 中正好有等待您使用的代码：

```
Config::Connect("/NodeList/*/$ns3::MobilityModel/CourseChange",
                MakeCallback(&CourseChange));
```

我们稍后将回到这个示例。
### Callback Signatures
好的，我找到了一个跟踪源，还找到了 Config 路径，那么我如何确定回调函数的返回类型和形式参数应该是什么呢？

最简单的方法是查看跟踪源在类的“Detailed Description”（详细说明）中给出的回调签名 typedef。

重复一下 ns3::RandomWalk2dMobilityModel 中的 “CourseChange” 跟踪源条目：

```
CourseChange: The value of the position and/or velocity vector changed.

Callback signature: ns3::MobilityModel::CourseChangeCallback
```

回调签名是一个指向相关 typedef 的链接，其中我们找到：

```
typedef void (* CourseChangeCallback)(std::string context, Ptr<const MobilityModel> * model);
```

TracedCallback 课程更改通知的签名。

如果使用 ConnectWithoutContext 函数连接回调，则从签名中省略上下文参数。

参数：

- [in] context：Trace 源提供的上下文字符串。
- [in] model：正在更改课程的 MobilityModel。
与上述相同，为了看到这个在 ns-3 代码库中的使用示例，使用类似以下的命令：

```
$ find . -name '*.cc' | xargs grep CourseChange | grep Connect
```

您可能会找到您的答案以及可工作的代码。例如，在此情况下，src/mobility/examples/main-random-topology.cc 中正好有等待您使用的代码：

```
static void
CourseChange(std::string context, Ptr<const MobilityModel> model)
{
  ...
}
```

请注意，此函数：

- 接受一个“context”字符串参数，稍后我们将详细描述（如果使用 ConnectWithoutContext 函数连接回调，则会省略上下文参数）。
- 将 MobilityModel 作为最后一个参数提供（如果使用 ConnectWithoutContext，则作为唯一参数）。
- 返回 void。

如果碰巧回调签名没有被记录，并且没有可供参考的示例，从源代码中确定正确的回调函数签名可能会有点具有挑战性。

在展开代码漫游之前，我将友好地告诉您一个找到它的简单方法：您的回调的返回值将始终为 void。

可以从声明中的模板参数列表中找到 TracedCallback 的形式参数列表。回顾一下我们当前的示例，这是在 mobility-model.h 中找到的：

```
TracedCallback<Ptr<const MobilityModel>> m_courseChangeTrace;
```

在==声明中的模板参数列表==和==回调函数的形式参数==之间存在一对一的对应关系。在这里，有一个模板参数，即 Ptr\<const MobilityModel>。这告诉您需要一个返回 void 的函数，并且带有 Ptr\<const MobilityModel>。例如：

```
void
CourseChange(Ptr<const MobilityModel> model)
{
  ...
}
```

如果您想要 Config::ConnectWithoutContext，这就是您需要的全部内容。如果您想要一个上下文，您需要 Config::Connect 并使用一个带有字符串上下文的 Callback 函数，然后模板参数为：

```
void
CourseChange(std::string context, Ptr<const MobilityModel> model)
{
  ...
}
```

如果您想要确保您的 CourseChangeCallback 函数仅在您的本地文件中可见，可以添加 static 关键字，并得到：

```
static void
CourseChange(std::string path, Ptr\<const MobilityModel> model)
{
  ...
}
```

这正是我们在 third.cc 示例中使用的。
#### Implementation
这一部分是完全可选的。对于那些对模板细节不熟悉的人来说，可能会有些困难。然而，如果你能理解这一部分，你将对很多 ns-3 低级技巧有很好的掌握。

所以，再次，让我们弄清楚“CourseChange”追踪源需要的回调函数签名是什么。这可能会有些痛苦，但你只需要做一次。通过这一部分后，你就能够看一个 TracedCallback 并理解它了。

我们首先需要查看追踪源的声明。回想一下，这是在 mobility-model.h 中，我们先前找到过：

```cpp
TracedCallback<Ptr<const MobilityModel>> m_courseChangeTrace;
```

这是一个模板的声明。模板参数在尖括号内，所以我们真正关心的是 TracedCallback<> 是什么。如果你绝对不知道它可能在哪里，grep 是你的好朋友。

我们可能会对 ns-3 源代码的某些声明感兴趣，所以首先进入 src 目录。然后，我们知道这个声明必须在某种头文件中，所以只需使用以下命令进行 grep：

```bash
$ find . -name '*.h' | xargs grep TracedCallback
```

你会看到 303 行的输出（我通过 wc 管道查看了一下）。尽管这可能看起来很多，但实际上并不是很多。只需将输出通过 more 管道，开始浏览。在第一页上，你将看到一些非常可疑的模板样式。

```cpp
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::TracedCallback()
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::ConnectWithoutContext(c ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::Connect(const CallbackB ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::DisconnectWithoutContext ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::Disconnect(const Callba ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::operator()() const ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::operator()(T1 a1) const ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::operator()(T1 a1, T2 a2 ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::operator()(T1 a1, T2 a2 ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::operator()(T1 a1, T2 a2 ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::operator()(T1 a1, T2 a2 ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::operator()(T1 a1, T2 a2 ...
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::operator()(T1 a1, T2 a2 ...
```

事实证明，所有这些都来自头文件 traced-callback.h，听起来很有希望。然后你可以查看 mobility-model.h，看到一行确认了这个猜测：

```cpp
#include "ns3/traced-callback.h"
```

当然，你也可以从另一个方向开始，看 mobility-model.h 中的 includes，并注意到 traced-callback.h 的 include，推断这一定是你想要的文件。

在任一种情况下，下一步是在你喜欢的编辑器中查看 src/core/model/traced-callback.h，看看发生了什么。

你会在文件顶部看到一个注释，应该让你感到欣慰：

```cpp
// An ns3::TracedCallback has almost exactly the same API as a normal ns3::Callback
// but instead of forwarding calls to a single function (as an ns3::Callback normally does),
// it forwards calls to a chain of ns3::Callback.
```

这应该听起来非常熟悉，让你知道你在正确的轨道上。

在这个注释后面，你会找到：

```cpp
template<typename T1 = empty, typename T2 = empty,
         typename T3 = empty, typename T4 = empty,
         typename T5 = empty, typename T6 = empty,
         typename T7 = empty, typename T8 = empty>
class TracedCallback
{
  ...
```

这告诉你 TracedCallback 是一个模板类。它有八个可能的类型参数，都有默认值。回去比较一下这个与你试图理解的声明：

```cpp
TracedCallback<Ptr<const MobilityModel>> m_courseChangeTrace;
```

在模板化的类声明中，typename T1 对应于上面声明中的 Ptr\<const MobilityModel>。所有其他类型参数都保持默认值。查看构造函数实际上并不能告诉你太多。唯一一个你看到过连接你的回调函数和追踪系统的地方是在 Connect 和 ConnectWithoutContext 函数中。如果你向下滚动，你会在这里找到一个 ConnectWithoutContext 方法：

```cpp
template<typename T1, typename T2,
         typename T3, typename T4,
         typename T5, typename T6,
         typename T7, typename T8>
void
TracedCallback<T1,T2,T3,T4,T5,T6,T7,T8>::ConnectWithoutContext ...
{
  Callback<void,T1,T2,T3,T4,T5,T6,T7,T8> cb;
  cb.Assign(callback);
  m_callbackList.push_back(cb);
}
```

你现在进入了这个大坑。当为上面的声明实例化模板时，编译器将使用 Ptr\<const MobilityModel> 替换 T1。

```cpp
void
TracedCallback<Ptr<const MobilityModel>>::ConnectWithoutContext ... cb
{
  Callback<void, Ptr<const MobilityModel>> cb;
  cb.Assign(callback);
  m_callbackList.push_back(cb);
}
```

你现在可以看到我们一直在讨论的所有内容的实现。该代码创建了一个正确类型的回调并将你的函数分配给它。这相当于我们在本节开头讨论的 `pfi = MyFunction`。然后，代码将回调添加到此源的回调列表中。唯一剩下的就是查看 Callback 的定义。使用与我们用来查找 TracedCallback 的相同 grep 技巧，你将能够找到文件 `./core/callback.h`。

如果你仔细查看文件，你会看到很多可能几乎难以理解的模板代码。然而，最终你会找到一些 Callback 模板类的 API 文档。幸运的是，这里有一些英语：

```cpp
// Callback template class.
// This class template implements the Functor Design Pattern. It is used to declare the type of a Callback:
// the first non-optional template argument represents the return type of the callback.
// the remaining (optional) template arguments represent the type of the subsequent arguments to the callback.
// up to nine arguments are supported.
```

我们正在尝试弄清楚

```cpp
Callback<void, Ptr<const MobilityModel>> cb;
```

声明的含义。现在我们有能力理解，第一个（非可选）模板参数 `void` 表示回调的返回类型。第二个（可选）模板参数 `Ptr<const MobilityModel>` 表示回调的第一个参数的类型。
### TracedValues
在本节的早些时候，我们呈现了一段简单的代码，使用了 `TracedValue<int32_t>` 来演示追踪代码的基础知识。我们只是简要地介绍了 `TracedValue` 究竟是什么，以及如何找到回调的返回类型和形式参数。

正如我们所提到的，==文件 `traced-value.h` 引入了用于追踪遵循值语义的数据的必要声明==。通常，==值语义只是意味着你可以传递对象本身，而不是传递对象的地址==。我们将此要求扩展到包括预定义的普通数据（POD）类型的完整集合的赋值样式运算符：

- `operator=`（赋值）
- `operator*=` `operator/=` `operator+=` `operator-=` `operator++`（前缀和后缀） `operator--`（前缀和后缀） `operator<<=` `operator>>=` `operator&=` `operator|=` `operator%=` `operator^=`

这实际上意味着你将能够追踪使用上述运算符对具有值语义的 C++ 对象进行的所有更改。

我们在上面看到的 `TracedValue<>` 声明提供了重载上述运算符并驱动回调过程的基础设施。使用 `TracedValue` 中的任何上述运算符时，它都会提供该变量的旧值和新值，本例中为 int32_t 值。通过检查 `TracedValue` 的声明，我们知道追踪接收函数将有参数 `(int32_t oldValue, int32_t newValue)`。`TracedValue` 回调函数的返回类型始终为 `void`，因此接收函数 `traceSink` 的预期回调签名将是：

```cpp
void (* traceSink)(int32_t oldValue, int32_t newValue);
```

在 `GetTypeId` 方法中的 `.AddTraceSource` 提供了通过 Config 系统将追踪源连接到外部世界的“钩子”。我们已经讨论了 `AddTraceSource` 的前三个参数：Config 系统的属性名称、帮助字符串和 `TracedValue` 类数据成员的地址。

示例中的最后一个字符串参数，例如“ns3::TracedValueCallback::Int32”，是回调函数签名的 typedef 名称。我们要求这些签名要被定义，并向 `AddTraceSource` 提供完全限定的类型名称，以便 API 文档可以将追踪源与函数签名链接起来。对于 `TracedValue`，签名很简单；对于 `TracedCallback`，我们已经看到 API 文档真的很有帮助。
### Real Example
让我们从其中一本关于 TCP 最知名的书中进行一个例子。“TCP/IP Illustrated, Volume 1: The Protocols” 由 W. Richard Stevens 撰写，是一本经典之作。我随意翻开书，发现了第 366 页上关于拥塞窗口和序列号随时间变化的一个很好的图表。Stevens 将其称为“Figure 21.10. Value of cwnd and send sequence number while data is being transmitted.” 让我们使用 ns-3 的追踪系统和 gnuplot 来重新创建该图中 cwnd 部分。
### Available Sources
首先要考虑的是我们希望如何获取数据。我们需要追踪什么？因此，让我们查看“所有追踪源”列表，看看我们有什么可以使用的。回顾一下，这可以在 ns-3 API 文档中找到。如果您浏览列表，最终会找到：

ns3::TcpSocketBase

- CongestionWindow: TCP 连接的拥塞窗口
- SlowStartThreshold: TCP 慢启动阈值（字节）

事实证明，ns-3 的 TCP 实现主要位于文件 src/internet/model/tcp-socket-base.cc，而拥塞控制变体位于文件如 src/internet/model/tcp-bic.cc。如果您事先不知道这一点，您可以使用递归 grep 的技巧：

```bash
$ find . -name '*.cc' | xargs grep -i tcp
```

您将找到一页又一页的 tcp 实例，指向这个文件。

打开 TcpSocketBase 的类文档，跳到 TraceSources 列表，您会发现：

**TraceSources**

- **CongestionWindow:** TCP 连接的拥塞窗口
- **Callback signature:** ns3::TracedValueCallback::Uint32

点击回调 typedef 链接，我们看到您现在知道要期望的签名：

```cpp
typedef void(* ns3::TracedValueCallback::Int32)(int32_t oldValue, int32_t newValue)
```

现在，您应该完全理解这段代码。==如果我们有指向 TcpSocketBase 对象的指针，我们可以连接到“CongestionWindow”追踪源==，如果提供了适当的回调目标。这与我们在本节开始时在简单示例中看到的追踪源相同，只是这次我们讨论的是 uint32_t 而不是 int32_t。而且我们知道必须提供具有该签名的回调函数。
### Finding Examples
最好总是尝试找到现有的代码，然后进行修改，而不是从头开始。因此，现在的首要任务是找到已经连接到“CongestionWindow”追踪源的一些代码，并看看我们是否可以修改它。像往常一样，grep 是您的好朋友：

```bash
$ find . -name '*.cc' | xargs grep CongestionWindow
```

这将指出一些有希望的候选项：examples/tcp/tcp-large-transfer.cc 和 src/test/ns3tcp/ns3tcp-cwnd-test-suite.cc。

我们还没有访问任何测试代码，所以让我们先看看那里。通常会发现测试代码相当简洁，因此这可能是一个非常好的选择。在您喜欢的编辑器中打开 src/test/ns3tcp/ns3tcp-cwnd-test-suite.cc 并搜索“CongestionWindow”。您会找到以下内容：

```cpp
ns3TcpSocket->TraceConnectWithoutContext("CongestionWindow",
    MakeCallback(&Ns3TcpCwndTestCase1::CwndChange, this));
```

这应该让您感到非常熟悉。我们在上面提到过，如果我们有一个指向 TcpSocketBase 的指针，我们可以连接到“CongestionWindow”追踪源。这正是我们在这里所拥有的；所以事实证明，这行代码正是我们想要的。让我们继续从这个函数（Ns3TcpCwndTestCase1::DoRun()）中提取我们需要的代码。如果您查看这个函数，会发现它看起来就像一个 ns-3 脚本。事实证明，它确实就是。这是测试框架运行的脚本，因此我们只需将其提取出来，然后在 main 中进行包装，而不是在 DoRun 中进行包装。与其一步一步地走过这一步，不如直接提供将此测试还原为本机 ns-3 脚本的文件 - examples/tutorial/fifth.cc。
### Dynamic Trace Sources
第五个示例 fifth.cc 展示了在使用任何类型的追踪源之前，您必须理解的一条非常重要的规则：在尝试使用 Config::Connect 命令之前，您必须确保连接的目标存在。这与说对象必须在尝试调用它之前实例化没有什么不同。尽管以这种方式陈述时这似乎很明显，但对于第一次尝试使用该系统的许多人来说，这确实是一个常见的问题。

让我们回到基础知识。在任何 ns-3 脚本中，存在三个基本的执行阶段:
第一个阶段有时被称为“配置时间”或“设置时间”，存在于您的脚本的主函数运行时，但在调用 Simulator::Run 之前。
第二个阶段有时被称为“模拟时间”，存在于 Simulator::Run 主动执行其事件的时间段内。在完成模拟执行后，Simulator::Run 将控制权返回给主函数。
当这发生时，脚本进入可以称为“拆卸阶段”的阶段，此阶段是在设置期间创建的结构和对象被拆卸和释放的时间。

==在尝试使用追踪系统时可能最常见的错误是：假设在模拟时间动态构建的实体在配置时间可用。特别是，ns-3 Socket 是一个通常由应用程序创建的动态对象，用于在节点之间进行通信。ns-3 应用程序始终与其关联的“启动时间”和“停止时间”。在绝大多数情况下，应用程序将不会尝试在其“启动时间”调用其 StartApplication 方法之前创建动态对象。这是为了确保在应用程序尝试执行任何操作之前，模拟已完全配置（在配置时间期间尝试连接到尚不存在的节点会发生什么？）。因此，在配置阶段，如果它们中的一个在模拟期间动态创建，则不能将追踪源连接到追踪接收器。==

解决这个难题的两种方法是：

1. 创建一个==在动态对象创建后运行的模拟器事件，并在执行该事件时挂钩追踪==；
2. 在==配置时间创建动态对象，然后在此时挂钩追踪，并在模拟时间将该对象提供给系统使用==。

在 fifth.cc 示例中，我们**采用了第二种方法**。这个决策要求我们创建 TutorialApp 应用程序，其唯一目的是将 Socket 作为参数。
### Walkthrough: `fifth.cc`
现在，让我们来看一下我们通过==解剖拥塞窗口测试构建==的示例程序。在您喜欢的编辑器中打开 examples/tutorial/fifth.cc。您应该会看到一些看起来很熟悉的代码：

```cpp
/* -*- Mode:C++; c-file-style:"gnu"; indent-tabs-mode:nil; -*- */
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
 * Foundation, Include., 59 Temple Place, Suite 330, Boston, MA  02111-1307  USA
 */

#include "tutorial-app.h"

#include "ns3/applications-module.h"
#include "ns3/core-module.h"
#include "ns3/internet-module.h"
#include "ns3/network-module.h"
#include "ns3/point-to-point-module.h"

#include <fstream>

using namespace ns3;

NS_LOG_COMPONENT_DEFINE("FifthScriptExample");
```

接下来的源代码是网络示意图以及一个解释上述关于 Socket 的问题的注释。

```cpp
// ===========================================================================
//
//         node 0                 node 1
//   +----------------+    +----------------+
//   |    ns-3 TCP    |    |    ns-3 TCP    |
//   +----------------+    +----------------+
//   |    10.1.1.1    |    |    10.1.1.2    |
//   +----------------+    +----------------+
//   | point-to-point |    | point-to-point |
//   +----------------+    +----------------+
//           |                     |
//           +---------------------+
//                5 Mbps, 2 ms
//
//
// We want to look at changes in the ns-3 TCP congestion window.  We need
// to crank up a flow and hook the CongestionWindow attribute on the socket
// of the sender.  Normally one would use an on-off application to generate a
// flow, but this has a couple of problems.  

// First, the socket of the on-off application is not created until Application Start time, so we wouldn't be
// able to hook the socket (now) at configuration time.  
回顾之前提到的顺序问题：配置时间 - 模拟时间 - 拆卸时间
这里的意思是应用程序的使用是在模拟时间，在此之前需要完成对应的配置时间内容

// Second, even if we could arrange a call after start time, the socket is not public so we
// couldn't get at it.
即使开始后能发出指令，由于socket的特异性，我们不一定能在指定端口收到calling

// So, we can cook up a simple version of the on-off application that does what
// we want.  On the plus side we don't need all of the complexity of the on-off
// application.  On the minus side, we don't have a helper, so we have to get
// a little more involved in the details, but this is trivial.
//
// So first, we create a socket and do the trace connect on it; then we pass
// this socket into the constructor of our simple application which we then
// install in the source node.
// ===========================================================================
```

这也应该是不言自明的。

ns-3 的早期版本在此程序中==使用了名为 MyApp 的自定义应用程序==。当前版本的 ns-3 将此移动到了一个单独的头文件（`tutorial-app.h`）和实现文件（`tutorial-app.cc`）中。==这个简单的应用程序允许在配置时间创建 `Socket`。==

```cpp
/**
 * Tutorial - a simple Application sending packets.
 */
class TutorialApp : public Application // 拥塞窗口的搭建
{
  public:
    TutorialApp();
    ~TutorialApp() override;
    /**
     * Register this type.
     * \return The TypeId.
     */
     
    static TypeId GetTypeId();

    /**
     * Setup the socket（设置 socket_API 的信息）
     * \param socket： The socket.
     * \param address： The destination address.
     * \param packetSize： The packet size to transmit.
     * \param nPackets： The number of packets to transmit.
     * \param dataRate： the data rate to use.
     */
    void Setup(Ptr<Socket> socket,
               Address address,
               uint32_t packetSize,
               uint32_t nPackets,
               DataRate dataRate);

  private:
    void StartApplication() override;
    void StopApplication() override;

    /// Schedule a new transmission.
    void ScheduleTx();
    /// Send a packet.
    void SendPacket();

    Ptr<Socket> m_socket;   //!< The transmission socket.
    Address m_peer;         //!< The destination address.
    uint32_t m_packetSize;  //!< The packet size.
    uint32_t m_nPackets;    //!< The number of packets to send.
    DataRate m_dataRate;    //!< The data rate to use.
    EventId m_sendEvent;    //!< Send event.
    bool m_running;         //!< True if the application is running.
    uint32_t m_packetsSent; //!< The number of packets sent.
};
```

您可以看到这个类继承自 ns-3 的 Application 类。如果您对继承的内容感兴趣，可以查看 `src/network/model/application.h`。TutorialApp 类必须重写 StartApplication 和 StopApplication 方法。这些方法在模拟期间需要 TutorialApp 开始和停止发送数据时自动调用。
### Starting/Stopping Applications
这段内容涉及了 ==ns-3 模拟器中事件如何启动的一些细节==。

如果您不打算深入了解 ns-3 的内部工作原理，可以忽略这一部分。然而，==如果您计划深入开发新的模型，了解这一部分可能是有益的==。

在 ns-3 脚本中，最常见的启动事件的方式之一是启动一个 Application。这是通过以下（希望熟悉的）ns-3 脚本代码实现的：

```cpp
ApplicationContainer apps = ...
apps.Start(Seconds(1.0));
apps.Stop(Seconds(10.0));
```

Application 容器代码（如果您感兴趣，可以查看 `src/network/helper/application-container.h`）遍历其包含的所有应用程序，并调用：

```cpp
app->SetStartTime(startTime);
```

作为 `apps.Start` 调用的结果，以及

```cpp
app->SetStopTime(stopTime);
```

作为 `apps.Stop` 调用的结果。

这些调用的最终结果是我们希望模拟器在适当的时间自动调用我们的应用程序，告诉它们何时启动和停止。对于 TutorialApp，它继承自 Application 类并重写了 `StartApplication` 和 `StopApplication` 方法。这些方法是由模拟器在适当的时间自动调用的函数。在 TutorialApp 的情况下，您会发现 `TutorialApp::StartApplication` 执行了对套接字的初始绑定和连接，然后通过调用 `TutorialApp::SendPacket` 开始了数据流。`TutorialApp::StopApplication` 通过取消任何待处理的发送事件停止生成数据包，然后关闭套接字。

ns-3 的一个好处是您可以完全忽略应用程序在何时“自动”由模拟器在正确时间调用的实现细节。但是，既然我们已经深入到 ns-3 的内部，那就继续吧。

如果您查看 `src/network/model/application.cc`，您会发现 Application 的 `SetStartTime` 方法只是设置了成员变量 `m_startTime`，而 `SetStopTime` 方法只是设置了 `m_stopTime`。然后，如果没有一些提示，这里的踪迹可能会中断。

重新追踪的关键是知道有一个全局列表，其中包含系统中的所有节点。每当在模拟中创建一个节点时，该节点的指针就会添加到全局 NodeList。

查看 `src/network/model/node-list.cc`，并搜索 NodeList::Add。公共静态实现调用了一个名为 `NodeListPriv::Add` 的私有实现。这是 ns-3 中一个相对常见的习惯用法。因此，查看 `NodeListPriv::Add`。在这里，您会发现：

```cpp
Simulator::ScheduleWithContext(index, TimeStep(0), &Node::Initialize, node);
```

这告诉我们，每当在模拟中创建一个节点时，作为副作用，将为该节点调度一次 `Initialize` 方法的调用，该调用在时间为零时发生。不要对这个名字读太多，它并不意味着节点将要开始做什么，它可以解释为对节点的一种信息性调用，告诉它模拟已经开始，而不是一种动作性调用，告诉节点开始执行某些操作。

因此，`NodeList::Add`间接地为时间零调度了一个对 `Node::Initialize` 的调用，以向新添加的节点告知模拟已经开始。如果您查看 `src/network/model/node.h`，您将在其中找不到名为 `Node::Initialize` 的方法。事实证明，`Initialize` 方法是从 `Object` 类继承的。系统中的所有对象都可以在模拟开始时得到通知，而 `Node` 类的对象只是这些对象中的一种。

接下来，查看 `src/core/model/object.cc` 并搜索 `Object::Initialize`。由于 ns-3 对象支持聚合，该代码并不像您可能期望的那样直接。`Object::Initialize` 中的代码然后遍历了所有已聚合在一起的对象，并调用它们的 `DoInitialize` 方法。这是 ns-3 中非常常见的一种习惯用法，有时被称为“模板设计模式”：公共非虚拟 API 方法，跨实现保持不变，并调用一个私有虚拟实现方法，由子类继承和实现。这些名称通常是像 `MethodName` 这样的公共 API，以及 `DoMethodName` 这样的私有 API。

这告诉我们应该在 `src/network/model/node.cc` 中寻找 `Node::DoInitialize` 方法，以继续我们的踪迹。如果您定位到代码，您将找到一个方法，该方法循环遍历节点中的所有设备，然后调用 `device->Initialize` 和 `application->Initialize` 分别。

您可能已经知道，`Device` 和 `Application` 类都是从 `Object` 类继承的，所以下一步将是查看当调用 `Application::DoInitialize` 时会发生什么。查看 `src/network/model/application.cc`，您将找到：

```cpp
void
Application::DoInitialize()
{
  NS_LOG_FUNCTION(this);
  m_startEvent = Simulator::Schedule(m_startTime, &Application::StartApplication, this);
  if (m_stopTime != TimeStep(0))
    {
      m_stopEvent = Simulator::Schedule(m_stopTime, &Application::StopApplication, this);
    }
  Object::DoInitialize();
}
```

到此为止，我们终于到达

了踪迹的尽头。如果您一直保持清晰，当您实现一个 ns-3 Application 时，您的新应用程序继承自 `Application` 类。您重写了 `StartApplication` 和 `StopApplication` 方法，并提供了启动和停止从您的新应用程序流出数据的机制。当在模拟中创建一个节点时，它被添加到一个全局 NodeList。将节点添加到此 NodeList 的操作导致为您的新节点调度一个事件，该事件将在模拟开始时调用该节点的 `Initialize` 方法。由于节点继承自 `Object`，因此当模拟开始时调用了该节点的 `Object::Initialize` 方法，进而调用了聚合到节点的所有对象（例如移动模型）的 `DoInitialize` 方法。由于节点对象已经重写了 `DoInitialize`，在模拟开始时调用了该方法。`Node::DoInitialize` 方法调用节点上所有应用程序的 `Initialize` 方法。由于应用程序也是对象，因此这将导致调用 `Application::DoInitialize`。当调用 `Application::DoInitialize` 时，它为 `StartApplication` 和 `StopApplication` 调用安排了事件。这些调用旨在在模拟期间开始和停止数据从应用程序流出。

这是又一个相对较长的旅程，但这只需要一次，您现在理解了 ns-3 的另一个非常深层次的部分。
### The TutorialApp Application
`TutorialApp` 应用程序当然需要一个构造函数和一个析构函数：

```cpp
TutorialApp::TutorialApp()
    : m_socket(nullptr),
      m_peer(),
      m_packetSize(0),
      m_nPackets(0),
      m_dataRate(0),
      m_sendEvent(),
      m_running(false),
      m_packetsSent(0)
{
}

TutorialApp::~TutorialApp()
{
    m_socket = nullptr;
}
```

下面这部分代码的存在正是我们最初编写这个 `Application` 的整个原因。

```cpp
void
TutorialApp::Setup(Ptr<Socket> socket,
                   Address address,
                   uint32_t packetSize,
                   uint32_t nPackets,
                   DataRate dataRate)
{
    m_socket = socket;
    m_peer = address;
    m_packetSize = packetSize;
    m_nPackets = nPackets;
    m_dataRate = dataRate;
}
```

这段代码应该相当容易理解。我们只是在==初始化成员变量==。从跟踪的角度来看，其中一个重要的变量是 `Ptr<Socket> socket`，在配置时我们需要将其提供给应用程序。回想一下，我们将创建 `Socket` 作为 `TcpSocket`（由 `TcpSocketBase` 实现），并在将其传递给 `Setup` 方法之前挂接其“CongestionWindow”跟踪源。

```cpp
void
TutorialApp::StartApplication()
{
    m_running = true;          // 开始创建模拟事件
    m_packetsSent = 0;         // 当前发包数是0
    m_socket->Bind();          // 在连接的本地端执行了所需的工作
    m_socket->Connect(m_peer); // 执行与 TCP 在 `Address m_peer` 上建立连接有关的操作
    SendPacket();
}
```

上述代码是覆盖实现的 `Application::StartApplication`，它将==由模拟器在适当的时间自动调用以启动我们的应用程序运行==。你可以看到它==执行了 `Socket Bind` 操作==。如果你熟悉伯克利套接字，这应该不会让你感到惊讶。==它在连接的本地端执行了所需的工作==，就像你可能期望的那样。==接下来的 `Connect` 将执行与 TCP 在 `Address m_peer` 上建立连接所需的操作==。

**现在应该清楚为什么我们需要将许多工作推迟到模拟时间，因为 `Connect` 需要一个完全运作的网络来完成**。在 `Connect` 之后，应用程序通过调用 `SendPacket` 开始创建模拟事件。

接下来的代码段向 `Application` 解释了如何停止创建模拟事件。

```cpp
void
TutorialApp::StopApplication()
{
    m_running = false;

    if (m_sendEvent.IsRunning()) // 说明该`Event` 处于等待执行或正在执行的状态
    {
        Simulator::Cancel(m_sendEvent); // 取消该事件，将其从模拟器事件队列中移
    }

    if (m_socket)
    {
        m_socket->Close();
    }
}

具体解释见下文：
```

每次安排模拟事件时，都会创建一个 `Event`：
- 如果 `Event` 处于*等待执行* 或*正在执行* 的状态，它的方法 `IsRunning` 将返回 `true`
- 在这段代码中，如果 `IsRunning()` 返回 true，我们会取消该事件，将其从模拟器事件队列中移除。
- 通过这样做，我们打破了 `Application` 用于持续发送 `Packets` 的事件链，使得 `Application` 变得静止。
- 在静止了 `Application` 之后，我们关闭套接字，从而拆除了TCP连接。

套接字实际上是在析构函数中删除的，当执行 `m_socket = 0` 时。这会删除对底层 `Ptr<Socket>` 的最后一个引用，从而调用该对象的析构函数。

回想一下，`StartApplication` 调用了 `SendPacket` 来启动描述 `Application` 行为的事件链。

```cpp
void
TutorialApp::SendPacket()
{
    Ptr<Packet> packet = Create<Packet>(m_packetSize);
    m_socket->Send(packet);

    if (++m_packetsSent < m_nPackets)
    {
        ScheduleTx();
    }
}
```

在这里，你看到 `SendPacket` 确实是这样做的。它创建了一个 `Packet`，然后执行了一个 `Send` 操作，如果你了解伯克利套接字，这可能是你预期看到的。

`Application` 的责任是继续安排事件链，因此接下来的代码调用 `ScheduleTx` 来安排另一个传输事件（`SendPacket`），直到 `Application` 决定已经发送足够的数据。

```cpp
void
TutorialApp::ScheduleTx()
{
    if (m_running)
    {
        Time tNext(Seconds(m_packetSize * 8 / static_cast<double>(m_dataRate.GetBitRate())));
        m_sendEvent = Simulator::Schedule(tNext, &TutorialApp::SendPacket, this);
    }
}
```

在这里，你可以看到 `ScheduleTx` 正是这样做的：
- 如果 `Application` 正在运行（如果 `StopApplication` 没有被调用），它将安排一个新的事件，该事件再次调用 `SendPacket`。
- 仔细的读者可能会注意到一些常常让新用户困扰的事情。`Application` 的数据速率就是那样。它与底层 `Channel` 的数据速率无关。这只是 `Application` 产生比特的速率。它**不考虑用于传输数据的各种协议或通道的任何开销**。如果将 `Application` 的数据速率设置为与底层 `Channel` 相同的数据速率，最终会导致缓冲区溢出。
### Trace Sinks
整个练习的==目的是通过TCP发出的跟踪回调来指示拥塞窗口已经更新==。下面的代码实现了相应的跟踪接收器：

```cpp
// 用于追踪TCP拥塞窗口变化的回调函数
static void
CwndChange(uint32_t oldCwnd, uint32_t newCwnd)
{
    NS_LOG_UNCOND(Simulator::Now().GetSeconds() << "\t" << newCwnd);
}
```

这段代码现在对你来说应该非常熟悉，所以我们不会深入讨论细节。==该函数在拥塞窗口发生变化时仅记录当前模拟时间和新的拥塞窗口值==。你可以想象，你==可以将生成的输出加载到图形程序（如gnuplot或Excel）中==，立即看到==随时间变化的拥塞窗口行为==的漂亮图表。

我们添加了一个新的跟踪接收器来显示数据包丢失的位置。我们将在这段代码中添加一个错误模型，所以我们希望演示这一过程的工作原理。

```cpp
static void
RxDrop(Ptr<const Packet> p) // 添加了一个新的跟踪接收器来显示数据包丢失的位置
{
    NS_LOG_UNCOND("RxDrop at " << Simulator::Now().GetSeconds());
}
```

这个跟踪接收器将连接到点对点NetDevice的“PhyRxDrop”跟踪源。当NetDevice的物理层丢弃一个数据包时，这个跟踪源就会触发。
- 如果你稍微查看一下源代码（src/point-to-point/model/point-to-point-net-device.cc），你会发现这个跟踪源指的是`PointToPointNetDevice::m_phyRxDropTrace`。然后，如果你在`src/point-to-point/model/point-to-point-net-device.h`中查找这个成员变量，你会发现它被声明为`TracedCallback<Ptr<const Packet>>`。这应该告诉你，回调目标应该是一个返回`void`的函数，并接受一个`Ptr<const Packet>`的单一参数（假设我们使用`ConnectWithoutContext`）——这正是我们上面所拥有的。
### Main Program
主函数开始时通过配置TCP类型为使用传统的 NewReno 拥塞控制变体，使用所谓的经典 TCP 丢失恢复机制。
当这个教程程序最初编写时，这些是默认的TCP配置，但随着时间的推移，ns3 TCP 已经发展到使用当前的Linux TCP 默认值，即“Cubic”和“Prr”的丢失恢复。
首先的语句还配置了命令行参数处理：
```cpp
int main(int argc, char* argv[])
{
    CommandLine cmd(__FILE__);
    cmd.Parse(argc, argv);

    // 使用 TCPNewReno 作为拥塞控制算法
    Config::SetDefault("ns3::TcpL4Protocol::SocketType", StringValue("ns3::TcpNewReno"));
    // 将TCP连接的 初始拥塞窗口 设置为： 1个数据包 ，并使用经典的 快速恢复算法 。
    Config::SetDefault("ns3::TcpSocket::InitialCwnd", UintegerValue(1));
    // 请注意，此配置仅用于演示如何在ns-3中配置TCP参数。否则，建议使用ns-3中TCP的默认设置。
    Config::SetDefault("ns3::TcpL4Protocol::RecoveryType",
                       TypeIdValue(TypeId::LookupByName("ns3::TcpClassicRecovery")));
   
    // 下面的代码现在对你来说应该非常熟悉：
    NodeContainer nodes;
    nodes.Create(2);

    PointToPointHelper pointToPoint;
    pointToPoint.SetDeviceAttribute("DataRate", StringValue("5Mbps"));
    pointToPoint.SetChannelAttribute("Delay", StringValue("2ms"));

    NetDeviceContainer devices;
    devices = pointToPoint.Install(nodes);

    // 这创建了两个节点，它们之间有一个点对点通道，就像文件开头的插图所示。
```
这是主函数中的一部分代码，用于配置TCP参数和创建网络拓扑。

>设计思路：人为设置错误以观察
>
>以下几行代码展示了一些新的东西：
如果我们跟踪一个表现完美的连接，我们最终会得到一个单调增加的拥塞窗口。
为了看到任何有趣的行为，我们真的希望引入链路错误，这将导致丢弃数据包，引发冗余的ACK并触发拥塞窗口的更有趣的行为。

ns-3提供了可以附加到通道的ErrorModel对象。
我们正在使用RateErrorModel，它允许我们以给定的速率向通道引入错误：
```cpp
Ptr<RateErrorModel> em = CreateObject<RateErrorModel>();
em->SetAttribute("ErrorRate", DoubleValue(0.00001));
devices.Get(1)->SetAttribute("ReceiveErrorModel", PointerValue(em));
```

上述代码实例化了一个 `RateErrorModel` 对象，并将 =="ErrorRate" 属性设置为所需的值==。然后，我们将实例化后的 `RateErrorModel` 设置为==点对点 NetDevice 使用的错误模型==。**这将导致一些重传，使我们的图表更加有趣**。

```cpp
InternetStackHelper stack;
stack.Install(nodes);

Ipv4AddressHelper address;
address.SetBase("10.1.1.0", "255.255.255.252");
Ipv4InterfaceContainer interfaces = address.Assign(devices);
```

上述代码应该很熟悉。它==在两个节点上安装了 Internet 栈，并为点对点设备创建接口并分配 IP 地址==。

```cpp
uint16_t sinkPort = 8080;
Address sinkAddress(InetSocketAddress(interfaces.GetAddress(1), sinkPort));
PacketSinkHelper packetSinkHelper("ns3::TcpSocketFactory",
                                  InetSocketAddress(Ipv4Address::GetAny(), sinkPort));
ApplicationContainer sinkApps = packetSinkHelper.Install(nodes.Get(1));
sinkApps.Start(Seconds(0.));
sinkApps.Stop(Seconds(20.));
```

这应该都很熟悉，除了：

```cpp
PacketSinkHelper packetSinkHelper("ns3::TcpSocketFactory",
                                  InetSocketAddress(Ipv4Address::GetAny(), sinkPort));
```

这段代码实例化了一个 `PacketSinkHelper`，并告诉它使用 `ns3::TcpSocketFactory` 类==创建套接字==。
这个类实现了一种称为“对象工厂”的设计模式，这是一种在抽象方式中指定用于创建对象的类的常用机制。
在这里，您不需要自己创建对象，而是提供给 `PacketSinkHelper` 一个字符串，该字符串指定用于创建对象的 TypeId 字符串，然后可以用来创建工厂创建的对象的实例。

其余的参数==告诉应用程序应该绑定到哪个地址和端口==。

```cpp
Ptr<Socket> ns3TcpSocket = Socket::CreateSocket(nodes.Get(0), TcpSocketFactory::GetTypeId());
ns3TcpSocket->TraceConnectWithoutContext("CongestionWindow", MakeCallback(&CwndChange));
```

第一条语句调用静态成员函数 `Socket::CreateSocket`，并==为创建套接字的对象工厂提供了一个节点和一个显式的 TypeId==。这是比上面的 `PacketSinkHelper` 调用稍微低级的调用，使用显式的 C++ 类型而不是由字符串引用的类型。==从概念上讲，它与上述调用是相同的==。

一旦 `TcpSocket` 被创建并附加到节点，我们就可以使用 `TraceConnectWithoutContext` 将 `CongestionWindow` ==跟踪源连接到我们的跟踪回调函数==。

回顾一下，我们编写了一个应用程序，**以便在配置时间结束时（`Simulator::Run` 被调用时）获取我们刚刚创建的 Socket（在配置时间内），并在模拟时间内使用它**。我们现在必须实例化该应用程序。我们没有费心==创建一个助手来管理应用程序==，所以我们必须“手动”创建和安装它。这实际上非常容易：

```cpp
Ptr<TutorialApp> app = CreateObject<TutorialApp>();                   // 建立一个管理应用程序的助手
app->Setup(ns3TcpSocket, sinkAddress, 1040, 1000, DataRate("1Mbps"));
nodes.Get(0)->AddApplication(app);  // 手动将 `TutorialApp` 应用程序添加到源节点
app->Start(Seconds(1.));
app->Stop(Seconds(20.));  // 告诉它何时开始和停止执行任务
```
- 第一行创建了一个类型为 `TutorialApp` 的对象 - 我们的应用程序。
- 第二行告诉应用程序使用哪个 Socket，连接到哪个地址，每次发送事件发送多少数据，生成多少发送事件以及以多大的速率从这些事件产生数据。
- 接下来，我们手动将 `TutorialApp` 应用程序添加到源节点，并显式调用应用程序的 `Start` 和 `Stop` 方法，告诉它何时开始和停止执行它的任务。

现在，我们需要将接收点对点 NetDevice 丢包事件连接到我们的 `RxDrop` 回调。

```cpp
devices.Get(1)->TraceConnectWithoutContext("PhyRxDrop", MakeCallback(&RxDrop));
```

现在应该很明显了，我们==正在从容器中获取接收节点 NetDevice 的引用==，并==将该设备上由属性 "PhyRxDrop" 定义的跟踪源连接到跟踪回调函数 `RxDrop`。==

最后，我们告诉模拟器覆盖任何应用程序，并在模拟的时间结束时停止处理事件。

```cpp
Simulator::Stop(Seconds(20));
Simulator::Run();
Simulator::Destroy();

return 0;
}
```

回想一下，==一旦调用 `Simulator::Run`，配置时间就结束了，进入了模拟时间==。我们通过创建应用程序和教它如何连接和发送数据来==协调的所有工作实际上都发生在这个函数调用期间。==

一旦 `Simulator::Run` 返回，模拟就完成了，我们进入了拆卸阶段。在这种情况下，`Simulator::Destroy` 处理了所有繁琐的细节，我们在它完成后返回一个成功的代码。
### Running `fifth.cc`
由于我们已经为您提供了文件 `fifth.cc`，如果您构建了您的发行版（在调试模式或默认模式下，因为它使用 `NS_LOG` - 请记住优化构建会优化掉 `NS_LOG`），它将等待您运行。

```bash
$ ./ns3 run fifth
1.00419 536
1.0093  1072
1.01528 1608
1.02167 2144
...
1.11319 8040
1.12151 8576
1.12983 9112
RxDrop at 1.13696
...
```

您可能立即看到在跟踪中使用任何种类的打印的一个==缺点==。我们得到了那些与我们的有趣信息混在一起的==冗余 ns3 消息==，==以及那些 RxDrop 消息==。**我们将很快解决这个问题**，但我确信您迫不及待地想要看到所有这项工作的结果。让我们将输出重定向到一个名为 `cwnd.dat` 的文件：

```bash
$ ./ns3 run fifth > cwnd.dat 2>&1
```

现在在您喜欢的编辑器中编辑 “cwnd.dat”，删除 ns3 构建状态和丢包行，只留下跟踪数据（您还可以在脚本中注释掉 `TraceConnectWithoutContext("PhyRxDrop", MakeCallback(&RxDrop));` 来轻松摆脱丢包打印）。

然后，您现在可以运行 gnuplot（如果已安装），并告诉它生成一些漂亮的图片：

```bash
$ gnuplot
gnuplot> set terminal png size 640,480
gnuplot> set output "cwnd.png"
gnuplot> plot "cwnd.dat" using 1:2 title 'Congestion Window' with linespoints
gnuplot> exit
```

现在，您应该有一个关于时间的拥塞窗口与时间的图形，存储在名为 “cwnd.png” 的文件中，看起来像：
![[Pasted image 20231202142119.png]]
### Using Mid-Level Helpers
在前面的部分中，我们展示了如何挂接跟踪源并从模拟中获取希望的有趣信息。也许你还记得我们早些时候在本章中将使用 std::cout 输出日志称为“粗糙工具”。我们还写过关于必须解析日志输出以分离出有趣信息的问题。你可能已经意识到，我们刚刚花了很多时间实现了一个示例，展示了我们声称要通过ns-3跟踪系统解决的所有问题！你是正确的。但请耐心等待，我们还没有完成。

我们想要做的最重要的一件事情之一是能够轻松地控制模拟输出的数量；而且我们还想将这些数据保存到文件中，以便以后参考。我们可以使用ns-3提供的==中级跟踪辅助工具==来做到这一点，从而完成整个画面。

我们提供了一个脚本，将在示例 fifth.cc 中开发的 cwnd 变化和丢包事件写入磁盘中的单独文件。cwnd 变化被存储为制表符分隔的ASCII文件，而丢包事件则存储在一个PCAP文件中。使这一切发生所需的更改非常小。
### Walkthrough: `sixth.cc`
让我们看一下从 fifth.cc 到 sixth.cc 所需的更改。在你喜欢的编辑器中打开 examples/tutorial/sixth.cc。你可以通过搜索 CwndChange 找到第一个更改。你会发现我们已经更改了跟踪接收器的签名，并在每个接收器中添加了一行代码，将跟踪的信息写入表示文件的流中。

```cpp
static void
CwndChange(Ptr<OutputStreamWrapper> stream, uint32_t oldCwnd, uint32_t newCwnd)
{
  NS_LOG_UNCOND(Simulator::Now().GetSeconds() << "\t" << newCwnd);
  *stream->GetStream() << Simulator::Now().GetSeconds() << "\t" << oldCwnd << "\t" << newCwnd << std::endl;
}

static void
RxDrop(Ptr<PcapFileWrapper> file, Ptr<const Packet> p)
{
  NS_LOG_UNCOND("RxDrop at " << Simulator::Now().GetSeconds());
  file->Write(Simulator::Now(), p);
}
```

我们==在 CwndChange 跟踪接收器中添加了一个 "stream" 参数==。这是一个保存（保持安全）C++输出流的对象。事实证明，这是一个非常简单的对象，但它管理了流的生命周期问题，并解决了即使是经验丰富的C++用户也会遇到的问题。原来，std::ostream 的拷贝构造函数被标记为私有。这意味着 std::ostream 不遵循值语义，不能在需要复制流的任何机制中使用。这包括 ns-3 回调系统，正如你可能记得的，它要求遵循值语义的对象。请注意，我们在 CwndChange 跟踪接收器实现中添加了以下行：

```cpp
*stream->GetStream() << Simulator::Now().GetSeconds() << "\t" << oldCwnd << "\t" << newCwnd << std::endl;
```

如果你用 std::cout 替换 *stream->GetStream()，代码将非常熟悉，如：

```cpp
std::cout << Simulator::Now().GetSeconds() << "\t" << oldCwnd << "\t" << newCwnd << std::endl;
```

这==说明 Ptr\<OutputStreamWrapper> 实际上只是为你携带一个 std::ofstream，==你可以在这里像使用任何其他**输出流**一样使用它。

在 RxDrop 中发生的情况类似，只是==传递的对象==（Ptr\<PcapFileWrapper>）==表示一个 PCAP 文件==。在跟踪接收器中有一个一行代码，用于将时间戳和被丢弃的数据包的内容写入 PCAP 文件：

```cpp
file->Write(Simulator::Now(), p);
```

当然，**如果我们有表示两个文件的对象**，我们需要在某处创建它们，并且还需要**导致它们被传递到跟踪接收器**。如果你查看 main 函数，你将找到新的代码来完成这个任务：

```cpp
AsciiTraceHelper asciiTraceHelper;  // 创建了 ASCII 跟踪文件
Ptr<OutputStreamWrapper> stream = asciiTraceHelper.CreateFileStream("sixth.cwnd");  
// 👆使用回调创建函数的变体来安排对象传递到接收器
ns3TcpSocket->TraceConnectWithoutContext("CongestionWindow", MakeBoundCallback(&CwndChange, stream));

...

PcapHelper pcapHelper; // 实例化了 PcapHelper 来对 PCAP 跟踪文件执行与 AsciiTraceHelper 相同的操作
Ptr<PcapFileWrapper> file = pcapHelper.CreateFile("sixth.pcap", std::ios::out, PcapHelper::DLT_PPP);
devices.Get(1)->TraceConnectWithoutContext("PhyRxDrop", MakeBoundCallback(&RxDrop, file));
```

1. 在上面的代码片段的第一部分，我们==创建了 ASCII 跟踪文件==，创建了一个负责管理它的对象，并==使用回调创建函数的变体来安排对象传递到接收器==。我们的 ASCII 跟踪助手提供了一组丰富的功能，以便轻松使用文本（ASCII）文件。我们只是在这里演示了文件流创建函数的使用。

2. *CreateFileStream 函数* 基本上**将实例化一个 std::ofstream 对象，并创建一个新文件（或截断现有文件）**。这个 std::ofstream 被封装在一个 ns-3 对象中，用于生命周期管理和拷贝构造函数问题的解决。

3. 然后，我们==取得代表文件的 ns-3 对象，并将其传递给 MakeBoundCallback()==。这个函数创建一个回调，就像 MakeCallback() 一样，但它在调用回调之前“绑定”一个新值到回调上。这个值被添加为回调的第一个参数，然后才调用。

4. 实际上，MakeBoundCallback(&CwndChange, stream) 导致跟踪源*在调用回调之前在形式参数列表的前面添加额外的 "stream" 参数*。这改变了 CwndChange 接收器的所需签名，以匹配上面显示的签名，其中包括 "额外" 参数 Ptr\<OutputStreamWrapper> stream。

在上面代码片段的第二部分中，我们==实例化了 PcapHelper 来==对 ==PCAP 跟踪文件==执行与 AsciiTraceHelper 相同的操作。代码行

```cpp
Ptr<PcapFileWrapper> file = pcapHelper.CreateFile("sixth.pcap", "w", PcapHelper::DLT_PPP);
```

**创建**了一个名为 "sixth.pcap" 的 **PCAP 文件**，**文件模式为 "w"**; 这意味着如果找到具有该名称的现有文件，新文件将被截断（内容被删除）;
最后一个参数是**新 PCAP 文件的“数据链路类型”**。

如果你熟悉 PCAP，这些与 bpf.h 中定义的 PCAP 库数据链路类型相同。在这种情况下，==DLT_PPP 表示 PCAP 文件将包含以点对点标头为前缀的数据包==。这是因为数据包来自我们的点对点设备驱动程序。其他常见的数据链路类型是 DLT_EN10MB（10 MB 以太网）适用于 csma 设备，以及 DLT_IEEE802_11（IEEE 802.11）适用于 wifi 设备。如果你有兴趣，可以在 src/network/helper/trace-helper.h 中看到它们的列表。列表中的条目与 bpf.h 中的条目相匹配，但我们复制它们以避免对 PCAP 源的依赖。

一个代表 PCAP 文件的 ns-3 对象从 CreateFile 返回，并在绑定回调中像 ASCII 情况一样使用。

重要的一点是，尽管这两个对象的声明方式非常相似，

```cpp
Ptr<PcapFileWrapper> file ...
Ptr<OutputStreamWrapper> stream ...
```

底层对象是完全不同的。例如，Ptr\<PcapFileWrapper> 是指向 ns-3 对象的智能指针，这是一个相当庞大的支持属性并且集成到 Config 系统中的东西。另一方面，Ptr\<OutputStreamWrapper> 是指向支持引用计数的对象的智能指针，它是一个非常轻量级的东西。在对该对象的“权限”进行任何假设之前，请记得查看你正在引用的对象。

例如，查看分发中的 src/network/utils/pcap-file-wrapper.h，并注意到，

```cpp
class PcapFileWrapper : public Object
```

通过继承，类 PcapFileWrapper 是一个 ns-3 对象。然后查看 src/network/model/output-stream-wrapper.h 并注意到，

```cpp
class OutputStreamWrapper : public SimpleRefCount<OutputStreamWrapper>
```

这个对象根本不是 ns-3 对象，它只是一个支持侵入式引用计数的 C++ 对象。

这里的要点是，仅仅因为你读到 Ptr\<something> 并不一定意味着 something 是一个你可以挂上 ns-3 Attributes 的 ns-3 对象。

现在，回到例子。如果你构建并运行这个例子，

```bash
$ ./ns3 run sixth
```

你会看到与运行 "fifth" 时相同的消息，但是在你的 ns-3 分发目录的顶层目录中将会出现两个新文件。

```
sixth.cwnd  sixth.pcap
```

由于 "sixth.cwnd" 是一个 ASCII 文本文件，你可以使用 cat 或你喜欢的文件查看器查看它。

```plaintext
1       0       536
1.0093  536     1072
1.01528 1072    1608
1.02167 1608    2144
...
9.69256 5149    5204
9.89311 5204    5259
```

你有一个制表符分隔的文件，其中包含一个时间戳，一个旧的拥塞窗口和一个新的拥塞窗口，适合直接导入到你的绘图程序中。文件中没有多余的打印，不需要解析或编辑。

由于 "sixth.pcap" 是一个 PCAP 文件，你可以使用 tcpdump 查看它。

```bash
tcpdump -r sixth.pcap
```

你有一个包含模拟中丢弃的数据包的 PCAP 文件。文件中没有其他数据包，也没有其他内容使事情变得复杂。
### Cor : 
这是一个漫长的旅程，但现在我们==已经到达了能够欣赏 ns-3 跟踪系统的地步==。我们从 TCP 实现和设备驱动程序的中间提取了重要事件。我们直接将这些事件存储在可以使用常见工具的文件中。我们在不修改任何核心代码的情况下完成了这一切，而且只用了 18 行代码：

```cpp
static void
CwndChange(Ptr<OutputStreamWrapper> stream, uint32_t oldCwnd, uint32_t newCwnd)
{
  NS_LOG_UNCOND(Simulator::Now().GetSeconds() << "\t" << newCwnd);
  *stream->GetStream() << Simulator::Now().GetSeconds() << "\t" << oldCwnd << "\t" << newCwnd << std::endl;
}

...

AsciiTraceHelper asciiTraceHelper;
Ptr<OutputStreamWrapper> stream = asciiTraceHelper.CreateFileStream("sixth.cwnd");
ns3TcpSocket->TraceConnectWithoutContext("CongestionWindow", MakeBoundCallback(&CwndChange, stream));

...

static void
RxDrop(Ptr<PcapFileWrapper> file, Ptr<const Packet> p)
{
  NS_LOG_UNCOND("RxDrop at " << Simulator::Now().GetSeconds());
  file->Write(Simulator::Now(), p);
}

...

PcapHelper pcapHelper;
Ptr<PcapFileWrapper> file = pcapHelper.CreateFile("sixth.pcap", "w", PcapHelper::DLT_PPP);
devices.Get(1)->TraceConnectWithoutContext("PhyRxDrop", MakeBoundCallback(&RxDrop, file));
```
### Trace Helpers
ns-3的跟踪助手提供了一个丰富的环境，用于配置和选择不同的跟踪事件并将它们写入文件。在之前的章节中，主要是在“构建拓扑”一节中，我们看到了设计用于在其他（设备）助手中使用的几种跟踪助手方法的各种变体。

也许你还记得看到了一些这样的变体：

```cpp
pointToPoint.EnablePcapAll("second");
pointToPoint.EnablePcap("second", p2pNodes.Get(0)->GetId(), 0);
csma.EnablePcap("third", csmaDevices.Get(0), true);
pointToPoint.EnableAsciiAll(ascii.CreateFileStream("myfirst.tr"));
```

然而，可能不太明显的是，系统中所有与跟踪相关的方法都有一个一致的模型。现在我们将花点时间来看一下“全局视图”。

目前，在ns-3中有两种主要用例用于跟踪助手：设备助手和协议助手。设备助手解决了通过（节点、设备）对来指定应启用哪些跟踪的问题。例如，您可能想指定应在特定节点上的特定设备上启用PCAP跟踪。这遵循自ns-3设备概念模型，以及各种设备助手的概念模型。由此自然产生的文件遵循 \<prefix>-\<node>-\<device> 的命名约定。

协议助手解决了通过协议和接口对指定应启用哪些跟踪的问题。这遵循自ns-3协议栈概念模型，以及Internet栈助手的概念模型。自然而然，跟踪文件应该遵循 \<prefix>-\<protocol>-\<interface> 的命名约定。

因此，跟踪助手自然地落入一个二维的分类法中。有一些微妙之处阻止了所有四个类别的行为完全相同，但我们确实努力使它们尽可能相似；并且在所有类别中的所有方法都有类似物时，我们会尽量让它们都能够工作。

```
             PCAP    ASCII
设备助手   \checkmark  \checkmark
协议助手   \checkmark  \checkmark
```

我们使用一种称为“mixin”的方法将跟踪功能添加到我们的助手类中。Mixin是一个在子类继承时提供功能的类。从mixin继承不被认为是一种专业化形式，而实际上是一种收集功能的方式。

让我们快速看一下这四种情况及其各自的mixin。
### Device Helpers
#### PCAP
这些助手的目标是使在ns-3设备中添加一致的PCAP跟踪功能变得容易。我们希望所有各种PCAP跟踪方式在所有设备上都能够以相同的方式工作，因此这些助手的方法会被设备助手继承。如果你想在查看实际代码时跟踪讨论，请查看 src/network/helper/trace-helper.h。

类 PcapHelperForDevice 是一个混合类，==提供了在ns-3设备中使用PCAP跟踪的高级功能==。每个设备必须实现从这个类继承的单个虚拟方法。

```cpp
virtual void EnablePcapInternal(std::string prefix, Ptr<NetDevice> nd, bool promiscuous, bool explicitFilename) = 0;
```

这个方法的签名反映了在这个级别上的以设备为中心的视图。从类 PcapUserHelperForDevice 继承的所有公共方法最终都会调用这个单一的、依赖于设备的实现方法。例如，最低级别的PCAP方法：

```cpp
void EnablePcap(std::string prefix, Ptr<NetDevice> nd, bool promiscuous = false, bool explicitFilename = false);
```

将直接调用 EnablePcapInternal 方法的设备实现。所有其他公共PCAP跟踪方法都建立在这个实现上，以提供额外的用户级功能。对用户来说，这意味着系统中的所有设备助手都将具有所有PCAP跟踪方法；并且如果设备正确实现了 EnablePcapInternal，这些方法在所有设备上都将以相同的方式工作。
#### Methods
在上面显示的每个方法中，都==有一个名为 promiscuous 的默认参数，默认值为 false。该参数表示跟踪不应在混杂模式下收集==。*如果确实希望跟踪包括设备看到的所有流量（并且设备支持混杂模式），只需在上述任何调用中添加一个 true 参数*。例如，

```cpp
Ptr<NetDevice> nd;
...
helper.EnablePcap("prefix", nd, true);
```

将在由 nd 指定的 NetDevice 上启用混杂模式捕获。

前两种方法还包括一个名为 explicitFilename 的默认参数，将在下面讨论。

鼓励你查看类 PcapHelperForDevice 的 API 文档以找到这些方法的详细信息；但总的来说：

- 通过向 EnablePcap 方法提供 Ptr\<NetDevice>，可以==在特定节点/网络设备对上启用PCAP跟踪==。Ptr\<Node> 是隐式的，因为网络设备必须属于正好一个节点。例如，

  ```cpp
  Ptr<NetDevice> nd;
  ...
  helper.EnablePcap("prefix", nd);
  ```

- 通过向 EnablePcap 方法提供表示对象名称服务字符串的 std::string，可以在特定节点/网络设备对上启用PCAP跟踪。==从名称字符串查找 Ptr\<NetDevice>==。同样，\<Node> 是隐式的，因为命名的网络设备必须属于正好一个节点。例如，

  ```cpp
  Names::Add("server" ...);
  Names::Add("server/eth0" ...);
  ...
  helper.EnablePcap("prefix", "server/ath0");
  ```

- 通过提供 NetDeviceContainer，可以在一组节点/网络设备对上启用PCAP跟踪。==对于容器中的每个 NetDevice，检查其类型==。==对于与设备助手管理的设备相同类型的每个设备，启用跟踪==。同样，\<Node> 是隐式的，因为找到的网络设备必须属于正好一个节点。例如，

  ```cpp
  NetDeviceContainer d = ...;
  ...
  helper.EnablePcap("prefix", d);
  ```

- 通过提供 NodeContainer，可以在一组节点/网络设备对上启用PCAP跟踪。==对于 NodeContainer 中的每个节点，迭代其附加的 NetDevices==。==对于容器中每个节点附加的每个 NetDevice==，检查该设备的类型。对于与设备助手管理的设备相同类型的每个设备，启用跟踪。

  ```cpp
  NodeContainer n;
  ...
  helper.EnablePcap("prefix", n);
  ```

- 还可以根据节点ID和设备ID以及显式 Ptr 启用PCAP跟踪。系统中的每个节点都有一个整数节点ID，连接到节点的每个设备都有一个整数设备ID。

  ```cpp
  helper.EnablePcap("prefix", 21, 1);
  ```

- 最后，可以为系统中由设备助手管理的相同类型的所有设备启用PCAP跟踪。

  ```cpp
  helper.EnablePcapAll("prefix");
  ```
#### Filenames
上述方法描述中隐含了通过实现方法构建完整文件名的过程。按照约定，ns-3系统中的PCAP跟踪文件的命名形式为 `<prefix>-<node id>-<device id>.pcap`。

正如先前提到的，系统中的==每个节点都将有一个系统分配的节点ID；而每个设备都将有一个相对于其节点的接口索引（也称为设备ID）==。因此，默认情况下，通过在Node 21上的第一个设备上启用跟踪，并使用前缀“prefix”创建的PCAP跟踪文件将是 `prefix-21-1.pcap`。

你始终可以使用ns-3对象名称服务来使这更加清晰。例如，如果你使用对象名称服务将名称“server”分配给Node 21，那么由此产生的PCAP跟踪文件名称将自动变为 `prefix-server-1.pcap`，如果你还将名称“eth0”分配给设备，你的PCAP文件名将自动获取这个并称为 `prefix-server-eth0.pcap`。

最后，上面显示的两个方法，

```cpp
void EnablePcap(std::string prefix, Ptr<NetDevice> nd, bool promiscuous = false, bool explicitFilename = false);
void EnablePcap(std::string prefix, std::string ndName, bool promiscuous = false, bool explicitFilename = false);
```

都有一个==名为 explicitFilename 的默认参数==。**当设置为 true 时，此参数禁用自动文件名完成机制，允许你创建显式文件名**。此选项仅在启用PCAP跟踪的单个设备的方法中可用。

例如，为了让设备助手在给定设备上创建一个名为 `my-pcap-file.pcap` 的单个混杂模式PCAP捕获文件，可以：

```cpp
Ptr<NetDevice> nd;
...
helper.EnablePcap("my-pcap-file.pcap", nd, true, true);
```

第一个 true 参数启用混杂模式跟踪，第二个告诉助手将前缀参数解释为完整文件名。
#### ASCII
ASCII跟踪助手混合类的行为与PCAP版本基本相似。如果你想在查看真实代码时跟踪讨论，请查看 `src/network/helper/trace-helper.h`。

类 `AsciiTraceHelperForDevice` 添加了在设备助手类中使用ASCII跟踪的高级功能。与PCAP情况类似，==每个设备必须实现从ASCII跟踪混合类继承的单个虚拟方法==。

```cpp
virtual void EnableAsciiInternal(Ptr<OutputStreamWrapper> stream,
                                 std::string prefix,
                                 Ptr<NetDevice> nd,
                                 bool explicitFilename) = 0;
```

这个方法的签名反映了在这个级别上设备为中心的视图；还反映了助手可能在写入共享输出流。从类 `AsciiTraceHelperForDevice` 继承的所有公共ASCII跟踪相关方法最终都将调用这个单一的、依赖于设备的实现方法。例如，最低级别的ASCII跟踪方法：

```cpp
void EnableAscii(std::string prefix, Ptr<NetDevice> nd, bool explicitFilename = false);
void EnableAscii(Ptr<OutputStreamWrapper> stream, Ptr<NetDevice> nd);
```

将直接调用 `EnableAsciiInternal` 方法的设备实现，提供有效的前缀或流。

所有其他公共ASCII跟踪方法将在这些低级功能的基础上构建，以提供额外的用户级功能。对用户来说，这意味着系统中的所有设备助手都将具有所有ASCII跟踪方法；如果设备正确实现了 `EnableAsciiInternal`，这些方法将在所有设备上以相同的方式工作。
### Methods
```cpp
void EnableAscii(std::string prefix, Ptr<NetDevice> nd, bool explicitFilename = false);
void EnableAscii(Ptr<OutputStreamWrapper> stream, Ptr<NetDevice> nd);

void EnableAscii(std::string prefix, std::string ndName, bool explicitFilename = false);
void EnableAscii(Ptr<OutputStreamWrapper> stream, std::string ndName);

void EnableAscii(std::string prefix, NetDeviceContainer d);
void EnableAscii(Ptr<OutputStreamWrapper> stream, NetDeviceContainer d);

void EnableAscii(std::string prefix, NodeContainer n);
void EnableAscii(Ptr<OutputStreamWrapper> stream, NodeContainer n);

void EnableAsciiAll(std::string prefix);
void EnableAsciiAll(Ptr<OutputStreamWrapper> stream);

void EnableAscii(std::string prefix, uint32_t nodeid, uint32_t deviceid, bool explicitFilename);
void EnableAscii(Ptr<OutputStreamWrapper> stream, uint32_t nodeid, uint32_t deviceid);
```
鼓励您仔细阅读AsciiTraceHelperForDevice类的API文档以获取这些方法的详细信息；但总体上来说...

ASCII跟踪提供的方法数量是PCAP跟踪的两倍。这是因为除了支持PCAP风格的模型，其中从每个唯一的节点/设备对的跟踪写入一个唯一的文件（一对一）之外，我们还支持一种模型：其中许多节点/设备对的跟踪信息写入一个共同的文件（多对一）。

这意味着<前缀>-<节点>-<设备>文件名生成机制被替换为引用共同文件的机制；API方法的数量翻倍，以支持所有可能的组合。

就像在PCAP跟踪中一样，您可以通过向EnableAscii方法提供Ptr\<NetDevice>来启用对特定（节点、网络设备）对的ASCII跟踪。Ptr\<Node>是隐式的，因为网络设备必须属于一个节点。例如，

```cpp
Ptr<NetDevice> nd;
...
helper.EnableAscii("prefix", nd);
```

前四种方法还包括一个名为explicitFilename的默认参数，其操作类似于PCAP情况中的等效参数。

==在这种情况下，不会将跟踪上下文写入ASCII跟踪文件，因为它们是多余的==。系统将==使用与PCAP部分描述的相同规则选择要创建的文件名==，**唯一的区别是文件将具有后缀.tr而不是.pcap。**

如果您想要在多个网络设备上启用ASCII跟踪并将所有跟踪发送到单个文件（多对一），您也可以通过使用一个对象引用单个文件来实现。我们在上面的“cwnd”示例中已经见过这种情况：

```cpp
Ptr<NetDevice> nd1;
Ptr<NetDevice> nd2;
...
Ptr<OutputStreamWrapper> stream = asciiTraceHelper.CreateFileStream("trace-file-name.tr");
...
helper.EnableAscii(stream, nd1);
helper.EnableAscii(stream, nd2);
```

在这种情况下，由于需要消除两个设备的跟踪歧义，跟踪上下文将被写入ASCII跟踪文件。请注意，==由于用户完全指定了文件名，字符串应包含.tr后缀以保持一致性。==

您可以==通过向EnablePcap方法提供表示对象名称服务字符串的std::string来启用对特定（节点、网络设备）对的ASCII跟踪==。Ptr\<NetDevice>从名称字符串查找。同样，由于命名的网络设备必须属于一个节点，所以\<Node>是隐式的。例如：

```cpp
Names::Add("client" ...);
Names::Add("client/eth0" ...);
Names::Add("server" ...);
Names::Add("server/eth0" ...);
...
helper.EnableAscii("prefix", "client/eth0");
helper.EnableAscii("prefix", "server/eth0");
```

这将导致==生成两个文件，分别命名为prefix-client-eth0.tr和prefix-server-eth0.tr，其中包含各个设备的跟踪信息==。由于所有的EnableAscii函数都被重载以接受流包装器，您也可以使用那种形式：

```cpp
Names::Add("client" ...);
Names::Add("client/eth0" ...);
Names::Add("server" ...);
Names::Add("server/eth0" ...);
...
Ptr<OutputStreamWrapper> stream = asciiTraceHelper.CreateFileStream("trace-file-name.tr");
...
helper.EnableAscii(stream, "client/eth0");
helper.EnableAscii(stream, "server/eth0");
```

这将**生成一个名为trace-file-name.tr的单一跟踪文件，其中包含两个设备的所有跟踪事件**。这些事件将通过跟踪上下文字符串进行消歧。

您可以通过提供一个**NetDeviceContainer**来为==一组（节点、网络设备）对启用ASCII跟踪==。对于容器中的每个NetDevice，都会检查其类型。对于具有正确类型的每个设备（与设备助手管理的相同类型），都会启用跟踪。同样，由于找到的网络设备必须属于一个节点，所以\<Node>是隐式的。例如，

```cpp
NetDeviceContainer d = ...;
...
helper.EnableAscii("prefix", d);
```

这将导致创建多个ASCII跟踪文件，每个文件都遵循\<prefix>-\<node id>-\<device id>.tr的约定。

将==所有跟踪组合到单个文件==中的方式与上面的示例类似：

```cpp
NetDeviceContainer d = ...;
...
Ptr<OutputStreamWrapper> stream = asciiTraceHelper.CreateFileStream("trace-file-name.tr");
...
helper.EnableAscii(stream, d);
```

您还可以通过提供一个**NodeContainer，在每个Node中迭代其连接的NetDevices**，为一组（节点、网络设备）对启用ASCII跟踪。对于容器中的每个Node，都会迭代其连接的NetDevices。对于连接到容器中的每个Node的每个NetDevice，都会检查该设备的类型。对于具有正确类型的每个设备（与设备助手管理的相同类型），都会启用跟踪。

```cpp
NodeContainer n;
...
helper.EnableAscii("prefix", n);
```

这将导致创建多个ASCII跟踪文件，每个文件都遵循\<prefix>-\<node id>-\<device id>.tr的约定。将所有跟踪组合到单个文件中的方式与上面的示例类似。

您还可以基于节点ID和设备ID以及显式的Ptr来启用ASCII跟踪。系统中的每个Node都有一个整数Node ID，连接到Node的每个设备都有一个整数设备ID。

```cpp
helper.EnableAscii("prefix", 21, 1);
```

当然，如上面所示，可以将跟踪组合到单个文件中。

最后，您可以**为系统中所有与设备助手管理的相同类型的设备启用ASCII跟踪**。

```cpp
helper.EnableAsciiAll("prefix");
```

这将导致==创建多个ASCII跟踪文件，每个文件都是系统中由助手管理的设备类型的一个==。所有这些文件都将遵循\<prefix>-\<node id>-\<device id>.tr的约定。将所有跟踪组合到单个文件中的方式与上面的示例类似。
### Filenames
上述基于前缀的方法描述中隐含着由实现方法构建完整文件名的约定。按照规定，ns-3系统中的ASCII跟踪文件的==格式为\<prefix>-\<node id>-\<device id>.tr。==

如前所述，系统中的每个节点都将有一个由系统分配的节点ID；每个设备都将有一个相对于其节点的接口索引（也称为设备ID）。因此，默认情况下，通过在节点21的第一个设备上启用跟踪，使用前缀“prefix”创建的ASCII跟踪文件将是prefix-21-1.tr。

您始终可以使用ns-3对象名称服务来使这一点更清晰。例如，如果您使用对象名称服务将名称“server”分配给节点21，那么生成的ASCII跟踪文件名称将自动变为prefix-server-1.tr；如果您还将名称“eth0”分配给设备，您的ASCII跟踪文件名称将自动获取此名称并称为prefix-server-eth0.tr。

几种方法中有一个名为explicitFilename的默认参数。当设置为true时，此参数禁用自动文件名完成机制，允许您创建显式文件名。此选项仅在那些以前缀为参数并在单个设备上启用跟踪的方法中可用。
### Protocol Helpers
#### PCAP
这些混合类的目标是使**向协议添加一致的PCAP跟踪功能**变得简单。我们希望所有==不同类型的PCAP跟踪在所有协议中都能够以相同的方式工作==，**因此这些辅助类的方法会被协议栈辅助类继承**。如果您想在查看实际代码时进行讨论，请查看src/network/helper/trace-helper.h。

在本节中，我们将说明这些方法如何应用于Ipv4协议。要指定类似协议中的跟踪，只需替换适当的类型。例如，使用Ptr\<Ipv6>而不是Ptr\<Ipv4>，并调用EnablePcapIpv6而不是EnablePcapIpv4。

**PcapHelperForIpv4类**提供了**在Ipv4协议中使用PCAP跟踪的高级功能**。启用这些方法的每个协议助手都必须实现从这个类继承的一个虚拟方法。例如，Ipv6将有一个单独的实现，但唯一的区别将在方法名称和签名上。不同的方法名称是为了消除从类Object派生的Ipv4和Ipv6之间的歧义，这两者都有相同的方法签名。

```cpp
virtual void EnablePcapIpv4Internal(std::string prefix,
                                    Ptr<Ipv4> ipv4,
                                    uint32_t interface,
                                    bool explicitFilename) = 0;
```

该方法的签名反映了在这个级别上的协议和接口中心的视图。从PcapHelperForIpv4类继承的所有公共方法都将归结为调用这个单一的依赖于设备的实现方法。例如，最低级的PCAP方法：

```cpp
void EnablePcapIpv4(std::string prefix, Ptr<Ipv4> ipv4, uint32_t interface, bool explicitFilename = false);
```

将直接调用EnablePcapIpv4Internal的设备实现。所有其他公共的PCAP跟踪方法都基于这个实现来提供额外的用户级功能。对用户来说，这意味着系统中的所有协议助手都将具有所有PCAP跟踪方法；==如果助手正确实现了EnablePcapIpv4Internal，这些方法将在跨协议时以相同的方式工作==。
##### Methods
这些方法旨在与设备版本的Node-和NetDevice-中心版本一一对应。与Node和NetDevice对的约束不同，我们使用协议和接口的约束。

请注意，与设备版本一样，这里有六种方法：

```cpp
void EnablePcapIpv4(std::string prefix, Ptr<Ipv4> ipv4, uint32_t interface, bool explicitFilename = false);
void EnablePcapIpv4(std::string prefix, std::string ipv4Name, uint32_t interface, bool explicitFilename = false);
void EnablePcapIpv4(std::string prefix, Ipv4InterfaceContainer c);
void EnablePcapIpv4(std::string prefix, NodeContainer n);
void EnablePcapIpv4(std::string prefix, uint32_t nodeid, uint32_t interface, bool explicitFilename);
void EnablePcapIpv4All(std::string prefix);
```

鼓励您查阅PcapHelperForIpv4类的API文档以获取这些方法的详细信息；但总体上来说...

您可以通过向==EnablePcap方法==提供 **Ptr\<Ipv4>** 和 **接口** 来==启用特定协议/接口对的PCAP跟踪==。例如，

```cpp
Ptr<Ipv4> ipv4 = node->GetObject<Ipv4>();
...
helper.EnablePcapIpv4("prefix", ipv4, 0);
```

您可以通过向EnablePcap方法==提供表示对象名称服务字符串的std::string，为特定的节点/网络设备对启用PCAP跟踪==。Ptr\<Ipv4>从名称字符串查找。例如，

```cpp
Names::Add("serverIPv4" ...);
...
helper.EnablePcapIpv4("prefix", "serverIpv4", 1);
```

您可以通过提供==Ipv4InterfaceContainer来为一组协议/接口对启用PCAP跟踪==。对于**容器中的每个Ipv4 / 接口对，将检查协议类型**。对于==具有正确类型的每个协议==（与设备助手管理的相同类型），==将为相应的接口启用跟踪==。例如，

```cpp
NodeContainer nodes;
...
NetDeviceContainer devices = deviceHelper.Install(nodes);
...
Ipv4AddressHelper ipv4;
ipv4.SetBase("10.1.1.0", "255.255.255.0");
Ipv4InterfaceContainer interfaces = ipv4.Assign(devices);
...
helper.EnablePcapIpv4("prefix", interfaces);
```

您可以通过提供==NodeContainer来为一组协议/接口对启用PCAP跟踪==。对于NodeContainer中的每个Node，都会找到适当的协议。对于每个协议，会枚举其接口，并在生成的协议对上启用跟踪。例如，

```cpp
NodeContainer n;
...
helper.EnablePcapIpv4("prefix", n);
```

您还可以==基于节点ID和接口启用PCAP跟踪==。在这种情况下，节点ID将被翻译为Ptr\<Node>，并在节点中查找适当的协议。生成的协议和接口用于指定生成的跟踪源。

```cpp
helper.EnablePcapIpv4("prefix", 21, 1);
```

最后，您可以==为系统中的所有接口启用PCAP跟踪==，关联协议的类型与设备助手管理的类型相同。

```cpp
helper.EnablePcapIpv4All("prefix");
```
##### Filenames
上述所有方法描述中都隐含了实现方法构建完整文件名的约定。按照惯例，==ns-3系统中为设备采集的PCAP跟踪文件的格式为“\<prefix>-\<node id>-\<device id>.pcap”。==

在**协议跟踪的情况下，协议和节点之间存在一对一的对应关系**。这是因为协议对象被聚合到节点对象中。由于系统中没有全局协议ID，我们使用相应的节点ID进行文件命名。因此，在自动选择跟踪文件名时存在文件名冲突的可能性。因此，协议跟踪的文件名约定发生了变化。

如前所述，系统中的每个节点都将有一个系统分配的节点ID。由于协议实例和节点实例之间存在一对一的对应关系，我们使用节点ID。每个接口相对于其协议都有一个接口ID。我们在协议助手中使用约定“\<prefix>-n\<node id>-i\<interface id>.pcap”来命名跟踪文件。

因此，默认情况下，通过在节点21上启用对Ipv4协议接口1的跟踪所创建的PCAP跟踪文件，使用前缀“prefix”将是“prefix-n21-i1.pcap”。

您始终可以使用ns-3对象名称服务来使这一点更清晰。例如，如果您使用对象名称服务将名称“serverIpv4”分配给节点21上的Ptr\<Ipv4>，则生成的PCAP跟踪文件名将自动变为“prefix-nserverIpv4-i1.pcap”。

其中几个方法都有一个名为explicitFilename的默认参数。当设置为true时，此参数将禁用自动文件名完成机制，允许您创建显式文件名。此选项仅在那些以前缀为参数并在单个设备上启用跟踪的方法中可用。
#### ASCII
ASCII跟踪助手的行为与PCAP情况非常相似。如果您想在查看实际代码时进行讨论，请查看src/network/helper/trace-helper.h。

在本节中，我们将以==Ipv4协议==为例来说明这些方法。要指定类似协议中的跟踪，只需替换适当的类型。例如，使用Ptr\<Ipv6>而不是Ptr\<Ipv4>，并调用EnableAsciiIpv6而不是EnableAsciiIpv4。

AsciiTraceHelperForIpv4类==为协议助手添加了使用ASCII跟踪的高级功能==。启用这些方法的每个协议都必须实现从这个类继承的一个虚拟方法。

```cpp
virtual void EnableAsciiIpv4Internal(Ptr<OutputStreamWrapper> stream,
                                     std::string prefix,
                                     Ptr<Ipv4> ipv4,
                                     uint32_t interface,
                                     bool explicitFilename) = 0;
```

该方法的签名反映了在这个级别上的协议和接口中心的视图，以及助手可能正在写入共享输出流的事实。从PcapAndAsciiTraceHelperForIpv4类继承的所有公共方法都将归结为调用这个单一的依赖于设备的实现方法。例如，最低级别的ASCII跟踪方法：

```cpp
void EnableAsciiIpv4(std::string prefix, Ptr<Ipv4> ipv4, uint32_t interface, bool explicitFilename = false);
void EnableAsciiIpv4(Ptr<OutputStreamWrapper> stream, Ptr<Ipv4> ipv4, uint32_t interface);
```

将直接调用EnableAsciiIpv4Internal的设备实现，直接提供前缀或流。所有其他公共ASCII跟踪方法将在这些低级功能的基础上构建，以提供额外的用户级功能。对用户来说，这意味着系统中的所有设备助手都将具有所有ASCII跟踪方法；如果协议正确实现了EnableAsciiIpv4Internal，这些方法将在跨协议时以相同的方式工作。
##### Methods
您被鼓励查阅PcapAndAsciiHelperForIpv4类的API文档以找到这些方法的详细信息，但总体上来说...

ASCII跟踪的方法数量是PCAP跟踪的两倍。这是因为除了PCAP风格模型，其中从每个唯一的协议/接口对写入一个唯一的文件，我们还支持一个模型，其中许多协议/接口对的跟踪信息被写入一个共同的文件。这意味着"\<prefix>-n\<node id>-\<interface>"文件名生成机制被替换为引用共同文件的机制，并且API方法的数量增加了一倍以允许所有组合。

就像在PCAP跟踪中一样，您可以通过==向EnableAscii方法==提供==Ptr\<Ipv4>==和==一个接口==来**启用对特定协议/接口对**的ASCII跟踪。例如，

```cpp
Ptr<Ipv4> ipv4;
...
helper.EnableAsciiIpv4("prefix", ipv4, 1);
```

在这种情况下，不会将任何跟踪上下文写入ASCII跟踪文件，因为它们将是多余的。系统将选择使用与PCAP部分中描述的相同规则创建文件名，只是文件将具有".tr"后缀而不是".pcap"。

如果您想要在==多个接口上启用ASCII跟踪并将所有跟踪发送到单个文件==，也可以通过使用对象引用单个文件来实现。我们在上面的“cwnd”示例中已经看到了类似的情况：

```cpp
Ptr<Ipv4> protocol1 = node1->GetObject<Ipv4>();
Ptr<Ipv4> protocol2 = node2->GetObject<Ipv4>();
...
Ptr<OutputStreamWrapper> stream = asciiTraceHelper.CreateFileStream("trace-file-name.tr");
...
helper.EnableAsciiIpv4(stream, protocol1, 1);
helper.EnableAsciiIpv4(stream, protocol2, 1);
```

在这种情况下，跟踪上下文被写入ASCII跟踪文件，因为它们是必需的，以区分两个接口的跟踪。请注意，由于用户完全指定了文件名，字符串应包含",tr"以保持一致性。

您还可以通过提供==表示对象名称服务字符串的std::string来为特定的协议启用ASCII跟踪==。从名称字符串查找Ptr\<Ipv4>。由于协议实例和节点之间存在一对一的对应关系，因此生成的文件名中的\<Node>是隐式的。例如，

```cpp
Names::Add("node1Ipv4" ...);
Names::Add("node2Ipv4" ...);
...
helper.EnableAsciiIpv4("prefix", "node1Ipv4", 1);
helper.EnableAsciiIpv4("prefix", "node2Ipv4", 1);
```

这将导致两个文件被命名为"prefix-nnode1Ipv4-i1.tr"和"prefix-nnode2Ipv4-i1.tr"，其中分别包含各个接口的跟踪文件。由于所有的EnableAscii函数都被重载以接受一个流包装器，您也可以使用这种形式：

```cpp
Names::Add("node1Ipv4" ...);
Names::Add("node2Ipv4" ...);
...
Ptr<OutputStreamWrapper> stream = asciiTraceHelper.CreateFileStream("trace-file-name.tr");
...
helper.EnableAsciiIpv4(stream, "node1Ipv4", 1);
helper.EnableAsciiIpv4(stream, "node2Ipv4", 1);
```

这将导致一个名为"trace-file-name.tr"的单一跟踪文件，其中包含两个接口的所有跟踪事件。这些事件将通过跟踪上下文字符串进行区分。

您还可以通过提供==Ipv4InterfaceContainer==为==一组协议/接口对启用ASCII跟踪==。对于容器中每个正确类型的协议，都会为相应的接口启用跟踪。再次强调，

由于每个协议和其节点之间存在一对一的对应关系，所以\<Node>是隐式的。例如，

```cpp
NodeContainer nodes;
...
NetDeviceContainer devices = deviceHelper.Install(nodes);
...
Ipv4AddressHelper ipv4;
ipv4.SetBase("10.1.1.0", "255.255.255.0");
Ipv4InterfaceContainer interfaces = ipv4.Assign(devices);
...
...
helper.EnableAsciiIpv4("prefix", interfaces);
```

这将导致创建多个ASCII跟踪文件，每个文件都遵循\<prefix>-n\<node id>-i\<interface>.tr的约定。将所有跟踪合并到单个文件中的方法与上面的示例类似：

```cpp
NodeContainer nodes;
...
NetDeviceContainer devices = deviceHelper.Install(nodes);
...
Ipv4AddressHelper ipv4;
ipv4.SetBase("10.1.1.0", "255.255.255.0");
Ipv4InterfaceContainer interfaces = ipv4.Assign(devices);
...
Ptr<OutputStreamWrapper> stream = asciiTraceHelper.CreateFileStream("trace-file-name.tr");
...
helper.EnableAsciiIpv4(stream, interfaces);
```

您还可以通过==提供NodeContainer为一组协议/接口对启用ASCII跟踪==。对于NodeContainer中的每个节点，都会找到适当的协议。对于每个协议，将枚举其接口，并在生成的对上启用跟踪。例如，

```cpp
NodeContainer n;
...
helper.EnableAsciiIpv4("prefix", n);
```

这将导致创建多个ASCII跟踪文件，每个文件都遵循\<prefix>-n\<node id>-i\<interface>.tr的约定。将所有跟踪合并到单个文件中的方法与上面的示例类似。

最后，您还==可以基于Node ID和接口ID启用ASCII跟踪==。在这种情况下，将Node ID转换为Ptr\<Node>，并在节点中查找适当的协议。使用生成的协议和接口来指定生成的跟踪源。

```cpp
helper.EnableAsciiIpv4("prefix", 21, 1);
```

当然，跟踪可以像上面展示的那样合并到单个文件中。

最后，您还可以为系统中所有与设备助手管理的协议类型相同的接口启用ASCII跟踪。

```cpp
helper.EnableAsciiIpv4All("prefix");
```

这将导致创建多个ASCII跟踪文件，每个文件都对应于系统中与助手管理的类型相同的每个接口。所有这些文件都将遵循\<prefix>-n\<node id>-i\<interface>.tr的约定。将所有跟踪合并到单个文件中的方法与上面的示例类似。
##### Filenames
上述基于前缀的方法描述中隐含的是由实现方法构建完整文件名的约定。按照规定，ns-3系统中的ASCII跟踪文件的格式为“\<prefix>-\<node id>-\<device id>.tr”

正如先前提到的，系统中的每个节点都将被分配一个系统分配的节点ID。由于协议和节点之间是一对一的对应关系，我们使用节点ID来标识协议的身份。给定协议上的每个接口都将相对于其协议具有接口索引（也称为接口）。因此，默认情况下，启用对Node 21的第一个设备进行跟踪所创建的ASCII跟踪文件，使用前缀“prefix”，将是“prefix-n21-i1.tr”。使用前缀以消除每个节点上的多个协议的歧义。

您始终可以使用ns-3对象名称服务使这更加清晰。例如，如果您使用对象名称服务将名称“serverIpv4”分配给Node 21上的协议，并且还指定了接口1，那么生成的ASCII跟踪文件名将自动变为“prefix-nserverIpv4-1.tr”。

其中一些方法具有名为explicitFilename的默认参数。当设置为true时，该参数会禁用自动文件名完成机制，并允许您创建显式文件名。此选项仅适用于那些采用前缀并在单个设备上启用跟踪的方法。
### Summary
ns-3提供了一个非常丰富的环境，允许用户在多个层次上定制从模拟中提取的信息种类。

有高级别的辅助函数，允许用户简单地控制对预定义输出的收集，实现了精细的粒度控制。有中级别的辅助函数，允许更有经验的用户定制信息的提取和保存方式；还有低级别的核心功能，允许专业用户改变系统以呈现以前未导出的新信息，使其能够立即被更高级别的用户访问。

这是一个非常全面的系统，我们意识到这对于新用户或不熟悉C++及其习惯用法的用户来说可能是很多内容需要理解的。我们认为跟踪系统是ns-3非常重要的一部分，因此建议尽可能熟悉它。可能情况是，一旦您掌握了跟踪系统，理解ns-3系统的其余部分将会相当简单。