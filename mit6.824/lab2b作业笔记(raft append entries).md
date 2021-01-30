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
.
