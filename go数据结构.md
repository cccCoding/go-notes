# go 数据结构

数据类型的本质：固定内存大小的别名。

数据类型的作用：编译器预算对象或变量分配内存空间的大小。

## 数组

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

声明一个指针变量，指向一个数组：

```go
arr := [6]int{5:9}	//第6个元素为9
var ptr *[6]int = &arr
```

需要注意的是，指针变量`ptr`的类型是`*[6]int`，也就是说它只能指向包含6个元素的整型数组，否则编译报错。

指针数组即数组的元素类型是指针。

```go
x,y := 1,2
var arrPtr = [5]*int{1:&x,3:&y}
```

### 函数间传递数组

函数参数只有值传递，即需要重新拷贝变量，可通过传递指针或者引用类型变量实现引用传递的效果。

如果变量是一个非常大的数组，拷贝导致的内存和性能开销大。可将数组指针作为参数传递，拷贝开销小，但也需要考虑到函数内数组指针修改会影响原数组的问题。

## 切片 slice

切片与数组类似，存放相同数据类型的元素，不同的是切片基于底层数组，可按需扩缩容。可以说切片是对数组的抽象。

切片的内部实现中有三个变量，指针 ptr，长度 len 和容量 cap。

* 指针：指针指向可以通过切片访问到的第一个元素。
* 长度：我们只能访问切片长度范围内的元素。
* 容量：表示可以扩展的最大大小。容量必须大于等于长度。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ibvOicqJ38kUdlc7Or1hfWBhI3saicOMy3Q6wE3ZFFMvRmibQ2cCAR3tTMO3VRdvp0wjOT3WCLl2MjhicLhCykyicuxw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 声明与初始化

#### make 函数创建

```go
s1 := make([]int,3,5)	// 创建一个可用的空切片，长度为3，容量为5
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

截取操作之后，得到两个共享底层数组的切片，但是两个切片能访问到底层数组的范围是不同的。

**截取获得的新切片的长度和容量计算**

对容量为 k 的切片执行`[i,j]`操作之后获得的新切片的长度和容量是`j-i`和`k-i`。

对容量为 k 的切片执行`[i,j,l]`操作之后获得的新切片的长度和容量是`j-i`和`l-i`，其中第三个变量 l 用于限定新切片的容量，l 不能超过原数组长度或原切片容量。

**新切片与原数组或者切片的关系**

截取得到的新切片和原数组或原切片是基于同一个底层数组的，所以当修改的时候，底层数组的值就会被改变，原切片的值也随之改变了。

可以把截取得到的新切片看做是原数组或原切片的一个视图。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/ibvOicqJ38kUdlc7Or1hfWBhI3saicOMy3Q8nShOv6RUwA0tkGLH8ic7P9RF3wPPvzl2ujibRFxkLkmnibBkoCgxIsAw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

#### nil 切片和空切片

**nil 切片**

```go
var s []int			// 声明了一个 nil 切片
fmt.Println(s == nil)   	// 输出 true
fmt.Println(len(s), cap(s)) // 输出：0 0
s = append(s, 1)	// 使用 append 函数可以为 nil 切片增加元素
```

切片的零值是 nil。因为切片是底层数组的引用，nil 切片指向底层数组的指针为 nil，即不指向任何底层数组。

**空切片**

```go
s := make([]int, 0)	// 1、使用 make 创建空的整型切片
s := []int{}		// 2、使用切片字面量创建空的整型切片
fmt.Println(s == nil)   	// 输出 false
fmt.Println(s)   	// 输出：[]
fmt.Println(len(s),cap(s))   // 输出：0 0
```

与 nil 切片一样，空切片的长度和容量也都是 0，说明切片底层的数组大小为 0，是一个空数组（没有分配任何的存储空间）。

> 不管是使用 `nil` 切片还是空切片，对其调用内置函数`append`、`len`和`cap`的效果都是一样的。

### 增长/扩容

使用内建函数 append 能够帮我们处理切片增长的一些细节。

* append 可往切片追加一个或多个值，然后返回一个新的切片。
* append 函数会使新的切片长度增加。
* 新切片容量是否增长取决于原切片剩余容量和需要追加的元素数目。当剩余容量不足时，append 函数会创建一个新的数组并将原数组元素拷贝到新数组中，再追加新的值。append 函数会智能地增加底层数组的容量，目前的算法是：当数组容量小于等于1024时，会成倍地增加；当超过1024，增长因子变为1.25，也就是说每次会增加25%的容量。

要**注意**的是，通过截取创建的切片，如果切片剩余容量能存下 append 追加的元素，切片长度增长而不扩容，追加的元素会改变原切片或数组的值；如果不能存下追加的元素，切片长度增长并进行扩容，即 append 函数创建一个新的底层数组，将原数组的值复制到新数组里，再追加新的值，因为新切片与原切片或数组的底层数组不再相同，追加的元素不会改变原切片或数组的值，且后续两个切片不再相互影响。

一般我们在创建新切片的时候，最好要让新切片的长度和容量一样，这样我们在追加操作的时候就会生成新的底层数组，和原有数组分离，就不会因为共用底层数组而引起奇怪问题。

### copy 函数

Go 提供了内置函数 copy，可以将一个切片复制到另一个切片。

```go
func copy(dst, src []Type) int	// 函数返回两者长度的最小值
```

如果 dst 切片为 nil 切片，copy 之后，dst 切片仍为 nil。而 nil 切片 append 非空切片后，变为非空切片。

copy 只会复制，不会追加。

### 函数间传递切片

切片在函数间以值的方式传递。由于切片的尺寸很小（在 64 位架构的机器上，一个切片需要 24 字节的内存：指针字段、长度和容量字段各需要 8 字节），在函数间复制和传递切片成本也很低。切片发生复制时，底层数组不会被复制，还是原数组。

### 删除切片中的元素

Go 没有提供删除切片元素的函数，可以通过截取和 append 函数实现。

```go
s := []int{1, 2, 3, 4, 5, 6}
s = append(s[:2], s[3:]...)    // 删除索引为2的元素
```

[其他骚操作](https://mp.weixin.qq.com/s/FymHB5hERd29ypUOCZaRJA)

## 映射 map

Hash 表是一种巧妙并且实用的数据结构，是一个无序的键值对集合，其中所有的 key 都是不同的，通过给定的 key 可以在常数时间复杂度内检索、更新和删除对应的 value。map 其实是一个 Hash 表的引用，能够基于键快速检索出数据，键就像索引一样指向与该键关联的值。总之，map 存储的是无序的键值对集合。

### 创建与初始化

#### make 函数创建

```go
m := make(map[keyType]valueType)
```

map 的 key 必须是可以使用 `==` 运算符做比较的类型，map 的 value 没有类型限制。

#### 字面量创建

```go
month := map[string]int{"January":1,"February":2,"March":3}
```

#### nil map 和空 map

**nil map**

```go
var month map[string]int	// 声明了一个 nil map
fmt.Println(month == nil)   // 输出：true
```

nil map 是不能存取键值对的，会报 panic 错误。可以使用 make 函数为其初始化。

map 的零值就是 nil，map 就是底层 Hash 表的引用。

**空 map**

```go
month := make(map[string]int)	// make 函数创建空 map

month := map[string]int{}		// 字面量创建空 map
fmt.Println(month)        		// 输出：map[]
```

### 特性

* `a := map[key]` 获取一个 map 中不存在的键值对时，返回值类型的零值。
* 可以通过 `if _, ok := map[key]; ok {}` 判断键值对是否存在。
* 可以通过内建函数`delete(map, key)` 删除键值对。当 map 不存在的相应的 key 时，不报错，相当于没操作。通过键值对是否还存在来判断删除是否成功。
* map 的遍历是无序的。
* 可以使用 len 函数返回 Map 中键值对的数量。
* map 是引用类型，和副本共享底层数据，这一点和 slice 相似。在函数间传递 map 时，其实传递的是 map 的引用，不会涉及底层数据的拷贝，如果在被调用函数中修改了 map，在调用函数中也会感知到 map 的变化。要得到一个不共享底层数据的 map 副本，可以创建新的空 map，迭代旧 map 给新 map 赋值。
* map 不是并发安全的，并发操作要加上锁。也可以使用 sync 包下并发安全的 map。
* 定义 map 时，
  1. 不推荐`map[string]Student`，map 的 value Student 的属性是不可以修改的。
  2. 推荐`map[string]*Student`，map 的 value Student 的属性是可以修改的，且效率高。

### 并发安全 map

sync 包下的 map 是并发安全的。todo

## Mutex

锁，sync 包下，提供 Lock 和 Unlock 方法，所有在 Lock 和 Unlock 之间的代码都只能由一个 goroutine 执行，避免竞态条件。如果一个协程已经持有了锁（Lock），其他协程试图获得该锁时会被阻塞，直到 Mutex 解除锁定（Unlock）。

可以用容量为 1 的 channel 处理竞态条件。例子：todo

## RWMutex

读写锁。RWMutex 基于 Mutex，在 Mutex 的基础上增加了读、写的信号量，使用了类似引用计数的读锁数量。Lock 和 Unlock 方法用于申请和释放写锁，RLock 和 RUnlock 方法用于申请和释放读锁。

读写锁适合读多写少的场景。

读锁与读锁兼容，读锁与写锁互斥，写锁和写锁互斥。在锁释放后才可以继续申请互斥的锁：

* 可以同时申请多个读锁。
* 有读锁时申请写锁将阻塞，有写锁时申请读锁将阻塞。
* 只要有写锁，申请读锁和写锁都将阻塞。

无论时 Mutex 还是 RWMutex 都不会和 goroutine 进行关联，可以在一个 goroutine 申请锁，在另一个 goroutine 释放锁。

## 集合 Set

Go 语言里面没有集合这种数据结构。可以使用 map 实现集合 Set，map 中的 key 都是唯一的。

```go
type Set struct {
    m map[string]bool
}

func NewSet() Set {
	m := make(map[string]bool)
	return Set{m: m}
}

func (s *Set) Contains(val string) bool {
	_, ok := s.m[val]
	return ok
}

func (s *Set) Add(val string) {
	s.m[val] = true
}

func (s *Set) Remove(val string) {
	delete(s.m, val)
}
```

使用 map 作为集合的底层数据结构的好处在于，map 基于 hash 表实现，减值查找效率高。使用这种方法可以少写很多代码。

## 常见题目

1. go的源码实现，各种数据结构
2. 数组切片区别
3. channel的数据结构、源码，为什么它可以做到线程安全？channel是怎么通过注册相关goroutine id实现消息通知的。
4. 如何用 channel 实现一个令牌桶？怎么用channel实现线程池，select的执行顺序一类的；
5. 通道异常情况，怎样判断channel是否关闭
6. 了解读写锁吗，原理是什么样的，为什么可以做到？
7. interface，底层结构
8. map，get/put/delete 过程，扩容机制；map，并发，与 sync 下 map 比较；map如何顺序读取？
9. slice的内存结构，len，cap，共享，扩容机制，巨型slice产生的垃圾回收问题；
10. mutex，mutex和channel作并发控制你喜欢用哪个，哪个快，为什么。
11. WaitGroup
12. function，引用类型
13. 实现 set