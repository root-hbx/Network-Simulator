我们的最后一个教程章节介绍了在ns-3版本3.18中添加的一些组件，这些组件仍在开发中。这个教程部分也是一个正在进行中的工作。
## Motivation
运行仿真的主要目的之一是生成输出数据，无论是用于研究目的还是仅仅为了了解系统。
在前一章中，我们介绍了跟踪子系统和示例程序 `sixth.cc`，从中生成了PCAP或ASCII跟踪文件。
这些跟踪对于使用各种外部工具进行数据分析非常有价值，对于许多用户来说，这种输出数据是收集数据的首选方式（以供外部工具分析）。

然而，除了生成跟踪文件之外，还有一些用例，包括：
1. 生成*不适合PCAP或ASCII跟踪的数据*，例如非数据包数据（例如协议状态机转换）。
2. 对于生成*跟踪文件的磁盘I/O要求* 对于大型仿真而言是禁止的或繁琐的。
3. 在仿真过程中需要*在线数据缩减或计算的需求*。这有一个很好的例子是为仿真定义终止条件，告诉它何时停止，当它收到足够的数据以在某个参数的估计周围形成足够窄的置信区间时。

ns-3数据收集框架旨在提供超出基于跟踪的输出的这些附加功能。我们建议对此主题感兴趣的读者查阅ns-3手册，以获取有关此框架的更详细的介绍；在这里，我们通过一个示例程序总结一些正在开发的功能。
## Example Code
教程示例 `examples/tutorial/seventh.cc` 与我们之前审查过的 `sixth.cc` 示例相似，除了一些更改。首先，它已经通过命令行选项启用了 IPv6 支持：

```cpp
CommandLine cmd;
cmd.AddValue ("useIpv6", "Use Ipv6", useV6);
cmd.Parse (argc, argv);
```

如果用户**指定了 `useIpv6` 选项，程序将使用 IPv6 而不是 IPv4 运行**。可以使用以下方式调用所有支持 CommandLine 对象的 ns-3 程序的帮助选项（请注意使用双引号）：

```bash
./ns3 run "seventh --help"
```

这将产生以下输出：

```plaintext
ns3-dev-seventh-debug [Program Arguments] [General Arguments]

Program Arguments:
    --useIpv6:  Use Ipv6 [false]

General Arguments:
    --PrintGlobals:              Print the list of globals.
    --PrintGroups:               Print the list of groups.
    --PrintGroup=[group]:        Print all TypeIds of group.
    --PrintTypeIds:              Print all TypeIds.
    --PrintAttributes=[typeid]:  Print all attributes of typeid.
    --PrintHelp:                 Print this help message.
```

这个默认值（意味着此处使用的是 IPv4，因为 `useIpv6` 为 false）可以通过切换布尔值来更改：

```bash
./ns3 run "seventh --useIpv6=1"
```

并查看生成的 pcap 文件，比如使用 tcpdump：

```bash
tcpdump -r seventh.pcap -nn -tt
```

这是对 IPv6 支持和命令行的简短讨论，这在本教程的前面部分也有介绍。有关命令行使用的专用示例，请参阅 `src/core/examples/command-line-example.cc`。

现在回到数据收集。在 `examples/tutorial/` 目录中，键入以下命令：`diff -u sixth.cc seventh.cc`，并检查此差异中的一些新行：

```cpp
+  std::string probeType;
+  std::string tracePath;
+  if (useV6 == false)
+    {
   ...
+      probeType = "ns3::Ipv4PacketProbe";
+      tracePath = "/NodeList/*/$ns3::Ipv4L3Protocol/Tx";
+    }
+  else
+    {
   ...
+      probeType = "ns3::Ipv6PacketProbe";
+      tracePath = "/NodeList/*/$ns3::Ipv6L3Protocol/Tx";
+    }
 ...
+   // Use GnuplotHelper to plot the packet byte count over time
+   GnuplotHelper plotHelper;
+
+   // Configure the plot.  The first argument is the file name prefix
+   // for the output files generated.  The second, third, and fourth
+   // arguments are, respectively, the plot title, x-axis, and y-axis labels
+   plotHelper.ConfigurePlot ("seventh-packet-byte-count",
+                             "Packet Byte Count vs. Time",
+                             "Time (Seconds)",
+                             "Packet Byte Count");
+
+   // Specify the probe type, trace source path (in configuration namespace), and
+   // probe output trace source ("OutputBytes") to plot.  The fourth argument
+   // specifies the name of the data series label on the plot.  The last
+   // argument formats the plot by specifying where the key should be placed.
+   plotHelper.PlotProbe (probeType,
+                         tracePath,
+                         "OutputBytes",
+                         "Packet Byte Count",
+                         GnuplotAggregator::KEY_BELOW);
+
+   // Use FileHelper to write out the packet byte count over time
+   FileHelper fileHelper;
+
+   // Configure the file to be written, and the formatting of output data.
+   fileHelper.ConfigureFile ("seventh-packet-byte-count",
+                             FileAggregator::FORMATTED);
+
+   // Set the labels for this formatted output file.
+   fileHelper.Set2dFormat ("Time (Seconds) = %.3e\tPacket Byte Count = %.0f");
+
+   // Specify the probe type, probe path (in configuration namespace), and
+   // probe output trace source ("OutputBytes") to write.
+   fileHelper.WriteProbe (probeType,
+                          tracePath,
+                          "OutputBytes");
+
    Simulator::Stop (Seconds (20));
    Simulator::Run ();
    Simulator::Destroy ();
```

仔细的读者在上面测试 IPv6 命令行属性时会注意到，`seventh.cc` 创建了一些新的输出文件：

- `seventh-packet-byte-count-0.txt`
- `seventh-packet-byte-count-1.txt`
- `seventh-packet-byte-count.dat`
- `seventh-packet-byte-count.plt`
- `seventh-packet-byte-count.png`
- `seventh-packet-byte-count.sh`

这些是通过引入的额外语句创建的；特别是通过 `GnuplotHelper` 和 `FileHelper`。此数据是通过==将数据收集组件连接到 ns-3 跟踪源==，并==将数据编组到格式化的 gnuplot 文件和格式化的文本文件中==产生的。在接下来的几节中，我们将逐一审查每一个。
## GnuplotHelper
`GnuplotHelper` 是一个**面向生成 gnuplot 图表**的 *ns-3 助手对象*，旨在以尽可能少的语句实现常见情况下的操作。它将 ns-3 跟踪源与数据收集系统支持的数据类型连接起来。并非所有 ns-3 跟踪源的数据类型都得到支持，但许多常见的跟踪类型得到了支持，包括具有普通旧数据（POD）类型的 TracedValues。

让我们来看一下这个助手生成的输出：

- `seventh-packet-byte-count.dat`
- `seventh-packet-byte-count.plt`
- `seventh-packet-byte-count.sh`

第一个是一个 gnuplot 数据文件，其中包含一系列用空格分隔的时间戳和数据包字节计数。我们将在下面介绍如何配置此特定的数据输出，但让我们继续查看输出文件。

文件 `seventh-packet-byte-count.plt` 是一个 gnuplot 绘图文件，可以从 gnuplot 中打开。了解 gnuplot 语法的读者可以看到，这将产生一个名为 `seventh-packet-byte-count.png` 的格式化输出 PNG 文件。

最后，==一个小的 shell 脚本 `seventh-packet-byte-count.sh` 通过 gnuplot 运行此绘图文件以生成所需的 PNG==（可以在图像编辑器中查看）；也就是说，执行以下命令：

```bash
sh seventh-packet-byte-count.sh
```

将生成 `seventh-packet-byte-count.png`。为什么一开始就没有生成这个 PNG？答案是通过提供 plt 文件，用户可以在生成 PNG 之前手动配置结果，如果需要的话。

PNG 图像的标题说明了这个图表是“Packet Byte Count vs. Time”的图表，它正在绘制与跟踪源路径对应的探测数据：

```
/NodeList/*/$ns3::Ipv6L3Protocol/Tx
```

请注意==跟踪路径中的通配符==。

总结一下，这个图表捕获的是在 Ipv6L3Protocol 对象的传输跟踪源处观察到的数据包字节的图表；主要是一个方向上的 596 字节的 TCP 段和另一个方向上的 60 字节的 TCP 确认（此跟踪源匹配了两个节点的跟踪源）。

这是如何配置的呢？需要提供一些语句。首先，必须声明并配置 `GnuplotHelper` 对象：

```cpp
// 使用 GnuplotHelper 绘制随时间变化的数据包字节数：
GnuplotHelper plotHelper;

// 配置图表：
// 
plotHelper.ConfigurePlot ("seventh-packet-byte-count",  // 第一个参数：是生成的输出文件的文件名前缀
                          "Packet Byte Count vs. Time", // 第二个参数：图表标题
                          "Time (Seconds)",             // 第三个参数：x 轴标签
                          "Packet Byte Count");         // 第四个参数：y 轴标签
```

到==目前为止，已经配置了一个空的图表==。文件名前缀是第一个参数，图表标题是第二个参数，x 轴标签是第三个参数，y 轴标签是第四个参数。

==下一步是配置数据，这是跟踪源的连接点==。

首先，在程序中上面声明了一些以后会用到的变量：

```cpp
std::string probeType;
std::string tracePath;
probeType = "ns3::Ipv6PacketProbe";
tracePath = "/NodeList/*/$ns3::Ipv6L3Protocol/Tx";
```

我们在这里使用它们：

```cpp
// 指定探测类型、跟踪源路径（在配置命名空间中），以及
// 要绘制的探测输出跟踪源（"OutputBytes"）。第四个参数
// 指定图表上数据系列标签的名称。最后一个参数
// 通过指定键应该放置在何处来格式化图表。

// 格式化图表要素：
plotHelper.PlotProbe (probeType,                     // 探测类型
                      tracePath,                     // 跟踪源路径（在配置命名空间中）
                      "OutputBytes",                 // 要绘制的探测输出跟踪源
                      "Packet Byte Count",           // 图表上数据系列标签的名称
                      GnuplotAggregator::KEY_BELOW); // 指定键应该放置在何处
```

前两个参数是探测类型的名称和跟踪源路径。当尝试使用此框架绘制其他跟踪时，这两个参数可能是最难确定的。这里的探测跟踪是 Ipv6L3Protocol 类的 Tx 跟踪源。当我们检查这个类的实现（`src/internet/model/ipv6-l3-protocol.cc`）时，我们可以观察到：

```cpp
.AddTraceSource ("Tx", "Send IPv6 packet to outgoing interface.",
                 MakeTraceSourceAccessor (&Ipv6L3Protocol::m_txTrace))
```

这说明 Tx 是变量 `m_txTrace` 的名称，它的声明如下：

```cpp
/**
 * \brief Callback to trace TX (transmission) packets.
 */
TracedCallback<Ptr<const Packet>, Ptr<Ipv6>, uint32_t> m_txTrace;
```

事实证明，这个特定的跟踪源签名由 Ipv6PacketProbe 类（我们在这里需要的类）的 Probe 类型支持。请参见文件 `src/internet/model/ipv6-packet-probe.{h,cc}`。

因此，在上面的 `PlotProbe` 语句中，我们看到==该语句正在使用与路径字符串匹配的跟踪源==（识别为探测类型 `Ipv6PacketProbe` 的 ns-3 探测类型）。如果我们不支持此探测类型（匹配跟踪源签名），我们将无法使用此语句（尽管可以使用一些更复杂的底层语句，如手册中所述）。

`Ipv6PacketProbe` 本身导出了一些跟踪源，用于从被探测的 Packet 对象中提取数据：

```cpp
TypeId
Ipv6PacketProbe::GetTypeId ()
{
  static TypeId tid = TypeId ("ns3::Ipv6PacketProbe")
    .SetParent<Probe> ()
    .SetGroupName ("Stats")
    .AddConstructor<Ipv6PacketProbe> ()
    .AddTraceSource ( "Output",
                      "The packet plus its IPv6 object and interface that serve as the output for this probe",
                      MakeTraceSourceAccessor (&Ipv6PacketProbe::m_output))
    .AddTraceSource ( "OutputBytes",
                      "The number of bytes in the packet",
                      MakeTraceSourceAccessor (&Ipv6PacketProbe::m_outputBytes))
  ;
  return tid;
}
```

`PlotProbe` 语句的第三个参数指定我们对此数据包中的字节数感兴趣；具体来说，我们对 `Ipv6PacketProbe` 的 “OutputBytes” 跟踪源感兴趣。最后，该语句的最后两个参数提供了此数据系列的图例（“Packet Byte Count”），以及一个可选的 gnuplot 格式语句（`GnuplotAggregator::KEY_BELOW`），我们希望图例键插入在图表下面。其他选项包括 `NO_KEY`、`KEY_INSIDE` 和 `KEY_ABOVE`。
## Supported Trace Types
截至目前为止，以下跟踪值在 Probes 中得到支持：

| TracedValue 类型 | Probe 类型              | 文件                            |
|-------------------|------------------------|---------------------------------|
| double            | DoubleProbe            | `stats/model/double-probe.h`     |
| uint8_t           | Uinteger8Probe         | `stats/model/uinteger-8-probe.h` |
| uint16_t          | Uinteger16Probe        | `stats/model/uinteger-16-probe.h`|
| uint32_t          | Uinteger32Probe        | `stats/model/uinteger-32-probe.h`|
| bool              | BooleanProbe           | `stats/model/uinteger-16-probe.h`|
| ns3::Time         | TimeProbe              | `stats/model/time-probe.h`       |

截至目前为止，以下 TraceSource 类型由 Probes 支持：

| TracedSource 类型                                | Probe 类型        | Probe 输出   | 文件                                     |
|--------------------------------------------------|-------------------|--------------|------------------------------------------|
| `Ptr<const Packet>`                               | PacketProbe       | OutputBytes   | `network/utils/packet-probe.h`           |
| `Ptr<const Packet>, Ptr<Ipv4>, uint32_t`          | Ipv4PacketProbe    | OutputBytes   | `internet/model/ipv4-packet-probe.h`     |
| `Ptr<const Packet>, Ptr<Ipv6>, uint32_t`          | Ipv6PacketProbe    | OutputBytes   | `internet/model/ipv6-packet-probe.h`     |
| `Ptr<const Packet>, Ptr<Ipv6>, uint32_t`          | Ipv6PacketProbe    | OutputBytes   | `internet/model/ipv6-packet-probe.h`     |
| `Ptr<const Packet>, const Address&`               | ApplicationPacketProbe | OutputBytes | `applications/model/application-packet-probe.h`|

正如可以看到的那样，只有很少一部分跟踪源得到支持，而且它们都是以输出 Packet 大小（以字节为单位）为导向的。然而，使用这些辅助工具可以支持大多数作为 TracedValues 提供的基本数据类型。
## FileHelper
FileHelper 类只是前面 GnuplotHelper 示例的变体。该示例程序提供了相同时间戳数据的格式化输出，如下所示：

```
Time (Seconds) = 9.312e+00    Packet Byte Count = 596
Time (Seconds) = 9.312e+00    Packet Byte Count = 564
```

提供了两个文件，一个用于节点 "0"，另一个用于节点 "1"，正如文件名所示。我们逐段来看代码：

```cpp
// Use FileHelper to write out the packet byte count over time
FileHelper fileHelper;

// Configure the file to be written, and the formatting of output data.
fileHelper.ConfigureFile ("seventh-packet-byte-count",
                          FileAggregator::FORMATTED);
```

文件帮助器文件前缀是==第一个参数==，接下来是==格式说明符==。

格式化的一些其他选项包括 `SPACE_SEPARATED`、`COMMA_SEPARATED` 和 `TAB_SEPARATED`。用户可以使用格式字符串更改格式（如果指定了 `FORMATTED`）：

```cpp
// Set the labels for this formatted output file.
fileHelper.Set2dFormat ("Time (Seconds) = %.3e\tPacket Byte Count = %.0f");
```

最后，必须==挂接感兴趣的跟踪源==。再次使用此示例中的 `probeType` 和 `tracePath` 变量，并==钩住探针的输出跟踪源 "OutputBytes"==：

```cpp
// Specify the probe type, trace source path (in configuration namespace), and
// probe output trace source ("OutputBytes") to write.
fileHelper.WriteProbe (probeType,
                       tracePath,
                       "OutputBytes");
```

在此跟踪源说明符中的通配符字段匹配两个跟踪源。与 GnuplotHelper 示例不同的是，在该示例中，两个独立的文件被写入磁盘。
## Summary
Data collection support is new as of ns-3.18, and basic support for providing time series output has been added. The basic pattern described above may be replicated within the scope of support of the existing probes and trace sources. More capabilities including statistics processing will be added in future releases.