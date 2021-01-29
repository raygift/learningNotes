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
.
Q：什么时候需要用锁？
A：
1. 不同的goroutine对同一个共享变量进行修改或读取操作
2. for循环忙等待某事件发生，使用条件锁时，需要申请互斥锁
3. for 循环中 锁的释放如果用 defer mu.Unlock()，需注意只有在for循环退出时才会释放锁
4. RPC请求前需要先释放锁，避免死锁，也避免RPC请求耗时过长导致长期持有锁

Q：什么时候用channel？
1. 作为生产者消费者时
2. 不建议用来唤醒其他goroutine 

Q：什么时候用条件锁？
1. 不同goroutine 之间唤醒时

## Test2A 测试现象及分析

#### 现象1：运行测试案例发现选举过程中出现两个leader 的情况，且两个leader 都成功当选并成功发送心跳
如下，
```
2021/01/28 15:25:54 check one leader 1
2021/01/28 15:25:55 term 0 heartbeat timeout, server 2 from follower becomes candidate
2021/01/28 15:25:55 term 0 heartbeat timeout, server 1 from follower becomes candidate
2021/01/28 15:25:55 server 2 voted itself
2021/01/28 15:25:55 server 1 voted itself
2021/01/28 15:25:55 server 2 request vote, but server 2 has no more vote chance
2021/01/28 15:25:55 server 1 request vote, but server 1 has no more vote chance
2021/01/28 15:25:55 server 2 win the election, got 2
```
* 分析：
正常一个term 中应该只有一个节点能获得大多数节点选票，怀疑是RequestVote 的处理逻辑有问题
果然，发现 attemptElection 中发送拉票请求时，sendRequestVote(server,args,reply) 的server 参数填的是 rf.me，也就是向自己发送3次投票请求...奇怪的是还能够获得两票，应该是计票的锁没控制好


### 问题：AppendEntries 处理逻辑中，已经当选为leader 的节点是否会因为 其他leader 的心跳而放弃leader 角色？
分析：若此leadder 与其他节点处于不同网络分区，导致自己虽然状态为leader ，但实际已有其他leader 上位并领导其他节点，因此本节点重新加入集群后需要退位，切换为follower；但若term 相同的leader 发来心跳，则应该是同一批竞争leader 的请求，

### 现象2: 不一定是term 最大的candidate 当选leader
低term 的节点当选leader 后，发送心跳给高term 的节点，会被拒绝并返回较高term 号，leader 再将自己term 更新为较高的term 号，下次发送心跳时便可成功

```
2021/01/28 11:29:21 someone check server 0 state, i'm follower in term 1
2021/01/28 11:29:21 someone check server 1 state, i'm follower in term 1
2021/01/28 11:29:21 someone check server 2 state, i'm candidate in term 2
2021/01/28 11:29:21 term 1 heartbeat timeout, server 0 from follower becomes candidate
2021/01/28 11:29:21 server 0 voted itself
2021/01/28 11:29:21 server 0 change to leader, under u knee and follow my command !
2021/01/28 11:29:21 server 0 in term 2 ready to send heartbeat
2021/01/28 11:29:21 server 1 stay as follower
2021/01/28 11:29:21 server 2 change to follower
```

### 其他注意点：
1. 若单独一个goroutine 对选举超时进行判断（不与由follower->candidate并发起选举合并到一个goroutine），则判断选举超时后并发送RequestVote 获得足够多选票后，在切换状态为leader前，需再次确定自身角色仍为candidate，原因是自身选举超时->再次发起选举并获得足够多选票 这一过程中，可能存在其他节点成功当选为leader，并已经成功发送出心跳，此时再发起选举请求，term与leader 一致，因此仍可能当选。但实际上就不应该再次发起选举

## TestReElection2A
-  若某节点与其他节点断开网络连接，则其可能会不断尝试增加term发起选举，导致其term号非常大，若此节点再次加入集群，判断是否可以当选为leader时，lastlogindex的比较就非常重要
-  代码通过TestInitialElection2A 后，再测试TestReElection2A，出现的现象是leader 仍在定时发送心跳，但两个follower 切换为candidate 后，无法获得多数选票，导致不断选举超时term 号不断增大

## re election 测试现象
一段时间后，两个节点停滞在candidate 状态未能判断选举超时，另外一个节点应该是被特意下线了，可能导致了剩下的两个节点都无法获得足够多的选票。
* 分析：正常情况下存活的两个节点应该可以继续判断心跳超时和选举超时，并在不同时间发起选举，先发起选举的节点更可能获得两张选票从而当选为leader
1. 问题排查：选举超时判断逻辑：应该没问题
<!-- 2. 问题定位：发现attemptElection 中只有在投票请求全部得到响应（无论投票与否）时，才会继续判断为赢得选举，而当有节点故障时，无法满足响应数==节点总数，导致在 cond.Wait()持续等待 -->
3. attempElection 和 attemptAppendEntries 两个方法中，并发向所有节点发送RPC 请求前的参数实例化，错误的放到了for 循环之外，导致每次发送RPC 都是使用的同一组args 和reply，导致reply 在其他goroutine 中被修改后，影响了其他goroutine 对RPC 响应的判断
4. RequestVote 中节点给自己投票的逻辑有问题，导致在已经给自己投过票之后，无法再次获得自己的选票；RequestVote 逻辑中应该首先判断候选人是不是自己，若是自己直接投票
5. 进一步发现，sendRequestVote 调用的RPC 请求，向自己发送RequestVote RPC 请求失败，并不是未获取到选票：节点被disconnect 后再次connect 到集群，无法处理RPC 请求了
6. 查看config.go 中的disconnect 与connect 逻辑，发现只是重置了标志位
7. 注意到labrpc.go 中提到 除非服务端异常或网络问题，否则Call() 可以保证返回true
8. 重新阅读lab2A 作业要求，注意到需要实现一个rf.Kill() 方法，config.go 中的crash1() 和cleanup() 调用了Kill 来对rf 的指定节点进行清理，而2A 两个测试案例都有defer cfg.cleanup() ，怀疑跟Kill 逻辑有关，未果
9. 通过打印 labrpc.go 中 endname对应的network 的 enabled 状态，发现在节点重新加入集群后，再次发送RPC 使用的endname 与config connect 时enable 的endname 不一样，使用的endname 对应的network enabled状态仍为false，怀疑为重新加入集群的节点 索引发生了变化(rf.me)

``` golang
	// 5
    // cfg.connect((leader2 + 1) % servers) 后，
	ok := rf.peers[server].Call("Raft.RequestVote", args, reply)
```

```golang
	// 2
	for voteForMeCount <= len(rf.peers)/2 && votedCount < len(rf.peers) {
		cond.Wait()
	}
```

```
2021/01/28 15:59:08 someone check server 0 state, i'm leader in term 4
2021/01/28 15:59:08 someone check server 1 state, i'm follower in term 4
2021/01/28 15:59:08 someone check server 2 state, i'm follower in term 4
2021/01/28 15:59:08 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:08 term 4 heartbeat timeout, server 1 from follower becomes candidate
2021/01/28 15:59:08 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:08 term 4 heartbeat timeout, server 2 from follower becomes candidate
2021/01/28 15:59:08 server 2 voted itself
2021/01/28 15:59:08 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:08 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:08 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:08 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:08 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 2 election timed out
2021/01/28 15:59:09 server 2 voted itself
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 1 election timed out
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:09 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:10 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:10 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:10 someone check server 2 state, i'm candidate in term 6
2021/01/28 15:59:10 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:10 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:10 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:10 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:10 server 0 in term 4 ready to send heartbeat
2021/01/28 15:59:10 someone check server 1 state, i'm candidate in term 6
2021/01/28 15:59:10 someone check server 2 state, i'm candidate in term 6
```

```
// 9 endname不是enabled状态

connect(2)
2021/01/29 00:07:28 ______________________endname nJhxkaF556AQMH7go9iz was true
2021/01/29 00:07:28 ______________________endname 7GzCbjTSuAdZmapddZ0w was true
2021/01/29 00:07:28 ______________________endname dTghHpE5iMC87NDjnt5e was true
2021/01/29 00:07:28 ______________________endname 7GzCbjTSuAdZmapddZ0w was true
2021/01/29 00:07:28 check one leader after 2 arises

2021/01/29 00:07:28 time.AfterFunc endname: EhtqotWY6vRMx7SuEMH5 enabled: false servername: 2 server: &{{0 0} map[Raft:0xc000021cc0] 17}
2021/01/29 00:07:28 time.AfterFunc endname: P3zBmhCvGod8ydghDsgG enabled: false servername: 1 server: &{{0 0} map[Raft:0xc000021bc0] 24}

2021/01/29 00:07:29 time.AfterFunc endname: OiW_Y6aHRxYgf8WbrmOh enabled: false servername: 0 server: &{{0 0} map[Raft:0xc000021ac0] 26}
2021/01/29 00:07:29 someone check server 0 state, i'm candidate in term 7
2021/01/29 00:07:29 someone check server 2 state, i'm candidate in term 7
2021/01/29 00:07:29 enabled: false servername: 2 server: &{{0 0} map[Raft:0xc000021cc0] 17}
2021/01/29 00:07:29 enabled: false servername: 1 server: &{{0 0} map[Raft:0xc000021bc0] 24}
2021/01/29 00:07:29 enabled: false servername: 0 server: &{{0 0} map[Raft:0xc000021ac0] 26}
2021/01/29 00:07:29 time.AfterFunc endname: P3zBmhCvGod8ydghDsgG enabled: false servername: 1 server: &{{0 0} map[Raft:0xc000021bc0] 24}
2021/01/29 00:07:29 enabled: false servername: 2 server: &{{0 0} map[Raft:0xc000021cc0] 17}
2021/01/29 00:07:29 enabled: false servername: 1 server: &{{0 0} map[Raft:0xc000021bc0] 24}
2021/01/29 00:07:29 enabled: false servername: 0 server: &{{0 0} map[Raft:0xc000021ac0] 26}
2021/01/29 00:07:29 time.AfterFunc endname: EhtqotWY6vRMx7SuEMH5 enabled: false servername: 2 server: &{{0 0} map[Raft:0xc000021cc0] 17}
2021/01/29 00:07:29 time.AfterFunc endname: P3zBmhCvGod8ydghDsgG enabled: false servername: 1 server: &{{0 0} map[Raft:0xc000021bc0] 24}
2021/01/29 00:07:29 time.AfterFunc endname: EhtqotWY6vRMx7SuEMH5 enabled: false servername: 2 server: &{{0 0} map[Raft:0xc000021cc0] 17}
2021/01/29 00:07:29 time.AfterFunc endname: 7GzCbjTSuAdZmapddZ0w enabled: false servername: 2 server: &{{0 0} map[Raft:0xc000021cc0] 17}
2021/01/29 00:07:29 enabled: false servername: 2 server: &{{0 0} map[Raft:0xc000021cc0] 17}
2021/01/29 00:07:29 enabled: false servername: 1 server: &{{0 0} map[Raft:0xc000021bc0] 24}
2021/01/29 00:07:29 enabled: false servername: 0 server: &{{0 0} map[Raft:0xc000021ac0] 26}
2021/01/29 00:07:29 time.AfterFunc endname: dTghHpE5iMC87NDjnt5e enabled: false servername: 2 server: &{{0 0} map[Raft:0xc000021cc0] 17}
2021/01/29 00:07:29 wait for the reply not ok , svcMeth: Raft.RequestVote, args: &{6 0 0 0}, reply: &{0 false}, endname: dTghHpE5iMC87NDjnt5e, rep: {false []}
2021/01/29 00:07:29 RPC ERROR!!!!!!!!!!!!!!!!!!!!!!!!!
2021/01/29 00:07:29 server 0 send RPC vote request to server 2 failed
2021/01/29 00:07:29 enabled: false servername: 0 server: &{{0 0} map[Raft:0xc000021ac0] 26}
2021/01/29 00:07:29 enabled: false servername: 1 server: &{{0 0} map[Raft:0xc000021bc0] 24}
2021/01/29 00:07:29 enabled: false servername: 2 server: &{{0 0} map[Raft:0xc000021cc0] 17}
2021/01/29 00:07:29 time.AfterFunc endname: P3zBmhCvGod8ydghDsgG enabled: false servername: 1 server: &{{0 0} map[Raft:0xc000021bc0] 24}
```