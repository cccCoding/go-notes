# go 并发(五)  WaitGroup&Context

Go 控制并发有两种经典的方式，一种是 WaitGroup，另外一种就是 Context。

## WaitGroup

```
func main() {
    var wg sync.WaitGroup

    wg.Add(2)
    go func() {
        time.Sleep(2*time.Second)
        fmt.Println("1号完成")
        wg.Done()
    }()
   go func() {
        time.Sleep(2*time.Second)
        fmt.Println("2号完成")
        wg.Done()
    }()
    wg.Wait()
    fmt.Println("好了，大家都干完了，放工")
}
```

这是一种控制并发的方式，这种尤其适用于，好多个 goroutine 协同做一件事情的时候，因为每个 goroutine 做的都是这件事情的一部分，只有全部的 goroutine 都完成，这件事情才算是完成，这是等待的方式。

## Context

此外还有另外一种场景，即需要我们主动的通知某一个 goroutine 结束。比如我们开启一个后台 goroutine 一直做事情，比如监控，现在不需要了，就需要通知这个监控 goroutine 结束。

### 使用 channel 控制单个 goroutine

```go
func main() {
	stop := make(chan bool)
	go func() {
		for {
			select {
			case <-stop:
				fmt.Println("监控退出，停止了...")
				return
			default:
				fmt.Println("goroutine监控中...")
				time.Sleep(time.Second)
			}
		}
	}()

	time.Sleep(5 * time.Second)
	fmt.Println("可以了，通知监控停止")
	stop<- true
	time.Sleep(1 * time.Second)
	fmt.Println("main exit")
}
```

可以看到，使用 channel + select 可以优雅地结束一个 goroutine。不过这种方式有一定局限性，在复杂一点的场景中，如果有很多 goroutine 都需要控制结束，又如果这些 goroutine 又衍生了更多 goroutine，这时使用 context 可以实现复杂情况下不同层级 goroutine 的控制。

### 使用 context 控制单个 goroutine

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go func(ctx context.Context) {
		for {
			select {
			case <-ctx.Done():
				fmt.Println("监控退出，停止了...")
				return
			default:
				fmt.Println("goroutine监控中...")
				time.Sleep(time.Second)
			}
		}
	}(ctx)

	time.Sleep(5 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	time.Sleep(time.Second)
	fmt.Println("main exit")
}
```

`context.Background()` 返回一个空的 Context，这个空的 Context 一般用于整个 Context 树的根节点。然后我们使用`context.WithCancel(parent)`函数，创建一个可取消的子 Context，然后当作参数传给 goroutine 使用，这样就可以使用这个子 Context 跟踪这个 goroutine。

在 goroutine 中，在 select 中调用`<-ctx.Done()`判断是否要结束，如果接受到值的话，就可以返回结束 goroutine 了；如果接收不到，就会继续进行监控。使用`cancel`函数就可以发出取消指令。

### 使用 Context 控制多个 goroutine

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go watch(ctx,"【监控1】")
	go watch(ctx,"【监控2】")

	time.Sleep(5 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	time.Sleep(1 * time.Second)
}

func watch(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name,"监控退出，停止了...")
			return
		default:
			fmt.Println(name,"goroutine监控中...")
			time.Sleep(time.Second)
		}
	}
}
```

示例中启动了2个监控 goroutine 进行不断的监控，每一个都使用了 Context 进行跟踪，当我们使用`cancel`函数通知取消时，这2个 goroutine 都会被结束。这就是 Context 的控制能力，它就像一个控制器一样，按下开关后，所有基于这个 Context 或者衍生的子 Context 都会收到通知，结束 goroutine。

### Context 接口

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

Context 接口共有4个方法。

* `Deadline` 方法用于获取设置的截止时间，到了这个截止时间，context 会自动发起取消请求；第二个返回值 ok 表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。
* `Done` 方法返回一个只读的 chan，类型为 `struct{}`，我们在 goroutine 中，如果该方法返回的chan 可以读取，则意味着 parent context 已经发起了取消请求，我们通过`Done`方法收到这个信号后，就应该做清理操作，然后退出 goroutine，释放资源。
* `Err` 方法返回取消的错误原因，到底因为什么 context 被取消。
* `Value` 方法获取该 context 上绑定的值，以键值对的形式存储，通过一个键可以获取对应的值，这个值一般是线程安全的。

Context 接口不需要我们实现，Go 已经有2个内置实现，代码中都是以这两个内置实现作为最顶层的 parent context，衍生出更多的子 context。

```go
ctx := context.Background()
ctx2 := context.TODO()
```

一般使用`context.Background()`作为 context 树结构最顶层的根 context。它本质上是一个 emptyCtx 结构体类型，是一个不可取消，没有设置截止时间，没有携带任何值的 context。

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
    return
}
func (*emptyCtx) Done() <-chan struct{} {
    return nil
}
func (*emptyCtx) Err() error {
    return nil
}
func (*emptyCtx) Value(key interface{}) interface{} {
    return nil
}
```

这就是 emptyCtx 实现 Context 接口的方法，可以看到，这些方法什么都没做，返回的都是 nil 或者零值。

### Context 的继承衍生

有了根 Context，就可以使用 context 包为我们提供的 `With` 系列的函数衍生更多的子 Context 了。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
func WithValue(parent Context, key, val interface{}) Context
```

这四个函数都接收一个 partent Context 参数，基于这个父 Context 创建出子 Context。可以理解为子 Context 继承父 Context，也可以理解为基于父 Context 的衍生。

通过这些函数，就创建了一棵 Context 树，树的每个节点都可以有任意多个子节点，节点层级可以有任意多个。

`WithCancel`函数返回子 Context，以及一个取消函数。该函数可以取消一个 Context，以及这个节点Context 下所有的所有的子 Context，不管有多少层级。

`WithDeadline` 函数和 `WithCancel` 差不多，它会多传递一个截止时间参数，意味着到了这个时间点，会自动取消 Context，当然我们也可以不等到这个时候，提前通过取消函数进行取消。

`WithTimeout` 和 `WithDeadline`基本上一样，这个表示是超时自动取消，是多少时间后自动取消Context 的意思。

`WithValue` 函数和取消 Context 无关，它是为了生成一个绑定了一个键值对数据的 Context，这个绑定的数据可以通过 `Context.Value` 方法访问到。应注意，这里的 key 必须是可比较的类型，value 的值要是线程安全的。使用 WithValue 传值，一般是必须的值，不要什么值都传递。

### Context 使用原则

1. 不要把 Context 放在结构体中，要以参数的方式传递。
2. 以 Context 作为参数的函数方法，应该把 Context 作为第一个参数。
3. 给一个函数方法传递 Context 的时候，不要传递 nil，如果不知道传递什么，就使用context.TODO。
4. Context 的 Value 相关方法应该传递必须的数据，不要什么数据都使用这个传递。
5. Context 是线程安全的，可以放心的在多个 goroutine 中传递。

### 原理

todo

https://mp.weixin.qq.com/s/SuYSo_APH2viuWiVxezyVQ
https://mp.weixin.qq.com/s/gArkr4NUQDcfSnExMY76iA
https://mp.weixin.qq.com/s/gpcibIUz2BP3xlTR-P8Xbg