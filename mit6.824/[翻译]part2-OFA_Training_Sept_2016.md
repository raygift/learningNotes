# Writing Application Programs for RDMA using OFA Software Part 2

## RDMA 的目标（p3）

- 高带宽使用率
- 低延迟
- 低CPU利用率

## 当前实现RDMA 的技术（p3）

- InfiniBand
- iWART
- RoCE

## 通道适配器 Channel Adapters 有关术语（p4）

- Infiniband
  - HCA：Host Channel Adapter
- iWARP
  - RNIC：RDMA Network Interface Card
- RoCE
  - NIC？？
- 通用
  - CA

## RMDA 主要特性（p5）

- 数据零copy传输
  - 数据从一个节点的用户内存直接移动到另一节点的用户内存
  - 没有CPU 介入
  - 不需要临时buffer
- 内核 by-pass
  - 用户可以直接访问CA
  - 没有kernel 级别的协议处理
  - 所有协议处理都在CA 完成

## 对比 AF_INET 与 RDMA （p6）

（AF_INET：IPv4 网络协议的套接字类型）

- 都使用了 client-server 模式
- 对于可靠传输都需要建立
- 都提供了可靠传输模式
  - AF_INET 中的TCP 提供了一种可靠的，有序的**字节流**（个人理解，流的前提是有发送和接收两方有连接建立，数据的发送可以分成多批，接收方可以对收到的多批数据流按顺序组合起来）
  - RDMA_PS_TCP 提供了一种可靠的，有序的消息序列
  - TCP 使用buffer 复制，RDMA 无需buffer

## 有buffer copy的TCP 传输示意图（p7）

![image-20210605152050904](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605152050904.png)

## 没有buffer copy的RDMA 传输示意图（p8）

![image-20210605152112324](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605152112324.png)

## 重要知识点（p9）

- 所有RDMA 传输都是用分离消息
  - 不是 TCP 流
- 我们主要讨论可靠传输
  - iWARP没有实现不可靠传输
  - 不可靠传输主要用在IB 广播中

## 客户端-服务端（C/S）模式（p10）

服务端

- 监听者
  - 等待来自客户端的连接

- 代理
  - 向一个客户端传输数据

客户端

- 连接到服务端的一个监听者
- 向服务端的代理传输数据

## CS 类比（p11）

- server（商业公司）
  - listener（呼叫中心）
    - 创建到client 的监听
    - 确定DNS 域名，端口（1-800）
    - 永远循环
      - 监听来自client（消费者） 的新的连接（电话）
      - 将新的连接分发给agent（消费者代表）
  - agent（消费者代表）
    - 接收来自listener 的连接
    - 向client 传输消息
    - 关闭连接
- client（消费者）
  - 解析DNS 域名，端口
  - 创建连接
  - 与agent 传输消息
  - 关闭连接



## OFA API（p13）

OFA API 与 socket 有一些类似

对于socket 程序员来说有许多新概念

- Verbs（在OFA API 中只是函数的另一个叫法）
- 保护域 Protection Domains
- 内存注册 Memory Registration
- 连接管理
- 显式的队列操作
- 显式的事件处理
- 显式的异步操作
- 显式的系统数据结构操作

## OFA API 编程风格（p14）

- （几乎所有）的CA 操作都是异步的
  - 程序调用一个函数来开启操作，函数向CA 传达请求并立刻返回
  - CA 操作并行进行
  - CA 在操作完成时“通知”程序；CA向程序传达操作的状态
- 很多程序是用线程来处理并行
- 很多系统级结构体是可见的
- 很多新术语，首字母以及缩写

## 异步CA 操作示意图（p15）

![image-20210605153248094](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605153248094.png)

## 异步操作数据传输（p16）

- Posting
  - 启动数据传输操作开始的术语
- Completion
  - 确定数据传输操作结束的术语
- 在Posting 和 Completion 之间，存有消息数据的用户内存区域是未被定义的，不应该被用户程序所修改

## Posting - Completion 示意图（p17）

![image-20210605153306671](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605153306671.png)

## OFA 数据传输类型（p18）

- send/receive - 类似传统的socket
- RDMA_WRITE - OFA 所特有
- RDMA_READ -  OFA 所特有
- Atomics - 只存在与IB，作为一个可选的实现

首先讨论Send/Receive，同时介绍基本概念

- 所有传输类型使用相同的verbs（函数）和数据结构（也就是send/receive 和write/read 使用的是一套函数和数据结构）

## Send/Recv 数据传输（p19）

![image-20210602182658618](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210602182658618.png)

## 程序主要构成（p20）

1. 传输Posting
2. 传输Completion
3. 内存注册
4. 连接管理
5. 杂项

- 接下来将从上到下进行讨论

## 每个构成都需要考虑（p21）

- 目的
- 数据结构
- 准备数据结构的verbs 函数
- 使用数据结构的verbs 函数
- 销毁数据结构的verbs 函数
- 与其他数据结构的关系

## verbs 的构成金字塔（p22）

下图展示了verbs金字塔，每个构成部分的所有verbs处于同一行，分为三列

- setup
- use
- break-down

每个部分包含了有关的verbs

作为一个路径图，用来表示从哪里执行，接下来要执行什么

新讲到的verbs 会在图中被高亮显示

![image-20210602183614024](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210602183614024.png)

## 数据结构的金字塔（p24）

下图展示了对数据结构分层的金字塔

每层包含了在程序不同阶段被verbs 启动和销毁的相关数据结构

![image-20210602183913338](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210602183913338.png)

## 数据结构关系（p26）

下图展示了OFA 主要数据结构及相互的关系

![image-20210602184757764](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210602184757764.png)

## 本ppt的讲解方式（p28）

- 从简单的构成和例子开始，然后深入
- 使用金字塔自上而下进行介绍
  - 主要构成
  - 程序员如何阅读最终代码
  - 数据传输是最重要，最令人感兴趣的
- 讨论程序构成
- 使用金字塔自底向上
  - 程序员如何使用API 构建程序
- 有关RDMA的所有技术，处理API 以便所有程序都可以运行在API 之上

## 工作代码（p29）

## Send/Recv 数据传输示意图（p30）

![image-20210602190235495](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210602190235495.png)

## 使用Send/Recv 的 Ping-Pong 示意图（p31）

## 使用Send/Recv 的 Blast 示意图（p32）



# 主要构成（p33）

# 主要构成1：Transfer Posting（p34）

通过这个机制给了CA 所有启动RDMA 传输数据的所需的信息

## Send/Recv 与socket 相似之处（p35）

- 在传输数据之前，client 必须与server 建立一个连接
- 发送方不知道远程接收方的虚拟内存地址
- 接收方不知道远程发送方的虚拟内存地址
- 每次消息的发送必须与消息的接收匹配（发送和接收方都是传输的一部分）

## Send/Recv 与socket 的不同之处（p36）

- 传统socket 传输是有buffer 的（send 相对 recv 的顺序是无关紧要的）
- OFA 传输是没有buffer 的（recv 必须在send 之前被提交）
- 当使用OFA 时，两端的虚拟内存都必须注册在本端（传统socket 没有注册内存的概念）
- OFA 异步传输操作（传统socket同步传输操作）



## Posting 接收数据（p38）

- 函数：ibv_post_recv()
- 参数：
  - QP
  - receive work request 列表的指针 - RWR
  - bad RWR 的指针
- 返回值：
  - ==0:所有RWR 被成功加入到recv queue 中
  - ! =0:错误码

## Posting 发送数据（p39）

- 函数：ibv_post_send()
- 参数
  - QP
  - send work request 列表的指针
  - bad SWR 的指针
- 返回值：
  - ==0 表示所有SWR 被成功加入 Send Queue
  - !=0 错误码



## posting 相关函数

## ibv_post_recv() 代码片段（p40）

```c
int our_post_recv(struct our_control *conn, struct ibv_recv_wr *recv_work_request,
                  struct our_options *options)
{
    struct ibv_recv_wr *bad_wr;
    int ret;
    errno = 0;
    ret = ibv_post_recv(conn->queue_pair, recv_work_request, &bad_wr);
    if (ret != 0)
    {
        if (our_report_wc_status(ret, "ibv_post_recv", options) != 0)
        {
            our_report_error(ret, "ibv_post_recv", options);
        }
    }
    return ret;
} /* our_post_recv */
```



## ibv_post_send() 代码片段（p41）

```c
int our_post_send(struct our_control *conn, struct ibv_send_wr *send_work_request,
                  struct our_options *options)
{
    struct ibv_send_wr *bad_wr;
    int ret;
    errno = 0;
    ret = ibv_post_send(conn->queue_pair, send_work_request, &bad_wr);
    If(ret != 0)
    {
        if (our_report_wc_status(ret, "ibv_post_send", options) != 0)
        {
            our_report_error(ret, "ibv_post_send", options);
        }
    }
    return ret;
} /* our_post_send */
```



## work request 数据结构金字塔(p42)

在Transfer Posting 层面有 ibv_recv_wr、ibv_send_wr、ibv_sge、ibv_qp 四个函数

## 最简单的work request 构成（p43）

一个消息的work request，作为一个 scatter-gather 元素；scatter-gather 作为整个消息的用户数据

![image-20210605222410887](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605222410887.png)

## Receive Work Request （RWR）（p44）

- 目的：告诉CA 把它收到的数据放到虚拟内存的什么位置
- 数据结构：struct ibv_recv_wr
- 程序员可见字段：
  - next
  - wr_id
  - sg_list
  - num_sge
- 必须在调用 ibv_post_recv() 之前，填充上述字段

## Send Work Request（SWR）（p45）

- 目的：告诉CA 将数据发送到哪里
- 数据结构：struct ibv_send_wr
- 程序员可见字段：
  - next
  - wr_id
  - sg_list
  - opcode ： IBV_WR_SEND
  - num_sge ：sg_list 数组的元素个数
  - send_flags：IBV_SEND_SIGNALED
- 必须在调用ibv_post_send() 之前填充上述字段



## Work Request List（p46）

- 允许一个ibv_post_recv() 或ibv_post_send() 向CA 发送多个传输操作
- 列表中每个WR 是独立的
- 列表中每个WR 生成它自己的completion
- WR list 中的 WR 类型都必须一样，要不全是Recv ，要不全是Send
- 实践中，大多数WR list 只包含 1 个WR



## send work request list 示意图（p48）

![image-20210605222042117](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605222042117.png)

## our_setup_recv_wr() 代码片段（p49）

```c
static void
our_setup_recv_wr(struct our_control *conn, struct ibv_sge *sg_list,
                  int n_sges, struct ibv_recv_wr *recv_work_request)
{
    /* set the user's identification to be pointer to itself */ r
        ecv_work_request->wr_id = (uint64_t)recv_work_request;
    /* not chaining this work request to other work requests */
    recv_work_request->next = NULL;
    /* point at array of scatter-gather elements for this recv */
    recv_work_request->sg_list = sg_list;
    /* number of scatter-gather elements in array actually being used */
    recv_work_request->num_sge = n_sges;
} /* our_setup_recv_wr */
```



## our_setup_send_wr() 代码片段（p50）

```c
static void
our_setup_send_wr(struct our_control *conn, struct ibv_sge *sg_list,
                  enum ibv_wr_opcode opcode, int n_sges, struct ibv_send_wr *send_work_request)
{
    /* set the user's identification to be pointer to itself */
    send_work_request->wr_id = (uint64_t)send_work_request;
    /* not chaining this work request to other work requests */
    send_work_request->next = NULL;
    /* point at array of scatter-gather elements for this send */
    send_work_request->sg_list = sg_list;
    /* number of scatter-gather elements in array actually being used */
    send_work_request->num_sge = n_sges;
    /* the type of send */
    send_work_request->opcode = opcode;
    /* set SIGNALED flag so every send generates a completion */
    send_work_request->send_flags = IBV_SEND_SIGNALED;
    /* not sending any immediate data */
    send_work_request->imm_data = 0;
} /* our_setup_send_wr */
```



## Scatter-gather Lists（p51）

- 每个work request 指向一个scatter-gather list ，即work request 中的 sg_list 字段
- sg list 是 scatter-gather 元素（SGE）组成的数组
- 数组中的每个元素的类型是 struce ibv_sge
- work request 给出了数组的长度 - num_sge 字段
- 实践中，大多数 scatter-gatter list 只包含一个SGE

## 有两个SGE 元素的WR 示意图（p53）

![image-20210605222308898](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605222308898.png)

## WR List 中的 sg_list 示意图（p54）

![image-20210605222729226](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605222729226.png)

## Recv scatter-gather 列表（数组）（p55）

- 目标：允许一个recv 操作将“线路里”单个消息切分成不同的虚拟内存块
- 如果 1 条消息在最大数据长度之后包含固定长度的header，sg 会有用
  - header 会被保存在内存的一个区域
  - 数据被保存到另一个区域
- sg_list 数组中每个元素描述了虚拟内存中的一个块（chunk）
- 实践中，大多数 sg_lists 只包含1个SGE

## ibv_post_recv() 函数执行期间的Scatter 示意图（p56）

![image-20210605222908176](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605222908176.png)



## Send scatter-gather 列表（数组）（p57）

- 目标：允许一个 send work request 从本地虚拟内存不同块将数据拉取到一起，合并成“线路上”的单个message（也即 gather）
- sg_list 每个元素描述了描述了本地虚拟内存中的一个块
- 对于在长度可变的数据之后有固定header长度的消息，sg会有用
  - 固定长度的header 存储在内存1个区域
  - 可变长度的data 存储在内存另1个区域
- 实践中，大部分sg_list 只有1个元素

## ibv_post_send() 期间的Gather 示意图（p58）

![image-20210605222957002](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605222957002.png)

## Scatter-Gather 元素（SGE）（p59）

- 目标：描述需要传输的虚拟内存块
- 数据结构：struce ibv_sge
- 程序员可见字段：
  - lkey：local key，本地内存注册的key
    - 必须覆盖内存块中所有字节
    - 必须对应QP 的 protection domain
  - addr：虚拟内存块的基地址
  - length：虚拟内存块的字节数
- 在调用ibv_post_recv() 或 ibv_post_sendI() 之前必须填充上述字段

## our_setup_sge() 代码片段（p60）

```c
/* fill in scatter-gather element for length bytes at registered addr */ static void
our_setup_sge(void *addr, unsigned int length, unsigned int lkey,
              struct ibv_sge *sge)
{
    /* point at the memory area */
    sge->addr = (uint64_t)(unsigned long)addr;
    /* set the number of bytes in that memory area */
    sge->length = length;
    /* set the registration key for that memory area */
    sge->lkey = lkey;
} /* our_setup_sge */
```



## ibv_send_wr 中的ibv_sge 示意图（p61）

![image-20210605223140501](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210605223140501.png)



## Queue Pair（QP）（p62）

- 所有RDMA 传输的一个统一数据结构
- 在数据传输操作中扮演着socket“fd” 的角色
- 重要的组成部分：
  - Protection domain：注册此QP 在传输中可以用到的内存区域
  - Send Completion Queue：完成用户尚未“捡起”的send 操作
  - Receive Completion Queue：完成用户尚未“捡起”的receive 操作

## QP 数据结构（p63）

- 目标：访问其他远程内存的主要结构
- 数据结构：struct ibv_qp
- 程序员可见的字段
  - pd
  - recv_cq：接收完成队列
  - send_cq：发送完成队列
  - qp_context：用户定义的此QP 的id
- 在QP 创建时，上述字段需要被初始化

## 数据结构金字塔中的QP（p64）

位于Transfer Posting 的ibv_qp

# 主要构成2：传输完成Transfer Completion（p65）



## 传输和完成队列示意图（p66）

![image-20210603121145439](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210603121145439.png)



## 函数金字塔中的Transfer Completion（p67）

- setup 阶段
  - ibv_create_cq
- use 阶段
  - ibv_poll_cq
  - ibv_wc_status_str
- break-down 阶段
  - ibv_destroy_cp

## 数据结构中的CQ 和WC （p68）



## 传输完成有关函数（p69）

- 目的：发现一个传输的完成
- 函数：ibv_poll_cq()
- 参数：
  - 完成队列 completion queue
  - work completion 分片数组 - wc
  - work completion 数组分片的数量
- 返回值：
  - 大于0 表示被填充的wc slots 的数量
  - <0 表示错误码

## CQ 数据结构（p70）

- 目的：保存CA 中有关已完成任务请求的信息，直到程序“将他们捡起”
- 数据结构：struct ibv_cq
- 程序员可见的字段：
  - cqe：此CQ 中最大的slots 数量
  - context：分配的cm_id 对应的verbs field
  - channel：提供给CA 返回完成事件（可空）
  - cq_context：用户为此CQ 定义的id
- 当创建CQ 时需要将上述字段初始化
- 系统返回的cqe 值可能比程序员提供的更大

## CQ 配置（p71）

- 函数：ibv_create_cq()
- 参数：
  - 与cm_id 有联系的verbs field
  - 新queue 中slots 的总数目
  - 用户定义的此queue 的id
  - completion channel
  - comp_vector
- 返回值
  - struct ibv_cq 新实例的指针

## CQ 销毁（p72）

- 函数：ibv_destory_cq()
- 参数：
  - ibv_create_cq() 返回的struct ibv_cq 实例的指针

## create_cq 代码片段（p73）

```c
static int
our_create_cq(struct our_control *conn, struct rdma_cm_id *cm_id,
              struct our_options *options)
{
    int ret;
    errno = 0;
    conn->completion_queue = ibv_create_cq(cm_id->verbs, options->send_queue_depth * 2, conn, NULL, 0);
    if (conn->completion_queue == NULL)
    {
        ret = ENOMEM;
        our_report_error(ret, "ibv_create_cq", options);
        return ret;
    }
    our_trace_ptr("ibv_create_cq", "created completion queue", conn->completion_queue, options);
    our_trace_ulong("ibv_create_cq", "returned cqe", conn->completion_queue->cqe, options);
  our_trace_ptr("ibv_create_cq", "returned completion queue channel",
	return 0;
} /* our_create_cq */
```

## Work Completion (WC)作业完成（p74）

- 保存当前已经完成的操作的数据结构
- 用户提供的作为ibv_poll_cq() 参数的空间
- CA 从cq 第一个元素拿到信息，填充到 work completion 字段
  - 生成此wc 的work request 的id
  - 有关此work 为何执行完成的信息
    - ==0 表示work 成功（IBV_WC_SUCCESS）
    - !=0 表示错误码

## WC 数据结构（p75）

- 目的：意味着网络适配器返回了work request 的有关状态信息
- 数据结构：struct ibv_wc
- 程序员可见字段：
  - wr_id：id 副本，id 是来自WR 的用户定义的wr_id
  - status：wr 成功或失败的标志
  - opcode：已完成的wr 的类型，分为IBV_WC_SEND 和IBV_WC_RECV 两类
  - byte_len：wr 传输的字节数
- 程序员需要在ibv_poll_cq() 返回之后检查上述字段

## ibv_poll_cq() 代码片段（p76）

```c
int our_await_completion(struct our_control *conn, struct ibv_wc *work_completion, struct our_options *options)
{
    int ret;
    /* busy wait for next work completion to appear in completion queue */
    do
    {
        errno = 0;
        ret = ibv_poll_cq(conn->completion_queue, 1, work_completion);
    } while (ret == 0);
    if (ret != 1)
    {
        /* ret cannot be 0, and should never be > 1, so must be < 0 */
        our_report_error(ret, "ibv_poll_cq", options);
    }
    else
    {
        ret = our_check_completion_status(conn, work_completion, options);
    }
    return ret;
} /* our_await_completion */
```

## 检查work 完成状态代码片段（p77）

```c
static int
our_check_completion_status(struct our_control *conn,
                            struct ibv_wc *work_completion, struct our_options *options)
{
    int ret;
    ret = work_completion->status;
    if (ret != 0)
    {
        if (ret == IBV_WC_WR_FLUSH_ERR)
        {
            our_report_string("ibv_poll_cq", "completion status", "flushed", options);
        }
        else if (our_report_wc_status(ret, "ibv_poll_cq", options))
        {
            our_report_ulong("ibv_poll_cq", "completion status", ret, options);
        }
    }
    return ret;
} /* our_check_completion_status */
```

## 报告work 完成状态的代码片段（p78）

```c
/* on entry, ret is known to be != 0 *
* Returns == 0 if ibv_wc_status message was printed (ret was valid status code) * != 0 otherwise
*/
int our_report_wc_status(int ret, const char *verb_name, struct our_options *options)
{
    /* ensure that ret is an enum ibv_wc_status value */
    if (ret < IBV_WC_SUCCESS || ret > IBV_WC_GENERAL_ERR)
        return ret;
    /* print the status error message */
    fprintf(stderr, "%s: %s returned status %d %s\n",
            options->message, verb_name, ret, ibv_wc_status_str(ret));
} /* our_report_wc_status */
```



## ibv_wc_status 枚举 - 位于verbs.h 文件中（p79）

## 数据传输过程涉及到的函数与数据结构（p80）

- 传输初始化
  - ibv_post_recv()
  - ibv_post_send()
  - Work Request - 分为 RWR，SWR
  - Scatter-Gather Element – SGE
  - Queue Pair - QP
- 传输完成
  - ibv_poll_cq()
  - Completion Queue - CQ
  - Work Completion - WC

## 使用Send/Recv 完成Ping-pong 示意图（p82）

![image-20210606122106392](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210606122106392.png)

## ping-pong 循环的客户端逻辑（p83）

```c
client_conn->wc_recv = client_conn->wc_send = 0;
while (client_conn->wc_recv < client_conn->limit)
{
    call our_post_recv() for client_conn->user_data_recv_work_request[0];
    call our_post_send() for client_conn->user_data_send_work_request[0];
    call our_await_completion() to get IBV_WC_SEND work_completion;
    client_conn->wc_send++;
    call our_await_completion() to get IBV_WC_RECV work_completion;
    if (work_completion.byte_len != client_conn->data_size)
        break;
    optionally call our_verify_data() to check recv data against send data
        client_conn->wc_recv++;
} /* while */
```

## ping-pong 循环的sever-agent 端逻辑（p84）

```c
call our_post_recv() for agent_conn->user_data_recv_work_request[0];
agent_conn->wc_recv = 0;
agent_conn->wc_send = 0;
while (agent_conn->wc_send < agent_conn->limit)
{
    call our_await_completion() to get IBV_WC_RECV work_completion;
    if (work_completion.byte_len != agent_conn->data_size)
        break;
    agent_conn->wc_recv++;
    call our_post_recv() for agent_conn->user_data_recv_work_request[0];
    call our_post_send() for agent_conn->user_data_send_work_request[0];
    call our_await_completion() to get IBV_WC_SEND work_completion;
    agent_conn->wc_send++;
} /*while */
```

# 主要构成3: 内存注册（p85）

## 数据结构金字塔上的PR 和MR（p86）

在memory registration 阶段的ibv_pd 和ibv_mr 

## Memory Registration 目标（p88）

- 目标：使得 CA 可以直接在host 内存之间传输数据，无需CPU 介入
- PD：用户定义的QP 和MR ，用于决定操作是否有权操作QP 和MR 
- MR：用户定义的内存区域，带有用户在PD 中定义的访问权限

## PD 和MR 的结构示意图（p89）

![image-20210606122603265](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210606122603265.png)

## Protection Domain - PD （p90）

- 意在控制CA访问主机系统内存
- 每个MR 都是一个PD 的成员
  - 多个MR 可以都是同一个PD 的成员
- 每个QP 都是一个PD 的成员
  - 多个QP 可以都是同一个PD 的成员
- 在QP 上传输的数据只能使用QP 对应的PD 下的MR

## PD 数据结构（p91）

- 目的：允许QP 能够使用MR 来进行传输
- 数据结构：struct ibv_pd
- 可见字段：
  - context ：cm_id 对应的verbs 字段
- PD 创建时需要初始化上述字段
- cm_id 在连接管理阶段被创建

## PD 配置/销毁（p92）

- PD 配置
  - 函数：ibv_alloc_pd()
  - 参数
    - context
  - 返回值：新创建的pd 的指针
- PD 的销毁
  - 函数：ibv_dealloc_pd()
  - 参数：要销毁的pd 的指针

## ibv_alloc_pd 代码片段（p93）

```c
static int
our_alloc_pd(struct our_control *conn, struct rdma_cm_id *cm_id,
             struct our_options *options)
{
    int ret = 0;
    errno = 0;
    conn->protection_domain = ibv_alloc_pd(cm_id->verbs);
    if (conn->protection_domain == NULL)
    {
        ret = ENOMEM;
        our_report_error(ret, "ibv_alloc_pd", options);
    }
    else
    {
        our_trace_ptr("ibv_alloc_pd", "allocated protection domain", conn->protection_domain, options);
    }
    return ret;
} /* our_alloc_pd */
```

## MR的作用（p94）

- MR 提供了允许CA 直接从主机内存读取数据，以及直接向主机内存写入数据的机制
- MR 是用户指定的一段连续的虚拟内存
- MR 的物理页被pin 到内存，从而不会在数据传输过程中被交换出内存
- MR 的物理内存与虚拟内存的映射是持久的（直到被取消注册）
- MR 的物理内存与虚拟内存的映射在注册过程中被写到CA 里
- 用户指定的PD
- 用户为本地和远程的CA指定访问权限，如下所示
- 系统为MR 生成本地key 和 远程key
  - lkey：被本地CA 访问MR 所使用
  - rkey：被远程CA 访问MR 所使用

## MR 数据结构（p96）

- 目的：定义RDMA 传输所用的MR
- 数据结构：struct ibv_mr
- 可见字段：
  - pd
  - lkey
  - rkey
  - addr：虚拟内存起始地址
  - length
- 程序员需要指定 pd，addr，和长度
- 系统返回lkey 和 rkey

## MR 配置（p97）

- 函数：ibv_reg_mr()
- 参数：
  - pd
  - region 起始地址
  - region 字节长度
  - region 访问权限
- 返回值：ibv_mr 新实例的指针

## ibv_reg_mr() 代码片段（p98）

```c
static struct ibv_mr *
our_setup_mr(struct our_control *conn, void *addr, unsigned int length,
             int access, struct our_options *options) struct ibv_mr *mr;
errno = 0;
mr = ibv_reg_mr(conn->protection_domain, addr, length, access);
if (mr == NULL)
{
    our_report_error(ENOMEM, "ibv_reg_mr", options);
}
return mr;
} /* our_setup_mr */
```

## MR 的访问权限（p99）

访问权限包括了每个CA 对本地内存拥有哪些访问权限

- IBV_ACCESS_LOCAL_WRITE：Allows local CA to write to local registered memory
- IBV_ACCESS_REMOTE_WRITE：Allows remote CA to write to local registered memory
- IBV_ACCESS_REMOTE_READ：Allows remote CA to read from local registered memory

默认情况，本地CA 始终被允许从本地注册的内存读取数据（访问权限代码为0）

注意： iWARP中如果使用了IBV_ACCESS_REMOTE_WRITE，同时需要IBV_ACCESS_LOCAL_WRITE

## Send/Recv 所需访问权限（p100）

- ibv_post_send() 带有 WR 的opcode 为 IBV_WR_SEND
  - 需要MR 只有默认local 读取权限
  - local CA从本地内存读取，向传输线路发送，"out onto the wire"
- ibv_post_recv()
  - 需要MR 至少有 IBV_ACCESS_LOCAL_WRITE
  - local CA 从传输线路读取，向本地内存写入， “in off the wire”

## Send/Recv 注意项！（p101）

- send/recv 在struct ibv_mr 中永远不会使用rkey 字段
- **send/recv 永远不会使用如下访问权限**
  - IBV_ACCESS_REMOTE_READ
  - IBV_ACCESS_REMOTE_WRITE

## Send/Recv 数据流示意图（p102）

![image-20210603185621290](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210603185621290.png)



## MR 销毁（p103）

- 函数： ibv_dereg_mr()
- 参数：ibv_reg_mr() 返回的 ibv_mr 实例的指针

## 配置client buffer（p104）

- ping-pong 的client 需要2个buffer（能够验证返回的“pong” 数据）
  - 一个包含原始的“ping”数据（需要默认本地读权限，用来向远程agent 发送数据）
  - 另一个获取返回的“pong”数据（需要本地写权限IBV_ACCESS_LOCAL_WRITE，用来接收远程agent 响应的数据）
- client 需要两个 work request
  - 一个向agent 发送 ping 数据
  - 一个接收从agent 返回的数据pong

## 使用 sent/recv 的 ping-pong 示意图（p105）

见p84

## 配置client buffer 的代码片段（p106）

```c
int our_setup_client_buffers(struct our_control *conn, struct our_options *options)
{
    int ret;
    int access[2] = {0, IBV_ACCESS_LOCAL_WRITE};
    /* client needs 2 buffers, first to send(), second to recv() */
    if ((ret = our_setup_user_data(conn, 2, access, options)) != 0)
        goto out0;
    /* client needs 2 work requests, first to send(), second to recv() */
    our_setup_send_wr(conn, &conn->user_data_sge[0], IBV_WR_SEND, 1, &conn->user_data_send_work_request[0]);
    our_setup_recv_wr(conn, &conn->user_data_sge[1], 1, &conn->user_data_recv_work_request[0]);
} /* our_setup_client_buffers */
```

## 配置用户数据代码片段（p107）

```c
static int
our_setup_user_data(struct our_control *conn, int n_user_bufs, int access[],
                    struct our_options *options)
{
    int ret, i;
    conn->n_user_data_bufs = 0;
    for (i = 0; i < n_user_bufs; i++)
    {
        /* allocate space to hold user data, plus 1 for '\0' */
        conn->user_data[i] = our_calloc(options->data_size + 1, options->message);
        if (conn->user_data[i] == NULL)
        {
            ret = ENOMEM;
            goto out1;
        }
        /* register each user_data buffer for appropriate access */
        ret = our_setup_mr_sge(conn, conn->user_data[i], options->data_size, access[i], &conn->user_data_mr[i], &conn->user_data_sge[i], options);
        if (ret != 0)
        {
            free(conn->user_data[i]);
            goto out1;
        }
        /* keep count of number of buffers allocated and registered */
        conn->n_user_data_bufs++;
    }         /* for */
    return 0; /* all user_data buffers set up ok */
out1:
    our_unsetup_buffers(conn, options);
out0:
    return ret;
} /* our_setup_user_data */
```

## 配置 mr sge（p109）

```c
/* register a memory addr of length bytes for appropriate access * and fill in scatter-gather element for it
*/
static int
our_setup_mr_sge(struct our_control *conn, void *addr, unsigned int length,
                 int access, struct ibv_mr **mr, struct ibv_sge *sge, struct our_options *options)
{
    /* register the address for appropriate access */
    *mr = our_setup_mr(conn, addr, length, access, options);
    if (*mr == NULL)
        return -1;
    /* fill in the fields of a single scatter-gather element */ 
    our_setup_sge(addr, length, (*mr)->lkey, sge);
    return 0;
} /* our_setup_mr_sge */
```

## 配置 agent buffer（p110）

- agent 需要 1 个 buffer
  - 需要 IBV_ACCESS_LOCAL_WRITE 来从远程client 接收数据
  - 需要 默认本地读权限 向远程client 发送返回数据
- agent 需要 2 个 work request
  - 一个从client 接收 ping 数据
  - 一个向client 返回 pong 数据

## 配置agent buffer 的代码片段（p111）

```c
int our_setup_agent_buffers(struct our_control *conn, struct our_options *options)
{
    int ret;
    int access[1] = {IBV_ACCESS_LOCAL_WRITE};
    /* agent needs 1 buffer for both recv() and send() */
    if ((ret = our_setup_user_data(conn, 1, access, options)) != 0)
        goto out0;
    /* fill in fields of user_data's work requests for agent's data buffer */
    our_setup_recv_wr(conn, &conn->user_data_sge[0], 1,
                      &conn->user_data_recv_work_request[0]);
    our_setup_send_wr(conn, &conn->user_data_sge[0], IBV_WR_SEND, 1,
                      &conn->user_data_send_work_request[0]);
} /* our_setup_agent_buffers */
```



# 主要构成4: 连接管理（p114）

- 目标：建立、维护和释放终端之间的连接
- 与传统socket 类似的 client-server 模式
- 与socket 类似，client 和server 执行的不同的步骤完成“约会”
- 与传统socket 相比需要考虑更多细节
- 在受限场景中，同步操作是可用的；实践中，使用异步操作

## 客户端建立连接的步骤（p116）

1. 创建cm_id
2. 绑定客户端RDMA 设备
   1. 将域名通过DNS 解析为内部地址
   2. 将内部地址对应到 本地RDMA 设备
   3. 得到 抵达server 的网络路由
3. 启动qp
   1. 分配pd
   2. 创建cq
   3. 创建qp
4. 初始化客户端buffer
5. 将客户端与server 连接

## 1.创建 cm_id

## 数据结构金子塔中的 cm_id （p118）

位于连接管理部分，rdma_cm_id

相关函数有：

	- 启动：rdma_create_id
	- 销毁：rdma_destroy_id

## 传输标识 cm_id（p120）

- 所有连接的统一数据结构
- 在连接操作中扮演类似socket 中fd 的角色
- 每个cm_id 拥有一个分配的QP，此连接上的所有传输都使用此QP

## rdma_cm_id 数据结构（p121）

- 目标：将其他数据接口组织在一起的主要结构
- 数据结构：struct rdma_cm_id
- 程序员可见字段：
  - qp
  - ps：port space
  - verbs：驱动和CA 的接口
  - channel：为程序传递连接事件的通道
- 在创建rdma_cm_id 时，需要初始化ps、verbs 和channel

## rdma_cm_id 的创建（p122）

- 函数：rdma_create_id
- 参数：
  - 从CA 向程序报告事件的channel
  - 返回rdma_cm_id 指针的存储空间
  - 用户为此cm_id 定义的id
  - 端口空间port space：RDMA_PS_TCP
- 返回值：
  - 0 表示cm_id 创建成功
  - !=0 表示创建失败的错误码

## rdma_cm_id 销毁（p123）

- 函数：rdma_destroy_id()
- 参数：
  - rdma_create_id() 返回的cm_id 的指针
- 返回值：
  - ==0 表示销毁成功
  - !=0 错误码

## our_create_id() 代码片段（p124）

```c
int our_create_id(struct our_control *conn, struct our_options *options)
{
    int ret;
    errno = 0;
    ret = rdma_create_id(NULL, &conn->cm_id, conn, RDMA_PS_TCP);
    if (ret != 0)
    {
        our_report_error(ret, "rdma_create_id", options);
        goto out0;
    }
    our_trace_ptr("rdma_create_id", "created cm_id", conn->cm_id, options);
    /* report new communication channel created for us and its fd */
    our_trace_ptr("rdma_create_id", "returned cm_id->channel", conn->cm_id->channel, options);
    our_trace_ulong("rdma_create_id", "assigned fd", conn->cm_id->channel->fd, options);
out0:
    return ret;
} /* our_create_id */
```



## 2. 绑定client 到RDMA 设备（p127）

函数金子塔中，连接管理阶段用到的函数有：rdma_resolve_addr	和 rdma_resolve_route

相比传统socket 更加详细的步骤

1. 将server 的DNS 域名（或者ip 地址）解析到addrinfo 和 sockaddr 结构中（可以使用传统的getaddrinfo()）
2. 使用本地路由表，将server 的address 结构绑定到本地RDMA address 和 本地RMDA 设备
3. 建立到server 地址的路由

## 2.1 解析地址（p128）

- 函数：rdma_resolve_addr()
- 参数：
  - cm_id
  - 客户端（本地）sockaddr（可以为空）- src_addr
  - 服务端的（远程）sockaddr - dst_addr
  - 最大等待事件 - timeout
- 返回值：
  - 0 表示成功将cm_id 绑定到RDMA 设备
  - -1 表示错误码
- 系统将填充 非空 src_addr

## 2.2 解析路由（p129）

- 目的：在网络中建立到server 的路由
- 函数：rdma_resolve_route()
- 参数：
  - cm_id
  - 最大等待事件 -  timeout
- 返回值：
  - 0 表示发现了到达远程RMDA 设备的路由
  - -1表示错误码

## our_client_bind() 代码片段（p130）

```c
int our_client_bind(struct our_control *client_conn, struct our_options *options)
{
    struct addrinfo *aptr, hints;
    int ret;
    /* get structure for remote host node on which server resides */
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_INET;
    hints.ai_socktype = SOCK_STREAM;
    ret = getaddrinfo(options->server_name, options->server_port, &hints, &aptr);
    if (ret != 0)
    {
        fprintf(stderr, "%s getaddrinfo server_name %s port %s: %s\n", options->message, options->server_name, options->server_port, gai_strerror(ret));
        return ret;
    }
    errno = 0;
    ret = rdma_resolve_addr(client_conn->cm_id, NULL, (struct sockaddr *)aptr->ai_addr, 2000);
    if (ret != 0)
    {
        our_report_error(ret, "rdma_resolve_addr", options);
        goto out1;
    }
    /* in this demo, rdma_resolve_addr() operates synchronously */
    errno = 0;
    ret = rdma_resolve_route(client_conn->cm_id, 2000);
    if (ret != 0)
    {
        our_report_error(ret, "rdma_resolve_route", options);
        goto out1;
    }
    /* in this demo, rdma_resolve_route() operates synchronously */ /* everything worked ok, fall thru, because ret == 0 already */
out1:
    freeaddrinfo(aptr);
    return ret;
} /* our_client_bind */
```

## 3. 配置qp（p132）

## 函数金子塔上qp相关函数（p133）

位于Transfer Posting层，Setup 阶段使用 rdma_create_qp，breakDown 阶段使用 rima_destroy_qp

## 数据结构金字塔上的 ibv_qp_init_attr（p134）

位于 transfer posting 阶段

## 配置qp 的步骤（p135）

1. 分配pd
2. 创建cq
3. 创建qp

## 配置qp 的代码片段（p136）

```c
int
our_setup_qp(struct our_control *conn, struct rdma_cm_id *cm_id,
struct our_options *options) {
  int ret;
/* create a protection domain */
  ret = our_alloc_pd(conn, cm_id, options); 
  if (ret != 0)
    goto err0;
/* create a completion queue */
  ret = our_create_cq(conn, cm_id, options); 
  if (ret != 0)
    goto err1;
  /* create a queue pair */
  ret = our_create_qp(conn, options); 
  if (ret != 0)
    goto err2;
/* everything worked ok */ 
  return ret;
  err2:
  	our_destroy_cq(conn, options);
  err1:
  	our_dealloc_pd(conn, options); 
  err0:
  	return ret;
} /* our_setup_qp */
```

## qp 配置（p138）

- 函数 rdma_create_qp()
- 参数
  - cm_id
  - pd
  - 初始化qp的结构化数据
- 返回值
  - 0 创建新qp 成功
  - -1 创建失败
- 系统使用新创建的 ibv_qp 实例填充 cm_id 的qp 字段

## ibv_qp_init_attr 数据结构（p139）

- 目的：将初始化一个新的qp 所需的值打包
- 数据结构：struct ibv_qp_init_attr
- 可见字段：
  - 所有字段均可见：新qp 所需的所有字段
- 在将此结构作为参数传递给rdma_create_qp() 用以创建新的qp之前，此结构的所有字段必须被填充上

## 配置qp 参数的代码段（p140）

```c
void
our_setup_qp_params(struct our_control *conn,
                    struct ibv_qp_init_attr *init_attr, struct our_options *options)
{
  memset(init_attr, 0, sizeof(*init_attr)); init_attr->qp_context = conn;
  init_attr->send_cq = conn->completion_queue; 
  init_attr->recv_cq = conn->completion_queue; 
  init_attr->srq = NULL;
  init_attr->cap.max_send_wr = options->send_queue_depth; 
  init_attr->cap.max_recv_wr = options->recv_queue_depth; 
  init_attr->cap.max_send_sge = options->max_send_sge; 
  init_attr->cap.max_recv_sge = options->max_recv_sge; 
  init_attr->cap.max_inline_data = 0;
  init_attr->qp_type = IBV_QPT_RC;
  init_attr->sq_sig_all = 0;
  init_attr->xrc_domain = NULL;
} /* our_setup_qp_params */
```

## 创建qp 的代码段（p141）

```c
static int
our_create_qp(struct our_control *conn, struct our_options *options) {
  struct ibv_qp_init_attr init_attr;
  int ret;
/* set up parameters to define properties of the new queue pair */
  our_setup_qp_params(conn, &init_attr, options);
  errno = 0;
  ret = rdma_create_qp(conn->cm_id, conn->protection_domain,
                       &init_attr);
  if (ret != 0) {
    our_report_error(ret, "rdma_create_qp", options);
  } else {
    conn->queue_pair = conn->cm_id->qp;
    our_trace_ptr("rdma_create_qp", "created queue pair", conn->queue_pair, options);
    our_trace_ulong("rdma_create_qp", "max_send_wr",
                    init_attr.cap.max_send_wr, options);
    our_trace_ulong("rdma_create_qp", "max_recv_wr",
                    init_attr.cap.max_recv_wr, options);
    our_trace_ulong("rdma_create_qp", "max_send_sge",
                    init_attr.cap.max_send_sge, options);
    our_trace_ulong("rdma_create_qp", "max_recv_sge",
                    init_attr.cap.max_recv_sge, options);
    our_trace_ulong("rdma_create_qp", "max_inline_data",
                    init_attr.cap.max_inline_data, options);
  } 
  return ret;
} /* our_create_qp */
```

## qp 的销毁函数（p143）

- 函数：rdma_destroy_qp()
- 参数：
  - rdma_create_qp() 的qp 指针
- 返回值：
  - 0 删除成功
  - !=0 错误码

## 4. 配置客户端buffer （之前已经完成）

## 5. 将client 连接到server（p144）

函数金字塔上 connect management 阶段 rdma_connect 、rdma_disconnect 函数

数据结构金子塔上 rdma_conn_param 

## client 连接到server 的函数（p147）

- 函数：rdma_connect()
- 参数：
  - cm_id
  - 初始化新连接的结构化数据
- 返回值
  - 0 表示连接成功建立
  - -1 表示连接建立失败
- 注意：在成功执行 rdma_connect() 之前必须将 子网管理器（subnet manager）运行起来

## rdma_conn_param 数据结构（p148）

- 目的：将初始化一个新连接所需数值打包
- 数据结构：struct rdma_conn_param
- 可见字段：所有新建连接所需字段
- 在将此结构作为参数传递给rdma_connect() 之前，需要填充所有字段

## 初始化conn 参数的代码片段（p149）

```c
void
our_setup_conn_params(struct rdma_conn_param *params) {
  memset(params, 0, sizeof(*params));
  params->private_data = NULL; 
  params->private_data_len = 0; 
  params->responder_resources = 2; 
  params->initiator_depth = 2; 
  params->retry_count = 5; 
  params->rnr_retry_count = 5;
} /* our_setup_conn_params */
```

## client_connect() 的代码片段（p150）

```c
int
our_client_connect(struct our_control *client_conn, struct our_options *options) {
  struct rdma_conn_param client_params;
  int ret;
  our_setup_conn_params(&client_params);
  errno = 0;
  ret = rdma_connect(client_conn->cm_id, &client_params); 
  if (ret != 0) {
    our_report_error(ret, "rdma_connect", options);
    return ret;
  }
/* in this demo, rdma_connect() operates synchronously */
/* client connection established ok */
  our_report_ptr("rdma_connect", "connected cm_id", client_conn->cm_id, options); 
  return ret;
} /* our_client_connect */
```

## client 的关闭和运行(p151)

- 一旦connect 已经建立连接，便可以开始传输数据（提交 work request 并等待work completion）
- 注意：在iWARP 中client 必须是发送第一条消息的一方
- 当完成传输时，client 必须断开连接，数据结构按照配置的相反顺序进行销毁

## rdma_disconnect()函数（p152）

- 函数：rdma_disconnect()
- 参数：
  - cm_id
- 返回值：
  - 0 表示成功断开连接
  - -1 表示未断开连接
  - EINVAL 表示远端首先断开了连接

## 自底向上配置client 总结（p153）

- rdma_create_id() - 创建 rdma_cm_id
- rdma_resolve_addr() - 将cm_id 与本地RDMA 设备绑定
- rdma_resolve_route() - 解析到达远程server 的路由
- ibv_alloc_pd() - 创建 ibv_pd ，即pd
- ibv_create_cq() -  创建ibv_cq ，即完成队列
- rdma_create_qp() - 创建qp
- ibv_reg_mr() - 创建 ibv_mr 
- rdma_connect() - 创建与远程server 的连接

## client 的main 代码（p154）

```c
int main(int argc, char *argv[])
{
    struct our_control *client_conn;
    struct our_options *options;
    int result;
    /* assume there is an error somewhere along the line */
    result = EXIT_FAILURE;
    /* process the command line options -- don't go on if any errors */
    options = our_process_options(argc, argv);
    if (options == NULL)
        goto out0;
    /* allocate our own control structure to keep track of new connection */
    client_conn = our_create_control_struct(options);
    if (client_conn == NULL)
        goto out1;
    if (our_create_id(client_conn, options) != 0)
        goto out2;
    if (our_client_bind(client_conn, options) != 0)
        goto out3;
    if (our_setup_qp(client_conn, client_conn->cm_id, options) != 0)
        goto out3;
    if (our_setup_client_buffers(client_conn, options) != 0)
        goto out4;
    if (our_client_connect(client_conn, options) != 0)
        goto out5;
    our_trace_ptr("Client", "connected our_control", client_conn, options);
    /* the client now ping-pongs data with the server */
    if (our_client_operation(client_conn, options) != 0)
        goto out6;
    /* the client finished successfully, continue into tear-down phase */
    result = EXIT_SUCCESS;
out6:
    our_disconnect(client_conn, options);
out5:
    our_unsetup_buffers(client_conn, options);
out4:
    our_unsetup_qp(client_conn, options);
out3:
    our_destroy_id(client_conn, options);
out2:
    our_destroy_control_struct(client_conn, options);
out1:
    our_unprocess_options(options);
out0:
    exit(result);
} /* main */
```

## client ping-pong 使用到的函数（p156）

- ibv_post_recv() - 开启接收pong 数据的操作
- ibv_post_send() - 开启发送ping 数据的操作
- ibv_poll_cq() - 获得发送ping 数据任务完成
- ibv_poll_cq() - 获得接收pong 数据任务完成

## client自底向上销毁（p157）

- **rdma_disconnect**
- **ibv_dereg_mr**
- **rdma_destroy_qp**
- **ibv_destroy_cp**
- **ibv_dealloc_pd**
- **rdma_destroy_id**

## ping-pong 的demo 0（p160）

（几个问题）

- 使用Send/Recv 语法
- 同步处理所有cm verbs
  - 没有明确创建 rdma_event_channel
- 忙等待任务完成
  - 没有明确创建 ibv_comp_channel
  - ibv_poll_cq() 不是阻塞的

## Server 的参与者们（p161）

- Listener 监听者
  - 旨在等待来自client 的连接请求触发cm event
  - 使用系统提供的、来自cm event 的信息创建agent
  - 永远不会向client 传送任何数据
- Agent 代理
  - 旨在向一个client 传输所有数据
  - 接收或拒绝client 的连接请求
  - 传输结束时负责断开与client 的连接

## rmda_cm_event 数据结构与相关函数（p162）

struct rdma_cm_event 位于数据结构金字塔的Connection Management 阶段

相关函数有：rdma_bind_addr、rdma_listen、rdma_get_cm_event、rdma_ack_cm_event、rdma_event_str

## server 端用到的新函数（p164）

- 只在连接管理中使用，处于异步的原因
  - rdma_bind_addr
  - rdma_listen
  - rdma_accept
  - rdma_get_cm_event
  - rdma_ack_cm_event
- 前三个函数类似传统socket 中的如下函数
  - bind()
  - listen()
  - accept()

## server 的main 函数代码（p166）

```c
int main(int argc, char *argv[])
{
    struct our_control struct our_options struct rdma_cm_id int result;
    /* assume there is an error somewhere along the line */
    result = EXIT_FAILURE;
    /* process the command line options -- don't go on if any errors */
    options = our_process_options(argc, argv);
    if (options == NULL)
        goto out0;
    /* allocate our own control structure for listener's connection */
    listen_conn = our_create_control_struct(options);
    if (listen_conn == NULL)
        goto out1;
    if (our_create_id(listen_conn, options) != 0)
        goto out2;
    if (our_listener_bind(listen_conn, options) != 0)
        goto out3;
    /* listener all setup, just wait for a client to request a connect */
    if (our_await_cm_event(listen_conn, RDMA_CM_EVENT_CONNECT_REQUEST,
                           "listener", &event_cm_id, options) != 0)
        goto out3;
    /* hand the client's request over to a new agent */
    if (our_agent(event_cm_id, options) != 0)
        goto out3;
    /* the agent finished successfully, continue into break-down phase */
    result = EXIT_SUCCESS;
out3:
    our_destroy_id(listen_conn, options);
out2:
    our_destroy_control_struct(listen_conn, options);
out1:
    our_unprocess_options(options);
out0:
    exit(result);
} /* main */
```



## 自底向上配置 Listener（p165）

- rdma_create_id
- rdma_bind_addr - 将 rdma_cm_id 绑定到本地RDMA 设备（个人笔记：在client 端使用的是 rdma_resolve_addr 绑定）
- rdma_listen() - 建立监听者的储备

## listener_bind 代码片段（p167）

```c
int our_listener_bind(struct our_control *listen_conn, struct our_options *options)
{
    struct addrinfo *aptr, hints;
    int ret;
    /* get structure for remote host node on which server resides */
    memset(&hints, 0, sizeof(hints));
    hints.ai_family = AF_INET;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_flags = AI_PASSIVE; /* this makes it a server */
    ret = getaddrinfo(options->server_name, options->server_port, &hints, &aptr);
    if (ret != 0)
    {
        fprintf(stderr, "%s: getaddrinfo server_name %s port %s %s\n", options->message, options->server_name, options->server_port, gai_strerror(ret));
        return ret;
    }
    errno = 0;
    ret = rdma_bind_addr(listen_conn->cm_id, (struct sockaddr *)aptr->ai_addr);
    if (ret != 0)
    {
        our_report_error(ret, "rdma_bind_addr", options);
        goto out1;
    }
    our_trace_ok("rdma_bind_addr", options);
    our_trace_ptr("rdma_bind_addr", "returned cm_id -> channel",
                  listen_conn->cm_id->channel, options);
    errno = 0;
    ret = rdma_listen(listen_conn->cm_id, OUR_BACKLOG);
    if (ret != 0)
    {
        our_report_error(ret, "rdma_listen", options);
        goto out1;
    }
    our_trace_ok("rdma_listen", options);
    our_trace_ptr("rdma_listen", "returned cm_id -> channel",
                  listen_conn->cm_id->channel, options); /* everything worked ok, fall thru, because ret == 0 already */
out1:
    freeaddrinfo(aptr);
    return ret;
} /* our_listener_bind */
```

## 自底向上 Listener 使用（p169）

## 疑问：为什么要为agent 创建新的cm_id ？

- rdma_get_cm_event() - 获取类型为 RDMA_CM_EVENT_CONNECT_REQUEST 的rdma_cm_event，它将会为agent 创建新的 rdma_cm_id
- rdma_ack_cm_event() - 确认rdma_cm_event 
- rdma_event_str() - 返回枚举值 rdma_cm_event_type 代码对应的可打印字符串

## 自底向上 Listener 销毁（p170）

- rdma_destory_id() - 销毁指定的rdma_cm_id



## our_await_cm_event() 代码段（p172）

```c
int our_await_cm_event(struct our_control *conn,
                       enum rdma_cm_event_type this_event_type, char *name, struct rdma_cm_id **cm_id, struct our_options *options)
{
    struct rdma_cm_event *cm_event;
    int ret;
    if (options->flags & TRACING)
    {
        fprintf(stderr, "%s: %s awaiting next cm event %d (%s) our_control %p\n",options->message, name, this_event_type, rdma_event_str(this_event_type), conn);
    }
    /* block until we get a cm_event from the communication manager */
    errno = 0;
    ret = rdma_get_cm_event(conn->cm_id->channel, &cm_event);
    if (ret != 0)
    {
        our_report_error(ret, "rdma_get_cm_event", options);
        goto out0;
    }
    if (options->flags & TRACING)
    {
        fprintf(stderr, "%s: %s got cm event %d (%s) cm_id %p "
                        "our_control %p status %d\n", options->message, name,cm_event->event, rdma_event_str(cm_event->event), cm_event->id, conn, cm_event->status);
    }
    if (cm_event->event != this_event_type)
    {
        fprintf(stderr, "%s: %s expected cm event %d (%s)\n",
                options->message, name,
                this_event_type, rdma_event_str(this_event_type));
        ret = -1;
    }
    else
    {
        if (cm_id != NULL)
        {
            *cm_id = cm_event->id;
        }
    }
    /* all cm_events returned by rdma_get_cm_event() MUST be acknowledged */
    rdma_ack_cm_event(cm_event);
out0:
    return ret;
} /* our_await_cm_event */
```





## server 参与者 - Agent（p175）



## Agent 概览（p177）

- 使用来自connect request 中的  rdma_cm_id 
- 配置其所有结构
- 调用 rdma_acept() 或 rdma_reject()
- 执行与client 的所有数据操作
- 在数据传输结束时执行 rdma_disconnect()



## 自底向上 Agent 配置（p178）

- 创建our_control
- 使用our_migrate_id() 将连接请求中的rdma_cm_id 移到新的结构体 our_control
- ibv_alloc_pd() - 创建pd，获得ibv_pd 结构体的实例
- ibv_create_cq() - 创建cq，获得ibv_cq 结构体的实例
- rdma_create_qp() - 创建qp，获得ibv_qp 结构体的实例
- ibv_reg_mr() - 创建mr，获得ibv_mr 结构体的实例
- ibv_post_recv() - 开始从client 接收第一条消息
- rdma_accept() - 接受client 的连接请求

## agent 代码片段（p179）

```c
static int
our_agent(struct rdma_cm_id *event_cm_id, struct our_options *options)
{
    struct our_control *agent_conn;
    int result;
    /* assume there is an error somewhere along the line */
    result = EXIT_FAILURE;
    agent_conn = our_create_control_struct(options);
    if (agent_conn == NULL)
        goto out0;
    if (our_migrate_id(agent_conn, event_cm_id, options) != 0)
        goto out1;
    if (our_setup_qp(agent_conn, agent_conn->cm_id, options) != 0)
        goto out2;
    if (our_setup_agent_buffers(agent_conn, options) != 0)
        goto out3;
    /* post first receive on the agent_conn */
    if (our_post_recv(agent_conn, &agent_conn->user_data_recv_work_request[0], options) != 0)
        goto out4;
    if (our_agent_connect(agent_conn, options) != 0)
        goto out4;
```

## migrate_id 代码片段（p180）

```c
int our_migrate_id(struct our_control *conn, struct rdma_cm_id *new_cm_id,
                   struct our_options *options)
{
    /* simple when we have not created our own channel */
    conn->cm_id = new_cm_id;
    new_cm_id->context = conn;
    /* report new cm_id created for us */
    our_trace_ptr("our_migrate_id", "migrated cm_id", conn->cm_id, options);
    return 0;
} /* our_migrate_id */
```



## agent 连接的代码片段（p181）

```c
int our_agent_connect(struct our_control *agent_conn, struct our_options *options)
{
    struct rdma_conn_param agent_params;
    int ret;
    our_setup_conn_params(&agent_params);
    errno = 0;
    ret = rdma_accept(agent_conn->cm_id, &agent_params);
    if (ret != 0)
    {
        our_report_error(ret, "rdma_accept", options);
    }
    else
    {
        /* in this demo, rdma_accept() operates synchronously */
        /* agent connection established ok */
        our_report_ptr("rdma_accept", "accepted cm_id", agent_conn->cm_id, options);
    }
    return ret;
} /* our_agent_connect */
```



## agent 在ping-pong 的使用（p182）

- ibv_poll_cq() - 获取接收ping 数据的任务完成
- ibv_post_recv() - 开启接收下一个ping 数据的操作
- ibv_post_send() - 开启发送pong 数据的操作
- ibv_poll_cq() - 获取发送pong数据的任务完成

## 自底向上 agent 的销毁（p183）

- rdma_disconnect() - 销毁连接到远程server 的连接
- ibv_dereg_mr() - 销毁mr 结构体 ibv_mr 的实例
- rdma_destory_qp() - 销毁qp 结构体 ibv_qp 实例
- ibv_destory_cp() - 销毁completion queue 结构体 ibv_cp 的实例
- ibv_dealloc_pd() - 销毁 pd 结构体 ibv_pd 的实例
- rdma_destory_id() - 销毁rdma_cm_id 实例

## agent 销毁代码片段（p184）

```c
our_trace_ptr("Agent", "accepted our_control", agent_conn, options); /* the agent now ping-pongs data with the client */
if (our_agent_operation(agent_conn, options) != 0)
    goto out5;
/* the agent finished successfully, continue into tear-down phase */
result = EXIT_SUCCESS;
out5 : our_disconnect(agent_conn, options);
out4 : our_unsetup_buffers(agent_conn, options);
out3 : our_unsetup_qp(agent_conn, options);
out2 : our_destroy_id(agent_conn, options);
out1 : our_destroy_control_struct(agent_conn, options);
out0 : return result;
} /* our_agent */
```



## ping-pong 例子 ping-sr-0（p185）

- 使用send/recv 语法实现ping-pong
- 同步处理所有cm verbs
  - 没有显式创建rdma_event_channel
  - rdma_get_cm_event() 是阻塞的
- busy poll 处理completion
  - 没有显式创建ibv_comp_channel
  - ibv_poll_cq() 不是阻塞的

## ping-sr-0  存在的问题（p186）

- 问题 1
  - client 和server 可以指定不同的消息的参数数量和消息的大小
- 问题 2
  - 一端可能会意外断开连接（由于网络异常，程序崩溃或被kill ）
- 问题 3
  - CPU 使用率非常高



## 问题1 的解决方案（p187）

- 在传输数据之前，让client 告知server 要传输的消息的数量和消息的大小
- client 使用rdma_connect() 向server 传输私有数据
- Listener 使用 rdma_get_cm_event() 从RDMA_CM_EVENT_CONNECT_REQUEST 获取到私有数据
- 有listener 启动的agent 使用私有数据设置data_size 和limit 的值

## ping-pong 例子 ping-sr-1（p188）

- 需要修订ping-sr-0 中少量的文件
  - 新的 struct our_connect_info 用来定义私有数据的内容（线路上的传输的数值必须遵从网络字节顺序）
  - 在client 调用rdma_connect() 之前，使用client 使用htonll() 获得的data_size 和limit 的值填充新的结构体
  - 在listener 调用rdma_get_cm_event() 之后，需要从rmda_cm_event 复制私有数据到新的结构体
  - 当rdma_cm_id 被迁移到agent，使用ntohll() 为新结构体的data_size 和 limit 设置值



## 问题2 的解决方案（p189）

- 问题的产生是因为：
  - 完成状态是异步处理的，ibv_poll_cq() 是非阻塞的
  - cm events 是同步处理的，rdma_get_cm_event() 是阻塞的

- 无论completions 还是 cm event 都是从CA 流回程序的状态信息案例
  - completions 传达了数据传输操作的结果
  - cm events 传达了连接管理操作的结果



## 从CA 到程序的状态流示意图（p190）

![image-20210604174348674](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210604174348674.png)

## 状态流类型的不同（p191）

- CA 通过向 completion queue 压入work completion 结构体（ibv_wc）来递交任务完成状态
  - struct ibv_wc 携带相关数据传输操作的结果
- CA 通过向 event channel 压入event 结构体（rdma_cm_event）来递交cm 事件
  - struct rdma_cm_event 携带相关连接管理操作的结果



## 默认状态流类型（p192）

- 任务完成（work completions）- 包含数据传输最终状态
  - 如果 ibv_comp_channel 没有被显式的创建，则所有完成都是异步的
  - ibv_poll_cq() 不会阻塞，只poll
- CM events - 包含连接管理的结果
  - 如果rdma_event_channel 不被显式创建，则所有cm 函数都是同步的
  - rdma_get_cm_event() 会阻塞，不会poll





## 处理来自CA 的状态流（p194）

- 有三种可能
  1. 不要显式的创建一个channel（默认情况）
  2. 显式创建一个channel
  3. 显式创建一个channel，并把它置为 O_NONBLOCK 非阻塞模式
- 上述三种可能情况可以应用到两种类型的流
  - cm 事件流 cm events
  - 完成流 completions
- 结果：有9种混合情况需要考虑

## Ping-Pong 练习 ping-sr-2e（p195）

- 基于ping-sr-1

- 明确创建了rdma_event_channel

- 很多cm 函数是异步操作的

  除了 rdma_get_cm_event()，仍然是阻塞的



##  数据结构金子塔上的rdma_event_channel (p196)

位于connection management阶段

函数金子塔上相关的函数有

	- Setup 阶段：rdma_create_event_channel
	- Use 阶段：rdma_migrate_id
	- 销毁阶段：rdma_destory_event_channel

## event_channel 的数据结构（p198）

- 目的：在连接管理中，向用户程序报告异步事件
- 数据结构：struct rdma_event_channel
- 可见字段：
  - fd，对系统文件描述符进行操作（短整型）
- 系统使用新分配给的系统文件描述符填充 fd 字段

## event_channel 配置（p199）

- 函数：rdma_create_event_channel()
- 参数：无
- 返回值：
  - rdma_event_channel 新实例的指针
- 系统使用新分配给的系统文件描述符填充 fd 字段

## event_channel 销毁（p200）

-  函数：rdma_destory_event_channel()
-  参数
   - rdma_event_channel 的指针

## create event_channel 代码片段（p201）

```c
conn->cm_event_channel = rdma_create_event_channel();
```

## event_channel 使用阶段代码片段 rdma_migrate_id()（p202）

## 疑问：根据现有conn ，生成新的channel，新的cm_id，原因是什么？？？

```c
/* already have a communication identifier,
* migrate it to use a new channel and set its context to be this new conn */
int our_migrate_id(struct our_control *conn, struct rdma_cm_id *new_cm_id,
                   struct our_connect_info *connect_info, struct our_options *options)
{
    int ret;
    /* replace agent's limit and data_size with values from connect_info */
    our_trace_uint64("option", "count", options->limit, options);
    options->limit = ntohll(connect_info->remote_limit);
    our_report_uint64("client", "count", options->limit, options);
    our_trace_uint64("option", "data_size", options->data_size, options);
    options->data_size = ntohll(connect_info->remote_data_size);
    our_report_uint64("client", "data_size", options->data_size, options);
    /* create our own channel */
    ret = our_create_event_channel(conn, options);
    if (ret != 0)
        goto out0;
    errno = 0;
    ret = rdma_migrate_id(new_cm_id, conn->cm_event_channel);
    if (ret != 0)
    {
        our_report_error(ret, "rdma_migrate_id", options);
        our_destroy_event_channel(conn, options);
    }
    else
    {
        conn->cm_id = new_cm_id;
        new_cm_id->context = conn;
        /* report new cm_id created for us */
        our_trace_ptr("rdma_migrate_id", "migrated cm_id", conn->cm_id, options);
    }
out0:
    return ret;
} /* our_migrate_id */
```



## 练习 ping-sr-2e（p204）

- 基于ping-sr-1

- 显式创建rdma_event_channel

- 很多cm 函数现在是异步操作的

  除了 rdma_get_cm_event ，仍然是阻塞的

- 没有解决问题2，因为在每一个异步cm 函数之后只是简单的调用our_await_cm_event()进行等待- 比如 our_bind_client() 那样

## 问题2 的解决方案（p205）

- 我们创建了一个 event channel，但 rdma_get_cm_event() 仍然是阻塞的，因此无法定时poll
- 作为替代，创建了新的用户线程来等待阻塞 rdma_get_cm_event()
  - 线程复制了事件相关信息到结构体our_control 的属性中
  - our_await_completion() 在已经存在的 ibv_poll_cq() 外层的“忙等待”循环中检查这些事件相关信息
  - our_await_cm_event() 与新线程同步

## 线程，而非子进程（p206）

- OFA 资源（连接，数据结构，注册内存区域）不能从父进程继承到子进程
- 类似的，OFA 资源无法在系统调用exec() 后存活
- 一个新的子进程或程序当然可以创建它自己的新的OFA 资源（连接，数据结构，registered memory）
- 然而，监听者无法创建agent 进程，只能创建agent 线程

## cm event 的新线程（p207）

- 在 prototypes.h ：
  - 在结构体 our_control 中增加属性来持有最新的cm event 信息以及mutex_lock 锁，从而更新是原子性的
- 在client.c 和agent.c：
  - 向main() 中增加 our_create_cm_event_thread() 的调用
- 在 process_cm_events.c :
  - 增加方法 our_create_cm_event_thread() 和our_cm_event_thread()
  - 修改方法 our_await_cm_event()
- 在 process_completions.c:
  - 修改our_await_completion()

## 将断开连接事件与断开状态变量统一（p208）

![image-20210606173521679](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210606173521679.png)

## ping-pong 例子 ping-sr-2（p209）

- 创建单独的异步线程来处理cm event
- 异步线程是阻塞的，不会忙等待
- 当线程获取到rdma_cm_event 时，它会从中保存下相关信息，从而使得主线程的忙等待循环可以发现它



## 问题3 的解决方案（p210）

- 问题是由于ibv_poll_cq() 中忙等待completion而导致的CPU 高使用率
- 为了解决此问题，需要消除忙等待
- 为了消除忙等待，需要阻塞直到异步完成事件发生

## 消除忙等待（p211）

- 忙等待 浪费了CPU 循环
  - 主要原因：增加了延迟
  - 如果有大量未使用的CPU 循环，则忙等待是没问题的
- 已经展示了使用一个异步线程处理 cm event，从而消除了忙等待的情况
- 先来来看下当处理事件完成（ibv_poll_cq() ）时如何消除忙等待

## 从CA 到程序的状态流示意图（p212）

![image-20210606174343947](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210606174343947.png)



##  来自CA 的CM 状态流类型（p213）

1. 同步完成状态
   1. 所有cm 函数被阻塞
   2. 没有显式创建channel
2. 大多数异步
   1. 很多 cm 函数 非阻塞，除了 rdma_get_cm_event()
   2. 显式创建channel
3. 完全异步
   1. 大多数cm 函数都是非阻塞，包括rdma_get_cm_event()
   2. 显式创建非阻塞模式的channel

## 完成事件的状态流类型（p214）

1. 完全同步？（笔误？）
   1. 所有完成函数都是非阻塞
   2. 不显式创建channel，需要忙等待
2. 部分异步
   1. 完成事件，以及ibv_get_cq_event()
   2. 显式创建channel，ibv_get_cq_event() 阻塞
3. 完全异步
   1. ibv_get_cq_event() 称为非阻塞
   2. 显式创建非阻塞的channel

## 完成事件channel 的状态流示意图（p217）

![image-20210606175023031](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210606175023031.png)

## 来自CA 的完成状态流

- 目前为止所有demo 都是完全异步的
  - 所有完成函数都是非阻塞的
  - 没有显式创建channel
- 显式创建channel会发生什么
  - 用到的函数不会有变化（与cm channel 的情况不同）
  - 需要新的函数来处理channel
    - ibv_req_notify_cq() 用来请求channel 中的事件
    - ibv_get_cq_event() 用来阻塞所有事件
    - ibv_ack_cq_event() 用来确认事件

## 完成队列的数据结构（p219）

- 目标：使得CA 可以通知程序任务完成出现在了CQ 中
- 数据结构：struct ibv_comp_channel
- 可见属性：
  - fd，但做系统文件标识符（短整型）
  - context，对应cm_id 的verbs 属性
- 当结构体被创建时，系统填写fd 的值

## comp_channel 的配置/销毁（p220）

- 配置
  - 函数：ibv_create_comp_channel()
  - 参数：
    - context：对应cm_id 的verbs 属性
  - 返回值：ibv_comp_channel 新实例的指针
- 销毁
  - 函数：ibv_destroy_comp_channel()
  - 参数：ibv_create_comp_channel() 返回的ibv_comp_channel实例的指针

## 创建comp_channel 的代码片段（p221）

```c
static int
our_create_comp_channel(struct our_control *conn, struct rdma_cm_id *cm_id,
                        struct our_options *options)
{
    int ret;
    /* create a completion channel for this cm_id */
    errno = 0;
    conn->completion_channel = ibv_create_comp_channel(cm_id->verbs);
    if (conn->completion_channel == NULL)
    {
        ret = ENOMEM;
        our_report_error(ret, "ibv_create_comp_channel", options);
    }
    else
    {
        ret = 0;
        our_trace_ptr("ibv_create_comp_channel", "created completion channel",
                      conn->completion_channel, options); our_trace_ulong("ibv_create_comp_channel", "assigned fd",
    }
    return ret;
} /* our_create_comp_channel */
```

## our_await_completion() 代码片段（p222）

```c
int our_await_completion(struct our_control *conn,
                         struct ibv_wc *work_completion, struct our_options *options)
{
    int ret;
    /* wait for next work completion to appear in completion queue */
    do
    {
        errno = 0;
        if (conn->latest_cm_event_type != RDMA_CM_EVENT_ESTABLISHED)
        {
            /* peer must have disconnected unexpectedly */ ret = conn->latest_status;
            if (ret == 0)
            {
                ret = ECONNRESET; /* Connection reset by peer */
            }
        }
        else
        { /* see if a completion has already arrived */
            ret = ibv_poll_cq(conn->completion_queue, 1, work_completion);
            if (ret == 0)
            {
                conn->n_1st_poll_zero++;
                /* no completion here yet, must wait for one */
                ret = our_wait_for_notification(conn, options);
                if (ret == 0)
                {
                    errno = 0;
                    ret = ibv_poll_cq(conn->completion_queue, 1, work_completion);
                    if (ret == 0)
                        conn->n_2nd_poll_zero++;
                    else
                        conn->n_2nd_poll_non_zero++;
                }
            }
            else
            {
                conn->n_1st_poll_non_zero++;
            }
        }
    } while (ret == 0);
    /* should have gotten exactly 1 work completion */ if (ret != 1)
    {
        /* ret cannot be 0, and should never be > 1, so must be < 0 */
        our_report_error(ret, "ibv_poll_cq", options);
    }
    else
    {
        ret = our_check_completion(conn, work_completion, options);
    }
    return ret;
} /* our_await_completion */
```

## our_wait_for_notification() 代码片段（p225）

```c
static int
our_wait_for_notification(struct our_control *conn, struct our_options *options)
{
    struct ibv_cq *event_queue;
    void *event_context;
    int ret;
    /* wait for a completion notification (this verb blocks) */
    errno = 0;
    ret = ibv_get_cq_event(conn->completion_channel, &event_queue, &event_context);
    if (ret != 0)
    {
        our_report_error(ret, "ibv_get_cq_event", options);
        goto out0;
    }
    conn->cq_events_that_need_ack++;
    if (conn->cq_events_that_need_ack == UINT_MAX)
    {
        ibv_ack_cq_events(conn->completion_queue, UINT_MAX);
        conn->cq_events_that_need_ack = 0;
    }
    /* request notification when next completion arrives into empty completion queue. * See examples on "man ibv_get_cq_event" for how an “extra event” may be
* triggered due to a race between this ibv_req_notify() and the subsequent
* ibv_poll_cq() that empties the completion queue.
* The number of occurrences of this race will be recorded in completion_stats[0] * and will be printed as the value of work_completion_array_size[0].
*/
    errno = 0;
    ret = ibv_req_notify_cq(conn->completion_queue, 0);
    if (ret != 0)
    {
        our_report_error(ret, "ibv_req_notify_cq", options);
    }
out0:
    return ret;
} /* our_wait_for_notification */
```

## ping-pong 例子 ping-sr-3（p227）

- 在显式创建的channel 上使用通知来阻塞完成事件
  - 通过消除忙等待降低CPU 使用率
  - 但往返的延迟上升了

## ping-pong 例子 ping-sr-4（p228）

- 将cm event channel 和completion channel 都设为非阻塞模式
- 使用POSIX 的poll() 来等待cm event 和completion event
- 与普通socket 同步编程的风格很接近
- 不需要额外的线程



## Blast 例子 blast-sr-2（p229）

## 使用Send/Recv 的Blast 示意图（p230）

![image-20210606180838556](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210606180838556.png)

## Blast 练习blast-sr-2e（p232）



## Send/Recv 结语（p233）

- 介绍了很多OFA 函数和OFA 数据结构
- 它们大多数在 RDMA_WRITE和RDMA_READ 中也适用
- 接下来讨论：
  - 两个连接管理层面的函数
  - 其他函数

# 主要构成5：杂项（p235）

## 其余CM 函数（235）

- connetion management阶段
  - rdma_get_local_addr()
  - rdma_get_peer_addr()
- 与传统socket 类似的
  - getsockname()
  - getpeername()

## 数据结构金字塔中 杂项数据结构（p236）

- ibv_conertxt
- ibv_device
- ibv_device_attr



## 结构体ibv_context（p238）

- 与已经介绍过的很多其他结构体有关
- 包含两个重要属性
  - ops：big "jump table" for the verbs
  - device：代表逻辑设备的结构体
- 通常不被程序员直接使用
- 下个demo 中将会看到

## 结构体 ibv_device（p239）

- CA 的基本代表
- 包括有趣的属性：
  - ops
  - name
  - dev_name
  - node_type
  - transport_type
- 通常不被程序员直接使用
- 下面例子中可以看到

## 获取RDMA 设备的列表（p240）

- 目的：决定系统中RDMA 设备的名称（以及其他信息）
- 函数：
  - rdma_get_devices()
- 参数：
  - RDMA 设备返回的整型值
- 返回值：ibv_context指针列表的指针

## 空闲设备列表（p241）

- 函数：rdma_free_devices()
- 参数：
  - list：rdma_get_devices() 返回的指针

## ibv_query_device()（p242）

- 目的：返回一个带有RDMA 设备信息的结构体
- 函数：ibv_query_device()
- 参数：
  - ibv_context 实例的指针
  - ibv_device_attr 实例的指针
- 系统将context 指定的RDMA 设备的相关信息填充到 device_attr 属性中

## ibv_device_attr 结构体（p243）

- 有很多对程序员可见的属性
  - 可查阅 <infiniband/verbs.h> 中的定义
- 每个属性提供有关设备的一种信息
  - 大多数属性包含了一个资源的最大数量
  - 对于标识和管理 目的很有用

## 示例设备（p244）

- 独立的用来打印主机上可用RDMA 设备列表的程序

- 使用了

  - rdma_get_devices()
  - rdma_free_devices()
  - ibv_query_device()

  以及

  - struct ibv_device_attr

## Send/Recv 结语（p245）

- 几乎所有已经提到的结构体和函数在RDMA_WRITE 和 RDMA_READ 传输中仍会用到
- 但RDMA READ/WRITE 的内容有所不同

## 完成的数据结构金字塔（p246）



