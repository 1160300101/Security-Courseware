# TCP/IP安全2——IPv4协议安全

### 哈尔滨工业大学 网络与信息安全 张宇 2018

---

背景：IPv4是互联网的核心协议，对其进行安全评估是必要的，但一直以来都缺乏一份“官方”文档。[RFC6274: Security Assessment of the Internet Protocol Version 4, 2011](https://www.ietf.org/rfc/rfc6274.txt)，是针对IPv4安全性评估的第一份全面的IETF文档。


## 1. IP头部


```
      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |Version|  IHL  |Type of Service|          Total Length         |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |         Identification        |Flags|      Fragment Offset    |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |  Time to Live |    Protocol   |         Header Checksum       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Source Address                          |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                    Destination Address                        |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                  [ Options ]                  |  [ Padding ]  |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

                 Figure 1: Internet Protocol Header Format
```

### 1.1 前32比特

- 最小IP头部大小20字节，检查`LinkLayer.PayloadSize >= 20`.
- Version：版本号为4；在以太网帧中，也会用`EtherType=0x0800`来表示IPv4数据报；若只根据以太类型来判断，而不检查版本号，则攻击者可能绕过NIDS中的模式匹配，例如用其他链路层“Protocol Type”来封装IPv4包。
- IHL（头部长度）：32比特（4字节）的整数倍；应不小于最小头部大小20字节，即大于等于5；同时，应小于整个包实际长度/4。
- TOS（服务质量）：实际网络中并没有大范围使用；
	- 在区分服务（Differentiated Service）中，攻击者可能伪造DSCP字段为`11x000`来获得更高的转发优先级，进而导致对其他低优先级流(`000000`)的DoS攻击；
	- 在显式拥塞通知（ECN）中，一台主机通过标记数据包中特定位置来通知路由器该主机具有ECN能力，路由器在发生轻微拥塞时，会采用RED机制来管理队列，而不会丢弃这类数据包，这导致该数据流有更高优先级；
- 总长度：16比特表示最大65535字节，但RFC791中规定所有主机应最大支持576字节，而目前多数实现支持至少9K字节。如果直接根据总长度字段来读取报文，可能出错，应该根据链路负载大小来检查总长度字段。
	- 许多运行带有Cisco Express Forwarding的Cisco IOS系统的设备，在处理总长度超过实际包大小时，会从缓冲中读取之前包的内容，可能对将之前包的分片包含进当前包中，导致信息泄漏。

### 1.2 ID域

ID：许多系统以协议无关、递增的方式来使用ID，这导致很多安全问题。例如，ID很容易重复，以及[Idle Scan](https://nmap.org/book/idlescan.html)：不使用自己的IP地址进行扫描来发现开放/（关闭/过滤）端口。

```
SCAN AN OPEN PORT:
Attacker                             Zombile                    Target
    |-------- (1) SYN/ACK ---------->   |                          |
    <------- (2) RST; ID=31337 ---------|                          |
    |-------------------- (3) SYN "from" Zombie ------------------>|
                                        <-- (4) SYN/ACK -----------|
                                        |--- (5) RST; ID=31338 --->|
    |-------- (6) SYN/ACK ---------->   |
    <------- (7) RST; ID=31339 ---------|
```

```
SCAN A CLOSED PORT:
Attacker                             Zombile                    Target
    |-------- (1) SYN/ACK ---------->   |                          |
    <------- (2) RST; ID=31337 ---------|                          |
    |-------------------- (3) SYN "from" Zombie ------------------>|
                                        <------ (4) RST -----------|
                                        |------ No response  ----->|
    |-------- (6) SYN/ACK ---------->   |
    <------- (7) RST; ID=31338 ---------|
```

```
SCAN A FILTERED PORT:
Attacker                             Zombile                    Target
    |-------- (1) SYN/ACK ---------->   |                          |
    <------- (2) RST; ID=31337 ---------|                          |
    |-------------------- (3) SYN "from" Zombie ------------------>|
                                        <------ No response -------|
    |-------- (6) SYN/ACK ---------->   |
    <------- (7) RST; ID=31338 ---------|
```

一些操作系统（例如Linux）将不分片的（DF位=1）数据报中的ID置零来避免信息泄漏，但一些中间盒设备会将DF位=1的包也分片，所以可能会导致ID碰撞。Linux（以及Solaris）后来按IP地址来设置ID，即不同的IP地址用不同的ID计数器；但仍存在一些问题，例如可以通过一个负载均衡器的ID来计算其后系统的数量。

由于ID只用来组装IP分片，只要在「原地址，目的地址，协议」下唯一就可以。因此，可以采用一个伪随机数生成器来产生ID。

### 1.2 Flags域与Fragment Offest域

Flags：

- Bit 0: 保留
- Bit 1: DF = 1， Don't Fragment
- Bit 2: MF = 1， More Fragments；0 Last Fragment

通常，DF被置1来实现Path-MTU Discovery（PMTUD）：当一个包“过大”，又不能被分片时，会被设备丢弃，并产生一个ICMP “fragmentation needed and DF bit set” error。这可能被利用来“绕过”IDS，使得IDS误以为该数据报到达了目标。

Fragment Offset：以8字节为单位，表示分片在原数据报中偏移量。该字段大小应在合理范围内，其他安全问题稍后会在IP分片与重组方面讨论。

### 1.3 TTL

TTL可以被用于实现以下功能：

- 估计主机距离：初始TTL通常设置成2的幂或255；
- 识别操作系统、物理设备：不同操作系统设置不同的初始值，同一中间盒设备之后的不同设备有不同的初始TTL；
- 绘制网络拓扑、定位主机：traceroute；多个监测点到目标的距离形成对目标的定位
- 绕过IDS：见IDS部分
- 提高应用安全、限制数据包传播：例如，通过设置TTL=255并检查，在BGP中限制直接邻居间会话

### 1.4 协议、头部校验和、源地址、目的地址

- 协议域没有特别的安全问题
- 头部校验和：一些设备可能不检查校验和，导致可以被利用来“绕过”IDS
- 源地址伪造，“一切罪恶的根源”
- 更多问题在后面的“寻址”部分讨论

### 1.5 IP选项

- 多字节选项，包含类型（1字节），长度（1字节），选项数据
	- 类型：1比特copied flag，2比特option class，5比特option number
- 单字节选项，只有类型字节：<0， 0>，“End of Option list”; <0, 1>, "No Operation"

- LSRR (Type=131) 松散源路由与记录路由
 	- 源地址始终为最初发包者，目的地址在每一跳都更新为LSRR选项中下一地址
	- 最终接收者用LSRR选项以源路径的逆序来应答
	- 绕过防火墙规则、到达本来不可达目标、隐秘地建立TCP连接：目的地址并不是最终目的地址
	- 拓扑发现：记录路由
	- 带宽耗尽攻击：令一个包在几个设备间来回传递
	- 应被禁止

一种利用LSRR来绕过防火墙的攻击方法：

- 攻击者A冒充节点V（源地址欺骗），令目的IP地址为T的数据包经过A
- T上的防火墙或应用以为是V来访问，T以A为中间节点将应答包发送给V

```
                    src  dst  LSRR             
                     |    |     | 
                from V to T via A
 Attacker (A)   —————————————————>   Target (T)  
       ^                                 |
       |        from T to V via A        |
       +—————————————————————————————————+ 
```


- SSRR（Type=137）严格源路由与记录路由
	- 与LSRR类似的安全问题
- Record Route（Type=7）
	- 可用于发现拓扑
- Timestamp（Type=68），记录处理数据包的时间戳，也可同时记录路由
	- 攻击者可获得目标的当前时间
	- 记录路由
	- 识别操作系统和设备（分析时钟漂移）
	- 应被禁止


## 2. IP机制

### 2.1 分片重组（Fragment Reassembly）

分片重组相关问题基本都源自其功能的复杂性：

- 端主机需要进行有状态操作，为分片分配缓冲；同时也，无法猜测分片需要多久到来
	- 重组超时时间在60秒到120秒之间
	- 攻击者可能估计伪造无法重组的分片，导致资源被故意耗尽
- 当前可用带宽比以往更高，导致一些互操作问题，
	- 高带宽导致ID更容易重复，而不同包的分片发生碰撞，例如在1Gbps带宽下，1kb的UDP包，567字节分片，在0.65秒内会导致ID值重复
- 重组过程复杂，分片可能乱序，重复，重叠，等等，导致实现存在Bug
	- 导致缓冲区溢出，可能会令系统崩溃
- 分片重组算法不清晰，导致多种实现
	- 发生分片重叠时，不同实现可能采用第一个分片，最后一个分片，或其它某个分片
	- 不同实现采用不同超时时间
	- 当缓冲区满时，不同实现下会得到不同重组结果
	- 上述实现的差异，会导致IDS重组结果和主机重组结果存在差异，从而被利用来绕过IDS
- 对策：
	- 专用缓冲区，(有选择地)清除缓冲区，减少分片超时时间
	- “Active Mapping”技术：令IDS有更多关于接收端的知识，避免被绕过

### 2.2 转发（Forwarding）与寻址（Addressing）

转发：

- 优先队列服务（Precedence-Ordered Queue Service）被攻击者利用来获得更好的服务
- 带有非缺省TOS的Weak TOS的包会被按照该TOS对应的路由来转发；而通常的转发方式是“最长匹配”
- 地址解析对缓冲管理的影响：向不存在主机发送大量数据包，导致接入路由器需要做ARP获得不存在主机的MAC地址，而同时要存储待转发的数据包；对策之一是“negative cache”
- 丢包：当丢弃一个分片时，也应该丢弃该数据包中其它分片，否则会导致DoS攻击风险

寻址：

- 应过滤不可达地址，例如在公开互联网上的私有地址、回环地址、组播地址（224/4）、实验地址（240/4），广播（-1）和网络地址（0）等
	- 特殊地址：{<子网号>，<主机号>}，`{0, 0}`（本网本机），`{0, 主机号}`，`{-1, -1}` (本地广播地址)

---

