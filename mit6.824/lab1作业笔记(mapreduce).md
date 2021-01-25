map reduce论文中3.1章节的mapreduce过程相关表述，在mit6.824的课程作业代码中是怎样实现的？

如：

1. input files是否需要切分成 M 片？
2. map 和 reduce 任务，是由master 分配的还是由worker 主动请求的？
3. RPC只请求读取中间结果文件？
4. map 和 reduce 任务，是否由RPC方式调用？



经过分析，以上问题得到如下结论：

1. lab1给出的串行执行的mapreduce代码实现中，程序读取每个输入文件，并传递给Map函数，并未将文件分片。

2. mit6.824作业说明中提到“Each worker process will ask the master for a task, read the task's input from one or more files, execute the task, and write the task's output to one or more files. ”，由此可判断，worker是主动向master请求任务的，worker需完成的工作有：向master请求任务，读取任务对应的文件，执行map/reduce任务，并将任务的执行结果写入输出文件。
3. “The workers will talk to the master via RPC.”，结合第2个问题worker需主动向master请求任务，推断worker与master之间的RPC调用，并非直接读取master持有的中间结果文件，而是请求任务，获得的任务重包含了需读取的文件位置信息。
4. 由第2个问题可知，map和reduce任务是需要由RPC方式实现的。



### master 和worker 的分工

---

master只管创建任务，检查任务执行状态

worker负责使用map和reduce函数，进行任务的实际处理，且worker无需使用 go协程实现并发，因为作业目标为：

“There will be just one master process, and one or more worker processes executing in parallel. ”，并且在说明如何运行代码时，要求为“In one or more other windows, run some workers”，也即每次执行mrworker.go，可视为只执行一个worker。



### worker 执行任务的逻辑

---

1. worker 内部代码是否先请求map任务，当master 反馈没有未处理的map 任务时，再向master 请求reduce 任务？
2. 接第1个问题，worker 内部的代码是否仍是串行，作业要求map 后得到的中间结果文件命名中包括map 和 reduce 任务的编号：“A reasonable naming convention for intermediate files is `mr-X-Y`, where X is the Map task number, and Y is the reduce task number”，但中间结果是map 任务执行完得到的，此时规定reduce 编号的意图是否为将此文件预先分配reduce 任务编号，以便worker 在请求reduce 任务时，可以直接读入相应的输入文件？

3. 请求reduce任务是持续循环请求直到所有reduce任务全部执行完成？



### mit6.824的文件命名要求

---

- 每个reduce 任务，对应一个结果文件 mr-out-X

- worker需将第 X 个 reduce 任务的执行结果，存放在 mr-out-X 文件中

- 一个 mr-out-X 文件中，需要针对每个 Reduce 函数的输出，存在一行数据



### map和reduce分工

---

1. map读取输入文件，对文件内容进行分词，单词出现一次，map函数就生成一个 key/value存入中间结果文件

2. reduce 读取中间结果文件，将文件中的 相同key对应的value求和，得到每个key最终的value

3. 中间结果文件的个数为 M * R，M为map任务数量，R为reduce任务数量

4. 中间结果文件中的内容为：

   - 经过排序的key/value，相同的key连续出现，但value值全部为“1”，并未求和

     

同一个key 可能出现在多个中间结果文件中，因为每个输入文件对应的中间结果文件互不相同

同一个key 也可能出现在多个最终结果文件中，因为reduce只是将每个中间结果文件内的相同key做value的求和，而不会跨多个中间结果文件求和



## map 任务状态切换

初步判断map任务有如下四种状态：

0. not ready：针对 reduce 任务，当1个map任务执行完成，会创建对应的R个reduce 任务，并记录每个reduce 任务对应的 中间结果文件，但此时不保证所有map任务已执行完，因此 reduce 任务尚不能分配给worker 去执行，否则可能会出现对应的reduce 任务读取的中间文件个数缺少
1. todo：初始化完成的map 任务，在wc场景下每个map任务对应一个输入文件
2. doing：已被分配给worker的map 任务
3. done：已被worker 执行完毕的map 任务
4. timeout：任务被分配给worker ，但超时未完成的任务；master需要定期检查 任务是否 timeout，并将已超时未完成任务重新分配



状态转换：

a. todo -> doing :任务被分配给worker时

b. doing -> done :



## 先完成map task 置为done，还是先 新建 reduce task？

1. 先完成 map task 更新状态为done，再新建 reduce task

   此时可能map task 更新为done 后，worker crash，但由于 map task 状态为done，因此master无法正常判断 task超时，也就无法将crash worker所持有的map task 重新分配给其他worker

2. 先新建 reduce task ，再更新 map task 状态为done

   此时可能在创建 reduce task后，map worker 发生crash，当master判断map task 保持doing状态 超时后，map task 被重新分配给其他worker，但由crash 的worker新建的reduce task以及对应的中间文件却无法回退删除，导致读取错误的中间结果文件

-  解决办法
  1. 先完成map task 更新状态 为 done，再新建reduce task，同时将 master判断 map task超时的条件改为 ：map task 保持doing 超过规定时间，或者map task即使为done，但对应的reduce task 保持notready 状态且超过规定时间
  2. 



## map 任务 为了处理 worker crash 的情况，是否需要先将 map 的结果存入临时文件，再rename为中间结果文件

mapreduce 论文中提到的，为了保证容错，保证异常出现后对最终结果没影响，需要 map 和 reduce 的任务输出结果是原子性提交的。

map 任务的输出结果提交包括两个动作：

	1. 生成中间结果文件
 	2. 更新 map 任务状态

reduce 任务的输出结果提交包括：

1. 生成最终结果文件
2. 更新reduce 任务状态


论文里 map 任务的结果首先缓存在worker 本地内存中，然后使用分区函数周期性的写入worker的本地磁盘R个文件（即中间结果文件），需要消费时，reduce worker 通过rpc 请求从map worker 所在主机磁盘读取中间结果文件


lab1提示为了确保 crash的worker生成的文件不被其他worker读取到，需要使用tmp文件，并在被写入完成时原子性执行rename操作

（如何判断写入完成：map任务读取完所有输入文件的内容，并将处理结果写入中间结果文件）



### Q：master每次判定一个worker执行任务超时后，其生成的文件删除掉不就可以了吗？？

A：
worker执行map任务超时，有可能是map worker已经crash，已经产生的任务输出由于存储在worker本地私有存储，worker crash后就无法被访问到，因此也无法进行删除；

worker 执行 reduce 任务超时，由于reduce 任务的结果存储在全局文件系统（如GFS）上，因此已执行完成的 reduce 任务无需再次执行；而未执行完成的reduce 任务，reduce任务已经开始执行，且执行的结果已经开始写入最终结果文件，此时worker crash的话，会导致 存在一个最终结果文件，此reduce 任务再分配给其他worker时，worker会遇到困扰：现存的最终结果文件是否内容正确？

这时可能就需要每次reduce 任务启动时，如果发现 已经存在对应的最终结果文件，就直接删除掉再重新生成。（不知道为啥没有选择此方式处理异常）



### Q：那是否将 master数据结构中记录的文件位置和文件名删除掉，就可以避免worker去读取已crash的worker的输出文件？

A：
1. 当reduce worker 失败crash时，无需进行master 数据结构记录删除操作，只需要重新执行一次 reduce 任务即可；结果文件个人感觉可以删除，尚不清楚为何论文未选择删除文件的方式处理异常。
2. 当 map worker 失败crash时，若map 任务尚未执行完，但 master 已经记录下 crash的 worker 生成的一些中间结果文件信息，如果出现此情况，由于中间结果存在worker的本地，就需要删除master中相关文件位置信息。但是，如果能 worker 输出结果的提交是原子性的，更新map 任务“已执行完”和记录中间结果文件信息两个操作要么都发生，要么都不发生，就无需再单独维护master中map任务的状态和记录的中间结果位置。



lab1中区别于mapreduce原论文的点在于：

1. lab1中 worker 使用同一个文件系统
2. lab1中不考虑master failed的情况


先使用临时文件再重命名为中间结果文件，使得一个map任务被判断为超时重新分配后，再次生成的临时结果文件与此前执行的文件隔离开


提交map任务的结果包括三个操作：

1. 重命名操作

2. 提交新reduce操作

3. 更新map任务状态操作

   
由于在lab1中未考虑master crash的情况，worker将提交map任务的三个操作实现交由master完成，可以保证三个步骤是原子性的，不会因为执行map任务的worker异常导致存在无效文件或无效的结果记录

提交 reduce 任务执行结果包括两个操作：
1. 重命名操作
2. 更新reduce 任务状态操作

类似map提交，worker将通过rpc调用master来完成最终reduce任务提交执行结果的两个操作，保证两个操作的原子性