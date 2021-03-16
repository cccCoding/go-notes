# go 数据结构 -- map & set

数据类型的本质：固定内存大小的别名。

数据类型的作用：编译器预算对象或变量分配内存空间的大小。

## 映射 map

map 是由 key-value 对组成的；key 只会出现一次。

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

### map 的两种 get 操作

Go 语言中读取 map 有两种语法。

```go
value := map[key]	// 当key不存在，返回value类型的零值
```

```go
value, ok := map[key]	// 当key不存在，返回value类型的零值和bool型变量提示key是否存在
```

这两种语法的实现是编译器在背后做的工作：分析代码后，将两种语法对应到底层两个不同的函数。

```go
// src/runtime/hashmap.go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)
```

#### 扩容





### 特性

* 可以通过内建函数`delete(map, key)` 删除键值对。当 map 不存在的相应的 key 时，不报错，相当于没操作。通过键值对是否还存在来判断删除是否成功。
* map 的遍历是无序的。
* 可以使用 len 函数返回 Map 中键值对的数量。
* map 是引用类型，和副本共享底层数据，这一点和 slice 相似。在函数间传递 map 时，其实传递的是 map 的引用，不会涉及底层数据的拷贝，如果在被调用函数中修改了 map，在调用函数中也会感知到 map 的变化。要得到一个不共享底层数据的 map 副本，可以创建新的空 map，迭代旧 map 给新 map 赋值。
* map 不是并发安全的，并发操作要加上锁。也可以使用 sync 包下并发安全的 map。
* 定义 map 时，
  1. 不推荐`map[string]Student`，map 的 value Student 的属性是不可以修改的。
  2. 推荐`map[string]*Student`，map 的 value Student 的属性是可以修改的，且效率高。



map，get/put/delete 过程

map 并发安全，与 sync 下 map 比较

map 如何顺序读取



### 并发安全 map

sync 包下的 map 是并发安全的。todo



### 底层原理

#### 什么是 map

map 通常是用哈希查找表（Hash table）或者搜索树（Search tree）实现的。
哈希查找表用一个哈希函数将 key 分配到不同的桶（bucket，也就是数组的不同 index）。这样，开销主要在哈希函数的计算以及数组的常数访问时间。在很多场景下，哈希查找表的性能很高。

哈希查找表一般会存在“碰撞”的问题，就是说不同的 key 被哈希到了同一个 bucket。一般有两种应对方法：链表法和 开放地址法。链表法将一个 bucket 实现成一个链表，落在同一个 bucket 中的 key 都会插入这个链表。开放地址法则是碰撞发生后，通过一定的规律，在数组的后面挑选“空位”，用来放置新的 key。

搜索树法一般采用自平衡搜索树，包括：AVL 树，红黑树。
自平衡搜索树法的最差搜索效率是 O(logN)，而哈希查找表最差是 O(N)。当然，哈希查找表的平均查找效率是 O(1)，如果哈希函数设计的很好，最坏的情况基本不会出现。还有一点，遍历自平衡搜索树，返回的 key 序列，一般会按照从小到大的顺序；而哈希查找表则是乱序的。

时间复杂度：todo

#### go map 底层实现

Go 语言采用的是哈希查找表，并且使用链表解决哈希冲突。

#### 内存模型

在源码中，表示 map 的结构体是 hmap，它是 hashmap 的“缩写”。

#### 创建 map

实际底层调用的是 makemap 函数，主要做的工作就是初始化 hmap 结构体的各种字段。

```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap
```

注意，这个函数返回的结果：*hmap，它是一个指针，而我们之前讲过的 makeslice 函数返回的是 Slice 结构体：

```go
func makeslice(et *_type, len, cap int) slice
```

slice 的结构体定义：

```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer // 元素指针
    len   int   // 长度
    cap   int   // 容量
}
```

结构体内部包含底层的数据指针。

makemap 和 makeslice 的区别，带来一个不同点：当 map 和 slice 作为函数参数时，在函数参数内部对 map 的操作会影响 map 自身；而对 slice 却不会。

主要原因：一个是指针（hmap），一个是结构体（slice）。Go 语言中的函数传参都是值传递，在函数内部，参数会被 copy 到本地。hmap指针 copy 完之后，仍然指向同一个 map，因此函数内部对 map 的操作会影响实参。而 slice 被 copy 后，会成为一个新的 slice，对它进行的操作不会影响到实参。

#### 哈希函数

map 的一个关键点在于，哈希函数的选择。在程序启动时，会检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。这是在函数 alginit() 中完成，位于路径：src/runtime/alg.go 下。

> hash 函数，有加密型和非加密型。加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种；非加密型的一般就是查找。在 map 的应用场景中，用的是查找。选择 hash 函数主要考察的是两点：性能、碰撞概率。

#### 扩容

使用哈希表的目的就是要快速查找到目标 key，然而，随着向 map 中添加的 key 越来越多，key 发生碰撞的概率也越来越大。bucket 中的 8 个 cell 会被逐渐塞满，查找、插入、删除 key 的效率也会越来越低。





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

