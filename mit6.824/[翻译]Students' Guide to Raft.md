Students' Guide to Raft（预计时长30分钟）

过去几个月，我从事MIT 的6.824 分布式系统课程的助教。此课程拥有一套基于Paxos 共识算法的实验课，但今年我们决定转移到Raft 算法。Raft 设计目标是更容易理解，我们希望这个调整可以让学生更加轻松。

本文以及相关的Instructors's Guide to Raft 文章，开启了我们Raft 的旅程，希望可以对Raft 协议的实现有所帮助，同时让学生们更好的理解Raft 内部原理。如果你在寻找 Paxos vs Raft 的对比，或者更多有关Raft 的教学分析，你应该去读一下Instructors’ Guide。本文最下方给出了6.824 学生常问到的一些问题，并给出了这些问题的答案。如果遇到了下方没有列出的问题，可以查看 [Q&A](https://thesquareplanet.com/blog/raft-qa/])页面。本文比较长，但所有提到的是大多数6.824学生（包括助教）实际遇到的问题。值得一读。

### 背景

在深入Raft 内部之前，一些背景介绍可能有帮助。6.824曾有一系列使用Go 构建Paxos的实验；使用Go 是因为简单易学，且可以方便的构建并行，分布式应用（goroutines随手可用）。通过4个实验，学生们构建一个有容错的，分片的key-value 存储。第一个实验要求构建一个一致性日志库，在此基础上第二个实验增加了kv存储，第三个实验在可容错的集群中对key空间进行分片，使用可容错的分片master 来处理配置变化。还有第四个实验要求学生能够处理机器的失败和恢复，分为磁盘未损坏和磁盘损坏两种情况。第四个实验也被当作默认的最终作业。

今年，我们决定使用Raft 重写所有上述实验。前三个实验还是一样，但由于在Raft 中已经有持久化和失败恢复设计，第四个实验被删掉了。本文将主要讨论第一个实验中的一些经验，因为第一个实验是与Raft 最直接相关的，尽管我也会提及一些基于Raft 构建应用的情况（这是第二个实验的内容）。

Raft，对于初学者来说，可以被Raft 网站上的一段话完整概括。

> Raft is a consensus algorithm that is designed to be easy to understand. It’s equivalent to Paxos in fault-tolerance and performance. The difference is that it’s decomposed into relatively independent subproblems, and it cleanly addresses all major pieces needed for practical systems. We hope Raft will make consensus available to a wider audience, and that this wider audience will be able to develop a variety of higher quality consensus-based systems than are available today.

（Raft 是一个为了易懂而设计共识算法。在容错和性能上等同于Paxos。不同之处在于将问题拆分为相关的独立子问题，并清晰的指明了现实系统中所需的所有关键部分。我们希望Raft 提供更广泛的共识，受众可以构建大量更高质量的基于共识的系统。）

[可视化](http://thesecretlivesofdata.com/raft/)提供了对于Raft 主要构成的很好的概览，论文也提供了为什么大量细节是必要的。如果还没读过论文，应该在阅读此文章前先去读一下原文，本文假设读者对Raft 比较熟悉。

在所有的共识协议中，魔鬼都存在于细节里。在没有失败的稳定状态里，Raft的行为很好理解，也可以以符合直觉的方式解释。比如，从可视化效果中可以看到，假设没有失败，最终会选出一个leader，并且所有被发送给leader 的请求都最终被应用到所有follower 上，如图右侧所示。然而，当发生消息延迟，网络分区，以及失败的节点被感知到时，每个if，but，以及and，都变得关键。特别是，有些bug会在学生作业中不断重复出现，只是因为在阅读论文时的误解或疏忽。这一问题不止针对Raft，而是在所有复杂分布式系统中都会遇到的。

### 实现Raft

Raft 的终极导读是Raft 论文的图2。这张图指定了Raft server之间每个RPC 通信的行为，提供了server 必须维持的许多恒定行为，并指定了哪些行为应该发生。我们将在本文剩余部分讨论很多图2的内容。需要严格遵照。

图2 定义了每种状态中，针对每个RPC 请求，每个server 应该做什么，...



实际上，图2 非常精准的提供了每个状态的处理逻辑，这些状态需要被严格对待，而非建议性的。例如，我们或许会在接收到appendentries 或requestvote RPC 的任何时候对节点选举计时器进行重置。重置超时计时器意味着其他节点认为已当选为leader 或 尝试成为leader，好像我们不应该对此进行干扰。然而，如果仔细阅读图2，有这样的表述：
> 如果在选举超时时间内，没有接收到当前leader 发送的 AppendEntries RPC请求或将选票投给候选者，节点应该从follower 切换为候选者

这样的描述与前述做法的差异非常重要，因为前述做法可能会导致在固定情景下的可用性降低（ps：屏蔽了本该出现的选举行为？）




### 细节的重要性

为了让讨论更具体，让我们思考一个困扰很多6.824 学生的问题。Raft 论文多次提到了 heartbeat RPC。具体来说，leader 会间断地（至少每个心跳周期内发送一次）向所有节点发送AppendEntries RPC 来阻止他们发起新一轮选举。如果leader 没有新entry 要发送给发送给follower，那发送的AppendEntries 不包含任何entry，此时被认为是一个心跳。

很多学生将心跳当作比较“特殊”的请求；当一个节点收到心跳时，会与非心跳AppendEntires 请求进行区别处理。尤其是有些人会在收到心跳时直接重置选举超时计时器，然后返回成功，而没有进行图2中提到的任何检查。这非常危险。**收到RPC 请求时，follower 隐性的告诉了leader 他们在请求参数 prevLogIndex 及之前的日志已经与leader 的日志同步一致。**在收到reply 后，leader 随后会（错误地）认为日志已经复制到大多数节点，并开始提交这些日志。

另外一个可能出现的问题（通常在修复了上述问题后出现），是收到心跳请求后，会将follower 日志中 prevLogIndex 之后的日志截断，并将AppendEntries 中包含的任何日志追加到follower 日志中。这也是不正确的。让我们回顾一下图2：

> If an existing entry conflicts with a new one (same index but different terms), delete the existing entry and all that follow it.
> （如果已存在的entry 与接收到的新entry 冲突（在相同的index 却有不同的term），则需要删除已存在的entry及其之后的entry）

此处的“if”很关键。如果follower 拥有leader 发送的所有entries，follower 一定不能截断它自己的日志。任何与leader 发送来的entry 保持一致的元素都必须被保留。这是因为我们可能会收到leader 发送的过期的AppendEntries，删除那些日志意味着将已经告诉leader 完成同步的日志进行回退。


### Debuggiing Raft

#### Livelocks

-  确保只在需要时重置选举超时计时器

    a) 从leader 接收到 AE RPC 请求（如果AE 的参数已经过期了，则不应该重置选举超时计时器）

    b) 开启新的选举

    c) 将选票投给其他节点

最后一种情况在不可靠网络中特别重要，因为follower 可能持有不同的日志；这种情况下，你可能经常由于只能获得少量选票。如果无论谁来请求选票时，都重置选举超时计时器，就没有对持有过期日志的节点和拥有更多日志的节点进行区分。

实际上，由于只有很少节点拥有足够新的日志记录，（日志不够新的）那些节点不太能够获得足够选票赢得选举。如果遵从论文图2，拥有更新日志的server 不会被落后的server 发起的选举打扰，新leader 也能够更快被选出。


- 何时发起一次新的选举（遵循图2 的指导）。特别的，注意到自己是candidate（），但选举超时计时器已经超时，此时server 应该发起新的选举。这对于避免系统因为延迟或丢失RPC 而暂停至关重要。

- 在处理RPC 请求之前，确保遵循了 “Rules for Servers” 中第二条原则：
  
> 如果RPC 请求或响应包含Term T > currentTerm，将currentTerm 设为 T，并转换为 follower

例如，如果你已经在当前term 中投过票，RequestVote 请求中拥有一个更高的Term，应该首先退位并接收他们的term号（同时重置votedFor），然后处理RPC，处理结果是投票给此发送RPC 的节点

#### 不正确的RPC 处理

1. AppendEntries RPC 过程中如下细节需要注意：

- 如果要返回 reply false，意味着应该立刻返回，不执行任何其他步骤
- 如果收到的AppendEntries RPC的prevLogIndex 参数超出了节点rf.log 的长度，应该像rf.log 中有此index 只是term 不匹配一样响应（reply false）
- 即使leader 没有发送任何entries，对于AppendEntries RPC 处理的检查逻辑（检查prevLogIndex及prevLogTerm）也应该被执行
- AppendEntries 的最后一步（#5）中的 min 是必要的，需要和最新一条entry 的index 计算（如果leaderCommit > commitIndex, 将 commitIndex 设为 leaderCommit 与 最新一条entry 的index 两者中较小的那个值）。
  
# TODO
Q：分析min的必要性

A：在执行到log 结尾时，仅仅将从lastApplied 到commitIndex 进行命令应用的方法终止是不够的。因为在leader 将entries 发送给follower后（与follower 本地entries 都匹配），你的日志中可能存在与leader 有冲突的entries【ps：虽然leader 将entries 发送过来，但follower 可能未完全追加到自己的log 中，导致与leader 不一致】。由于第三步#3 决定了follower 只会删除与leader 冲突的entries，而如果leaderCommit 超过了 leader 发送给你的entries 的index，你可能会应用错误的entries。

2. RequestVote RPC 过程中需要注意：
- 准确实现5.4章节描述的 “up-to-date log”的检查是非常重要的。不要仅仅检查一下log的长度！


#### 可能破坏论文中原则的情况

Raft 论文对于如何实现每个RPC handler 做了详细说明，但仍然存在大量未指定的规则实现。这些都被列在图2 右侧的“Rules for Servers”中。其中有一些描述是自解释的，但仍然存在一些需要仔细设计从而避免破坏Rules的点：

- 在执行过程的任何时候满足 commitIndex > lastApplied 时，每次只应用一个entry。立刻执行apply entry 时，这一条并不是特别重要（例如，在AppendEntries RPC 处理函数中立即apply），但是需要保证只被一个实体执行。特别的，需要一个指定的“applier”，或者需要在应用时使用锁，从而其他routine 不会同时检测到有一些entries 需要被应用并且尝试进行应用。
- 确保对 commitIndex > lastApplied 进行定期检查，或在 commitIndex 更新后进行检查（也就是，在 matchIndex 更新后进行检查）。例如，如果在向其他节点发送 AE RPC 时检查 commitIndex，你可能需要在下一个 entry 被追加到日志之前一直等待，然后才能应用刚刚发送的entry，并得到来自节点的确认。
- 如果一个leader 发送了AE RPC，然后被拒绝了，拒绝原因不是日志的不一致（即由于term较小被拒绝，只可能出现在当前term 过期之后），此时leader 应该立刻退位，并且不更新nextIndex。如果更新nextIndex ，可能在紧接着重新当选时，与重置nextIndex 的操作产生竞争。
- leader 不允许更新之前term 的commitIndex（或者未来term 的commitIndex）。也就是论文中所述，需要特别的检查 log[N].term == currentTerm。这是由于Raft leader 不能确认来自其他term 中的entry 是否已被提交，而一旦提交就不再允许修改。论文中图8 对此进行了描述。


有一个困惑点是区分 nextIndex 与 matchIndex。特别的，你可能会注意到 matchIndex = nextIndex - 1，因此可以不需要 matchIndex。但这是不安全的。

.

.

.

#### Term 困惑

Term 困惑指节点们被来自其他旧term 的RPC 请求搞得很困惑。大体上，当接收RPC 时这本不算问题，因为图2 中的原则准确的描述了当收到旧term 时应该如何做。然而，图2 没有讨论当收到RPC 响应中term 更旧时应该怎么做。根据经验，我们发现目前为止最简单的处理方式是，首先记录下reply 中的term 号（可能比你当前的term 大），然后将你当前term与你发送RPC 时的term 号进行比较。如果两者不同，则丢掉reply 并返回。只有当term 号相同时才继续对reply 进行处理。可以对此处理方式进一步优化，但此方法实际效果不错。如果不这么做，你将付出大量的血汗泪...

一个相关但不完全相同的问题是，

#### 一些优化

日志压缩...

加速leader 向follower 同步entry 时nextIndex 的回溯，原论文没有特别提到，可能是因为作者认为对于大多数开发者而言不是必须的。论文中没有明确leader 在接收到RPC 响应中有冲突的index 及 term 后如何处理nextIndex 。我们认为作者可能想让你遵循的原则是：
- 如果follower 的日志中没有 prevLogIndex，应该在处理RPC 请求是返回 coflictIndex=len(log),conflictTerm=None.
- 如果follower 的日志中存在 prevLogIndex，但对于的term 并不匹配，应该返回 conflictTerm=log[prevLogIndex].Term，然后在follower 的日志中找到匹配 conflictTerm 的第一条entry的index
- 在leader 收到表示存在冲突的响应后，首先应该在日志中查找匹配 conflictTerm 的entries。如果找到了term 匹配conflictTerm 的entry，应该将nextIndex 设为匹配此term 的entry 最大index +1
- 若在leader 中找不到conflictTerm对应的entry，leader 应该将nextIndex =conflictIndex

一种折衷方案是只使用conflictIndex（而忽略conflictTerm），这会简化代码实现，但leader 有时会为了同步日志发送更多不必要的entries