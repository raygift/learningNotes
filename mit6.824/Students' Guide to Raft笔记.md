
Q：收到RPC请求后，什么情况下需要重置选举超时计时器

A：论文给出的节点需要转换为candidate的情况为：
    1. 经过选举超时时间，没有接收到当前leader 发送的AppendEntires RPC 请求
    2. 经过选举超时时间，没有对candidate进行投票

转变为candidate 需要选举超时计时器到期；

需要选举超时机器器到期，则不应该随意重置计时器；

转变为candidate 后，肯定需要重置选举超时计时器。

因此除了上述情况外，其他情况都不应该转换为candidate，也就是不要轻易重置选举超时计时器，而是等它自然到期

原文强调了收到AppendEntries RPC 需要来自“当前leader”，以及收到RequestVote RPC 且“没有投票”时，需要让超时计时器自然过期，因此这两种情况不需要重置计时器。

1. 节点收到AppendEntires RPC 时，需要判断是否来自“当前leader ”。如何确定当前leader 就很关键；
   1. 当前leader 是指集群当前真正的leader，还是接收RPC 请求的节点记录的rf.LeaderId？
   2. 节点只能根据自己记录的rf.LeaderId 来判断是否RPC 请求来自当前节点。因此在一个节点收到新当选leader 的第一个AppendEntires 请求时，由于leader 的term 号码更大，将自己的term 更新的同时，需要记录下leaderId，再重置选举超时计时器；若发送RPC请求的为旧leader，则term号会一定小于当前节点term，则拒绝AppendEntires 请求，此时也不能重置选举超时计时器

2. 节点收到RequestVote RPC时，若经过判断后决定投票，当节点不是follower 时还需要放弃leader 或candidate 角色，同时重置选举超时计时器；若决定不投票，则不能重置选举超时计时器。

综上所述，除了上述两种情况，其他时候处理RPC 时都不应该重置计时器，比如：
    - 收到当前leader 之外的其他节点发送来的AppendEntires RPC 请求时，不重置计时器。（旧leader 发送的心跳不发生计时器重置）
    - 收到节点发送的RequestVote RPC请求，且未投票给发送者

____


Q：除了处理RPC 请求，还有哪些情况需要重置选举超时计时器？

A：

在lab2 作业描述中，建议使用time.Sleep 来实现超时计时器，而非timer。则实现具体包括两点：

1. 使用lastTimeMark 记录上次计时器重置的时间点
2. 使用time.Sleep(超时时长T) 来控制检查是否超时的逻辑，每隔T 时长就执行一次超时检查，判断当前时间与 lastIimeMark 的时间间隔是否超过超时时间T

由于未使用定时器，对于计时器的重置，除了Students' Guide to Raft 的提到的RPC 处理的情况，还需要在检查出超时后，手动重置计时器
1. 检查出心跳超时后
2. 检查出选举超时后

____

Q：收到AppendEntries RPC 请求，且arg.Term 大于rf.currentTerm 时，节点需如何处理？

A：收到AppendEntries RPC后，需要做如下几个操作

   1. 判断Term大小，若args.Term 更大，则更新rf.currentTerm；
   
   2. 除此之外还有几个动作（更新节点角色状态，更新rf.LeaderId，重置选举超时计时器）分析如下：
      1. 可以更新节点状态为follower，因为节点获得了大多数选票后，才成为的leader，即使是之前的旧leader 由于意外情况更新了term 号，也会先变为candidate ，赢得选举之前无法发送更大term号的RPC请求；
      2. 可以更新rf.LeaderId为args.LeaderId，因为发送AppendEntires RPC 的节点只可能是当前的leader 和旧的leader，而旧leader 的term 较小请求会被拒绝
      3. 可以重置选举超时计时器，因为已经更新leaderId ，且已经转化为follower，需要更新超时计时器，以便后续判断心跳超时的情况

____

