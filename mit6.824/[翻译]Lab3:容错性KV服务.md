## 简介

在本实验中你将使用在lab2 实现的Raft 库构建一个容错性KV 存储服务。你的key/value 服务会是一个复制状态机，由几个使用Raft 进行复制的key/value 节点构成。你实现的key/value 服务应该可以在大多数节点存活且可通信时，正常处理客户端的请求，不受其他异常节点或网络分区的影响。

服务支持三种操作：Put(key, value), Append(key, arg), 以及Get(key)。kv 服务提供你了一个简单的 kev/value 键值对数据库。Keys 和values 是字符串。Put() 将替换数据库中特定key 的值，Append(key, arg)在key 的值后面追加arg，Get() 获取key 对应的当前value。对于不存在的key 执行Get 会得到空字符串。对不存在的key 执行Append 会想执行Put 一样。客户端通过带有Put/Append/Get 方法的 Clerk 与服务进行通信。Clerk 管理与server 的RPC 交互。

你的服务必须向调用Clerk 的应用提供强一致性。这是我们对于强一致性的定义：如果每次调用一个方法，Get/Put/Append 应该表现得像系统中只存在一份状态一样，每次调用应该观察到先前顺序执行对于状态的改变。对于并行调用，返回值和最终状态必须表现的像按照某种顺序每次执行一个方法一样。如果调用在时间上是重叠的，那他们就是并发的，比如client X 调用Clerk.Put()，然后client Y 调用Clerk.Append()，然后client X 的调用完成返回。更进一步，一次调用必须能够感知到所有之前已完成调用的影响（即技术上要求线性化）。

强一致性对于应用更加方便，因为这意味着所有客户端看到的状态一样，且都可能看到最新的状态。在单机上提供强一致性比较简单。如果服务是多副本的就比较困难，因为所有server必须对并发请求选择相同的执行顺序，同时必须避免使用过期的状态向客户端进行返回。

本lab 有两部分，Part A 中，你将实现kv存储而无需考虑Raft 的日志会无限增长。Part B 中，你将实现快照（论文第7章），快照允许Raft 将旧的日志内容作废。

你应该重点阅读Raft 论文的第7、8章。为了拥有更广的视野，读一下Chubby，Paxos Made Live，Spanner，Zookeeper，Harp，Viewstamped Replication，以及[Bolosky et al.](http://static.usenix.org/event/nsdi11/tech/full_papers/Bolosky.pdf)

尽快开始

## 开始吧

我们在src/kvraft 中提供了一些骨架代码和测试用例。你需要编辑kvraft/client.go ，kvraft/server.go ，或许还包括 kvraft/common.go

执行如下命令进行测试，别忘了使用 git pull 拉取最新代码。

```shell
$ cd ~/6.824
$ git pull
...
$ cd src/kvraft
$ go test
...
$
```

## Part A: 不考虑日志压缩的Key/value 服务

