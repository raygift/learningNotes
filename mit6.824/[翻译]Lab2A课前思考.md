Lecture 6

Suppose we have the scenario shown in the Raft paper's Figure 7: a cluster of seven servers, with the log contents shown. The first server crashes (the one at the top of the figure), and cannot be contacted. A leader election ensues. For each of the servers marked (a), (d), and (f), could that server be elected? If yes, which servers would vote for it? If no, what specific Raft mechanism(s) would prevent it from being elected?

课程6
- 问题
假设遇到raft 论文中图7的场景：集群共7个server ，server 拥有的日志如图所示。第一个server 崩溃后无法被其他server 联系上。此时新一轮选举开始。图中 a , d, f节点，哪个可能被选为leader？ 如果可能被选为leader ，他们分别会获得哪些节点的选票？如果不可能成为leader，是由于Raft 的哪些机制避免了其当选？

![image-20210123174317858.png](https://rayalioss.oss-cn-shanghai.aliyuncs.com/img/image-20210123174317858.png)

图7：当位于图最上方的leader 统治时，follower 的日志可能是下方a 到f 几种情况。每个被分割的格子代表一个日志记录；格子中的数字是记录的term号。Follower 可能会丢失记录（如a-b），可能会包含额外未提交的记录（如c-d），也可能两种情况同时存在（如e-f）。例如场景f 可能是这样发生的：term 2 中的f 作为leader 追加了几条记录，在提交记录前崩溃了；崩溃后很快又重启，在term 3重新成为leader，并再次追加了几条记录；在term 2和term 3 中追加的记录尚未提交时，此节点再次崩溃 并在后续几个term 中未能恢复。

- 分析
1. 针对节点a ，判断超时后term + 1，即currentTerm 为 7，向其他节点发送RequestVote RPC；其他节点对a 发出的RequestVote RPC做如下判断：

   - 将currentTerm 与自身term比较，若小于自身term，则拒绝投票；本例中b\c\d\e\f 均不会因此条件拒绝向a 投票

   - 判断candidate的日志是否至少与receiver的日志一样新；根据5.4.1，对比日志记录新旧的规则为：对比日志中最后一条记录，记录term更大者代表日志更新。若最后一条记录的term 一样，则谁的记录更长代表日志更新。本例中a的最后一条记录term 为6，因此 d 会拒绝向a 投票；a最后一条记录的index 为9，因此c 会拒绝向a投票

    综上，节点a 最多可获得4张选票，分别来自 b、e、f及自己，超过半数，因此有可能成为leader

1. 针对节点b，与节点a类似进行判断，由于currentTerm=4+1=5（比a、c、d、f的term小），最后一条记录的index为4（比其他所有节点都小），得到最多可获得1张选票，无法超过半数，因此不可能成为leader
2. 针对节点c 进行类似判断，由于currentTerm= 6+1 =7（不小于其他任何节点），最后一条记录的index 为11（比d 的小），最多可得到5张选票，超过半数因此可能成为leader
3. 针对节点d 进行类似判断，由于currentTerm=7+1 （不小于其他任何节点），最后一条记录的index 为12（不小于其他任何节点），最多可得到6张选票，超过半数因此可能成为leader
4. 针对节点e ，由于currentTerm= 4+1 =5（小于a、c、d），最后一条记录的term为4，index 为7（比a、c、d 的小），最多可得到3张选票，未超过半数无法当选
5. 针对节点f，由于currentTerm= 3+1 =4（小于a、c、d），最后一条记录的term 为3，小于其余所有节点，最多只能得到1张选票，因此无法当选