# Chapter 0 Configuration of Experimental Environment

Aim
> 1. 使用 Mininet 的Python API搭建 k=4 的 fat tree 拓扑； 
> 2. 使用 pingall 查看各主机之间的连通情况; 
> 3. 若主机之间未连通，分析原因并解决（使用 wireshark 抓包分析） 
> 4. 若主机连通，分析数据包的路径（ovs-appctl fdb/show 查看MAC表） 
> 5. 不可以使用controller

Reference
- [Mininet](http://mininet.org/)
- [OpenVSwitch](https://www.openvswitch.org/)
- [WireShark](https://www.wireshark.org/)
## Part 1 实验环境搭建
方式1：
- 使用 virtual box 镜像搭建 虚拟机软件 virtual box ，可在[官网](https://download.virtualbox.org/virtualbox/6.1.32/VirtualBox-6.1.32-149290-Win.exe)查询
- 本实验提供 virtual box 虚拟机的镜像文件（ sdn_exp_2023.ova ）已配置 Mininet 和 Ryu 
- 环境搭建步骤如下： 安装 virtual box 导入镜像文件 sdn_exp_2023.ova （ root 账户密码并未设置，需要的同学可以参考 sudo passwd root 指令）

方式2：
- 使用 VMWare 镜像搭建 虚拟机软件 VMWare ，可在官网自行下载
- 本实验也提供 VMWare 虚拟机镜像文件（ sdn_exp_2023_vmware ），已配置 Mininet 和 Ryu 。
- 环境搭建步骤如下： 
	- 安装 VMWare 
	- 导入镜像文件 sdn_exp_2023_vmware （密码 sdn ）

方式3：
源码安装
```bash
# 参考视频 
# `Workstaion`和`Ubuntu`的安装：https://www.bilibili.com/video/BV1ng4y1z77g 
# SDN环境搭建（`Mininet`）：https://www.bilibili.com/video/BV1nC4y1x7Z8 

# 安装mininet 
git clone https://github.com/mininet/mininet.git 
cd mininet/util 
sudo ./install.sh -n3v 

# 安装wireshark 
sudo add-apt-repository ppa:wireshark-dev/stable 
sudo apt update 
sudo apt install wireshark
```

说明：
针对 Apple Silicon
- 本质上安装任一Linux虚拟机都可以使用
	- VMware
	- VirtualBox
	- Parallels Desktop
	- ...
- 但是Orbstack不行！
	- 它轻量化，既可以开docker也可以开vm
	- 它的linux虚拟机内核没有支持openvswitch
	- 除非你手动编译内核 : )
- 具体问题详见笔者在mininet-github中提出的[issue](https://github.com/mininet/mininet/issues/1221)
## Part 2 实验工具介绍
### 2.1 三板斧
- mininet：用来在单台计算机上创建一个包含多台网络设备的虚拟网络
- Open vSwitch : Mininet 中使用的虚拟交换机
- WireShark : 抓包工具

### 2.2 Mininet
- 启动
```bash
# shell prompt 
mn -h          # 查看mininet命令中的各个选项 
sudo mn -c # 不正确退出时清理mininet 
sudo mn      # 创建默认拓扑，两个主机h1、h2连接到同一交换机s1
```

![[Pasted image 20240412095839.png|400]]

- 常用命令
```bash
# inputs in Mininet CLI
nodes
links
net  # show the whole network topo now
dump  # show the detailed information of current net-topo
xterm h1  # open a Terminal Simulator for node-h1
sh [CMD]  # 
h1 ping -c3 h2  # h1 send PING to h2 for 3 times
ping all  # PING in every node-pair
h1 ifconfig  # lookup the Interface and configuration of h1
h1 arp  # lookup the ARP map of h1
link s1 h1 down/up  # disconnect/connect the link between s1 and h1
exit # exit the mininet CLI
```

![[Pasted image 20240412101100.png|400]]

### 2.3 创建拓扑
#### CLI 创建
原始版，详见Lecture 4内容
```bash
sudo mn --mac --topo=tree,m,m
```
- --mac 指定mac地址从1开始递增，而不是无序的mac，方便观察
- --topo 指定拓扑参数，可选用single和linear等参数

#### 自建拓扑 - Method1
- 用 Mininet Python API 创建自定义拓扑
- 通过命令行运行 pwd = 'mininet/custom/topo-2sw-2host.py'
```python
from mininet.topo 
import Topo 
class MyTopo( Topo ): 
	"Simple topology example." 
	def build( self ): 
		"Create custom topo." 
		# Add hosts and switches 
		leftHost = self.addHost( 'h1' ) 
		rightHost = self.addHost( 'h2' ) 
		leftSwitch = self.addSwitch( 's3' ) 
		rightSwitch = self.addSwitch( 's4' ) 
		# Add links 
		self.addLink( leftHost, leftSwitch ) 
		self.addLink( leftSwitch, rightSwitch ) 
		self.addLink( rightSwitch, rightHost )

topos = { 'mytopo': ( lambda: MyTopo() ) }
```

运行
```bash
mn> cd ~/sdn/mininet/custom 
mn> sudo mn --custom topo-2sw-2host.py --topo mytopo --controller=none
```

#### 自建拓扑 - Method2
```python
# sudo python topo_recommend.py 
from mininet.topo import Topo 
from mininet.net import Mininet 
from mininet.cli import CLI 
from mininet.log import setLogLevel 

class S1H2(Topo): 
	def build(self): 
		s1 = self.addSwitch('s1') 
		h1 = self.addHost('h1') 
		h2 = self.addHost('h2') 
		
		self.addLink(s1, h1) 
		self.addLink(s1, h2) 
		
	def run(): 
		topo = S1H2() 
		net = Mininet(topo) 
		net.start() 
		CLI(net) 
		net.stop() 
		
if __name__ == '__main__': 
	setLogLevel('info') # output, info, debug 
	run()
```

运行
```bash
sudo python topo_recommend.py
```
### 2.4 OpenVSwitch (OVS)
#### 查看交换机基本信息
```bash
su
mn
ovs-vsctl show
```
- `ovs-vsctl` 命令允许查看、修改和管理 OVS 的配置，包括网桥、端口、控制器、流表等
- `show` 参数指示该命令显示当前 OVS 的配置信息
- vsctl 的全称是 "Virtual Switch Control"

![[Pasted image 20240412103105.png|400]]

#### 生成树协议
```bash
ovs-vsctl set bridge s1 stp_enable=true # open "STP" for Sw.s1
ovs-vsctl get bridge s1 stp_enable # check if STP is open for s1
ovs-vsctl list bridge # 列出所有 OVS 网桥及其相关信息
```

#### 查看mac表
- 启动mininet，记得要禁用控制器，否则MAC表可能学习不到内容
- 对每个Switch执行`ovs-vsctl del-fail-mode Node_Name`，否则MAC表仍可能学不到内容
- `pingall`令所有主机发送数据包，防止“沉默主机”现象
- `ovs-appctl fdb/show Node_Name`查看每个节点的MAC表

![[Pasted image 20240412103849.png|400]]

![[Pasted image 20240412103939.png|400]]

del-fail-mode XXX: 
>该选项表示“删除先前设置的故障模式”，以允许交换机重新恢复到默认行为或者进行新的配置

![[Pasted image 20240412103958.png|400]]

![[Pasted image 20240412104030.png|400]]

ovs-appctl fdb/show XXX:
>显示指定网桥的转发数据库（FDB）内容
>- XXX: 网桥的名称or标识符
>- FDB：是一个表，存放的是 _MAC地址和与之关联的端口_ 信息

### 2.5 WireShark
- 抓交换机的packet
	- `sudo wireshark`，并选择相应的端口
- 抓主机的packet
	- 在mininet CLI中执行`xterm h1`，打开host1的终端
- 在h1终端里运行wireshark

## Part 3 实验解析
>我的实验环境
>- Linux Ubuntu 22.04LTS 物理机

### 3.1 实验代码
```python
from mininet.topo import Topo
from mininet.net import Mininet
from mininet.cli import CLI
from mininet.log import setLogLevel

'''
Tutorial
- I prefer to use Method 1, which is to build the Fat-Tree topology via "Mininet Python API" and use CLI to run
- pwd:  ~/mininet/custom/lab1_fattree.py
- When you run the script, you should use the orders below:
    > cd ~/sdn/mininet/custom
    > sudo mn --custom lab1_fattree.py --topo fattreetopo --controller=none

'''

class FatTree(Topo):
    # offer the configuration of Fat-Tree
    def build(self, k=4):
        # total number in this topology
        self.k = k
        self.pods = k
        self.aggrSw = self.pods * (k // 2)
        self.edgeSw = self.pods * (k // 2)
        self.coreSw = (k // 2) ** 2
        # host number in each pod
        self.PodHost = (k // 2) ** 2

        # utilize the arguments above to build topo
        self.addCoreSw()
        self.addAggrSw()
        self.addEdgeSw()
        self.addHosts()
        self.setLink()

    def addCoreSw(self):
        for sw in range(self.coreSw):
            self.addSwitch('core{}'.format(sw + 1), 
                         failMode='standalone', stp=True)
        # coreSw is identified by its own ID, which means it can be presented as 1 element turple

    def addAggrSw(self):
        for pod in range(self.pods):
            for sw in range(self.k // 2):
                self.addSwitch('aggr{}{}'.format(pod + 1, sw + 1), 
                             failMode='standalone', stp=True)
        # aggrSw have to be presented as (pod, sw) in turple

    def addEdgeSw(self):
        for pod in range(self.pods):
            for sw in range(self.k // 2):
                self.addSwitch('edge{}{}'.format(pod + 1, sw + 1), 
                             failMode='standalone', stp=True)
        # edgeSw have to be presented as (pod, sw) in turple

    def addHosts(self):
        for pod in range(self.pods):
            for sw in range(self.k // 2):
                for hst in range(self.k // 2):
                    self.addHost('host{}{}{}'.format(pod + 1, sw + 1, hst + 1), 
                               failMode='standalone', stp=True)
        # host have to be presented as (pod, sw, hst) in turple
    
    def setLink(self):
        for pod in range(self.pods):
            # aggrSw -> coreSw
            for aggr in range(self.k // 2):
                for core in range(self.k // 2):
                    self.addLink('aggr{}{}'.format(pod + 1, aggr + 1), 
                               'core{}'.format(core + aggr * (self.k // 2) + 1))
                    # For coreSw is identified by its own ID
            
            # aggrSw -> edgeSw
            for aggr in range(self.k // 2):
                for edge in range(self.k // 2):
                    self.addLink('aggr{}{}'.format(pod + 1, aggr + 1), 
                               'edge{}{}'.format(pod + 1, edge + 1))
            
            # edgeSw -> host
            for edge in range(self.k // 2):
                for hst in range(self.k // 2):
                    self.addLink('edge{}{}'.format(pod + 1, edge + 1), 
                               'host{}{}{}'.format(pod + 1, edge + 1, hst + 1))



topos = { 'fattreetopo': ( lambda: FatTree() ) }
```

### 3.2 实验结果
![[Pasted image 20240413205811.png]]

the pingall is successful