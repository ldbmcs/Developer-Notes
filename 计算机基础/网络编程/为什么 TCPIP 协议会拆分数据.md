# 为什么 TCP/IP 协议会拆分数据

> 原文链接：[为什么 TCP/IP 协议会拆分数据](https://draveness.me/whys-the-design-tcp-segment-ip-packet/)

> 为什么这么设计（Why’s THE Design）是一系列关于计算机领域中程序设计决策的文章，我们在这个系列的每一篇文章中都会提出一个具体的问题并从不同的角度讨论这种设计的优缺点、对具体实现造成的影响。如果你有想要了解的问题，可以在文章下面留言。

TCP/IP 协议簇建立了互联网通信协议的概念模型，该协议簇的两个主要协议就是 TCP 和 IP 协议。这两个协议不仅能够保证数据会从源机器的源进程发送到目标机器的目标进程中，还能保证数据的不重不漏以及发送的顺序[1](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:1)。

![2021-02-21-LoMC57](https://image.ldbmcs.com/2021-02-21-LoMC57.jpg)

**图 1 - TCP/IP 协议簇**

当应用层协议使用 TCP/IP 协议传输数据时，TCP/IP 协议簇可能会将应用层发送的数据分成多个包依次发送，而数据的接收方收到的数据可能是分段的或者拼接的，所以它需要对接收的数据进行拆分或者重组。本文会分别从 IP 协议和 TCP 协议两个角度出发分析为什么应用层写入的数据包会被 TCP/IP 协议拆分发送：

- IP 协议会分片传输过大的数据包（Packet）避免物理设备的限制；
- TCP 协议会分段传输过大的数据段（Segment）保证传输的性能；

## 1. 最大传输单元

IP 协议是用于传输数据包的协议，作为网络层协议，它能提供数据的路由和寻址功能，让数据通过网络到达目的地[2](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:2)。**不同设备之间传输数据前，需要先确定一个 IP 数据包的大小上限，即最大传输单元**（Maximum transmission unit，即 MTU），**MTU 是 IP 数据包能够传输的数据上限**。

MTU 的值不是越大越好，更大的 MTU 意味着更低的额外开销，更小的 MTU 意味着更低的网络延迟。每一个物理设备都有自己的 MTU，两个主机之间的 MTU 依赖于底层的网络能力，它由整个链路上 MTU 最小的物理设备决定[3](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:3)，如下图所示，网络路径的 MTU 由 MTU 最小的红色物理设备决定，即 1000：

![2021-02-21-rZOqIr](https://image.ldbmcs.com/2021-02-21-rZOqIr.jpg)

**图 2 - 路径最大传输单元发现**

路径最大传输单元发现（Path MTU Discovery，PMTUD）是用来确定两个主机传输路径 MTU 的机制，它的工作原理如下[4](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:4)：

1. 向目的主机发送 IP 头中 DF 控制位为 1 的数据包，DF 是不分片（Don’t Fragment，DF）的缩写；
2. 路径上的网络设备根据数据包的大小和自己的 MTU 做出不同的决定：
   1. 如果数据包大于设备的 MTU，就会丢弃数据包并发回一个包含该设备 MTU 的 ICMP 消息；
   2. 如果数据包小于设备的 MTU，就会继续向目的主机传递数据包；
3. 源主机收到 ICMP 消息后，会不断使用新的 MTU 发送 IP 数据包，直到 IP 数据包达到目的主机；

> ICMP 是互联网控制消息协议（Internet Control Message Protocol，ICMP），它能在 IP 主机之间传递控制消息[5](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:5)。

以太网对数据帧的限制一般都是 1500 字节[6](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:6)，在一般情况下，IP 主机的路径 MTU 都是 1500，去掉 IP 首部的 20 字节，**如果待传输的数据大于 1480 节，那么该 IP 协议就会将数据包分片传输。**

IP 协议数据分片对传输层协议是透明的，假设我们使用 UDP 协议传输 2000 字节的数据，加上 UDP 8 字节的协议头[7](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:7)，IP 协议需要传输 2008 字节的数据。如下图所示，当 IP 协议发现待传输的数据大于 1480 字节，就会将数据分成下面的两个数据包：

![2021-02-21-akawpF](https://image.ldbmcs.com/2021-02-21-akawpF.jpg)

**图 3 - 分片传输的 UDP 数据**

1. 20 字节 IP 协议头 + 8 字节 UDP 协议头 + 1472 字节数据；
2. 20 字节 IP 协议头 + 528 字节数据；

**数据的接收方在收到数据包时会对分片的数据进行重组，不过因为第二个数据包中不包含 UDP 协议的相关信息，一旦发生丢包，整个 UDP 数据报就无法重新拼装。如果 UDP 数据报需要传输的数据过多，那么 IP 协议就会大量分片，增加了不稳定性。**

如果 IP 协议没有数据包大小的限制，那么上层可以以消息为单位传输数据，自然就不存在分片和组装的需求，不过因为物理设备的 MTU 限制，想要保证数据传输的可靠性和稳定性还需要传输层的配合。

## 2. 最大分段大小

TCP 协议是面向字节流的协议，应用层交给 TCP 协议的数据并不会以消息为单位向目的主机发送，**应用层交给 TCP 协议发送的数据可能会被拆分到多个数据段中。**

TCP 协议引入了**最大分段大小**（Maximum segment size，MSS）这一概念，它是 TCP 数据段能够携带的数据上限[8](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:8)。在正常情况下，TCP 连接的 MSS 是 MTU - 40 字节[9](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:9)，即 1460 字节；不过如果通信双方没有指定 MSS 的话，在默认情况下 MSS 的大小是 536 字节[10](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:10)。

**IP 协议的 MTU 是物理设备上的限制，它限制了路径能够发送数据包的上限，而 TCP 协议的 MSS 是操作系统内核层面的限制，通信双方会在三次握手[11](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:11)时确定这次连接的 MSS。**一旦确定了 MSS，TCP 协议就会对应用层交给 TCP 协议发送的数据进行拆分，构成多个数据段。

需要注意的是，IP 协议和 TCP 协议虽然都会对数据进行拆分，但是 IP 协议以数据包（Package）为单位组织数据，而 TCP 协议以数据段（Segment）为单位组织数据。

如下图所示，如果 TCP 连接的 MSS 是 1460 字节，应用层想要通过 TCP 协议传输 2000 字节的数据，那么 TCP 协议会根据 MSS 将 2000 字节的数据拆分到两个数据段中：

![2021-02-21-GorDZH](https://image.ldbmcs.com/2021-02-21-GorDZH.jpg)

**图 4 - 分段传输的 TCP 数据**

- 20 字节 IP 头 + 20 字节 TCP 头 + 1460 字节数据；
- 20 字节 IP 头 + 20 字节 TCP 头 + 540 字节数据；

从应用层的角度来看，两个数据段中 2000 字节的数据构成了发送方想要发送的消息，但是 TCP 协议是面向字节流的，向协议写入的数据会以流的形式传递到对端。

TCP 协议为了保证可靠性，会通过 IP 协议的 MTU 计算出 MSS 并根据 MSS 分段避免 IP 协议对数据包进行分片。因为 IP 协议对数据包的分片对上层是透明的，如果协议不根据 MTU 做一些限制，那么 IP 协议的分片会导致部分数据包失去传输层协议头，**一旦数据包发生丢失就只能丢弃全部数据。**

我们可以通过一个例子分析 MSS 存在的必要性。如下图所示，假设 TCP 协议中不存在 MSS 的概念，因为每个数据段的大小没有上限，当 TCP 协议交给 IP 层发送两个 1600 字节（包括 IP 和 TCP 协议头）的数据包时，由于物理设备的限制，IP 协议的路径 MTU 为 1500 字节，所以 IP 协议会对数据包分片：

![2021-02-21-zDBRax](https://image.ldbmcs.com/2021-02-21-zDBRax.jpg)

**图 4 - 分片传输的 TCP 数据**

四个数据包中只有两个会包含 TCP 协议头，即控制位、序列号等信息，剩下的两个数据包中不包含任何信息。当 IP 协议传输数据丢包时，TCP 协议的接收方没有办法对数据包进行重组，所以整个 TCP 数据段都需要重传，带来了更多额外的重传和重组开销。

## 3. 总结

数据拆分的根本原因说到底还是物理设备的限制，不过每一层协议都受限于下一层协议做出的决定，并依赖下层协议重新决定设计和实现的方法。虽然 TCP/IP 协议在传输数据时都需要对数据进行拆分，但是它们做出拆分数据的设计基于不同的上下文，也有着不同的目的，我们在这里总结一下两个网络协议做出类似决定的原因：

- IP 协议拆分数据是因为物理设备的限制，一次能够传输的数据由路径上 MTU 最小的设备决定，一旦 IP 协议传输的数据包超过 MTU 的限制就会发生丢包，所以我们需要通过路径 MTU 发现获取传输路径上的 MTU 限制；
- **TCP 协议拆分数据是为了保证传输的可靠性和顺序**，作为可靠的传输协议，为了保证数据的传输顺序，它需要为每一个数据段增加包含序列号的 TCP 协议头，如果数据段大小超过了 IP 协议的 MTU 限制， 就会带来更多额外的重传和重组开销，影响性能。

通过本文的分析，相信各位读者不仅了解了为什么 TCP/IP 协议会拆分数据，也了解了为什么 UDP 协议的数据报不应该超过 MTU - 28 字节，一旦超过该限制，IP 协议的分片机制会增加 UDP 数据报无法重组的可能性[12](https://draveness.me/whys-the-design-tcp-segment-ip-packet/#fn:12)。到最后，我们还是来看一些比较开放的相关问题，有兴趣的读者可以仔细思考一下下面的问题：

- IP 协议的分片机制都会导致哪些问题？
- TCP 连接双方是如何确定连接的 MSS？这个值是动态的么？

> 如果对文章中的内容有疑问或者想要了解更多软件工程上一些设计决策背后的原因，可以在博客下面留言，作者会及时回复本文相关的疑问并选择其中合适的主题作为后续的内容。

## 4. 延伸阅读

- [为什么 TCP 建立连接需要三次握手](https://draveness.me/whys-the-design-tcp-three-way-handshake)
- [为什么 TCP 协议有性能问题](https://draveness.me/whys-the-design-tcp-performance)
- [为什么 UDP 头只有 8 个字节](https://draveness.me/whys-the-design-udp-minimum-header)

## 5. 参考

1. Wikipedia: Internet protocol suite https://en.wikipedia.org/wiki/Internet_protocol_suite
2. What is the Internet Protocol? https://www.cloudflare.com/learning/ddos/glossary/internet-protocol/
3. Maximum transmission unit https://en.wikipedia.org/wiki/Maximum_transmission_unit
4. Mogul, J. and S. Deering, “Path MTU discovery”, RFC 1191, DOI 10.17487/RFC1191, November 1990, https://www.rfc-editor.org/info/rfc1191.
5. Wikipedia: Internet Control Message Protocol https://en.wikipedia.org/wiki/Internet_Control_Message_Protocol
6. Postel, J. and J. Reynolds, “Standard for the transmission of IP datagrams over IEEE 802 networks”, STD 43, RFC 1042, DOI 10.17487/RFC1042, February 1988, https://www.rfc-editor.org/info/rfc1042.
7. 为什么 UDP 头只有 8 个字节 https://draveness.me/whys-the-design-udp-minimum-header
8. Wikipedia: Maximum segment size https://en.wikipedia.org/wiki/Maximum_segment_size
9. Borman, D., “TCP Options and Maximum Segment Size (MSS)”, RFC 6691, DOI 10.17487/RFC6691, July 2012, https://www.rfc-editor.org/info/rfc6691. 
10. Braden, R., Ed., “Requirements for Internet Hosts - Communication Layers”, STD 3, RFC 1122, DOI 10.17487/RFC1122, October 1989, https://www.rfc-editor.org/info/rfc1122. 
11. 为什么 TCP 建立连接需要三次握手 https://draveness.me/whys-the-design-tcp-three-way-handshake
12. How is the MTU is 65535 in UDP but ethernet does not allow frame size more than 1500 bytes https://serverfault.com/questions/246508/how-is-the-mtu-is-65535-in-udp-but-ethernet-does-not-allow-frame-size-more-than