# go 函数与方法

## 函数

函数是一段能够完成特定功能的代码段，可以接收输入参数或者能够返回想要的值。

### 声明

```go
func funcName(paramName type) returntype {
    // 函数体
}
```

### 特性

* 函数允许有多个返回值。

* 在函数定义的时候，可以给所有的返回值分别命名，Go 会自动创建这些变量，在函数结束时，直接使用`return`就可以将返回值返回。

* 函数参数只有值传递，没有引用传递，即全部需要重新拷贝变量，可通过传递指针或者引用类型变量实现引用传递的效果。

* 函数也是一种类型。如果两个函数的入参类型和返回值类型均相同，则它们是同种类型。在将函数作为参数传递给另一个函数时或将函数作为返回值时，函数类型就会显得很有用了。

  ```go
  type calculateFunc func(int,int) int
  func calculate(a,b int,f calculateFunc) (r int) {
      r = f(a,b)
      return
  }
  ```

* 函数不能重载（函数名相同，入参等不同）。

### 可变参数函数

可变参数函数是接收可变数量的参数的函数。只允许最后一个参数是可变参数。

Go 中的一些内置函数都是可变参数函数，如 append() 函数：

```go
func append(slice []Type, elems ...Type) []Type
```

在上面的函数声明中，elems 就是可变参数，可以接收任意数目的实参。

可变参数的工作原理就是把可变参数转为切片，然后传入函数中。在函数中打印可变参数的类型和值，可见传入函数中的 elems 实际是 []Type 类型的切片。

可以不传可变参数，这种情况下，elems 是一个长度和容量都为 0 的 nil 切片。

#### 切片传递给可变参数函数

```go
func change(s ...string) {
    fmt.Printf("%T\n",s)
    fmt.Println(s)
}

func main() {
    slice := []string{"Hello","World","Go"}
    change(slice...)	// 使用语法糖，切片加上后缀 ...
}
```

使用语法糖，直接在切片后加上 ... 后缀，可将切片直接传入函数，不会根据可变参数创建新的切片。也因为这样，在函数中改变切片的值，会影响到函数外的切片。

也可以通过将数组转化成切片传递给可变函数参数。

```go
func change(s ...string) {
    fmt.Printf("%T\n",s)
    fmt.Println(s)
}

func main() {
    slice := [3]string{"Hello","World","Go"}
    change(slice[:]...)
}
```

### 匿名函数

函数可以作为一个值，这样便可以将函数赋给一个变量或者返回一个函数变量。

```go
var add = func(a,b int) (r int) {
    r = a + b
    return
}

func main() {
    fmt.Println("a + b =",add(1,2))
}
```

上面的代码，创建了一个匿名函数，赋给了一个全局变量`add`。Go 编译器会自动判别变量`add`的类型，这里是`func(int, int) int`。

匿名函数可以赋值，也可以立即执行：

```go
func main() {
    sum := func(a,b int) int {
        return a + b
    }(1,2)
    fmt.Println(sum)
}
```

#### 闭包

Go 语言中闭包是引用了自由变量的函数，被引用的自由变量和函数一同存在，即使已经离开了自由变量的环境也不会被释放或者删除，在闭包中可以继续使用这个自由变量，因此，简单的说：

>  函数 + 引用环境 = 闭包

同一个函数与不同引用环境组合，可以形成不同的实例，如下图所示。

![img](http://c.biancheng.net/uploads/allimg/180814/1-1PQ41F62I51.jpg)

一个函数类型就像结构体一样，可以被实例化，函数本身不存储任何信息，只有与引用环境结合后形成的闭包才具有“记忆性”，函数是编译期静态的概念，而闭包是运行期动态的概念。

Go 语言支持匿名函数，可作为闭包。

```go
func Increase() func() int {
    // ---------闭包
	n := 0
	return func() int {	// 匿名函数作为返回值，不如理解为闭包作为函数的返回值
		n++
		return n
	}
    // ---------
}

func main() {
	in := Increase()	// 闭包被返回赋予一个同类型的变量时，同时赋值的是整个闭包的状态，该状态会一直存在外部被赋值的变量in中，直到in被销毁，整个闭包也被销毁。
	fmt.Println(in())	// 1
	fmt.Println(in())	// 2
}
```

并发编程时，一定要处理好循环中的闭包引用的外部变量。

```go
func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		/*go func() {
			fmt.Println(i)	// 输出 5 5 5 5 2，多个5，因为i最终会被赋值为5，这候协程才去拿值打印 
			wg.Done()
		}()*/
		go func(i int) {
			fmt.Println(i)	// 乱序输出 4 2 1 3 0，因为开启协程时，实参i的值就被拷贝到形参，实际打印顺序和协程实际执行顺序有关
			wg.Done()
		}(i)
	}
	wg.Wait()
}
```

### 内建函数

* new 和 make

`new(T)` 和 `make(T, args)` 是 Go 语言内建函数，用来分配内存，但适用的类型不一样。new 为 T 类型的变量分配已置零的内存空间，并返回地址（变量是一个指针），适用于值类型，如 int、数组、结构体等。make 得到的是初始化之后的 T 的引用，用来分配引用类型，只适用于 slice、map 和 channel。

当使用 new 创建对象时返回的是对象的地址，所以如果你将 new 返回值作为参数传递，实际传递的是对象的引用。在被调函数中，对象的修改会影响到对象的原始值。

通过 make 创建的切片可以 copy，其他方式不行。

通过 make 创建的 map 是空 map，通过字面量形式创建的 map 是 nil，切片同理。

使用 make 创建 map 变量时可以指定第二个参数，不过会被忽略。

* cap 和 len

cap 获取容量，len 获取长度。

cap 适用于数组、数组指针、slice 和 channel。不适用于 map。

可以用 len 返回 map 的元素个数。

* close

主要用来关闭 channel。

* append

用来追加元素到数组、slice 中。

* panic 和 recover

用来做错误处理。

todo：panic，recover，defer 的关系。

当协程遇到 panic 时，会遍历该协程的 defer 并依次执行（执行顺序是先进后出）。如果在 defer 执行的过程遇到 recover 则停止 panic，返回 recover 继续向下执行，如果没有遇到 recover，遍历完 defer，然后向 stderr 抛出 panic 信息。如果执行第一个 defer 出现异常，顺序执行下一个 defer？

## 方法

### 声明

```go
func (receiver Type) funcName(...Type) Type {...} 
```

方法声明和函数类似，区别在于：方法声明时，在 func 和方法名之间会增加一个额外的参数(receiver Type)，其中 receiver 称为接收者，Type 可以是任意合法的类型。可以说，该方法属于类型 Type。

即：方法属于某一种类型，且有接收者。

### 特性

* 方法的接收者是调用者的副本。
* 我们可以在方法内部访问 receiver 的每一个成员。

* 必须保证类型和其方法定义在同一个包里。如果做不到，可以创建类型别名。
* 每个方法声明的时候，编译器会各自声明相对应的隐式函数。
* 不能用多级指针调用方法。

### 值接收者方法和指针接收者方法

接收者类型为 T 的方法称为值接收者方法；接收者类型为 *T 的方法称为指针接收者方法。

其中 T 必须满足一下条件：

* T 必须是自定义类型。
* T 的定义必须和方法的声明在同一个包内。
* T 不能是接口类型或者接口指针类型。

值接收者方法和指针接收者方法最大区别在于，在方法中修改指针接收者的值会影响到调用者的值（除非这个类型是应用类型的别名类型，如切片），而修改值接收者的值不会。因为方法的接收者是调用者的副本，一个是值的副本，一个是指针的副本，指针的副本指向的还是原来的值。

#### 值方法和指针方法的使用场景

根据场景考虑使用值接收者还是指针接收者方法。

* 使用值方法时，如果变量很大，拷贝成本高，这时应考虑使用指针方法。
* 确认需要在方法中直接修改调用者的值时，使用指针方法；否则，可能会造成混乱，随着代码的增长，很容易就会出现错误，因为调用堆栈深处的某个地方改变了指针指向的值。

**当确认需要在方法中直接修改调用者的值，或值接收者变量很大，拷贝成本高时，应考虑使用指针接收者方法。其他情况建议使用值接收者方法。**

### 方法集

方法集是一组关联到自定义类型的值或指针的方法。

一个自定义类型 T 的方法集仅包括它的值接收者方法，而该类型的指针类型 *T 的方法集包括它的值接收者方法和指针接收者方法。

调用指针接收者方法也可以写成值调用`a.A()`形式，编译器会自动帮我们转成指针调用`(&a).A()`，以满足接收者的要求。调用值接收者方法也可以写成指针调用，因为值接收者方法属于该类型的指针类型 *T 的方法集。

值接收者的方法可以使用值或者指针调用，而对于指针接收者的方法，用一个指针或者一个可取得地址的值来调用都是合法的。编译器不总能自动获得到一个值的地址，要保证可寻址的结构体才可以调用该结构体指针接收者的方法。如下代码会报错：

```go
// 定义接口 notifier
type notifier interface {
    notify()
}

type user struct {
    name string
    email string
}

func (u *user)notify()  {
    fmt.Printf("Sending user email to %s<%s>\n", u.name, u.email)
}

// 接收一个 notifier 接口类型的参数，并发送通知
func sendNotification(n notifier)  {
    n.notify()
}

func main() {
    u := user{"Seeklaod", "email@gmail.com"}
    sendNotification(u)		// 编译器不能自动获得到值的地址，不能自动转译
}
```

报错信息为：

```
cannot use u (type user) as type notifier in argument to sendNotification:
user does not implement notifier (notify method has pointer receiver)
```

### 非结构体类型的方法

可以在 Go 任一合法类型上定义方法。

```go
package main
import "fmt"

type myInt int

func (i myInt) echo ()  {
    fmt.Println(i)
}

func main() {
    var a myInt
    a = 20
    a.echo()
}
```

但必须保证类型和方法定义在同一个包里，否则编译报错。对于一些基本类型如 int，可以创建类型别名。

## 为什么有了函数还需要方法

1. Go 不是纯粹的面向对象的语言且不支持类，通过类型的方法可以实现和类相似的功能，又不会像类那样显得很“重”；
2. 同名的方法可以定义在不同的类型上，但是函数名不允许相同。