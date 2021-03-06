# IP 协议

## IP 即网际协议

TCP/IP 的心脏是互联网层， 这一层主要是由 IP（Internet Protocol）和 ICMP（Internet Control Message Protocol）两个协议组成。

### IP 相当于 OSI 参考模型中的第三层

IP(IPv4, IPv6)相当于 OSI 参考模型中的第 3 层--网络层。 网络层的主要作用是`实现终端节点之间的通信`.

前面可以知道， 数据链路层的主要作用是在互连在同一种数据链路的节点之间进行包传递。 而一旦跨越多个数据链路， 就需要借助网络层。 网络层可以跨越不同的数据链路： 即实现在不同数据链路上实现两端节点之间的数据包传输。

如同旅行一样，我们要去一个很远的地方， 可能需要先做飞机，火车，在做公交车到达目的地。 但是如果我们去旅行社， 它就会预订好旅途过程中所有所需的票据， 甚至指定详细的行程表。 数据链路层就是类似飞机票在单一区间进行传输。 而旅行社保证将你送到最终的旅游景点，则是类似网络层。

![show](https://image.cjyong.com/blog/tcpip/55.png)

## IP 基础知识

IP 大致分为三大作用模块， 它们是 IP 寻址， 路由（最终节点为止的转发）以及 IP 分包与组包。

### IP 地址属于网络层地址

前一节我们知道， 在数据链路层中使用 MAC 地址来标识不同的计算机。 作为网络层的 IP， 也有这种地址信息。 一般叫做 IP 地址。

![show](https://image.cjyong.com/blog/tcpip/56.png)

无论那一台主机与那种数据链路连接， 其 IP 地址形式都保持不变。 网络层对数据链路层的某些特性进行了抽象。 数据链路的类型对 IP 地址形式透明。 另外在网桥或交换集线器等物理层或数据链路层数据包转发设备中， 不需要设置 IP 地址。 因为这些设备只负责将 IP 包转化为 0,1 比特流转发或对数据链路帧的数据部分进行转发， 而不需要 IP 协议。

### 路由控制

路由控制（Routing）是指将分组数据发送到最终目标地址的功能。 即时网络非常复杂， 也可以通过路由控制确定到达目标地址的通路。 一个数据包之所以可以成功到达最终的目标地址， 全靠路由控制。

Hop 中文称为`跳`。 是指网络中的一个区间。 IP 包正是在网络中一个个跳间被转发。 因此 IP 路由也叫做多跳路由。 每一个区间决定包在下一个跳被转发的路径。

![show](https://image.cjyong.com/blog/tcpip/57.png)

多跳路由是指路由器或主机在转发 IP 数据包时只指定下一个路由器或主机， 而不是将到最终目标地址为止的所有通路全部指定出来。 因为每一跳只指定下一跳的位置。 这里以火车旅游的具体说明：

![show](https://image.cjyong.com/blog/tcpip/58.png)

为了将数据包发给目标主机， 所有主机都维护着一个路由控制表（Routing Table）。 该表记录 IP 数据在下一步应该发给那个路由器。 IP 包根据这个路由表在各个数据链路上进行传输。

![show](https://image.cjyong.com/blog/tcpip/59.png)

### 数据链路的抽象化

IP 实现了在多个不同数据链路之间通信的协议。 数据链路根据种类的不同各有特点。 对这些不同数据链路的相异特性进行抽象化也是 IP 的重要作用。 如数据链路的地址（MAC）可以抽象为 IP 地址，因此对 IP 来说， 无论底层数据链路是使用以太网还是无线 LAN 或是 PPP， 都是一视同仁。

不同数据链路有个最大的区别就是它们各自的最大传输单位（MTU， Maximum Transmission Unit)不同。 如同很多运输公司运送包裹时所限定的包裹大小：

![show](https://image.cjyong.com/blog/tcpip/60.png)

以太网 MTU：1500 字节， FDDI 中是 4352 字节， ATM 中位 9180. 但是在 IP 协议中， 可能传输的数据远远大于这个值， 这就需要进行分包进行传输。

### IP 属于面向无连接型的

IP 面向无连接。 即在发包之前， 不需要建立与对端目标地址之间的连接。 主要原因是： 简化和提速。 面向连接比面向无连接处理相对复杂， 甚至管理都很繁琐。 如果需要面向有连接， 可以委托上一层提供这些服务。

IP 提供尽力服务（Best Effort), 意指`为了把数据包发送到最终目标地址， 尽最大努力`, 然而不保证最后是否收到。 如果需要保证有连接， 可以使用上一层协议 TCP 协议。

## IP 地址的基础知识

使用 TCP/IP 通信时， 使用 IP 地址识别主机和路由器。 为了保证正常通信， 有必要为每个设备配置正确的 IP 地址。 在互联网通信中， 全世界都必须设定正确的 IP 地址，否则无法进行正常的通信。

### IP 地址的定义

IP 地址（IPv4 地址）由 32 位正整数表示。 TCP/IP 要求将这样的 IP 地址分配给每一个参与通信的主机。 IP 地址在计算机内以二进制方式处理， 在外面使用的时候， 人们通常每 8 位分成一组， 共 4 组， 每组以“.“隔开， 再将每组数转化为 10 进制。

![show](https://image.cjyong.com/blog/tcpip/61.png)

2^32 = 4,294,967,296 最多可以运行 43 亿台计算机连接到网络。 实际上 IP 地址并非根据主机数量进行配置， 每一台主机上的每一个网卡都可以设置一个或者多个 IP 地址。 并且一台路由器上都会配置两个以上的网卡。

![show](https://image.cjyong.com/blog/tcpip/62.png)

### IP 地址由网络和主机两部分标识组成

IP 地址由“网络标识（网络地址）”和“主机标识（主机地址）”两部分组成。

![show](https://image.cjyong.com/blog/tcpip/63.png)

网络标识在数据链路的每个段都配置不同的值。 网络标识必须保证相互连接的每个段的地址不相重复。 相同段内相连的主机必须有相同的网络地址。 IP 地址的“网络标识”则不允许在同一个网段中重复出现。

因此通过设置网络地址和主机地址， 在相互连接的整个网络中保证每台主机的 IP 地址都不会相互重叠。 即： IP 地址具有唯一性。

![show](https://image.cjyong.com/blog/tcpip/64.png)

IP 包在被转发到某个路由器时， 正是利用目标的 IP 地址的网络标识进行路由。 因为即使不看主机标识， 只要一见到网络标识就可以判断是否为该网段内的主机。

### IP 地址的分类

IP 地址分为四个级别， 分别为 A 类， B 类， C 类， D 类。 根据 IP 地址中第 1 位到第 4 位比特列对其网络标识和主机标识进行区分。

![show](https://image.cjyong.com/blog/tcpip/65.png)

A 类：IP 地址是首位以 0 开头的地址，1 到 8 位是它的网络标识。 用十进制标识为： 0.0.0.0 - 127.0.0.0 是 A 类的网络地址。 A 类地址的后 24 位相当于主机标识。 因此， 一个网段内可容纳的主机地址上限为：16,777,214 个。

B 类： IP 地址以 10 开头的地址。 从第 1 位到 16 位都是网络标识。 十进制标识： 128.0.0.1 - 191.255.0.0 是 B 类的网络地址。 B 类地址的后 16 位相当于主机标识。 因此一个网段内可以容纳主机上限为 65,534 个。

C 类： IP 地址以 110 开头的地址。 从第 1 位到 24 位是它的网络标识。 十进制标识： 192.168.0.0 - 239.255.255.0 是 C 类的网络地址。 C 类地址的后 8 位相当于主机标识。 因此一个网段可容纳的主机地址上限为 254 个。

D 类： D 类 IP 地址以 1110 开头。 从第 1 位到 32 位都是网络标识。十进制标识： 244.0.0.0 - 239.255.255.255 是 D 类的网地址。 D 类地址没有主机标识， 常用于多播。

> 在分配 IP 地址的时候， 需要注意。 需要使用比特为表示主机地址时， 不可以全部为 0 或者全部为 1. 因为全部为 0 表示对应的网络地址或 IP 地址不可获知的情况下才使用。 全为 1 的主机地址通常作为广播地址。 所以分配过程需要去除这两种情况。 所以 C 类每个网段最大只能有 254（2^8 - 2）个主机地址的原因。 其它网段类似。

### 广播地址

广播地址用于在同一个链路中相互连接的主机之间发送数据包。 IP 地址中的主机部分全部设置为 1， 即为广播地址。 如 172.20.0.0 用二进制表示为：

10101100.00010100.00000000.00000000

将地址主机部分设为 1， 则形成广播地址：

10101100.00010100.11111111.11111111

十进制表示为：172.20.255.255

#### 两种广播

广播分为本地广播和直接广播。

本地广播： 本地网络内进行广播。 如在网络地址 192.168.0.0/24 的情况下， 广播地址是 192.168.0.255. 因为这个广播地址会被路由器屏蔽， 所以不会到达 192.168.0.0/24 以外的其他链路。
在不同的网路之间的广播叫做直接广播， 如网络地址为 192.168.0.0/24 的主机向 192.168.1.0/24 的目标地址发送 IP 包（192.168.1.255），从而使得 192.168.1.1 - 191.168.1.254 的主机都可以接收到这个包。

![show](https://image.cjyong.com/blog/tcpip/66.png)

### IP 多播

多播用于将包发送给特定组内的所有主机, 由于其直接使用 IP 协议,因此也不存在可靠传输. 对于向多台主机同时发送数据包, 在效率上的要求也日益提高. 在多播功能实现之前, 都是使用广播的方式. 那时的广播将数据发送给所有终端主机, 再由这些主机 IP 之上的一层去判断是否有必要接收数据, 否则丢弃. 然而这种方式会带来很多必要的流量, 会给那些毫无关系的网络或主机带来影响. 并且广播无法穿透路由, 如果想给其他网段发送同样的包, 就必须采用别的方法.

![show](https://image.cjyong.com/blog/tcpip/67.png)

多播使用 D 类地址(32 位都是网络标识, 不存在主机标识), 因此首 4 位是`1110`, 就可以认为是多播地址. 剩下的 28 位可以成为多播的组编号.

![show](https://image.cjyong.com/blog/tcpip/68.png)

从 224.0.0.0 到 239.255.255.255 都是多播地址的可用范围. 其中从 224.0.0.0 到 224.0.0.255 的范围不需要路由控制, 在同一个链路也能实现多播, 而在这个范围之外设计多播地址会给全网所有组内成员发送多播的包.

此外,对于多播, 所有的主机(路由器以外的主机和终端主机)必须属于 224.0.0.1 的组, 所有的路由器必须属于 224.0.0.2 的组. 除了地址外, 还需要 IGMP(Integrnet Group Management Protocol)等协议的支持.类似的组还有很多:

![show](https://image.cjyong.com/blog/tcpip/69.png)

### 子网掩码

#### 分类造成浪费

一个 IP 地址只要确定了分类, 也就确定了它的网络标识和主机标识. 如 A 类地址前 8 位, B 类地址的前 16 位, C 类地址的前 24 位分别是它们各自的网络标识部分. 如:
A 类: 255. 0. 0. 0
B 类: 255. 255. 0. 0
C 类: 255. 255. 255. 0

网络标识相同的计算机必须属于同一个链路层. 如, 架构 B 类 IP 网络时, 理论上运行一个链路内运行 6 万 5 千多台计算机连接. 然而, 在实际的网络架构中, 一般不会存在这么多计算机一起的情况. 直接使用 A 类和 B 类地址, 是非常浪费的. 随着互联网的覆盖范围的扩大, 对网络地址的需求也越来越大, 为了减少这种浪费, 人们开始一种新的组合来减少这种浪费.

#### 子网与子网掩码

现在,一个 IP 地址的网络标识和主机标识已不再受限于该地址的类别, 而是由一个叫做`子网掩码`的识别码通过子网网络地址细分比 A 类, B 类, C 类更小粒度的网络. 本质上就是将 A 类, B 类, C 类等分类地址中的主机地址部分用作子网地址, 可以将原网络分为多个物理网络的一种机制.

加入子网之后, 一个 IP 地址就有了两种识别码. 一是 IP 地址本身, 另一个是表示网络部的子网掩码. 子网掩码用二进制的方式表示的话, 也是一个 32 位的数字. 它对应 IP 地址网络标识部分的为全部为`1`, 对应 IP 地址主机标识的部分则为`0`. 由此, 一个 IP 地址可以不再受限于自己的类别, 而是可以使用这样的子网掩码自由地定位自己的网络标识长度. 当然, 子网掩码必须是 IP 地址的首位开始连续的`1`.

如: 以 172.20.100.52 这个 IP 地址, 如果你需要让前 26 位的地址作为网络地址(后 6 位地址作为主机标识), 单纯使用 A,B,C,D 类地址都是不符合的. 但是如果我们使用子网掩码(前 26 位为 1,后 6 位为 0),就可以完成了. 这时候的 IP 地址标识为:

IP 地址: 172. 20. 100. 52
子网掩码: 255. 255. 255. 192.

网络地址: 172. 20. 100. 0
子网掩码: 255. 255. 255. 192

广播地址: 172. 20. 100. 63
子网掩码: 255. 255. 255. 192

另一种标识方式为:

IP 地址: 172. 20. 100. 52 /26

网络地址: 172. 20. 100. 0 /26

广播地址: 172. 20. 100. 63 /26

![show](https://image.cjyong.com/blog/tcpip/70.png)

### CIDR 与 VLSM

随着互联网的发展, IP 地址的数量开始缺乏, 无法满足正常的需求. 于是人们放弃 IP 地址的分类, 采用任意长度分割 IP 地址的网络标识和主机标识. 这种方式叫做 CIDR(Classless Inter-Domain Routing). 由于 BGP(Border Gateway Protocol, 边界网关协议)对应了 CIDR, 所以不受 IP 地址分类的限制自由分配.

根据 CIDR, 连续多个 C 类地址就可以划分到一个较大的网络内. CIDR 可以更有效的利用当前的 IPv4 地址, 同时通过路由集中降低了路由器的负担.

如利用 CIDR 技术将 203.183.224.1 到 203.183.225.254 的地址合为同一个网络(本来是 2 个 C 类地址).

![show](https://image.cjyong.com/blog/tcpip/71.png)

在 CIDR 应用到互联网的初期, 网络内部采用固定长度的子网掩码机制. 也就是当子网掩码的长度固定为/25 以后, 域内所有的子网掩码都得使用同样的长度, 然而, 有些部门可能有 500 台主机, 另一些部门可能只有 50 台主机. 如果全部采用统一标准的话, 就很难架构一个高效的网络结构. 为此人们提出组织内使用可变长度的, 高效的 IP 地址分配方式: VLSM(Variable Length Subnet Mask,可变长子网掩码)
. 它可以通过域间路由协议转换为 RIP2 以及 OSPF 实现. 根据 VLSM 可以将网络地址在主机数为 500 个时, 将子网掩码长度为/23, 主机数为 50 时将子网掩码的长度为/26. 理论上可以将 IP 地址的利用率提高至 50\$.

### 全局地址与私有地址

随着互联网的普及, IP 地址不足的问题日趋显著. 如果按照先前的原则分配唯一地址的话, 会有 IP 地址耗尽的危险. 于是出现了一种新的技术, 它不要求为每一台主机或路由器分配一个固定的 IP 地址, 而是在必要的时候只为相应数量的设备分配唯一的 IP 地址. 尤其是对于那些没有连接互联网的设备, 只要保证在这个网络内地址唯一, 可以不用考虑互联网即可配置相应的 IP 地址. 不过, 即使让网络各自随意设置 IP 地址, 也可能出现问题. 不过, 即使每个独立的网络各自随意地设置 IP 地址, 也可能出现问题, 于是又出现私有网络的 IP 地址. 它的地址范围:

A 类: 10.0.0.0 - 10.255.255.255 10/8
B 类: 172.16.0.0 - 172.31.255.255 172.16/12
C 类: 192.168.0.0 - 192.168.255.255 192.168/16

在这范围内的地址,叫做私有 IP. 其他的 IP 地址称为全局 IP. 全局 IP 需要保证在整个互联网范围内保持一致, 而私有 IP 则不需要. 私有 IP 和全局 IP 的 NAT 技术诞生之后, 配有私有地址 IP 的主机联网时, 则通过 NAT 进行通信. 这也就成为了解决 IP 地址分配问题的主流方案.

![show](https://image.cjyong.com/blog/tcpip/72.png)

## 路由

发送数据包所使用的地址是网络层的地址, 即 IP 地址, 然而仅仅有 IP 地址还不足以实现将数据包发送到对端目标地址, 在发送数据过程中还需要类似`指明路由器或主机`的信息. 实现 IP 通信的主机和路由器都必须持有一张这样的表才能进行数据发送.

### IP 地址与路由控制

IP 地址中的网络地址部分用于进行路由控制.

![show](https://image.cjyong.com/blog/tcpip/73.png)

路由控制表中记录着网络地址与下一步应该发送到路由器地址. 在发送 IP 包时, 首先确定 IP 包首部中的目标地址, 再从路由控制表中找到与改地址具有相同网络地址的记录, 在根据记录将 IP 包转发给相应的下一个路由器. 如果路由控制表中存在多条相同网络地址的记录, 就选择一个最为吻合(相同位数最多的)的网络地址. 如在 172.20.100.52 的网络地址与 172.20/16 和 172.20.100/24 两项都匹配. 此时, 应该选择匹配度最大的 172.20.100/24.

#### 默认路由

如果一个路由表需要包含所有的网络及其子网的信息, 就会造成浪费. 这是使用默认路由(Default Route)就是一个不错的选择. 默认路由: 指路由表中任何一个地址都可以与之匹配, 一般标记为 0.0.0.0/0 或 default.

#### 主机路由

IP 地址/32 类型的地址称为主机路由(Host Route). 例如, 192.168.153.15/32 就是一种主机路由. 标明整个 IP 地址的所有位都参与路由. 进行主机路由, 意味着要基于主机上网卡配置的 IP 地址本身, 而不是基于网络地址部分进行路由.

#### 环回地址

环回地址是在同一个计算机上的程序之间进行网络通信时所使用的一个默认地址. 计算机使用一个特殊的 IP 地址 127.0.0.1 作为环回地址. 一般表示为 localhost, 使用之后数据包不会流向网络.

### 路由控制表的聚合

利用网络地址的 bite 分布可以有效进行分层配置. 对内使用子网掩码构建不同的网段, 对外呈现出同一个网络地址. 这样可以更好地构建网络, 通过路由信息的聚合可以有效地减少路由表的条目.

![show](https://image.cjyong.com/blog/tcpip/74.png)

## IP 分割处理与再构成处理

### 数据链路不同, MTU 存在差异

不同的数据链路的最大传输单元(MTU)不同. 而 IP 属于数据链路层上一层, 它必须不受限于不同链路的 MTU 大小, 需要对其进行封装处理, 抽象化底层的数据链路.

![show](https://image.cjyong.com/blog/tcpip/75.png)

### IP 报文的分片和重组

任何一台主机都有必要对 IP 分片(IP Fragmentation)进行相应的处理. 分片: 网络上遇到比较大的报文而无法一下子发送出去时就会进行处理.

如图: 以太网的默认 MTU 是 1500 字节, 因此 4342 字节的 IP 数据无法在一个帧中发送完成, 这时路由器将此 IP 数据包划分为 3 个分片进行发送.

![show](https://image.cjyong.com/blog/tcpip/76.png)

进过分片之后的 IP 数据报重组时只能在目标主机进行, 路由器只能进行分片, 但不会进行重组.

### 路径 MTU 发现

分片机制也有自己的不足, 首先增加了路由器的处理负荷. 随着时代的变迁, 计算机网络的物理传输速度不断上升. 这些高速的链路, 对计算机网络提出了更高的要求. 另外, 人们对网络安全的要求也提高了, 路由器需要做的处理也越来越多, 如网络过滤, 只要允许, 是不希望路由器进行 IP 数据包的分片处理的.

为了解决这种问题, 就产生了一个新的技术`路径MTU发现(Path MTU Discovery)`. 所谓的路径 MTU 是指从发送端到接收端主机之间不需要分片时最大的 MTU 的大小. 即路径中存在的所有数据链路中最小的 MTU. 而路径 MTU 发现从发送主机按照路径 MTU 的大小对数据进行分片后进行发送. 进行路径 MTU 发现, 可以避免在中途的路由器上进行分片处理, 也可以在 TCP 中发送更大的包.

![show](https://image.cjyong.com/blog/tcpip/77.png)

路径 MTU(Path MTU)发现的工作原理如下:
首先在发送端主机发送 IP 数据报时将其首部的分片禁止标志位设为 1. 途中路由器即时遇到了需要分片才能处理的大包, 也不会进行分片, 而是直接丢弃. 随后通过一个 ICMP 的不可达消息将数据链路层上的 MTU 值发送给主机.

下一次,从发送给同一个目标主机的 IP 数据报获得 ICMP 所通知的 MTU 值以后, 将它设置为当前的 MTU. 发送主机根据这个 MTU 对数据进行分片处理. 如此反复, 直到数据包被正确送到目标主机为止没有收到任何 ICMP, 认为最后一次 ICMP 所通知的 MTU 值就是最终合适的 MTU 值. 这个值最多缓存 10 分钟, 到时候就会重新进行获取.

![show](https://image.cjyong.com/blog/tcpip/78.png)

## IPv6

### IPv6 的必要性

IPv6(IP version 6)是为了根本解决 IPv4 地址耗尽的问题而被标准化的网际协议. IPv4 的地址长度为 4 个 8 位字节, 即 32bit. 而 IPv6 的地址长度则是原来的 4 倍, 即 128 比特, 一般写成 8 个 16 位字节.

从 IPv4 切换到 IPv6 极其耗时, 需要将网络中所有的主机和路由器的 IP 地址进行重新设置. 当互联网普及之后, 替换更是非常艰巨的.

IPv6 不仅可以解决 IPv4 地址耗尽的问题, 甚至是弥补 IPv4 中绝大多数的缺陷. 人们也正着力于进行 IPv4 和 IPv6 之间相互通信和兼容性方面的测试.

### IPv6 的特点

IPv6 具备以下几个特点:

- IP 地址的扩大与路由控制表的聚合: IP 地址依然使用互联网分层构造. 分配与其地址结构相适应的 IP 地址, 尽可能避免路由表的膨大.
- 性能提升: 包首部长度采用固定值(40 字节), 不再采用首部校验码. 简化首部结构, 减轻路由器负荷. 路由器不再做分片处理.(通过路径 MTU 只由发送端主机进行分片处理).
- 支持即插即用功能: 即使没有 DHCP 服务器也可以实现自动分配 IP 地址.
- 采用认证与加密功能: 应对伪造 IP 地址的网络安全功能以及防止线路窃听功能 IPsec.
- 多播, Mobile IP 成为拓展功能: 多播和 Mobile IP 成为了 IPv6 中的拓展功能.

### IPv6 中 IP 地址的标记方法

IPv6 地址长度为 128 位, 所能代表的数字高达 38 位数. 足以为人们可以想象到的所有主机的路由器分配地址. 如果将 IPv6 地址像 IPv4 的地址一样用十进制数据表示的话, 是 16 个数字的序列.由于使用 16 个数字序列表示显得有些麻烦, 因此 IPv6 的表示方法有些诧异. 一般人们将 128 比特 IP 地址以每 16 比特为一组, 每组使用:进行分割. 如果连续出现 0 可以使用::表示. 但是一个 IP 地址只能出现一次两个连续的冒号.

![show](https://image.cjyong.com/blog/tcpip/79.png)

### IPv6 地址的结构

IPv6 类似 IPv4 通过 IP 地址的前几位标识 IP 地址的种类. 在互联网通信中, 使用一种全局的单播地址. 是互联网中唯一的一个地址, 不需要正式分配 IP 地址. 在私有网络中, 可以使用唯一本地地址. 该地址根据一定算法生成随机数并融入到地址中. 可以像 IPv4 的私有地址一样自由使用.

![show](https://image.cjyong.com/blog/tcpip/80.png)

### 全局单播地址

全局单播地址是指世界上唯一的一个地址.

![show](https://image.cjyong.com/blog/tcpip/81.png)

n = 48, m = 16, 128 - n - m = 64. 即前 64 位比特为网络标识, 后 64 位为主机标识. 通常接口 ID 中保存 64 比特的 MAC 地址的值, 如果不想暴露, 可以设置为一个`临时地址`.

### 链路本地单播地址

在同一个链路内的唯一地址, 不经过路由器, 接口 ID 存放 64 比特的 MAC 地址.

![show](https://image.cjyong.com/blog/tcpip/82.png)

### 唯一本地地址

不进行互联网通信时所使用的地址. 内部网络与外部网络进行通信时一般会通过 NAT 或网关进行.

![show](https://image.cjyong.com/blog/tcpip/83.png)

### IPv6 分段处理

只在起点为发送端主机上进行, 路由器不参与分片, 减少路由负荷, 提高网速.

## IPv4 首部

通过 IP 进行通信时, 需要在数据的前面加入 IP 首部信息. IP 首部中包含用于 IP 协议进行发包控制时所有的必要信息. 了解 IP 首部的结构, 就可以对 IP 所提供的的功能有一个详解把握.

![show](https://image.cjyong.com/blog/tcpip/84.png)

### 版本(Version)

4bit 构成, 表示标识 IP 首部的版本号. IPv4 的版本号即为 4, 因此这个字段上的值也是`4`.

![show](https://image.cjyong.com/blog/tcpip/85.png)

### 首部长度(IHL: Internet Header Length)

4bit 构成, 表明首部的大小, 单位为 4 字节. 对于没有可选项的 IP 包, 首部长度设置为`5`. 也就是说, 当没有可选项时, IP 首部的长度为 20 字节.

### 区分服务(TOS: Type of Service)

![show](https://image.cjyong.com/blog/tcpip/86.png)

由于实现起来较为复杂且没有多大作用, 几乎没有使用这个字段.

### DSCP 段和 ECN 段

DSCP(Differential Services Codepoint, 差分服务代码点)(5bit)是 TOS 的一部分. 现在统称 DiffServ, 用来进行质量控制.如果 3-5 位的值为 0, 0-2 为则被称为类别选择代码点. 这样可以像 TOS 的优先度那样提供 8 中类型的质量控级别. 对于不同级别采用的措施由 DiffServ 的运营管理者指定.

ECN(Explicit Congestion Notification, 显式拥塞通告)(2bit)用来报告网络拥堵情况: ECT(ECT-Capable Transport)和 CE(Congenstion Experienced)当路由器在转发 ECN 为 1 的包的过程中, 如果出现了网络拥堵情况, 就将 CE 的值设为 1.

### 总长度(Total Length)

IP 首部和数据部分结合起来的总字节数. 该子段长 16bit. 所以 IP 包的最大长度为: 2^16: 65535 字节.

### 标识(ID: Identification)

由 16 比特构成, 用于分片重组.同一个分片的标识值相同, 不同分片则是不同. 此外还需要根据目标地址, 源地址和协议进行确定是否同一个分片.

### 标志(Flags)

3bit 数据:

![show](https://image.cjyong.com/blog/tcpip/87.png)

### 片偏移(FO: Fragment Offset)

由 13 比特构成, 用来标识分片中每一个分段相对原始数据的位置. 第一个分片对应的值为 0. FO 占据 13 位, 最多可以标识 2^13=8192 个相对位置. 单位为 8 字节, 因此最大可表示原始数据 8 x 8192 = 65536 字节的位置.

### 生存时间(TTL: Time to Live)

8 比特构成, 最初是记录该包在网络上的生存时间. 现在指可以中转多少个路由器, 没经过一个路由器将该值减一, 到 0 则丢弃该包.

### 协议(Protocol)

8 比特构成, 表示 IP 首部的下一个首部属于哪个协议. 部分协议编号如图:

![show](https://image.cjyong.com/blog/tcpip/88.png)

### 首部校验和(Header Checksum)

16 比特构成, 只校验数据包的首部, 不校验数据部分. 用来确保 IP 数据包不被破坏. 计算过程: 将该校验和清零, 然后以 16 比特为单位划分 IP 首部, 并用 1 补数计算所有的 16 位字的和. 最后将这个和的 1 补数赋值个首部校验和字段.

如:
IP 头：
45 00 00 31
89 F5 00 00
6E 06 00 00（校验字段）
DE B7 45 5D -> 222.183.69.93
C0 A8 00 DC -> 192.168.0.220

        计算：
        4500 + 0031 +89F5 + 0000 + 6e06+ 0000 + DEB7 + 455D + C0A8 + 00DC =3 22C4
        0003 + 22C4 = 22C7
        ~22C7 = DD38      ->即为应填充的校验和

当接受到 IP 数据包时，要检查 IP 头是否正确，则对 IP 头进行检验，方法同上：

计算：
4500 + 0031 +89F5 + 0000 + 6E06+ DD38 + DEB7 + 455D + C0A8 + 00DC =3 FFFC
0003 + FFFC = FFFF
得到的结果是全一，正确。

### 源地址(Source Address)

32 比特构成, 表示发送端 IP 地址.

### 目标地址(Destination Address)

32 比特构成, 表示接收端 IP 地址.

### 可选项(Options)

长度可变, 可以自行添加, 如: 安全级别, 源路径, 路径记录, 时间戳.

### 填充(Padding)

在有可选项的时候, 首部的长度可能不是 32 比特的整数倍, 因此可以向字段填充 0, 调整为 32 比特的整数倍.

### 数据(Data)

存入数据.

## IPv6 首部格式

IPv6 的 IP 数据首部格式相比 IPv4 发生了巨大的变化, 为了减轻路由器的负担, 省略了首部校验和字段. 路由器不再需要计算校验和, 从而提高了包的转发效率. 另外分片处理的识别码成为了可选项. 为了让 64 位 CPU 的计算机处理起来更加方便, IPv6 的首部及可选项都是 8 个字节构成.

![show](https://image.cjyong.com/blog/tcpip/89.png)

### 版本号控制(Version)

与 IPv4 一样, 由 4 比特构成, IPv6 的版本号为 6.

### 通信量类(Traffic Class)

相当于 IPv4 中的 TOS 字段, 暂时保留.

### 流标号(Flow Label)

由 20 比特构成, 准备用于服务质量(QoS: Quality Of Service)控制. 未来研究的课题之一.

### 有效载荷长度(Payload Length)

指包的数据部分, IPv4 的 TL(Total Length)是指包括首部在内的所有长度. 这里的值不包括首部, 只表示数据部分的长度.

### 下一个首部(Next Header)

相当于 IPv4 中的协议字段, 8bit 构成, 通常表示 IP 的上一层协议是 TCP 或 UDP.

### 跳数限制(Hop Limit)

8bit, 类似 IPv4 中的 TTL.

### 源地址组成(Source Address)

128 比特构成, 发送端 IP 地址.

### 目标地址(Destination Address(

128 比特构成, 接收端 IP 地址.

### 拓展首部

IPv6 的首部长度固定, 无法将可选项加入其中, 而是通过拓展首部进行了有效拓展.IPv4 中的可选项长度固定为 40 字节, 而 IPv6 没有这种限制.

![show](https://image.cjyong.com/blog/tcpip/90.png)

![show](https://image.cjyong.com/blog/tcpip/91.png)
