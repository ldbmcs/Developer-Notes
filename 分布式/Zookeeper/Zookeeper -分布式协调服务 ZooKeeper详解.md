# 详解分布式协调服务 ZooKeeper

> 原文链接：[详解分布式协调服务 ZooKeeper](https://draveness.me/zookeeper-chubby/)

在 2006 年，Google 发表了一篇名为 [The Chubby lock service for loosely-coupled distributed systems](https://static.googleusercontent.com/media/research.google.com/en//archive/chubby-osdi06.pdf) 的论文，其中描述了一个分布式锁服务 Chubby 的设计理念和实现原理；作为 Google 内部的一个基础服务，虽然 Chubby 与 GFS、Bigtable 和 MapReduce 相比并没有那么大的名气，不过它在 Google 内部也是非常重要的基础设施。

![2021-02-23-NciKlE](https://image.ldbmcs.com/2021-02-23-NciKlE.jpg)

相比于名不见经传的 Chubby，作者相信 Zookeeper 更被广大开发者所熟知，作为非常出名的分布式协调服务，Zookeeper 有非常多的应用，包括发布订阅、命名服务、分布式协调和分布式锁，这篇文章主要会介绍 Zookeeper 的实现原理以及常见的应用，但是在具体介绍 Zookeeper 的功能和原理之前，我们会简单介绍一下分布式锁服务 Chubby 以及它与 Zookeeper 之间的异同。

## 1. Chubby

作为分布式锁服务，Chubby 的目的就是允许多个客户端对它们的行为进行同步，同时也能够解决客户端的环境相关信息的分发和粗粒度的同步问题，GFS 和 Bigtable 都使用了 Chubby 以解决主节点的选举等问题。在网络上你很难找到关于 Chubby 的相关资料，我们只能从 [The Chubby lock service for loosely-coupled distributed systems](https://ai.google/research/pubs/pub27897) 一文中窥见它的一些设计思路、技术架构等信息。

虽然 Chubby 和 Zookeeper 有着比较相似的功能，但是它们的设计理念却非常不同，Chubby 在论文的摘要中写道：

> We describe our experiences with the Chubby lock service, which is intended to provide coarse-grained locking as well as reliable (though low-volume) storage for a loosely-coupled distributed system.

从论文的摘要中我们可以看出 Chubby 首先被定义成一个**分布式的锁服务**，它能够为分布式系统提供**松耦合、粗粒度**的分布式锁功能，然而我们并不能依赖于它来做一些重量的数据存储。

![2021-02-23-61vAps](https://image.ldbmcs.com/2021-02-23-61vAps.jpg)

Chubby 在设计时做了两个重要的设计决定，一是提供完整、独立的分布式锁服务而非一个用于共识的库或者服务，另一个是选择提供小文件的读写功能，使得主节点能够方便地发布自己的状态信息。

### 1.1 系统架构

Chubby 总共由两部分组成，一部分是用于提供数据的读写接口并管理相关的配置数据的服务端，另一部分就是客户端使用的 SDK，为了提高系统的稳定性，每一个 Chubby 单元都由一组服务器组成，它会使用 [共识算法](https://draveness.me/consensus) 从集群中选举出主节点。

![2021-02-23-JsNpgy](https://image.ldbmcs.com/2021-02-23-JsNpgy.jpg)

在一个 Chubby Cell 中，**只有**主节点会对外提供读写服务，其他的节点其实都是当前节点的副本（Replica），它们只是维护一个数据的拷贝并会在主节点更新时对它们持有的数据库进行更新；客户端通过向副本发送请求获取主节点的位置，一旦它获取到了主节点的位置，就会向所有的读写请求发送给主节点，直到其不再响应为止。写请求都会通过一致性协议传播到所有的副本中，当集群中的多数节点都同步了请求时就会认为当前的写入已经被确认。

当主节点宕机时，副本会在其租约到期时重新进行选举，副本节点如果在宕机几小时还没有回复，那么系统就会从资源池中选择一个新的节点并在该节点上启动 Chubby 服务并更新 DNS 表。

![2021-02-23-M44QAb](https://image.ldbmcs.com/2021-02-23-M44QAb.jpg)

主节点会不停地轮询 DNS 表获取集群中最新的配置，每次 DNS 表更新时，主节点都会将新的配置下发给 Chubby 集群中其他的副本节点。

由于这篇文章我们主要介绍的是 Zookeeper，所以对于 Chubby 就介绍到这里了，感兴趣的读者可以查看 [The Chubby lock service for loosely-coupled distributed systems](https://ai.google/research/pubs/pub27897) 了解更多相关的内容。

## 2. Zookeeper

很多人都会说 Zookeeper 是 Chubby 的一个开源实现，这其实是有问题的，它们两者只不过都提供了具有层级结构的命名空间：

![2021-02-23-hAoGGg](https://image.ldbmcs.com/2021-02-23-hAoGGg.jpg)

Chubby 和 Zookeeper 从最根本的设计理念上就有着非常明显的不同，在上文中我们已经提到了 Chubby 被设计成一个分布式的锁服务，它能够为分布式系统提供松耦合、粗粒度的分布式锁功能，然而我们并不能依赖于它来做一些重量的数据存储，而 Zookeeper 的论文在摘要中介绍到，它是一个能够为分布式系统提供**协调**功能的服务：

> In this paper, we describe ZooKeeper, a service for co- ordinating processes of distributed applications.

Zookeeper 的目的是为客户端构建复杂的协调功能提供简单、高效的核心 API，相比于 Chubby 对外提供已经封装好的更上层的功能，Zookeeper 提供了更抽象的接口以便于客户端自行实现想要完成的功能。

![2021-02-23-b39I6B](https://image.ldbmcs.com/2021-02-23-b39I6B.jpg)

Chubby 直接为用户提供封装好的锁和解锁的功能，内部完成了锁的实现，只是将 API 直接暴露给用户，而 Zookeeper 却需要用户自己实现分布式锁；总的来说，使用 Zookeeper 往往需要客户端做更多的事情，但是也享有更多的自由。

### 2.1 技术架构

与 Chubby 集群中，多个节点只有一个能够对外提供服务不同，Zookeeper 集群中所有的节点都可以对外提供服务，但是集群中的节点也分为主从两种节点，所有的节点都能处理来自客户端的读请求，但是只有主节点才能处理写入操作：

> 这里所说的 Zookeeper 集群主从节点实际上分别是 Leader 和 Follower 节点。

![2021-02-23-ZiDR0H](https://image.ldbmcs.com/2021-02-23-ZiDR0H.jpg)

客户端使用 Zookeeper 时会连接到集群中的任意节点，所有的节点都能够直接对外提供读操作，但是写操作都会被从节点路由到主节点，由主节点进行处理。

Zookeeper 在设计上提供了以下的两个基本的顺序保证，线性写和先进先出的客户端顺序：

![2021-02-23-hvoq3U](https://image.ldbmcs.com/2021-02-23-hvoq3U.jpg)

其中线性写是指所有更新 Zookeeper 状态的请求都应该按照既定的顺序串行执行；而先进先出的客户端顺序是指，所有客户端发出的请求会按照发出的顺序执行。

### 2.2 Zab 协议

在我们简单介绍 Zookeeper 的技术架构之后，这一节将谈及 Zookeeper 中的 Zab 协议，Zookeeper 的 Zab 协议是为了解决分布式一致性而设计出的一种协议，它的全称是 Zookeeper 原子广播协议，它能够在发生崩溃时快速恢复服务，达到高可用性。

![2021-02-23-Ai3Rwm](https://image.ldbmcs.com/2021-02-23-Ai3Rwm.jpg)

如上一节提到的，客户端在使用 Zookeeper 服务时会随机连接到集群中的一个节点，所有的读请求都会由当前节点处理，而写请求会被路由给主节点并由主节点向其他节点广播事务，与 2PC 非常相似，如果在所有的节点中超过一半都返回成功，那么当前写请求就会被提交。

当主节点崩溃时，其他的 Replica 节点会进入崩溃恢复模式并重新进行选举，Zab 协议必须确保提交已经被 Leader 提交的事务提案，同时舍弃被跳过的提案，这也就是说当前集群中最新 `ZXID` 最大的服务器会被选举成为 Leader 节点；但是在正式对外提供服务之前，新的 Leader 也需要先与 Follower 中的数据进行同步，确保所有节点拥有完全相同的提案列表。

![2021-02-23-3iyS0n](https://image.ldbmcs.com/2021-02-23-3iyS0n.jpg)

在上面提到 ZXID 其实就是 Zab 协议中设计的事务编号，它是一个 64 位的整数，其中最低的 32 位是一个计数器，每当客户端修改 Zookeeper 集群状态时，Leader 都会以当前 ZXID 值作为提案的编号创建一个新的事务，在这之后会将当前计数器加一；ZXID 中高的 32 位表示当前 Leader 的任期，每当发生崩溃进入恢复模式，集群的 Leader 重新选举之后都会将 epoch 加一。

### 2.3 Zab 和 Paxos

Zab 和 Paxos 协议在实现上其实有非常多的相似点，例如：

- 主节点会向所有的从节点发出提案；
- 主节点在接收到一组从节点中 50% 以上节点的确认后，才会认为当前提案被提交了；
- Zab 协议中的每一个提案都包含一个 epoch 值，与 Paxos 中的 Ballot 非常相似；

因为它们有一些相同的特点，所以有的观点会认为 Zab 是 Paxos 的一个简化版本，但是 Zab 和 Paxos 在设计理念上就有着比较大的不同，两者的主要区别就在于 Zab 主要是为构建高可用的主备系统设计的，而 Paxos 能够帮助工程师搭建具有一致性的状态机系统。

作为一个一致性状态机系统，它能够保证集群中任意一个状态机副本都按照客户端的请求执行了相同顺序的请求，即使来自客户端请求是异步的并且不同客户端的接收同一个请求的顺序不同，集群中的这些副本就是会使用 Paxos 或者它的变种对提案达成一致；在集群运行的过程中，如果主节点出现了错误导致宕机，其他的节点会重新开始进行选举并处理未提交的请求。

但是在类似 Zookeeper 的高可用主备系统中，所有的副本都需要对增量的状态更新顺序达成一致，这些状态更新的变量都是由主节点创建并发送给其他的从节点的，每一个从节点都会严格按照顺序逐一的执行主节点生成的状态更新请求，如果 Zookeeper 集群中的主节点发生了宕机，新的主节点也必须严格按照顺序对请求进行恢复。

总的来说，使用状态更新节点数据的主备系统相比根据客户端请求改变状态的状态机系统对于请求的执行顺序有着更严格的要求。

> 这一节对于 Zab 和 Paxos 区别的介绍大都来自于 [Zab vs. Paxos](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab+vs.+Paxos) ，有兴趣的读者可以阅读相关的内容。

### 2.4 消息广播模式

说白了就是 `ZAB` 协议是如何处理写请求的，上面我们不是说只有 `Leader` 能处理写请求嘛？那么我们的 `Follower` 和 `Observer` 是不是也需要 **同步更新数据** 呢？总不能数据只在 `Leader` 中更新了，其他角色都没有得到更新吧？

不就是 **在整个集群中保持数据的一致性** 嘛？如果是你，你会怎么做呢？

废话，第一步肯定需要 `Leader` 将写请求 **广播** 出去呀，让 `Leader` 问问 `Followers` 是否同意更新，如果超过半数以上的同意那么就进行 `Follower` 和 `Observer` 的更新（和 `Paxos` 一样）。当然这么说有点虚，画张图理解一下。

![](https://image.ldbmcs.com/2020-03-11-021812.jpg)

嗯。。。看起来很简单，貌似懂了🤥🤥🤥。这两个 `Queue` 哪冒出来的？答案是 **`ZAB` 需要让 `Follower` 和 `Observer` 保证顺序性** 。何为顺序性，比如我现在有一个写请求A，此时 `Leader` 将请求A广播出去，因为只需要半数同意就行，所以可能这个时候有一个 `Follower` F1因为网络原因没有收到，而 `Leader` 又广播了一个请求B，因为网络原因，F1竟然先收到了请求B然后才收到了请求A，这个时候请求处理的顺序不同就会导致数据的不同，从而 **产生数据不一致问题** 。

所以在 `Leader` 这端，它为每个其他的 `zkServer` 准备了一个 **队列** ，采用先进先出的方式发送消息。由于协议是 **通过 `TCP` **来进行网络通信的，保证了消息的发送顺序性，接受顺序性也得到了保证。

除此之外，在 `ZAB` 中还定义了一个 **全局单调递增的事务ID `ZXID`** ，它是一个64位long型，其中高32位表示 `epoch` 年代，低32位表示事务id。`epoch` 是会根据 `Leader` 的变化而变化的，当一个 `Leader` 挂了，新的 `Leader` 上位的时候，年代（`epoch`）就变了。而低32位可以简单理解为递增的事务id。

定义这个的原因也是为了顺序性，每个 `proposal` 在 `Leader` 中生成后需要 **通过其 `ZXID` 来进行排序** ，才能得到处理。

### 2.5 崩溃恢复模式

说到崩溃恢复我们首先要提到 `ZAB` 中的 `Leader` 选举算法，当系统出现崩溃影响最大应该是 `Leader` 的崩溃，因为我们只有一个 `Leader` ，所以当 `Leader` 出现问题的时候我们势必需要重新选举 `Leader` 。

`Leader` 选举可以分为两个不同的阶段，第一个是我们提到的 `Leader` 宕机需要重新选举，第二则是当 `Zookeeper` 启动时需要进行系统的 `Leader` 初始化选举。下面我先来介绍一下 `ZAB` 是如何进行初始化选举的。

假设我们集群中有3台机器，那也就意味着我们需要两台以上同意（超过半数）。比如这个时候我们启动了 `server1` ，它会首先 **投票给自己** ，投票内容为服务器的 `myid` 和 `ZXID` ，因为初始化所以 `ZXID` 都为0，此时 `server1` 发出的投票为 (1,0)。但此时 `server1` 的投票仅为1，所以不能作为 `Leader` ，此时还在选举阶段所以整个集群处于 **`Looking` 状态**。

接着 `server2` 启动了，它首先也会将投票选给自己(2,0)，并将投票信息广播出去（`server1`也会，只是它那时没有其他的服务器了），`server1` 在收到 `server2` 的投票信息后会将投票信息与自己的作比较。**首先它会比较 `ZXID` ，`ZXID` 大的优先为 `Leader`，如果相同则比较 `myid`，`myid` 大的优先作为 `Leader`**。所以此时`server1` 发现 `server2` 更适合做 `Leader`，它就会将自己的投票信息更改为(2,0)然后再广播出去，之后`server2`  收到之后发现和自己的一样无需做更改，并且自己的 **投票已经超过半数** ，则 **确定 `server2` 为 `Leader`**，`server1` 也会将自己服务器设置为 `Following` 变为 `Follower`。整个服务器就从 `Looking` 变为了正常状态。

当 `server3` 启动发现集群没有处于 `Looking` 状态时，它会直接以 `Follower` 的身份加入集群。

还是前面三个 `server` 的例子，如果在整个集群运行的过程中 `server2` 挂了，那么整个集群会如何重新选举 `Leader` 呢？其实和初始化选举差不多。

首先毫无疑问的是剩下的两个 `Follower` 会将自己的状态 **从 `Following` 变为 `Looking` 状态** ，然后每个 `server` 会向初始化投票一样首先给自己投票（这不过这里的 `zxid` 可能不是0了，这里为了方便随便取个数字）。

假设 `server1` 给自己投票为(1,99)，然后广播给其他 `server`，`server3` 首先也会给自己投票(3,95)，然后也广播给其他 `server`。`server1` 和 `server3` 此时会收到彼此的投票信息，和一开始选举一样，他们也会比较自己的投票和收到的投票（`zxid` 大的优先，如果相同那么就 `myid` 大的优先）。这个时候 `server1` 收到了 `server3` 的投票发现没自己的合适故不变，`server3` 收到 `server1` 的投票结果后发现比自己的合适于是更改投票为(1,99)然后广播出去，最后 `server1` 收到了发现自己的投票已经超过半数就把自己设为 `Leader`，`server3` 也随之变为 `Follower`。

> 请注意 `ZooKeeper` 为什么要设置奇数个结点？比如这里我们是三个，挂了一个我们还能正常工作，挂了两个我们就不能正常工作了（已经没有超过半数的节点数了，所以无法进行投票等操作了）。而假设我们现在有四个，挂了一个也能工作，**但是挂了两个也不能正常工作了**，这是和三个一样的，而三个比四个还少一个，带来的效益是一样的，所以 `Zookeeper` 推荐奇数个 `server` 。

那么说完了 `ZAB` 中的 `Leader` 选举方式之后我们再来了解一下 **崩溃恢复** 是什么玩意？

其实主要就是 **当集群中有机器挂了，我们整个集群如何保证数据一致性？**

如果只是 `Follower` 挂了，而且挂的没超过半数的时候，因为我们一开始讲了在 `Leader` 中会维护队列，所以不用担心后面的数据没接收到导致数据不一致性。

如果 `Leader` 挂了那就麻烦了，我们肯定需要先暂停服务变为 `Looking` 状态然后进行 `Leader` 的重新选举（上面我讲过了），但这个就要分为两种情况了，分别是 **确保已经被Leader提交的提案最终能够被所有的Follower提交** 和 **跳过那些已经被丢弃的提案** 。

确保已经被Leader提交的提案最终能够被所有的Follower提交是什么意思呢？

假设 `Leader (server2)` 发送 `commit` 请求（忘了请看上面的消息广播模式），他发送给了 `server3`，然后要发给 `server1` 的时候突然挂了。这个时候重新选举的时候我们如果把 `server1` 作为 `Leader` 的话，那么肯定会产生数据不一致性，因为 `server3` 肯定会提交刚刚 `server2` 发送的 `commit` 请求的提案，而 `server1` 根本没收到所以会丢弃。

![](https://image.ldbmcs.com/2020-03-11-021953.jpg)

那怎么解决呢？

聪明的同学肯定会质疑，**这个时候 `server1` 已经不可能成为 `Leader` 了，因为 `server1` 和 `server3` 进行投票选举的时候会比较 `ZXID` ，而此时 `server3` 的 `ZXID` 肯定比 `server1` 的大了**。(不理解可以看前面的选举算法)

那么跳过那些已经被丢弃的提案又是什么意思呢？

假设 `Leader (server2)` 此时同意了提案N1，自身提交了这个事务并且要发送给所有 `Follower` 要 `commit` 的请求，却在这个时候挂了，此时肯定要重新进行 `Leader` 的选举，比如说此时选 `server1` 为 `Leader` （这无所谓）。但是过了一会，这个 **挂掉的 `Leader` 又重新恢复了** ，此时它肯定会作为 `Follower` 的身份进入集群中，需要注意的是刚刚 `server2` 已经同意提交了提案N1，但其他 `server` 并没有收到它的 `commit` 信息，所以其他 `server` 不可能再提交这个提案N1了，这样就会出现数据不一致性问题了，所以 **该提案N1最终需要被抛弃掉** 。

![](https://image.ldbmcs.com/2020-03-11-022014.jpg)

## 3. 实现原理

这一节会简单介绍 Zookeeper 的一些实现原理，重点会介绍以下几个部分的内容：文件系统、临时/持久节点和通知的实现原理。

### 3.1 文件系统

了解或者使用 Zookeeper 或者其他分布式协调服务的读者对于使用类似文件系统的方式比较熟悉，与 Unix 中的文件系统份上相似的是，Zookeeper 中也使用文件系统组织系统中存储的资源。

![2021-02-23-j4RAt9](https://image.ldbmcs.com/2021-02-23-j4RAt9.jpg)

Zookeeper 中其实并没有文件和文件夹的概念，它只有一个 Znode 的概念，它既能作为容器存储数据，也可以持有其他的 Znode 形成父子关系。

Znode 其实有 `PERSISTENT`、`PERSISTENT_SEQUENTIAL`、`EPHEMERAL` 和 `EPHEMERAL_SEQUENTIAL` 四种类型，它们是**临时与持久、顺序与非顺序**两个不同的方向组合成的四种类型。

![2021-02-23-rTMnXl](https://image.ldbmcs.com/2021-02-23-rTMnXl.jpg)

临时节点是客户端在连接 Zookeeper 时才会保持存在的节点，一旦客户端和服务端之间的连接中断，当前连接持有的所有节点都会被删除，而持久的节点不会随着会话连接的中断而删除，它们需要被客户端主动删除；Zookeeper 中另一种节点的特性就是顺序和非顺序，如果我们使用 Zookeeper 创建了顺序的节点，那么所有节点就会在名字的末尾附加一个序列号，序列号是一个由父节点维护的单调递增计数器。

### 3.2 通知

常见的通知机制往往都有两种，一种是客户端使用『拉』的方式从服务端获取最新的状态，这种方式获取的状态很有可能都是过期的，需要客户端不断地通过轮询的方式获取服务端最新的状态，另一种方式就是在客户端订阅对应节点后由服务端向所有订阅者推送该节点的变化，相比于客户端主动获取数据的方式，服务端主动推送更能够保证客户端数据的实时性。

作为分布式协调工具的 Zookeeper 就实现了这种服务端主动推送请求的机制，也就是 `Watch`，当客户端使用 `getData` 等接口获取 Znode 状态时传入了一个用于处理节点变更的回调，那么服务端就会主动向客户端推送节点的变更：

```java
public byte[] getData(final String path, Watcher watcher, Stat stat)
```

从这个方法中传入的 `Watcher` 对象实现了相应的 `process` 方法，每次对应节点出现了状态的改变，`WatchManager` 都会通过以下的方式调用传入 `Watcher` 的方法：

```java
Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
    WatchedEvent e = new WatchedEvent(type, KeeperState.SyncConnected, path);
    Set<Watcher> watchers;
    synchronized (this) {
        watchers = watchTable.remove(path);
    }
    for (Watcher w : watchers) {
        w.process(e);
    }
    return watchers;
}
```

Zookeeper 中的所有数据其实都是由一个名为 `DataTree` 的数据结构管理的，所有的读写数据的请求最终都会改变这颗树的内容，在发出读请求时可能会传入 `Watcher` 注册一个回调函数，而写请求就可能会触发相应的回调，由 `WatchManager` 通知客户端数据的变化。

![2021-02-23-k08QL9](https://image.ldbmcs.com/2021-02-23-k08QL9.jpg)

通知机制的实现其实还是比较简单的，通过读请求设置 `Watcher` 监听事件，写请求在触发事件时就能将通知发送给指定的客户端。

### 3.3 会话

在 Zookeeper 中一个非常重要的概念就是会话，客户端与服务器之间的任何操作都与 Zookeeper 中会话的概念有关，比如我们再上一节中提到的临时节点生命周期以及通知的机制等等，它们都是基于会话来实现的。

每当客户端与服务端建立连接时，其实创建了一个新的会话，在每一个会话的生命周期中，Zookeeper 会在不同的会话状态之间进行切换，比如说：CONNECTING、CONNECTED、RECONNECTING、RECONNECTED 和 CLOSE 等。

![2021-02-23-pjc48X](https://image.ldbmcs.com/2021-02-23-pjc48X.jpg)

作为 Zookeeper 中最重要的概念之一，每一个 Session 都包含四个基本属性，会话的唯一 ID、会话超时时间、下次会话的超时时间点和表示会话是否被关闭的标记。

`SessionTracker` 是 Zookeeper 中的会话管理器，它负责所有会话的创建、管理以及清理工作，但是它本身只是一个 Java 的接口，定义了一系列用于管理会话的相关接口：

```java
public interface SessionTracker {
    public static interface Session {
        long getSessionId();
        int getTimeout();
        boolean isClosing();
    }
    public static interface SessionExpirer {
        void expire(Session session);

        long getServerId();
    }

    long createSession(int sessionTimeout);
    boolean trackSession(long id, int to);
    boolean commitSession(long id, int to);
    boolean touchSession(long sessionId, int sessionTimeout);
    void setSessionClosing(long sessionId);
    void shutdown();
    void removeSession(long sessionId);
}
```

与其他的长连接一样，Zookeeper 中的会话也需要客户端与服务端之间进行心跳检测，客户端会在超时时间内向服务端发送心跳请求来保证会话不会被服务端关闭，一旦服务端检测到某一个会话长时间没有收到心跳包就会中断当前会话释放服务器上的资源。

## 4. 应用

作为分布式协调服务，Zookeeper 能够为集群提供分布式一致性的保证，我们可以通过 Zookeeper 提供的最基本的 API 组合成更高级的功能：

```java
public class Zookeeper {
    public String create(final String path, byte data[], List<ACL> acl, CreateMode createMode)
    public void delete(final String path, int version) throws InterruptedException, KeeperException
    public Stat exists(final String path, Watcher watcher) throws KeeperException, InterruptedException
    public byte[] getData(final String path, Watcher watcher, Stat stat) throws KeeperException, InterruptedException
    public Stat setData(final String path, byte data[], int version) throws KeeperException, InterruptedException
    public void sync(final String path, VoidCallback cb, Object ctx)
}
```

在这一节中，我们将介绍如何在生产环境中使用 Zookeeper 实现发布订阅、命名服务、分布式协调以及分布式锁等功能。

### 4.1 发布订阅

通过 Zookeeper 进行数据的发布与订阅其实可以说是它提供的最基本功能，它能够允许多个客户端同时订阅某一个节点的变更并在变更发生时执行我们预先设置好的回调函数，在运行时改变服务的配置和行为：

```java
ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
zk.getData("/config", new Watcher() {
    public void process(WatchedEvent watchedEvent) {
        System.out.println(watchedEvent.toString());
    }
}, null);
zk.setData("/config", "draven".getBytes(), 0);

// WatchedEvent state:SyncConnected type:NodeDataChanged path:/config
```

发布与订阅是 Zookeeper 提供的一个最基本的功能，它的使用非常的简单，我们可以在 `getData` 中传入实现 `process` 方法的 `Watcher` 对象，在每次改变节点的状态时，`process` 方法都会被调用，在这个方法中就可以对变更进行响应动态修改一些行为。

![2021-02-23-7HkK2e](https://image.ldbmcs.com/2021-02-23-7HkK2e.jpg)

通过 Zookeeper 这个中枢，每一个客户端对节点状态的改变都能够推送给节点的订阅者，在发布订阅模型中，Zookeeper 的每一个节点都可以被理解成一个主题，每一个客户端都可以向这个主题推送详细，同时也可以订阅这个主题中的消息；只是 Zookeeper 引入了文件系统的父子层级的概念将发布订阅功能实现得更加复杂。

```java
public static enum EventType {
    None(-1),
    NodeCreated(1),
    NodeDeleted(2),
    NodeDataChanged(3),
    NodeChildrenChanged(4);
}
```

如果我们订阅了一个节点的变更信息，那么该节点的子节点出现数量变更时就会调用 `process` 方法通知观察者，这也意味着更复杂的实现，同时和专门做发布订阅的中间件相比也没有性能优势，在海量推送的应用场景下，消息队列更能胜任，而 Zookeeper 更适合做一些类似服务配置的动态下发的工作。

### 4.2 命名服务

除了实现服务配置数据的发布与订阅功能，Zookeeper 还能帮助分布式系统实现命名服务，在每一个分布式系统中，客户端应用都有根据指定名字获取资源、服务器地址的需求，在这时就要求整个集群中的全部服务有着唯一的名字。

在大型分布式系统中，有两件事情非常常见，一是不同服务之间的可能拥有相同的名字，另一个是同一个服务可能会在集群中部署很多的节点，Zookeeper 就可以通过文件系统和顺序节点解决这两个问题。

![2021-02-23-7N2QeT](https://image.ldbmcs.com/2021-02-23-7N2QeT.jpg)

在上图中，我们创建了两个命名空间，`/infrastructure` 和 `/business` 分别代表架构和业务部门，两个部门中都拥有名为 `metrics` 的服务，而业务部门的 `metrics` 服务也部署了两个节点，在这里使用了命名空间和顺序节点解决唯一标志符的问题。

```java
ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
zk.create("/metrics", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL);
zk.create("/metrics", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL);
List children = zk.getChildren("/", null);
System.out.println(children);

// [metrics0000000001, metrics0000000002]
```

使用上面的代码就能在 Zookeeper 中创建两个带序号的 `metrics` 节点，分别是 `metrics0000000001` 和 `metrics0000000002`，也就是说 Zookeeper 帮助我们保证了节点的唯一性，让我们能通过唯一的 ID 查找到对应服务的地址等信息。

### 4.3 协调分布式事务

Zookeeper 的另一个作用就是担任分布式事务中的协调者角色，在之前介绍 [分布式事务](https://draveness.me/distributed-transaction-principle) 的文章中我们曾经介绍过分布式事务本质上都是通过 2PC 来实现的，在两阶段提交中就需要一个协调者负责协调分布式事务的执行。

```java
ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
String path = zk.create("/transfer/tx", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL);

List ops = Arrays.asList(
        Op.create(path + "/cohort", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL),
        Op.create(path + "/cohort", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL),
        Op.create(path + "/cohort", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT_SEQUENTIAL)
);
zk.multi(ops);
```

当前节点作为协调者在每次发起分布式事务时都会创建一个 `/transfer/tx` 的持久顺序节点，然后为几个事务的参与者创建几个空白的节点，事务的参与者在收到事务时会向这些空白的节点中写入信息并监听这些节点中的内容。

![2021-02-23-JVnd43](https://image.ldbmcs.com/2021-02-23-JVnd43.jpg)

所有的事务参与者会向当前节点中写入提交或者终止，一旦当前的节点改变了事务的状态，其他节点就会得到通知，如果出现一个写入终止的节点，所有的节点就会回滚对分布式事务进行回滚。

使用 Zookeeper 实现强一致性的分布式事务其实还是一件比较困难的事情，一方面是因为强一致性的分布式事务本身就有一定的复杂性，另一方面就是 Zookeeper 为了给客户端提供更多的自由，对外暴露的都是比较基础的 API，对它们进行组装实现复杂的分布式事务还是比较麻烦的，对于如何使用 Zookeeper 实现分布式事务，我们可以在 [ZooKeeper Recipes and Solutions](https://zookeeper.apache.org/doc/r3.4.4/recipes.html) 一文中找到更为详细的内容。

### 4.4 分布式锁

在数据库中，锁的概念其实是非常重要的，常见的关系型数据库就会对排他锁和共享锁进行支持，而 Zookeeper 提供的 API 也可以让我们非常简单的实现分布式锁。

```java
ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
final String resource = "/resource";

final String lockNumber = zk
        .create("/resource/lock-", null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

List<String> locks = zk.getChildren(resource, false, null);
Collections.sort(locks);

if (locks.get(0).equals(lockNumber.replace("/resource/", ""))) {
    System.out.println("Acquire Lock");
    zk.delete(lockNumber, 0);
} else {
    zk.getChildren(resource, new Watcher() {
        public void process(WatchedEvent watchedEvent) {
            try {
                ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
                List locks = zk.getChildren(resource, null, null);
                Collections.sort(locks);

                if (locks.get(0).equals(lockNumber.replace("/resource/", ""))) {
                    System.out.println("Acquire Lock");
                    zk.delete(lockNumber, 0);
                }

            } catch (Exception e) {}
        }
    }, null);
}
```

如果多个服务同时要对某个资源进行修改，就可以使用上述的代码来实现分布式锁，假设集群中存在一个资源 `/resource`，几个服务需要通过分布式锁保证资源只能同时被一个节点使用，我们可以用创建临时顺序节点的方式实现分布式锁；当我们创建临时节点后，通过 `getChildren` 获取当前等待锁的全部节点，如果当前节点是所有节点中序号最小的就得到了当前资源的使用权限，在对资源进行处理后，就可以通过删除 `/resource/lock-00000000x` 来释放锁，如果当前节点不是最小值，就会注册一个 `Watcher` 等待 `/resource` 子节点的变化直到当前节点的序列号成为最小值。

上述代码在集群中争夺同一资源的服务器特别多的情况下会出现羊群效应，每次子节点改变时都会通知当前节点，造成资源的浪费，我们其实可以将 `getChildren` 换成 `getData`，让当前节点只监听前一个节点的删除事件：

```java
Integer number = Integer.parseInt(lockNumber.replace("/resource/lock-", "")) - 1;
String previousLock = "/resource/lock-" + String.format("%010d", number);

zk.getData(previousLock, new Watcher() {
    public void process(WatchedEvent watchedEvent) {
        try {
            if (watchedEvent.getType() == Event.EventType.NodeDeleted) {
                System.out.println("Acquire Lock");
                ZooKeeper zk = new ZooKeeper("localhost", 3000, null);
                zk.delete(lockNumber, 0);
            }
        } catch (Exception e) {}
    }
}, null);
```

在新的分布式锁实现中，我们减少了每一个服务需要关注的事情，只让它们监听需要关心的数据变更，减少 Zookeeper 发送不必要的通知影响效率。

![2021-02-23-X2tguC](https://image.ldbmcs.com/2021-02-23-X2tguC.jpg)

分布式锁作为分布式系统中比较重要的一个工具，确实有着比较多的应用，同时也有非常多的实现方式，除了 Zookeeper 之外，其他服务例如 Redis 和 etcd 也能够实现分布式锁，为分布式系统的构建提供支持，不过在这篇文章中就不展开介绍了。

## 5. 总结

我们在这篇文章中简单介绍了 Google 的分布式锁服务 Chubby 以及同样能够提供分布式锁服务功能的 Zookeeper。

作为分布式协调服务，Zookeeper 的应用场景非常广泛，不仅能够用于服务配置的下发、命名服务、协调分布式事务以及分布式锁，还能够用来实现微服务治理中的服务注册以及发现等功能，这些其实都源于 Zookeeper 能够提供高可用的分布式协调服务，能够为客户端提供分布式一致性的支持，在后面的文章中作者也会介绍其他用于分布式协调的服务。

## 6. 参考

- [Distributed Coordination with Zookeeper](https://www.igvita.com/2010/04/30/distributed-coordination-with-zookeeper/)
- [ZooKeeper: Wait-free coordination for Internet-scale systems](https://www.usenix.org/legacy/event/usenix10/tech/full_papers/Hunt.pdf)
- [The Chubby lock service for loosely-coupled distributed systems](https://ai.google/research/pubs/pub27897)
- [Zookeeper vs Chubby](http://catkang.github.io/2017/10/10/zookeeper-vs-chubby.html)
- [Zab vs. Paxos](https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab+vs.+Paxos)
- [Zab: High-performance broadcast for primary-backup systems](Zab: High-performance broadcast for primary-backup systems)
- [ZooKeeper Recipes and Solutions](https://zookeeper.apache.org/doc/r3.4.4/recipes.html)
- [分布式事务的实现原理](https://draveness.me/distributed-transaction-principle)

