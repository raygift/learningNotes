lab 3的目标为使用lab 2 完成的Raft ，实现一个分布式的kv 存储，使用Raft 管理kv在多个节点上的多个副本。

client.go 需要完成向server.go 发送RPC，提交对kv 的操作，包括Put/Get/Append

server.go 需要处理client 发送的Put/Get/Append 操作，将对应的kv 使用Raft 完成提交后，将结果响应给client

common.go 定义了RPC 的参数结构

Q: Raft 日志中存储什么信息？kv 数据如何存储？
A: 日志存储client 发送来的操作（put/append/get），操作完成提交后，对应的kv 保存在本地


Q：kvraft/server.go 调用raft.go 完成cmd 的提交时，容易出现对于管道ApplyCh 的死锁问题

A：Raft 逻辑中cmd 在apply 之后，需向ApplyCh 管道发送应用完成的消息，同时在kvraft 的server 中，需要在执行完Start(cmd) 后从ApplyCh 中等待cmd 应用完成的消息，此时若raft.go 与 kvraft/server.go 都试图持有ApplyCh 的锁，则会造成死锁；具体情形为：
1. kvraft/server.go 持有锁，并执行Start(cmd)
2. raft.go 接收到Start(cmd)调用，将日志AppendEntries 给其他follower，判断完成commit 后，进行apply，然后在持有锁的情况下，尝试向ApplyCh 发送消息
3. 此时由于kvraft/server.go 对于ApplyCh 持有锁，并阻塞在 <-ApplyCh ，raft.go 试图向Applych 发送消息无法成功
4. raft.go 等待kvraft/server.go 放弃对于ApplyCh 的锁，而kvraft/server.go 由于raft.go 没有向ApplyCh 发送消息而阻塞无法释放锁，最终导致死锁