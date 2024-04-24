# Chapter 1 Mininet Walkthrough

## Part 1 Everyday Mininet Usage

- `$` preceeds Linux commands that should be typed at the shell prompt
- `mininet>` preceeds Mininet commands that should be typed at Mininet’s CLI,
- `#` preceeds Linux commands that are typed at a root shell prompt

### Display Startup Options

display a help message describing Mininet’s startup options

```bash
sudo mn -h
```

### Start Wireshark

To view control traffic using the OpenFlow Wireshark dissector(解析器), first open wireshark in the background:

```bash
sudo wireshark &
```

And then it will show that:

```bash
❯ sudo wireshark &
[1] 966630
[1]  + 966630 suspended (tty output)  sudo wireshark 
```

### Interact with Hosts and Switches

Start a minimal topology and enter the CLI:

```bash
sudo mn
```

The default topology is the `minimal` topology, which includes

- one OpenFlow kernel switch connected to two hosts,
- and the OpenFlow reference controller.

This topology could also be specified on the command line with `--topo=minimal`. Other topologies are also available out of the box; see the `--topo` section in the output of `mn -h`.

All four entities (2 host processes, 1 switch process, 1 basic controller) are now running in the Virtual Machine. The controller can be outside the VM, and instructions for that are at the bottom.

If no specific test is passed as a parameter, the Mininet CLI comes up:

```bash
❯ sudo mn       
[sudo] root-hbx 的密码： 
*** Creating network
*** Adding controller
*** Adding hosts:
h1 h2 
*** Adding switches:
s1 
*** Adding links:
(h1, s1) (h2, s1) 
*** Configuring hosts
h1 h2 
*** Starting controller
c0 
*** Starting 1 switches
s1 ...
*** Starting CLI:
mininet> 
```

#### Basic Orders

- Display Mininet CLI commands:

```bash
mininet> help
```

- Display nodes:

```bash
mininet> nodes
```

- Display links:

```bash
mininet> net
```

- Dump information about all nodes:

```bash
mininet> dump
```

and You should see the switch and two hosts listed.

```bash
# the demo for the orders above
mininet> nodes
available nodes are: 
c0 h1 h2 s1
```

```bash
# the demo for the orders above
mininet> net
h1 h1-eth0:s1-eth1
h2 h2-eth0:s1-eth2
s1 lo:  s1-eth1:h1-eth0 s1-eth2:h2-eth0
c0
```

```bash
# the demo for the orders above
mininet> dump
<Host h1: h1-eth0:10.0.0.1 pid=968943> 
<Host h2: h2-eth0:10.0.0.2 pid=968945> 
<OVSSwitch s1: lo:127.0.0.1,s1-eth1:None,s1-eth2:None pid=968950> 
<OVSController c0: 127.0.0.1:6653 pid=968936> 
mininet> 
```

#### Orders concerning Individuals

You should be careful with "namespace" here! It's a bit confusing.

If the first string typed into the Mininet CLI is a host, switch or controller name, the command is executed on that node. Run a command on a host process:

```bash
mininet> h1 ifconfig -a
```

You should see the host’s `h1-eth0` and loopback (`lo`) interfaces.

Note that this interface (`h1-eth0`) is not seen by the primary Linux system when `ifconfig` is run, because it is specific to the _network namespace of the host_ process.

In contrast, the switch by default runs in the root network namespace, so running a command on the “switch” is the same as running it from a regular terminal:

```bash
mininet> s1 ifconfig -a
```

This will show the switch interfaces, plus the VM’s connection out (`eth0`).

For other examples highlighting that the __hosts have isolated network state__, run `arp` and `route` on both `s1` and `h1`.

It would be possible to place every host, switch and controller in its own isolated network namespace, but there’s no real advantage to doing so, unless you want to replicate a complex multiple-controller network. Mininet does support this; see the `--innamespace` option.

Note that __only the network is virtualized__; each host process sees the same set of processes and directories. For example, print the process list from a host process:

```bash
mininet> h1 ps -a
    PID TTY          TIME CMD
   2209 tty2     00:18:15 Xorg
   2346 tty2     00:00:00 gnome-session-b
 966451 pts/1    00:00:00 zsh
 966489 pts/1    00:00:00 zsh
 966490 pts/1    00:00:00 zsh
 966492 pts/1    00:00:00 gitstatusd-linu
 966630 pts/1    00:00:00 sudo
 968897 pts/1    00:00:00 sudo
 968925 pts/2    00:00:00 mn
 969013 pts/41   00:00:00 ovs-testcontrol
 975524 pts/42   00:00:00 ps
```

 This should be the exact same as that seen by the root network namespace:

```bash
mininet> s1 ps -a
    PID TTY          TIME CMD
   2209 tty2     00:18:15 Xorg
   2346 tty2     00:00:00 gnome-session-b
 966451 pts/1    00:00:00 zsh
 966489 pts/1    00:00:00 zsh
 966490 pts/1    00:00:00 zsh
 966492 pts/1    00:00:00 gitstatusd-linu
 966630 pts/1    00:00:00 sudo
 968897 pts/1    00:00:00 sudo
 968925 pts/2    00:00:00 mn
 969013 pts/41   00:00:00 ovs-testcontrol
 975633 pts/45   00:00:00 ps
mininet> 
```

It would be possible to __use separate process spaces with Linux containers__, but currently __Mininet doesn’t__ do that. \

Having everything run in the “root” process namespace is convenient for debugging, because it allows you to see all of the processes from the console using `ps`, `kill`, etc.

### Test connectivity between hosts

Now, verify that you can ping from host 0 to host 1:

```bash
mininet> h1 ping -c 1 h2
```

If a string appears later in the command with a node name, that node name is replaced by its IP address; this happened for h2.

```bash
mininet> h1 ping -c 1 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=6.37 ms

--- 10.0.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 6.372/6.372/6.372/0.000 ms
```

You should see OpenFlow control traffic:

1. The first host ARPs for the MAC address of the second, which causes a `packet_in` message to go to the controller.
2. The controller then sends a `packet_out` message to flood the broadcast packet to other ports on the switch (in this example, the only other data port).
3. The second host sees the ARP request and sends a reply.
4. This reply goes to the controller, which sends it to the first host and pushes down a flow entry.
5. Now the first host knows the MAC address of the second, and can send its ping via an ICMP Echo Request.
6. This request, along with its corresponding reply from the second host, both go the controller and result in a flow entry pushed down (along with the actual packets getting sent out).

If we try to repeat this ping again:

```bash
mininet> h1 ping -c 1 h2
PING 10.0.0.2 (10.0.0.2) 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.854 ms

--- 10.0.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.854/0.854/0.854/0.000 ms
mininet> 

```

At the first time, it costs us 6.37ms while costs only 0.854ms now!

- We can see a much lower `ping` time for the second try (< 100us).
- A flow entry covering ICMP `ping` traffic __was previously installed in the switch__, so no control traffic was generated, and __the packets immediately pass through__ the switch.

An easier way to run this test is to use the Mininet CLI built-in `pingall` command, which does an all-pairs `ping`:

```bash
mininet> pingall
```

### Run a simple web server and client

Remember that `ping` isn’t the only command you can run on a host! Mininet hosts can run any command or application that is available to the underlying Linux system (or VM) and its file system.

You can also enter any `bash` command, including job control (`&`, `jobs`, `kill`, etc..)

Firstly, check the version of your python:

```bash
mininet> py sys.version
3.10.12 (main, Nov 20 2023, 15:14:05) [GCC 11.4.0]
```

Next, try starting a simple HTTP server on `h1`, making a request from `h2`, then shutting down the web server:

```bash
mininet> h1 python -m http.server 80 &
mininet> h2 wget -O - h1
--2024-04-13 16:02:28--  http://10.0.0.1/
正在连接 10.0.0.1:80... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度： 3254 (3.2K) [text/html]
正在保存至: ‘STDOUT’

-                     0%[                    ]       0  --.-KB/s               <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN" "http://www.w3.org/TR/html4/strict.dtd">
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<title>Directory listing for /</title>
</head>
<body>
<h1>Directory listing for /</h1>
<hr>
<ul>
<li><a href=".bash_history">.bash_history</a></li>
<li><a href=".bash_logout">.bash_logout</a></li>
<li><a href=".bashrc">.bashrc</a></li>
<li><a href=".cache/">.cache/</a></li>
<li><a href=".config/">.config/</a></li>
<li><a href=".conky/">.conky/</a></li>
<li><a href=".conkyrc">.conkyrc</a></li>
<li><a href=".continue/">.continue/</a></li>
<li><a href=".dotnet/">.dotnet/</a></li>
<li><a href=".fonts/">.fonts/</a></li>
<li><a href=".gitconfig">.gitconfig</a></li>
<li><a href=".gnupg/">.gnupg/</a></li>
<li><a href=".icons/">.icons/</a></li>
<li><a href=".lesshst">.lesshst</a></li>
<li><a href=".local/">.local/</a></li>
<li><a href=".moc/">.moc/</a></li>
<li><a href=".mozilla/">.mozilla/</a></li>
<li><a href=".nv/">.nv/</a></li>
<li><a href=".oh-my-zsh/">.oh-my-zsh/</a></li>
<li><a href=".p10k.zsh">.p10k.zsh</a></li>
<li><a href=".pki/">.pki/</a></li>
<li><a href=".profile">.profile</a></li>
<li><a href=".python_history">.python_history</a></li>
<li><a href=".rest-client/">.rest-client/</a></li>
<li><a href=".shell.pre-oh-my-zsh">.shell.pre-oh-my-zsh</a></li>
<li><a href=".ssh/">.ssh/</a></li>
<li><a href=".sudo_as_admin_successful">.sudo_as_admin_successful</a></li>
<li><a href=".themes/">.themes/</a></li>
<li><a href=".triton/">.triton/</a></li>
<li><a href=".venv/">.venv/</a></li>
<li><a href=".vim/">.vim/</a></li>
<li><a href=".viminfo">.viminfo</a></li>
<li><a href=".vimrc">.vimrc</a></li>
<li><a href=".vmware/">.vmware/</a></li>
<li><a href=".vscode/">.vscode/</a></li>
<li><a href=".zcompdump">.zcompdump</a></li>
<li><a href=".zcompdump-root-hbx-5.8.1">.zcompdump-root-hbx-5.8.1</a></li>
<li><a href=".zcompdump-root-hbx-5.8.1.zwc">.zcompdump-root-hbx-5.8.1.zwc</a></li>
<li><a href=".zsh_history">.zsh_history</a></li>
<li><a href=".zshrc">.zshrc</a></li>
<li><a href="ai/">ai/</a></li>
<li><a href="Coding/">Coding/</a></li>
<li><a href="CS/">CS/</a></li>
<li><a href="Downloads/">Downloads/</a></li>
<li><a href="HTML%20%E5%85%A5%E9%97%A8%E6%95%99%E7%A8%8B.pdf">HTML 入门教程.pdf</a></li>
<li><a href="issue/">issue/</a></li>
<li><a href="killm.sh">killm.sh</a></li>
<li><a href="llm.sh">llm.sh</a></li>
<li><a href="Mimosa-Dark/">Mimosa-Dark/</a></li>
<li><a href="mininet/">mininet/</a></li>
<li><a href="Obsidian_Linux/">Obsidian_Linux/</a></li>
<li><a href="obsidian_note/">obsidian_note/</a></li>
<li><a href="snap/">snap/</a></li>
<li><a href="victorconky/">victorconky/</a></li>
<li><a href="windows%E4%B8%8A%E7%9A%84%E9%87%8D%E8%A6%81%E6%96%87%E4%BB%B6/">windows上的重要文件/</a></li>
<li><a href="%E4%B8%8B%E8%BD%BD/">下载/</a></li>
<li><a href="%E5%85%AC%E5%85%B1%E7%9A%84/">公共的/</a></li>
<li><a href="%E5%9B%BE%E7%89%87/">图片/</a></li>
<li><a href="%E6%96%87%E6%A1%A3/">文档/</a></li>
<li><a href="%E6%A1%8C%E9%9D%A2/">桌面/</a></li>
<li><a href="%E6%A8%A1%E6%9D%BF/">模板/</a></li>
<li><a href="%E8%A7%86%E9%A2%91/">视频/</a></li>
<li><a href="%E9%9F%B3%E4%B9%90/">音乐/</a></li>
</ul>
<hr>
</body>
</html>
-                   100%[===================>]   3.18K  --.-KB/s    用时 0s  

2024-04-13 16:02:28 (279 MB/s) - 已写入至标准输出 [3254/3254]
```

### Exit and Cleanup

Exit the mininet CLI and back to your terminal

```bash
mininet> exit
```

If the mininet crashes for some reason, you should clean it up

```bash
sudo mn -c
```

## Part 2: Advanced Startup Options

### Changing Topology Size and Type

The default topology is a single switch connected to two hosts listed in Part1.

You could change this to a different topo with `--topo`, and pass parameters for that topology’s creation.

For example, to verify all-pairs ping connectivity with one switch and three hosts:
Run a regression test:

```bash
# --test + order / --topo + Sw/Hst configuration
$ sudo mn --test pingall --topo single,3
```

Another example, with a linear topology (where each switch has one host, and all switches connect in a line):

```bash
$ sudo mn --test pingall --topo linear,4
```

Parametrized topologies are one of Mininet’s most useful and powerful features.

### Link variations

Mininet 2.0 allows you to set link parameters, and these can even be set automatially from the command line:

```bash
 $ sudo mn --link tc,bw=10,delay=10ms
 mininet> iperf
 ...
 mininet> h1 ping -c10 h2
```

- tc: 表示使用 Linux 中的 Traffic Control（流量控制）工具来模拟链路
- bw: bandwidth of this link
- delay: the delay for each link is 10 ms

From the message listed above, we can know that:

> the round trip time (RTT) should be about 40 ms (10x4), since the ICMP request traverses two links (one to the switch, one to the destination) and the ICMP reply traverses two links coming back.

You can customize each link using [Mininet’s Python API](https://github.com/mininet/mininet/wiki/Introduction-to-Mininet), but for now you will probably want to continue with the walkthrough.

### Adjustable Verbosity

"Adjustable Verbosity(冗长)" 意味着能够根据需要调整程序或系统输出的详细程度：

- 在软件开发或系统管理中，通常会有不同级别的详细程度来描述输出信息，从简洁的摘要到详细的调试信息。
- 通过可调节的详细程度，用户可以根据自己的需求来控制输出的丰富程度，这对于诊断问题、调试代码或监控系统状态非常有用。
- 等价于ns-3中的日志组件 Log ( )

The default verbosity level is `info`, which prints what Mininet is doing during startup and teardown. Compare this with the full `debug` output with the `-v` param:

```bash
$ sudo mn -v debug
...
mininet> exit
```

Lots of extra detail will print out. Now try `output`, a setting that prints CLI output and little else:

```bash
$ sudo mn -v output
mininet> exit
```

Outside the CLI, other verbosity levels can be used, such as `warning`, which is used with the regression tests to hide unneeded function output.

### Custom Topologies

In fact, there are 2 main methods to customize your own topology

You can learn more details in my [personal study note](https://github.com/root-hbx/SDN_Spring_2024) concerning different ways to design topology

Custom topologies can be easily defined as well, using a simple Python API, and an example is provided in `custom/topo-2sw-2host.py`. This example connects two switches directly, with a single host off each switch:

When a custom mininet file is provided, it can add new topologies, switch types, and tests to the command-line. For example:

```bash
$ sudo mn --custom ~/mininet/custom/topo-2sw-2host.py --topo mytopo --test pingall
```

### ID = MAC

- By __default, hosts start with randomly assigned MAC addresses__.
- This can make debugging tough, because every time the Mininet is created, the MACs change, so correlating control traffic with specific hosts is tough.

The `--mac` option is super-useful, and __sets the host MAC and IP addrs to__ small, unique, easy-to-read __IDs__.

Before:

```bash
$ sudo mn
...
mininet> h1 ifconfig
h1-eth0  Link encap:Ethernet  HWaddr f6:9d:5a:7f:41:42
          inet addr:10.0.0.1  Bcast:10.255.255.255  Mask:255.0.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:6 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:392 (392.0 B)  TX bytes:392 (392.0 B)
mininet> exit
```

After:

```bash
$ sudo mn --mac
...
mininet> h1 ifconfig
h1-eth0  Link encap:Ethernet  HWaddr 00:00:00:00:00:01
          inet addr:10.0.0.1  Bcast:10.255.255.255  Mask:255.0.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
mininet> exit
```

- In contrast, the MACs for switch data ports reported by Linux will remain random.
- This is because you can ‘assign’ a MAC to a data port using OpenFlow.
- This is a somewhat subtle point which you can probably ignore for now.

### XTerm Display

For more complex debugging, you can start Mininet so that it spawns one or more xterms.

To start an `xterm` for every host and switch, pass the `-x` option:

```bash
sudo mn -x
```

After a second, the xterms will pop up, with automatically set window names.

Alternately, you can bring up additional xterms as shown below.

- By default, only the hosts are put in a separate namespace
- the window for each switch is unnecessary (that is, equivalent to a regular terminal), but can be a convenient place to run and leave up switch debug commands, such as flow counter dumps.

Xterms are also useful for __running interactive commands__ that you may need to cancel, for which you’d like to see the output.

For example:

In the xterm labeled “switch: s1 (root)”, run:

```bash
# ovs-ofctl dump-flows tcp:127.0.0.1:6654
```

Nothing will print out; the switch has no flows added. To use `ovs-ofctl` with other switches, start up mininet in _verbose mode (详细模式)_ and look at the passive listening ports for the switches when they’re created.

Now, in the xterm labeled “host: h1”, run:

```bash
# ping 10.0.0.2
```

Go back to `s1` and dump the flows: # ovs-ofctl dump-flows tcp:127.0.0.1:6654

You should see multiple flow entries now.

Alternately (and generally more convenient), you could use the `dpctl` command built into the Mininet CLI without needing any xterms or manually specifying the IP and port of the switch.

You can tell whether an xterm is in the root namespace by checking ifconfig; if all interfaces are shown (including `eth0`), it’s in the root namespace. Additionally, its title should contain “(root)”.

Close the setup, from the Mininet CLI:

```bash
mininet> exit
```

and the xterms should automatically close.

### Other Switch Types

Other switch types can be used. For example, to run the user-space switch:

```bash
$ sudo mn --switch user --test iperf
```

Note the __much lower TCP iperf-reported bandwidth__ compared to that seen earlier with the kernel switch.

Traits:

- If you do the ping test shown earlier, you should notice a much higher delay, since __now packets must endure additional kernel-to-user-space transitions__.
- The ping time will be more variable, as the user-space process representing the host may be scheduled in and out by the OS.
- On the other hand, the user-space switch __can be a great starting point for implementing new functionality__, especially where software performance is not critical.

Another example switch type is Open vSwitch (OVS), which comes preinstalled on the Mininet VM. The iperf-reported TCP bandwidth should be similar to the OpenFlow kernel module, and possibly faster:

```bash
$ sudo mn --switch ovsk --test iperf
```

### Mininet Benchmark

To record the time to set up and tear down a topology, use test ‘none’:

```bash
$ sudo mn --test none
```

Add some supplementary materials:

> Benchmark（基准测试）是通过 __在已知条件下运行一系列测试__ 来评估系统、设备或软件性能的过程。基准测试 __旨在量化系统在特定负载或条件下的性能水平__，通常涉及测量各种指标，如处理速度、内存利用率、响应时间、吞吐量等。这些测试可以用于比较不同系统之间的性能差异，或者评估系统在不同条件下的性能表现。

> 在计算机领域，基准测试可用于评估 CPU、内存、存储设备、网络设备等硬件的性能，也可用于评估软件、数据库系统、操作系统和网络性能。Benchmark 结果可以帮助用户选择最佳的硬件或软件配置，或者优化系统以提高性能。

> 对 Mininet 进行基准测试可以帮助评估其在不同条件下的性能表现，例如，创建大型网络拓扑时的性能、数据包转发速率、延迟和吞吐量等。Benchmark 测试还可以用于比较不同版本的 Mininet 或不同硬件配置下的性能差异。

## Part 3: Mininet Command-Line Interface (CLI) Commands

### Display Options

To see the list of Command-Line Interface (CLI) options, start up a minimal topology and leave it running. Build the Mininet:

```
$ sudo mn
```

Display the options:

```
mininet> help
```

### Python Interpreter

- If __the first phrase__ on the Mininiet command line is `py`, then that command is __executed with Python__.
- This might be useful for extending Mininet, as well as probing its inner workings.
- Each host, switch, and controller has an associated Node object.

At the Mininet CLI, run:

```
mininet> py 'hello ' + 'world'
```

Print the _accessible local variables_:

```
mininet> py locals()
```

Next, see the _methods and properties_ available for a node, using the dir() function:

```
mininet> py dir(s1)
```

You can read the on-line _documentation for methods available on a node_ by using the help() function:

```
mininet> py help(h1) (Press "q" to quit reading the documentation.)
```

You can also evaluate _methods of variables_:

```
mininet> py h1.IP()
```

Here are some results below:

```bash
mininet> py 'hello' + 'hbx'
hellohbx
```

```bash
mininet> py locals()
{'net': <mininet.net.Mininet object at 0x7fef65e34fa0>, 'h1': <Host h1: h1-eth0:10.0.0.1 pid=1013172> , 'h2': <Host h2: h2-eth0:10.0.0.2 pid=1013174> , 's1': <OVSSwitch s1: lo:127.0.0.1,s1-eth1:None,s1-eth2:None pid=1013179> , 'c0': <OVSController c0: 127.0.0.1:6653 pid=1013165> }
```

```bash
mininet> py dir(s1)
['IP', 'MAC', 'OVSVersion', 'TCReapply', '__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', '_popen', '_uuids', 'addIntf', 'argmax', 'attach', 'batch', 'batchShutdown', 'batchStartup', 'bridgeOpts', 'checkSetup', 'cleanup', 'cmd', 'cmdPrint', 'cmds', 'commands', 'config', 'configDefault', 'connected', 'connectionsTo', 'controlIntf', 'controllerUUIDs', 'datapath', 'decoder', 'defaultDpid', 'defaultIntf', 'delIntf', 'deleteIntfs', 'detach', 'dpctl', 'dpid', 'dpidLen', 'execed', 'failMode', 'fdToNode', 'inNamespace', 'inToNode', 'inband', 'intf', 'intfIsUp', 'intfList', 'intfNames', 'intfOpts', 'intfs', 'isOldOVS', 'isSetup', 'lastCmd', 'lastPid', 'linkTo', 'listenPort', 'master', 'monitor', 'mountPrivateDirs', 'name', 'nameToIntf', 'newPort', 'opts', 'outToNode', 'params', 'pexec', 'pid', 'pollOut', 'popen', 'portBase', 'ports', 'privateDirs', 'protocols', 'read', 'readbuf', 'readline', 'reconnectms', 'sendCmd', 'sendInt', 'setARP', 'setDefaultRoute', 'setHostRoute', 'setIP', 'setMAC', 'setParam', 'setup', 'shell', 'slave', 'start', 'startShell', 'stdin', 'stdout', 'stop', 'stp', 'terminate', 'unmountPrivateDirs', 'vsctl', 'waitExited', 'waitOutput', 'waitReadable', 'waiting', 'write']
```

```bash
mininet> py h1.IP()
10.0.0.1
mininet> 
```

### Link Up/Down

For fault tolerance testing, it can be helpful to bring links up and down.

To disable both halves of a virtual ethernet pair:

```
mininet> link s1 h1 down
```

You should see an OpenFlow Port Status Change notification get generated. To bring the link back up:

```
mininet> link s1 h1 up
```

### XTerm Display

To display an xterm for h1 and h2:

```
mininet> xterm h1 h2
```

## Part 4: Python API Examples

The [examples directory](https://github.com/mininet/mininet/tree/master/examples) in the Mininet source tree includes examples of how to use Mininet’s Python API, as well as potentially useful code that has not been integrated into the main code base.

### SSH daemon per host

One example that may be particularly useful runs an SSH daemon on every host:

```
$ sudo ~/mininet/examples/sshd.py
```

From another terminal, you can ssh into any host and run interactive commands:

```
$ ssh 10.0.0.1
$ ping 10.0.0.2
...
$ exit
```

Exit SSH example mininet:

```
$ exit
```

You will wish to revisit the examples after you’ve read the [Introduction to Mininet](https://github.com/mininet/mininet/wiki/Introduction-to-Mininet), which introduces the Python API.

### Other Uses

It's highly recommended to refer to [Intro2Mininet](https://github.com/mininet/mininet/wiki/Introduction-to-Mininet)

## Part 5: Next Steps to master Mininet

Congrats! You’ve completed the Mininet Walkthrough. Feel free to try out new topologies and controllers or check out the source code.

- If you haven’t done so yet, you should definitely go through the [OpenFlow tutorial](https://github.com/mininet/openflow-tutorial/wiki)
- Although you can get reasonably far using Mininet’s CLI, Mininet becomes much more useful and powerful when you master its Python API. The [Introduction to Mininet](https://github.com/mininet/mininet/wiki/Introduction-to-Mininet) provides an introduction to Mininet and its Python API.
