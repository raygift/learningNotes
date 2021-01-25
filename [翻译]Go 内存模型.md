# Go 内存模型



## 简介

Go 的内存模型特指 在一个goroutine 中读取变量时，可以读取到另外一个goroutine 中对相同变量的写入 的条件



## 建议

被多个goroutines 同时访问的数据，程序对此数据的访问必须串行进行。

为了串行访问数据，使用channel 或其他同步原语来保护数据，例如 sync 和 sync/atomic 包中的同步原语。

如果你必须阅读本文剩余内容来搞清楚你的程序的运行，你太聪明了。

不要聪明。（？？？）



## Happens Before

使用单个goroutine，读取和写入必须表现的像按照程序指定的顺序执行一样。也就是说，编译器和处理器只有在重新排序不会改变语言规范所定义的goroutine行为时，才会对读写操作重新排序。由于进行了重新排序，不同goroutine 所观察到的执行顺序可能不一样。比如，如果一个goroutine 执行 a=1;b=2;，另外一个goroutine 可能先观察到b 值的变化，在观察到a 值的变化。

为了说明读写的需求，我们定义了Go 程序中内存操作执行的特定的顺序：happens before。如果事件e1 happens before e2 ，则称 e2 happens after e1；如果e1 不是 happens before e2，且也不是happens after e2，则称e1 和e2 happen concurrently。

单个goroutine，happens-before 顺序是程序中所表达的顺序。

当满足如下两个条件时，对于变量v 的读操作r 可以观察到对于v 的写操作w：

1. r 不是 happen before w 的
2. 在w 之后r 之前，没有其他对于v 的写操作w'

为了保证对于变量v 的读取操作r 可以发现对于v 的写入操作w，需确保w 是唯一允许写入的。也就是说，在满足如下两个条件时，可保证r 可以观察到w 的写入：

1. w happens befor r
2. 除了w之外，其他对于共享变量v 的写入操作，要么happens before w，要么happens after r

这一组条件比上一组更严格；它要求没有与w 或r 并发的其他写入操作。

针对单个goroutine，不会有并发，上述两组条件是等价的：读取操作r 可以观察到最近一次向v 执行写入操作w 所写入的值。当多个goroutine 访问共享变量v 时，必须使用同步事件来建立 happens-before 条件，以确保读操作可以读到期望的写入值。

变量v 使用对应类型的零值初始化，在内存模型中初始化也是一次写入操作。

针对 比单个机器单词大的值的读取和写入，按照多个没有指定顺序的机器单词大小的操作处理

## 同步

### 初始化

程序初始化运行在单个goroutine，但这个初始化的goroutine可能会创建其他多个goroutines，这些被创建的goroutine 是并行的。

如果包p 导入了包q，q初始化函数的执行完成 happens before p的所有初始化init函数

main.main 启动函数 happens after 所有初始化函数执行完成

### Goroutine 创建

go 表达式创建一个新的goroutine happends before goroutine 执行的开始。

比如下面这个程序：

```golang
var a string

func f() {
	print(a)
}

func hello() {
	a = "hello, world"
	go f()
}
```

调用 hello 函数将会在未来某时间点打印出“hello, world”（可能是在 hello 函数 已经返回之后）。

### Goroutine 销毁

goroutine 的退出不保证 happen before 程序的任意事件。比如如下程序：

``` golang
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```

赋值操作之前没有任何同步事件，因此无法保证它可以被其他goroutine观测到。实际上，激进的编译器可能会将整个go表达式删除。

如果要其他任意goroutine需要观测到一个goroutine 所产生的影响，使用诸如锁或者channel通信之类的同步机制来建立相对的顺序。

### Channel 通信

Channel 通信是goroutine 之间同步的主要方法。通过制定channel 发送的每条数据都对应着一个从channel的接收，且发送和接收常常在不同的goroutine 里。

通过channel 发送 happens before 通过channel 接收响应数据。

下面的程序保证能打印出“hello,world”。向a 写入数据的操作 happens before 向c发送数据，向c 发送数据 happens before 从c接收数据，从c 接收数据 happens before 打印结果。

```golang
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```

channel 的关闭 happens before 由于channel 关闭而导致接收者返回一个零值。

在上面的例子中，用 close(c) 代替 c<-0 来暂停程序，同样可以保证相同的效果。

从一个 无缓冲 channel 接收 happens before 完成向channel 中发送。

如下程序同样可以保证打印出“hello, world”（与上一个程序类似，但使用的是无缓冲的channel ）。向a 写入的操作 happens before 从c 读取操作 happens befores 向c 发送的操作 happens before 打印操作

``` golang
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}

func main() {
	go f()
	c <- 0
	print(a)
}
```

如果channel 存在缓冲区（比如 c=make(chan int, 1)）那么此程序就无法保证打印出 hello world。（可能会打印出空字符、crash或者进行其他处理）。

_在容量为C的channel 上的第k 次接收 happens before 完成向channel的第k+C次发送。_

本条规则概括了之前有缓冲channel 的规则。允许一个buffered channel 为计数信号建模：channel 中的数据个数与活跃使用的个数对应，channel的容量与同时使用的最大个数对应，发送一个数据需要得到信号，而接收数据释放信号。这是一个常见的并发限制方法。

下面的程序为work list 中的每条记录开启一个goroutine，goroutine 共同使用 limit channel，确保了同时最多有三个正在工作的函数。

```golang
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```

### 锁

sync 包实现了两种锁类型，sync.Mutex 和 sync.RWMutex。

对于sync.Mutex 和 sync.RWMutex 的任意变量 l，以及n < m, 称 调用 n 的 l.Unlock() happens before 完成调用 m 的 l.Lock() 的返回。

如下程序保证可打印出 helloworld。 （在f 中）第一次调用l.Unlock() happens before （在main 中）第二次完成调用 l.Lock() 的返回，第二次得到l.Lock()的返回又 happens before 打印print。

```golang
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```

_对于任何调用 sync.RWMutex 变量 l 的 l.RLock ，存在n，调用n 的 l.Unlock 后 l.RLock 发生（且返回），对l.RUnlock 的匹配 happens before 调用 n+1的l.Lock_

### Once

sync 包通过使用Once 类型为初始化当前多个goroutine 提供了安全的机制。多线程可针对特定的f 执行 once.Do(f)，但只有一个可成功运行 f()，其他线程会在f() 返回前一直被阻塞。

_once.Do(f) 的单次f() 调用（的返回） happens before 任意once.Do(f) 的返回_

下面程序中 对于twoprint() 的调用可以保证调用且只调用 setup 函数 一次。 setup 函数会在所有print 调用之前执行完成。

### 错误的同步

注意读操作r 可能会读取到与r 同时发生的写操作w 写入的值。即使发生这种情况，也并不意味着r 之后的读取操作可以读取到w 之前的写操作。

在如下程序中，g可能打印出2，然后打印0.

```golang
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```

这一事实导致了一些普遍做法失效。

双重检查锁是避免上述同步问题的一种方法。比如，twoprint 程序的如下写法可能导致错误：

```golang
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

上面代码无法保证在doprint 函数中能观察到done 被赋值时，同样也能观察到a 被赋值。此版本会错误地打印空字符串，而不是正常打印helloworld。

另一种常见的错误是对一个值的忙等待，busy waiting，如下代码中：

```golang
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```

跟之前所述类似，在main函数中，观测到done被赋值时无法保证对a 的赋值同样也能被观测到，因此此程序也可能打印出空字符串。更糟糕的是，无法保证对于done 的赋值可以在main 中被观测到，因为在两个线程之间没有同步事件。main 中的循环可能是无限循环，无法结束。

此问题还有一些微妙的变体，如下程序所示：

```golang
type T struct {
	msg string
}

var g *T

func setup() {
	t := new(T)
	t.msg = "hello, world"
	g = t
}

func main() {
	go setup()
	for g == nil {
	}
	print(g.msg)
}
```

此程序中，即使main 中的循环观察到了 g!=nil 后退出循环，无法保证可以观察到被初始化的g.msg 。

对于上述有问题的程序，解决办法都是一样的：使用显式的同步