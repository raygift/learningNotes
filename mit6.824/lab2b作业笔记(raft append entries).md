lab2B 目标为实现Raft 的日志提交

测试案例TestBasicAgree2B：使用cfg.one 分三次提交三个命令

config.go/cfg.one() 调用raft.go/Start() 尝试完成日志记录的追加，但由于one()选中的节点不一定是leader，因此可能会被直接拒绝；若选中leader 提交命令，leader 在不产生异常被剥夺领导权利的情况下，将日志记录同步给follower 并完成提交。

Start() 将日志记录append 到自己的log 属性中，并通知leader 的gorotine开始向follower 同步记录

Q：发送带有日志记录的goroutine，与发送心跳的goroutine 是合用，还是独立使用？

A：发送心跳的goroutine 需要定时执行，而同步日志只有在新记录追加到leader 后才执行，因此计划使用单独goroutine 发送日志记录
.

Q：candidate 是否需要接收leader 发送来的AppendEntries 来更新自己的日志

A：candidate 未成为leader 之前，收到leader 发送的心跳，会放弃竞选且转为follower；若收到来自leader 的AppendEntries 请求，且满足term 大于自己的term，以及满足log matching 的要求，则认可leader 的统治，转为follower
.

Q：每次AppendEntries 肯定不会携带所有历史entry，只需要携带最后一条完成提交的记录后面的记录即可

A：raft 节点的nextIndex 和 matchIndex 分别决定了发送向follower 的日志包含的记录的最小和最大index
.

Q：follower 对于AppendEntries 与heartBeart 的处理，出了Entries 属性是否为空，在其他属性上的处理是否有区别？譬如heartbeat 是否需要携带正确的 leaderCommit，prevLogIndex，prevLogTerm？

A：
论文5.3“The leader keeps track of the highest index it knows to be committed, and it includes that index in future AppendEntries RPCs (including heartbeats) so that the other servers eventually find out. ”
.

Q：日志记录被大多数节点接收，和日志记录被应用到本地状态机，是否表达的意思完全一样？
A：不完全一样，被大多数节点接收可以被应用到状态机的记录成为已提交，记录变为已提交后才能应用到状态机。

论文5.3“The leader decides when it is safe to apply a log entry to the state machines; such an entry is called committed.”

日志被大多数节点接收是一个集群全局状态，针对某一条日志记录，若已被leader 成功发送到大多数节点，则认为此日志记录已完成提交"commited"；

日志记录被应用到本地状态机，是每个节点的局部状态；

一旦一条日志记录成为已提交状态后，节点leader 将其应用到本地状态机，并向所有follower 发消息告知本条记录可以应用到各自的本地状态机

一旦follower 知道leader 将日志记录应用到了本地状态机，则他们也会按照日志顺序将记录应用到自己的状态机

.

Q: 再议AppendEntries RPC 请求参数中prevLogIndex 的作用（做lab2a 心跳请求时已分析过一次所有RPC 参数含义，本次主要研究 prevLogIndex、commitIndex、matchIndex）

A：思考日志记录从leader 流向follower 的过程实现

整个日志复制过程中，leader 需要完成的任务分为4个部分：
1. 接收新的日志记录
2. 根据follower 已复制日志index，确定同步日志范围，并将要同步的日志记录发送给follower
3. 将复制到大多数follower 的记录提交，应用到状态机并发送提交已完成提交的记录索引号给follower
4. 向客户端反馈已成功应用到状态机的记录

leader 为完成任务，需要具备的属性如下：
- leader 为完成新日志记录的接收，需要有如下属性
1. []Entry，存储新记录具体内容

- leader 为完成与follower 同步日志记录，需要知道如下信息：
1. nextIndex[i]，每个follower 的日志已完成哪些记录的复制；同步的记录范围为nextIndex[i]~len(rf.log)-1

- leader 为了完成日志的提交，需要使用如下属性：
1. commitIndex，存储已复制到大多数节点，即已提交，可应用到状态机的最大index
2. matchIndex[i]，存储节点上持有的已提交（尚未应用）记录的最大index

- 为了向客户端反馈已成功应用到状态机的记录，需要用到
1. lastApplied，存储已应用到leader 状态机的最大index，已应用到节点的记录无法再次被修改


整个日志复制过程中，follower 需要完成的处理分为：
1. 判断leader 是否已失去统治
2. 判断与leader 数据一致程度，是否与leader 记录的相符合（因为leader 是根据一致程度，决定同步给follower 的具体日志范围）
3. 复制leader 发送来的正确范围的记录
4. 修改此前已完成复制的记录的提交状态，并应用到状态机

follower 为了完成如上处理，需要具备的属性如下：
- 判断leader 是否已失去统治
1. term，对比leader 的term 与自己的currentTerm

- 判断与leader 数据一致程度
1. prevLogIndex、prevLogTerm 用于判断follower 节点已存储的记录是否与leader 认为的相同

- 复制leader 发送来的正确范围的记录
1. prevLogIndex + 1 为新增加记录开始的索引号
2. []Entry，leader 发送的具体记录内容

- 更新此前已完成复制的记录的提交状态
1. leaderCommit，leader 认为哪些记录已完成提交可以应用到状态机了

.

Q：follower 接收到AppendEntries RPC 请求后的处理

A：
- 接受到AppendEntries 后，无论是否为心跳包，都需要进行如下处理：
1. 更新自己的currnetTerm
2. 更新自己记录的leaderId
3. 更新可以提交的记录index
4. 更新接到leader 心跳或日志同步请求的时间
5. 将自己的状态切换为（或维持）follower
- 若请求是日志同步请求，还需要进行如下处理：
1. 修改与leader 不一致的未提交记录
2. 追加leader 新增的日志记录

Q: 再议AppendEntries RPC 请求参数中matchIndex[] 的作用

A: 论文中对于matchIndex 属性的解释只有一处两句话，在图2 "Rules for Servers"的 “Leader” 部分

> If there exists an N such that N > commitIndex, a majority of matchIndex[i] ≥ N,and log[N].term == currentTerm: set commitIndex = N (§5.3, §5.4).
> 
> 翻译：如果存在 N 满足 N > commitIndex，matchIndex 中大多数值大于N，且 log[N].term 与currentTerm 相等，则将 commitIndex置为 N

向Leader 提供commitIndex 的最新值，当matchIndex 数组中记录的大多数节点对应的值都大于当前节点的 commitIndex ，且log[N] 对应的term 为当前term 时，更新commitIndex 值（由于节点只提交当前term的记录，之前term的记录被默认已提交）

可以看到matchIndex 在此处的作用是：存储leader 已经完成向follower 复制的记录索引号，并在某索引号大于peer/2个之后判断已完成提交，将索引号最终同步给 commitIndex 来存储已提交记录的状态

那么在代码实现时，便可以在使用goroutine 成功向各个节点并发发送新记录后，首先更新matchIndex[peerindex] 的值；在外部循环判断 matchIndex[] 中超过半数的最大值，更新到 commitIndex

Q：为什么同时需要nextIndex[] 和matchIndex[] 两个数组来记录leader 和follower 的同步状态？

A：
- matchIndex 主要用于在复制entry期间判断是否超过半数已完成复制；实际使用中，leader 当选后重置所有matchIndex[]初始值为leader.log 的最大index + 1 ，且在某个leader统治期间，matchIndex 的值是单调增加的，不会变小
- nextIndex 主要用于leader 判断向follower 发送哪些entries；实际使用中，leader 当选后重置所有matchIndex[]初始值为0 ，在follower 复制滞后时，nextIndex 需要减小以便判断与leader 保持一致的entry 最大的index


Q：检查 matchIndex[] 中哪些index 复制超过半数的逻辑，是否需要单独一个goroutine？

A：如果不单独goroutine 执行，只能在发送AppendEntries 的goroutine 中，在得到RPC 响应后更新计数并判断是否已提交。但此做法的问题是只有在有新记录写入时，才会触发Leader 发送AppendEntries ，如果某次新增记录的同步未完成，会阻塞leader 接收后续新增记录

Q：分析 lastApplied

A：
- 作用：所有节点都有lastApplied，用来记忆已经完成提交并应用到本地状态机的entry
- 更新：独立一个goroutine 持续判断是否rf.commitIndex > rf.lastApplied，若满足则使用较大的 rf.commitIndex 更新 rf.lastApplied
- 更新后向applyCh 依次发送所有完成apply 的命令

## TestFailAgree2B

现象：cfg.one() retry 为true 时，会出现重复记录cmd 106的情况，导致报 “out of order” 错误

```golang
	cfg.one(106, servers, true)
```

```
2021/02/07 15:14:01 rf.Start(106) ok: true ,index1: 7
2021/02/07 15:14:01 somebody claimed to be the leader and to have submitted our command; wait a while for agreement.
2021/02/07 15:14:01 server 1 in term 3 ready to send append entries, log: [{0 <nil>} {1 101} {1 102} {1 103} {1 104} {1 105} {1 106} {3 106}]
.
.
.
2021/02/08 21:45:37 apply error: server 1 apply out of order 7
```

config.go 中对于 retry 有如下注释

> // if retry==true, may submit the command multiple		// 如果 retry==true，可能会多次提交命令
> 
> // times, in case a leader fails just after Start().	// 以避免leader 在Start() 之后就发生了异常

通过分析如下几个问题解决错误：

1. config.go 中如何控制重发 106 命令的？

    config.go 中的 one() 方法在10秒内会不断重试，若retry 为false ，则在任意server中检测到2秒后nCommitted(收到的命令对应index) 与提交的cmd 仍不一致时，会在 控制10秒的一次循环之后完后，再次对相同的cmd 执行Start(cmd)；若 retry 为true ，则对于上述情况（超过2秒但小于10秒未能读取到发送的cmd），不会再次执行Start，而是直接报错

2. "out of order" 什么意思？

    在2秒内，由客户端发送的cmd 并未被复制到集群中所有节点上；此时会尝试再次发送cmd 106，从而导致第二次记录的106对应的index 并不是预期的序号6

3. 将106、107 retry 标志改为false，测试会由于在写入106 时无法达成一致而测试失败

```
--- FAIL: TestFailAgree2B (4.39s)
    config.go:508: one(106) failed to reach agreement
```

4. 观察日志还有有大量由于 leader 的nextIndex[peerid] 大于 follower 的 log 长度，且出现数量与发送日志的次数不一致

    分析发现是发送心跳且遇到nextIndex 不一致时，没有对nextIndex[peerid] 进行递减更新

5. 调整nextIndex 后发现正确的nextIndex 只有在客户端发送下一次 cmd 时才会进行同步，原因是同步日志的命令循环等待 新cmd 的条件锁触发

修改同步日志逻辑，尝试在每次缩小nextIndex 后同样触发一次logCond，使得leader 向follower 同步Entries

```
	// 单独一个goroutine
	// 作为leader ，当收到新的日志记录时，向follower 同步日志
    	go func() {

        }
```

Q：问题：“follower 被disconnect 后再re-join ，是否会立刻发起新一轮选举影响当前term？”

    A：follower 被disconnect ，但节点仍存活，判断心跳超时后会term+1 成为候选者，因此再次加入集群时会影响当前leader，但如果此follower 落后的日志太多，不包含某一条已提交的日志，则由于 Leader Completeness Property 无法当选为leader
