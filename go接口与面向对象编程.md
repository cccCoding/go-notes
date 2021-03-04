# go 接口与面向对象编程

## 接口 interface

https://mp.weixin.qq.com/s/eDdrHwg0g7kLutDs-ejNpw

https://mp.weixin.qq.com/s/nY9oGeiZo6T6TdYTb34Llg

### 什么是接口

在 Go 语言中，接口是一组方法的集合，但不包含方法的实现、是抽象的，接口中也不能包含变量。当一个类型 T 提供了接口中所有方法的定义时，就说 T 实现了接口。接口指定类型应该有哪些方法，类型决定如何去实现这些方法。

与 Java 接口的区别：在一些面向对象的编程语言中，例如 Java，接口定义了对象的行为，只指定了对象应该做什么。行为的具体实现取决于对象。todo

### interface 的源码实现，内部结构

todo

### 接口声明

接口的声明类似于结构体，使用类型别名且需要关键字 interface：

```go
type Shape interface {
    Area() float32
}
```

### 接口实现

任何实现了方法 Area() 的类型 T，我们就说它实现了接口 Shape。

一个类型可以实现多个接口。

### 接口类型值

#### 静态类型、动态类型

变量的类型在声明时指定、且不能改变，称为静态类型。接口的静态类型就是接口本身，接口没有静态值，它指向的是动态值。接口类型的变量存的是实现接口的类型的值，该值就是接口的动态值，实现接口的类型就是接口的动态类型。

接口的动态类型又称为**具体类型**，当我们访问接口类型的时候，返回的是底层动态值的类型。

#### nil 接口值

当且仅当动态值和动态类型都为 nil 时，接口类型变量的值才为 nil。

### 空接口 interface{}

一个不包含任何方法的接口，称之为空接口，形如：interface{}。因为空接口不包含任何方法，所以任何类型都默认实现了空接口。

函数形参为 interface{} 时，可以接收任何类型的值。

```go
func Println(a ...interface{}) (n int, err error) {}
```

### 类型断言

类型断言可以用来获取接口的底层值。

```go
value := i.(Type)
```

其中 i 是接口类型变量，Type 是类型或接口。编译时会自动检测 i 的动态类型与 Type 是否一致。如果 Type 未实现接口 i，编译时报错；如果 i 的动态值不是 Type，则会报 panic 错误。

```go
value, ok := i.(Type)
```

可使用两个返回值语法，go 会自动检查上面提到的两种情况，我们只需要通过变量 ok 判断结果是否正确即可。

### 类型选择

类型选择用于将接口的具体类型与各种 case 语句中指定的多种类型进行匹配比较。

```go
func switchType(i interface{}) {
    switch i.(type) {
    case string：
        fmt.Printf("string and value is %s\n", i.(string))
    case int:
        fmt.Printf("int and value is %d\n", i.(int))
    default:
        fmt.Printf("Unknown type\n")
    }
}
```

其中 i 是接口类型变量，type 是固定关键字。只有接口类型才可以进行类型选择，使用这个可以获得接口的具体类型，每一个 case 中的类型必须实现了 i 接口。

### 接口嵌套

go 语言中，接口不能去实现或集成别的接口，但是可以通过嵌套接口创建新接口。

```go
type Math interface {
    Shape
    Object
}
type Shape interface {
    Area() float32
}
type Object interface {
    Perimeter() float32
}
```

通过嵌套接口 Shape 和 Object，创建了新接口 Math。任何类型如果实现了接口 Shape 和 Object 定义的方法，则说类型也实现了接口 Math。

### 使用指针接收者和值接收者实现方法

除了通过值接收者实现接口，还可以通过指针接收者实现接口。

```go
type Shape interface {
    Area() float32
}
type Square struct {
    side float32
}
func (s *Square) Area() float32 {
    return s.side * s.side
}
```

值接收者的方法可以使用值或者指针调用，而对于指针接受者的方法，用一个指针或者一个可取得地址的值来调用都是合法的。

应注意，接口存储的具体值是不可寻址的。？

## 面向对象编程

https://mp.weixin.qq.com/s/HzMVsMVcr4O_4rfPPIXnqg

https://mp.weixin.qq.com/s/gB1DtheWudofohjKGXcmGQ

Go 语言没有对象的概念，但是 struct 类型有着和对象类似的特性。struct 类型可以定义自己的属性和方法。Go 语言中的“继承”和多态围绕 struct 展开。

### 多态

多态就是同一个接口，使用不同的实例而执行不同操作。

### 嵌入类型

**嵌入类型是指将已有的类型直接声明在新的结构类型里。**Go 语言没有继承，但是可以通过组合的方式实现代码的复用。

```go
type User struct {
    Name string
    Email string
}
type Admin struct {
    User
    Level string
}
func (u *User) Speak()  {
    fmt.Println("I am user",u.Name)
}
```

上面的代码定义了两个结构体 User 和 Admin，Admin 有一个匿名成员 User（定义结构体时只指定成员类型，不用指定成员名，Go 会自动地将成员类型作为成员名）。将 User 嵌入 Admin，Admin 是被嵌入的类型，也称**外部类型**，User 是**内部类型**。Speak() 是 User 的方法。

通过嵌入，内部类型的属性、方法可以为外部类型所用。此外，外部类型还可以定义自己的属性和方法，甚至和内部类型相同的方法，这样内部类型的方法就会被“屏蔽”。

如果内部类型实现接口 A，可以认为外部类型也实现了接口 A。

假设外部结构体类型是 S，内部类型是 T，调用内部类型方法的提升规则如下：

* T 嵌入 S，外部类型 S 可以通过值类型或指针类型调用内部类型 T 的值接收者方法；
* T 嵌入 S，外部类型 S 只能通过指针类型调用内部类型 T 的指针接收者方法；
* *T 嵌入 S，外部类型 S 可以通过值类型和指针类型调用内部类型 T 的值接收者方法和指针接收者方法；

总结成一句话：不管是 T 嵌入 S，还是 *T 嵌入 S，外部类型 S 唯独通过值类型不能调用内部类型 T 的指针方法外，其他情况下内部类型 T 的方法都可以被外部类型 S 访问。

```go
type Admin struct {
    *User            // 通过指针方式组合
    Level string
}
```

在 Go 语言中，每种类型都是不同的，但不同的类型可以实现同一接口，将它们绑定在同一接口上，用作函数或者方法的输入（输出）参数。