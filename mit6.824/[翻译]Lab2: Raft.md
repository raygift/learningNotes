

lab2 是具有容错能力的kv存储系统系列的第一课。



本lab将带你实现 Raft，这是一个被复制的状态机协议。在下一个lab中你将构建一个在raft基础之上的 kv服务。然后，便可以在多副本状态机上“shard”你的服务以获得更高的性能。



多副本（replicated）服务通过在多副本server上存储完成的状态信息（如数据）来实现容错。即使在部分服务器出现异常（崩溃或网络故障），副本机制也可以保证持续提供服务。多副本所面临的挑战是故障可能导致副本持有的数据产生不一致。



Raft将客户端请求放到序列中，称作 the log，同时确保所有副本服务器读取相同的log。每个副本按照log的顺序执行客户端请求，并将执行结果保存到本地日志；副本按照同样的顺序执行同样的请求处理，从而保证多副本之间的状态一致性。如果一个server异常单随后被恢复，Raft会将其log更新到最新状态。在 server中的一个majority存活并可以与其他server通信时，Raft会持续运行。如果不存在这样1个majority，Raft将无法运行，当条件满足时会再次启动。



在本lab中，将要的Raft作为一个拥有被分配的方法GO对象类型，旨在在大型服务中被当作一个module使用。一组Raft实例通过RPC与其他实例通信，保持副本log状态。你的Raft接口将支持指定的一些数字命令，也被称作log entries。这些 entries 被使用 索引号码编号。带有给定index的log entry 将最终被提交。提交时，Raft应该向引用module的服务发送 log entry来完成log entry的执行。



你应该遵循 extended Raft 论文的设计，特别需要注意图2.你将实现论文中的大部分内容，包括保存持久状态，并在node 失败时读取持久状态，然后完成重启。你无需实现论文第六章的cluster membership changes；对于论文第七部分 log compaction/snapshootting 的实现，将安排在下一个lab中。



这个[绪论][https://thesquareplanet.com/blog/students-guide-to-raft/]对你或许有用，它提供了关于并发的 locking 和 structure 建议。为了获得更宽广的视野，看一下Paxos，Chubby，Paxos Made Live，Spanner，Zookeeper，Harp，Viewstamped Replication，以及Bolosky et al.



此lab包括三部分。



## 协作机制

不能抄作业，可以跟同学讨论，不能看别人代码。不能公开自己的代码。



## 开始

继续在lab1的代码中开展工作。

在src/raft/raft.go中提供了骨架。

测试代码在src/raft/test_test.go中。

使用git pull git://g.csail.mit.edu/6.824-golabs-2020 6.824 获取最新代码

执行如下命令开始你的工作

```bash
$ cd ~/6.824
$ git pull
...
$ cd src/raft
$ go test
Test (2A): initial election ...
--- FAIL: TestInitialElection2A (5.04s)
        config.go:326: expected one leader, got none
Test (2A): election after network failure ...
--- FAIL: TestReElection2A (5.03s)
        config.go:326: expected one leader, got none
...
$

```



## 有关代码

在 raft/raft.go中添加代码实现Raft。文件中已有骨架代码和RPC的例子。



你实现的Raft必须支持如下接口，测试代码和最终的kv服务程序将会使用到。在raft.go的注释中有更多信息。

```go
// create a new Raft server instance:
rf := Make(peers, me, persister, applyCh)

// start agreement on a new log entry:
rf.Start(command interface{}) (index, term, isleader)

// ask a Raft for its current term, and whether it thinks it is leader
rf.GetState() (term, isLeader)

// each time a new entry is committed to the log, each Raft peer
// should send an ApplyMsg to the service (or tester).
type ApplyMsg
```



服务调用 Make(peers,me,...)来创建一个 Raft peer. 调用时传入的 peers 参数 是一个 Raft peer网络标识的数组，使用RPC。me 参数创建的peer 在 peer数组中的索引号。 Start(commond)  启动 Raft 向  多副本log中添加 commond 的操作。 Start() 执行后立刻返回，无需等待log append完成。服务期望你的代码为每个新提交的log 实体 向 Make()参数中的 applyCh 信道发送一个 ApplyMsg 消息。



raft.go 包括了发送 RPC 请求（sendRequestVote()) 和 接收RPC请求 (RequestVote())的样例代码。你的 Raft peers 应该使用 Go 包 labrpc（在 src/labrpc中）进行RPC的 转换。测试案例会告诉 labrpc 将RPC们延迟，重新排序，甚至直接丢弃，来模拟网络异常情况。你可以临时性的修改 labrpc ，但请确保Raft运行时使用原始的 labrpc ，交作业时会检查并作为打分依据。Raft 实例之间只能使用RPC交互；例如，不能通过共享的Go变量或者文件来通信。



后续的lab都是在本lab基础上，因此需要安排充分的时间来完成 “soild code”



## 2A部分

实现Raft的leader选举 和 心跳（没有log entries的appendEntries RPC）。2A部分的目标是针对一个要被选举的leader，如果没有异常失败则继续作为leader，如果旧的leader挂了，或者旧leader发出/接收的包丢了，则一个新的leader会进行接管。运行 go test -run 2A 来测试你的代码。



- 不能简单的直接运行Raft实现；而应该使用测试的方式运行，比如 go test -run 2A.
- 按照论文的 Figure2 进行工作。目前你需要关注的是 发送和接收 投票请求的RPC，针对选举相关server的规则，leader选举有关的状态，
- 为了leader 选举，向 raft.go 中的 Raft 结构体增加  Figure 2 状态。你还需要定义一个结构体用来 保存每个log entry 的信息。
- 完成RequestVoteArgs 和 RequestVoteReply 结构体。修改 Make() 来创建一个后台运行的 goroutine，当一段时间未收到其他peer的消息时，此goroutine发送 RequestVote 来触发 leader 选举。通过这种方式，一个peer将知道谁是leader，是否已经存在leader，或者自己成为leader。实现RequestVote() RPC hanlder 从而让 server们投票。
- 为了实现心跳，定义一个 AppendEntries RPC结构体（尽管你可能还并不需要所有的参数），并让leader定期向外发送此结构体。 实现一个 AppendEntries RPC Handler方法来将选举过期，避免当已经有一个server被选为leader时其他server也成为leader。
- 确保不同peer中的选举失效时间不会同时触发，或者所有peer都投票给自己从而导致无法选出leader。
- 测试案例要求 leader 发送心跳RPC请求不能超过每秒10次
- 测试案例要求你在旧的leader失败后5秒钟内选举出一个新leader（如果多数peer仍能通信）。然而请注意，leader选举可能需要很多轮才能避免 选举分票（当发生丢包或候选人不幸多次选择了同一个随机的退避时间）。为了保证即使需要多轮投票，选举仍能在5秒内完成，你必须选择一个足够短的 选举超时时间（心跳间隔也类似）。
- 论文中 5.2 章节提到了选举超时的范围为 150到300毫秒。只有当 leader 在每个150ms内连续发送多个心跳时，这个范围才起作用。因为测试案例限制了每秒只能发送10个心跳，你不得不使用大于论文中给出的 150～300ms 的超时时间，但也不要太大，因为太大可能导致无法在5秒中内完成选举。
- Go 的 rand 将会很有用。
- 你的代码需要能够定期执行，或者在延迟后及时执行。最简单的方法是创建一个拥有 time.Sleep() 循环的goroutine。不要使用 Go 的 time.Timer 、 time.Ticker，这俩方法很难正确使用。
- 阅读有关 [locking][https://pdos.csail.mit.edu/6.824/labs/raft-locking.txt] 和 [structure][https://pdos.csail.mit.edu/6.824/labs/raft-structure.txt] 的内容
- 如果代码无法通过测试，多读读 论文的 Figure 2；leader 选举的 完整逻辑分布在 此Figure的多个部分。
- 别忘了实现 GetState().
- 测试脚本 调用 你实现的 rf.Kill() 来彻底关闭一个实例。你可以调用 rf.killed() 来检查 Kill() 是否已经被调用。你应该会想要在每个循环中检查一次，从而避免已经挂了的 Raft实例打印出令人困惑的信息。
- 调试代码的一个好办法是在 peer 发送或接收信息是插入print语句，并用 go test -run 2A > out 来将信息收集到一个out文件中。然后通过查看 out文件的信息，排查哪些代码偏离了预期设计。util.go中的 DPrintf 是控制打印的开关，在调试不同问题时可能会有帮助。
- Go RPC 只发送结构体中用大写字母开头的字段。 sub-structures 命名时也必须用大写字母开头（比如，使用数组记录的 log 的字段）。 labgob 包会针对这事儿报出 warning，千万别忽略。
- 使用 go test -race来检测代码，并解决报告出的 争用。



通过测试时，将看到如下输出

```bash
$ go test -run 2A
Test (2A): initial election ...
  ... Passed --   4.0  3   32    9170    0
Test (2A): election after network failure ...
  ... Passed --   6.1  3   70   13895    0
PASS
ok      raft    10.187s
$
```



每个 passed 对应了5个数字，分别是：测试执行的时间(s)，Raft peer 的数量（通常是 3或5），测试过程中发送的RPC数量，RPC消息中总字节数，Raft 报告提交的log entries数量。你得到的值可能与上方不同。可以选择忽略这些数字，但它们可能会有利于对 你的代码发送的RPC数量进行完整性检查。在 lab 2、3、4中，如果执行所有test超过600秒，或者执行单个测试超过120秒，测试将失败。


## 2B 部分
**任务：实现追加新日志记录的leader 和follower 的代码，通过go test -run 2B 测试**

**提示：**
- 运行git pull 拉取最新实验代码
- 首要目标是通过 TestBasicAgree2B(). 先实现 Start()，然后编写代码实现通过 AppendEntires RPC 请求完成发送和接收新的日志记录，参考论文 Figure2.
- 需要实现选举约束（论文中第5.4.1章节）
- Lab 2B的前面几个测试失败的原因可能会是当leader 存活时仍保持重复的选举。寻找下选举定时器管理中的bug，或者不要在赢得选举后立马发送心跳
- 可能需要重复检查一些事件是否发生的循环，不要让循环持续不停的执行，因为这会降低代码运行速度导致测试失败。在每轮循环中使用Go的条件锁，或者插入time.Sleep(10* time.Millisecond)
- 为了完成后续lab ，代码写清楚一些。[structure](http://nil.csail.mit.edu/6.824/2020/labs/raft-structure.txt), [locking](http://nil.csail.mit.edu/6.824/2020/labs/raft-locking.txt) 和 [guide](https://thesquareplanet.com/blog/students-guide-to-raft/) 这些连接能给你一些建议。

代码运行太慢可能会导致测试失败。可以使用time 命令检查实际耗时和cpu耗时。典型输出如下：
```shell
$ time go test -run 2B
Test (2B): basic agreement ...
  ... Passed --   1.6  3   18    5158    3
Test (2B): RPC byte count ...
  ... Passed --   3.3  3   50  115122   11
Test (2B): agreement despite follower disconnection ...
  ... Passed --   6.3  3   64   17489    7
Test (2B): no agreement if too many followers disconnect ...
  ... Passed --   4.9  5  116   27838    3
Test (2B): concurrent Start()s ...
  ... Passed --   2.1  3   16    4648    6
Test (2B): rejoin of partitioned leader ...
  ... Passed --   8.1  3  111   26996    4
Test (2B): leader backs up quickly over incorrect follower logs ...
  ... Passed --  28.6  5 1342  953354  102
Test (2B): RPC counts aren't too high ...
  ... Passed --   3.4  3   30    9050   12
PASS
ok      raft    58.142s

real    0m58.475s
user    0m2.477s
sys     0m1.406s
$
```

"ok raft 58.142s" 表示Go 检测到2B 的测试用例实际执行了58.142 秒(wall-clock)。"user 0m2.477s" 表示CPU 耗时 2.477秒，也即执行代码的实际耗时（除去CPU 等待和休眠的时间）。如果你的代码运行2B 测试用例超过现实时间1分钟，或者CPU 时间5秒，就可能会因为太慢导致失败。这时需要检查休眠或等待RPC 超时的耗时，没有休眠或等待 条件或通道消息的循环，或者大量的RPC 请求。