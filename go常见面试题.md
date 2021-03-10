# go 常见面试题

https://www.zhihu.com/question/60952598?sort=created

## 语言比较

### 可见性

Java 中属性的可见性，无论是方法、常量还是变量，都可以通过在声明前添加关键字 public、protected 或 private 来定义。声明为 public 的类成员可以从任何地方访问，private 成员只能由定义它们的类访问，protected 成员可以被继承该类的子类访问。

对于可见性，Go 语言采用了一种更简单的方法。Go 不支持继承，而是选择组合、嵌入和接口来支持代码重用和多态。可见性变为 public 或 private。在 Go 中，类型、变量、函数或方法的可见性用首字母大写声明为 public，用首字母小写声明为 private。

### 协程支持

Go 语言最大的特色就是从语言层面支持并发。通过轻量的 goroutines 和 channel 可以简便地处理并发问题。

### 面向对象/多态

Java 中类可以实例化出对象，这些对象包含内部作用域的变量属性，类行为在方法中定义。在 Go 里面类称为 struct，是用户定义的类型，它包含具有声明数据类型的字段集合。

Java 强制使用其必需的属性实例化类？而 Go 中的 struct 类型可以通过多种方式实例化。

Java 提供基于类的封装，而 Go 通过其包系统实现封装。

### 接口

Java 接口是显式实现的。implements 关键字在具体的 LocalStorage 类和 Storage 接口之间提供了显式绑定。Go 也提供了接口支持，尽管是隐式的。Go 中的接口有两层含义，首先它是一组方法的集合，其次它是一种类型。以上面的例子为例，当任何类型定义一个具有 File 参数的 Save 方法从而满足接口声明时，就会发生隐式绑定。

### 错误及异常处理

程序发生错误时，需要以特殊的方式通知到程序开发者。PHP 通过抛出异常来处理错误，这与其他编程语言非常相似。

代码被一个 try/catch 块包围并相应地进行处理。PHP 为开发人员提供了抛出异常的功能，而将捕获异常或再次抛出的责任留给消费类。通常常见的异常由全局异常处理程序捕获和处理。

Go 语言没有提供 try/catch 来处理异常。Go 中的错误是一种类型，作为函数的返回值。按照 Go 惯例，错误一般作为函数最后一个返回值，如果没有发生错误，一般返回 nil。

Go 中的错误处理非常冗长，因为需要对每个函数检查其错误返回值是否为 nil 值。

Go 语言的设计和约定鼓励我们检查出现错误的地方，而不是抛出或有时捕获异常。

### 函数/方法

Go 函数支持多返回值。

### 指针和引用

### 泛型

Go 暂不支持泛型。

### VM

### 依赖

## 数据结构

见《数据结构》

## 技术原理

### defer

见《go defer》

### 循环

```go
for k,v range slice {
    ...
}
```

for range 表达式中参与循环的是 slice 的副本。slice 可能是数组，切片或 map，应注意数组的副本是另一个数组，切片的副本其指向的底层数组还是原来那个数组。

v 是 for 循环内的局部变量，是 slice 元素值的拷贝，而不是引用。

```go
for i := range slice {
    slice = append(slice, i)
}
```

上面代码中参与循环的是 slice 的副本，循环内改变 slice 长度不影响循环次数，因此不会出现死循环，能正常结束。

### init 函数及包初始化

init 函数用于程序执行前进行包的初始化，比如初始化包里的变量等。

* 一个包甚至一个源文件中可以出现多个 init 函数。
* 同一个包中多个 init 函数，执行顺序：
  * 如果多个 init 函数在不同的源文件中，则按源文件名以字典序从小到大排序，小的先被执行。准确来说，应是按提交给编译器的源文件名顺序为准，只是在提交编译器之前，go 命令行工具对源文件名按字典序排序了。
  * 同一包且同一源文件中的 init 函数，则按其出现在文件中的先后顺序依次初始化。
* 不同包 init 函数的执行顺序由包导入的依赖关系决定，从依赖的最顶层到最底层。
* init 函数在代码中不能被显示调用，也不能被引用。

包的初始化：

* 包的初始化顺序由包导入的依赖关系决定，从依赖的最顶层到最底层。
* 一个包被引用多次也只初始化一次。
* 不可出现循环导包。解决循环导包的常见思路：todo

同一 go 文件：

* 初始化顺序： (1) 导入的包； (2) 当前包中的变量常量； (3) 当前包的 init 函数； (4) main 函数。
* 包级别变量常量按其出现在文件中的先后顺序依次初始化。还有一大原则就是被引用的先初始化，比如某个变量需要依赖其他变量，则被依赖的变量先初始化。

![image-20210224143401133](go中文网题目笔记.assets/image-20210224143401133.png)

### 值类型与引用类型

见《指针与引用》

### 指针和引用

见《指针与引用》

### ... 的使用

1. 定义可变参数函数

   ```go
   func change(str string, s ...string) {
       fmt.Printf("%T\n",s)
       fmt.Println(s)
   }
   ```

2. 切片作为函数的可变参数

   ```go
   var a = []int{1,2,3}
   change("b", a...)
   ```

3. 数组定义

   ```go
   var arr = [...]int{1,2,3,4,5}
   ```

   编译器会自动确定数组长度。

### goroutine

见《goroutine与并发编程》

### select 使用

见《goroutine与并发编程》

### context 使用

见《go context》

### 声明和赋值问题

* 单变量声明

  如`x := 1`，用于声明之前未声明过的变量 x，并赋值为 1。

* 多变量声明

  如`x, y := 1, 2`，左边的变量有一个是未声明过的就可以。如果变量 x 与同名已定义的变量 x 不在同一个作用域中，那么 go 会重新定义这个变量。

* 结构体字段不能声明

  `data.result, err := work()`出错

  := 操作符不能用于结构体字段声明，应修改为赋值的方式

  ```go
  var err error
  data.result, err = work()
  ```

* 被赋值变量可取址

  对于类似`x = y`的赋值操作，必须知道 x 的地址，才能够将 y 的值赋给 x。map 的 value 不可寻址，所以不能赋值。

* 多重赋值

  ````go
  var i, j int
  s := make([]string, 0)
  j = 1
  i，s[j-1] = 1，"a" 
  ````

  最后一行分为两个步骤执行，有先后顺序：

  1. 计算等号左边的索引表达式和取址表达式，接着计算等号右边的表达式
  2. 赋值

  所以上面赋值实际先计算索引`j-1`的值，然后执行赋值操作`i, s[0] = 1, "s"`

* 字面量初始化数组、slice 和 map 时，最好是在每个元素后面加上逗号，即使是声明在一行或者多行都不会出错。

  ```go
  x := []int{1,2,}
  y := []int{
      1,
      2,
  }
  ```

### 常量

* 常量无法使用&取址。
* 常量声明组中如不指定类型和初始化值，则与上一行非空常量相同。
* iota
  * https://www.cnblogs.com/zsy/p/5370052.html
  * iota 是 go 语言中的常数计数器，只能在常量的表达式中使用。iota 在 const 关键字出现时将被重置为 0，const 中每新增一行常量声明将使 iota 计数一次。

### 字符与字符串

#### 字符

go 中没有 char 字符类型，内置数据类型 byte 和 rune 可作为字符类型。byte 和 rune 实质上都是整数，使用 byte 和 rune 用于强调字符而不是整型值。

字符在 go 中用单引号括起来表示，字符默认的类型是 rune。也可以直接使用 byte 或 rune 指明变量的类型。

```go
var myByte = 'A'
fmt.Printf("%T\n", myByte)	// int32
var myByte2 byte = 'B'
fmt.Printf("%T\n", myByte2)	// uint8
var myByte3 rune = 'C'
fmt.Printf("%T\n", myByte3)	// int32
```

byte 是 uint8 的别名，用来表示 ASCII 字符。rune 是 int32 的别名，可表示的字符更多，用来表示以 UTF-8 格式编码的 Unicode 码点。在平时计算中文字符时用 rune。

byte 类型的 'A' 可以转成整型值97；类似的，rune 类型的 Unicode 字符 '♥' 可以转成对应的 Unicode 码点 U+2665 （U+ 用来表示 unicode，2665是十六进制数值），实质上也是整型。

```go
var myByte byte = 'a'
var myRune rune = '♥'
// 输出byte变量myByte和对应的十进制数字，rune变量myRune和对应的Unicode码
fmt.Printf("%c = %d and %c = %U\n", myByte, myByte, myRune, myRune)		// a = 97 and ♥ = U+2665
```

#### 字符串 string

* go 语言中，字符串 string 是只读的 byte 切片。

  ```go
  var a = "1h好"
  for i := range a {
      fmt.Printf("%T\n", a[i])	// uint8 uint8 uint8
  }
  ```

* 将 string 转成 rune 切片

  ```go
  s := "hello"
  r := []rune(s) 
  ```

* string 类型变量的空值为 "" 。不能赋值为 nil，也不能判断是否等于 nil。

### 比较

* 不同类型不能进行比较。
* map、slice 和 function 属于不可比较类型，不能通过 == 比较，只能判断是否为 nil。
* map，slice 可以参考用 reflect.DeepEqual 方法来进行比较，经过反射操作，效率低。也可自己实现比较方法。
* 数组长度是数组类型的一部分，不同长度的数组为不同类型，不能进行比较。
* 结构体比较
  * 结构体只能比较是否相等。
  * 结构体可比较的前提是所有成员变量的类型均可比较。
  * 相同类型的结构体才能进行比较。结构体类型是否相同不但与成员变量的类型有关，还与成员变量的顺序有关。
  * 如果结构体的所有成员都可以比较，可通过 == 或 != 比较是否相等，比较时逐项进行比较，如果每一项都相等，则两个结构体相等。

### nil 值问题

nil 只能赋值给引用类型的变量。

`var x = nil` 会导致编译错误，因为 nil 为上述类型的零值，如果不指定变量类型，编译器猜不出来变量的具体类型，导致编译错误。

### 代码编译合规规则

* 函数中声明的变量必须要使用，但可以有未使用的全局变量。函数的参数未使用也是可以的。
* 常量声明但未使用是能通过编译的，常量编译后是一个简单值的标识符。
* 导入的包如果未使用，代码不能通过编译。如果需要导入包又不想使用（只想执行包的初始化），可以用 `_ "fmt"` 使通过编译。
* 只有 `i++` 和`i--` 形式的自增自减，且只能作为独立语句，不能作为表达式。
* Go 是不提供任何隐式类型转换的，如果要在不同类型的数字之间执行相加、相减等操作，必须进行类型转换，转换成你需要的类型，语法：`T(v)`，T 就是目标类型，v 是想转的值。

### 反射

### go runtime

运行时

### go 编译

go 编译怎么从源码编译到二进制文件

### go build, go install 和 go get

* go build : 编译出可执行文件
* go install : go build + 把编译后的可执行文件放到 GOPATH/bin 目录下
* go get : git clone + go install

### 输入输出问题

- 如果某个类型实现了 String() 方法，当格式化输出的时候会自动使用 String() 方法。

- 格式化输出

  ```go
  i := -5
  fmt.printf("%+d", i)
  ```

  其中 %d 表示输出十进制数字，+ 表示输出数值的符号。

### 闭包

todo 

## 工程实践

### 编程规范

[Uber Go 语言编程规范](https://mp.weixin.qq.com/s/Q_-3UK0DH2FjFzY6eiesJg)

### Go 项目开发中常见错误

https://mp.weixin.qq.com/s/PJ6yhpwTfdIzlDWPjPy8Tw

https://mp.weixin.qq.com/s/DfoGMjmjyd6mIMa_DMeF4A

https://mp.weixin.qq.com/s/LjSlxdo5PFVJcUhH3UDU0A

### go 命令

### 包管理，go mod 使用

### go proxy

https://mp.weixin.qq.com/s/EXEgyEXF0O3_-yEGWRVSmQ

### 如何调试一个 go 程序，线上问题排查、调优

### slice 内存泄漏分析，go 内存逃逸分析

### 多小才是小对象，为什么小对象多了会造成 gc 压力

### 如何写单元测试和基准测试，什么是测试驱动开发

### go 异常处理

### go 错误处理

### Sessions 和 Cookies

### 如何限流

### websocket

### 长连接和短链接

### Go 如何解析 json 内部结构不确定的情况

https://mp.weixin.qq.com/s/I1wS0JBNnagmPtSxWrrM3Q

### 依赖注入

https://mp.weixin.qq.com/s/sjg6WkCpPU-FfpqTMjj70g

https://mp.weixin.qq.com/s/_TibuJIm2edic0KGWDjwgw



go 语言中文网公众号 面试题 day61
19 锁失效，应该 data 指针作为 test() 的方法接收者



