
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
   2. 节点只能根据自己记录的rf.LeaderId 来判断是否RPC 请求来自当前节点。因此在一个节点收到新当选leader 的第一个AppendEntires 请求时，由于leader 的term 号码更大，将自己的term 更新的同时，需要记录下leaderId，在之后的RPC 处理中，首先对比发送者是否与自己记录的rf.LeaderId 一致，若一致，则重置选举超时计时器；若不一致，则需要先对比Term 判断是否为新leader 当选，若term 号较小或者不满足check 2，则拒绝AppendEntires 请求，此时也不能重置选举超时计时器。

2. 节点收到RequestVote RPC时，若经过判断决定投票，若不是follower 则同时放弃leader 或candidate 角色，同时重置选举超时计时器；若决定不投票，则不能重置选举超时计时器。

总结，除了上述两种情况，其他时候处理RPC 时都不应该重置计时器，比如：
    - 收到当前leader 之外的其他节点发送来的AppendEntires RPC 请求时，不重置计时器。（旧leader 发送的心跳不发生计时器重置）
    - 收到节点发送的RequestVote RPC请求，且未投票给发送者

Q：收到AppendEntries RPC 请求，且arg.Term 大于rf.currentTerm 时，节点需如何处理？

A：收到AppendEntries RPC后，需要做如下几个操作

1. 判断Term大小，若args.Term 更大，则更新rf.currentTerm；对于剩余几个动作（更新节点角色状态，更新rf.LeaderId，重置选举超时计时器）分析如下：
   1. 可以更新节点状态，因为节点获得了大多数选票后，才成为的leader，即使是之前的旧leader 由于意外情况更新了term 号，也会先变为candidate ，赢得选举之前无法发送更大term号的RPC请求；
   2. 可以更新rf.LeaderId，因为发送
   3. 不可以重置选举超时计时器，因为尚未判断发送请求的节点是否为当前leader，args.LeaderId ?= rf.leaderId
2. 当判断