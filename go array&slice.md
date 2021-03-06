# go 数据结构 -- array&slice

数据类型的本质：固定内存大小的别名。

数据类型的作用：编译器预算对象或变量分配内存空间的大小。

## 数组 array

数组是同一种数据类型的固定长度的序列，指向一段连续的内存空间。

### 声明

```go
arr := [6]int{1,2,3,4,5,6}
```

### 特性

* 因为数组的内存分布是连续的，所以数组访问任一元素的效率很高，O(1)。
* 元素类型和数组长度都是数组类型的一部分，不同长度的数组是不同的类型。
* 数组是值类型，改变其副本的值不会改变原数组的值。

### 数组指针和指针数组

**数组指针**

即声明一个指针变量，指向一个数组：

```go
arr := [6]int{5:9}	//第6个元素为9
var ptr *[6]int = &arr
```

需要注意的是，指针变量`ptr`的类型是`*[6]int`，也就是说它只能指向包含6个元素的整型数组，否则编译报错。

**指针数组**

即数组的元素类型是指针。

```go
x,y := 1,2
var arrPtr = [5]*int{1:&x,3:&y}
```

### 函数间传递数组

如果实参是一个非常大的数组，因函数参数只有值传递，即需要重新拷贝变量，拷贝导致的内存和性能开销较大。可将数组指针作为参数传递，拷贝开销小，但也需要考虑到函数内数组指针修改会影响原数组。

## 切片 slice

切片与数组类似，存放相同数据类型的元素，不同的是切片基于底层数组，可按需扩缩容。切片是底层数组的一个视图，也可以说切片是对数组的抽象。

切片的内部实现中有三个变量，指针 ptr，长度 len 和容量 cap。

```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer // 指针，数据存储在底层数组中，而指针指向可以通过切片访问到的第一个元素。
    len   int   		 // 长度，我们只能访问切片长度范围内的元素。
    cap   int   		 // 容量，表示可以扩展的最大大小。容量必须大于等于长度。
}
```

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ibvOicqJ38kUdlc7Or1hfWBhI3saicOMy3Q6wE3ZFFMvRmibQ2cCAR3tTMO3VRdvp0wjOT3WCLl2MjhicLhCykyicuxw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

应注意底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。

### 声明与初始化

#### make 函数创建

```go
s1 := make([]int,3,5)	// 创建一个可用的空切片，长度为3，容量为5，容量也可以不传
fmt.Println(s1)			// [0 0 0]
```

#### 字面量创建

```go
s2 := []int{1,2,3,4,5}  // 创建长度和容量都是5的整型切片
```

#### 截取已有的数组或者切片创建

```go
a := []int{1,2,3,4,5}
t := a[3:4]
```

截取得到的切片，和原切片或原数组共享底层数组，但是两者能访问到底层数组的范围是不同的。

**截取获得的新切片的长度和容量计算**

对容量为 k 的切片执行`[i,j]`操作之后，获得的新切片的长度和容量是`j-i`和`k-i`。

对容量为 k 的切片执行`[i,j,l]`操作之后，获得的新切片的长度和容量是`j-i`和`l-i`，其中第三个变量 l 用于限定新切片的容量，j 和 l 必须在原数组或者原 slice 的**容量**范围内（小于等于 k）。

**新切片与原数组或者切片的关系**

截取得到的新切片和原数组或原切片是基于同一个底层数组的，所以当修改的时候，底层数组的值就会被改变，原切片的值也随之改变了。

可以把截取得到的新切片看做是原数组或原切片的一个视图。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ibvOicqJ38kUdlc7Or1hfWBhI3saicOMy3Q8nShOv6RUwA0tkGLH8ic7P9RF3wPPvzl2ujibRFxkLkmnibBkoCgxIsAw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

但如果因为执行 append 操作使得新 slice 底层数组扩容，移动到了新的位置，两者就不会相互影响了。所以，问题的**关键在于两者是否会共用底层数组**。

#### nil 切片和空切片

**nil 切片**

```go
var s []int					// 声明了一个 nil 切片
fmt.Println(s == nil)   	// 输出 true
fmt.Println(len(s), cap(s)) // 输出：0 0
s = append(s, 1)			// 使用 append 函数可以为 nil 切片增加元素
```

切片的零值是 nil。因为切片是底层数组的引用，nil 切片指向底层数组的指针为 nil，即不指向任何底层数组。

**空切片**

```go
s := make([]int, 0)			// 1、使用 make 创建空的整型切片
s := []int{}				// 2、使用切片字面量创建空的整型切片
fmt.Println(s == nil)   	// 输出 false
fmt.Println(s)   			// 输出：[]
fmt.Println(len(s), cap(s)) // 输出：0 0
```

与 nil 切片一样，空切片的长度和容量也都是 0，说明切片底层的数组大小为 0，是一个空数组（没有分配任何的存储空间）。

不管是使用 nil 切片还是空切片，对其调用内置函数 append、len 和 cap 的效果都是一样的。官方建议尽量使用 nil 切片。

nil slice 可以直接 append ，因为 append 最终都是调用 mallocgc 来向 Go 的内存管理器申请到一块内存，然后再赋给原来的 nil slice 或 empty slice。

### 增长/扩容

使用内建函数 append 能够帮我们处理切片增长的一些细节。

* append 可往切片追加一个或多个值，然后返回一个新的切片。应注意 Go 编译器不允许调用了 append 函数后不使用返回值。
* append 函数会使新的切片长度增加。
* append 函数实际上是往底层数组添加元素。
* 新切片容量是否增长取决于原切片剩余容量和需要追加的元素数目。当剩余容量不足时，append 函数会创建一个新的数组并将原数组元素拷贝到新数组中，再追加新的值。append 函数会智能地增加底层数组的容量，目前的算法是：当数组容量小于等于1024时，会成倍地增加；当超过1024，增长因子变为1.25，也就是说每次会增加25%的容量（这个说法是错误的，还有内存对齐的操作）。

要注意的是，通过截取创建的切片，如果切片剩余容量能存下 append 追加的元素，切片长度增长而不扩容，追加的元素会改变原切片或数组的值；如果不能存下追加的元素，切片长度增长并进行扩容，扩容操作为 append 函数创建一个新的底层数组，将原数组的值复制到新数组里，再追加新的值，因为新切片与原切片或数组的底层数组不再相同，追加的元素不会改变原切片或数组的值，且后续两个切片不再相互影响。**关键在于两者是否会共用底层数组**。

**一般我们在创建新切片的时候，最好要让新切片的长度和容量一样，这样我们在追加操作的时候就会生成新的底层数组，和原有数组分离，就不会因为共用底层数组而引起奇怪问题。**

### copy 函数

Go 提供了内置函数 copy，可以将一个切片复制到另一个切片。

```go
func copy(dst, src []Type) int	// 函数返回两者长度的最小值
```

如果 dst 切片为 nil 切片，copy 之后，dst 切片仍为 nil。而 nil 切片 append 非空切片后，变为非空切片。

copy 只会复制，不会追加。

### 函数间传递切片

切片在函数间以值的方式传递。当直接用切片作为函数参数时，可以改变切片的元素，不能改变切片本身；想要改变切片本身，可以将改变后的切片返回，函数调用者接收改变后的切片或者将切片指针作为函数参数。

由于切片的尺寸很小（在 64 位架构的机器上，一个切片需要 24 字节的内存：指针字段、长度和容量字段各需要 8 字节），在函数间复制和传递切片成本很低。

### 删除切片中的元素

Go 没有提供删除切片元素的函数，可以通过截取和 append 函数实现。

```go
s := []int{1, 2, 3, 4, 5, 6}
s = append(s[:2], s[3:]...)    // 删除索引为2的元素
```

[其他骚操作](https://mp.weixin.qq.com/s/FymHB5hERd29ypUOCZaRJA)

### 切片垃圾回收

巨型 slice 产生的垃圾回收问题