# go 并发之 select

Go 语言最大的特色就是从语言层面支持并发。通过轻量的 goroutines 和 channel 可以简便地处理并发问题。

### select

#### 使用

select 的用法类似 switch，但 select 不会有输入值而且只用于通道操作。select 用于从多个发送或接收通道操作中进行选择。如果没有 default case，语句会阻塞直到其中有通道可以操作，如果有多个通道可以操作，会随机选择其中一个 case 执行。如果有 default case，select 语句不会阻塞，如果其他通道操作还没有准备好，将会直接执行 default 分支。

```go
select {
case s1 := <-ch1:
	fmt.Println(s1)
case s2 := <-ch2:
    fmt.Println(s2)
default:
    fmt.Println("no case ok")
}
```

case 分支中如果通道是 nil，该分支就会被忽略。

如果所有 case 空分支中的通道都是 nil，且没有 default case，就变成 select{} 语句。空 select 会阻塞协程，引起死锁。

#### 添加超时时间

有时候，我们不希望立即执行 default 语句，而是希望等待一段时间，若一段时间后还没有可操作的通道，则执行规定的语句。

```go
select {       // 会发送阻塞
case s1 := <-ch1:
    fmt.Println(s1)
case s2 := <-ch2:
    fmt.Println(s2)
case <-time.After(2*time.Second):     // 等待 2s
    fmt.Println("no case ok")
}
```

