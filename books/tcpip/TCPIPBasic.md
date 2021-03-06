# TCP/IP 基础

TCP(Transmission Control Protocol)和 IP(Internet Protocol)是互联网的众多通信协议中最为著名的.

## TCP/IP 出现的背景及其历史

### 起源

20 世纪 60 年代, 许多大学和研究机构开始着力于新的通信技术. 其中有一家以美国国防部(DoD, The Department)为中心的组织也展开了类似的研究. DoD 希望在通信传输的过程中, 即使遭到了敌方的攻击和破坏, 也可以进过迂回线路实现最终通信, 保证通信不中断. 为了实现这种类型的网络, 分组交换技术(参考第一节, 数据头部携带发送端和接收端地址, 一条线路可以多方同时使用)便应运而生.

![show](https://image.cjyong.com/blog/tcpip/17.png)

![show](https://image.cjyong.com/blog/tcpip/18.png)

### ARPANET 的诞生

1969 年, 为验证分组交换技术的实用性, 研究人员搭建一套网络. 起初, 该网络只连接了美国西海岸的大学和研究所等 4 个节点. 后来, 随着美国国防部的重点开发和相关技术的飞速发展, 普通用户也逐渐加入其中, 发展成了后来巨大规模的网络. 该网络被人们称作 ARPANET, 也是全球互联网的鼻祖. 这也证明了基于分组交换技术通信方法是可信性.

### TCP/IP 的诞生

ARPANET 的实验, 不仅仅是利用几所大学与研究机构组成的主干网络进行分组交换的实验, 还会进行互连计算机之间提供可靠传输的综合性通信协议的实验. 20 世纪 70 年代前半叶, ARPANET 中的一个研究机构研发出了 TCP/IP, 在这之后, 直到 1982 年, TCP/IP 的具体规范才被定下来, 并于 1983 年成为 ARPANET 网络唯一指定的协议.

![show](https://image.cjyong.com/blog/tcpip/19.png)

### 商用互联网的启蒙

研究互联网最初的目的是用于实验和研究, 到了 1990 年逐渐被引入公司企业及一般家庭. 也出现了专门提供互联网接入服务的公司(ISP, Internet Service Provider), 同时基于互联网技术的新型应用, 比如在线游戏, SNS, 视频通信等商用服务也如雨后春笋般不断涌现出来.

## TCP/IP 的标准化

20 世纪 90 年代, ISO 开展了 OSI 这一国际标准协议的标准化进程. 然而, OSI 协议并没有得到普及, 真正被广泛使用的是 TCP/IP 协议.

### TCP/IP 的具体含义

TCP/IP 并不是指 TCP 和 IP 两种协议, 实际上 TCP/IP 协议是指 TCP/IP 协议群.

![show](https://image.cjyong.com/blog/tcpip/20.png)

### TCP/IP 标准化精髓

TCP/IP 的协议的标准化过程与其他的标准化过程有所不同, 具有两大特点: 开发性和注重实用性(即被标准化的协议能否被实际应用).

### TCP/IP 规范: RFC

TCP/IP 的协议由 IETF 讨论制定, 制定的结果被人们列入 RFC(Request For Comment)文档并在互联网上公布. RFC 文档不仅记录了协议规范内容, 还包括了协议的实现和运用的相关信息, 以及实验方面的信息.

RFC 文档通过编号组织每一个协议的标准化请求(后期换成 STD 方式进行管理编号), 一旦在 RFC 文档中进行了规定, 就无法进行修改, 如果必须进行修改或者废弃某一部分的内容, 就必须重新发布一个新的 RFC 文档, 同时旧的 RFC
文档就会作废, 并且该新的 RFC 文档需要声明拓展了那些 RFC 以及需要作废那些已有的 RFC 文档.

### TCP/IP 标准化流程

![show](https://image.cjyong.com/blog/tcpip/21.png)

### RFC 的获取方法

`http://www.rfc-editor.org/rfc/`

`ftp://ftp.rfc-editor.org/in-notes/`

## 互联网基础知识

互联网通信时, 需要相应的网络协议, TCP/IP 原本就是为了使用互联网而开发制定的协议族, 互联网的协议就是 TCP/IP. 互联网一词原本的含义就是网际网, 意指连接一个又一个网络. 互联网中的每个网络都是由骨干网(BackBone)和末端网(Stub)组成. 每个网络之间通过 NOC 相连. 如果网络的运营商不同, 它的网络连接方式和使用方法也会有差异. 连接这种异构网络需要 IX(Internet Exchange)的支持. 总之, 互联网就是众多异构的网络通过 IX 互连的一个巨型网络.

![show](https://image.cjyong.com/blog/tcpip/22.png)

## TCP/IP 协议分层模型

### TCP/IP 与 OSI 参考模型

![show](https://image.cjyong.com/blog/tcpip/23.png)

### 硬件(物理层)

TCP/IP 的最底层是负责数据传输的硬件. 这种硬件就相当于以太网或电话线路等物理层的设备.

### 网络接口层(数据链路层)

网络接口层利用以太网中的数据链路层进行通信, 因此属于接口层. 也就是, 把它当做 NIC 起作用的`驱动程序`. 驱动程序是在操作系统与硬件之间起桥梁作用的软件.

### 互联网层(网络层)

互联网层使用 IP 协议: 基于 IP 地址转发分包数据.

![show](https://image.cjyong.com/blog/tcpip/24.png)

TCP/IP 分层中的互联网层和传输层的功能通常由操作系统提供, 尤其是路由器, 它必须得实现互联网层转发分组数据包的功能.

- **_IP_**: IP 是跨越网络传送数据包, 使整个互联网都能收到数据的协议. IP 协议使数据可以发送到地球的另一端, 这区间它使用 IP 地址作为主机的标识.
- **_ICMP_**: IP 数据包在发送途中一旦发生异常导致无法到达对端目标地址, 需要给发送端发送一个异常通知. ICMP 就是为了这一功能而制定的. 有时也用来诊断网络的健康情况.
- **_ARP_**: 从分组数据包的 IP 地址中解析出物理地址(MAC 地址)的一种协议.

### 传输层

传输层最主要的功能就是可以让应用程序之间实现通信. 计算机内部, 通常同时运行多个程序. 为此必须分清哪些程序与哪些程序进行通信, 识别这些应用程序的就是端口号.

- **_TCP_**: TCP 是一种面向有连接的传输层协议, 它可以保证两端通信主机之间的通信可达. TCP 可以正确处理在传输过程中丢包, 传输顺序乱掉等异常情况, 另外 TCP 还能有效利用带宽, 缓解网络拥堵. 然而为了建立和断开连接, 有时它需要至少 7 次发包, 导致网络流量的浪费.
- **_UDP_**: UDP 是一种面向无连接的传输层协议. UPD 不会关注对端是否真的收到传送过去的数据, 适合分组数据较少或多播, 广播通信以及视频通信等多媒体领域.

### 应用层

![show](https://image.cjyong.com/blog/tcpip/25.png)

## TCP/IP 分层模型与通信示例

### 数据包首部

![show](https://image.cjyong.com/blog/tcpip/26.png)

### 发送数据包

假设甲给乙发送电子邮件, 内容为: "早上好". 而从 TCP/IP 通信上看, 是从一台计算机 A 向另一台计算机 B 发送电子邮件.

1. 应用程序处理: 新建邮件, 填好邮箱, 输入内容"早上好", 点击发送. 首先, 应用程序开始进行编码处理, 然后建立 TCP 连接, 利用 TCP 连接发送数据.
2. TCP 模块处理: TCP 根据应用的指示, 负责建立连接, 发送数据以及断开连接. TCP 提供将应用层发来的数据顺利发送到对端的可靠传输. 为了实现这一个功能, 需要在应用层数据的前端附加一个 TCP 首部. TCP 首部包含了源端口号和目标端口号, 序号以及校验和. 并将附加了 TCP 首部的包再发送给 IP.
3. IP 模块的处理: IP 将 TCP 传递过来的数据, 并添加上自己的 IP 首部. IP 首部包含接收端 IP 地址以及发送端 IP 地址. 紧随 IP 首部的还有用来判断 TCP 还是 UDP 的信息. IP 包生成后, 参考路由控制表决定接受此 IP 包的路由或主机. 随后, IP 包将被发送给连接这些路由器或主机网络接口的驱动程序, 以实现真正发送数据.(如果不知道接收端的 MAC 地址, 可以利用 ARP(Address Resolution Protocol)查找. 只要知道 MAC 地址, 就可以将 MAC 地址和 IP 地址交给以太网的驱动程序, 实现数据传输.
4. 网络接口(以太网)的处理: 从 IP 传送过来的 IP 包, 对于以太网驱动来说不过就是数据. 给这个数据附加上以太网首部并进行发送处理. 以太网首部包含接收端 MAC 地址, 发送端 MAC 地址以及标识以太网类型的以太网数据协议. 硬件生成 FCS(Frame Check Sequence, 防止数据包被噪音破坏)添加到包尾.

![show](https://image.cjyong.com/blog/tcpip/27.png)

![show](https://image.cjyong.com/blog/tcpip/28.png)

### 数据包接收处理

1. 网络接口(以太网驱动)处理: 从以太网包首部找到 MAC 地址判断是否发送给自己的包. 如果不是, 则丢弃. 如果是, 就查找以太网包首部中的类型域从而确定以太网协议所传送过来的数据类型. 这个例子就是 IP 包, 因此再将数据传给处理 IP 的子程序, 如果是不支持的协议类型(或不能识别)就丢弃数据.
2. IP 模块处理: 判断 IP 地址和自己的 IP 地址是否匹配, 如果匹配则接受数据并查找上一层的协议, 如果上一层协议是 TCP 就将 IP 包首部之后的部分传给 TCP 处理. 如果是 UDP 则将 IP 包首部后面的部分传给 UDP 处理.
3. TCP 模块处理: 计算校验和, 判断数据是否被破坏. 然后检查是否在按照序号接受数据. 最后检查端口号, 确定的应用程序. 接收完毕, 接收端发送一个"确认回执"给发送端. 如果这个回执没有发送到发送端, 发送端就会认为接收端没有接收到数据而一直反复发送.
4. 应用程序的处理: 解析数据就可以获知邮件的收件人地址是乙的地址. 如果主机 B 上没有乙的邮件信箱, 主机 B 就会发送一个"无此收件地址"的报错信息. 将邮件存放到本地之后, 发送一个"处理正常"的回执给发送端.
