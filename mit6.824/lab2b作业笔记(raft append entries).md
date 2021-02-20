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


Q：问题，对于reconnect 重新加入的节点，由于日志内容落后较多，leader 递减nextIndex 并完成同步后，cfg.log[peeridx] 对应的内容并未与rf.log 一致，从而导致仍会出现 "out of order"

```
2021/02/13 19:05:45 server 1 in term 2 ready to send append entries, log: [{0 <nil>} {1 101} {1 102} {1 103} {1 104} {1 105}]
2021/02/13 19:05:45 server 0 got append entries RPC, before this rf.log is : [{0 <nil>} {1 101}]
2021/02/13 19:05:45 server 0 got append entries : [{1 102} {1 103} {1 104} {1 105}], after this rf.log is : [{0 <nil>} {1 101} {1 102} {1 103} {1 104} {1 105}]
2021/02/13 19:05:45 appendentries from server 1 to peer 2 in term 2 success
2021/02/13 19:05:45 appendentries from server 1 to peer 0 in term 2 success
2021/02/13 19:05:45 appendentries from server 1 to peer 1 in term 2 success
2021/02/13 19:05:45 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:05:45 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:05:45 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:05:45 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:05:45 committed, applymsg {true 103 3}
2021/02/13 19:05:45 committed, applymsg {true 104 4}
2021/02/13 19:05:45 committed, applymsg {true 102 2}
2021/02/13 19:05:45 committed, applymsg {true 105 5}
2021/02/13 19:05:45 apply error: server 0 apply out of order 3, cfg.logs[0]=map[1:101 3:103]

```
A： 分析cfg.log[peeridx] 与raft 节点同步entry 的逻辑

在config.go 的start1 方法中，有独立的goroutine 循环接收 applyCh 中传递的消息，并将消息的command 应用到各个cfg.log 数组，当

```golang
	go func() {
		for m := range applyCh {
            // some other codes
				v := m.Command
                cfg.mu.Lock()
                // some other codes
				for j := 0; j < len(cfg.logs); j++ {
                    // some other codes
        			cfg.logs[i][m.CommandIndex] = v //

```



Q：bug，发现leader 后退nextIndex 时会退过头，原因是定时心跳attemptHeartbeat 中的回退逻辑 和 定时同步日志attemptAppendEntries 的回退逻辑都会判断nextIndex 是否需要回退，两部分代码在判断回退和执行回退时，没有控制在同一个原子操作中，导致多执行了回退

A：解决办法，考虑定时心跳与 定时同步日志 是否可以在一个方法中实现；

【视频课程Fault Tolerance 中提到】更新提交状态可以在定时心跳 或下一次客户端发送请求时，同步到其他follower

```
2021/02/13 19:20:18 leader 1 记录的 nextindex 与 follower 0 的len 不符 ，则 follower 拒绝该请求: len(rf.log): 2, args.PrevLogIndex: 4, args.PrevLogTerm: 1
2021/02/13 19:20:18 server 1 send appendEntries to peer 0 failed because follower disagree nextIndex
2021/02/13 19:20:18 leader 1 记录的 nextindex 与 follower 0 的len 不符 ，则 follower 拒绝该请求: len(rf.log): 2, args.PrevLogIndex: 4, args.PrevLogTerm: 1
2021/02/13 19:20:18 appendentries from server 1 to peer 2 in term 2 success
2021/02/13 19:20:18 appendentries from server 1 to peer 1 in term 2 success
2021/02/13 19:20:18 server 1 send appendEntries to peer 0 failed because follower disagree nextIndex
2021/02/13 19:20:18 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:20:18 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:20:18 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:20:18 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:20:18 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:20:18 server 1 in term 2 ready to send append entries, log: [{0 <nil>} {1 101} {1 102} {1 103} {1 104} {1 105} {1 106}]
2021/02/13 19:20:18 leader 1 记录的 nextindex 与 follower 0 的len 不符 ，则 follower 拒绝该请求: len(rf.log): 2, args.PrevLogIndex: 2, args.PrevLogTerm: 1
2021/02/13 19:20:18 server 1 send appendEntries to peer 0 failed because follower disagree nextIndex
2021/02/13 19:20:18 appendentries from server 1 to peer 1 in term 2 success
2021/02/13 19:20:18 leader 1 记录的 nextindex 与 follower 0 的len 不符 ，则 follower 拒绝该请求: len(rf.log): 2, args.PrevLogIndex: 2, args.PrevLogTerm: 1
2021/02/13 19:20:18 server 1 send appendEntries to peer 0 failed because follower disagree nextIndex
2021/02/13 19:20:18 appendentries from server 1 to peer 2 in term 2 success
2021/02/13 19:20:18 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:20:18 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:20:18 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:20:18 nCommitted failed , nd: 0 , cmd: 106
2021/02/13 19:20:18 server 1 in term 2 ready to send append entries, log: [{0 <nil>} {1 101} {1 102} {1 103} {1 104} {1 105} {1 106}]
2021/02/13 19:20:18 appendentries from server 1 to peer 1 in term 2 success
2021/02/13 19:20:18 server 0 got append entries RPC, before this rf.log is : [{0 <nil>} {1 101}]
2021/02/13 19:20:18 server 0 got append entries : [{1 101} {1 102} {1 103} {1 104} {1 105} {1 106}], after this rf.log is : [{0 <nil>} {1 101} {1 101} {1 102} {1 103} {1 104} {1 105} {1 106}]
```



Q：有关ApplyMsg 的发送逻辑

A：ApplyMsg 的注释明确了每个节点在日志提交后都需要发送apply 信号

// ApplyMsg
//   each time a new entry is committed to the log, each Raft peer
//   should send an ApplyMsg to the service (or tester)
//   in the same server.

## 惊天bug

由于TestFailAgree2B 始终无法通过阻塞了好久，终于在检查每个节点向AppleMsg 发送消息时的惊天bug（新goroutine 与当前goroutine 逻辑执行顺序），原始代码如下，原本是想使用goroutine 提高apply 的速度，但未使用条件锁；将代码去掉goroutine 改为顺序从上到下执行后test passed！

```golang
	if last < commit {
	// 发出完成提交的信号
	for i := 0; i <= commit-last; i++ {
		go func(num int) {
			if last+num < l {
				var am ApplyMsg
				am.CommandValid = true
				am.Command = rf.log[last+num].Command
				am.CommandIndex = last + num
				DPrintf("committed, applymsg %v", am)
				applyCh <- am
			}
		}(i)
	}
	rf.mu.Lock()
	rf.lastApplied = commit // 此处不会等到本地所有已经commit 未applied 的entry 全部apply 后才会执行，而是在创建goroutin 后就直接执行，导致未apply 就更新了lastApplied 
    rf.mu.Unlock()
}


// 修改后
	if last < commit {
		// 发出完成提交的信号
		for i := 0; i <= commit-last; i++ {
			num := i
			// go func(num int) {
			if last+num < l {
				var am ApplyMsg
				am.CommandValid = true
				am.Command = rf.log[last+num].Command
				am.CommandIndex = last + num
				DPrintf("committed, applymsg %v", am)
				applyCh <- am
				rf.mu.Lock()
				rf.lastApplied = last + num
				rf.mu.Unlock()
			}
			// }(i)
		}
    }


// 第二版修改后，每次只commit 一个entry
	num := 1
	if last < commit && last+num < l {
		var am ApplyMsg
		am.CommandValid = true
		am.Command = rf.log[last+num].Command
		am.CommandIndex = last + num
		DPrintf("server %v , rf.commmitIndex: %v, rf.lastApplied: %v, index %v has committed, applying %v", rf.me, rf.commitIndex, rf.lastApplied, am.CommandIndex, am)
		applyCh <- am
		rf.lastApplied = last + num
		DPrintf("server %v applied entry %v", rf.me, rf.lastApplied)
    }
    
```


### 命中AppendEntries RPC Handler 中的常见bug

在TestRejoin2B 中，leader[1] 网络故障后仍然继续接收到了客户端发来的103命令，而leader[2]上线，接收了新的命令102并同步给了server 3，随后leader[2]下线而leader leader1重新加入集群中，此时server3可当选为leader3，随后leader2 再依次重新加入到集群中，此过程中leader1 始终认为自己是leader，直到收到来自 leader 3的AE；leader1在处理leader3的AE 请求时，发现自己的term 小于args.term，因此将自己的term 更新为leader3 的term，但随后有进行了commitIndex 的更新，更新为leader3 的commitIndex：2，而leader1的日志index 2处的entry与leader3 并不相同，却仍被正常提交。

```
2021/02/17 17:52:43 server 1 got append entries RPC from leader 0, rf.commitIndex becomes args.LeaderCommit : 2
 args.PrevLogIndex:2, args.PrevLogTerm:2, rf.log[args.PrevLogIndex].Term :1, rf.log:[{0 <nil>} {1 101} {1 102} {1 103} {1 104} {1 104}]
 ```

由于是在leader3 发送的心跳中，leader1 的commitIndex 被更新，因此未能判断 rf.log[args.PrevLogIndex].Term != args.PrevLogTerm

解决办法：参考“Students' Guide to Raft”，无论是心跳还是entry复制的AppendEntries RPC，接收方处理时都需要判断 prevLogIndex 及其对应的term 是否一致


#### TestBackup2B 
- 执行过程：

1. 启动集群，共5个节点，server 3 当选为leader ，接收cmd1 并同步给其他四个server，完成entry 提交与应用。此时0～4节点的term 为1，commitIndex 和 lastApplied 为1，log为rf.log:[{0 <nil>} {1 4749218449257879937}]
2. 节点 0、1、2 失联，client 向 leader (server3) 继续提交50个cmd ，由于当前集群只有两个节点保持连接，这50个cmd 最终不会被提交。节点3、4会停留在currentTerm==1、prevLogIndex==51、 PrevLogTerm==1 的状态，commitIndex 与 lastApplied 都无法更新
   args.PrevLogIndex:51, args.PrevLogTerm:1, rf.log[args.PrevLogIndex].Term :1, rf.log:[{0 <nil>} {1 4749218449257879937} {1 1855858829347029437} {1 6064511845378019266} {1 4074025787349461737} {1 1729842773342124357} {1 5878115942153162009} {1 4203614180795530326} {1 6114285857444218633} {1 1130439025067070545} {1 1007127494374722794} {1 5579595292507767211} {1 3676984531283809221} {1 3887659469000788463} {1 7957412569407556047} {1 7895682276417979629} {1 8594231191404225457} {1 2549690124068658399} {1 2973777934259882249} {1 3012050633014163632} {1 1810965702010056505} {1 8751566600226137548} {1 9160771703858858766} {1 6353906890913485198} {1 6210393961438527180} {1 3762028379362083087} {1 6966691324463771496} {1 4903323047553128751} {1 1120842134720185172} {1 7749717370736614052} {1 4729761562443998552} {1 283456105679729302} {1 2913850700876262340} {1 4157090096405684118} {1 8677039700716686475} {1 7592545756362692868} {1 6800060660311626802} {1 5648653891655658204} {1 4470677237499788178} {1 1766524145547167077} {1 727099867604034299} {1 2456158311864587255} {1 9119455559026796568} {1 470374567693542523} {1 5255259275368189015} {1 2934747111327212759} {1 7915532199668273531} {1 3711251727697154998} {1 740864264745286843} {1 9215111598185959589} {1 5579070350775191715} {1 6752685207613195779}]
3. 随后仅存的两个节点（3、4）被失联，而此前失联的三个节点（0、1、2）重新连接，由于失联太久，三个节点已经经过了不止一个选举超时周期，因此term 号大于等于2，三者中一个server会当选为leader，假设是server1，此时server1 的term 假设为3；
4. server1 当选为第二任leader后，向0、2节点发送心跳，0、2节点更新自己的term 号为3，此时三个节点的 term==3、commitIndex 和 lastApplied 为1，log仍为rf.log:[{0 <nil>} {1 4749218449257879937}]
5. client 向当前leader（server1）发送50个cmd，server1 将entry 同步给server0、2，并完成提交和apply后，三个节点状态为：term==3，commitIndex 和 lastApplied 为51，log为
   
   args.PrevLogIndex:51, args.PrevLogTerm:3, rf.log[args.PrevLogIndex].Term :3, rf.log:[{0 <nil>} {1 7255189831682426249} {3 1583025306301594904} {3 902053348073043401} {3 781651668790491352} {3 1481599390465983571} {3 6463476479807115367} {3 8445412155880125743} {3 3471296965458108735} {3 6285924962173665866} {3 8365310424698600027} {3 3111502500966660219} {3 1148691511556135173} {3 4650775382778241028} {3 3543962724978541679} {3 4247423719389723516} {3 7436197308018582876} {3 8675337145838580613} {3 1748883033541615377} {3 5982639908681908211} {3 7033660935220056188} {3 7390678216211272870} {3 6207345065176575407} {3 5245382235122044433} {3 1493875603167129306} {3 5884292581912523003} {3 2790468796304035943} {3 9045435393978096430} {3 1676697489478019583} {3 1485933365810846903} {3 5510369969299391076} {3 4164061923305869616} {3 2750995329402748042} {3 5264803890640666864} {3 4512312782679884109} {3 3826400229729880945} {3 8978323866737275265} {3 8650148588689732027} {3 8547189959292180997} {3 5777787251298354922} {3 1564560627618828286} {3 3681585990040709379} {3 324471390369427067} {3 6249393578915949163} {3 4053384573989069142} {3 4089426761816014372} {3 1629629138807020182} {3 8198778611903389279} {3 1927401585102344364} {3 4186171590431031024} {3 3231534636844248426} {3 934944236632477893}]
6. server0 被失联，此时集群只剩leader 1和follower 2，client再次发送50个cmd，server1 和server 2 的状态更新为 term==3、commitIndex 和 lastApplied 为51，log长度为101
    
    args.PrevLogIndex:101, args.PrevLogTerm:3, rf.log[args.PrevLogIndex].Term :3, rf.log:[{0 <nil>} {1 7255189831682426249} {3 1583025306301594904} {3 902053348073043401} {3 781651668790491352} {3 1481599390465983571} {3 6463476479807115367} {3 8445412155880125743} {3 3471296965458108735} {3 6285924962173665866} {3 8365310424698600027} {3 3111502500966660219} {3 1148691511556135173} {3 4650775382778241028} {3 3543962724978541679} {3 4247423719389723516} {3 7436197308018582876} {3 8675337145838580613} {3 1748883033541615377} {3 5982639908681908211} {3 7033660935220056188} {3 7390678216211272870} {3 6207345065176575407} {3 5245382235122044433} {3 1493875603167129306} {3 5884292581912523003} {3 2790468796304035943} {3 9045435393978096430} {3 1676697489478019583} {3 1485933365810846903} {3 5510369969299391076} {3 4164061923305869616} {3 2750995329402748042} {3 5264803890640666864} {3 4512312782679884109} {3 3826400229729880945} {3 8978323866737275265} {3 8650148588689732027} {3 8547189959292180997} {3 5777787251298354922} {3 1564560627618828286} {3 3681585990040709379} {3 324471390369427067} {3 6249393578915949163} {3 4053384573989069142} {3 4089426761816014372} {3 1629629138807020182} {3 8198778611903389279} {3 1927401585102344364} {3 4186171590431031024} {3 3231534636844248426} {3 934944236632477893} {3 4797926847303419382} {3 2676641368782196979} {3 8802909192736657729} {3 7509116350526251607} {3 7773086379511580912} {3 1286746188963029064} {3 7715158084796294028} {3 5198317416519733543} {3 2053571276061514831} {3 4626532358900782112} {3 2437482068712889952} {3 6905350158292031509} {3 3815671157635492279} {3 2582883636719616753} {3 224001108387984045} {3 6833203248968679705} {3 2193946223861269574} {3 6279718079826203415} {3 7317927032162008860} {3 9038956878643194642} {3 5798078715236093194} {3 5636264690910321781} {3 645866513750538024} {3 3767944519174867260} {3 6358633205469903048} {3 1915704084620560681} {3 420047204800081914} {3 2840096584766488906} {3 5487795007030292709} {3 1462468548205523955} {3 4784575153694439012} {3 3418606976400335509} {3 7760804037662182531} {3 5593626818930868727} {3 2071838751041469833} {3 6891576589785116571} {3 9215925486854448503} {3 3451424825540910912} {3 6217909920614682195} {3 3168043065986078815} {3 2917853433562801242} {3 8058491645939236407} {3 2952166591146939821} {3 1492601754616605468} {3 1066758391807774912} {3 5564670125128906530} {3 1595141481206097965} {3 6836710438258328923} {3 2575380669326945440} {3 8049745831267020561}]
7. server1和server2也被失联，此时全部节点均失联；然后重新连接server 3、4以及server0（注意server 0 为第二任leader下的成员），三个节点的状态分别为：
   1. server3，第一任leader，自认为仍是leader，因此term 号未发生变化仍为1，currentTerm==1、prevLogIndex==51、 PrevLogTerm==1
   2. server4，由于较早与集群失联，term 号由于多次选举超时不断变大，假设currentTerm==4，而prevLogIndex==51、 PrevLogTerm==1 与server3 一样
   3. server0，第二任leader 下的follower，失联后term号可能会由于选举超时增大，但失联时间较短因此小于server4 的term号，假设为currentTerm==3，而prevLogIndex==51、 PrevLogTerm==3，commitIndex 和 lastApplied 为51

8. 此时三个存活节点0、3、4中，只有server0 可能当选为leader，因为它持有更多已提交的entry。但由于其term 较小，需要在其他两个server 发送过RequestVote RPC 更新term 后，才能成功发送RequestVote 并当选
   **但观察发现server0 迟迟不发起新的选举，其他两个注定无法当选的server 反倒频频选举超时term+1。** 怀疑是选举超时的时间标志更新逻辑问题，在不是心跳请求的RequestVote RPC Handler 中是否要更新时间标志？
   
   A：在 Students' Guide to Raft 中提到，在选举超时时间内未收到当前leader 的AE RPC 或当前节点未投票给其他candidate 时，会重置选举超时计时器，因此在RequestVote RPC Handler 中，如果投票给了candidate，还是要更新时间标志timeMark；但在心跳请求的处理中（AppendEntries RPC Handler中），若发送者不是当前节点认可的leader 则不应该重置选举超时（但仍需要更新term+1 以及更新自己角色成为follower，且重置votedFor），重置选举超时以及更新谁是leader 的动作在下一轮此leader 发送来AE 时处理，避免其他leader 发来一个AE 后异常退出，而follower 未能及时发现旧leader 已经下线。

   对于失联的leader再次加入集群中时，由于日志落后较多，在尝试发送一次RequestVote 失败后会很快再次增加term 号发起新一轮选举，这就导致了拥有更加up-to-date 日志的节点始终处于 接收 落后leader 发送来的更大term 并更新自己term的状态

9. 终于！！！发现分析方向上昏头了！！！本测试案例目标是“Test (2B): leader backs up quickly over incorrect follower logs ...”，只要做nextIndex 回溯的优化就可以了...但之前发现的leader rejoin 问题仍然是常见的问题之一，RequestVote Handler 的逻辑中几个判断条件和属性更新的顺序比较重要。

10. 优化backtracking ，参考Students' Guide to Raft，优化nextIndex 回退
11. 然并软，测试案例执行失败的原因并不是回退太慢，而是10秒中左右没有能够选举出新的leader。大脸来的太快...

12. 问题回到了8.
13. TestBackup2B 本质上似乎一个older leader rejoin 的场景，但在rejoin 后尚未选出leader 之前，client 就尝试提交新的cmd，将提交cmd 代码去掉后，可以正常选出leader ，说明提交新cmd 的逻辑影响了leader 的选举




### 心跳超时检测频率

不一定与发送心跳的频率类似，使用选举超时检测频率即可

### kill

在 cfg.cleanup() 之后，紧接着使用make_config 创建新集群，节点的term 会延续cleanup 之前的term ，直到make_config 执行完成，此时打印出来的日志中可能仍会出现上一次make 的节点的相关行为日志，可根据term 、RPC 请求是否成功 来进行判断，避免影响本测试案例的分析

### RPC Handler 中更新term

在RPC handler 中接收到大于rf.currentTerm的 args.Term 时，无论是否接收日志（ApeendEntries RPC Handler中）或投票（RequestVote RPC Handler中），都需要更新本节点的term 号，因此需要在判断其他执行逻辑之前，首先记录下当前rf.currentTerm 和 args.Term，在结束处理前将rf.currentTerm =Max(rf.currentTerm, args.Term)



