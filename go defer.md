# go defer

## 什么是 defer

defer 是 Go 语言提供的一种用于注册延迟调用的机制，每一次 defer 都会把函数压入栈中，当前函数返回前再把延迟函数取出并执行。

defer 语句并不会马上执行，而是会进入一个栈，当前函数 return 前，会按先进后出（FILO）的顺序执行，也就是说最先被定义的 defer 语句最后执行。先进后出的原因是后面定义的函数可能会依赖前面的资源，自然要先执行；否则，如果前面先执行，那后面函数的依赖就没有了。

defer 常用于在函数执行结束前关闭资源（文件句柄、数据库连接等）。

## defer 三条行为规则

1. 延迟函数的参数在 defer 语句出现时就已经确定，是原值的拷贝。

   > defer 语句定义时，对外部变量的引用方式有两种，分别是作为函数参数和作为闭包引用。作为函数参数，则在 defer 定义时就把值传递给 defer，并被缓存起来；作为闭包引用，则会在 defer 函数真正调用时根据整个上下文确定当前的值。

2. 延迟函数按先进后出顺序执行。

3. 延迟函数可能操作主函数的具名返回值。

   避免掉坑的关键是要理解这条语句：

   ```go
   return xxx
   ```

   这条语句并不是一个原子指令，经过编译之后得到以下指令：

   1. 返回值 = xxx
   2. 调用 defer 函数（如果有）
   3. 空的 return

   在主函数拥有具名返回值时，该值会被初始化成一个局部变量，函数内部可以像使用局部变量一样使用该返回值。如果 defer 语句操作该返回值，可能会改变返回结果。

   ```go
   func foo() (ret int) {
   	defer func() {
   		ret++
   	}()
   	return 0	// 可拆解为：1. ret = 0；2. 执行defer语句 ret++；3. return
   }
   
   func main() {
   	fmt.Println(foo())	// 打印 1，函数真正返回前，在defer中对返回值做了+1操作
   }
   ```

   

## panic、defer 和 recover

### panic

panic 可以用来终止程序并且可以自定义错误信息。当发生 panic 时，会发生如下情况：

1. 当前执行函数立即终止。
2. 与当前执行线程相关的所有 defer 函数将会被执行。
3. 整个程序会终止。

```go
func executePanic() {
	defer func() {
		fmt.Println("defer exacute")
	}()
	panic("This is Panic Situation")
	fmt.Println("The function executes Completely")
}

func main() {
	defer func() {
		fmt.Println("main defer exacute")
	}()
	executePanic()
	fmt.Println("Main block is executed completely...")
}
```

打印信息

```
defer exacute
main defer exacute
panic: This is Panic Situation

goroutine 1 [running]:
main.executePanic()
	E:/goproject/leetcode/main.go:9 +0x62
main.main()
	E:/goproject/leetcode/main.go:20 +0x4b

Process finished with exit code 2
```

### recover

一旦发生 panic，程序将会终止。然而在实际生产环境中，发生错误终止的情况是不允许的。我们需要一种从错误中恢复的机制，通过恢复代码避免程序的意外终止。

因为无论执行函数是否会发生 panic，函数返回时，defer 函数总是会被执行，所以我们可以在 defer 函数中编写恢复代码。

在 defer 函数中，我们需要检测程序执行时是否发生过 panic，这时我们需要执行 recover 函数。一旦我们执行了 recover 函数，就可以接收到 panic 函数传递的错误信息。这些错误信息作为 panic 的返回输出到 recover 函数。我们不允许正在执行的程序发生意外终止，而是要重新获得对程序的控制。程序控制权重新交还给调用函数，这样调用函数便可以接着向下继续执行。

看以下例子。

```go
func recoveryFunction() {
    if recoveryMessage := recover(); recoveryMessage != nil {
        fmt.Println(recoveryMessage)
    }
    fmt.Println("This is recovery function...")
}

func executePanic() {
    defer recoveryFunction()
    panic("This is Panic Situation")
    fmt.Println("The function executes Completely")
}

func main() {
    executePanic()
    fmt.Println("Main block is executed completely...")
}
```

上面的代码，在 defer 函数内部，我们调用了 recover 函数，该函数返回 panic 抛出的错误信息。因为我们使用了 recover 函数，所以程序并不会立即终止。相反，程序的控制权将会返回给主函数并得以继续执行。看下输出：

```
This is Panic Situation
This is recovery function...
Main block is executed completely...

Process finished with exit code 0
```

可以看到，executePanic 函数在 panic 后终止，返回前执行 defer 函数，在 defer 函数中，recover 函数接收到 panic 抛出的错误信息，程序不会异常终止，返回 main 函数继续运行。



参考文章：

https://my.oschina.net/renhc/blog/2870345