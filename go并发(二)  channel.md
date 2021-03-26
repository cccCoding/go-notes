# go 并发(二)  channel

## CSP 并发模型

CSP（Communicating Sequential Processes），是用于描述两个独立的并发实体通过共享 channel（管道）进行通信的并发模型。

## go 并发模型

Go 语言中有两种并发编程模型，除了普遍认知的多线程共享内存模型，还把 CSP 的思想融入到语言的核心里，基于 goroutine 和 channel 实现了其特有的 CSP 并发模型，使并发编程成为 Go 的一个独特优势。

goroutine 和 channel 是一对组合。goroutine 是执行并发的实体，而每个实体之间通过 channel 通信来实现数据共享。

## go 并发原则

> Do not communicate by sharing memory; instead, share memory by communicating.

**不要通过共享内存来通信，而要通过通信来实现内存共享。**

即不推荐使用 sync 包里的 mutex 等组件，而是使用 channel 进行并发编程。可以使用原子函数、互斥锁等 ，但使用 channel 更优雅。

但两者其实都是必要且有效的。实际上 channel 的底层就是通过 mutex 来控制并发的，只是 channel 是更高一层次的并发编程原语，封装了更多的功能。

是选择 sync 包里的底层并发编程原语还是 channel，参考如下决策树：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx61F8HFwuahzAeYQHZE9s3iaNVkZUOjGWXgbv5ZJuwULd5qWfqUvKfk6qFGmg9ialrw5m1bmlJoibBdEQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## channel

### 什么是 channel

channel 是 goroutine 之间通信的管道。channel 是线程安全的，并提供“先进先出”的特性。

### 使用

**声明和初始化**

```go
var c chan int        // 声明了一个 nil 通道
c = make(chan int)    // 初始化了一个无缓冲通道，其值是一个地址，类型是 chan int
c = make(chan int, 100)	// 初始化了一个缓冲区大小为100的通道
```

**发送和接收**

```go
go func() {c <- 100}()	// 发送数据到通道中
i := <-c				// 从通道中接收数据
```

### 实现原理

#### 数据结构

```go
type hchan struct {
    // chan 里元素数量
    qcount   uint
    // chan 底层循环数组的长度
    dataqsiz uint
    // 指向底层循环数组的指针，只针对有缓冲的 channel
    buf      unsafe.Pointer
    // chan 中元素大小
    elemsize uint16
    // chan 是否被关闭的标志
    closed   uint32
    // chan 中元素类型
    elemtype *_type // element type
    // 已发送元素在循环数组中的索引
    sendx    uint   // send index
    // 已接收元素在循环数组中的索引
    recvx    uint   // receive index
    // 等待接收的 goroutine 队列
    recvq    waitq  // list of recv waiters
    // 等待发送的 goroutine 队列
    sendq    waitq  // list of send waiters

    // 保护 hchan 中所有字段
    lock mutex
}
```

`buf` 指向底层循环数组，只有缓冲型的 channel 才有。

`sendx`， `recvx` 均指向底层循环数组，表示当前可以发送和接收的元素位置索引值（相对于底层数组）。

`sendq`， `recvq` 分别表示被阻塞的 goroutine，这些 goroutine 由于尝试读取 channel 或向 channel 发送数据而被阻塞。

`waitq` 是 `sudog` 的一个双向链表，而 `sudog` 实际上是对 goroutine 的一个封装：

```go
type waitq struct {    
	first *sudog    
	last  *sudog
}
```

`lock` 用来保证每个读 channel 或写 channel 的操作都是原子的。

例如，创建一个容量为 6 的，元素为 int 型的 channel 数据结构如下 ：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx61F8HFwuahzAeYQHZE9s3iaN68pkCK3yL1Vk3icicO3OlTSicXnsY3dzcMBCGQyeLY4HsUNBT9hGgzxvg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### 创建

当使用 make 函数创建通道时，底层创建函数如下：

```go
func makechan(t *chantype, size int64) *hchan
```

从函数原型来看，创建的 chan 是一个指针，和 map 相似。

#### 接收和发送

对 channel 的发送和接收操作都会在编译期间转换成为底层的发送接收函数。

channel 的发送和接收操作本质上都是 “值的拷贝”。

### 缓冲通道

对于无缓冲通道，读写通道会立马阻塞当前协程。一个协程被通道操作阻塞后，Go 调度器会去调用其他可用的协程，这样程序就不会一直阻塞。

而对于缓冲通道，写不会阻塞当前通道，直到通道满了，同理，读操作也不会阻塞当前通道，除非通道没数据。

也可以说无缓冲的 channel 是同步的，而有缓冲的 channel 是异步的。

创建一个缓冲通道：

```go
ch := make(chan type, capacity)
```

capacity 是缓冲大小，必须大于 0。内置函数 len()、cap() 可以计算通道的长度和容量。

如果缓冲通道是关闭状态但有数据，仍然可以读取数据。

### 单向通道

主要用在通道作为参数传递的时候，Go 提供了自动转化，双向转单向。

使用单向通道主要是可以提高程序的类型安全性，程序不容易出错。

### 关闭通道

使用内置函数`close(ch)`可以关闭通道。

#### 判断通道关闭状态

* 数据接收方可以通过返回状态判断通道是否已关闭：

  ```go
  val, ok := <- ch
  ```

  ok 用于判断 ch 是否已关闭且没有缓冲值可以读取。为 true，该通道还可以进行读写操作；为 false，通道已关闭且没有缓冲数据，不能再进行数据传输，返回的对应类型的零值。

* for range 读取通道，通道关闭，for range 自动退出

  ```go
  for v := range ch {
  	fmt.Println(v)
  }
  ```

  应注意，使用 for range 读取一个通道，数据写入完毕后必须关闭通道，否则协程阻塞。

#### 如何优雅地关闭 channel

关于 channel 的使用，有几点不方便的地方：

1. 在不改变 channel 自身状态的情况下，无法获知一个 channel 是否关闭。
2. 关闭一个 closed channel 会导致 panic。所以，如果关闭 channel 的一方在不知道 channel 是否处于关闭状态时就去贸然关闭 channel 是很危险的事情。
3. 向一个 closed channel 发送数据会导致 panic。所以，如果向 channel 发送数据的一方不知道 channel 是否处于关闭状态时就去贸然向 channel 发送数据是很危险的事情。

应该如何优雅地关闭 channel？

根据 sender 和 receiver 的个数，分下面几种情况：

1. 一个 sender，一个 receiver
2. 一个 sender， M 个 receiver
3. N 个 sender，一个 reciver
4. N 个 sender， M 个 receiver

对于 1，2，只有一个 sender 的情况就不用说了，直接从 sender 端关闭就好了。

第 3 种情形下，优雅关闭 channel 的方法是增加一个传递关闭信号的 channel，receiver 通过信号 channel 下达关闭数据 channel 的命令。senders 监听到关闭信号后，停止发送数据。

```go
func main() {
	rand.Seed(time.Now().UnixNano())

	const Max = 100000
	const NumSenders = 1000

	dataCh := make(chan int, 100)
	stopCh := make(chan struct{})

	var wg sync.WaitGroup
	wg.Add(NumSenders)

	for i := 0; i < NumSenders; i++ {
		go func() {
			for {
				select {
				case <-stopCh:
					wg.Done()
					return
				case dataCh <- rand.Intn(Max):
				}
			}
		}()
	}

	go func() {
		for value := range dataCh {
			if value == Max-1 {
				fmt.Println("send stop signal to senders.")
				close(stopCh)
				return
			}
			fmt.Println(value)
		}
	}()

	wg.Wait()
	fmt.Println("main exit")
}
```

stopCh 就是信号 channel，receiver 关闭 stopCh 来通知 senders 停止发送数据。上面的代码没有明确关闭 dataCh，在 Go 语言中，对于一个 channel，如果最终没有任何 goroutine 引用它，不管 channel 有没有被关闭，最终都会被 gc 回收。所以，在这种情形下，所谓的优雅地关闭 channel 就是不关闭 channel，让 gc 代劳。

第四种情况和第三种情况不同，这里有 M 个 receiver，如果还是采用第三种方案，由 receiver 直接关闭 stopCh 的话，就会重复关闭一个 channel，导致 panic。因此需要增加一个中间人，M 个 receiver 都向他发送关闭 dataCh 的请求，中间人收到第一个请求后下达关闭 dataCh 的指令（关闭 stopCh）。当然，这里的 N 个 sender 也可以向中间人发送关闭 dataCh 的请求。

```go
func main() {
	rand.Seed(time.Now().UnixNano())

	const Max = 100000
	const NumSenders = 1000
	const NumReceivers = 10

	dataCh := make(chan int, 100)
	stopCh := make(chan struct{})
	toStop := make(chan string, NumSenders + NumReceivers)
	exitCh := make(chan struct{})

	go func() {
		s := <-toStop
		fmt.Println("toStop s:", s)
		close(stopCh)
		exitCh <- struct{}{}
	}()

	for i := 0; i < NumSenders; i++ {
		go func(id string) {
			for {
				value := rand.Intn(Max)
				if value == 0 {
					toStop <- "sender#" + id
					return
				}
				select {
				case <-stopCh:
					return
				case dataCh <- value:
				}
			}
		}(strconv.Itoa(i))
	}

	for i := 0; i < NumReceivers; i++ {
		go func(id string) {
			for {
				select {
				case <- stopCh:
					return
				case value := <-dataCh:
					if value == Max-1 {
						toStop <- "receiver#" + id
						return
					}
					fmt.Println(value)
				}
			}
		}(strconv.Itoa(i))
	}

	select {
	case <-exitCh:
		time.Sleep(time.Second)
	}
	fmt.Println("main exit")
}
```

代码里 toStop 就是中间人的角色，使用它来接收 senders 和 receivers 发送过来的关闭 dataCh 请求。这里同样没有真正关闭 dataCh，让 gc 代劳。

### 常见异常操作

**操作 nil 或 closed channel**

* close 关闭一个 nil channel，会引起 panic。
* close 关闭一个 closed channel，会引起 panic。
* 往一个 nil channel 发送数据，会造成阻塞。
* 从一个 nil channel 接收数据，会造成阻塞。

* 往一个 closed channel 发送数据，会引起 panic。
* 从一个 closed channel 接收数据，返回已缓冲数据或者零值。

**阻塞、死锁**

* 无缓冲通道写或者读数据，当前协程阻塞。
* 有缓冲通道已满，再往通道中写数据，当前协程阻塞。
* 使用 for range 读取一个通道，数据写入端写入完毕后必须关闭通道，否则 for range 语句所在协程阻塞。
* 空 select 语句`select {}`，没有 case 分支，当前协程阻塞。
* 当前协程阻塞又没有其他可用协程时，死锁。

**内存泄漏**

* channel 可能会引发 goroutine 泄漏，原因是 goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 goroutine 一直处于等待队列中。

### 通道应用

**停止信号**

channel 多用于停止信号的场景，关闭 channel 或者向 channel 发送一个元素，使得接收 channel 的那一方获知此信息，进而做后续操作。

**任务定时**

与 timer 结合，实现超时控制。

```go
// 等待 1 分钟后，如果 dataCh 还没有读出数据或者被关闭，就直接结束。
select {
    case <-time.After(time.Minute):
	case <-dataCh:
    	fmt.Println("do something")
}
```

或定期执行任务。

```go
// 每隔一秒执行任务
ticker := time.Tick(time.Second)
for {
    select {
        case <- ticker:
        fmt.Println("do something")
    }
}
```

**解耦生产方和消费方**

生产方往 taskCh 塞任务，消费方循环从 channel 中拿任务，解耦。

**控制并发数**

某些场景因为资源限制等原因，需要控制并发数量。

```go
limit := make(chan int, 3)
for _, w := range work {
    go func() {
        limit <- 1
        w()
        <-limit
    }()
}
```

真正执行任务，访问第三方的动作在 w() 中完成，在执行 w() 之前，先要从 limit 中拿“许可证”，拿到许可证之后，才能执行 w()，并且在执行完任务，要将“许可证”归还。这样就可以控制同时运行的 goroutine 数。

还有一点要注意的是，如果 w() 发生 panic，那“许可证”可能就还不回去了，因此需要使用 defer 来保证。