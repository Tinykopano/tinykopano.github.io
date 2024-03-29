---
title: In Search of an Understandable Consensus Algorithm (Extended Version)
date: 2022-12-01 14:22:00 +0800
categories: [Course,Paper]
tags: [distributed system,consensus algorithm, Raft]
---

# 寻找一种可理解的共识算法

## 摘要

Raft是一种共识算法，用于管理复制的日志。它产生的结果等同于（多）Paxos，并且它和Paxos一样高效，但它的结构与Paxos不同；这使得Raft比Paxos更容易理解，同时也为构建实用系统提供了更好的基础。为了提高可理解性，Raft将共识的关键元素分开，如leader、日志复制和安全等关键要素，并执行了更强的一致性，以减少必须考虑的状态数量。一个用户研究的结果表明，Raft比Paxos更容易被学生掌握。Raft还包括一个新的机制来改变群集成员，它使用重叠多数来保证安全。

<!-- more -->

## 一、  引言

共识算法允许一组机器作为一致的团体工作，可以在一些成员的故障中幸存下来。正因为如此，它们在构建可靠的大规模软件系统中发挥了关键作用。Paxos[15, 16]在过去十年中主导了关于共识算法的讨论：大多数共识的实现都是基于Paxos或受其影响，Paxos已经成为教学生了解共识的主要工具。

不幸的是，Paxos是相当难以理解的，尽管有许多尝试使它更容易理解。此外，它的结构需要复杂的变化以支持实际的系统。因此，无论是系统构建者和学生都在为Paxos而努力。

在我们为Paxos努力之后，我们开始寻找一种新的共识算法，为系统建设和教育提供一个更好的基础。我们的方法与普通方法不同，因为我们的主要目标是可理解性：我们能否为实用系统定义一个共识算法，并以一种明显比Paxos更容易学习的方式来描述它？此外，我们希望该算法能够促进直觉的发展，这对系统建设者来说是至关重要的。重要的是，不仅要让算法发挥作用，而且要让它显而易见地发挥作用。

这项工作的结果是一种叫做Raft的共识算法。在设计Raft的过程中，我们应用了特定的技术来提高可理解性，包括分解（Raft将leader选举、日志复制和安全分开）和减少状态空间（相对于Paxos，Raft减少了非确定性的程度和服务器之间的不一致方式）。对两所大学的43名学生进行的用户研究表明，Raft明显比Paxos更容易理解：在学习了两种算法之后，这些学生中有33人能够更好地回答有关Raft的问题，而不是有关Paxos的问题。

Raft在许多方面与现有的共识算法相似（最显著的是Oki和Liskov的Viewstamped Replication[29, 22]），但它有几个新的特点。

- **强leader**：与其他共识算法相比，Raft使用了一种更强的领导力形式。比如说。日志条目只从leader流向其他服务器。 这简化了对复制日志的管理 并使Raft更容易理解。

- **leader选举**：Raft使用随机的计时器来选举leader。这在任何共识算法已经需要的心跳上只增加了少量的机制，同时简单而迅速地解决了冲突。

- **成员变更**：Raft改变集群中的服务器集的机制使用了一种新的联合共识方法，在这种方法中，两个不同配置的多数在过渡期间重叠在一起。这使得集群在配置变化期间能够继续正常运行。

我们相信Raft优于Paxos和其他共识算法，无论是出于教育目的还是作为实施的基础。它比其他算法更简单，更容易理解；它的描述很完整，足以满足实际系统的需要。它有几个开源的实现，并被几个公司使用；它的安全属性已被正式规定和证明；它的效率可与其他算法相媲美。

本文的其余部分介绍了复制状态机问题（第2节），讨论了Paxos的优点和缺点（第3节），描述了我们对可理解性的一般方法（第4节），提出了Raft共识算法（第5-8节），评估了 Raft（第9节），并讨论了相关工作（第10节）。

## 二、  复制状态机

一致性算法是从复制状态机[37]的背景下提出的。在这种方法中，一组服务器上的状态机产生相同状态的副本，并且在一些机器宕掉的情况下也可以继续运行。复制状态机在分布式系统中被用于解决很多容错的问题。例如，大规模的系统中通常都有一个集群leader，像 GFS、HDFS 和 RAMCloud，典型应用就是一个独立的的复制状态机去管理leader选举和存储配置信息并且在leader宕机的情况下也要存活下来。比如 Chubby 和 ZooKeeper。

![](figure01.JPG "复制状态机的架构")

<div style="text-align: center;"><b>图1</b>：复制状态机的架构。一致性算法管理着来自客户端指令的复制日志。状态机从日志中处理相同顺序的相同指令，所以产生的结果也是相同的。</div>

复制状态机通常都是基于复制日志实现的，如图 1。每一个服务器存储一个包含一系列指令的日志，并且按照日志的顺序进行执行。每一个日志都按照相同的顺序包含相同的指令，所以每一个服务器都执行相同的指令序列。因为每个状态机都是确定的，每一次执行操作都产生相同的状态和同样的序列。

保证复制日志相同就是一致性算法的工作了。在一台服务器上，一致性模块接收客户端发送来的指令然后增加到自己的日志中去。它和其他服务器上的一致性模块进行通信来保证每一个服务器上的日志最终都以相同的顺序包含相同的请求，尽管有些服务器会宕机。一旦指令被正确的复制，每一个服务器的状态机按照日志顺序处理他们，然后输出结果被返回给客户端。因此，服务器集群看起来形成一个高可靠的状态机。

实际系统中使用的一致性算法通常含有以下特性：

- 在非拜占庭错误情况下，包括网络延迟、分区、丢包、冗余和乱序等错误都可以保证*安全性*（绝对不会返回一个错误的结果） 。
- 只要任何大多数的服务器都在运行，并能相互之间以及与客户进行通信，它们就能完全发挥作用（_可用_）。因此，一个典型的由五个服务器组成的集群可以容忍任何两个服务器的故障。服务器被停止就认为是失败；它们后来可能从稳定存储的状态中恢复并重新加入集群。
- 不依赖时序来保证一致性：物理时钟错误或者极端的消息延迟只有在最坏情况下才会导致可用性问题。
- 通常情况下，一条指令可以尽可能快的在集群中大多数节点响应一轮远程过程调用时完成；小部分比较慢的节点不会影响系统整体的性能。

## 三、  Paxos有哪些问题

在过去的十年里，Leslie Lamport的Paxos协议[15]几乎成了共识的代名词：它是课程中最常教授的协议，大多数共识的实现都以它为起点。Paxos首先定义了一个能够在单一决策上达成协议的协议，例如单一复制的日志条目。我们把这个子集称为single-decree Paxos。然后，Paxos结合这个协议的多个实例，以促进一系列的决定，如日志（多Paxos）。Paxos既保证了安全性，又保证了有效性，而且它支持集群成员的变化。它的正确性已被证明，而且在正常情况下是有效的。

不幸的是，Paxos有两个显著的缺点。第一个缺点是Paxos特别难理解。完整的解释[15]是出了名的不透明；而且在付出巨大努力之后，也很少有人能成功地理解它。因此，已经有一些人试图用更简单的术语来解释Paxos[16, 20, 21]。这些解释集中在single-decree子集上，但它们仍然具有挑战性。在对NSDI 2012的与会者进行的非正式调查中，我们发现很少有人对Paxos感到满意，即使在经验丰富的研究人员中。我们自己也在为Paxos挣扎；直到读了几个简化的解释和设计我们自己的替代协议后，我们才能够理解完整的协议，这个过程几乎花了一年时间。

我们假设，Paxos的不透明性来自于它选择single-decree子集作为基础。Single-decree Paxos是密集而微妙的：它分为两个阶段，没有简单的直观解释，不能被独立理解。正因为如此，很难发展出关于single-decree协议为何有效的直觉。Multi-Paxos的组成规则大大增加了复杂性和微妙性。我们认为，就多个决定达成共识的整体问题（即一个日志而不是一个条目）可以用其他更直接和明显的方式进行分解。

Paxos的第二个问题是，它没有为建立实际的实现提供一个良好的基础。原因之一是没有广泛认同的multi-Paxos的算法。Lamport的描述主要是关于single-decree Paxos的；他勾画了multi-Paxos的可能方法，但缺少许多细节。已经有一些尝试来充实和优化Paxos，如[26]、[39]和[13]，但这些尝试都不尽相同。彼此之间以及与Lamport的草图不同。诸如Chubby[4]等系统已经实现了类似Paxos的算法，但在大多数情况下，它们的细节还没有被公布。

此外，Paxos架构对于构建实用系统来说是一个糟糕的架构；这也是single-decree分解的另一个后果。例如，独立地选择一个日志条目集合，然后将它们拼接成一个连续的日志，这没有什么好处；这只会增加复杂性。围绕日志设计一个系统更简单、更有效，新的条目是以受限的顺序依次追加的。另一个问题是，Paxos的核心是使用对称的点对点方法（尽管它最终建议采用弱的leader形式作为性能优化）。这在一个只做一个决定的简化世界中是有意义的，但很少有实际的系统使用这种方法。如果必须做出一系列的决定，首先选举一个leader，然后让leader协调这些决定，这样做更简单，也更快。

因此，实际的系统与Paxos没有什么相似之处。每一个实现都是从Paxos开始的，发现实现它的困难，然后开发一个明显不同的架构。这既费时又容易出错，而理解Paxos的困难又加剧了这个问题。Paxos的表述对于证明其正确性的定理来说可能是一个很好的表述，但是真正的实现与Paxos有很大的不同，所以证明的价值不大。以下是一个典型的评论，来自于Chubby实现者:

> 在Paxos算法的描述和真实世界系统的需求之间存在着巨大的差距······最终的系统将基于一个未证明的协议[4]。

由于这些问题，我们得出结论，Paxos既不能为系统建设也不能为教育提供一个良好的基础。考虑到共识在大规模软件系统中的重要性，我们决定看看我们是否能设计出一种比Paxos具有更好特性的替代性共识算法。Raft就是这个实验的结果。

## 四、  为可理解性设计

我们在设计Raft时有几个目标：它必须为系统建设提供一个完整而实用的基础，从而大大减少开发人员所需的设计工作量；它必须在所有条件下都是安全的，并且在典型的操作条件下可用；它必须对普通操作有效。但我们最重要的目标和最困难的挑战是可理解性。必须让广大受众能够舒适地理解该算法。此外，还必须有可能发展关于该算法的直觉，以便系统构建者能够进行扩展，而这在现实世界的实现中是不可避免的。

在Raft的设计中，有很多地方我们必须在备选方法中做出选择。在这些情况下，我们根据可理解性对备选方案进行评估：解释每个备选方案有多难？(例如，它的状态空间有多复杂，它是否有微妙的影响？），读者完全理解该方法及其影响有多容易？

我们认识到，这种分析有很大程度的主观性；然而，我们使用了两种普遍适用的技术。第一种技术是众所周知的问题分解方法：在可能的情况下，我们把问题分成独立的部分，可以相对独立地解决、解释和理解。例如，在Raft中，我们将leader选举、日志复制、安全和成员变更分开。

我们的第二个方法是通过减少需要考虑的状态数量来简化状态空间，使系统更加连贯，并尽可能消除非确定性。具体来说，不允许日志有漏洞，而且Raft限制了日志相互之间不一致的方式。尽管在大多数情况下，我们试图消除非确定性，但在某些情况下，非确定性实际上提高了可理解性。特别是，随机化的方法引入了非确定性，但它们倾向于通过以类似的方式处理所有可能的选择来减少状态空间（"choose any; it doesn’t matter"）。我们使用随机化来简化Raft leader选举算法。

## 五、  Raft共识算法

Raft是一种用于管理第2节所述形式的复制日志的算法。图2概括了该算法的浓缩形式以供参考，图3列出了该算法的关键属性；这些图中的内容将在本节的其余部分逐一讨论。

![](figure02.JPG "Raft共识算法的简明摘要")

<div style="text-align: center;"><b>图2</b>：Raft共识算法的简明摘要（不包括成员变化和日志压缩）。左上角方框中的server行为被描述为一组独立和重复触发的规则。其它小节，如§5.2表示讨论特定功能的地方。一个正式的规范[31]更精确地描述了该算法。</div>

Raft通过首先选举一个特等的leader来实现共识，然后让leader完全负责管理复制的日志。leader接受来自客户端的日志条目，将其复制到其他服务器上，并告诉服务器何时可以将日志条目应用到他们的状态机上。这种一个leader的模式可以简化对复制日志的管理。例如，leader可以决定在日志中放置新条目的位置，而不需要咨询其他服务器，并且数据以一种简单的方式从leader流向其他服务器。一个leader可能会失败或与其他服务器断开连接，在这种情况下，会选出一个新的leader。

考虑到leader的选举方法，Raft将共识问题分解为三个相对独立的子问题，这将在下面的小节中讨论：

- **leader选举**：当现有的leader失败后，新的leader必须被选举产生（§5.2）。
- **日志复制**：leader必须接收客户端的日志条目然后通过集群复制，强制其他服务器的日志和leader的一样（§5.3）。
- **安全性**：Raft的关键安全属性是图3中的状态机安全属性：如果任何服务器对其状态机应用了一个特定的日志条目，那么其他服务器就不能对相同的日志索引应用不同的命令。第5.4节描述了Raft如何确保这一属性；该解决方案涉及对第5.2节中描述的选举机制的额外限制。

在介绍了共识算法之后，本节讨论了系统中的可用性问题和计时的作用。

![](figure03.JPG "Raft保证")

<div style="text-align: center;"><b>图3</b>：Raft保证这些属性总是为true。编号表示每个属性的讨论的章节。</div>

### 5.1 Raft基础

一个Raft集群包含多个服务器；五个是典型的数量，这使得系统可以容忍两个故障。在任何时候，每个服务器都处于三种状态之一：leader、follower或candidate。在正常操作中，有且仅有一个leader，其他所有的服务器都是follower。follower是被动的：他们自己不发出任何请求，只是回应leader和candidate的请求。leader处理所有客户端的请求（如果客户端联系follower，follower将其重定向到leader）。第三种状态，candidate，被用来选举一个新的leader，如第5.2节所述。图4显示了这些状态和它们的转换；下面将讨论这些转换。

![](figure04.JPG "Server状态")

<div style="text-align: center;"><b>图4</b>：Server状态。Follower只对来自其他服务器的请求作出回应。如果一个follower没有收到通信。它就会成为candidate并发起选举。一个candidate收到整个集群中大多数人的投票，就会成为新的leader。leader通常会一直运行到他们失败为止。</div>

![](figure05.JPG "时间与任期")

<div style="text-align: center;"><b>图5</b>：时间被划分为几个任期，每个任期以选举开始。选举成功后，由一位leader管理整个集群，直到任期结束。有些选举失败，在这种情况下，任期结束时没有选择leader。在不同的服务器上，可以在不同的时间观察到任期之间的转换。</div>

如图5所示，Raft将时间划分为任意长度的任期。任期用连续的整数来编号。每个任期以选举开始，其中一个或多个candidate试图成为leader，如第5.2节所述。如果一个candidates 在选举中获胜，那么他将在剩下的任期内担任leader。在某些情况下，选举的结果为分裂选票。在这种情况下，任期将在没有leader的情况下结束；新的任期（重新选举）将很快开始。Raft确保在一个给定的任期内最多有一个leader。

不同的服务器可能会在不同的时间观察到任期转换，在某些情况下，一个服务器可能不会观察到一个选举甚至整个任期。任期在Raft中充当了逻辑时钟[14]，它们允许服务器检测过时的信息，如过时的leader。每个服务器都存储一个当前的任期编号，该编号随时间单调地增加。每当服务器进行通信时，就会交换当前任期；如果一个服务器的当前任期比另一个服务器的小，那么它就将其当前任期更新为较大的值。如果一个candidate或leader发现它的任期已经过时，它将立即恢复到follower状态。如果一个服务器收到的请求是一个过时的任期编号，它将拒绝该请求。

Raft服务器使用远程过程调用（RPCs）进行通信，基本的共识算法只需要两种类型的RPCs。RequestVote RPCs由candidate在选举期间发起（第5.2节），AppendEntries RPCs由leader发起，用于复制日志条目并提供一种心跳形式（第5.3节）。第7节增加了第三个RPC，用于在服务器之间传输快照。如果服务器没有及时收到响应，它们会重试RPC，并且为了获得最佳性能，它们会并行地发出RPC。

### 5.2 leader选举

Raft使用心跳机制来触发leader选举。当服务器启动时，它们开始是follower。只要服务器收到来自leader或candidate的有效RPC，它就一直处于follower状态。leader定期向所有follower发送心跳（AppendEntries RPCs，不携带任何日志条目），以保持他们的权威。如果follower在一段被称为选举超时的时间内没有收到任何通信，那么它就认为没有可行的leader，并开始选举以选择一个新的leader。

为了开始选举，follower增加它的当前任期并过渡到candidate状态。然后，它为自己投票，并向集群中的每个其他服务器发出RequestVote RPCs。candidate一直处于这种状态，直到发生以下三种情况之一：（a）它赢得了选举，（b）另一个服务器确立了自己成为leader，或者（c）一段时间内没有赢家。这些结果将在下面的段落中分别讨论。

如果一个candidate在同一任期内获得了整个集群中大多数服务器的投票，那么它就赢得了选举。每台服务器在给定的任期内最多为一名candidate投票，以先来后到为原则（注：第5.4节对投票增加了一个额外的限制）。少数服从多数的原则保证了最多只有一名candidate能够在某一任期内赢得选举（图3中的选举安全属性）。一旦一个candidate在选举中获胜，它就成为leader。然后，它向所有其他服务器发送心跳信息，以建立其权威并防止新的选举。

在等待投票的过程中，candidate可能会收到一个来自另一个服务器的 AppendEntries RPC，该RPC来自另一服务器，声称自己是leader。如果leader的任期（包括在其RPC中）至少与candidate的当前任期一样大，那么candidate就会就承认该leader是合法的，并返回到follower状态。如果RPC中的任期小于candidate的，那么candidate就会拒绝RPC，并继续处于candidate状态。

第三种可能的结果是candidate既没有赢得选举也没有输掉选举：如果许多follower同时成为candidate，票数可能被分割，因此没有candidate获得多数票。当这种情况发生时，每个candidate都会超时，并通过增加其任期和启动新一轮的RequestVote RPC来开始新的选举。然而，如果没有额外的措施，分裂选票可能会无限期地重复。

Raft使用随机的选举超时，以确保分裂投票很少发生，并能迅速解决。为了从一开始就防止分裂投票，选举超时是从一个固定的时间间隔中随机选择的（例如150-300ms）。这就分散了服务器，所以在大多数情况下，只有一个服务器会超时；它赢得了选举，并在任何其他服务器超时之前发送心跳信号。同样的机制被用来处理分裂投票。每个候选人在选举开始时重新启动其随机选举超时，并等待超时过后再开始下一次选举；这减少了在新的选举中再次出现分裂票的可能性。第9.3节显示，这种方法可以迅速选出一个leader。

选举是说明可理解性如何指导我们在设计方案之间进行选择的一个例子。最初我们计划使用一个排名系统：每个candidate被分配一个独特的排名，用来在竞争的candidate之间进行选择。如果一个candidate发现了另一个排名更高的candidate，它就会回到follower状态，这样排名更高的candidate就能更容易地赢得下一次选举。我们发现这种方法在可用性方面产生了一些微妙的问题（如果一个排名较高的服务器失败了，一个排名较低的服务器可能需要超时并再次成为candidate，但如果它过早地这样做，它可能会重置选举leader的进展）。我们对算法进行了多次调整，但每次调整后都会出现新的corner case。最终我们得出结论，随机重试的方法更加明显和容易理解。

### 5.3 日志复制

一旦一个leader被选出，它就开始为客户端请求提供服务。每个客户请求都包含一个要由复制的状态机执行的命令。leader将命令作为一个新的条目附加到它的日志中，然后向其他每个服务器并行地发出AppendEntries RPCs来复制该条目。当条目被安全复制后（如下所述），leader将条目应用于其状态机，并将执行结果返回给客户端。如果follower崩溃或运行缓慢，或者网络数据包丢失，leader会无限期地重试AppendEntries RPCs（甚至在它回应了客户端之后），直到所有follower最终存储所有日志条目。

![](figure06.JPG "日志条目")

<div style="text-align: center;"><b>图6</b>：日志是由条目组成的，这些条目按顺序编号。每个条目都包含创建它的任期（每个框中的数字）和状态机的命令。如果一个条目可以安全地应用于状态机，那么该条目就被认为已经提交。</div>

日志的组织方式如图6所示。每个日志条目都存储了一个状态机命令，以及leader收到该条目时的任期编号。日志条目中的任期编号被用来检测日志之间的不一致，并确保图3中的一些属性。每个日志条目都有一个整数的索引，用来标识它在日志中的位置。

leader决定何时将日志条目应用于状态机是安全的；这样的条目被称为已提交。Raft保证已提交条目是持久化的，并且最终会被所有可用的状态机执行。一旦创建该条目的leader将其复制到大多数服务器上，该日志条目就会被提交（例如，图6中的条目7）。这也会提交leader日志中所有之前的条目，包括之前leader创建的条目。第5.4节讨论了在leader变更后应用这一规则时的一些微妙之处，它还表明这种提交的定义是安全的。leader会跟踪它所知道的已提交的最高索引，并且它在未来的AppendEntries RPC（包括心跳）中包括该索引，以便其他服务器最终发现。一旦follower得知一个日志条目被提交，它就会将该条目应用到它的本地状态机（按日志顺序）。

我们设计的Raft日志机制在不同服务器上的日志之间保持了高度的一致性。这不仅简化了系统的行为，使其更具可预测性，而且是确保安全的重要组成部分。Raft维护以下属性，它们共同构成了图3中的日志匹配属性：
- 如果不同日志中的两个条目具有相同的索引和任期，那么它们存储的是同一个命令。
- 如果不同日志中的两个条目具有相同的索引和任期，那么日志中的所有前面的条目都是相同的。

第一个属性来自于这样一个事实，即一个leader在一个给定的任期中最多创建一个具有给定日志索引的条目，并且日志条目永远不会改变它们在日志中的位置。第二个属性是由AppendEntries的一个简单一致性检查来保证的。当发送AppendEntries RPC时，leader包括其日志中紧接新条目之前的条目的索引和任期。如果follower在其日志中没有找到具有相同索引和任期的条目，那么它将拒绝新条目。一致性检查作为一个归纳步骤：日志的初始空状态满足了日志匹配属性，并且每当日志被扩展时，一致性检查都会保留日志匹配属性。因此，每当AppendEntries成功返回时，leader知道follow的日志与自己的日志在新条目之前是相同的。

在正常运行期间，leader和follower的日志保持一致，所以AppendEntries一致性检查从未失败。然而，leader崩溃会使日志不一致（旧leader可能没有完全复制其日志中的所有条目）。这些不一致会在一系列leader和follower的崩溃中加剧。图7说明了follower的日志可能与新leader的日志不同的方式。follower可能缺少leader的条目，可能有leader没有的额外条目，或者两者都有。日志中缺失和多余的条目可能跨越多个任期。

![](figure07.JPG "follower日志情况")

<div style="text-align: center;"><b>图7</b>：当顶部的leader上位后，在follower的日志中可能会出现（a-f）中的任何一种情况。每个框代表一个日志条目；框里的数字是其任期。一个follower可能缺少条目（a-b），可能有额外的未提交的条目（c-d），或者两者都有（e-f）。例如，如果该服务器是第2期的leader，在其日志中增加了几个条目，然后在提交任何条目之前就崩溃了；它很快重新启动，成为第3期的leader，并在其日志中增加了几个条目；在第2期或第3期的任何条目被提交之前，该服务器又崩溃了，并在几个任期内一直处于停机状态。</div>

在Raft中，leader通过强迫follower复制自己的日志来处理不一致的问题。follower的日志与自己的日志重复。这意味着follower日志中的冲突条目将被覆盖用leader日志中的条目覆盖。第5.4节将表明如果再加上一个限制，这就是安全的。

为了使follower的日志与自己的日志保持一致，leader必须找到两个日志一致的最新日志条目，删除follower日志中此后的任何条目，并向follower发送此后leader的所有条目。所有这些动作都是为了响应AppendEntries RPCs所进行的一致性检查而发生的。leader为每个follower维护一个nextIndex，它是leader将发送给该follower的下一个日志条目的索引。当leader第一次上台时，它将所有的nextIndex值初始化为其日志中最后一条的索引（图7中的11）。如果follower的日志与leader的日志不一致，在下一个AppendEntries RPC中，AppendEntries一致性检查将失败。在拒绝之后，leader会递减nextIndex并重试AppendEntries RPC。最终，nextIndex将达到一个点，即leader和follower的日志匹配。当这种情况发生时，AppendEntries将会成功，它将删除follower日志中任何冲突的条目，并从leader的日志中添加条目（如果有的话）。一旦AppendEntries成功，follower的日志就会与leader的日志一致，并且在剩下的时间里都会保持这种状态。

如果需要，该协议可以被优化以减少被拒绝的AppendEntries RPC的数量。例如，当拒绝一个AppendEntries请求时，follower可以包括冲突条目的任期和它为该任期存储的第一个索引。有了这些信息，leader可以递减nextIndex以绕过该任期中的所有冲突条目；每个有冲突条目的任期将需要一个AppendEntries RPC，而不是每个条目一个RPC。在实践中，我们怀疑这种优化是否有必要，因为故障不常发生，而且不太可能有很多不一致的条目。

有了这种机制，leader在上位后不需要采取任何特别的行动来恢复日志的一致性。它只是开始正常的操作，而日志会自动收敛以应对AppendEntries一致性检查的失败。一个leader永远不会覆盖或删除它自己日志中的条目（图3中leader的只可追加属性）。

这种日志复制机制表现出第2节中所描述的理想的共识属性：只要大多数服务器是正常的，Raft就可以接受、复制和应用新的日志条目；在正常情况下，一个新的条目可以通过单轮RPC复制到集群的大多数；而且一个缓慢的follower不会影响性能。

### 5.4 安全性

前面的章节描述了Raft如何选举leader和复制日志条目。然而，到目前为止所描述的机制还不足以确保每个状态机以相同的顺序执行完全相同的命令。例如，当leader提交几个日志条目时，一个follower可能无法使用，然后它可能被选为leader，并用新的条目覆盖这些条目；结果，不同的状态机可能执行不同的命令序列。

本节通过增加对哪些服务器可以被选为leader的限制，完善了Raft算法。该限制确保任何给定任期的leader都包含了之前任期中提交的所有条目（图3中的leader完整性属性）。考虑到选举限制，我们会使提交的规则更加精确。最后，我们提出一个leader完整性属性的简略证明，并说明它如何保证复制状态机的正确行为。

#### 5.4.1 选举约束

在任何基于leader的共识算法中，leader必须最终存储所有已提交的日志条目。在一些共识算法，例如Viewstamped Replication [22]，即使leader没有最初包含所有已提交的条目。这些算法包含额外的机制来识别丢失的条目，并在选举过程中或之后不久将它们传送给新的leader。不幸的是，这导致了相当多的额外机制和复杂性。Raft使用了一种更简单的方法，即它保证所有先前提交的条目的所有条目都存在于每个新leader身上。其选举的那一刻起就存在于每个新leader身上，而不需要将这些条目转移到leader。这意味着日志条目只在一个方向流动，即从leader到follower，而且leader永远不会覆盖他们日志中的现有条目。

Raft使用投票程序来防止candidate赢得选举，除非其日志包含所有已提交的条目。candidate必须与集群中的大多数人联系才能当选，这意味着每个已提交的条目必须至少存在于其中一个服务器中。如果candidate的日志至少和该多数人中的任何其他日志一样是最新的（这里的 "最新 "在下面有精确的定义），那么它将包含所有已提交的条目。RequestVote RPC实现了这一限制：RPC包括有关candidate日志的信息，如果投票人自己的日志比candidate的日志更及时，则拒绝投票。

Raft通过比较日志中最后条目的索引和任期来确定两个日志中哪一个是最新的。如果日志的最后条目有不同的任期，那么任期较晚的日志是更最新的。如果日志以相同的任期结束，那么日志更长的就是最新的。

#### 5.4.2 从前一个任期提交条目

如第5.3节所述，一旦一个条目被存储在大多数服务器上，leader就知道其当前任期的条目被提交。如果leader在提交条目之前崩溃了，未来的leader将试图完成对该条目的复制。然而，尽管前一任期的条目被存储在大多数服务器上，leader仍然不能立即得出结论该条目已被提交。图8展示了这样一种情况：一个旧的日志条目被存储在大多数服务器上，但仍然可以被未来的leader所覆盖。

![](figure08.JPG "图8")

<div style="text-align: center;"><b>图8</b>：一个时间序列显示了为什么leader不能使用旧任期的日志条目来确定提交。在（a）中，S1是leader，并部分复制了索引2的日志条目。在(b)中，S1崩溃了；S5凭借S3、S4和它自己的投票当选为第3任期的leader，并接受了日志索引2的不同条目。在（c）中，S5崩溃了；S1重新启动，被选为leader，并继续复制。在这一点上，第2项的日志条目已经在大多数服务器上复制，但它没有被提交。如果S1像(d)那样崩溃，S5可以被选为leader（有S2、S3和S4的投票），并用它自己的第3任期的条目覆盖该条目。然而，如果S1在崩溃前在大多数服务器上复制了其当前任期的条目，如(e)，那么这个条目就被提交了（S5不能赢得选举）。在这一点上，日志中所有前面的条目也被提交。</div>

为了消除类似图8中的问题，Raft从不通过计算副本来提交以前的日志条目。只有leader当前任期的日志条目是通过计算副本来提交的；一旦当前任期的条目以这种方式被提交，那么由于日志匹配特性，所有之前的条目都被间接地提交。在某些情况下，leader可以安全地断定一个较早的日志条目已被提交（例如，如果该条目存储在每个服务器上），但Raft为简单起见采取了更保守的方法。

Raft在提交规则中产生了这种额外的复杂性，因为当leader复制前几期的条目时，日志条目会保留其原来的任期编号。在其他共识算法中，如果一个新的leader复制之前 "任期 "的条目，它必须用新的 "任期编号 "来做。Raft的方法使得对日志条目的推理更加容易，因为它们在不同的时间和不同的日志中保持着相同的任期编号。此外，与其他算法相比，Raft的新leader从以前的任期中发送较少的日志条目（其他算法必须发送多余的日志条目，在它们被提交之前对其重新编号）。

#### 5.4.3 安全性争论

考虑到完整的Raft算法，我们现在可以更精确地论证leader完整性属性是否成立（这个论证是基于安全证明的，见第9.2节）。我们假设了leader完整性属性不成立，然后我们证明其矛盾。假设任期T的leader（leader<sub>T</sub>）从其任期中提交了一个日志条目，但是这个日志条目没有被未来某个任期的leader所存储。考虑最小的任期U > T，其leader（leader<sub>U</sub>）不存储该条目。

![](figure09.JPG "图9")

<div style="text-align: center;"><b>图9</b>：如果S1（任期T的leader）在其任期内提交了一个新的日志条目，而S5被选为后来任期U的leader，那么至少有一个服务器（S3）接受了这个日志条目，并且也投票给S5。</div>

1. 在leader<sub>U</sub>当选时，已提交条目必须不在leader<sub>U</sub>的日志中（leader从不删除或覆盖条目）。
2. leader<sub>T</sub>在集群的大多数上复制了该条目，而leader<sub>U</sub>从集群的大多数上收到了投票。因此，至少有一台服务器（"投票者"）既接受了leader<sub>T</sub>的条目，又投票给leader<sub>U</sub>，如图9所示。投票者是达成矛盾的关键。
3. 投票者必须在投票给leader<sub>U</sub>之前接受leader<sub>T</sub>的已提交条目；否则它就会拒绝leader<sub>T</sub>的AppendEntries请求（它当前的任期会比T高）。
4. 投票人在投票给leader<sub>U</sub>时仍然存储了该条目，因为每个介入的leader都包含该条目（根据假设），leader从不删除条目，而follower只有在与leader冲突时才会删除条目。
5. 投票人将自己的投票权授予leader<sub>U</sub>，所以leader<sub>U</sub>的日志肯定和投票人的一样是最新的。这导致了两个矛盾中的一个矛盾。
6. 首先，如果投票人和leader<sub>U</sub>共享最后一个日志任期，那么leader<sub>U</sub>的日志至少要和投票人的一样长，所以它的日志包含了投票人日志中的每一个条目。这是一个矛盾，因为投票者包含了已提交的条目，而leader<sub>U</sub>被认为不包含。
7. 否则，leader<sub>U</sub>的最后一个日志任期一定比投票者的大。而且，它比T大，因为投票人的最后一个日志任期至少是T（它包含任期T的提交条目）。创建leader<sub>U</sub>的最后一个日志任期的较早的leader，其日志中一定包含了已提交的条目（根据假设）。那么，根据日志匹配属性，leader<sub>U</sub>的日志也必须包含已提交的条目，这就是一个矛盾。
8. 这就完成了这个矛盾。因此，所有大于T的任期的leader必须包含任期T中所有在任期T中已提交的条目。
9. 日志匹配属性保证了未来的leader也将包含间接提交的条目，如图8(d)中的索引2。

鉴于leader完备性属性，我们可以证明图3中的状态机安全属性，即如果一个服务器在其状态机上应用了一个给定索引的日志条目，那么没有其他服务器会在同一索引上应用一个不同的日志条目。当一个服务器在其状态机上应用一个日志条目时，其日志必须与leader的日志相同，直到该条目，并且该条目必须被提交。现在考虑任何服务器应用一个给定的日志索引的最低任期；日志完整性属性保证所有更高任期的leader将存储相同的日志条目，因此在以后的任期中应用索引的服务器将应用相同的值。因此，状态机安全属性成立。

最后，Raft要求服务器按照日志索引顺序应用条目。结合状态机安全属性，这意味着所有的服务器将以相同的顺序对其状态机应用完全相同的日志条目集。

### 5.5 Follower和candidate宕机

在这之前，我们一直关注的是leader的失败。follower和candidate的崩溃比leader的崩溃要简单得多，而且它们的处理方式都是一样的。如果一个follower或candidate崩溃了，那么未来发送给它的RequestVote和AppendEntries RPC将会失败。Raft通过无限期地重试来处理这些失败；如果崩溃的服务器重新启动，那么RPC将成功完成。如果服务器在完成RPC后但在响应前崩溃，那么它将在重新启动后再次收到相同的RPC。Raft的RPC是幂等的，所以不会有危害。例如，如果一个follower收到一个AppendEntries请求，其中包括已经存在于其日志中的日志条目，它就会在新的请求中忽略这些条目。

### 5.6 计时和可用性

我们对Raft的要求之一是安全不能依赖于计时：系统不能因为某些事件发生得比预期的快或慢而产生错误的结果。然而，可用性（系统及时响应客户的能力）必须不可避免地取决于时间。例如，如果消息交换的时间超过了服务器崩溃之间的典型时间，那么candidate就不会保持足够长的时间来赢得选举；没有一个稳定的leader，Raft就无法取得进展。

leader选举是Raft中计时最关键的部分。只要系统满足以下时间要求，Raft就能选出并维持一个稳定的leader：

`broadcastTime ≪ electionTimeout ≪ MTBF`

在这个不等式中，broadcastTime是一台服务器向集群中的每台服务器并行发送RPC并接收其响应所需的平均时间；electionTimeout是第5.2节中描述的选举超时；MTBF是单台服务器的平均故障间隔时间。broadcast time应该比election timeout少一个数量级，这样leader就可以可靠地发送心跳信息，以防止follower开始选举；考虑到选举超时使用的随机方法，这种不平等也使得分裂投票不太可能发生。选举超时应该比MTBF小几个数量级，这样系统才能稳步前进。当leader崩溃时，系统将在大约选举超时的时间内不可用；我们希望这只占总体时间的一小部分。

broadcastTime和MTBF是底层系统的属性，而elelctionTimeout是我们必选的。 Raft的RPC通常需要接收者将信息持久化，因此broadcastTime可能从0.5毫秒到20毫秒不等，具体取决于存储技术。因此，electionTimeout可能在10毫秒至500毫秒之间。典型服务器的MTBF一般是几个月甚至更长，很容易就满足了计时的需求。

## 六、  集群成员变更

到目前为止，我们一直假设集群配置（参与共识算法的服务器集合）是固定的。在实践中，偶尔有必要改变配置，例如在服务器故障时更换服务器或改变复制的程度。虽然这可以通过关闭整个集群，更新配置文件，然后重新启动集群来完成，但这将使集群在改变期间不可用。此外，如果有任何手动步骤，就有可能出现操作错误。为了避免这些问题，我们决定将配置变更自动化，并将其纳入Raft共识算法中。

![](figure10.JPG "图10")

<div style="text-align: center;"><b>图10</b>：直接从一个配置切换到另一个配置是不安全的，因为不同的服务器会在不同时间切换。在这个例子中，集群从三个服务器增长到五个。不幸的是，有一个时间点，两个不同的leader可以在同一任期内当选，一个是旧配置的多数（C<sub>old</sub>），另一个是新配置的多数（C<sub>new</sub>）。</div>

为了使配置变更机制安全，在过渡期间必须没有任何一点可以让两个leader当选为同一任期。不幸的是，任何服务器直接从旧配置切换到新配置的方法都是不安全的。不可能一次性原子化地切换所有的服务器，所以集群在过渡期间有可能分裂成两个独立的多数（见图10）。

为了确保安全，配置变化必须使用两阶段的方法。实现这两个阶段的方法有很多种。例如，一些系统 (如[22]）使用第一阶段禁用旧的配置，使其不能处理客户端请求；然后第二阶段启用新的配置。在Raft中，集群首先切换到一个过渡性配置，我们称之为联合共识；一旦联合共识被提交，系统就会过渡到新的配置。联合共识结合了新旧两种配置：

- 日志条目被复制到两个配置中的所有服务器。
- 任何一个配置的服务器都可以作为leader。
- 达成协议（选举和进入承诺）需要新旧配置分别获得多数票。

联合共识允许单个服务器在不同时间在配置之间转换，而不影响安全。此外，联合共识允许集群在整个配置变化过程中继续为客户请求提供服务。

![](figure11.JPG "图11")

<div style="text-align: center;"><b>图11</b>：配置变更的时间线。虚线表示已经创建但未提交的配置条目，实线表示最新提交的配置条目。leader首先在其日志中创建 C<sub>old,new</sub> 配置条目，并将其提交给 C<sub>old,new</sub>（C<sub>old</sub> 的大多数和 C<sub>new</sub> 的大多数）。然后，它创建了C<sub>new</sub>条目，并将其提交给多数的C<sub>new</sub>。在这个时间点上，C<sub>old</sub>和C<sub>new</sub>都不能独立做出决定。</div>

集群配置使用复制日志中的特殊条目进行存储和通信；图11说明了配置变化过程。当leader收到将配置从C<sub>old</sub>改成C<sub>new</sub>的请求时，它将联合共识的配置（图中的C<sub>old,new</sub>）存储为一个日志条目，并使用前面描述的机制复制该条目。一旦某个服务器将新的配置条目添加到其日志中，它就会将该配置用于所有未来的决策（一个服务器总是使用其日志中的最新配置，无论该条目是否被提交）。这意味着leader将使用 C<sub>old,new</sub> 的规则来决定 C<sub>old,new</sub> 的日志条目何时被提交。如果leader崩溃了，一个新的leader可能会在C<sub>old</sub>或C<sub>old,new</sub>下被选择，这取决于获胜的候选人是否已经收到了C<sub>old,new</sub>。在任何情况下，C<sub>new</sub>都不能在这段时间内做出单边决定。

一旦C<sub>old,new</sub>被承诺，无论是C<sub>old</sub>还是C<sub>new</sub>都不能在未经对方批准的情况下做出决定，而且leader完整性属性确保只有拥有C<sub>old,new</sub>日志记录的服务器才能被选为leader。现在，leader创建描述C<sub>new</sub>的日志条目并将其复制到集群中是安全的。同样，这个配置一旦被看到，就会在每个服务器上生效。当新的配置在C<sub>new</sub>的规则下被提交后，旧的配置就不重要了，不在新配置中的服务器可以被关闭。如图11所示，没有任何时候C<sub>old</sub>和C<sub>new</sub>可以同时做出单边决定；这保证了安全。

对于重新配置，还有三个问题需要解决。第一个问题是，新服务器最初可能不会存储任何日志条目。如果它们在这种状态下被添加到集群中，可能需要相当长的时间才能赶上，在此期间，可能无法提交新的日志条目。为了避免可用性差距，Raft在配置改变之前引入了一个额外的阶段，在这个阶段，新的服务器作为非投票成员加入集群（leader将日志条目复制给他们，但他们不被考虑为多数）。一旦新的服务器赶上了集群的其他部分、 重新配置就可以按上述方法进行。

第二个问题是，集群leader可能不是新配置的一部分。在这种情况下，一旦它提交了C<sub>new</sub>日志条目，leader就会下台（返回到follower状态）。这意味着会有一段时间（在它提交C<sub>new</sub>的时候），leader在管理一个不包括自己的集群；它复制日志条目，但不把自己算在多数中。leader过渡发生在C<sub>new</sub>被提交的时候，因为这是新配置可以独立运行的第一个点（它将始终有可能从C<sub>new</sub>中选择一个leader）。在这之前，可能只有C<sub>old</sub>的一个服务器可以被选为leader。

第三个问题是，被移除的服务器（那些不在C<sub>new</sub>中的服务器）会扰乱集群。这些服务器不会收到心跳，所以它们会超时并开始新的选举。然后他们会发送带有新任期编号的RequestVote RPCs，这将导致当前的leader恢复到follower状态。一个新的leader最终将被选出，但被移除的服务器将再次超时，这个过程将重复，导致可用性差。

为了防止这个问题，当服务器认为存在一个当前的leader时，他们会忽略RequestVote RPC。具体来说，如果一个服务器在听到当前leader的最小选举超时内收到RequestVote RPC，它不会更新其任期或授予其投票。这并不影响正常的选举，每个服务器在开始选举之前至少要等待一个最小的选举超时。然而，这有助于避免被移除的服务器的干扰：如果一个leader能够得到其集群的心跳，那么它就不会被较大的任期数字所废黜。

## 七、  日志压缩

Raft的日志在正常运行期间不断增长，以纳入更多的客户端请求，但在一个实际的系统中，它不能无限制地增长。随着日志的增长，它占据了更多的空间，需要更多的时间来重放。如果没有某种机制来丢弃积累在日志中的过时信息，这最终会导致可用性问题。

快照是最简单的压缩方法。在快照中，整个当前的系统状态被写入稳定存储的快照中，然后丢弃截至该点的整个日志。快照在Chubby和ZooKeeper中使用，本节的其余部分描述了Raft中的快照。

增量压缩的方法，如日志清理[36]和日志结构化合并树[30, 5]，也是可能的。这些方法一次性对部分数据进行操作，因此它们将压缩的负载更均匀地分散在时间上。它们首先选择一个已经积累了许多删除和覆盖对象的数据区域，然后将该区域的活对象重写得更紧凑，并释放该区域。与快照相比，这需要大量的额外机制和复杂性，因为快照总是对整个数据集进行操作，从而简化了问题。虽然日志清理需要对Raft进行修改，但状态机可以使用与快照相同的接口实现LSM树。

![](figure12.JPG "图12")

<div style="text-align: center;"><b>图12</b>：一个服务器用一个新的快照替换其日志中已提交的条目（索引1到5），该快照只存储当前的状态（本例中的变量x和y）。快照最后包含的索引和术语用于定位快照在日志第6条之前的位置。</div>

图12显示了Raft中快照的基本思路。每台服务器独立进行快照，只覆盖其日志中已提交的条目。大部分的工作包括状态机将其当前状态写入快照。Raft还在快照中包含了少量的元数据：最后包含的索引是快照所取代的日志中最后一个条目的索引（状态机应用的最后一个条目），最后包含的术语是这个条目的术语。这些被保留下来是为了支持快照后的第一个日志条目的AppendEntries一致性检查，因为该条目需要一个先前的日志索引和术语。为了实现集群成员的变化（第6节），快照还包括日志中的最新配置，即最后包含的索引。一旦服务器完成写入快照，它可以删除所有的日志条目，直到最后包含的索引，以及任何先前的快照。

尽管服务器通常是独立进行快照的，但leader偶尔也必须向落后的follower发送快照。这种情况发生在leader已经丢弃了它需要发送给follower的下一个日志条目。幸运的是，这种情况在正常操作中不太可能发生：一个跟上leader的follower已经有了这个条目。然而，一个特别慢的follower或一个新加入集群的服务器（第6节）就不会有这样的情况。让这样的follower跟上的方法是，leader通过网络向它发送一个快照。

![](figure13.JPG "图13")

<div style="text-align: center;"><b>图13</b>：InstallSnapshot RPC的摘要。快照被分割成若干chunk进行传输；通过给follower提供各个chunk，leader以此表明其存活状态，因此follower可以重置其选举定时器。</div>

leader使用一个叫做InstallSnapshot的新RPC来向落后于它的follower发送快照；见图13。当follower收到这个RPC的快照时，它必须决定如何处理其现有的日志条目。通常情况下，快照将包含新的信息，而不是在接收者的日志中。在这种情况下，follower会丢弃它的整个日志；它都被快照取代了，而且可能有与快照冲突的未提交的条目。相反，如果follower收到描述其日志前缀的快照（由于重传或错误），那么快照覆盖的日志条目被删除，但快照之后的条目仍然有效，必须保留。

这种快照方法背离了Raft的强leader原则，因为follower可以在leader不知情的情况下进行快照。然而，我们认为这种背离是合理的。虽然有一个leader有助于在达成共识时避免冲突的决定，但在快照时已经达成了共识，所以没有决定冲突。数据仍然只从leader流向follower，只是follower现在可以重新组织他们的数据。

我们考虑了另一种基于leader的方法，即只有leader会创建一个快照，然后它将这个快照发送给它的每个follower。然而，这有两个缺点。首先，向每个follower发送快照会浪费网络带宽，并减缓快照过程。每个follower都已经拥有产生自己快照所需的信息，而且对于服务器来说，从其本地状态产生一个快照通常要比在网络上发送和接收一个快照便宜很多。第二，leader的实现将更加复杂。例如，leader需要在向follower发送快照的同时，向他们回复新的日志条目，这样就不会阻碍新的客户请求。

还有两个影响快照性能的问题。首先，服务器必须决定何时进行快照。如果服务器快照的频率过高，就会浪费磁盘带宽和能源；如果快照的频率过低，就会有耗尽其存储容量的风险，而且会增加重启时重放日志的时间。一个简单的策略是，当日志达到一个固定的字节大小时进行快照。如果这个大小被设定为明显大于快照的预期大小，那么用于快照的磁盘带宽就会很小。

第二个性能问题是，写一个快照可能需要相当长的时间，我们不希望因此而延误正常的操作。解决方案是使用写时复制技术，这样就可以接受新的更新而不影响正在写入的快照。例如，用功能数据结构构建的状态机自然支持这一点。另外，可以使用操作系统的写时拷贝支持（例如Linux上的fork）来创建整个状态机的内存快照（我们的实现采用了这种方法）。

## 八、  客户端交互

本节介绍了客户端如何与Raft交互，包括客户端如何找到集群leader以及Raft如何支持可线性化语义[10]。这些问题适用于所有基于共识的系统，而且Raft的解决方案与其他系统相似。

Raft的客户端将他们所有的请求发送给leader。当客户端第一次启动时，它连接到一个随机选择的服务器。如果客户端的第一选择不是leader，该服务器将拒绝客户端的请求，并提供它最近听到的leader的信息（AppendEntries请求包括leader的网络地址）。如果leader崩溃了，客户端的请求就会超时；然后客户端会在随机选择的服务器上再次尝试。

我们对Raft的目标是实现可线性化的语义（每个操作看起来都是瞬时执行的，在其调用和响应之间的某个点上正好执行一次）。然而，正如目前所描述的那样，Raft可以多次执行一个命令：例如，如果leader在提交日志条目后但在响应客户端之前崩溃，客户端将用一个新的leader重试该命令，导致它被第二次执行。解决方案是让客户端为每个命令分配唯一的序列号。然后，状态机跟踪为每个客户端处理的最新序列号，以及相关的响应。如果它收到一个序列号已经被执行的命令，它会立即响应，而不重新执行该请求。

只读操作可以在不向日志中写入任何内容的情况下进行处理。然而，如果没有额外的措施，这将有返回陈旧数据的风险，因为响应请求的leader可能已经被它不知道的较新的leader所取代了。可线性化的读取必须不返回陈旧的数据，Raft需要两个额外的预防措施来保证这一点而不使用日志。首先，leader必须拥有关于哪些条目被提交的最新信息。leader完整性属性保证leader拥有所有已提交的条目，但在其任期开始时，它可能不知道哪些是。为了找到答案，它需要从其任期内提交一个条目。Raft通过让每个leader在其任期开始时向日志提交一个空白的无操作条目来处理这个问题。第二，leader在处理只读请求之前，必须检查它是否已经被废黜（如果最近的leader已经当选，那么它的信息可能是过时的）。Raft通过让leader在响应只读请求之前与集群中的大多数人交换心跳信息来处理这个问题。另外，leader可以依靠心跳机制来提供一种租约形式[9]，但这要依靠时间来保证安全（它假定了有界的时钟偏移）。

## 九、  实现和评估

我们将Raft实现为复制状态机的一部分，该状态机存储RAMCloud[33]的配置信息并协助RAMCloud协调器的故障转移。Raft的实现包含大约2000行的C++代码，不包括测试、注释或空行。源代码可以免费获得[23]。根据本文的草稿，还有大约25个独立的第三方开源实现[34]，处于不同的开发阶段。另外，各种公司也在部署基于Raft的系统[34]。

本节的其余部分使用三个标准对Raft进行评估：可理解性、正确性和性能。

### 9.1 可理解性

为了衡量Raft相对于Paxos的可理解性，我们对斯坦福大学高级操作系统课程和加州大学伯克利分校分布式计算课程的高年级本科生和研究生进行了一项实验性研究。我们录制了Raft和Paxos的视频讲座，并制作了相应的测验。Raft的讲座涵盖了本文的内容，除了日志压缩；Paxos的讲座涵盖了足够的材料来创建一个等效的复制状态机，包括single-decree Paxos、multi-decree Paxos、重新配置，以及实践中需要的一些优化（如leader选举）。测验测试了对算法的基本理解，还要求学生对角落的情况进行推理。每个学生都看了一个视频，做了相应的测验，看了第二个视频，做了第二个测验。大约一半的参与者先做Paxos部分，另一半先做Raft部分，以便考虑到个人表现的差异和从研究的第一部分获得的经验。我们比较了参与者在每次测验中的得分，以确定参与者是否对Raft表现出更好的理解。

![](table01.JPG "Paxos的偏见")

<div style="text-align: center;"><b>表1</b>：可能存在针对Paxos的偏见的担忧，采取了哪些措施来对抗这种偏见，并提供了哪些额外资料。</div>

我们试图使Paxos和Raft之间的比较尽可能地公平。实验在两个方面对Paxos有利： 43名参与者中的15名报告说以前对Paxos有一些经验，而且Paxos的视频比Raft的视频长14%。正如表1所总结的，我们采取了措施来减少潜在的偏见来源。我们所有的材料都可供查阅[28, 31]。

![](figure14.JPG "测验结果散点图")

<div style="text-align: center;"><b>图14</b>：比较43名参与者在Raft和Paxos测验中的表现的散点图。对角线以上的点（33）代表在Raft上得分较高的参与者。</div>

平均而言，参与者在Raft测验中的得分比Paxos测验高4.9分（在可能的60分中，Raft的平均得分是25.7分，Paxos的平均得分是20.8分）；图14显示了他们的个人得分。成对的t检验表明，在95%的置信度下，Raft分数的真实分布比Paxos分数的真实分布的平均值至少要大2.5分。

我们还创建了一个线性回归模型，根据三个因素预测新学生的测验分数：他们参加了哪次测验，他们之前的Paxos经验程度，以及他们学习算法的顺序。该模型预测，测验的选择会产生有利于Raft的12.5分的差异。这明显高于观察到的4.9分的差异，因为许多实际的学生之前有Paxos的经验，这对Paxos有很大的帮助，而对Raft的帮助则略小。奇怪的是，模型还预测已经参加过Paxos测验的人在Raft上的得分要低6.3分；虽然我们不知道为什么，但这似乎在统计上是有意义的。

![](figure15.JPG "算法可理解性调查")

<div style="text-align: center;"><b>图15</b>：使用5分制，参与者被问及（左）他们认为哪种算法更容易在一个有效的、正确的和高效的系统中实现，以及（右）哪种算法更容易向一个CS研究生解释。</div>

我们还在测验后对参与者进行了调查，看看他们认为哪种算法更容易实现或解释；这些结果显示在图15。绝大多数参与者报告说Raft更容易实施和解释（每个问题41人中有33人）。然而，这些自我报告的感受可能不如参与者的测验分数可靠，而且参与者可能因为知道我们的假设--Raft更容易理解而产生偏差。

关于Raft用户研究的详细讨论，可参见[31]。

### 9.2 正确性

我们已经为第5节中描述的共识机制开发了一个正式的规范和安全证明。形式规范[31]使用TLA+规范语言[17]使图2中总结的信息完全精确。它长约400行，作为证明的主题。它本身对实现Raft的人来说也是有用的。我们已经使用TLA证明系统[7]机械地证明了对数完整性属性。然而，这个证明依赖于没有经过机械检查的不变量（例如，我们没有证明规范的类型安全）。此外，我们已经写了一个关于状态机安全属性的非正式证明[31]，它是完整的（它仅仅依赖于规范）和相对精确的（它大约有3500字）。

### 9.3 性能

Raft的性能与其他共识算法（如Paxos）相似。对于性能来说，最重要的情况是当一个已建立的leader在复制新的日志条目时。Raft使用最少的信息数量（从leader到一半集群的单次往返）实现了这一点。进一步提高Raft的性能也是可能的。例如，它很容易支持批处理和管道化请求，以获得更高的吞吐量和更低的延迟。文献中已经为其他算法提出了各种优化方案；其中许多方案可以应用于Raft，但我们将此留给未来的工作。

我们使用我们的Raft实现来衡量Raft的leader选举算法的性能，并回答两个问题。首先，选举过程是否快速收敛？第二，leader崩溃后能实现的最小停机时间是多少？

![](figure16.JPG "检测和替换一个崩溃的leader的时间")

<div style="text-align: center;"><b>图16</b>：检测和替换一个崩溃的leader的时间。上图改变了选举超时的随机性，下图是对最小选举超时的刻度。每条线代表1000次试验（"150-150ms "的100次试验除外），并对应于选举超时的特定选择；例如，"150-155ms "意味着选举超时是在150ms和155ms之间随机和统一选择的。测试是在一个由五个服务器组成的集群上进行的，广播时间大约为15ms。九个服务器的集群的结果是相似的。</div>

为了测试leader选举性能，我们反复让五个服务器集群的leader崩溃，并计时检测崩溃和选举新leader所需的时间（见图16）。为了产生一个最坏的情况，每次试验中的服务器都有不同的日志长度，所以一些候选人没有资格成为leader。此外，为了鼓励分裂投票，我们的测试脚本在终止其进程之前触发了leader的心跳RPC的同步广播（这接近于leader在崩溃前复制新的日志条目的行为）。领头羊在其心跳间隔内被均匀地随机崩溃，这个间隔是所有测试中最小选举超时的一半。因此，最小的可能停机时间大约是最小选举超时的一半。

图16中的顶部图表显示，选举超时中的少量随机性足以避免选举中的分裂票。在没有随机性的情况下，在我们的测试中，由于许多分裂的选票，领导人选举的时间一直超过10秒。仅仅增加5毫秒的随机性就有很大的帮助，导致中位数停机时间为287毫秒。使用更多的随机性可以改善最坏情况下的行为：使用50ms的随机性，最坏情况下的完成时间（超过1000次试验）是513ms。

图16中的底图显示，通过减少选举超时，可以减少停机时间。在选举超时12-24ms的情况下，平均只需要35ms就能选出一个leader（最长的一次试验用了152ms）。然而，将超时时间降低到超过这个点，就违反了Raft的时间要求：leader很难在其他服务器开始新的选举之前广播心跳。这可能会导致不必要的leader更换，降低系统的整体可用性。我们建议使用保守的选举超时，如150-300ms；这样的超时不太可能导致不必要的leader变更，并且仍然会提供良好的可用性。

## 十、  相关工作

有许多与共识算法有关的出版物，其中许多属于以下类别之一：

- Lamport对Paxos的原始描述[15]，以及试图更清楚地解释它[16, 20, 21]。
- 对Paxos的阐述，填补了缺失的细节，修改了算法，为实现提供了更好的基础[26, 39, 13]。
- 实现共识算法的系统，如Chubby [2, 4], ZooKeeper [11, 12], 和Spanner [6]。Chubby和Spanner的算法还没有被详细公布，尽管两者都声称是基于Paxos。ZooKeeper的算法已经公布了更多的细节，但它与Paxos有很大的不同。
- 可以应用于Paxos的性能优化[18, 19, 3, 25, 1, 27]。
- Oki和Liskov的Viewstamped Replication (VR)，这是一种与Paxos差不多同时开发的共识的替代方法。最初的描述[29]与分布式交易的协议交织在一起，但在最近的更新中[22]，核心共识协议被分离出来。VR使用了一种基于leader的方法，与Raft有许多相似之处。

Raft和Paxos的最大区别是Raft的强leader模式：Raft将leader选举作为共识协议的一个重要部分，它将尽可能多的功能集中在leader身上。这种方法导致了更简单的算法，更容易理解。例如，在Paxos中，leader选举与基本的共识协议是正交的：它只是作为一种性能优化，并不是实现共识的必要条件。然而，这导致了额外的机制： Paxos包括一个两阶段的基本共识协议和一个单独的leader选举机制。相比之下，Raft将leader选举直接纳入共识算法，并将其作为共识的两个阶段中的第一阶段。这导致了比Paxos更少的机制。

与Raft一样，VR和ZooKeeper也是基于leader的，因此与Paxos相比，Raft有许多优势。然而，Raft的机制比VR或ZooKeeper少，因为它将非leader的功能降到最低。例如，Raft中的日志条目只向一个方向流动：从AppendEntries RPCs的leader向外流动。在VR中日志条目是双向流动的（leader可以在选举过程中接收日志条目）；这导致了额外的机制和复杂性。ZooKeeper公布的描述也是将日志条目同时传输给leader和从leader那里传输，但其实现显然更像Raft[35]。

Raft的消息类型比我们所知的任何其他基于共识的日志复制算法都少。例如，我们计算了VR和ZooKeeper用于基本共识和成员变更的消息类型（不包括日志压缩和客户端交互，因为这些几乎是独立于算法的）。VR和ZooKeeper各自定义了10种不同的消息类型，而Raft只有4种消息类型（两个RPC请求和它们的响应）。Raft的消息比其他算法的消息更密集一些，但它们总体上更简单。此外，VR和ZooKeeper的描述是在leader变更期间传输整个日志；为了优化这些机制，将需要额外的消息类型，以便它们能够实用。

Raft的强leader方法简化了算法，但它排除了一些性能优化。例如，Egalitarian Paxos（EPaxos）在某些条件下可以通过无leader的方法获得更高的性能[27]。EPaxos利用了状态机命令的交换性。只要其他同时提出的命令与之换算，任何服务器都可以只用一轮通信来提交一个命令。然而，如果同时提出的命令不能相互交换，EPaxos需要额外的一轮通信。由于任何服务器都可以提交命令，EPaxos可以很好地平衡服务器之间的负载，并能够在广域网环境中实现比Raft更低的延迟。然而，它给Paxos增加了很大的复杂性。

其他工作中已经提出或实施了几种不同的集群成员变化方法，包括Lamport的原始提议[15]、VR[22]和SMART[24]。我们为Raft选择了联合共识的方法，因为它利用了共识协议的其他部分，所以成员资格变更所需的额外机制非常少。Lamport的基于α的方法不是Raft的选择，因为它假设没有leader也能达成共识。与VR和SMART相比，Raft的重新配置算法的优点是，成员变化可以在不限制正常请求的处理的情况下发生；相反，VR在配置变化期间停止所有的正常处理，而SMART对未处理的请求数量施加了类似α的限制。Raft的方法也比VR或SMART增加了更少的机制。

## 十一、  结论

算法的设计通常以正确性、效率和/或简洁性为主要目标。尽管这些都是有价值的目标，但我们认为可理解性也同样重要。在开发者将算法转化为实际的实现之前，其他的目标都无法实现，而实际的实现将不可避免地偏离和扩展公布的形式。除非开发者对算法有深刻的理解，并能对其产生直觉，否则他们将很难在其实现中保留其理想的属性。

在本文中，我们讨论了分布式共识的问题，其中一个被广泛接受但难以理解的算法--Paxos，多年来一直在挑战学生和开发者。我们开发了一种新的算法--Raft，我们已经证明它比Paxos更容易理解。我们还认为Raft为系统建设提供了一个更好的基础。将可理解性作为主要的设计目标，改变了我们设计Raft的方式；随着设计的进展，我们发现自己反复使用了一些技术，例如分解问题和简化状态空间。这些技术不仅提高了Raft的可理解性，而且也使我们更容易相信它的正确性。

## References

[1] BOLOSKY, W. J., BRADSHAW, D., HAAGENS, R. B., KUSTERS, N. P., AND LI, P. Paxos replicated state machines as the basis of a high-performance data store. In Proc. NSDI’11, USENIX Conference on Networked Systems Design and Implementation (2011), USENIX, pp. 141–154.

[2] BURROWS, M. The Chubby lock service for looselycoupled distributed systems. In Proc. OSDI’06, Symposium on Operating Systems Design and Implementation(2006), USENIX, pp. 335–350.

[3] CAMARGOS, L. J., SCHMIDT, R. M., AND PEDONE, F. Multicoordinated Paxos. In Proc. PODC’07, ACM Symposium on Principles of Distributed Computing (2007), ACM, pp. 316–317.

[4] CHANDRA, T. D., GRIESEMER, R., AND REDSTONE, J. Paxos made live: an engineering perspective. In Proc. PODC’07, ACM Symposium on Principles of Distributed Computing (2007), ACM, pp. 398–407.

[5] CHANG, F., DEAN, J., GHEMAWAT, S., HSIEH, W. C., WALLACH, D. A., BURROWS, M., CHANDRA, T., FIKES, A., AND GRUBER, R. E. Bigtable: a distributed storage system for structured data. In Proc. OSDI’06, USENIX Symposium on Operating Systems Design and Implementation (2006), USENIX, pp. 205–218.

[6] CORBETT, J. C., DEAN, J., EPSTEIN, M., FIKES, A., FROST, C., FURMAN, J. J., GHEMAWAT, S., GUBAREV, A., HEISER, C., HOCHSCHILD, P., HSIEH, W., KANTHAK, S., KOGAN, E., LI, H., LLOYD, A., MELNIK, S., MWAURA, D., NAGLE, D., QUINLAN, S., RAO, R., ROLIG, L., SAITO, Y., SZYMANIAK, M., TAYLOR, C., WANG, R., AND WOODFORD, D. Spanner: Google’s globally-distributed database. In Proc. OSDI’12, USENIX Conference on Operating Systems Design and Implementation (2012), USENIX, pp. 251–264.

[7] COUSINEAU, D., DOLIGEZ, D., LAMPORT, L., MERZ, S., RICKETTS, D., AND VANZETTO, H. TLA+ proofs. In Proc. FM’12, Symposium on Formal Methods (2012), D. Giannakopoulou and D. M´ery, Eds., vol. 7436 of Lecture Notes in Computer Science, Springer, pp. 147–154.

[8] GHEMAWAT, S., GOBIOFF, H., AND LEUNG, S.-T. The Google file system. In Proc. SOSP’03, ACM Symposium on Operating Systems Principles (2003), ACM, pp. 29–43.

[9] GRAY, C., AND CHERITON, D. Leases: An efficient faulttolerant mechanism for distributed file cache consistency. In Proceedings of the 12th ACM Ssymposium on Operating Systems Principles (1989), pp. 202–210.

[10] HERLIHY, M. P., AND WING, J. M. Linearizability: a correctness condition for concurrent objects. ACM Transactions on Programming Languages and Systems 12 (July 1990), 463–492.

[11] HUNT, P., KONAR, M., JUNQUEIRA, F. P., AND REED, B. ZooKeeper: wait-free coordination for internet-scale systems. In Proc ATC’10, USENIX Annual Technical Conference (2010), USENIX, pp. 145–158.

[12] JUNQUEIRA, F. P., REED, B. C., AND SERAFINI, M. Zab: High-performance broadcast for primary-backup systems. In Proc. DSN’11, IEEE/IFIP Int’l Conf. on Dependable Systems & Networks (2011), IEEE Computer Society, pp. 245–256.

[13] KIRSCH, J., AND AMIR, Y. Paxos for system builders. Tech. Rep. CNDS-2008-2, Johns Hopkins University, 2008.

[14] LAMPORT, L. Time, clocks, and the ordering of events in a distributed system. Commununications of the ACM 21, 7 (July 1978), 558–565.

[15] LAMPORT, L. The part-time parliament. ACM Transactions on Computer Systems 16, 2 (May 1998), 133–169.

[16] LAMPORT, L. Paxos made simple. ACM SIGACT News 32, 4 (Dec. 2001), 18–25.

[17] LAMPORT, L. Specifying Systems, The TLA+ Language and Tools for Hardware and Software Engineers. AddisonWesley, 2002.

[18] LAMPORT, L. Generalized consensus and Paxos. Tech. Rep. MSR-TR-2005-33, Microsoft Research, 2005.

[19] LAMPORT, L. Fast paxos. Distributed Computing 19, 2 (2006), 79–103.

[20] LAMPSON, B. W. How to build a highly available system using consensus. In Distributed Algorithms, O. Baboaglu and K. Marzullo, Eds. Springer-Verlag, 1996, pp. 1–17.

[21] LAMPSON, B. W. The ABCD’s of Paxos. In Proc. PODC’01, ACM Symposium on Principles of Distributed Computing (2001), ACM, pp. 13–13.

[22] LISKOV, B., AND COWLING, J. Viewstamped replication revisited. Tech. Rep. MIT-CSAIL-TR-2012-021, MIT, July 2012.

[23] LogCabin source code. http://github.com/logcabin/logcabin.

[24] LORCH, J. R., ADYA, A., BOLOSKY, W. J., CHAIKEN, R., DOUCEUR, J. R., AND HOWELL, J. The SMART way to migrate replicated stateful services. In Proc. EuroSys’06, ACM SIGOPS/EuroSys European Conference on Computer Systems (2006), ACM, pp. 103–115.

[25] MAO, Y., JUNQUEIRA, F. P., AND MARZULLO, K.Mencius: building efficient replicated state machines for WANs. In Proc. OSDI’08, USENIX Conference on Operating Systems Design and Implementation (2008), USENIX, pp. 369–384.

[26] MAZIERES, D. Paxos made practical. http://www.scs.stanford.edu/˜dm/home/papers/paxos.pdf, Jan. 2007.

[27] MORARU, I., ANDERSEN, D. G., AND KAMINSKY, M.There is more consensus in egalitarian parliaments. In Proc. SOSP’13, ACM Symposium on Operating System Principles (2013), ACM.

[28] Raft user study. http://ramcloud.stanford.edu/˜ongaro/userstudy/.

[29] OKI, B. M., AND LISKOV, B. H. Viewstamped replication: A new primary copy method to support highly-available distributed systems. In Proc. PODC’88, ACM Symposium on Principles of Distributed Computing (1988), ACM, pp. 8–17.

[30] O’NEIL, P., CHENG, E., GAWLICK, D., AND ONEIL, E. The log-structured merge-tree (LSM-tree). Acta Informatica 33, 4 (1996), 351–385.

[31] ONGARO, D. Consensus: Bridging Theory and Practice.PhD thesis, Stanford University, 2014 (work in progress).http://ramcloud.stanford.edu/˜ongaro/thesis.pdf.

[32] ONGARO, D., AND OUSTERHOUT, J. In search of an understandable consensus algorithm. In Proc ATC’14,USENIX Annual Technical Conference (2014), USENIX.

[33] OUSTERHOUT, J., AGRAWAL, P., ERICKSON, D., KOZYRAKIS, C., LEVERICH, J., MAZIERES ` , D., MITRA, S., NARAYANAN, A., ONGARO, D., PARULKAR, G., ROSENBLUM, M., RUMBLE, S. M., STRATMANN, E., AND STUTSMAN, R. The case for RAMCloud. Communications of the ACM 54 (July 2011), 121–130.

[34] Raft consensus algorithm website.http://raftconsensus.github.io.

[35] REED, B. Personal communications, May 17, 2013.

[36] ROSENBLUM, M., AND OUSTERHOUT, J. K. The design and implementation of a log-structured file system. ACMTrans. Comput. Syst. 10 (February 1992), 26–52.

[37] SCHNEIDER, F. B. Implementing fault-tolerant services using the state machine approach: a tutorial. ACM Computing Surveys 22, 4 (Dec. 1990), 299–319.

[38] SHVACHKO, K., KUANG, H., RADIA, S., AND CHANSLER, R. The Hadoop distributed file system. In Proc. MSST’10, Symposium on Mass Storage Systems and Technologies (2010), IEEE Computer Society, pp. 1–10.

[39] VAN RENESSE, R. Paxos made moderately complex.Tech. rep., Cornell University, 2012.
