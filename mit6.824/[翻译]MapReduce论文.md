## 摘要
---
MapReduce 是一个编程模型，实现了处理和生成大量数据集。开发人员指定一个map函数来处理kv对，生成一组kv对中间结果；还会指定一个reduce函数将所有中间结果中具有相同key的value值merge起来。如本文将要展示的那样，很多现实中的任务都可以用此模型表示。

使用此中函数式编程风格的程序自动实现了并行化，运行在庞大的集群上。运行中的系统将输入的数据分片，在一组机器上调度程序的执行，处理机器的异常，并管理着机器内部的通信需求。这使得开发者可以在没有任何并行和分布式系统的基础的前提下，利用到庞大的分布式系统资源。

我们的`MapReduce`运行在庞大的、由普通服务器组成的集群上，且集群是高度可扩展的：一个典型的mapReduce任务可以处理上千台机器上的数TB数据. 开发者使用系统也非常简单：谷歌的集群上每天有上百个MapReduce程序被创建，上千个MapReduce任务被执行。

## 1   介绍

过去五年，本文作者以及其他很多就职于谷歌的同事实现了上百个特殊目的的计算，处理了大量原始数据，如爬虫爬取的文档，web请求日志等等，为了处理各种各样的衍生数据，如反转的索引，各种web文档图结构的变量，每个主机爬取到的页面数量汇总，指定某天连续查询的集合等。这些计算大部分理论上是很简单直接的。然而，输入的数据通常很大因此计算必须在成百上千台机器上分布执行，从而在可接受时间内执行完毕。如何将计算并行，如何分布数据，以及如何处理失败情况，这些问题使原本简单的计算问题变得复杂，处理这些问题需要更多复杂的代码。

为了应对这种复杂问题，我们设计了一种新的抽象来帮助我们完成快速执行简单计算的目标，并且隐藏掉并行，容错、分布式数据、负载均衡等复杂细节。这种抽象受到Lisp和其他函数式编程语言中map和reduce思想的启发。我们意识到大部分计算需要为每个输入的逻辑“record”应用一个map操作，从而可以计算处理中间过程的kv键值对，需要为拥有同一个key的所有value应用一个reduce操作，从而可以将衍生数据聚合。通过对有用户指定map和reduce操作的函数式模型的使用，我们可以轻松实现大量计算任务的并行，并且可以实现将重新执行作为主要容错机制。

此项工作主要的贡献是提出了自动化并行和大规模分布式计算的一个简单却又高效的接口，结合此接口的实现获得了在由普通PC组成的庞大集群上的高性能。

第二章描述了基本的编程模型，并提供了几个案例。第三章描述了MapReduce接口的一个基于集群计算环境的实现。第四章描述了几个我们发现有用的编程模型精选。第五章针对多种类型的任务展示了我们的成果性能卓越。第六章探索MapReduce在Google的使用，包括我们以它为基础重写产品索引系统的经验。第七章讨论相关及未来工作。

## 2 编程模型

计算接受一组 key/value键值对集合作为输入，并输出一组key/value键值对。MapReduce的使用者通过两个函数表达此计算：Map函数和Reduce函数。

Map函数由使用者定义，接受输入键值对，并生成一组中间过渡的key/value键值对。MapReduce库将所有中间过渡的value根据相同的key I拼成一组，并将它们传给Reduce函数。

Reduce函数也是由使用者定义，接受一个中间过渡的key I以及该key对应的value集合。Reduce将这些value合并得到更小的value的集合。一般每次调用Reduce会得到0或1个输出值。中间过渡的value值们通过一个迭代器被提供给用户的reduce函数。这使得我们可以处理很多条因为太大无法放入内存的value集合。

### 2.1 案例

word-count问题，计算大量文件中每个单词出现的次数。伪代码如下：

```
map(String key, String value):
    // key: document name
    // value: document contents for each word w in value:
    EmitIntermediate(w, "1");

reduce(String key, Iterator values): // key: a word
    // values: a list of counts
    int result = 0;
        for each v in values:
            result += ParseInt(v);
        Emit(AsString(result));
```
map函数给每个单词增加一个出现次数（伪代码中用“1”表示）。reduce函数将每个单词所有的出现次数求和。

另外，程序员通过代码给***mapreduce特定对象***提供输入输出文件的文件名，以及可选的调节参数。调用MapRecuce函数时，将特定对象传递给MapReduce函数。用户代码和MapReduce库被链接到一起（mapreduce是用C++实现的）。附录A包括了本例完整的程序代码

### 2.2 类型

此前的伪代码用string类型的输入和输出，而理论上由用户定义的map和reduce 函数已经被分配了类型：
```
    map     (k1,v1)       ->  list(k2,v2)
    reduce  (k1,list(v2)) ->  list(v2)
```

？？？例如：输入的key和value是从与输出的key和value不同的域抽取得到的。更进一步，临时key和value来自与输出key和value相同的域。

本文的C++实现给用户定义函数的输入和输出是string类型，交由用户编程完成string和适当类型的转换。

### 2.3 更多案例

如下几个简单有趣的程序可以很轻松的通过mapreduce模型加速。

**分布式检索**：如果与提供的模式匹配，map函数增加一行。reduce函数是一个恒等函数，仅将传入的临时结果数据复制到输出文件中。

**计算URL访问频率**：map函数处理web页面请求的日志，输出 <URL,1>。reduce函数将所有相同URL的value求和，得到一个<URL,求和结果>键值对。

**反转web链接图**：针对被命名为source的页面中指向target的每一个链接，map函数输出<target,source>键值对。reduce函数将给定target的所有source拼接起来，输出键值对：<target,list(source)>

**每个Host的检索词向量**：一个检索词向量代表出现在一个文档或一组文档中最重要的单词，用<word,frequency>键值对表示。map函数为每个输入文件输出 <hostname,term vector>键值对（hostname从文件URL中提取得到）。reduce函数针对每个host，拿到所有向量。将向量加到一起，剔除出现不频繁的term，输出<hostname,term vector>键值对。

**倒排索引**：map函数处理每个文件，得到一系列<word,document ID>键值对。reduce函数接收针对特定word的所有键值对，将key相同的document ID排序得到<word,list(docuement ID)>键值对。所有输出键值对的集合构成了一个简单的倒排索引。可以很轻松的增强此算法来跟踪单词的位置。

**分布式排序**：map函数从每个record中提取key，得到<key,record>键值对。reduce函数将所有键值对原样输出。此计算依赖4.1节阐述的分区工具，以及4.2节阐述的排序特性。

## 3 实现

### 3.1  Execution Overview

通过自动将输入数据分片到长度为M的集合中，Map调用分布在多台服务器。输入的分片可以在不同服务器上并行被处理。通过使用分片函数（如hash(key)modR）将中间结果的key分成R片，Reduce的调用也是分布式的。分片数量R 和分片函数由开发者指定。

图1 展示了我们实现的MapReduce的整体流程。当用户的程序调用MapReduce 函数时，后续动作会依次发生（图1 中被编号的标签对应下述的步骤）：

1. MapReduce库首先将输入文件切分成M 片，一般每片有16MB 或64MB （大小由用户使用可选参数控制）。然后在集群的服务器中启动很多程序的副本。

2. 程序副本中，其中一个是master。其余的是执行master分配给的任务的worker。存在M个map任务和R个reduce任务待分配。master选择空闲的worker并分配给它一个map任务或是reduce任务。

3. 被分配给map任务的worker读取统一输入分片中的内容。从输入数据中解析key/value对，并将每个键值对传递给用户定义的Map函数。由Map函数产生的中间结果kv对被缓存在内存中。

4. 内存中的kv对最终被写入本地磁盘，通过分割函数分到R个区域中。存储到本地磁盘的位置信息被回传给master，master将位置信息进一步提供给执行reduce的worker。

5. 当reduce worker接收到中间结果的kv对在磁盘的位置信息后，使用RPC从master的本地磁盘读取这些数据。读取到kv对后，worker将kv对根据key排序，使得相同的key可以连续出现。因为可能有很多不同的key被map到一个reduce任务中，因此排序是必要的。如果中间结果数据两太大，内存中无法处理排序时，可能需要使用外部排序。

6. reduce worker 将排序过的中间结果数据枚举，对枚举到的每个唯一的key，将该key和其对应的value集合传递给用户的Reduce函数。Reduce函数的输出结果被追加到这个reduce任务的最终输出文件中。

7. 当所有map和reduce任务被执行完毕，master会唤醒用户程序。此时用户程序中调用的MapReduce会返回。

成功执行完毕后，mapreduce执行得到的输出保存在R个输出文件（每个reduce任务对应一个文件，文件名由用户指定）。一般而言，用户无需将R个输出文件合并为一个，它们经常被当作另一个MapReduce任务的输入，或者在其他可以处理被切分成多个输入文件的分布式程序中当作输入使用。

### 3.2 Master的数据结构

master有多种数据结构。针对每个map和reduce任务，master保存状态信息（空闲、处理中、处理完成），并保存worker服务器的标识（for non-idle 任务）。

master 是中间结果存储位置信息从map任务传递给reduce任务的管道。因此，针对每个完成的map任务，master存储了其产生的R个中间文件的存储位置和大小信息。当map任务执行完成时，这些存储位置和文件大小信息被更新。这些信息被增量推送给正在执行reduce任务的reduce worker。

### 3.3 容错

MapReduce 库被设计用成百上千的机器来处理大量数据，因此对于机器异常的容错必须足够优雅。

### worker 失败

master 定期ping每个 worker。如果在固定的时间内未收到worker的响应，master会将此worker标记为已失败。此worker完成的所有map任务将会被重置到初始的idle状态，从而合理地分配给其他worker。类似的，失败的worker上正在执行的任何map或reduce任务同样被重置为idle，以便被再次分配。

发生失败时，已经完成的map任务被重新执行是因为这些map任务的执行结果是存储在失败worker本地磁盘的，这些中间结果无法被其他worker访问。同样情况下已经完成的reduce任务无需重新执行，因为它们的执行结果是存储在GFS中。

如果一个map任务最开始由worker A执行，后来由于worker A失败，任务被worker B执行，map 任务的重新执行将会告知所有执行reduce 任务的worker。任何尚未从worker A读取数据的reduce任务 将会改为从worker B读取数据。

MapReduce 能够从大范围的worker失败中恢复。例如，在 mapreduce操作期间，由于网络维护导致集群中80台服务器同时无法访问数分钟。mapreduce 的master 只需简单的将无法访问的服务器完成的map任务重新执行，并继续执行后续工作，最终完成mapreduce操作。

### master 失败

很容易让master 记录上面所描述的master数据结构的定期检查点。如果master 任务挂掉，可以从最后一个检查点状态启动一个新的副本。然而，如果只有一个master，此时master挂掉是很难恢复的；因此如果master失败，我们的实现会中断mapreduce计算。客户端可以在需要时检查条件并重试mapreduce操作。

### 错误产生中的语义

当用户提供的 map 和 reduce 操作对于输入数据是确定性的函数时，分布式的实现会与执行正常的程序得到一样的输出。

要达到上述目标，依赖map 和reduce 任务输出的原子提交操作。每个正在执行的任务将输出写入私有的临时文件。一个reduce 任务生成一个结果文件，一个map 任务生成 R个中间结果文件（针对每个reduce 任务生成1个中间结果文件）。当一个map任务完成时，worker向 master发送信息，信息内容包括了 R个临时文件的文件名。如果master收到的信息对应一个已经完成的map任务，这些信息将会被忽略。否则，master将收到的R个中间结果文件信息记录到数据结构中。

当一个reduce 任务执行完成时，reduce worker执行原子操作，将临时的输出文件重命名为最终输出文件。如果多台服务器执行同样的reduce任务，针对同一个名称的结果文件，将会被执行多次重命名操作。我们依赖底层文件系统提供的重命名操作的原子性，来保证最终的结果有且仅有一个reduce task生成的结果文件。

大部分map 和 reduce 操作都是确定性的，我们的场景和顺序执行十分类似的这一事实，让程序开发人员很容易的预测他们所写程序的行为。当map 和 reduce 操作存在不确定性时，我们提供的场景即使变弱，但仍然是可以被理解的。在不确定的操作场景中，特定reduce 任务 R1的输出 与 顺序执行不确定程序R1所得到的结果是相等的。然而，另外一个reduce 任务 R2 的输出，可能与顺序执行 R2 的另一个不同的不确定性任务的输出一致。（In the presence of non-deterministic operators, the output of a particular reduce task R1 is equivalent to the output for R1 produced by a sequential execution of the non-deterministic program. However, the output for a different reduce task R2 may correspond to the output for R2 produced by a different sequential execution of the non-deterministic program）

考虑 map 任务 M 和reduce 任务 R1 和 R2. 前提是只执行了一次Ri，将 Ri 执行的结果记为 e(Ri)。由于e(R1)可能已经读取了 由 M的一次执行而产生的输出，同时 e(R2)可能已经读取了由 M 的另外一次执行产生的输出，此时较弱的语义产生。


