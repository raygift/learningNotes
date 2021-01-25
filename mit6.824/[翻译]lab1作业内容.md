tips

- 代码git clone后，无需添加go.mod，先修改GO111MODULE=auto, 然后直接进入src/main执行 go build -buildmode=plugin ../mrapps/wc.go 即可

your job翻译

任务内容为实现一个分布式MapReduce, 包括master和worker两部分。master进程只能有一个, worker进程可以有一个或多个并行. 在真实的系统中, workers会在一群不同服务器上运行, 但本实验中所有worker将在单个机器运行. workers与master通过RPC通信. 每个worker进程向master请求一个任务, 从一个或多个文件中读取任务的输入, 执行任务, 并将任务的输出写入一个或多个文件中. master必须关注节点在规定时间内未完成分配给它任务的情况（在本实验中, 时间为10秒），并将同样的任务给到其他worker.

已经有一些代码帮助你上手. master和worker的"main" routines位于 main/mrmaster.go 和 main/mrworker.go; 不要修改这些文件. 你应该将你的代码放到 mr/master.go, mr/worker.go和 mr/rpc.go.

在word-count MapReduce应用运行你代码的方法如下.

首先, 确认word-count 插件是最新构建的:
```shell
$ go build -buildmode=plugin ../mrapps/wc.go
```

在main目录, 运行master.
```shell
$ rm mr-out*
$ go run mrmaster.go pg-*.txt
```

mrmaster.go 的 pg-*.txt 参数是输入的文件; 每个文件对应一个"split", 并且是一个Map 任务的输入.

在一个或多个windows中, 执行一些workers:
 ```
$ go run mrworker.go wc.so
 ```

当workers和master执行结束, 查看mr-out-* 中的输出. 当你完成此实验, 输出文件排序结合后应该和顺序的输出匹配, 如下所示:

```
$ cat mr-out-* | sort | more
A 509
ABOUT 2
ACT 8
```
main/test-mr.sh 是一个测试脚本, 用来检查输入为 pg-xxx.txt文件时 MapReduce 应用生成的 wc 和 indexer. 脚本还是检查你的代码实现运行Map 和Reduce 任务是并行的, 并且代码可以从执行任务时崩溃的workers恢复.

如果现在执行测试脚本, 将会由于master 不能完成而挂起
```
$ cd ~/6.824/src/main
$ sh test-mr.sh
*** Starting wc test.
```

可以将mr/master.go 中的Done 函数 ret := false 改为true, master将会立刻退出. 然后:
```
$ sh ./test-mr.sh
*** Starting wc test.
sort: No such file or directory
cmp: EOF on mr-wc-all
--- wc output is not the same as mr-correct-wc.txt
--- wc test: FAIL
$
```

test 脚本期望针对每个reduce 任务有一个命名为mr-out-X 的输出文件. mr/master.go 和mr/worker.go中由于实现为空, 不会产生这些文件(或者产生其他任何操作), 因此test 脚本会失败.

当完成实验后, test 脚本应该得到类似如下的输出:

```
$ sh ./test-mr.sh
*** Starting wc test.
--- wc test: PASS
*** Starting indexer test.
--- indexer test: PASS
*** Starting map parallelism test.
--- map parallelism test: PASS
*** Starting reduce parallelism test.
--- reduce parallelism test: PASS
*** Starting crash test.
--- crash test: PASS
*** PASSED ALL TESTS
$
```

仍会有一些来自Go RPC 包的报错, 忽略这些信息, 无需关注:
```
2019/12/16 13:27:09 rpc.Register: method "Done" has 1 input parameters; needs exactly three
```

A few rules翻译
- map phase应该将中间结果的keys分散到nReduce的reduce任务桶里, nReduce是 main/mrmaster.go传递给 MakeMaster() 的参数
- worker的实现应该将第 X 次 reduce任务的输出结果放到 mr-out-X 文件里
- 一个 mr-out-X 文件针对每一个Reduce方法的输出对应一行数据. 每行数据通过使用 Go 的"%v %v"格式生成, 使用key 和 value 命名. 参考 main/mrsequential.go 的行信息"this is the correct format". 如果你的实现与这个格式差距较大，test脚本将会失败
- 可以修改 mr/worker.go, mr/master.go,和 mr/rpc.go. 也可以临时修改其他文件用来测试, 但确保最终使用原始版本代码可以执行; 作业将检查其他文件是否与原始版本匹配.
- worker 应该将中间结果的Map输出到当前目录的文件, 以便worker可以稍后读取它们作为Reduce 任务的输入
- main/mrmaster.go 希望mr/master.go 去实现一个Done() 函数, 此函数在MapReduce 任务成功完成时返回 true; 此时, mrmaster.go将会退出
- 当任务成功完成时, worker进程应该退出. 实现这一目标的一个简单方法是使用call() 的返回值: 如果worker 与master 通信失败, worker可以假设master 已经因为任务完成而退出, 因此worker也可以终止了. 这取决于你的设计, 你可能会发现master给workers一个"please exit"的任务同样有帮助.

Hints翻译

- 着手进行此实验可以从修改 mr/worker.go 中的 Worker() 方法向master发送请求任务的RPC开始. 然后修改master 来响应请求，发送一个 暂未开始 的map任务的文件名. 然后修改worker 读取文件并调用Map方法, 就像mrsequential.go 中做的一样.
- Map 和 Reduce 方法的应用在使用Go 插件包运行时从.so结尾的文件被加载.
- 如果修改了任何 mr/ 目录下的文件, 可能会需要使用像 go build -buildmode=plugin ../mrapps/wc.go 命令来re-build所有用到的MapReduce插件.
- 本实验的一个前提是workers 共享一个文件系统. 在所有workers运行在单机时这是显而易见的, 但如果workers运行在多台不同机器上，就需要一个像GFS一样的全局文件系统.
- 给中间结果文件命名的通常使用 mr-X-Y 格式, X代表Map任务编号, Y代表 reduce 任务编号.
- worker的map 任务需要一种将k/v对快速存入文件的方法, 同时要能在reduce 任务起见正确读取存储的k/v对. 一个可行方案是使用Go的 encoding/json 包，将k/v 对存入JSON文件中:
```
enc:= json.NewEncoder(file)
for _,kv := ... {
    err := enc.Encode(&kv)
}
```
对于文件的回读:
```
dec := json.NewDecoder(file)
for {
    var kv KeyValue
    if err := dec.Decode(&kv); err != nil {
        break
    }
    kva = append(kva, kv)
}
```

- worker 的map 部分可以使用 ihash(key) 方法(位于 worker.go 中)来为 得到的key 选择 reduce 任务.
- 可以从 mrsequential.go 中借鉴一些代码来读取 Map的输入文件、 为Map 和Reduce 之间的中间kv结果排序, 以及将Reduce 的输出存储到文件中.
- Master 作为RPC server, 将会并发. 不要忘记对共享数据使用锁.
- 使用Go 的race 探测器, go build -race 和 go run -race.test-mr.sh 提供了如何开启race 探测器用来测试的描述
- Workers 有时需要等待, 比如直到最后一个map任务完成后reduces任务才能启动. 一种可能的实现方式是worker定期向master请求任务，在每次请求之间使用 time.Sleep() 休眠等待；另一种方式是对 master 中相关RPC hanlder 使用time.Sleep()或 sync.Cond实现循环等待. Go 针对每个RPC在各自线程上运行handler , 因此一个hanlder 处于等待并不会影响master 处理其他RPC请求.
- master无法准确分辨如下worker: 崩溃的workers, 虽然未崩溃但因为某些原因中止的workers, 以及虽然正在执行但速度太慢不可用的worker. 最佳实践是让master 等待较长时间, 然后放弃并将任务分配给其他worker. 本实验中, 此情况下让master 等待10秒钟; 之后master应该假设这个worker已经挂了(虽然可能事实并非如此).
- 可以使用 mrapps/crash.go 应用插件测试崩溃恢复. 它会在Map 和 Reduce 方法中随机退出.
- 为了保证在出现crash时写入的部分文件不被读取到, MapReduce论文提到一个技巧, 使用临时文件并自动在完成写入后进行重命名. 本实验中你可以使用 ioutil.TempFile 来创建临时文件并使用 os.Rename 实现自动重命名.
- test-mr.sh 会执行子目录 mr-tmp 下所有程序, 因此如果执行出了问题想查看中间结果或输出文件时, 可以在那里找到.