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

key/value servers（简称“kvservers”）中的每个server 都将分配一个Raft 节点。Clerks 向被分配了Raft leader 节点的server 发送Put(), Append(), 以及Get() RPC 请求。kvserver 代码 将 Put/Append/Get 操作提交给Raft，从而Raft 日志拥有了Put/Append/Get 操作序列。所有的kvserver 按照Raft 日志的顺序执行操作，将操作应用到它们的 kv 数据库上；目的是让server 维护 kv 数据库的一致性副本。

Clerk 有时不知道哪个kvserver 是Raft leader。如果Clerk 向错误的kvserver 发送了RPC 请求，或者请求无法到达kvserver，Clerk 需要向另一个kvserver 重新尝试发送。如果kv 服务向它的Raft 日志提交了操作（并随后将操作应用到kv 状态机），leader 通过RPC 向Clerk 返回执行结果。如果操作未能完成提交（例如，如果leader 被新选举出的leader 取代），server 将报告错误，Clerk 选择另外一个server 进行重试。

你实现的kvservers 不应该直接通信，而应该只通过Raft 互相交互。对于所有Lab 3的内容，始终能够通过Lab 2 的测试用例是必须的前提条件。

____
**任务**

本实验第一个任务是在没有消息丢失，没有崩溃节点的情况下实现kv server。

你需要在client.go 的 Clerk Put/Append/Get 方法里添加RPC 发送的代码，并实现 server.go 的PutAppend() 和Get() RPC 请求的处理逻辑。处理逻辑中需要使用Start() 向Raft 的日志键入一个 Op ；你需要补充server.go 中定义的 Op 结构体，从而能够描述一个 Put/Append/Get 操作。每个server 需要执行Raft 提交给他们的 Op 命令，即出现在 applyCh 中的命令。RPC handler 应该注意到Raft 何时提交 Op，然后返回RPC 响应给请求方。

当你稳定的通过测试用例中的第一个“One client”测试时，才能算完成了本任务。

- 提示1：在调用Start() 之后，你的kvservers 将需要等待Raft 完成共识。已经完成共识的command 会被发送到 applyCh 里。你的代码需要在PutAppend() 和Get() handlers 使用Start() 向Raft 日志提交command期间，持续读取 applyCh。注意kvserver 和Raft 库之间的死锁。
- 提示2：可以向 ApplyMsg 中添加新字段，也可以向RPC 中添加字段，例如AppendEntries。
- 提示3：如果不属于网络分区中大多数节点，kvserver 不应该完成Get() RPC 处理（因此它不会提供过期数据）。一个简单的解决方案是将每个Get()（Put 和Append 也类似）写入Raft 日志。你不必实现论文第8章的只读操作优化。
- 提示4：最好从开始就加锁，因为避免死锁的要求有时会影响到整体代码设计。使用 go test -race 来检查代码是没有竞争的（race-free）
____

现在，你需要继续修改代码来处理网络和server 异常的情况。你面临的一个问题是Clerk 可能会多次发送一个RPC 请求直到它找到积极响应RPC 的kvserver。如果一个leader 在向Raft 日志提交entry 后紧接着异常，Clerk 可能就收不到响应，此时会尝试向另一个leader 重复发送请求。每次调用Clerk.Put() 或 Clerk.Append() 都应该在一次操作中得到响应，因此你需要确保重复发送不会导致server 重复执行两次。

____
**任务**

增加代码处理异常情况，应付重复的 Clerk 请求，包括同一个term中Clerk 向kvserver leader 发送请求，等待响应超时后在下一个term 中重复向新leader 发送请求。这个被重复发送的请求应该只执行一次。你的代码需要通过 go test -run 3A 测试。

- 提示1：你的代码需要处理这种情况：leader 为Clerk 的RPC 请求调用了 Start，但在此请求提交到日志之前失去了leader 的身份。此种情况下你应该安排Clerk 再次发送请求给其他server，直到找到新的leader 为止。一种实现方式是server 通过Start() 在相同的index 上返回了不同的cmd，或者Raft 的Term 已经更新，从而判断自己已经失去leader 身份。如果前任leader 处于另一个网络分区，它将无法感知到新的leader；但同分区的其他follower 也不会与新leader 进行通信，因此在这个案例中server 和client 无限等待直到分区问题恢复，也没啥问题（note：不会造成重复请求cmd 的提交）
- 提示2：你可能需要修改Clerk 来记录上次RPC 是哪个server 作为leader 进行响应的，并将下一个RPC 请求首先发送给那个server。此举可以避免在每次发送RPC 时寻找leader 的时间，从而可以助你足够快速的执行操作并通过测试案例。
- 提示3：你将需要对客户端操作进行唯一标示，以便可以确认kv 服务对相同的操作只执行一次。
- 提示4：检查重复执行的操作应该在较短时间内释放server 的内存，例如可以让每个RPC 请求代表此前的RPC 请求已经收到了响应。假设一个client 同时只调用一次Clerk 是合理的。

____

你的代码应该可以像下方所示一样通过所有Lab 3A 测试用例：
```shell
$ go test -run 3A
Test: one client (3A) ...
  ... Passed --  15.1  5 12882 2587
Test: many clients (3A) ...
  ... Passed --  15.3  5  9678 3666
Test: unreliable net, many clients (3A) ...
  ... Passed --  17.1  5  4306 1002
Test: concurrent append to same key, unreliable (3A) ...
  ... Passed --   0.8  3   128   52
Test: progress in majority (3A) ...
  ... Passed --   0.9  5    58    2
Test: no progress in minority (3A) ...
  ... Passed --   1.0  5    54    3
Test: completion after heal (3A) ...
  ... Passed --   1.0  5    59    3
Test: partitions, one client (3A) ...
  ... Passed --  22.6  5 10576 2548
Test: partitions, many clients (3A) ...
  ... Passed --  22.4  5  8404 3291
Test: restarts, one client (3A) ...
  ... Passed --  19.7  5 13978 2821
Test: restarts, many clients (3A) ...
  ... Passed --  19.2  5 10498 4027
Test: unreliable net, restarts, many clients (3A) ...
  ... Passed --  20.5  5  4618  997
Test: restarts, partitions, many clients (3A) ...
  ... Passed --  26.2  5  9816 3907
Test: unreliable net, restarts, partitions, many clients (3A) ...
  ... Passed --  29.0  5  3641  708
Test: unreliable net, restarts, partitions, many clients, linearizability checks (3A) ...
  ... Passed --  26.5  7 10199  997
PASS
ok      kvraft  237.352s
```
每个 Passed 后面的数字分别代表着 实际时间，peer的个数，RPC 发送的字节数（包括 client 发送的RPC），kv操作个数（Clerk 的Get/Append/Put 调用的kv 操作数）
