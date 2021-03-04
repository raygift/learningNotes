
## Raft 和 Paxos

我们所设的Paxos 实验和Raft 实验之间最大的区别是Paxos 实验基于Paxos，single-value 共识算法，不是Multi-Paxos。后者增加单轮 soft leader commits 特性，且通常被误认为就是Paxos。

基于Paxos 构建复制状态机最基本的思想是运行多Paxos 实例。为每个实例准备的消息随后被使用所属日志中的index“打标签”。Multi-Paxos 在此基础上进行了一些优化，同时也增加了算法的复杂性。对于6.824，我们认为优化性能带来的复杂度的增加对于教学没什么帮助。相反，我们要求学生基于Paxos 设计他们自己的简单协议来维护日志副本，并基于简单协议实现复制状态机。

作为对比，Raft 提供了完整的协议来构建一个分布式、持久化日志、包含如持久化、leader 选举、单轮共识（single-round agreement）以及快照等诸多优化的系统。相比Paxos，Raft 与Multi-Paxos 更像，无论是在特征集，性能还是复杂性方面。单Paxos 共识（即非Multi-Paxos）比Raft 概念上更加简单。

## 实现 Raft

