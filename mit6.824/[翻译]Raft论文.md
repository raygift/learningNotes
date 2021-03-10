---
typora-copy-images-to: upload
---

## 摘要

Raft 是一个用来管理日志副本的共识（consensus）算法。它产生等同于（多）Paxos的效果，且同样高效，但结果与Paxos有区别；这令 Raft 比 Paxos 更容易理解 ，也为打造实践系统提供了更好的基础。为了加强理解，Raft 将共识的元素分离开，诸如 leader选举，log replication，safety，而且增加了较强程度的一致性，来减少必须要考虑的状态的数量。从学习效果来看，Raft比Paxos对于学生来说更加容易。 Raft还包括了一个改变集群成员关系的机制，此机制被用来确保安全。



## 1 引言

共识算法让一系列机器可以像一个整体一样工作，在其中一些成员异常时也能够存活。因为这个原因，共识算法在构建可靠的大型软件系统时非常重要。 Paxos 在过去十年统治了共识算法的讨论。绝大多数共识算法的实现都基于Paxos或受到其影响，Paxos已经成为共识算法教学的主要工具。

不幸的是，Paxos太难懂了，即使已经有大量让它更加容易理解的尝试。更进一步，Paxos架构需要复杂的变化来支持实际系统。因此，系统开发者和学生们都被Paxos所折磨。

我们在被Paxos折磨后，决定寻找一个新的、可以提供更好系统构建和学习的共识算法。我们的目的如此与众不同，主要目标是 更容易被理解：是否可以为系统实践构建一种比Paxos更容易学习的共识算法？更进一步，我们想让这个算法加速直觉的发展，这对于系统开发者很重要。因此，重要的不只是算法可以运行，为什么可以有效运行也很重要。

本篇的成果是叫做 Raft 的共识算法。在 Raft 设计中我们应用了特别的技术来提高可理解性，包括分解（Raft分为leader选举，log replication，以及safety）和状态空间的缩减（与Paxos相比，Raft降低了不确定性的程度，并且减少了server之间不一致性的情况）。在2所大学43名学生中的推广证明了 Raft明显比Paxos更加易懂：在学习了两个算法后，相比于Paxos，33位学生可以做到更好地回答Raft的问题。

Raft 在很多方式上与现有的共识算法类似（其中最突出的一个是  Oki 和 Liskov 的 Viewstamped Replication 算法），同时Raft还有几个创新特性：

- 强leader：Raft 使用了一个比其他共识算法更强的leadership。比如， log entries 只存在于从leader与到其他server。这简化了 日志副本的管理，使得 Raft更容易被理解。
- leader 选举：Raft 使用随机定时器来选举leader。这只是在其他共识算法已经存在的心跳机制上增加了一个小创新，但却更加简单和快速的解决了冲突问题。
- membership changes: Raft改变集群中server的机制使用了新的 joint 共识方法。。。这使得集群可以在 configuration改变时，仍能继续正常运行。

我们相信Raft比Paxos或其他共识算法更先进，无论时在可理解性上，还是作为基础实现时。它比其他算法更简单和易懂；由于描述的足够完整，它也可以很好的满足实际系统的需求；它有很多开源的实现，且被很多公司使用；它的安全性已被明确和证明；它的效率有足够的竞争力。

本文后续章节介绍了 复制状态机问题（第二章），讨论了Paxos的优劣（第三章），描述了我们的方法的易懂性（第四章），展示了Raft共识算法（第五-八章），评价了Raft（第九章），并讨论了相关工作（第十章）。



## 2 复制状态机

共识算法一般被在共识状态机中提起。在本文方法中，server集合的状态机计算 同一个状态的可标识副本，即使某些server宕机仍可正常运转。复制状态机被用来解决分布式系统中大量异常的容错问题。比如，拥有单个集群leader的大型系统，比如GFS，HDFS，以及RAMCloud，通常使用一份分布式复制状态机来管理诸如leader选举，存储配置信息等在leader异常时必须存活的服务。复制状态机的案例还包括Chubby和zookeeper。

复制状态机通常使用日志复制来实现，如图1所示。每个server存储包含一系列命令的日志，日志的状态机是排序执行的。每个日志包括了同样顺序的同样指令，因此每个状态机执行同样顺序的命令。因为这种状态机是确定的，每个状态机计算都可得到同样状态、同样顺序的输出。

保持日志副本的一致性是共识算法的任务。server上的共识模型从客户端接收命令，并将命令加入到本地日志中。共识模型与其他server上的共识模型通信，确保每个日志最终包含同样顺序的同样请求，即使有某些server出现异常。一旦命令被正确复制，所有server的状态机都按照日志顺序执行，输出将返回给客户端。因此，server需要构建一个唯一的高可用的状态机。

实际系统中的共识算法普遍有如下特性：

- 确保 **安全**（绝不会返回错误结果），在所有非拜占庭情况下都是如此，包括 网络延迟，分区，丢包，重复，乱序。（__一般地，把出现故障、crash 或 fail-stop，即不响应，但不会伪造信息的情况称为“非拜占庭错误”  non-byzantine fault 或“故障错误”  Crash Fault ;伪造信息恶意响应的情况称为“拜占庭错误”  Byzantine Fault ，对应节点为拜占庭节点。__）
- 在多数节点正常运行且可以和其他节点及客户端通信时，拥有完整的功能可用性。如此一来，拥有五个节点的集群可以容忍任意两个节点出现异常。假如节点因为停机而被认定为异常，它们可能会从持久化存储中将状态恢复，并重新加入集群。
- 不依赖于时间来确保日志一致性：错误的时钟，或者额外的消息延迟导致的最坏结果尚可接受。
- 通常情况下，在集群大多数节点向单轮远程调用作出响应时，便认为一个命令已执行完成；少数慢节点不影响系统整体性能。

## 3 Paxos 错在哪了

过去十年中，Leslie Lamport的Paxos协议已经成为共识算法代名词：它是课堂教学中最常见的协议，大多数共识算法实现时也将它作为起点。Paxos首先定义了在单个决策中达成一致的协议，比如单个被复制的日志记录（replicated log engry）。我们将此类归纳为 单决策Paxos（single-degree）。Paxos将多个这类协议实例集合起来，用以满足一系列的决策需要（multi-Paxos）。Paxos保证安全性和存活性，同时也支持集群中成员的变动。Paxos的正确性已经被证明，且在很多场景中是有效的。

不幸的是，Paxos有两个缺点。第一个缺点是Paxos异常难懂。其完整阐述的晦涩众人皆知；很少有人成功理解它，也不得不付出非常大的努力。因此，有一些让Paxos变得简单的尝试。这些尝试集中在 single-decree 部分，要做简单的解释也非常具有挑战性。从一些场合，我们了解到很少有人轻松愉悦的使用Paxos，即使是有经验的研究者们。我们也被Paxos所折磨；直到读了一些简单的解释，并设计了自己的一套共识算法后，才能对其有所理解，这花费了几乎一整年时间。

我们猜测Paxos的晦涩源于其选择了 single-decree 作为它的理论基础。Single-decree Paxos 难懂而且精密：它被分为两阶段，两阶段没有简单直观的解释，无法单独去理解。因此很难直观理解为什么 single-decree协议可以起作用。重要的是multi-paxos 更加复杂和精密。我们认为在多决策（如一个日志文件而非单个记录）上达到共识可以用其他方式拆解，从而变得更加直观、清晰。

Paxos的第二个缺点是没有提供一个构建实际系统的好的基础。这有multi-Paxos算法未被广泛理解有关。Lamport的阐述主要围绕 single-Paxos，但丢失了很多细节。有很多尝试解释和优化Paxos的工作，例如[26],[39]和[13]，但这些都与Lamport的描述有些区别。一些系统像Chubby实现了类Paxos的算法，但它们的实现细节未被公开。

Paxos也不是一个构建实际系统的好架构；这也是受到single-decree的限制。例如，单独选择一个日志记录集合然后将它们合并到 sequential log中，仅仅是增加了复杂度。当新的日志纪录以固定顺序依次增加时，围绕一个日志设计一个系统是更加简单高效的。Paoxs的另一个问题是使用了对等p2p作为其核心（尽管已经建议使用弱leader来进行优化）。这在只需要作出一个决定的简单世界中是有效的，但没有现实系统使用。如果必须要做一系列的决定，首选选出一个leader，然后让leader来做决定是简单且快速的。

结果就是，现实系统与Paxos没有什么性。每个系统都由Paxos开始，实现过程中发现诸多困难，然后在架构上作出明显改变。这非常耗时且易出错，而Paxos让人难以理解又使得问题更加恶化。Paxos理论在证明正确性时可能表现优秀，但实际实现时的巨大差异导致这些证明没有意义。Chubby的评论很有代表性：

> 在Paxos算法与现实世界的需求之间有明显的差距...最终系统将基于一个未经证明的协议[4]

由于以上这些问题，我们认为Paxos没有有为构建现实系统、算法教学提供一个好的基础。由于大型软件系统中共识算法的重要性，我们决定研究下是否可以设计一个比Paxos更好的共识算法。Raft便是我们的答案。

## 4 为可理解性而设计

我们设计Raft有几个目标：必须为系统建设提供完整和可实践的基础，从而显著减少开发者的大量设计工作；在任何条件下都保证安全，在典型操作中保持可靠；对于不同操作必须高效。但我们最重要也是最具挑战的目标是，具有可理解性。必须让大部分人可以顺利的理解算法。进一步，必须培养此算法的直觉，从而让系统构建者可以在现实实践中拓展算法的使用。

Raft的设计中有许多关键点，我们必须在众多方案中进行选择。每当遇到这种情景，我们选择方案都会基于可理解性：解释每个算法的困难程度（比如，状态空间复杂度如何，是否存在微小的暗示？），对于读者来说理解词算法和其实现的苦难程度如何？

我们意识到这样的分析有很强的主观色彩；尽管如此，我们使用两种普遍使用的方法。第一个方法是 ：将问题尽可能的拆解成可以被独立的解决、解释和理解的小问题。例如，Raft中我们对leader选举，log复制，安全和成员变换进行了拆解。

第二种方法是简化状态空间来减少需要理解的状态数量，使得系统更加统一，尽可能排除不确定性。举例来说，logs不允许存在空洞（holes），Raft限制了logs可能与其他节点不同的情况。尽管在大多数情况下我们都试图排除不确定性，某些场景中不确定性的存在提高了可理解性。特别的，随机方法增加了不确定性，但通过在一个相似的fashion中（“choose any；没关系”）处理所有可能的情况，却有利于减少状态空间。我们使用随机来简化Raft的leader选举算法。

## 5 Raft 共识算法

Raft 是一个管理第2章描述的日志副本的算法。图2 总结了算法的固定构成，图3 列出了算法的关键属性；本章将讨论这些特性的构成元素。

> ### 状态
>
> | 所有server上的持久状态（响应RPC请求之前更新到持久存储上） |                                                              |
> | --------------------------------------------------------- | ------------------------------------------------------------ |
> | currentTerm                                               | server观察到的最新term（启动时初始化为0，单调增加）          |
> | votedFor                                                  | 当前term收到选票的candidateId（若没选票则为null）            |
> | log[]                                                     | 日志记录；每条记录包含状态机要执行的命令，以及leader收到记录时的term编号（index从1开始） |
>
> | 所有server上的 易失 状态 |                                                |
> | ------------------------ | ---------------------------------------------- |
> | commitIndex              | 已知需要被提交的最大index（初始为0，单调增加） |
> | lastApplied              | 已应用到状态机的最大index（初始为0，单调增加） |
>
> | leader上的易失状态 |                                                              |
> | ------------------ | ------------------------------------------------------------ |
> | nextIndex[]        | 针对每个server，下一个待发送给该server的日志记录的index（初始为leader最后日志index+1） |
> | matchIndex[]       | 针对每个server，已知待复制到该server的日志记录的最大值（初始为0，单调增加） |
>
> 

> ### AppendEntries RPC
>
> - leader复制日志记录时调用（5.3章）；也用于作为心跳（5.2章）。
>
> | 参数         |                                                              |
> | ------------ | ------------------------------------------------------------ |
> | term         | leader的term                                                 |
> | leaderId     | 以便follower可以将client的请求重定向给leader                 |
> | prevLogIndex | index of log entry immediately preceding new ones；新记录之前的日志记录的索引；新记录指entries参数中所包含的要复制到follower的日志 |
> | prevlogTerm  | prevLogIndex 记录的term                                      |
> | entries[]    | 要存储的日志记录（发送心跳时为空；可以发送多个提高效率）     |
> | leaderCommit | leader的commitIndex【已知需要被提交的最大index】             |
>
> | 返回值  |                                                          |
> | ------- | -------------------------------------------------------- |
> | term    | currentTerm，leader用来更新自己                          |
> | success | 如果follower包含的记录匹配了 prevLogIndex 和 prevLogTerm |
>
> - 接收者的实现逻辑：
>   1. 如果 term < currentTerm ，返回false
>   2. 如果 在prevLogIndex处不包含 term 与 prevLogTerm 匹配的记录，返回false
>   3. 如果已存在的记录与新记录冲突（同样的index 对应不同的term），删除已存在的记录，且此记录之后的记录也都删除（5.3章节）
>   4. 将未存在于日志中的新记录追加到日志里
>   5. 如果 leaderCommit > commitIndex，设置 commitIndex == min(leaderCommit, index of last new entry )
>
> 
>
> ### RequestVote RPC
>
> 被candidate调用，收集选票（5.2章节）
>
> | 参数         |                                               |
> | ------------ | --------------------------------------------- |
> | term         | candidate的term                               |
> | candidateId  | candidate请求投票                             |
> | lastLogIndex | candidate的最后一条日志记录的index（5.4章节） |
> | lastLogTerm  | candidate的最后一条日志记录的term（5.4章节）  |
>
> | 返回值      |                                   |
> | ----------- | --------------------------------- |
> | term        | currentTerm ，供candidate更新自己 |
> | voteGranted | 为true时代表candidate获得了选票   |
>
> - 接受者的实现逻辑：
>   1. 如果 term < currentTerm，返回false（5.1章节）
>   2. 如果votedFor或candidated为空，candidate的日志至少与receiver的日志一样新，授权投票（5.2，5.4章节）
>
> 
>
> ### server 的规则
>
> #### 对于所有server
>
> - 如果 commitIndex > lastApplied：增加lastApplied，向状态机应用log[lastApplied]（5.3章节）
> - 如果RPC请求或响应包括 term T > currentTerm：将currentTerm设为T，切换为 follower（5.1章节）
>
> #### 对于follower（5.2章节）
>
> - 响应来自candidate 和 leader 的RPC请求
> - 如果选举超时且未能从当前 leader 收到 AppendEntries RPC，或允许向candidate 投票：切换为 candidate
>
> #### 对于candidate（5.2章节）
>
> - 在切换为candidate时，开启选举：
>   - 增加 currentTerm
>   - 投票给自己
>   - 重置选举超时定时器
>   - 向其他所有节点发送 RequestRPC请求
> - 如果获得大多数节点的选票：当选为leader
> - 如果接受到新leader的 AppendEntries RPC请求：切换为follower
> - 如果选举超时：开启新的一轮选举
>
> #### 对于leader
>
> - 赢得选举时：向所有其他节点发送 初始空的AppendEntries RPC（即心跳）请求；空闲时会重复发送心跳已避免新一轮选举（5.2章节）
> - 如果从client 接收到命令：向本地日志追加记录，在应用到状态机之后发送响应（5.3章节）
> - 如果 最后一条日志记录的 index > follower 的 nextIndex ：发送带有从 nextIndex开始日志记录的 AppendEntries RPC
>   - 如果发送成功：更新 记录的follower 对应的 nextIndex 和 matchIndex（5.3）
>   - 如果AppendEntries 由于日志不一致导致失败：减小 nextIndex 并冲高女士（5.3）
> - 如果存在 N 满足 N > commitIndex，大多数 matchIndex[i] >= N，且 log[N].term == currentTerm： 将 commitIndex 设为 N （5.3，5.4章节）
>
> **图2：Raft共识算法的总结（未包含成员变动和日志压缩）。第一个框中的服务器行为被描述为一组独立触发的规则。标注的章节号说明了讨论某些过程的具体章节。参考文献[31]对于算法有更细致的描述**



Raft通过选举一个 明显的 leader 来实现共识，然后授予leader管理日志副本的全部责任。leader 接受来自客户端的 log 记录，在其他server上复制这些日志，并在可以将日志记录应用到状态机时通知server。拥有一个leader使得管理日志副本更加简单。例如，leader可以决定在日志的什么地方存放记录，而无需咨询其他server，数据流只是简单的从leader 流下其他server。leader可能会失败或无法从其他server连接到，这种情况下会有新的leader被选举出来

拥有了leader 设计，Raft 将共识问题分解为三个相对独立的子问题，将在下面的子章节进行讨论：

- Leader 选举：当前leader出现fail时，必须选出新的leader（5.2章节）
- 日志复制：leader 必须接收客户端发来的日志记录，并在集群范围内进行复制，强制其他日志与leader维护的日志保持一致（5.3章节）
- 安全：Raft最关键的安全属性是 图3 中的状态机安全性：如果任何server的状态机应用了特定的一条日志记录，其他server不得针对相同的日志索引应用不同的命令。5.4章节描述了Raft如何保证此机制；解决办法在选举机制上包含了5.2章节所阐述的附加约束。

在给出共识算法后，本章讨论了可用性问题和系统中定时器的角色。

> |              Raft算法特性               |                                                              |
> | :-------------------------------------: | :----------------------------------------------------------- |
> |      Election Safety（选举安全性）      | 一个term内最多只会选出1个leader。详见5.2节                   |
> | Leader Append-Only（Leader 只追加纪录） | leader绝对不会重写或删除它日志中的记录；leader只向日志中追加新记录。详见5.3节 |
> |        Log Matching（日志匹配）         | 如果两个日志包含一条相同index和相同term的记录，则两个日志中此index之前的记录都相同。详见5.3节 |
> |  Leader Completeness（Leader 完备性）   | 如果一条记录在给定term中被提交，则此记录一定存在于后续term 的leader 的日志中（PS：即持有之前term已提交的记录是其他节点可以在后续term 中被选为leader 的必要条件）。详见5.3节 |
> |  State Machine Safety（状态机安全性）   | 如果一个server向状态机应用了一条日志记录，其他server不能在同样的index应用不同的记录。详见5.4.3节 |
>
> 图3：Raft 保证任意时间均满足以上特性，上图标注出了具体讨论每个特性的章节号



### 5.1 Raft基础

一个Raft集群包括几个server；5个server是比较典型的情况，可以使得系统容忍两个节点异常。在任意时间点，每个server都处于三种状态之一：leader，follower，或者candidate。正常情况下只会有1个leader，其他server都将处于follower状态。follower们比较消极：他们不处理请求，而只是简单的响应来自leader和candidate的请求。leader处理所有客户端的请求（如果1个客户端与follower通信，follower会将其重定向到leader）。第三种状态，candidate，如5.2章节所述，是用来选举出新的leader的。图4展示了集中状态的转换；状态转换后续会讨论。

> ![image-20210109212554805](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210109212554805.png)
>
> 图4：节点状态。follower 只响应其他节点的请求。如果follower收不到响应信息，它将成为一个candidate 并发起一次选举。一个candidate 获得整个集群大多数节点的选票时会成为新的leader。leader 会履行职责直到他们出现异常。

Raft 将时间分割成任意长度的term，如图5所示。term被用连续的整数进行编号。每个term都从一次选举开始，选举时会有一个或多个candidates试图成为leader，如5.2所述。如果某个candidate赢得了选举，它将在此term剩余时间承担leader角色。在某些情况下选举可能会产生平票（split vote）。此时term将会在没有leader的情况下结束；很快一个新的term将会开始（伴随着一次新的选举）。Raft保证term中只会有1个leader。



> ![image-20210109212659044](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210109212659044.png)
>
> 图5：时间被划分为term，每个term从一次选举开始。

不同的server可能会在不同时间发现term的切换，在某些情况下1个server可能不会感知到选举或整个term。term在Raft里作为一个逻辑时钟，允许server发现诸如失效的leader等废弃的信息。每个server存储一个当前term编号，此编号随着时间单调递增。无论server之间合适通信，当前的term都会变化；如果某个server的当前term小于其他server的term，server会更新到较大的term。如果一个candidate或leader发现它的term过期了，它会立刻向follower回复状态。如果一个server接收到一个带有过期term编号的请求，将会拒绝掉此请求

Raft的server之间通过RPC进行通信，最基本的共识算法只需要两类RPC。RequestVote RPC被candidate在选举期间初始化（5.2章节），AppendEntries RPC被leader初始化用来复制日志记录，并提供一个心跳的结构（5.3章节）。第7部分增加了第三个RPC，用来在server之间传输快照。如果没能及时收到消息，server会重发RPC，且通过并发RPC来达到最好的性能。



### 5.2 leader 选举

Raft使用心跳机制来触发leader 选举。当server启动时，首先作为follower。server在收到来自leader或candidate合法的RPC请求前会保持follower状态。Leader定期向所有follower发送心跳（不带日志记录的AppendEntries RPC）来维护他们的统治。如果一个follower在一段选举超时时期内未收到信息，它会假定已经无可用的leader，进而发起一次新leader的选举。

为了开始一次选举，follower增加当前的term编号，并转入candidate状态。随后，它投票给自己，且并行的向集群中每个server发送RequestVote RPC请求。一个candidate会保持在candidate状态，直到满足如下三个条件之一：(a) 它赢得本次选举，(b) 另外一个server成为leader，(c) 在本次选举时段内没有leader被选出。本论文后续章节将会分别讨论这三种情况。

如果能够在同一个term中，获得整个集群大多数server的投票，一个candidate将赢得本次选举。每个server在给定的term中至多只能投票给一个candidate，基于先来先得票原则（注：5.4章节对于投票增加了一个附加约束）。这一主要规则确保了最多只能有1个candidate在一个term中赢得选举（图3中的选举安全特性）。一旦一个candidate赢得选举，它就成为了leader。随后它向所有其他server发送心跳信息，建立起它的统治，并停止新的选举。

在等待投票期间，candidate可能会收到来自其他server的申请成为leader的Append'Entires RPC。如果leader的term（包含在其RPC请求中）不小于candidate的当前term，那么candidate会认为此leader是合法的，并转为follower状态。如果RPC中的term小于candidate的当前term，candidate会拒绝RPC请求并维持在candidate状态。

第三种可能结果是candidate既未赢的选举，也未输掉选举：如果多个follower同时成为 candidate，投票会出现分票导致没有candidate获得大多数选票。当发生此情况时，所有candidate会将当前选举超时，并通过增加term以及开始另一轮的RequestVote RPC请求，来发起新一轮的选举。然而，如果没有特殊处理，分票情况将会反复发生。

Raft使用随机选举超时来确保很少发生分票情况，并且在发生分票时可以快速处理完成。首先为了避免分票，选举超时时间在一个固定的间隔内随机确定（比如 150-300ms）。这一方法将server分散开从而在大多数情况下仅有一个server会超时；它会在其他server超时之前赢得选举并发送心跳。处理分票使用同样的机制。在一次选举开始时，每个candidate重启一个随机的选举超时，在开始新的选举前会等待超时发生；此机制降低了新一轮选举中出现分票的可能性。9.3章节表明此方法快速的选取出了leader。

选举作为一个例子，体现了可理解性如何指导我们在众多可替代设计中做选择。原本我们计划使用分级系统：每个candidate被分配一个唯一的级别，级别用来在相互竞争的candidate中进行选择。如果一个candidate发现另一个candidate级别更高，它就转为follower状态，来保证更高级别的candidate可以更轻松赢得下一次选举。但我们发现这种方法引出了新的可用性问题（如果更高级别的server异常了，低级别的server可能需要超时并重新成为candidate，**但如果这些发生的太快，它会重置选举leader的进展？？？**）。我们对算法做了多次优化，但每次优化后会出现新的问题。最终我们总结出随机重试方法是更加明确和可理解。



### 5.3 日志复制

一旦一个leader被选举出来，它便开始处理客户端请求。每个client的请求包含了一个被复制状态机执行的command。leader将此命令当作一个新纪录追加到它的日志中，然后并行向所有其他server发送AppendEntries RPC请求复制此记录。当此记录被安全的复制后（如下所述），leader将此记录应用到它的状态机，并向client返回执行结果。如果follower崩溃或运行缓慢，或者出现网络丢包，leader会不断重试AppendEntries RPC（即使在它已经向client发送了响应之后），直到所有follower最终存储下所有日志记录。

日志被如图6所示那样被组织起来。当被日志记录被leader接收时，每个日志记录存储一个带有term号码的状态机命令。日志记录中的term号被用来检查日志之间的一致性，确保图3中的一些特性。每个日志记录还拥有一个整形的索引来标识在日志中的位置。

> ![image-20210109165422178](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210109165422178.png)
>
> 图6：日志由纪录构成，记录被顺序编号。每个纪录包含它被创建的term编号（每个格子中的数字）以及提供给状态机的命令。如果将纪录应用到状态机是安全的，则认为此记录已提交。

leader决定何时将日志记录应用到状态机是安全的；这样的记录被称作 committed（已提交）。Raft保证已提交记录的持久性，且最终会被所有可用状态机执行。一旦创造记录的leader在多数server上复制了此记录，一条日志记录便会被提交（例如：图6中的记录7）。**此过程同时会提交leader中较早的日志记录，包括由早期其他leader创建的记录。**5.4章节会讨论在leader变更后使用此规则时的微妙之处（subtleties），且展示了这样定义提交是安全的。leader 持续跟踪它所知道的需提交的记录中最大的索引号，并在未来的AppendEntries RPC请求（包括心跳请求）中包含此索引号，从而让此索引号最终会被其他server发现。一旦一个follower感知到一个日志记录被提交，它会将此记录（按照日志顺序）应用到它本地的状态机。

我们设计了Raft日志机制来保证不同server日志之间的高度一致性。此设计不仅简化了系统行为，使得系统更加可预测，更是保证安全性的重要组成部分。Raft维护了如下特性，共同构成了图3中的Log  Matching Property：

- 如果不同日志中的两条记录拥有同样的index和term，则它们存储了同样的命令。
- 如果不同日志中的两条记录拥有同样的index和term，则日志中此前的记录都是一致的。

第一个特性来自这样的事实：在给定的term中，一个leader对给定的log index 至多创建一条日志记录，日志记录在日志中的位置不会改变。第二个特性由AppendEntries执行的简单一致性检查保证。当发送AppendEntries RPC请求时，leader 将日志记录里紧挨着新记录的之前的记录对应的index 和term 包含在请求参数中。如果follower在它们本地日志中未找到相同index和term的记录，便会拒绝新的记录。一致性检查归纳步骤如下：日志初始空状态满足 Log Matching Property，无论何时日志被扩展，一致性检查维持Log Matching Property。结果就是，无论何时AppendEntries返回成功，leader可以知道follower的日志在新记录上与自己保持一致。

普通操作过程中，leader和follower的日志保持一致，AppendEntries的一致性检查始终成功。然而，leader崩溃会让日志不一致（旧leader可能尚未将其日志中所有记录完成提交fully replicated）。这类不一致可能会混杂一系列的leader和follower的崩溃。图7举例说明了follower的日志可能与新leader的日志不同。follower 可能丢失已被提交到leader上的记录，也可能会存在尚未提交到leader的记录，也可能两种情况都存在。日志丢失或多余记录可能会横跨多个term。

Raft中，leader通过强制follower复制leader的日志来解决不一致情况。这意味着folloer日志中与leader冲突的记录将被leader的记录覆盖。5.4章节将展示当增加一个约束时，这种处理方法是安全的。

为了将follower的日志与leader的日志一致起来，leader必须找到两个日志文件中一致的最后一条记录，删掉此纪录后follower上的所有记录，并将leader中此记录后的所有剩余记录发给follower。所有这些动作都在AppendEntries RPC请求的响应过程中发生。leader为每个follower维护一个 nextIndex标志，表示leader将发送给这个follower的下一个日志记录的index。当一个leader刚接任时，它会将所有 nextIndex的值初始化为它自己日志的最后一条记录的index（图7中11）。如果follower的日志与leader不一致，下一个AppendEntries RPC请求中的AppendEntries一致性检查将会失败。被拒绝后，leader 减小 nextIndex并重试 AppendEntries RPC请求。最终nextIndex 将会退到 leader和follower 保持一致的点。此时AppendEntries将会成功，所有follower中与leader冲突的记录被移除，leader中nextIndex及所有后续记录将被追加到follower日志中。一旦AppendEntries成功，follower的日志将会与leader保持一致，并在后续term中继续保持。

如有必要，可以优化协议来降低被拒绝的AppendEntries RPC请求的数量。比如，当拒绝一个AppendEnrtries请求时，follower可以使用冲突记录发生时的term和此term的第一条记录索引。通过这些信息，leader可以将nextIndex 减小到此term中所有冲突记录之前；这样便不再需要每条记录一个AppendEntries，而是每个存在记录冲突的term需要1个AppendEntries RPC。实践中，我们感觉这么做没啥必要，因为失败本身就很少发生，不会存在很多不一致的记录。

通过上述机制，leader在开始统治时无需对日志一致性存储做任何额外处理。只需要执行常规操作，日志自动converge（汇集） ，作为对AppendEntries一致性检查失败的响应。leader不会重写或删除它自己日志文件的记录（图3中Leader的 Append-Only 特性）。

> ![image-20210123174317858](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210123174317858.png)
>
> 图7：当位于图最上方的leader 统治时，follower 的日志可能是下方a 到f 几种情况。每个被分割的格子代表一个日志记录；格子中的数字是记录的term号。Follower 可能会丢失记录（如a-b），可能会包含额外未提交的记录（如c-d），也可能两种情况同时存在（如e-f）。例如场景f 可能是这样发生的：term 2 中的f 作为leader 追加了几条记录，在提交记录前崩溃了；崩溃后很快又重启，在term 3重新成为leader，并再次追加了几条记录；在term 2和term 3 中追加的记录尚未提交时，此节点再次崩溃 并在后续几个term 中未能恢复。

### 5.4 安全性

前面章节描述了Raft如何选举leader，如何复制日志记录。然而，目前为止所描述的机制还无法满足每个状态机准确的按照同样顺序执行同样命令。例如，leader提交一些日志记录时，follower可能失联无法访问，随后失联的follower可能会被选举为新的leader并用新的记录进行重写；结果就是，不同状态机可能执行不同的命令序列（sequences）。

本章通过在哪些server可以被选为leader的问题上增加一个约束，来对Raft算法进行完善。此约束可确保在任何term中leader都包含了之前term中的所有记录提交（图3中的leader完整性）。拥有了此约束，我们可以让提交的规则更加明确。最终，我们提供了一个leader完整性特性的证明概述，并展示了其如何对复制状态机进行纠正。

#### 5.4.1 选举约束

所有基于leader的共识算法中，leader最终都必须存储被提交的所有日志记录。有些共识算法，比如Viewstamped Replication，即使server最初不包括所有已提交的记录，仍可能当选leader。这些算法使用附加机制来定位丢失的记录，并在选举过程中或选举刚刚结束后后提交给新leader。不幸的是，这么做增加了相当多的附加机制和复杂度。Raft使用更加简单的方法，保证了新的leader从选举开始时就包含了所有此前term中提交的记录，无需再次将这些记录发送给此leader。这意味着日志记录的流向只有一种，即从leader流向follower，且leader永远不会重写它自己日志中的记录。

除非candidate的日志包括了所有已提交的记录，否则Raft的投票程序会拒绝此candidate赢得选举成为leader。为了选举成功，candidate必须与集群中大部分节点通信，这意味着所有被提交的记录至少存在这些被通信的节点中某一台上。如果candidate的日志和大部分被通信节点的日志都至少保持一样新（up-to-date下面有精确的定义），则candidate便拥有所有已提交的记录。RequestVote RPC实现了此约束：此RPC包括了candidate日志的信息，如果投票者发现候选者的日志不如自己的新，则拒绝投票给它。

如何判断两个日志更加新，Raft通过比较日志中最后一条记录的index和term。如果日志中有不同term的最新记录，则有更新term的日志被认为是更新的。如果日志结束时的term一样，则记录较长的日志更新。

#### 5.4.2 提交之前term 中的记录

如5.3章节所述，一旦记录被多数server存储，leader便知道当前term的此条记录已被提交。如果在提交记录之前leader崩溃，未来新上任的leader将尝试完成此记录的复制。**然而，leader无法立刻完成前一个term中的被多数server存储后完成提交某记录。**图8假设了一个情景，旧的日志记录被存储到多数节点上，但仍会被新上任的leader重写覆盖。

> ![image-20210108175426602](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210108175426602.png)
>
> ​	图8：时序图展示了为什么leader不能使用旧term的日志记录决定如何提交。
>
> (a)时间点， S1 是leader，index 2处部分节点存储了副本；
>
> (b)时间点，S1 崩溃了；在term3时S5获得S3，S4的选票，加上自己的选票，被选举为leader，且在index 2位置接受了一个新的记录；
>
> (c)时间点， S5 崩溃；S1完成重启，被选为leader，并继续进行记录复制。此时，term2中的记录已经被复制到大多数节点，但尚未提交。
>
> 如果S1 如(d)所示再次崩溃，S5 可能被选举为leader（获得S2，S3，S4的选票），并使用其已复制的来自term 3 的记录对index 2位置进行重写。
>
> 然而，如果如(e)所示，S1 在崩溃前完成了当前term 中记录复制到大多数节点的操作，index 3 处的记录被认为已提交。此时index 3之前的记录也都变为已提交（**在将term 4 /index 3 的记录复制到节点后，也会对index 3 之前的记录进行复制**），而S5 也就无法赢得选举（由于index 3被S1提交，导致S5 最后一条记录的index /term小于 大多数节点的最后一条记录的index/term ，从而无法赢得选举）。
>
> **由图8可知，Leader Completeness 特性增加的特殊之处在于，当前term 提交的记录之前的记录会被直接认为已提交，而无需再通过判断是否已复制到大多数节点上；这一特性无法避免图中d 所示情况，因为S5成为leader 时，S1 的index 3记录并未完成复制到大多数节点；但这一特性确定了旧term 记录如何完成提交（结合Raft leader在每次向follower 复制记录时，会检查两者之间有区别的记录，并强制将follower 的记录改为与leader 一致；），避免了当前leader 崩溃后，被后续节点更改旧term 记录的情况**

为了避免图8中的问题，**Raft不会根据之前term 中记录的副本数量来判断其提交状态；只有当前term 的记录会通过计算副本数量来判断是否已提交（ps：若leader某term 的记录已被提交，则更早term 中的记录一定已被提交）。**由于日志匹配特性（ Log Matching Property），一旦当前term中的记录完成了提交，所有更早的记录直接被认为已提交。虽然在一些情况下leader 可以安全的完成旧term 中日志记录的提交（例如当记录已被每个节点存储时），但Raft为了简化问题使用了更加保守的策略。

Raft 在提交规则上引入了额外复杂性，这是由于当leader从前一个term复制记录时，记录保留了原始的term编号。在其他共识算法中，如果新的leader 从之前的“term”复制记录，必须使用新的“term number”。Raft的策略更加容易让人理解，因为他们跨越时间和日志，保留了相同的term编号。另外，与其他算法相比，Raft中的新leader 从旧的term中发送出的日志记录更少（其他算法在提交记录之前，必须发送冗余的记录以便给相同的记录分配新term号）（笔记：改变term号后的记录可视为新纪录，Raft避免了将旧纪录重新变成新纪录发送到各个节点的情况）。

#### 5.4.3 有关安全性的讨论

在给出完整的Raft算法后，现在可以对Leader Completeness Property 进行更详细的表述了（此讨论基于安全性证明，详见9.2章节）。采用反证法，假设没有Leader Completeness Property，推导出一个矛盾。假设term T中的leader （记为leaderT）在它的term中提交了一个日志记录，但此记录没有被后续term中的其他新leader所存储，将后续term最小的编号记为 U， U>T，leaderU为term U中的leader，且未存储上述记录。

1. 在LeaderU被选为leader时，已提交的记录必然不存在于 LeaderU的日志中（leader 从不会删除会重写记录）。
2. LeaderT将此记录在大多数节点进行了复制， leaderU收到了大多数节点的选票。因此，至少有一个投票人既收到了leaderT发出的记录，又投票给了leaderU，（**然而如果能够收到leaderT发出的记录，就不会将选票投给leaderU**），将会产生矛盾，如图9所示。
3. 投票者必定是在投票给leaderU之前就已经接受了leaderT提交的记录；否则它会拒绝来自leaderT的 AppendEntries请求（当前term大于leaderT提交的记录的term）
4. 若投票者在投票给leaderU之后，仍然接受leaderT提交的记录，由于每个“中介”leader 已经存储了这条记录（），且leader不会remove记录，follower只有在与leader的记录冲突时才会删除记录
5. 投票者被允许向leaderU进行投票，则leaderU的日志与投票者的状态一样新。这导致了两个矛盾之一。
6. 首先，如果投票者和leaderU有同样的最新日志term，那leaderU的日志至少与投票者一样，因此它的日志里包含了投票者中的所有记录。这就产生了矛盾，因为投票者包含leaderU不包含的已提交记录。
7.  另一方面，leaderU的最后日志term号一定大于投票者的。此外，也要大于T，因为投票者的最后日志term至少是T（因为包含了T中提交的日志）。而生成leaderU最后日志记录的前leader一定包含了已提交的记录（）。根据Log Matching Property，leaderU的日志也一定包含此已提交记录，与假设矛盾。
8. 命题得证。即，term号大于T的所有leader都必定包含term T中被提交的所有记录。
9. Log Matching Property保证了后续leader会包含被间接提交的记录，正如 图8(d)所示日志记录index 2那样。

> ![image-20210108182118408](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210108182118408.png)
>
> 图9：如果S1（term T时的leader）在term T内提交了一个新的日志记录，S5在term U被选举为新leader，则必定至少有一个 server（图中S3）已经接受了此日志记录并投票给了S5

通过 Leader Completeness Property，我们可以提供图3中的State Machine Safety Property，如果一个server已经在它的状态机给定索引处存储了一个日志记录，没有其他server可以在相同的索引处存储不同的其他日志记录。当server在状态机应用了一条日志记录时，它的日志中此记录的位置必须与leader的日志保持一致，且此记录必定已经被提交。现在考虑所有server的给定日志索引位置都已提交的最低term；Log Completeness Property 保证所有更高编号的term的leader将存储同样的日志记录，因此后续的term将对状态机的此索引位置应用同样的值。如此这般，保证了State Machine Safety Property。

最后，Raft 要求server根据日志index顺序应用日志记录。结合State Machine Safety Property，这意味着所有servers将对它们的状态机已相同的顺序，应用完全相同的日志记录集合。

#### 5.5 Follower和候选者崩溃

截止目前，我们的注意力始终在leader的异常上。follower和candidate的异常比leader的异常更容易处理，且两者的处理方式一样。如果follower或candidate异常，那么后续发送给它的RequestVote 和AppendEntries RPC请求将会失败。Raft 针对这些失败进行无限的重试；如果崩溃的节点重启，那RPC终将会发送成功。如果server在成功接受到这些请求，但未来得及响应之前发生异常，那在重启后会收到相同的RPC请求。Raft RPC是幂等的，因此重复收到相同请求没有副作用。比如，如果Follower 接受 包含日志记录的AppendEntries 请求，且此记录已经存在于follower的日志中，它会忽略新请求中的这些重复记录。

#### 5.6 Timing和可用性

我们对于Raft的要求之一是安全性不能依赖时序：系统不能只因为一些事件发生的比预期更快或更慢就最终得到错误结果。然而，可用性（系统及时响应客户端的能力）必然依赖时序。比如，如果信息交换所需时间比server崩溃普遍时间要长，候选者将因为无法保持足够长运行时间而无法赢得选举；没有一个可靠的leader，Raft无法正常运行。

leader选举是Raft中最依赖时序的一个环节。在满足如下时序要求时，Raft可以正常选举并维护一个稳定的leader：

> BroadcastTime << electionTimeout << MTBF
>
> 广播时间 远小于 选举超时时间 远小于 MTBF

不等式中 broadcastTime 是 server 向集群中所有server 并发发送 RPC 并收到响应的平均时间；electionTImeout 是5.2节提到的选举超时时间；MTBF 是单个节点两次异常的平均时间间隔。广播时间应该比选举超时时间小一个数量级，以便leader可以向follower可靠地发送心跳信息，避免进行下一轮选举；设置随机的选举超时时间使得分票更不容易发生。选举超时时间应该比MTBF小几个数量级，从而使得系统可以稳定运行。当leader 崩溃时，系统将出现不可用并且持续时间接近选举超时时间；我们希望这只占总时间的一小部分。

广播时间和MTBF时底层系统的特性，选举超时时间则是Raft算法中要确定的。Raft 的RPC请求普遍需要将消息写到存储里，因此广播时间在0.5 ms ～ 20ms，这取决于底层存储技术。进而得到，选举超时时间大概需要介于 10ms ～ 500ms之间。一般MTBF可达到几个月甚至更久，这很容易满足时序要求。


### 6 集群节点变更


### 7 日志压缩

Raft 的日志在接收更多客户端请求的常规处理过程中逐渐变大，在现实系统中，不能让它无限增长下去。随着日志变得越来越大，replay 的耗时也需要更多的空间和时间。如果不采取措施删除日志积累的已被淘汰的信息，这将最终导致可用性问题。

快照是最简单的日志压缩方式。快照策略中，整个当前系统的状态被写入持久存储上的快照中，然后将此时刻之前的所有日志丢弃。快照在Chubby 和Zookeeper 中都有使用，本章后续部分将阐述Raft 中的快照。

压缩的更多方法，诸如日志清理[36]和 LSM tree[30, 5]，也是可能的。这些操作每次操作一小部分数据，因此将压缩的工作量在时间维度进行了负载。他们首先选择已经加速了很多被删除和被重写对象的数据的一个区域，然后将此区域存活的对象以更加高的压缩比进行重写，释放此区域的空间。相对快照而言，这些策略需要额外的机制和更高的复杂性，而快照通过始终操作全部数据对问题进行了简化。由于日志清理需要对Raft 进行修改，状态机可以使用与快照相同的接口实现LSM tree。

图12 展示了Raft 中快照的基本思想。每个server 快速的完成快照，快照覆盖了提交到日志中的最新entries。状态机将状态写入快照构成了大部分工作内容。Raft 在快照中还包含少量的元数据：_last included index_ 是快照替换的日志中最后一个entry的index（状态机应用的最后一个entry），_last included term_ 是此entry 对应的term。元数据被保护用来支持AppendEntries 针对第快照中第一个entry 的一致性检查，因此那个entry 需要前序entry 的index和term。为了支持集群成员变更（第6章），快照还包括日志中最新的配置。一旦server 完成了快照写入，就可以删掉 last included index 之前的entries，以及早前的快照。

尽管server 正常情况下独立生成快照，个别情况下leader 还必须向落后的follower 发送快照。在leader 已经将需要发送到follower 的next log 废弃后，此种情况就会发生。幸运的是，这不是一个常见的操作：与leader 保持一致的follower 已经存有这个entry。然而，一个意外的慢follower 或者集群中新加入的server （如第6章所述）就不会持有这个entry。将这类follower 的日志更新到up-to-date 的办法就是让leader 把快照发送给follower。

leader 使用新的一个叫做InstallSnapshot RPC 来向太落后的follower 发送快照；详见图13。当一个follower 接收到此RPC 发送来的快照后，它必须决定要对日志中已有的entry 进行什么操作。通常快照会包含接收者中不包含的新信息。这种情况下，follower 会将它的所有日志废弃；日志被快照内容取代，可能存在未提交的、与快照冲突的entry。相反如果follower 收到一个快照表示了它日志中的一个前缀（由于重发或者错发），那么被快照覆盖的entries 会被删除，但日志中快照范围之后的entry 仍然是合法有效的，必须被保留。

上述快照策略偏离了Raft 的强leader 原则，因为follower 可以在leader 知晓的情况下生成快照。然而，我们认为这种偏离是公平的。在使用leader 避免达成共识过程中出现冲突的同时，在生成快照时共识已经达成，因此不会有冲突的决定。数据仍然只从leader 流向follower，只是follower 现在可以重新组织它们的数据。

让我们考虑一个基于leader 的替代方案，此方案中只有leader 可以创建快照，随后leader 将快照发送给每个follower。然而，这个方案有两处缺陷。首先，向每个follower 发送快照浪费了网络带宽，降低了快照执行速度。每个follower 已经拥有了自己生成快照所需的信息，而且通常server 从本地状态生成一个快照比通过网络发送接收快照的代价更低。其次，leader 的实现将会很复杂。例如，leader 需要将向follower 发送快照和向follower 复制entries 并行执行，还要不能阻塞处理新的客户端请求。

影响快照性能的因素还有两个。首先，servers 必须决定何时生成快照。如果一个server 频繁生成快照，会造成对磁盘io带宽和性能的浪费；如果快照频率过低，会带来存储容量用尽的风险，也增加了重启时重新执行快照所需的时间。一个简单的策略是当日志达到固定大小时进行快照。如果这个固定大小设定的明显比预期的快照大小更大，那么快照占用的磁盘io带宽将会很小。

第二个性能问题是写快照需要一定的时间，而我们不希望这导致正常操作有延迟。解决办法是使用 **copy-on-write** 技术，从而新的操作不会影响快照的生成。例如，基于函数式数据结构构建的状态机天然支持此技术。或者，操作系统的 copy-on-write 支持（例如 linux 上的fork）可以被用来创建整个状态机的一个内存中的快照（我们的实现使用了此方法）

### 8 客户端交互

本章阐述客户端如何与Raft 交互，包括了客户端如何找到集群的leader，以及Raft 如何提供线性语义。这些问题出现在所有基于共识的系统中，Raft 的解决方案与其他系统也类似。

Raft 的客户端将所有请求发送给leader。当一个client 第一次启动时，它随机选择一个server 进行连接。如果第一次选择的不是leader，server 将拒绝客户端的请求，并返回server 接收到的最新的AppendEntries 中表明的leader 给客户端参考（AppendEntries 请求包含了leader 的网络地址）。如果leader 崩溃，client 的请求将会超时；clients 随后会尝试选择随机的server 再次尝试发送。

我们对于Raft 的目标是实现线性语义（每个操作在请求和响应之前的某一时刻即时执行，且只执行一次）。然而，如上所述Raft 会多次执行同一个命令：例如，如果leader 在提交entry之后，响应给client之前崩溃，此时client 将向新leader 进行命令重试，导致命令被执行了两次。解决方案是为client 发送的每个命令添加一个唯一序列号。然后状态机跟踪每个client 发送的最新序列号，以及相关的响应。如果收到一个已经执行过的命令，不需要重复执行，直接响应结果。

只读操作在不向日志执行任何写入就能处理。然而如果没有额外处理，会存在返回过期数据的风险，因为响应请求的leader 可能已经被新的leader 取代。线性读取必须返回最新数据，不使用log，Raft 需要两个额外操作来保证不返回过期数据。第一条，leader 必须在已经提交的entry 上有最新信息。Leader Completeness Property 保证了leader 拥有所有已提交的entry，但在term 初始阶段，leader 可能不确定哪些entry是已提交的。为了找出已提交entry，需要提交一个本term 中的entry。Raft 让每个leader 在term 开始时向日志中提交一个空操作entry 来解决这个问题。第二条，leader 在处理一个只读请求之前，必须检查自己是否已经被免职（如果有更新的leader 被选举出来，此前的leader 信息可能就是过时的）。Raft 通过在处理只读操作前与大多数节点完成心跳来解决这个问题。另外，leader 可以依靠心跳机制来提供租约[9]，但这可能依赖于计时的安全性（它假设时钟误差是有限的）