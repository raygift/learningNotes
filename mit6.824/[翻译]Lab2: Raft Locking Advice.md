Raft Locking Advice                                                         

If you are wondering how to use locks in the 6.824 Raft labs, here are
some rules and ways of thinking that might be helpful.                      

Rule 1: Whenever you have data that more than one goroutine uses, and

at least one goroutine might modify the data, the goroutines should
use locks to prevent simultaneous use of the data. The Go race
detector is pretty good at detecting violations of this rule (though)
it won't help with any of the rules below).                                 

Rule 2: Whenever code makes a sequence of modifications to shared
data, and other goroutines might malfunction if they looked at the
data midway through the sequence, you should use a lock around the
whole sequence.

An example:

  rf.mu.Lock()
  rf.currentTerm += 1
  rf.state = Candidate
  rf.mu.Unlock()

It would be a mistake for another goroutine to see either of these
updates alone (i.e. the old state with the new term, or the new term
with the old state). So we need to hold the lock continuously over the
whole sequence of updates. All other code that uses rf.currentTerm or
rf.state must also hold the lock, in order to ensure exclusive access
for all uses.

The code between Lock() and Unlock() is often called a "critical
section." The locking rules a programmer chooses (e.g. "a goroutine
must hold rf.mu when using rf.currentTerm or rf.state") are often
called a "locking protocol".

Rule 3: Whenever code does a sequence of reads of shared data (or
reads and writes), and would malfunction if another goroutine modified
the data midway through the sequence, you should use a lock around the
whole sequence.

An example that could occur in a Raft RPC handler:

  rf.mu.Lock()
  if args.Term > rf.currentTerm {
   rf.currentTerm = args.Term
  }
  rf.mu.Unlock()

This code needs to hold the lock continuously for the whole sequence.
Raft requires that currentTerm only increases, and never decreases.
Another RPC handler could be executing in a separate goroutine; if it
were allowed to modify rf.currentTerm between the if statement and the
update to rf.currentTerm, this code might end up decreasing
rf.currentTerm. Hence the lock must be held continuously over the
whole sequence. In addition, every other use of currentTerm must hold
the lock, to ensure that no other goroutine modifies currentTerm
during our critical section.

Real Raft code would need to use longer critical sections than these
examples; for example, a Raft RPC handler should probably hold the
lock for the entire handler.

Rule 4: It's usually a bad idea to hold a lock while doing anything
that might wait: reading a Go channel, sending on a channel, waiting
for a timer, calling time.Sleep(), or sending an RPC (and waiting for the
reply). One reason is that you probably want other goroutines to make
progress during the wait. Another reason is deadlock avoidance. Imagine
two peers sending each other RPCs while holding locks; both RPC
handlers need the receiving peer's lock; neither RPC handler can ever
complete because it needs the lock held by the waiting RPC call.

Code that waits should first release locks. If that's not convenient,
sometimes it's useful to create a separate goroutine to do the wait.

Rule 5: Be careful about assumptions across a drop and re-acquire of a
lock. One place this can arise is when avoiding waiting with locks
held. For example, this code to send vote RPCs is incorrect:

  rf.mu.Lock()
  rf.currentTerm += 1
  rf.state = Candidate
  for <each peer> {
    go func() {
      rf.mu.Lock()
      args.Term = rf.currentTerm
      rf.mu.Unlock()
      Call("Raft.RequestVote", &args, ...)
      // handle the reply...
    } ()
  }
  rf.mu.Unlock()

The code sends each RPC in a separate goroutine. It's incorrect
because args.Term may not be the same as the rf.currentTerm at which
the surrounding code decided to become a Candidate. Lots of time may
pass between when the surrounding code creates the goroutine and when
the goroutine reads rf.currentTerm; for example, multiple terms may
come and go, and the peer may no longer be a candidate. One way to fix
this is for the created goroutine to use a copy of rf.currentTerm made
while the outer code holds the lock. Similarly, reply-handling code
after the Call() must re-check all relevant assumptions after
re-acquiring the lock; for example, it should check that
rf.currentTerm hasn't changed since the decision to become a
candidate.

It can be difficult to interpret and apply these rules. Perhaps most
puzzling is the notion in Rules 2 and 3 of code sequences that
shouldn't be interleaved with other goroutines' reads or writes. How
can one recognize such sequences? How should one decide where a
sequence ought to start and end?

One approach is to start with code that has no locks, and think
carefully about where one needs to add locks to attain correctness.
This approach can be difficult since it requires reasoning about the
correctness of concurrent code.

A more pragmatic approach starts with the observation that if there
were no concurrency (no simultaneously executing goroutines), you
would not need locks at all. But you have concurrency forced on you
when the RPC system creates goroutines to execute RPC handlers, and
because you need to send RPCs in separate goroutines to avoid waiting.
You can effectively eliminate this concurrency by identifying all
places where goroutines start (RPC handlers, background goroutines you
create in Make(), &c), acquiring the lock at the very start of each
goroutine, and only releasing the lock when that goroutine has
completely finished and returns. This locking protocol ensures that
nothing significant ever executes in parallel; the locks ensure that
each goroutine executes to completion before any other goroutine is
allowed to start. With no parallel execution, it's hard to violate
Rules 1, 2, 3, or 5. If each goroutine's code is correct in isolation
(when executed alone, with no concurrent goroutines), it's likely to
still be correct when you use locks to suppress concurrency. So you
can avoid explicit reasoning about correctness, or explicitly
identifying critical sections.

However, Rule 4 is likely to be a problem. So the next step is to find
places where the code waits, and to add lock releases and re-acquires
(and/or goroutine creation) as needed, being careful to re-establish
assumptions after each re-acquire. You may find this process easier to
get right than directly identifying sequences that must be locked for
correctness.

(As an aside, what this approach sacrifices is any opportunity for
better performance via parallel execution on multiple cores: your code
is likely to hold locks when it doesn't need to, and may thus
unnecessarily prohibit parallel execution of goroutines. On the other
hand, there is not much opportunity for CPU parallelism within a
single Raft peer.)

Raft 锁 建议

如果你对 Raft 实验中如何使用锁有疑问，本文给你一些可能有帮助的原则和方法。

原则1: 无论何时，若有超过1个goroutine 使用的数据，并且至少有一个goroutine 可能会修改此数据，那么在goroutine 中应该使用锁来避免同时使用此数据，Go race 检测器在检测本原则的冲突上有很好的效果（尽管 race 对下面其他原则无效）

 原则2: 无论何时，代码对共享数据有顺序的修改，并且如果共享数据的中间状态被其他goroutine 读取到可能引起错误时，应该给修改顺序整个过程加上锁

例如：

```golang
  rf.mu.Lock()
  rf.currentTerm += 1
  rf.state = Candidate
  rf.mu.Unlock()
```

如果单独修改rf的某一个变量会导致其他goroutine 发生错误（如: 旧的state 和新的term，或者新的term 对应旧的state）。因此我们需要在整个更新过程中持有锁。其他使用到rf.currentTerm 或rf.state 的代码也必须使用锁，来保证对变量的修改可以被同时访问到。

在Lock() 和 Unlock() 之间的代码常被称为"critical section"。程序选择的锁方式（如 "当goroutine 使用rf.currentTerm 或rf.state 时必须使用rf.mu"）常被称为“locking protocol”。

原则3：无论何时代码对共享变量进行一系列读取（或者读写），如果其他goroutine 正在修改变量则导致执行异常，必须在整个读写过程中持有锁。

在Raft RPC 处理代码中有如下案例：

```golang
  rf.mu.Lock()
  if args.Term > rf.currentTerm {
   rf.currentTerm = args.Term
  }
  rf.mu.Unlock()
```

代码需要在整个过程中持续持有锁。Raft 要求currentTerm 递增，绝对不能运行中变小。在另外一个goroutine 中可能执行着RPC 处理；如果在if 语句和 更新rf.currentTerm 语句之间另一个goroutine 执行修改rf.currentTerm 的操作，最终代码执行完rf.currentTerm 可能会变小。因此必须在整个更新操作过程中持有锁。另外，其他使用到currentTerm 的goroutine 必须持有锁，来确保在 critical section 执行期间中没有其他goroutine 修改currentTerm。

实际的Raft 代码可能需要使用比实例代码更长的 critical section，Raft 的RPC 处理函数可能在整个处理期间都持有锁。

原则4：在执行需要等待的操作时持有锁是个不好的做法：读取Go channel，向channel 发送数据，等待计时器，调用time.Sleep()，或者发送RPC（并等待响应）。原因之一是你可能想在等待期间让其他goroutine 继续执行。另一个原因是避免死锁。想象一下持有锁的两个节点互相发送RPC ；两个节点的RPC 处理函数都需要获得对方持有的锁；任何一个节点都无法完成执行RPC 的处理，因为它需要拿到等待RPC 调用时所持有的锁。

耗时久的代码需要首先释放锁。如果不方便做，有时创建独立的goroutine 进行等待是个有用的方法。

原则5：小心对锁释放和重新获取的假设。这可能导致持有锁的等待失效。比如下面发送投票RPC 的代码是错误的：

```golang
  rf.mu.Lock()
  rf.currentTerm += 1
  rf.state = Candidate
  for <each peer> {
    go func() {
      rf.mu.Lock()
      args.Term = rf.currentTerm
      rf.mu.Unlock()
      Call("Raft.RequestVote", &args, ...)
      // handle the reply...
    } ()
  }
  rf.mu.Unlock()
```

这段代码使用单独的goroutine 发送RPC。错误之处在于args.Term 可能与确定变为candidate 的代码处的rf.currentTerm 不同。在外部代码创建goroutine 和goroutine 读取到rf.currentTerm 之间可能花费了大量时间；例如，可能过去了很多个term 而goroutine 读取到rf.currentTerm 时，节点已经不再是candidate 。解决办法之一是当外部代码持有锁时创建rf.currentTerm 的拷贝给其创建的goroutine 使用 。类似的，Call() 之后的响应处理代码在重新获取锁之后，必须再次检查所有相关假设；例如，如果决定切换状态为candidate，必须检查rf.currentTerm 未发生变化（pc：若term 发生变化，此时状态可能已经不再是follower）。

理解和实现这些原则有一定难度。或许最让人困惑的是原则2 和原则3 中的code sequence 过程中不能和其他goroutine 的读写交替进行。如何感知这些过程？如何决定这些过程的开始和结束？

方法之一是启动时不获取锁，仔细考虑在什么地方需要锁来保证正确性。这个方法比较难，因为需要对并发代码的正确性有很好的理解。

更实用的方法是观察如果没有并发（没有同时执行goroutine），则完全不需要锁。但因为需要在独立的goroutine 中发送RPC 来避免等待，在RPC 系统创建goroutine 来执行RPC 处理方法时是一定有并发的。可以通过找出所有goroutine 的开始位置（RPC 处理函数，在Make() 创建的后台go协程，&c），在每个goroutine 的最开始获取锁，在goroutine 完成并返回时释放锁。这种锁策略确保了不会并行执行任何操作；锁确保了在一个goroutine 被允许开始执行前，前一个goroutine 已经执行完成。没有并行执行，也就不可能打破第1、2、3、5条原则。如果每个goroutine 的代码都正确的独立运行（单独执行时没有其他并行goroutine），在更高层面的并发中使用锁时就更可能保持正确。因此就可以避免对正确性推理和对临界区的判断。

然而原则4可能是个问题。因此下一步需要找到程序等待在何处，按需添加锁释放和重新获取锁的代码，在每次重新获取锁之后注意重新确认假设条件。你将会发现相比对必须加锁的过程的直接判断，这种方法更加简单的保证了正确性。

（顺便提一句，上述方法牺牲了通过在多核上并行从而获得更高性能的任何可能：代码在不必要时使用了锁，这种非必要锁的使用使得goroutine 的并行执行失效。但话说回来，单个Raft 节点中没多少CPU 并行的机会）







