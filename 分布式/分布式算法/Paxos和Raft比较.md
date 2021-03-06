> 转载：[分布式中的一致性算法：Paxos和Raft比较](https://www.jianshu.com/p/ee645aeba365)

## 1. 概述

分布式中的一致性可以被描述为在协作解决问题的一组操作之间达成一致的行为。随着开源分布式计算和存储平台的兴起，一致性算法已成为复制的基本工具。**其中Paxos和Raft是最受欢迎的一致性算法，通过消除单点故障来提高系统的弹性**。

虽然Paxos在分布式一致性方面主导着学术和商业话语，但协议本身太复杂而无法推理，因此需要更易理解的算法。研究人员对Paxos进行了广泛的研究，而Raft在工程师中非常受欢迎。Raft的受欢迎程度来自这样一个事实：尽管研究人员对Paxos感兴趣，但工程师仍然需要阅读几篇论文，以便能够理解并创建解决实际问题的解决方案，并在通信步骤方面提供良好的性能。此外，他们仍然需要填补自定义实施的一些空白，这些实施有时会变得非常脆弱。

为了克服这个障碍，Diego Ongaro和John Ousterhout创建了一个名为Raft的新一致性算法，**它被设计为更容易理解，并为构建实用系统提供了比Paxos更好的基础**。虽然Raft为分布式系统的复杂世界带来了一些新鲜血液，但它仍然与Paxos有许多共同之处。例如，**两者都要选出一位负责决定协商一致的领导者**。

在这篇博文中，我们将简要介绍Paxos和Raft之间的相同点和不同点。首先，我们将描述一致的算法是什么。其次，我们将描述如何使用一致性算法的实例来构建复制解决方案。然后我们将描述如何在算法和一些安全和活跃属性中选出领导者。

## 2. 一致性

分布式系统的特征在于一组安全性和活跃性或两者的混合。简单地讲，安全是一种财产，规定在执行程序期间不会发生任何不良事件。另一方面，活跃性规定最终会发生一些好事。

一致性的目标是使一组服务器在一个值上达成一致，所以活跃的特征在于最终每个服务器都可以决定一个值。安全性表明没有两台服务器来设定值。

不幸的是，服务器执行算法步骤可能比其他服务器花费更长时间，并且可能崩溃并停止处理一致性算法。邮件可能会延迟，无序传递或丢失。这些方面使得一致性算法的实施变得非常困难，并迫使它们在“不稳定”期间降低标准并保持安全。确切地说，当系统变得“稳定”时虽然未知，但最终它将保持足够长的“稳定”，以便一致性算法能够做出决定达成最终的一致性。

在稳定运行中，系统需要两个通信步骤：leader - （1） - > servers - （2） - > leader：

![2020-07-31-H2r7XR](https://image.ldbmcs.com/2020-07-31-H2r7XR.jpg)

领导者向所有服务器发送它想要达成协议的值，并且每个服务器回复给领导者，通知他已经接受了请求。因此，**当领导者从法定数量（n/2+1节点）的服务器接收消息时，就达成了协议**。

请注意，我们在此分析中省略了两条消息：将服务器希望与领导者达成协议的值转发给服务器的消息以及通知服务器已达到该值协议的消息。如果服务器将接受消息发送到所有服务器，或者在领导者发送给服务器的下一条消息中捎带信息，则后一条消息可能不是必需的。

## 3. 复制

为了实现复制，运行一致性算法的几个实例，并且每个实例都被限制在复制日志中的一个槽条目中，该条目可能会持久存储在磁盘上。领导者可以并行运行多个实例以填充不同的插槽，从而提高性能。但是，并行度高度依赖于硬件、使用的网络和应用程序。

每个领导者都对自己当选时增加的一轮或一个周期负有独特的责任：

![2020-07-31-IgQ4l3](https://image.ldbmcs.com/2020-07-31-IgQ4l3.jpg)

## 4. 领导人选举

Paxos和Raft都认为最终会有一个领导者，所有稳定的服务器都会信任，而一个领导者负责一个周期（Term）。如果怀疑现任领导人有问题，新领导人将提出一个新任期，必须大于前一任期。

**在Raft中，服务器向其他服务器发送“领导请求”，并且在认为自己是领导者之前期望大多数服务器的回复。如果它没有得到大多数服务器的回复或者接收到另一个服务器已成为领导者的消息，它将超时并重新开始新的选举过程**。服务器每个Term只能投票给一个领导者请求。

但是，**Paxos并没有真正定义服务器如何成为领导者**。为简单起见，研究人员利用服务器id（整数）等进程之间的先后排名。因此，没有被怀疑的排名最高或最低的服务器成为新的领导者。虽然这是一个简单直观的解决方案，但它需要在服务器之间划分术语空间：新术语=旧术语+ N，其中N是服务器的最大数量。

Raft对领导者选举过程施加限制：**只有最新的服务器才能成为领导者**。基本上，它保证领导者拥有以前周期中的所有已提交条目，并且不需要了解它不知道的复制日志中的旧条目。因此，在成为领导者之后，服务器可以简单地在其他服务器上“强加”其“愿望”。

然而，Paxos允许任何服务器成为领导者。因此，服务器必须在开始在其他服务器上“强加”其“愿望”之前了解过去，提高了灵活性但也伴随着额外的复杂性。

![2020-07-31-p5VrpF](https://image.ldbmcs.com/2020-07-31-p5VrpF.jpg)

在Raft中，服务器1或服务器2可以成为领导者。而在Paxos中，任何一个都可以。

## 5. 安全

由于系统的异步性质，服务器可能在不同时间感知故障和选举。这意味着服务器可能会以不同的方式临时运行，但最终所有服务器都会收敛到一个Term。

在任何情况下，如果服务器从比其当前版本更早的Term获得消息，则这意味着发送者要么是领导者，要么试图成为旧Term中的一个，并且接收者必须拒绝该消息并通知发送者。

如果服务器从一个大于当前的Term获得消息，这意味着有一个新Term和一个新的领导者，并且接收者必须开始接受领导者的“愿望”。

但是，两种算法都必须小心避免覆盖旧领导做出的决定，从而违反安全规定。这就是Raft和Paxos分歧的地方，我们可以看到Raft使用了简单而优雅的方法。

如上所述，Raft对领导者选举算法施加限制，只有最新的服务器才能成为领导者：

**Raft通过比较日志中最后一个条目的索引和术语来确定两个日志中哪一个更新。如果日志包含具有不同术语的最后一个条目，则具有较晚术语的日志将更新。如果日志以相同的术语结束，则更长的日志更新是最新的。**

然后，领导者只需要确保服务器中的复制日志最终收敛，这是通过施加以下限制来完成的：如果服务器之前没有接受插槽的值，"n"服务器不能接受插槽的值"n - 1"。领导者包括当前请求中的先前日志条目的术语，并且如果其先前请求的术语与领导者发送的术语匹配，则服务器仅接受该请求。否则，它要求领导者首先发送先前失踪的请求，如此反复"n - 2"和  "n - 3"等。

在Paxos中，任何服务器都可以成为领导者，因此避免决策不被覆盖的任务变得有点复杂，因为新的领导者必须在开始“强加”它之前找出其他服务器已处理的内容。希望“在别人身上。这是Paxos算法的准备阶段，必须在选出新领导者后运行一次。准备消息包含新术语和插槽号，"n"通过该插槽号可以达到所有先前条目的协议。服务器回复有关大于的插槽的信息"n"，此信息用于限制新领导者为这些插槽建议的值。

## 6. 活跃度

只要大多数服务器还活着，保证n/2+1节点正常就能够提供服务。

## 7. 结论

我们已经展示了Raft和Paxos之间的相似之处，**关键的区别在于如何选出领导者并保持安全。在Raft中，只有最新的服务器才能成为领导者，而Paxos允许任何服务器成为领导者。**然而，这种灵活性伴随着额外的复杂性。

请注意，Raft和Paxos中的领导者可能会成为瓶颈，因为所有流量都会通过它。

