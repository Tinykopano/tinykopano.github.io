---
title: A simple totally ordered broadcast protocol
date: 2023-02-01 14:22:00 +0800
categories: [Course,Paper]
tags: [distributed system,consensus algorithm,Zab]
---

# 一种简单的全序列的广播协议

## 摘要

这是一个关于ZooKeeper使用的全序列的广播协议的简短概述，称为Zab。它在概念上很容易理解，容易实现，而且性能高。在本文中，我们介绍了ZooKeeper对Zab的要求，展示了该协议的使用方法，并概述了该协议的工作方式。

<!-- more -->

## 一、  引言

在雅虎，我们开发了一个高性能的高可用协调服务，称为ZooKeeper[9]，允许大规模的应用程序执行协调任务，如领导者选举、状态传播和会合。这个服务实现了一个数据节点的分层空间，称为znodes，客户使用它来实现他们的协调任务。我们发现这项服务很灵活，其性能很容易满足我们在雅虎的网络规模、关键任务应用程序的生产需求。ZooKeeper放弃了锁，而是实现了无等待的共享数据对象，对这些对象的操作顺序有很强的保证。客户端库利用这些保证来实现他们的协调任务。一般来说，ZooKeeper的主要前提之一是更新顺序对应用程序来说比其他典型的协调技术（如阻塞）更重要。

嵌入ZooKeeper的是一个全序列的广播协议： Zab。在实现我们的客户端保证时，有序的广播是至关重要的；在每个ZooKeeper服务器上维护ZooKeeper状态的副本也是必要的。这些副本使用我们的全序列的广播协议保持一致，如复制的状态机[13]。本文主要介绍ZooKeeper对这个广播协议的要求以及对其实现的概述。

一个ZooKeeper服务通常由三到七台机器组成。我们的实现支持更多的机器，但三到七台机器提供了足够的性能和恢复力。客户端连接到任何一台提供服务的机器上，并始终对ZooKeeper的状态有一个一致的看法。该服务最多可以容忍f个崩溃故障，它至少需要2f+1个服务器。

应用程序广泛地使用ZooKeeper，并且有几十到几千个客户端同时访问它，所以我们需要高吞吐量。我们将ZooKeeper设计为读写操作比例高于2:1的工作负载；但是，我们发现ZooKeeper的高写吞吐量也允许它用于一些写为主的工作负载。ZooKeeper通过为每个服务器上的ZooKeeper状态的本地副本的读取提供服务来提供高的读取吞吐量。因此，容错性和读取吞吐量都可以通过增加服务器来扩展服务。写入吞吐量不会因为增加服务器而扩大；相反，它受限于广播协议的吞吐量，因此我们需要一个高吞吐量的广播协议。

![](figure01.jpg "Zookeeper服务组件")

<div style="text-align: center;"><b>图1</b>：Zookeeper服务的逻辑组件。</div>

图1显示了ZooKeeper服务的逻辑构成。读取请求是由包含ZooKeeper状态的本地数据库提供服务。写入请求从ZooKeeper请求转化为幂等的事务，并在生成响应之前通过Zab发送。许多ZooKeeper的写请求在本质上是有条件的：一个znode只有在没有任何子节点的情况下才能被删除；一个znode在创建时可以附加一个名字和一个序列号；对数据的改变只有在它处于预期的版本时才会被应用。即使是无条件的写请求，也会以不等价的方式修改元数据，如版本号。

通过将所有的更新通过一台服务器（被称为领导者）发送，我们将非幂等的请求转化为幂等的事务。在本文中，我们使用术语事务来表示幂等版本的请求。领导可以进行这种转化，因为它对复制的数据库的未来状态有一个完美的视角，可以计算出新记录的状态。幂等事务是这个新状态的记录。ZooKeeper在很多方面都利用了幂等事务，这不在本文的讨论范围之内，但是我们的事务的幂等性质也允许我们在恢复期间放松我们的广播协议的排序要求。

## 二、  要求

我们假设有一组进程既实现了原子广播协议又使用了它。为了保证在ZooKeeper中正确地转化为幂等请求，每次都必须有一个领导者，所以我们通过协议的实现强制要求有一个这样的进程。我们在介绍协议的更多细节时，会更多地讨论它。

ZooKeeper对广播协议提出了以下要求：

**可靠的传递** 如果一个消息m被一个服务器传递，那么它最终会被所有正确的服务器传递。

**全序列** 如果消息a在消息b之前被一台服务器传递，那么每台传递a和b的服务器都会在b之前传递a。

**因果序列** 如果消息a在因果上先于消息b，并且两个消息都被送达，那么a必须在b之前排序。

为了保证正确性，ZooKeeper还要求有以下前缀属性：

**前缀属性**： 如果m是领导者L最后交付的消息，那么L在m之前提出的任何消息必须
也必须被传递；

请注意，一个进程可能被选举多次。但是，就这个前缀属性而言，每次都算作是一个不同的领导者。

有了这三个保证，我们就可以维护ZooKeeper数据库的正确副本：

1. 可靠性和全序列保证确保所有的副本都有一个一致的状态；
2. 因果序列保证复制体的状态从使用Zab的应用程序的角度来看是正确的；
3. 领导根据收到的请求向数据库提出更新。

值得注意的是，Zab考虑的因果关系有两种类型：

1. 如果两个消息，a和b，是由同一个服务器发送的，并且a在b之前被提出，我们说a在因果关系上先于b；
2. Zab假设每次只有一个领导者服务器可以提交提议。如果领导者发生变化，任何先前提议的消息都会因果地先于新领导者提议的消息。

我们的故障模型是有状态恢复的崩溃-故障。我们不假设同步时钟，但我们假设服务器对时间的感知速度大致相同。(我们使用超时来检测故障。)组成Zab的进程有持久的状态，所以进程可以在故障后重新启动并使用持久的状态来恢复。这意味着一个进程可能有部分有效的状态，比如缺少最近的事务，或者更有问题的是，该进程可能有先前从未交付的事务，现在必须被跳过。

我们规定要处理f故障，但我们也需要处理相关的可恢复故障，例如停电。为了从这些故障中恢复，我们要求在传递消息之前将消息记录在一个法定磁盘的磁盘介质上。(非常非技术性的、务实的、面向操作的原因使我们无法将UPS设备、冗余/专用网络设备和NVRAM等设备纳入我们的设计中)。

尽管我们没有假设拜占庭式故障，但我们确实使用摘要来检测数据损坏。我们还在协议包中添加了额外的元数据，并使用它们来检查是否正常。如果我们检测到数据损坏或理智性检查失败，我们将中止服务器进程。

操作环境和协议本身的独立实现的实际问题使得实现完全拜占庭容忍系统对我们的应用来说是不现实的。事实也表明，实现真正可靠的独立实现需要的不仅仅是编程资源[11]。到目前为止，我们的大多数生产问题要么是影响所有副本的实现错误，要么是在Zab的实现范围之外，但影响Zab的问题，如网络错误配置。

ZooKeeper使用内存数据库，并在磁盘上存储事务日志和定期快照。Zab的事务日志同时也是数据库的写前事务日志，因此一个事务只会被写到磁盘上一次。由于数据库是在内存中，而且我们使用千兆接口卡，所以瓶颈是写入时的磁盘I/O。为了缓解磁盘I/O的瓶颈，我们分批写入事务，这样我们就可以在一次写入磁盘中记录多个事务。这种批处理发生在副本上，而不是在协议层面上，所以这个实现更接近于数据库的分组提交[4, 6]，而不是消息打包[5]。我们选择不使用消息打包，以尽量减少延迟，同时仍然通过分批I/O到磁盘获得打包的大部分好处。

我们的崩溃-失败与状态恢复模型意味着，当服务器恢复时，它将读取其快照并重放该快照之后的所有已交付事务。因此，在恢复过程中，原子式广播不需要保证最多传递一次。我们使用幂等事务意味着，只要在重启时保持秩序，一个事务的多次交付是可以的。这是对全序列要求的放宽。具体来说，如果a在b之前被交付，在一次失败后a被重新交付，b也将在a之后被重新交付。

我们的其他性能要求是：

**低延迟**： ZooKeeper被应用程序广泛使用，我们的用户期待低响应时间。

**爆发式的高吞吐量**： 使用ZooKeeper的应用程序通常是以读为主的工作负载，但偶尔也会发生激进的重新配置，导致写入吞吐量的巨大峰值。

**平稳的故障处理**： 如果一个不是领导者的服务器发生故障，而仍然有一个正确的服务器的法定人数，应该不会出现服务中断的情况。

## 三、  为什么需要另一种协议

可靠的广播协议可以根据应用要求呈现不同的语义。例如，Birman和Joseph提出了两个原语，ABCAST和CBCAST，它们分别满足全序列和因果序列[2]。Zab也提供了因果和全序列保证。

在我们的案例中，Paxos[12]是一个好的协议候选者。Paxos有许多重要的特性，例如，尽管有很多进程失败，但仍能保证安全，允许进程崩溃和恢复，并使操作在现实的假设下在三个通信步骤内提交。我们观察到，有几个现实的假设，我们可以使我们简化Paxos并获得高吞吐量。首先，Paxos可以容忍消息丢失和消息的重新排序。通过使用TCP在成对的服务器之间进行通信，我们可以保证交付的消息是以先进先出的顺序交付的，这使我们能够满足每个提议者的因果关系，即使服务器进程有多个未完成的提议消息。然而，Paxos并不能直接保证因果关系，因为它不需要先进先出的通道。

提议者是为Paxos中的不同实例提出价值的代理。为了保证进展，必须有一个提议者提出，否则提议者可能会在一个给定的实例上永远争论下去。这样一个活跃的提议者就是一个领导者。当Paxos从一个领导者的失败中恢复过来时，新的领导者会确保所有部分传递的消息都被完全传递，然后从老领导者离开的实例编号开始恢复提议。由于多个领导者可以为一个给定的实例提出一个值，因此出现了两个问题。首先，提议可能发生冲突。Paxos使用投票来检测和解决冲突的提议。第二，仅仅知道一个给定的实例编号已经被提交是不够的，程序还必须能够弄清楚哪个值已经被提交。Zab通过确保对于一个给定的提议号只有一个消息提议来避免这两个问题。这就避免了对投票的需要，并简化了恢复。在Paxos中，如果一个服务器认为它是一个领导者，它将使用更高的投票来从以前的领导者手中接过领导权。然而，在Zab中，一个新的领导者不能接管领导权，直到有足够多的服务器放弃前一个领导者。

获得高吞吐量的另一种方法是通过限制每个广播信息的协议信息复杂性，如固定定序器环（FSR）协议[8]。随着系统的增长，FSR的吞吐量不会减少，但延迟会随着进程的增加而增加，这在我们的环境中是不方便的。当组的稳定性足够长时，虚拟同步也能实现高吞吐量[1]。然而，任何服务器故障都会触发服务的重新配置，因此在这种重新配置期间会造成短暂的服务中断。此外，这种系统中的故障检测器必须监测所有的服务器。这样的故障检测器的可靠性对于重新配置的稳定性和速度至关重要。基于领导者的协议也依靠故障检测器来实现有效性，但这样的故障检测器一次只监测一台服务器，也就是领导者。正如我们在下一节所讨论的，我们不使用固定的四分之一或分组来写，只要领导者不失败，就能保持服务的可用性。

根据Defago等人[3]的分类，我们的协议有一个固定的排序者，我们称之为领导者。这样的领导者是通过领导者选举算法选出的，并与其他服务器的法定人数同步，称为跟随者。由于领导者必须管理到所有追随者的消息，这种固定排序器的决定在组成系统的服务器之间不均匀地分配了与广播协议有关的负荷。不过，我们采取这种方法是出于以下原因：

- 客户端可以连接到任何服务器，而服务器必须在本地提供读取操作，并维护客户端的会话信息。这种追随者进程（不是领导者的进程）的额外负载使得负载分布更加均匀；
- 涉及的服务器数量很少。这意味着网络通信开销不会成为影响固定排序器协议的瓶颈；
- 实施更复杂的方法是没有必要的，因为这个简单的方法提供了足够的性能。

例如，拥有一个移动的排序器会增加实现的复杂性，因为我们必须处理丢失令牌等问题。另外，我们选择远离基于通信历史的模型，如基于发件人的模型，以避免此类协议的二次信息复杂性。目的地协议也有类似的问题[8]。

依赖领导者表示我们需要从领导者的失败中恢复，以保证进展。我们使用与视图变化有关的技术，如Keidar和Dolev的协议[10]。与他们的协议不同的是，我们不使用群组通信来操作。如果有新的服务器加入或离开（可能是崩溃），那么我们不会引起视图的改变，除非这种事件与领导者的崩溃相对应。

## 四、  协议

Zab的协议包括两种模式：恢复和广播。当服务开始时或领导者失败后，Zab会过渡到恢复模式。当一个领导者出现，并且有足够数量的服务器与领导者同步了它们的状态时，恢复模式结束。同步它们的状态包括保证领导者和新的服务器有相同的状态。

一旦领导者拥有一组满足法定数量的同步追随者，它就可以开始广播消息。正如我们在介绍中提到的，ZooKeeper服务本身使用一个领导者来处理请求。领导者是通过启动广播协议来执行广播的服务器，领导者之外的任何服务器如果需要广播消息，都会先将其转发给领导者。通过使用从恢复模式中出现的领导者作为处理写请求和协调广播协议的领导者，我们消除了从写请求领导者向广播协议领导者转发广播消息的网络延迟。

一旦领导者与一组满足法定数量的追随者同步，它就开始广播消息。如果一个Zab服务器在领导者积极广播消息时上线，该服务器将以恢复模式启动，发现并与领导者同步，并开始参与消息广播。该服务一直处于广播模式，直到领导者失败或不再有法定数量的追随者。任何追随者的法定人数都足以让领导者和服务保持活跃。例如，一个由三个服务器组成的Zab服务，其中一个是领导者，另外两个服务器是追随者，将转移到广播模式。如果其中一个追随者死亡，服务将不会中断，因为领导者仍然有一个法定人数。如果追随者恢复了，而另一个死了，仍然不会出现服务中断。

### 4.1 广播

![](figure02.jpg "消息传输协议")

<div style="text-align: center;"><b>图2</b>：消息传递协议。该协议的多个实例可并发运行。当领导者发布COMMIT后消息就会传递。</div>

我们在原子广播服务运行时使用的协议，称为广播模式，类似于一个简单的两阶段提交[7]：一个领导者提出请求，收集投票，最后提交。图2说明了我们协议的信息流。我们能够简化两阶段提交协议，因为我们没有中止；跟随者要么承认领导者的提议，要么放弃领导者。没有中止也意味着，一旦有足够数量的服务器确认了提议，我们就可以提交，而不是等待所有服务器的回应。这种简化的两阶段提交本身不能处理领导者的失败，所以我们将添加恢复模式来处理领导者的失败。

广播协议使用FIFO（TCP）信道进行所有的通信。通过使用先进先出（FIFO）信道，保留排序保证变得非常容易。消息通过FIFO信道按顺序传递；只要消息在收到时被处理，就能保持顺序。

领导者广播一个要传递的消息的建议。在提议消息之前，领导者分配了一个单调增加的唯一ID，称为zxid。因为Zab保留了因果序列，所以交付的消息也将按其zxid排序。广播包括将附有消息的提议放入每个追随者的外发队列中，通过先进先出通道发送给追随者。当追随者收到一个提议时，它将其写入磁盘，可以时进行批处理，并在提议出现在磁盘介质上时立即向领导者发送确认。当领导者收到来自法定人数的ACK时，领导者将广播COMMIT并在本地传递消息。跟随者在收到领导者发出的COMMIT时，就会传递消息。

请注意，如果追随者之间互相广播ACK，领导者就不必发送COMMIT。这种修改不仅增加了网络流量，而且还需要一个完整的通信图，而不是简单的星形拓扑，从启动TCP连接的角度来看，星形拓扑更容易管理。在我们的实现中，维护这个图和跟踪客户端的ACK也被认为是一个不可接受的复杂问题。

### 4.2 恢复

这个简单的广播协议能很好地工作，直到领导者失败或失去了追随者的法定数量。为了保证进展，需要一个恢复程序来选举一个新的领导者，并使所有的服务器达到一个正确的状态。对于领袖选举，我们需要一种高概率成功的算法，以保证有效性。领导者选举协议不仅使领导者知道自己是领导者，而且还使法定人数同意这一决定。如果选举阶段错误地完成了，服务器将不会取得进展，它们最终会超时，并重新开始领袖选举。在我们的实现中，我们有两个不同的领导者选举的实现。如果有一个正确的服务器的法定人数，最快的一个只需几百毫秒就能完成领导者选举。

恢复程序的部分复杂性在于，在给定的时间内，可能有大量的提议在传递。传递中的提议的最大数量是一个可配置的选项，但默认是1000个。为了使这样的协议在领导者失败的情况下仍能工作，我们需要做出两个具体的保证：我们必须永远不忘记已交付的消息，并且我们需要放弃被跳过的消息。

在一台机器上传递的信息必须在所有机器上传递，即使该机器发生故障。这种情况很容易发生，如果领导者提交了一条消息，然后在COMMIT到达任何其他服务器之前失败，如图3所示。因为领导者提交了消息，客户可能在消息中看到了事务的结果，所以事务最终必须被传递到所有其他的服务器上，以便客户看到一个一致的服务视图。

![](figure03.jpg "不能忘记的消息")

<div style="text-align: center;"><b>图3</b>：不能忘记的消息的一个例子。服务器1是领导者。它失败了，而且它是唯一一个看到COMMIT2消息的服务器，广播协议必须确保消息2在所有正确的服务器上被提交。</div>

反之，跳过的信息必须保持跳过。同样，如果一个提议是由领导者产生的，而领导者在其他人看到提议之前就失败了，这种情况就很容易发生。例如，在图3中，没有其他服务器看到第3号提议，所以在图4中，当服务器1重新启动时，它需要确保在与系统重新整合时丢弃第3号提议。如果服务器1成为一个新的领导者，并在消息100000001和100000002被交付后承诺了3，我们将违反我们的排序保证。

记忆已交付信息的问题可以通过领导者选举协议的一个简单调整来处理。如果领导者选举协议保证新的领导者在服务器的法定人数中拥有最高的提议数，那么新当选的领导者也将拥有所有承诺的消息。在提出任何新的消息之前，新当选的领导者首先要确保其事务日志中的所有消息都已经被提出并被法定的跟随者承诺。请注意，让新的领导者成为具有最高zxid的服务器进程是一种优化，因为在这种情况下，新当选的领导者不需要从一组追随者中找出包含最高zxid的服务器，并获取缺失的事务。

所有正常运行的服务器要么是领导者，要么是追随者。领导者确保其追随者已经看到了所有的提议，并且已经交付了所有已经承诺的提议。它通过排队向新连接的追随者发送它所拥有的、追随者没有看到的任何提议，然后为所有这些提议排队COMMIT，直到提交的最后一条消息，来完成这一任务。在所有这些消息被排队后，领导者将追随者添加到广播列表中，以获得未来的提议和ACK。

![](figure04.jpg "需要被跳过的消息")

<div style="text-align: center;"><b>图4</b>：需要被跳过的消息的一个例子。服务器1有一个未被提交的消息3的提议。由于后来的提议已经被提交，消息3必须被忘记。</div>

跳过已经提出但从未送达的消息也很容易处理。在我们的实现中，zxid是一个64位的数字，其中低32位作为一个简单的计数器使用。每个提议都会增加该计数器。高阶的32位是epoch。每次新的领导者接管时，它将获得其日志中最高的zxid的历时，增加历时，并使用一个zxid，其历时位设置为这个新的历时，而计数器设置为零。使用epoch来标记领导权的变化，并要求一个法定的服务器承认一个服务器是该epoch的领导，我们避免了多个领导以相同的zxid发布不同的提议的可能性。这种方案的一个好处是，我们可以在领导者失败时跳过实例，从而加快和简化恢复过程。如果一台已经停机的服务器在重新启动时使用了一个从未从上一个epoch交付的提议，那么它就不能成为一个新的领导者，因为每一个可能的服务器法定人数中都有一个具有较新epoch提议的服务器，因此具有更高的zxid。当这个服务器作为追随者连接时，领导者会检查追随者最新提议的epoch的最后承诺提议，并告诉追随者将其事务日志截断（即忘记）到该epoch的最后承诺提议。在图4中，当服务器1连接到领导者时，领导者告诉它从其事务日志中清除提议3。

## 五、  结束语

我们能够快速地实现这个协议，而且事实证明它在生产中是稳健的。最重要的是，我们实现了高吞吐量和低延时的目标。在一个非饱和的系统中，延迟是以毫秒为单位的，因为无论服务器的数量如何，典型的延迟将对应于数据包传输延迟的4倍。爆发性负载也得到了很好的处理，因为当一个法定人数的Ack提议时，消息就被提交。一个慢速的服务器不会影响突发的吞吐量，因为不包含慢速服务器的快速四重奏可以检查消息。最后，只要有一个正确的服务器的法定人数，领导者就会提交消息，失败的追随者也不会影响服务的可用性甚至是吞吐量。

由于使用了这些协议的有效实现，我们有一个系统的实现，在读写比为2:1或更高的工作负载组合中，每秒达到几万到几十万的操作。这个系统目前正在生产中使用，它被雅虎的爬虫和雅虎的广告系统等大型应用所使用。