Raft Structure Advice                                                   

A Raft instance has to deal with the arrival of external events         
(Start() calls, AppendEntries and RequestVote RPCs, and RPC replies),   
and it has to execute periodic tasks (elections and heart-beats).       
There are many ways to structure your Raft code to manage these         
activities; this document outlines a few ideas.

Each Raft instance has a bunch of state (the log, the current index,    
&c) which must be updated in response to events arising in concurrent   
goroutines. The Go documentation points out that the goroutines can     
perform the updates directly using shared data structures and locks,
or by passing messages on channels. Experience suggests that for Raft   
it is most straightforward to use shared data and locks.

A Raft instance has two time-driven activities: the leader must send    
heart-beats, and others must start an election if too much time has     
passed since hearing from the leader. It's probably best to drive each  
of these activities with a dedicated long-running goroutine, rather     
than combining multiple activities into a single goroutine.             

The management of the election timeout is a common source of            
headaches. Perhaps the simplest plan is to maintain a variable in the   
Raft struct containing the last time at which the peer heard from the   
leader, and to have the election timeout goroutine periodically check   
to see whether the time since then is greater than the timeout period.  
It's easiest to use time.Sleep() with a small constant argument to      
drive the periodic checks. Don't use time.Ticker and time.Timer;        
they are tricky to use correctly.                                       

You'll want to have a separate long-running goroutine that sends        
committed log entries in order on the applyCh. It must be separate,     
since sending on the applyCh can block; and it must be a single         
goroutine, since otherwise it may be hard to ensure that you send log   
entries in log order. The code that advances commitIndex will need to   
kick the apply goroutine; it's probably easiest to use a condition      
variable (Go's sync.Cond) for this.

Each RPC should probably be sent (and its reply processed) in its own   
goroutine, for two reasons: so that unreachable peers don't delay the   
collection of a majority of replies, and so that the heartbeat and      
election timers can continue to tick at all times. It's easiest to do   
the RPC reply processing in the same goroutine, rather than sending     
reply information over a channel.

Keep in mind that the network can delay RPCs and RPC replies, and when  
you send concurrent RPCs, the network can re-order requests and         
replies. Figure 2 is pretty good about pointing out places where RPC    
handlers have to be careful about this (e.g. an RPC handler should      
ignore RPCs with old terms). Figure 2 is not always explicit about RPC  
reply processing. The leader has to be careful when processing          
replies; it must check that the term hasn't changed since sending the   
RPC, and must account for the possibility that replies from concurrent  
RPCs to the same follower have changed the leader's state (e.g.         
nextIndex).


Raft 结构结构建议

Raft 实例需要处理外部事件（Start() 调用，AppendEntries 和 RequestVote RPC

请求，以及RPC响应）

并且还需要执行定时任务（选举和心跳）。

有很多管理这些行为的代码结构组织方式；本文提供一些点子

每个Raft实例有很多状态（日志，当前index，&c）需要在当前goroutines 中响应事件时

进行状态更新。

Go 文档指出goroutines 可以通过使用共享数据结构和锁来直接优化更新，

或者通过向channel 里传递消息。经验告诉我们对于Raft 使用共享数据和锁是最简单的方式

一个Raft 实例有个事件驱动的动作：leader必须发送心跳，

其他follower必须在太久没收到leader 发出的心跳时发动一次选举。

可能驱动这些动作的最佳方式是：

为每个动作使用一个长期运行的goroutine

而不是将多个动作集中到一个goroutine 中

选举超时的管理是一个让人头疼的问题

也许最简单的方式是在Raft 的结构中维护一个变量

变量包含从leader 获取到心跳信息的最后时间

然后让选举超时 goroutine定期检查此变量

判断最后一次接收到心跳的时间到当前时间间隔是否超过设定的超时周期

使用带有常量参数的 time.Sleep() 来驱动定期检查是最简单的方式

不要使用 time.Ticker 和 time.Timer;

这俩方法很难正确使用

你可能会想使用一个单独长期运行的goroutine 来向applyCh 发送

已提交的日志记录。这个goroutine 必须单独使用

因为向applyCh 发送数据可能会被阻塞；另外还必须仅用一个goroutine

因为多个goroutine 会导致很难确定发送的日志记录的顺序。

更新commitIndex 的代码需要与apply goroutine 同时触发；

使用条件变量（Go 的 sync.Cond）可能是最简单的实现方式。

每个RPC 应该在它自己的goroutine 中被发送（以及处理响应）

这是出于两个原因：1. 不可达的节点不会影响对于大多数节点响应的收集

2. 保证心跳和选举计时器始终正常工作。

在同一个goroutine 中处理 RPC的响应是最简单的

相比通过channel 发送响应的信息而言

牢牢记住，网络可能会将RPC 请求和响应延迟，

当你发送当前的RPC 时，网络可能会对于请求和响应重新排序

图2 很好的指出了RPC handler 必须小心处理这种情况

（比如handler 应该忽略旧term 的RPC请求或响应。

图2 对于RPC 响应的处理不是很清楚

leader 在处理RPC 响应时必须小心；

leader 必须对发送RPC 后term是否发生了变化进行检查

并且必须考虑 向相同follower 发送的RPC

的响应已经改变了leader 的状态（比如 nextIndex）