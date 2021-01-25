## 824 Lab2

本lab目标是根据raft论文，主要是图2和第5章的raft关键逻辑描述，完成lab2；lab2 又分了三部分，分别是
- 2A：实现基于rpc的投票请求；
- 2B：
- 2C：

开始做2A，首先向从测试用例test_test.go/TestInitialElection2A 入手，发现了一个函数，由于对Go testing包不熟悉，本文件搜索没找到函数声明，还以为是testing包里的函数，又转头去看了下Go test...未果搜索后发现在 raft/config.go文件里

读make_config时奇怪为什么建立完整的raft ，raft 中日志记录的持久化，持久化状态记录，都是通过 config 这个结构体实现的，config包的说明也只是说支持 Raft tester
。
。
。
config 包是用来组织支撑 Raft 测试案例的，那么就必须组织起一个Raft集群，包括明确集群节点个数，节点之前建立RPC连接，定义好传输命令的通道
.
.
.
回到2A的目标，是实现 Raft 的选举投票部分，测试案例中有关 2A 的案例名称分别是 TestInitialElection2A 和 TestReElection2A，很明显是需要实现选举的初始化和选举leader，
.
.
.
当发送 RequestVote 时，发送方（即候选者本人）需要提供如下信息：
1. 发送方当前的term
2. 发送方的id
3. 发送方存储的最后一条日志记录的index
4. 发送方存储的最后一条日志记录的term号

处理RequestVote 的 follower，根据如下规则判断是否投票给请求方：
1. 请求方的term是否大于等于接收者的term号。若小于则拒绝投票给请求方，若不小于则继续判断
2. 请求方的voteFor字段是null，或者是其id，且请求方的日志至少与接收方的日志一样新，则投票给请求者
3. 如何判断请求方与接收方的日志谁更新？

Q：在作为候选者发起投票时，如何判断有其他节点成功当选leader？
A：
。

Q：lab2建议使用time.Sleep() 执行定时或延后执行的任务，都有哪些需要定时执行的任务？
A：
1. 作为follower，定时检查心跳是否超时
2. 作为leader，定时发送心跳
3. 作为candidate，定时检查选举超时

Q：需要单独维护多少个goroutine？
A：已知需要维护goroutine进行状态检查的有：
1. 心跳超时检查
2. 选举超时检查
3. 
。
Q：节点个数与并发数量有什么关系？

.
Q: RequestVote RPC 请求参数的作用，lastLogIndex 与lastLogTerm 的含义
A:
1. 候选者term，提供给投票人用以判断候选者的term是否足够新
2. candidateID，提供给投票人用来记录选票投给了哪个候选者
3. lastLogIndex，候选者持有的日志记录中最后一条记录的index
4. lastLogTerm，候选者持有的日志记录中最后一条记录的term

接收到RequestVote 的节点，会根据候选者的 lastLogIndex 与 lastLogTerm ，来与自身进行比对，当发现在最后一条日志记录的index 相同，但记录的term不同时，说明产生了论文图8中的情况，此时候选者为S5，而投票者是S1/S2/S3中的一个，此时S5无法获得S1/S2/S3 的选票，也就无法当选。保证了 Leader Completeness Property（领导者完整特性，即只有包含所有可提交记录的候选者才能当选为leader，也可表述为leader 拥有所有已提交的日志）

.
Q: AppendEntries RPC 请求参数的作用，prevLogIndex 与prevLogTerm 的含义
A:
1. term，leader的term，leader当选后，向其他节点发送心跳宣称自己当选，其他候选者将对比leader的term和自己的term，如果leader的term不小于候选者自己的term，则候选者放弃竞选，成为follower接收leader的统治；若候选者的term大于leader的term，则候选者会拒绝承认此leader，并继续向所有server发送投票请求
2. leaderId，leader的编号，以便follower 可以将客户端的请求重定向到 leader进行处理
3. prevLogIndex，之前刚刚新的日志记录的index？？？说人话：leader 已知的可提交的最大记录索引
4. prevLogTerm，prevLogIndex 所对应记录的term，用来提供给follower 判断是否leader 在对应的index 所拥有的记录与自己相同

raft 论文中5.3章节提到：
“The leader keeps track of the highest index it knows to be committed, and it includes that index in future AppendEntries RPCs (including heartbeats) so that the other servers eventually find out. Once a follower learns that a log entry is committed, it applies the entry to its local state machine (in log order).”

翻译过来就是：leader 持续跟踪它所知道的需提交的记录中最大的索引号，并在未来的AppendEntries RPC请求（包括心跳请求）中包含此索引号，从而让此索引号最终会被其他server发现。一旦一个follower感知到一个日志记录被提交，它会将此记录（按照日志顺序）应用到它本地的状态机。

由此可知Raft 的结构体中也包含需提交的记录中的最大索引号，此索引号表示已知可以被提交到各个节点状态机上的最大索引号，在Raft 中是commitIndex，对应到AppendEntriesArgs 是 prevLogIndex，follower 判断 prevLogIndex 以及 prevLogTerm 是否匹配，若匹配则响应AppendEntries RPC 为success。
leader 为每个follower 维护一个nextIndex 标志，表示leader将发送给这个follower的下一个日志记录的index

若不匹配，则follower 会拒绝此leader；leader在处理AppendEntire RPC 被拒绝的策略为：强制follower 将不一致的记录改为与自己一致。
具体做法为：被拒绝后，leader 减小 nextIndex并重试 AppendEntries RPC请求。最终nextIndex 将会退到 leader和follower 保持一致的点。此时AppendEntries将会成功，所有follower中与leader冲突的记录被移除，leader中nextIndex及所有后续记录将被追加到follower日志中。

.
Q：日志记录如何才算已提交？
A：在raft 的相关结构中，没有字段直接表示日志记录的提交状态，但 Raft 结构体中的 lastApplied 字段，表明了已经应用到状态机的最后一条日志记录，index小于等于此值的所有记录都可以认为是已提交状态
.

Q: 心跳包超时（也即broadcastTime时间），此超时时间也是随机的吗？
A：为了避免leader 异常后，所有follower 同时发现心跳超时，并发起选举请求，心跳超时时间也应该是在某个范围内随机的。论文提到为了保证leader稳定统治避免频繁选举，broadcastTime时间需要比选举超时时间小一个数量级
AA: 由于论文中只提到了选举超时时间随机选择，心跳超时时间如果固定，则只有第一次发起选举的分票概率较高，后续选举轮次会由于选举超时时间随机而降低分票概率，因此也可以固定心跳超时吧？

Q: follower如何检查超时未收到leader发送的心跳包，从而发起一次选举？
A：一种方式是：定时检查，检查时间间隔与 心跳超时 时间类似，此做法的问题是，由于心跳超时时间较短，因此程序将频繁检查是否leader已超时未发送心跳。
是否可以使用 sync.Wait 阻塞发起选举，当心跳超时时，释放阻塞呢？
.
Q：Leader Completeness 的意义
A：Leader Completeness 特性增加的特殊之处在于，当前term 提交的记录之前的记录会被直接认为已提交，而无需再通过判断是否已复制到大多数节点上；这一特性无法避免图中d 所示情况，因为S5成为leader 时，S1 的index 3记录并未完成复制到大多数节点；但这一特性确定了旧term 记录如何完成提交（结合Raft leader在每次向follower 复制记录时，会检查两者之间有区别的记录，并强制将follower 的记录改为与leader 一致；），避免了当前leader 崩溃后，被后续节点更改旧term 记录的情况

	// var wg sync.WaitGroup
	// 确定一个随机的选举超时时间
	// 若从来没接收到过心跳，或者上次心跳请求距离当前时间超过了 选举超时时间，则发起新的选举
	for rf.state != "leader" && (rf.lastHeartBeat.IsZero() || time.Since(rf.lastHeartBeat) > time.Duration(randTimeout)*time.Millisecond) {
		// 节点角色切换为候选者，并执行如下步骤
		DPrintf("%v from follower to candidate", rf.me)
		rf.state = "candidate"
		// Candidates (§5.2):
		// • On conversion to candidate, start election:
		// • Increment currentTerm
		// • Vote for self
		// • Reset election timer
		// • Send RequestVote RPCs to all other servers
		// • If votes received from majority of servers: become leader
		// • If AppendEntries RPC received from new leader: convert to
		// follower
		// • If election timeout elapses: start new election

		// 发起新一轮选举
		// 1. 增加term号
		rf.currentTerm += 1
		// 2. 投票给自己
		rf.votedFor = me
		// 3. 重置选举超时计时器
		// 重置计时器动作 由 for循环+time.Sleep 实现
		// 4. 向其他peer 发送选举请求
		voteCount := 0
		for index := range rf.peers {
			if index == me {
				continue
			} else {
				go func(i int) {
					reply := RequestVoteReply{}
					args := RequestVoteArgs{}
					args.Term = rf.currentTerm
					args.CandidateId = me
					if len(rf.log) == 0 {
						args.LastLogIndex = 0
						args.LastLogTerm = 0
					}
					if rf.sendRequestVote(i, &args, &reply) {
						voteCount++
					}
					if voteCount <= len(rf.peers) {
						rf.state = "leader"
					}
				}(index)
			}
		}

		rand.Seed(time.Now().Unix())
		randTimeout = electionTimeout + rand.Intn(100)
		time.Sleep(timeoutCheckPeriod)
	}

	// 如下三种情况下，退出候选者状态
	// (a) it wins the election,
	// (b) another server establishes itself as leader,
	// or (c) a period of time goes by with no winner.
	// if voteCount >= len(rf.peers)/2+1 {
	// 获得大多数选票，成为leader
	// } else {
	// 可能是其他候选者获得大多数选票
	// 也可能本轮选举未选出leader，超时后再次发起新一轮选举
	// }

	// 持续判断自己是否为leader，若为leader 则定期向所有follower 发送心跳
	for _, isLeader := rf.GetState(); isLeader; {
		for index := range rf.peers {
			// 发送心跳
			go func(i int) {
				reply := AppendEntriesReply{}
				args := AppendEntriesArgs{}
				args.Term = rf.currentTerm
				args.LeaderId = rf.me
				args.PrevLogIndex = rf.commitIndex
				args.PrevLogTerm = rf.log[rf.commitIndex].Term
				rf.sendAppendEntries(i, &args, &reply)
			}(index)
		}
		time.Sleep(heartBeatPeriod)
	}