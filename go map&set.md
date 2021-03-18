# go 数据结构 -- map & set

## 映射 map

### 什么是 map

map 是由一组键值对组成的抽象数据结构，并且键只会出现一次。

map 通常是用哈希查找表（Hash table）或者搜索树（Search tree）实现的。

哈希查找表用一个哈希函数将 key 分配到不同的桶（bucket，也就是数组的不同 index）。这样，开销主要在哈希函数的计算以及数组的常数访问时间。在很多场景下，哈希查找表的性能很高。

哈希查找表一般会存在“碰撞”的问题，就是说不同的 key 被哈希到了同一个 bucket。一般有两种应对方法：链表法和开放地址法。链表法将一个 bucket 实现成一个链表，落在同一个 bucket 中的 key 都会插入这个链表。开放地址法则是碰撞发生后，通过一定的规律，在数组的后面挑选“空位”，用来放置新的 key。

搜索树法一般采用自平衡搜索树，包括：AVL 树，红黑树。自平衡搜索树法的最差搜索效率是 O(logN)，而哈希查找表最差是 O(N)。当然，哈希查找表的平均查找效率是 O(1)，如果哈希函数设计的很好，最坏的情况基本不会出现。

两种实现方案还有一点区别，遍历自平衡搜索树，返回的 key 序列一般是从小到大的顺序序列；而哈希查找表是乱序的。

**Go 语言中的 map 使用哈希查找表实现的，并且使用链表解决哈希冲突。**

### 底层原理

参考[《深度解密Go语言之map》](https://mp.weixin.qq.com/s/2CDpE5wfoiNXm1agMAq4wA)

#### map 内存模型

在源码中，表示 map 的结构体是 hmap，它是 hashmap 的“缩写”。

```go
// A header for a Go map.
type hmap struct {
    // 元素个数，调用 len(map) 时，直接返回此值
    count     int
    flags     uint8
	// buckets 的对数 log_2
    B         uint8
    // overflow 的 bucket 近似数
    noverflow uint16
    // 计算 key 的哈希的时候会传入哈希函数
    hash0     uint32
    // 指向 buckets 数组，大小为 2^B，如果元素个数为0，就为 nil
    buckets    unsafe.Pointer
    // 二倍扩容的时候，buckets 长度会是 oldbuckets 的两倍
    oldbuckets unsafe.Pointer
    // 指示扩容进度，小于此地址的 buckets 迁移完成
    nevacuate  uintptr
    extra *mapextra // optional fields
}
```

buckets 指针指向大小为 2^B 的 bucket 数组，每个 bucket 里存储了 key 和 value。bucket 的结构体名称为 bmap，就是“桶”，桶里面有 8 个 cell，最多装 8 个 key。

```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

不同 key 之所以会落入同一个桶，是因为他们经过哈希计算后，哈希结果属于同一类。在桶内，又会根据 key 计算出来的哈希值的高 8 位来判断 key 落入桶中的那个位置。

hmap 的内存模型如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx61pib1iaeK6CYYicjtlSl0HrycEvYofWxQWP0fnXSqqfwRFKt8HSJ7HP2qic0mqfEv9w82B0Qvpg1OJNg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

bmap 的内存模型如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx61pib1iaeK6CYYicjtlSl0HrycRIjnUcLIJJSRzDeGXQW7eFbcsfIF69fHIyy8RgHj7f9ibI4pQVUwyHA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 overflow 指针连接起来。

注意到 key 和 value 是各自放在一起的，并不是 `key/value/key/value/...` 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间。

#### 哈希函数

map 的一个关键点在于，哈希函数的选择。在程序启动时，会检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。

> hash 函数，有加密型和非加密型。加密型的一般用于加密数据、数字摘要等，典型代表就是 md5、sha1、sha256、aes256 这种；非加密型的一般就是查找。在 map 的应用场景中，用的是查找。选择 hash 函数主要考察的是两点：性能、碰撞概率。

#### key 定位过程

key 经过哈希计算后得到哈希值，共 64 个 bit 位，计算它到底要落在哪个桶时，只会用到最后 B 个 bit 位。B 是 上文中 hmap 结构体的一个字段。如果 B = 5，那么桶的数量，也就是 buckets 数组的长度是 2^5 = 32。

例如，现在有一个 key 经过哈希函数计算后，得到的哈希结果是：

```
10010111 | 000011110110110010001111001010100010010110010101010 │ 01010
```

用最后的 5 个 bit 位，也就是 `01010`，值为 10，也就是 10 号桶。这个操作实际上就是取余操作，但是取余开销太大，所以代码实现上用的位操作代替。

再用哈希值的高 8 位，找到此 key 在 bucket 中的位置，这是在寻找已有的 key。若桶内没有该 key，新加入的 key 会找到第一个空位，放入，若没有空位，需要在 bucket 后面挂上 overflow bucket，放入 overflow bucket。

bucket 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。冲突的解决手段是用链表法：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到对应的桶，再去遍历 bucket 中的 key。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx61pib1iaeK6CYYicjtlSl0HrycFFpgwNjSpHLP9sTiaPTrGe9icBPkycO2pbKvibTddsnjk5YrDe0VicwGjA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上图中，假定 B = 5，所以 bucket 总数就是 2^5 = 32。首先计算出待查找 key 的哈希，使用低 5 位 `00110`，找到对应的 6 号 bucket，使用高 8 位 `10010111`，对应十进制 151，在 6 号 bucket 中寻找 tophash 值（HOB hash）为 151 的 key，找到了 2 号槽位，再对 2 号槽位的 key 做哈希，如果与待查找 key 的哈希相等，这样整个查找过程就结束了。

如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket。如果 hmap 中没有此 key，那就会返回一个 key 相应类型的零值。

遍历 bucket 和它所有的 overflow bucket，相当于遍历一个 bucket 链表。而里层循环就是遍历这个 bucket 里所有的 cell，或者说所有的槽位。

#### 扩容

使用哈希表的目的就是要快速查找到目标 key，然而，随着向 map 中添加的 key 越来越多，key 发生碰撞的概率也越来越大。bucket 中的 8 个 cell 会被逐渐塞满，查找、插入、删除 key 的效率也会越来越低。最理想的情况是很多 bucket，一个 bucket 只装一个 key，这样，就能达到 `O(1)` 的效率，但这样空间消耗太大，用空间换时间的代价太高。

Go 语言采用一个 bucket 里装载 8 个 key，定位到某个 bucket 后，还需要再定位到具体的 key，这实际上又用了时间换空间。当然，这样做要有一个度，不然所有的 key 都落在了同一个 bucket 里，直接退化成了链表，各种操作的效率直接降为 O(n)。因此，需要有一个指标来衡量前面描述的情况，这就是装载因子。Go 源码里这样定义装载因子：

```go
loadFactor := count / (2^B)
```

count 就是 map 的元素个数，2^B 表示 bucket 数量。

再来说**触发 map 扩容的时机**。在向 map 插入新 key 的时候，会进行条件检测，符合下面这 2 个条件，就会触发扩容：

1. 装载因子超过阈值，源码里定义的阈值是 6.5。
2. overflow 的 bucket 数量过多：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15。

第 1 点：我们知道，每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满了的情况下，装载因子算出来的结果是 8。因此当装载因子超过 6.5 时，表明很多 bucket 都快要装满了，查找效率和插入效率都变低了。在这个时候进行扩容是有必要的。

第 2 点：是对第 1 点的补充。就是说在装载因子比较小的情况下，这时候 map 的查找和插入效率也很低，而第 1 点识别不出来这种情况。表面现象就是计算装载因子的分子比较小，即 map 里元素总数少，但是 bucket 数量多（真实分配的 bucket 数量多，包括大量的 overflow bucket）。

造成这种情况的原因是不停地插入、删除元素。先插入很多元素，导致创建了很多 bucket，但是装载因子达不到第 1 点的临界值，未触发扩容来缓解这种情况。之后，删除元素降低元素总数量，再插入很多元素，导致创建很多的 overflow bucket，但就是不会触犯第 1 点的规定。overflow bucket 数量太多，导致 key 会很分散，查找插入效率低得吓人，因此有第 2 点规定。这就像是一座空城，房子很多，但是住户很少，都分散了，找起人来很困难。

对于命中条件 1，2 的限制，都会发生扩容。但是扩容的策略并不相同，毕竟两种条件应对的场景不同。

对于条件 1，元素太多，而 bucket 数量太少，很简单：将 B 加 1，bucket 最大数量（2^B）直接变成原来 bucket 数量的 2 倍。于是，就有新老 bucket 了。注意，这时候元素都在老 bucket 里，还没迁移到新的 bucket 来，只是新 bucket 最大数量变为原来最大数量的 2 倍。

对于条件 2，其实元素没那么多，但是 overflow bucket 数特别多，说明很多 bucket 都没装满。解决办法就是开辟一个新 bucket 空间，将老 bucket 中的元素移动到新 bucket，使得同一个 bucket 中的 key 排列地更紧密。这样，原来在 overflow bucket 中的 key 可以移动到 bucket 中来。结果是节省空间，提高 bucket 利用率，map 的查找和插入效率自然就会提升。

再来看**扩容的具体实现**。由于 map 扩容需要将原有的 key/value 重新搬迁到新的内存地址，如果有大量的 key/value 需要搬迁，会非常影响性能。因此 Go map 的扩容采取了一种称为“渐进式”的方式，原有的 key 并不会一次性搬迁完毕，每次最多只会搬迁 2 个 bucket。所以当检测符合扩容条件，只是分配新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。在后续每次插入、修改或删除操作时，在执行具体操作前，都会尝试进行搬迁，即检查如果 oldbuckets 不为空，说明还没有搬迁完毕，就开始搬，然后再执行插入、修改或删除操作。

在执行搬迁时，遍历此 bucket 的所有的 cell，将有值的 cell copy 到新的地方。bucket 还会链接 overflow bucket，它们同样需要搬迁。

对于条件 1，要重新计算 key 的哈希，才能决定它到底落在哪个 bucket。例如，原来 B = 5，计算出 key 的哈希后，只用看它的低 5 位，就能决定它落在哪个 bucket。扩容后，B 变成了 6，因此需要多看一位，它的低 6 位决定 key 落在哪个 bucket。这称为 `rehash`。

对于条件 2，从老的 buckets 搬迁到新的 buckets，由于 bucktes 数量不变，因此可以按序号来搬，比如原来在 0 号 bucktes，到新的地方后，仍然放在 0 号 buckets。

因此，某个 key 在搬迁前后 bucket 序号可能和原来相等，也可能是相比原来加上 2^B（原来的 B 值），取决于 hash 值 第 6 bit 位是 0 还是 1。

这也解释了为什么**遍历 map 是无序的**。因为 map 在扩容后，会发生 key 的搬迁，而遍历的过程，就是按顺序遍历 bucket，同时按顺序遍历 bucket 中的 key。

当然，如果我就一个硬编码的 map，我也不会向 map 进行插入删除的操作，按理说每次遍历这样的 map 都会返回一个固定顺序的 key/value 序列。但是 Go 杜绝了这种做法，因为这样会给新手程序员带来误解，以为这是一定会发生的事情，在某些情况下，可能会酿成大错。所以 Go 在我们在遍历 map 时，并不是固定地从 0 号 bucket 开始遍历，每次都是从一个随机值序号的 bucket 开始遍历，并且是从这个 bucket 的一个随机序号的 cell 开始遍历。这样，即使你是一个写死的 map，仅仅只是遍历它，也不太可能会返回一个固定序列的 key/value 对了。“迭代 map 的结果是无序的”这个特性是从 go 1.0 开始加入的。

#### map 的遍历

因为有扩容的过程，而扩容不是一个原子的操作，所以在很长时间里，map 的状态都是处于一个中间态：有些 bucket 已经搬迁到新家，而有些 bucket 还待在老地方。因此，map 的遍历并不是简单地遍历所有的 bucket 以及它后面挂的 overflow bucket，然后挨个遍历 bucket 中的所有 cell，从有 key 的 cell 中取出 key 和 value。

遍历如果发生在扩容的过程中，就会涉及到遍历新老 bucket 的过程，这时需判断当前 bucket 是否已经搬迁。如已搬迁，则遍历新 bucket  里的数据。否则，遍历旧 bucket 里在扩容裂变后将会分配到新 bucket 中的数据。

所以 map 遍历的核心在于理解 2 倍扩容时，老 bucket 会分裂到 2 个新 bucket 中去。而遍历操作，会按照新 bucket 的序号顺序进行，碰到老 bucket 未搬迁的情况时，要在老 bucket 中找到将来要搬迁到新 bucket 来的 key。

#### map 的赋值

关键在于定位 key 要安置的地址。先对 key 做哈希定位到所在 bucket，然后找到 key 安置在整个 bucket 中的位置。在找 key 在整个 bucket 中的位置时，如果遍历 bucket 没有找到此 key，意味着是插入新 key，那 key 的安置地址就是第一次发现的“空位”，如果这个 bucket 的 key 都已经放置满了，需要在 bucket 后面挂上 overflow bucket。如果遍历 bucket 找到了此 key，意味着是更新旧 key，那 key 的安置地址就是 key 现有位置。

#### map 的删除

核心还是找到 key 的具体位置。找到对应位置后，对 key 和 value 进行“清零”操作：

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

#### map 创建的底层原理

实际底层调用的是 makemap 函数，主要做的工作就是初始化 hmap 结构体的各种字段，例如计算 B 的大小，设置哈希种子 hash0 等。

```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap
```

注意，这个函数返回的结果是 *hmap，是一个指针，而创建切片底层调用的 makeslice 函数返回的是 slice 结构体：

```go
func makeslice(et *_type, len, cap int) slice
```

slice 的结构体定义，结构体内部包含底层的数据指针。

```go
// runtime/slice.go
type slice struct {
    array unsafe.Pointer // 元素指针
    len   int   // 长度
    cap   int   // 容量
}
```

makemap 和 makeslice 的区别，带来一个不同点：当 map 和 slice 作为函数参数时，在函数参数内部对 map 的操作会影响 map 自身，而对 slice 却不会。原因是一个是指针（*hmap），一个是结构体（slice）。

#### nil map 和空 map

**nil map**

```go
var month map[string]int	// 声明了一个 nil map
fmt.Println(month == nil)   // 输出：true
```

map 的零值是 nil。

nil map 是不能存取键值对的，会报 panic 错误。

**空 map**

```go
// month := make(map[string]int)	// make 函数创建空 map
month := map[string]int{}		// 字面量创建空 map
fmt.Println(month)        		// 输出：map[]
```

### map 的两种 get 操作

Go 语言中读取 map 有两种语法。

```go
value := map[key]	// 当key不存在，返回value类型的零值
```

```go
value, ok := map[key]	// 增加bool型变量返回，提示key是否存在
```

这两种语法的实现是编译器在背后做的工作：分析代码后，将两种语法对应到底层两个不同的函数。

```go
// src/runtime/hashmap.go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer
func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)
```

根据 key 的不同类型，编译器还会将查找、插入、删除的函数用更具体的函数替换，以优化效率。

### 其他

* 使用 len 函数返回 map 中键值对的数量。
* 可以通过内建函数`delete(map, key)` 删除键值对。当 map 不存在的相应的 key 时，不报错，相当于没操作。
* 要得到一个不共享底层数据的 map 副本，可以创建新的空 map，迭代旧 map 给新 map 赋值。
* map 不是并发安全的，并发操作会导致 panic，要加上读写锁。也可以使用 sync 包下并发安全的 map。
* 定义 map 时，当 value 是自定义结构体时：
  1. 不推荐`map[string]Student`，map 的 value Student 的属性是不可以修改的。
  2. 推荐`map[string]*Student`，map 的 value Student 的属性是可以修改的，且效率高。
* map 的遍历是乱序的，要顺序输出，可使用 slice 额外保存顺序的 key，再通过 slice 去读取。

### 并发安全 map

sync 包下的 map 是并发安全的。此外，可以自定义加锁的map结构。

[《谈Go语言中并发Map的使用》](https://blog.csdn.net/u014029783/article/details/100204075)

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

使用 map 作为集合的底层数据结构的好处在于，map 基于 hash 表实现，键值查找效率高，还可以少写很多代码。

