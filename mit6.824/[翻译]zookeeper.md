# ZooKeeper: 针对大型网络系统的无等待共识算法

## 摘要
本文中，我们阐述了ZooKeeper，一个分布式应用的共识处理服务。作为要求苛刻的基础设置的一部分，ZooKeeper 旨在为客户端构建更复杂的共识提供简单和高性能的底层服务。包括了副本集中服务中的群发消息，共享注册器，分布式锁服务。ZK 暴露出来的接口有共享注册器所拥有的无等待（wait-free）优势，事件驱动机制类似分布式文件系统中的独立缓存一样提供了一个简单的，有效的共享服务。

ZK 接口实现了高性能服务。加上无等待的特性，ZooKeeper 为每个客户端提供了处理请求时FIFO 的保证，并保证了所有会改变ZK 状态的请求的线性化。这些设计使得 实现本地server 读取请求的高性能处理管道成为可能。我们展示了目标负载，读写绿从2:1 到100:1, ZK 每秒可以处理成百上千的交易。这样的性能变现让ZK 被广泛应用与客户端应用程序。

## 1 简介

大规模分布式应用需要不同的协调构成。#TODO

实现协调的一种方式是为每种不同的协调需求开发不同的协调服务。#TODO

当设计协调服务时，我们避免了在server 端实现特定的原型，而是选择了暴露一个API，使得应用开发者可以实现他们自己的原型。#TODO

当设计ZK 的API 时，我们移除了阻塞原型，比如锁。#TODO

尽管无等待的特性对于性能和容错很重要，但对于协调来说还不够。#TODO

ZK 服务由一组服务器构成，服务器使用了副本来达到高可用和高性能。#TODO

为了保证update 操作满足线性化，我们实现了一个基于leader 的原子官博机制，称为Zab[24]。#TODO

在客户端缓存数据是提高读取性能的重要手段。#TODO

本论文中我们讨论了ZooKeeper 的设计和实现思想。

总结一下，本论文我们做出的主要贡献是：

**协调内核(Coordination kernel)**：我们提出了一种无等待的协调服务，使用在分布式系统中是弱一致性保证的。特别的，我们阐述了*coordination kernel* 的设计和实现，并已经在很多底层应用中用来实现多样的协调技术。

**协调副本(Coordination recipes)**：我们展示了如何在构建更高级协调原型时使用ZooKeeper，即使是分布式系统中常见的阻塞和强一致性的协调原型。

**协调的经验(Experience with Coordination)**：我们分享了使用ZooKeeper 的方法，以及衡量性能的方法。

## 2 ZooKeeper 服务

客户端使用ZooKeeper 客户端的库，通过一个客户端API 向ZK 发送请求。

本章节中，我们首先提供了ZK 服务较高级别的视角。然后讨论了客户端与ZK 交互所使用的API。

**专业术语**. 本文中我们使用 client 来描述ZK 服务的一个用户，server 来描述提供ZK 服务的一个进程，用znode 来描述在ZK 数据中内存数据节点，ZK 数据用被称为 *data tree*的分级命名空间组织起来。我们还使用term 的更新和写入来表示任意修改了 data tree 状态的操作。当连接到ZK 时，client 建立一个 *session* ，并获得一个请求接收者的session handle。

### 2.1 服务概览

ZooKeeper 提供给它的客户端数据节点（znode）的集合概述，znode 根据分级命名空间组织起来。

client 可以创建两种类型的znode：

**定期（Regular）**: Client 通过明确的创建和删除来维护regular znodes；

**短期（Ephemeral）**: Client 创建 ephemeral znodes，且会明确的删除，或者当创建它们的session终止时（人为或由于异常），让系统自动删除这些znode。

另外，当创建一个新的znode时，client 可以设置一个 sequential 标志。

ZooKeeper 实现了watch，允许client 及时接收变化的通知，而不需要投票。

**数据模型(Data model)**. 
ZK 的数据模型本质是一个文件系统，带有简化的、只允许读写的API，或者说是一个有key分级的key/value 表。


与文件系统中的文件不同，znodes 不是被设计用来实现通用数据存储的。znodes 是client 应用的抽象映射，通常是为了达到协调目的的元数据。

尽管znodes 不是被用来存储通用数据的，ZK 还是允许client 存储一些分布式计算中可能被用作元数据或者配置信息的数据。

**Sessions**. 
一个client 连接到ZK 并开启一个session。Session 有一个被分配的超时时间。ZK 认为在session 超时时间内没有接收到任何信息的client 是发生了client 异常的。当client 明确关闭session handle 或者ZK 判断client 异常时，一个session 就结束了。通过session，client 获取到一个反映出client 操作执行后状态改变成功的信息。Session 允许client 透明的从ZK 集群的一个server 移到另一个server，由此持久化是跨ZK 节点的。


### 2.2 客户端API

下方展示了一些有关的ZK API，并讨论每个请求的场景。

**create(path, data, flags)**: 

delete(path, version):

exists(path, watch):

getData(path, watch):

setData(patch, data, version):

getChildren(path, watch):

sync(path):

所有方法都有同步和异步版本的API。当需要执行一个ZK 操作且没有其他并行任务时，程序会使用同步API，来调用ZK 并阻塞。然而异步API 允许程序并行执行多个ZK操作和其他任务。ZK client 保证了每个操作回调被有序的唤醒。

需要注意ZK 不使用handles 访问znodes。每个请求都不带有要操作的znode 的完整路径。这不仅简化了API（无需open() 或close() 方法），而且减少了server 需要维护的额外状态。

每种更新方法携带了一个版本号，用来实现条件更新。如果zone实际版本号与期望的版本号不相符，更新操作将由于 unexpected version 错误而失败。如果版本号是-1，表示不进行版本检查。

### 2.3 ZooKeeper 保证了啥

ZK 有两个基本的有序保证

**线性写入**：所有更新ZK 状态的请求都是可串行化；

**先进先出客户端顺序**：客户端发来的所有请求都是按照他们被客户端发出的顺序执行的

请注意我们定义的线性化与Herlihy[15] 的原始描述有区别，我们称为A-linearizability（异步线性化）。在Herlihy 原始定义中，客户端一次只能有一个请求（客户端是单线程的）。在我们的定义中，允许客户端并行发送多个操作，并且由此我们可以选择对于同一个客户端发出的突出的操作不指定特定顺序，或者保证FIFO 顺序。我们选择了后者作为ZK 的特性。注意到为线性化对象所持有的结果，同样被异步线性化对象所持有，因为满足异步线性化的系统同样满足线性化。由于只有更新请求是异步线性化的，ZK 进程从本地每个副本读取请求。这允许向系统中增加server 来达到服务线性扩展。

为了明白上述两个ZK 保证如何交互，考虑如下情景。由一堆进程构成的系统选举了一个leader 来向worker 进程发送命令。当新leader 开始掌管系统时，它必须改变大量配置参数，并在完成时通知其他进程。此时我们有两个重要需求：

- 在新leader 开始进行改变时，我们不希望其他进程开始使用新leader 正在改变的配置；
- 如果新leader 在完成配置更新之前挂了，我们不希望进程们使用只更新了一部分的配置。

注意到分布式锁，诸如Chubby 提出的锁，可以满足第一个诉求，但对于第二个是不充分的。使用ZK，新leader 可以指定ready znode 的一个路径，其他进程只有在这个路径下的znode 存在时才会使用其配置。新leader 通过删除 ready 来对配置进行更改，对多种配置的znodes 进行更新，随后创建 ready。所有这些变更可以是管道线性的，且可以异步执行已尽快更新配置的状态。尽管一个变更操作的延迟是2毫秒，如果逐个处理，一个新leader 将耗费10秒来更新5000个不同的znode；通过使用异步处理将花费少于1秒的时间。由于顺序的保证，如果一个进程发现了ready 的znode，它必须能够看到新leader 实施的所有配置变更。如果新leader 在ready znode 创建出来之前挂掉，其他进程可以判断配置没有被完成，就不会采用。

上述情景还存在一个问题：当一个进程在新leader 开始改变配置之前发现了ready 的存在并在配置正在改变时进行了读取。这个问题通过对通知的顺序保证来解决：如果一个client 正在监听变化，client 将会在改变发生后读取到新状态之前，接收到通知消息。因此，如果读取ready znode 的进程向znode 请求接收改变的通知，它将会在可以读取到任何新配置之前收到通知。

当client 拥有与ZK 通信的自有通道时会出现另外一个问题。例如，考虑两个client A 和B 共享一个ZK 配置的情况，且通过共享通道与ZK 通信。如果A 改变了ZK 中的共享配置文件，并通过共享通道通知B，B在重新读取配置时希望读到A 做的变更。如果B 的ZK 副本稍微落后于A ，它可能会看不到新的配置。使用上述保证（线性写、客户端FIFO），可以确定在B 重新读取前，处理写入的最新信息可以被B 获取到。为了更高效的处理此场景，ZK 提供了 sync 请求：当紧接着有读取操作时，（当前read request）形成一个 slow read。sync 导致server 在执行read 之前完成所有挂起的写入请求的应用。这个原型与ISIS 的flush 原型的思想很像。

ZK 还有如下两个存活和持久保证：如果ZK 的大多数节点存活，与ZK 服务的通信就是可用的；如果对于一个更新操作返回成功，则此操作可以承受在集群可恢复的限度内任意数量节点的崩溃，变更的数据能够保证持久化。

### 2.4 原型示例

本章将展示如何使用ZK API来实现更强大的原型。ZK 服务对更强大的原型知之甚少，因为更强大的原型完全使用ZK 的client API 在客户端实现。一些常见的原型，诸如组成员身份和配置管理同样也是wait-free 的。其他诸如集合点（rendezvous），client 需要等待一个事件。尽管ZK 是wait-free 的，我们可以使用ZK 实现有效的阻塞原型。ZK 的顺序保证可以构建有关系统状态的有效推理，watch 可以实现有效的等待。

**配置管理**

ZK 可以在分布式应用中用来实现动态配置。最简单的方式是将配置存储到znode 中，zc。进程使用zc 的全路径名启动。启动的进程通过读取zc 获得配置，读取时watch 标志设为true。如果在zc 中的配置被更新，进程会收到通知，然后读取新的配置，依然把watch 设为true。

注意到在此场景中，watch 如其他大多数场景中一样，被用来确定进程拥有最新的信息。例如，如果监听zc 的进程被通知zc 有更新，且在读取zc 之前又有另外三个对于zc 的更新，进程没有收到这三个的通知事件。这并不影响进程的行为，因为后来的三个事件发给进程的通知是进程已经知道的：所持有的有关zc 的信息已经过期了。

**集合点**

有时在分布式系统中，系统配置最终是啥样并不是清晰的先验知识。例如，client 可能想启动一个master 进程和多个worker 进程，但进程启动是由调度器完成的，因此client 无法事先了解诸如提供给worker 连接到master 的地址和端口等信息。我们通过ZK 使用一个集合点znode，zr来解决此场景的问题。zr 是由client 创建的一个node。client 将zr 的全路径名作为master 和worker 进程的启动参数传递。当master 启动时，master 将它所使用的地址和端口信息填入zr。当worker 启动时，他们读取zr 并将watch 设为true。如果zr 已经被填入信息，worker 会等待zr 被更新。如果zr 是一个短暂的 node，master 和worker 进程可以监听zr 被删除，当client 结束时完成自己的清理。

**集群管理**

我们利用临时节点的特性来实现集群管理。特别的，我们利用临时节点允许观察到创建了node 的session的状态的特性。通过指定一个znode，zg来代表集群。当集群的一个进程成员启动时，会在zg 下创建子znode。如果每个进程拥有一个唯一的名称或标识，则唯一名称或标识可以用来作为子znode 的名称；另一方面，进程创建带有SEQUENTIAL 标志的znode 来获得一个唯一名称。进程将进程信息放入子znode 的数据中，比如有关进程的地址和端口。

在进程正常启动，并在zg 下创建了子znode 之后，完全不需要做其他操作。如果进程失败或结束，zg 下代表进程的子znode 会自动被移除。

进程们可以通过简单的列出zg 的孩子来获得集群的信息。如果一个进程想要监控集群的变化，可以设置watch 标志为true，当收到变更通知时刷新集群信息（始终将watch 标志设为true）。

**简单锁**

尽管ZK 不是一个锁服务，但可以用它来实现锁。使用ZK 的应用通常使用按照需求定制的同步原型，如上面所展示的那样。接下来我们将展示如果使用ZK 实现锁，从而展示ZK 可以实现多种通用的同步原型。

最简单的锁实现是使用“lock files”。这个锁由一个znode 代表。为了获取锁，一个client 尝试创建指定的带有EPHEMERAL 标志的znode。如果创建成功，client 持有锁。否则，client 可以读取到这个znode 并设置watch 标志为true 以接收当前leader 挂掉的通知。client 在挂掉或删除znode 时释放锁。一旦观察到znode 被删除，其他等待这个锁的client 将重新尝试获取锁。

尽管简单锁机制是有效的，但它存在一些问题。首先，会受到羊群效应的影响。如果有很多client 等待获取锁，在锁被释放时会被所有等待的client 争抢，尽管只会有一个client 可以获取到。其次，简单锁只实现了排他锁。接下来的两个原型将展示如何解决这两个问题。

**没有羊群效应的简单锁**

我们定义一个锁znode l来实现没有羊群效应的简单锁。我们将请求锁的client 排成一列，每个client 按照请求到达的顺序获得锁。期望获得锁的client 将按照如下操作：

```
Lock
1 n = create(l + “/lock-”, EPHEMERAL|SEQUENTIAL) 
2 C = getChildren(l, false)
3 if n is lowest znode in C, exit
4 p = znode in C ordered just before n
5 if exists(p, true) wait for watch event 
6 goto 2

Unlock
1 delete(n)
```

第1行的SEQUENTIAL 标志确定了所有client 尝试获取锁的顺序。如果在第3行，client 的znode 有最低的sequence号，client 将持有锁。否则，client 将等待一些znode 的删除，那些znode 是持有锁或者将要在本client 的znode 之前获得锁的。通过只监听本client 的znode 之前的znode，当锁被释放或者锁请求被抛弃时只唤醒一个进程，从而避免了羊群效应。一旦被client 监听的znode 离开，client 必须检查是否持有锁。（之前的锁请求可能已经被抛弃，存在有更低序列号的znode 仍在等待或者正在持有锁。）

释放锁与删除代表锁请求的znode n 一样简单。通过创建时使用的EPHEMERAL 标志，崩溃的进程会自动清理任意锁请求，或者释放他们持有的任意锁。

综上，锁模式有如下优势：
    1. znode 的删除只会引发一个client 被唤醒，因为每个znode 被确切的另一个client 监听，因此就不存在羊群效应；
    2. 没有轮询和超时；
    3. 因为我们实现锁的方式，我们可以通过浏览ZK 数据查看到锁争抢的数量，打断锁，以及调试锁的问题。

**读写锁**

为了实现读写锁，我们对上述锁过程稍作改动，将读锁和写锁区分开。释放锁的过程与全局锁的案例中一样。

```
Write Lock
1 n = create(l + “/write-”, EPHEMERAL|SEQUENTIAL) 
2 C = getChildren(l, false)
3 if n is lowest znode in C, exit
4 p = znode in C ordered just before n
5 if exists(p, true) wait for event 
6 goto 2

Read Lock
1 n = create(l + “/read-”, EPHEMERAL|SEQUENTIAL)
2 C = getChildren(l, false)
3 if no write znodes lower than n in C, exit
4 p = write znode in C ordered just before n
5 if exists(p, true) wait for event
6 goto 3
```

这个锁程序与之前的锁稍有不同。写锁与没有羊群效应的简单锁只是命名上不同。然而读锁可能被共享，第3行和第4行与之前的逻辑有区别，因为只有更早的写锁znode 会阻止client 获取一个读锁。当有多个client 等到读锁时，随着序列号更低的"write-" znode被删除，多个client接到通知可能出现羊群效应；实际上这是一个预期中的行为，在可以获得锁时，所有试图读的client 都应该被唤醒。

**双屏障（double barrier）**

双屏障可以让client 同步计算的开始和结束。当加入barrier 的进程数量达到barrier 阈值时，进程开始他们的计算，并在完成计算后离开barrier。在ZK 中，使用一个znode 代表一个barrier，计作b。每个进程p 使用b 注册到实体上 - 注册通过创建一个b 的child znode 来实现，当准备好离开时取消注册 - 将b 下的child znode 删除。当b 的child znode 数量超过barrier 的阈值时，进程可以进入barrier。当所有进程将他们的children 删除时，进程可以离开barrier。我们使用watch 高效地等待进入和推出条件被满足。为了进入，进程对属于b 的ready 子znode 是否存在进行监听，ready 子znode 由进程创建，此进程使得children数量达到barrier 阈值。为了离开，进程监听特定child 的消失，只需要在这个特定znode 被删除时检查一次退出条件是否满足即可。

