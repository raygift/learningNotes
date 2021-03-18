lab 3的目标为使用lab 2 完成的Raft ，实现一个分布式的kv 存储，使用Raft 管理kv在多个节点上的多个副本。

client.go 需要完成向server.go 发送RPC，提交对kv 的操作，包括Put/Get/Append

server.go 需要处理client 发送的Put/Get/Append 操作，将对应的kv 使用Raft 完成提交后，将结果响应给client

common.go 定义了RPC 的参数结构

Q: Raft 日志中存储什么信息？kv 数据如何存储？
A: 日志存储client 发送来的操作（put/append/get），操作完成提交后，对应的kv 保存在本地

