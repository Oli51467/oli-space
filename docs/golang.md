## context

参考文章&视频：[Bilibili-小徐先生1212-解说Golang context实现原理](https://www.bilibili.com/video/BV1EA41127Q3/)

[Golang context 实现原理](https://mp.weixin.qq.com/s/AavRL-xezwsiQLQ1OpLKmA)

[Go标准库Context | 李文周的博客 (liwenzhou.com)](https://www.liwenzhou.com/posts/Go/context/)

对于一个调用链路，context可以很好地做到多个goroutine之间的信息传递、信息共享，可以实现取消信号、截止时间等操作。

### Context接口

`context.Context`是一个接口，该接口定义了四个需要实现的方法。

Context是线程安全的，可以放心的在多个goroutine中传递

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)	// 当前Context被取消的时间	
    Done() <-chan struct{}								// 这个Channel会在当前工作完成或者上下文被取消之后关闭，多次调用Done方法会返回同一个Channel
    Err() error
    Value(key interface{}) interface{}
}
```

其中：

- `Deadline`方法需要返回当前`Context`被取消的时间，也就是完成工作的截止时间（deadline）
- `Done`方法需要返回一个`Channel`，这个Channel会在当前工作完成或者上下文被取消之后关闭，多次调用`Done`方法会返回同一个Channel；
- ```Err```方法会返回当前```Context```结束的原因，它只会在```Done```返回的Channel被关闭时才会返回非空的值:
  - 如果当前`Context`被取消就会返回`Canceled`错误
  - 如果当前`Context`超时就会返回`DeadlineExceeded`错误
- `Value`方法会从`Context`中返回键对应的值，对于同一个上下文来说，多次调用`Value` 并传入相同的`Key`会返回相同的结果，该方法仅用于传递跨API和进程间跟请求域的数据

Go内置两个函数：`Background()`和`TODO()`，这两个函数分别返回一个实现了`Context`接口的`background`和`todo`。这两个内置的上下文对象一般是作为最顶层的`partent context`，衍生出更多的子上下文对象。

`background`和`todo`本质上都是`emptyCtx`结构体类型，是一个**不可取消**，**没有设置截止时间**，**没有携带任何值**的Context。它与空结构体不同，他有明确且独一无二的地址。

```go
// An emptyCtx is never canceled, has no values, and has no deadline. It is not
// struct{}, since vars of this type must have distinct addresses.
type emptyCtx int

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

// Background returns a non-nil, empty Context. It is never canceled, has no
// values, and has no deadline. It is typically used by the main function,
// initialization, and tests, and as the top-level Context for incoming
// requests.
func Background() Context {
	return background
}
```

### With函数

#### WithCancel

```func WithCancel(parent Context) (ctx Context, cancel CancelFunc)```

`WithCancel`返回带有新Done通道的父节点的**副本**。当调用返回的cancel函数或当关闭父上下文的Done通道时，将关闭返回上下文的Done通道，无论先发生什么情况。

#### WithDeadline

```func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)```

返回父上下文的**副本**，并将deadline调整为不迟于d。如果父上下文的deadline已经早于d，则WithDeadline等同于父上下文。

当截止日过期时，或当调用返回的cancel函数时，或者当父上下文的Done通道关闭时，返回上下文的Done通道将被关闭，以最先发生的情况为准。

```go
func main() {
	d := time.Now().Add(50 * time.Millisecond)	// 定义了一个50毫秒之后过期的deadline
	ctx, cancel := context.WithDeadline(context.Background(), d)

	// 尽管ctx会过期，但在任何情况下调用它的cancel函数都是很好的实践。
	// 如果不这样做，可能会使上下文及其父类存活的时间超过必要的时间。
	defer cancel()

	// 等待1秒后打印overslept退出或者等待ctx过期后退出。
	select {
	case <-time.After(1 * time.Second):
		fmt.Println("overslept")
	case <-ctx.Done():
		fmt.Println(ctx.Err())	// 因为ctx 50毫秒后就会过期，所以ctx.Done()会先接收到context到期通知，并且会打印ctx.Err()的内容。
	}
}
```

#### WithTimeout

```func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)```

`WithTimeout`返回`WithDeadline(parent, time.Now().Add(timeout))`

#### WithValue

`WithValue`函数能够将请求作用域的数据与 Context 对象建立关系

`func WithValue(parent Context, key, val interface{}) Context`

所提供的键必须是可比较的，并且不应该是`string`类型或任何其他内置类型

## 数据结构

### string

在64位机器下，所有字符串的长度16个字节

```go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
type StringHeader struct {
	Data uintptr // 指向存储字符串的字节数组
	Len  int		 // 表示字节数组的长度而不是字符个数 
}
```

在UTF-8编码下，可能三个Byte表示一个汉字，也可能一个Byte表示一个字母

#### 字符串的切分

```s = string([]rune(s)[:3])```

### slice

#### 楔子

##### 示例1:

```go
func Test_slice(t *testing.T) {
	s := make([]int, 10)	// 长度为10，容量也为10
	s = append(s, 10)
	t.Logf("s: %v, len of s: %d, cap of s: %d", s, len(s), cap(s))
}
```

运行结果：

```go
=== RUN   Test_slice
    slice_test.go:8: s: [0 0 0 0 0 0 0 0 0 0 10], len of s: 11, cap of s: 20
--- PASS: Test_slice (0.00s)
PASS
```

##### 示例2:

```go
func Test_slice1(t *testing.T) {
	s := make([]int, 0, 10)	// 长度为0，容量为10
	s = append(s, 10)
	t.Logf("s: %v, len of s: %d, cap of s: %d", s, len(s), cap(s))
}
```

运行结果：

```go
=== RUN   Test_slice1
    slice_test.go:14: s: [10], len of s: 1, cap of s: 10
--- PASS: Test_slice1 (0.00s)
PASS
```

##### 示例3:

```go
func Test_slice2(t *testing.T) {
	s := make([]int, 10, 11)	// 长度为10，容量为11
	s = append(s, 10)
	t.Logf("s: %v, len of s: %d, cap of s: %d", s, len(s), cap(s))
}
```

运行结果：

```go
=== RUN   Test_slice2
    slice_test.go:20: s: [0 0 0 0 0 0 0 0 0 0 10], len of s: 11, cap of s: 11
--- PASS: Test_slice2 (0.00s)
PASS
```

##### 示例4:

```go
func Test_slice4(t *testing.T) {
	s := make([]int, 10, 12)	// s长度为10，容量为12
	s1 := s[8:]								// 截取切片s index=8往后的内容赋给s1
	t.Logf("s1: %v, len of s1: %d, cap of s1: %d", s1, len(s1), cap(s1))
}
```

运行结果：

```go
=== RUN   Test_slice4
    slice_test.go:26: s1: [0 0], len of s1: 2, cap of s1: 4
--- PASS: Test_slice4 (0.00s)
PASS
```

截取所得新切片 s1 的长度和容量强依赖于原切片 s 的长度和容量，并在此基础上减去头部 8 个未使用到的单位。

##### 示例5:

```go
func Test_slice5(t *testing.T) {
	s := make([]int, 10, 12) // s的长度为10，容量为12
	s1 := s[8:9]             // 截取切片s index为[8, 9)范围内的元素赋给切片s1
	t.Logf("s1: %v, len of s1: %d, cap of s1: %d", s1, len(s1), cap(s1))
}
```

运行结果：

```go
=== RUN   Test_slice5
    slice_test.go:32: s1: [0], len of s1: 1, cap of s1: 4
--- PASS: Test_slice5 (0.00s)
PASS
```

##### 示例6:

```go
func Test_slice6(t *testing.T) {
	s := make([]int, 10, 12) // s的长度为10，容量为12
	s1 := s[8:]              // 截取切片s index=8往后的内容赋给s1
	s1[0] = -1							 // 改变s1是否会影响到s
	t.Logf("s: %v", s)
}
```

运行结果：

```go
=== RUN   Test_slice6
    slice_test.go:39: s: [0 0 0 0 0 0 0 0 -1 0]
--- PASS: Test_slice6 (0.00s)
PASS
```

s1 是在 s 基础上截取得到的，属于一次引用传递，底层共用同一片内存空间，其中 s[x] 等价于 s1[x+8]. 因此修改了 s1[0] 会直接影响到 s[8]。

##### 示例7:

```go
func Test_slice7(t *testing.T) {
	s := make([]int, 10, 12) // s的长度为10，容量为12
	v := s[10]               // 会越界
	t.Logf("v: %v", v)
}
```

##### 示例8:

```go
func Test_slice8(t *testing.T) {
	s := make([]int, 10, 12) // s的长度为10，容量为12
	s1 := s[8:]
	s1 = append(s1, []int{10, 11, 12}...)
	v := s[10] 							// 会越界
	t.Logf("v: %v", v)
}
```

- 在 s 的基础上截取产生了 s1，此时 s1 和 s 复用的是同一份内存地址
- 接下来执行 append 操作时，由于 s 预留的空间不足，s1 会发生扩容
- s1 扩容后，会被迁移到新的空间地址，此时 s1 已经和 s 做到真正意义上的完全独立，意味着修改 s1 不再会影响到 s
- s 继续维持原本的长度值 10 和容量值 12，因此访问 s[10] 会panic

#####  示例8.1

```go
func Test_slice8-1(t *testing.T) {
	s := make([]int, 10, 12) // s的长度为10，容量为12
	s1 := s[8:]
	s1 = append(s1, 11)
	v := s[10] 							 // 仍然会越界
}
```

s1在追加后只会改变s1对应示例的slice的len字段，而这两个slice实例背后对应的存储数据的内存数据的地址是相同的。所以在```s1 = append(s1, 11)```后s1的len是3，cap是4；s的len仍然是10，cap是12，虽然`s[10]`的位置上已经被设置为11，但访问的位置小于len，故仍然会panic。

##### 示例9:

```go
func Test_slice9(t *testing.T) {
	s := make([]int, 10, 12) // s的长度为10，容量为12
	s1 := s[8:]				 			 // 截取切片s index=8往后的内容赋给s1
	changeSlice(s1)
	t.Logf("s: %v", s)
}

func changeSlice(s1 []int) {
	s1[0] = -1
}
```

运行结果：

```go
=== RUN   Test_slice9
    slice_test.go:60: s: [0 0 0 0 0 0 0 0 -1 0]
--- PASS: Test_slice9 (0.00s)
PASS
```

##### 示例10:

```go
func Test_slice10(t *testing.T) {
	s := make([]int, 10, 12) // s的长度为10，容量为12
	s1 := s[8:]              // 截取切片s index=8往后的内容赋给s1
	changeSlice1(s1)		     // 在方法中对s1进行append追加操作
	t.Logf("s: %v, len of s: %d, cap of s: %d", s, len(s), cap(s))
	t.Logf("s1: %v, len of s1: %d, cap of s1: %d", s1, len(s1), cap(s1))
}

func changeSlice1(s1 []int) {
	s1 = append(s1, 10)
}
```

运行结果：

```go
=== RUN   Test_slice10
    slice_test.go:71: s: [0 0 0 0 0 0 0 0 0 0], len of s: 10, cap of s: 12
    slice_test.go:72: s1: [0 0], len of s1: 2, cap of s1: 4
--- PASS: Test_slice10 (0.00s)
PASS
```

虽然切片是引用传递，但是在方法调用时，传递的会是一个新的 slice。

因此在局部方法 changeSlice1 中，虽然对 s1 进行了 append 操作，但这这会在局部方法中这个独立的 slice 中生效，不会影响到原方法 Test_slice10 当中的 s 和 s1 的长度和容量。

##### 示例11:

```go
func Test_slice11(t *testing.T) {
	s := []int{0, 1, 2, 3, 4}
	s = append(s[:2], s[3:]...) // 截取s中index=2前面的内容，并在此基础上追加index=3后面的内容
	t.Logf("s: %v, len of s: %d, cap of s: %d", s, len(s), cap(s))
	v := s[4]
	t.Logf("v: %v", v)	// 会越界
}
```

运行结果：

```go
=== RUN   Test_slice11
    slice_test.go:82: s: [0 1 3 4], len of s: 4, cap of s: 5
--- FAIL: Test_slice11 (0.00s)
```

##### 示例12:

```go
func Test_slice12(t *testing.T) {
	s := make([]int, 512)
	s = append(s, 1)
	t.Logf("len of s: %d, cap of s: %d", len(s), cap(s))
}
```

运行结果：

```go
=== RUN   Test_slice12
    slice_test.go:90: len of s: 513, cap of s: 1024
--- PASS: Test_slice12 (0.00s)
PASS
```

不同golang版本的运行结果不同。

在64位机器下，所有切片的长度都是24字节（每个字段占8字节）。

**切片实质是对底层数组的引用**

```go
type slice struct {
	array unsafe.Pointer	// 指向连续内存空间地址的起点
	len   int							// 逻辑意义上slice中实际存放了多少个元素
	cap   int							// 实际存储数据的字节数组长度
}
```

由于在slice中存放了内存空间的地址，在传递slice切片的时候，虽然进行的是值拷贝，但内部存放的地址是相同的，因此**对于slice本身属于引用传递**。因此在局部方法中如果触发了slice的扩容，会影响拷贝出的副本的len和cap字段，但是不会对外部的slice中的len和cap字段产生影响

#### 初始化

1. 通过字面量创建，在编译时创建一个数组，再创建一个slice，将数组的引用传进去

   ```go
   arr := [3]int{1,2,3}
   // 内部调用
   slice {
     arr, 3,3
   }
   ```

2. make：运行时创建数组

   ```go
   slice := make([]int, 10)
   ```

   调用```runtime.makeslice()```方法，返回切片的指针

   ```go
   func makeslice(et *_type, len, cap int) unsafe.Pointer {
       // 结合每个元素的大小以及切片的容量，计算出初始化切片所需要的内存空间大小
       mem, overflow := math.MulUintptr(et.size, uintptr(cap))
       if overflow || mem > maxAlloc || len < 0 || len > cap {
           // 倘若容量超限，len 取负值或者 len 超过 cap，直接 panic
           mem, overflow := math.MulUintptr(et.size, uintptr(len))
           if overflow || mem > maxAlloc || len < 0 {
               panicmakeslicelen()
           }
           panicmakeslicecap()
       }
       // 调用位于 runtime/malloc.go 文件中的 mallocgc 方法，为切片进行内存空间的分配
       return mallocgc(mem, et, true)
   }
   ```

3. 通过数组创建

   ```go
   arr := [5]int{0,1,2,3,4}	// 此时slice的长度和容量均为5
   slice := arr[1:4]
   ```

   创建后将0忽略，cap=4，len=3

#### 截取元素

对切片截取时，本质是引用传递操作，不论如何截取，底层复用的都是同一块内存空间的数据，但截取动作会创建出一个新的slice示例

#### 追加元素

1. 添加后的len小于容量cap时，编译时只调整len，并将追加的数据添加到数组后面

2. 添加后的len大于容量cap时，编译时调用```runtime.growslice()```，**原来的数组不要了，新开一个数组**。因为数组在内存中必须连续，原来的数组后面的内存地址可能会越界。

3. 扩容时（v1.19）：

   1. 倘若切片元素大小为 0（元素类型为 struct{}），则直接复用一个全局的 zerobase 实例，直接返回
   2. 如果期望容量大于当前容量的两倍就会使用期望容量。
   3. 否则，如果当前切片长度小于256，则直接将容量翻倍。
   4. 如果当前切片长度大于等于256，则在当前容量的基础上扩容1/4的比例并且累加192的数值，如此循环直到得到的新容量大于等于预期容量为止。
   5. 结合 mallocgc 流程中，对内存分配单元 mspan 的等级制度，推算得到实际需要申请的内存空间大小。
   6. 调用 mallocgc，对新切片进行内存初始化。
   7. 调用 memmove 方法，将老切片中的内容拷贝到新切片中。
   8. 返回扩容后的新切片。

   切片在扩容时，**并发不安全**，需要加锁。

```go
func growslice(et *_type, old slice, cap int) slice {
	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.cap < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
}
```

#### 删除元素

本质是内容截取

如果需要删除 slice 中间的某个元素，操作思路则是采用内容截取加上元素追加的复合操作，可以先截取待删除元素的左侧部分内容，然后在此基础上追加上待删除元素后侧部分的内容：

```go
func Test_slice(t *testing.T){
    s := []int{0,1,2,3,4}
    // 删除 index = 2 的元素
    s = append(s[:2],s[3:]...)
    // s: [0,1,3,4], len: 4, cap: 5
    t.Logf("s: %v, len: %d, cap: %d", s, len(s), cap(s))
}
```

当我们需要删除 slice 中的所有元素时，也可以采用切片内容截取的操作方式：s[:0]. 这样操作后，slice header 中的指针 array 仍指向远处，但是逻辑意义上其长度 len 已经等于 0，而容量 cap 则仍保留为原值。

```go
func Test_slice(t *testing.T){
    s := []int{0,1,2,3,4}
    s = s[:0]
    // s: [], len: 0, cap: 5
    t.Logf("s: %v, len: %d, cap: %d", s, len(s), cap(s))
}
```

#### 切片拷贝

##### 浅拷贝

对切片的字面量进行赋值传递即可，这样相当于创建出了一个新的 slice 实例，但是其中的指针 array、容量 cap 和长度 len 仍和老的 slice实例相同。

```go
func Test_slice(t *testing.T) {
    s := []int{0, 1, 2, 3, 4}
    s1 := s
}
```

##### 深拷贝

创建出一个和 slice**容量大小相等**的独立的内存区域，并将原 slice 中的元素一一拷贝到新空间中。可以调用系统方法 copy。

```go
func Test_slice(t *testing.T) {
    s := []int{0, 1, 2, 3, 4}
    s1 := make([]int, len(s))
    copy(s1, s)
}
```

### runtime.hmap

```go
// A header for a Go map.
type hmap struct {
	count     int 	 // live cells == size of map.
	flags     uint8
	B         uint8  // log_2 of buckets (can hold up to loadFactor * 2^B items)
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
	hash0     uint32 // hash seed

	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0. 指向bmap数组的指针
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
	extra *mapextra 
}

type mapextra struct {
	// If both key and elem do not contain pointers and are inline, then we mark bucket
	// type as containing no pointers. This avoids scanning such maps.
	// However, bmap.overflow is a pointer. In order to keep overflow buckets
	// alive, we store pointers to all overflow buckets in hmap.extra.overflow and hmap.extra.oldoverflow.
	// overflow and oldoverflow are only used if key and elem do not contain pointers.
	// overflow contains overflow buckets for hmap.buckets.
	// oldoverflow contains overflow buckets for hmap.oldbuckets.
	// The indirection allows to store a pointer to the slice in hiter.
  // 指向溢出桶的数组
	overflow    *[]*bmap
	oldoverflow *[]*bmap

	// 下一个可用的溢出桶的地址
	nextOverflow *bmap
}

bucketCntBits = 3
bucketCnt = 1 << bucketCntBits

// A bucket for a Go map.
type bmap struct {
	// tophash存放每个key的hash的最高字节
  // If tophash[0] < minTopHash, 则tophash[0]指示桶疏散状态
  // bucketCnt = 8， 一个桶只能放8组数据
	tophash [bucketCnt]uint8
  // Followed by bucketCnt keys and then bucketCnt elems.
  // 因为keys和elems的类型不确定，所以没写死，会在编译时放入
	// keys和elems分别为key的数组和value的数组
	// Followed by an overflow pointer. 指向溢出桶的指针
}
```

#### hmap的初始化

1. make

   编译时会调用```map.makemap()```，会根据传入的预估map的大小计算B并创建一些桶和溢出桶

2. 字面量

   元素少于25个时，编译时同样会调用```map.makemap()```，并赋值

   元素多于25个时，编译新建一个k数组和v数组，循环赋值

#### hmap的访问

![](../imgs/golang-map-get.png)

#### hmap的扩容

map溢出桶挂的太多时，退化为链表，性能严重下降。**在追加时判断是否扩容。**

```go
func mapassign() {
  ...
  // 装载因子超过6.5（平均每个槽6.5个key） 或溢出桶的数量超过了普通桶
	if !h.growing() && (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
}
```

##### 等量扩容

由于删除操作，导致溢出桶的数量太多，但实际数据没有那么多，溢出桶的数据从而非常稀疏，实质是数据整理。

##### 翻倍扩容

实际数据太多，导致负载因子太大。

##### 扩容的步骤

1. 创建一组新桶承载新的数据。令```hmap.oldbuckets```指针指向原有的桶数组、令```hmap.buckets```指向新的桶数组，标记```hmap.flags```为扩容状态，更新```hmap.extra```的信息

2. 采用渐进式驱逐的方案，每次操作一个旧桶时，将旧桶的数据驱逐到新桶。在读取时不进行驱逐，只判断读取新桶还是旧桶。

   > hmap.B指示了取hash操作时观察hash的最后B位作为桶号，假设扩容前B是2，扩容后B是3，对key为‘a’哈希后最后3位若为110，则去6号桶，若为010，则去2号桶

3. 所有oldbuckets驱逐完成后，将oldbuckets的内存回收。

#### hmap的并发问题

如果一个map**在扩容中**，A协程在读取某个key的数据时，另一个协程B在修改key对应的数据，操作完后，将旧桶的数据驱逐到了新桶。则A协程会读到错误的数据或者找不到数据。

### sync.Map

```go
// Map is like a Go map[interface{}]interface{} but is safe for concurrent use
// by multiple goroutines without additional locking or coordination.
// Loads, stores, and deletes run in amortized constant time.
//
// The Map type is specialized. Most code should use a plain Go map instead,
// with separate locking or coordination, for better type safety and to make it
// easier to maintain other invariants along with the map content.
//
// The Map type is optimized for two common use cases: (1) when the entry for a given
// key is only ever written once but read many times, as in caches that only grow,
// or (2) when multiple goroutines read, write, and overwrite entries for disjoint
// sets of keys. In these two cases, use of a Map may significantly reduce lock
// contention compared to a Go map paired with a separate Mutex or RWMutex.
//
// The zero Map is empty and ready for use. A Map must not be copied after first use.
type Map struct {
	mu Mutex

	// read contains the portion of the map's contents that are safe for
	// concurrent access (with or without mu held).
	//
	// The read field itself is always safe to load, but must only be stored with
	// mu held.
	//
	// Entries stored in read may be updated concurrently without mu, but updating
	// a previously-expunged entry requires that the entry be copied to the dirty
	// map and unexpunged with mu held.
	read atomic.Value // readOnly

	// dirty contains the portion of the map's contents that require mu to be
	// held. To ensure that the dirty map can be promoted to the read map quickly,
	// it also includes all of the non-expunged entries in the read map.
	//
	// Expunged entries are not stored in the dirty map. An expunged entry in the
	// clean map must be unexpunged and added to the dirty map before a new value
	// can be stored to it.
	//
	// If the dirty map is nil, the next write to the map will initialize it by
	// making a shallow copy of the clean map, omitting stale entries.
	dirty map[interface{}]*entry

	// misses counts the number of loads since the read map was last updated that
	// needed to lock mu to determine whether the key was present.
	//
	// Once enough misses have occurred to cover the cost of copying the dirty
	// map, the dirty map will be promoted to the read map (in the unamended
	// state) and the next store to the map will make a new dirty copy.
	misses int
}

// readOnly 是一个以原子方式存储在 Map.read 字段中的不可变结构。
type readOnly struct {
	m       map[interface{}]*entry
	amended bool // true if the dirty map contains some key not in m.
}

// An entry is a slot in the map corresponding to a particular key.
type entry struct {
	// p points to the interface{} value stored for the entry.
	//
	// If p == nil, the entry has been deleted, and either m.dirty == nil or
	// m.dirty[key] is e.
	//
	// If p == expunged, the entry has been deleted, m.dirty != nil, and the entry
	// is missing from m.dirty.
	//
	// Otherwise, the entry is valid and recorded in m.read.m[key] and, if m.dirty
	// != nil, in m.dirty[key].
	//
	// An entry can be deleted by atomic replacement with nil: when m.dirty is
	// next created, it will atomically replace nil with expunged and leave
	// m.dirty[key] unset.
	//
	// An entry's associated value can be updated by atomic replacement, provided
	// p != expunged. If p == expunged, an entry's associated value can be updated
	// only after first setting m.dirty[key] = e so that lookups using the dirty
	// map find the entry.
	p unsafe.Pointer // *interface{} 可以指向任何数据
}
```

![](../imgs/golang-sync-map.png)

#### 优势

sync.Map使用了readMap和dirtyMap，分离了可能引发扩容的操作：不会引发扩容的操作（查、改）使用readMap，可能引发扩容的操作（新增）使用dirtyMap。sync.Map中的锁mu是锁dirtyMap的，**只能有一个协程同时访问dirtyMap**。

#### 追加操作

![](../imgs/golang-sync-map-append.png)

#### 追加后读写

追加后读时，如果在readMap中没获取到key，查看```amended```字段是否为true，如果是，则去dirtyMap中读，如果在dirtyMap中读到，将```sync.Map.misses +=1```

#### dirty提升

当```misses=len(dirty)```时，说明dirty在被频繁读到了不可忍受的地步。此时将readMap指向dirtyMap，dirtyMap变为nil。**再次追加时才会重建dirtyMap为readMap**，回到初始状态

![](../imgs/golang-sync-map-dirty-promote.png)

#### 删除操作

##### 正常删除

直接将对应存储数据的指针置为nil

![](../imgs/golang-sync-map-delete.png)

##### 追加后删除

在追加后，dirty提升之前删除。

先查找readMap是否存在对应的key，如果不存在，查看amended字段是否为true，如果是，则到dirtyMap中查找对应的key，将存储value的指针置为nil。

与正常删除不同的是，在dirtyMap提升为readMap后，需要根据readMap重建dirtyMap，**如果发现readMap对应key的value的指针为nil，则不重建该key。并将该key在readMap中指向的值置为```expunged```**

这么做的意义是，提醒后面操作该key的协程：该key已经被删除，且没有被重建到dirtyMap中，如果该协程要删除该key的话，直接将其全部删掉而不是将value的指针置为nil。

### interface

接口值的底层表示

```go
type iface struct {
	tab  *itab
	data unsafe.Pointer		// 指向的地址才是实现某个接口的结构体本身
}

type itab struct {
	inter *interfacetype	// 接口的类型
	_type *_type					// 接口装载值的类型
	hash  uint32 
	_     [4]byte
	fun   [1]uintptr 			// variable sized. 记录了结构体实现的方法 可以做类型断言 判读是否实现了特定的方法 从而判断是不是某种接口类型
}
```

空接口底层表示：

```go
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

为什么空接口可以表示所有数据：编译时将数据包装成eface，将eface传进去

如果eface既没有类型也没有数据，那么这个eface对应的接口就是nil。

**空接口不一定是nil：只有数据和类型都是nil是才是nil**，例如：

```go
var a interface{}
fmt.println(a == nil) // true 空接口的零值是nil
var c = *int
a = c
fmt.println(a == nil) // false 此时a对应的eface中data为nil，type为*int
```

### nil

```var nil Type // Type must be a pointer, channel, func, interface, map, or slice type```

nil可以是空指针，或者是```channel```、```func```、```interface```、```map```、```slice```的零值，每种类型的零值各不相等。

空结构体的指针不是nil，但都是zerobase

## Goroutine

### 线程

每个进程可以有多个线程，线程使用系统分配给进程的内存，线程之间共享内存。线程的操作、切换、调度开销大。

```go
// m runtime2.go 线程的抽象
type m struct {
	g0 	 *g		// goroutine with scheduling stack 调度其他协程的协程
  curg *g   // current running goroutine
  ...
  mOS				// 针对不同操作系统记录的线程结构
}
```

### 协程

**协程运行在线程上**。通过不断切换线程上的数据实现复用线程，CPU不需要在线程间来回切换。协**程可以恢复到任意的线程上**，协程使用线程的资源，不需要等待CPU的调度。

#### 协程的本质

```go
// runtime2.go g 协程的数据结构
type g struct {
	stack stack					// 协程栈
  ...
  sched gobuf
 	atomicstatus uint32	// 协程状态
  goid int64					// 协程id
  stackguard0	uintptr				
}

// stack 堆栈地址
type stack struct {
 	lo uintptr		// 指示协程栈的低地址
  hi uintptr		// 指示协程栈的高地址
}

// gobuf 目前程序的运行现场
type gobuf struct {
	sp   uintptr	// stack pointer
	pc   uintptr	// progress counter
	...
}
```

#### 协程如何在线程上运行？

##### 1. 单线程循环 （Go 0.x）

由g0 stack执行```schedule```方法，从队列中找到一个可以执行的协程执行```execute```，处理后执行```gogo```方法：底层由汇编实现，在其中人为地在要执行的协程的协程栈顶插入一条```goexit()```方法，然后用```JMP```指令跳转到要执行协程的gobuf结构体中的pc所指的那一行，开始执行业务代码。

![](../imgs/goroutine-work1.png)

业务代码在栈中执行完毕后回退至```goexit()```方法，该方法会调用```runtime.goexit1()```

```go
// goexit1 will be invoked by g stack
func goexit1() {
	...
  mcall(goexit0)	// mcall switches from the g to the g0 stack and invokes fn(g)
}
```

```go
// goexit0 will be executed by g0 stack
func goexit0() {
	// modify the status of gp because g is no longer schedule anymore
  ...
  schedule()
}
```

##### 2. 多线程循环 （Go 1.0）

![](../imgs/goroutine-work2.png)

操作系统并不知道协程的存在，**多个线程**访问全局协程队列需要争夺锁，且协程顺序执行，无法并行

##### 3. G-M-P调度

核心思想：每次从全局队列中取的时候，一次性取多个可执行的协程（一个batch）放在本地队列中，全部执行完后再取下一个batch执行，大大减低了锁冲突的概率。

P持有一些G，每次获取G的时候不用从全局找

###### p的数据结构

```go
type p struct {
	m muintptr 					// 指示本地队列p服务的线程 back-link to associated m (nil if idle)
  // Queue of runnable goroutines. Accessed by one thread without lock
  runqhead uint32
  runqtail uint32
  runq     [256]guintptr
  runnext  guintptr   // 下一个要执行的协程的地址
}
type muintptr uintptr
```

![](../imgs/goroutine-gmp.png)

```go
// schedule where g0 invoke
func schedule() {
	...
	pp := mp.p.ptr()
	// find in local runq
  if gp == nil {
    gp, inheritTime = runqget(mp.p.ptr())
  }
  // global runq
  if gp == nil {
    lock(&sched.lock)
    gp, inheritTime = findrunnable()
    unlock(&sched.lock)
  }
  // steal work
  if gp == nil {
    stealWork()
  }
}

// runqget get a executable g from local queue
func runqget(_p_ *p) (gp *gm inheritTime bool) {
	...
  next := _p_.runnext
  if _p_.runnext.cas(next, 0) {
  	return next.ptr(), true
  }
}

// stealWork attempts to steal a runnable goroutine or timer from any P.
func stealWork() {
  ...
}
```

当新建一个协程时，随机寻找一个本地队列p，将新协程放入p的```runnext```（插队），如果本地队列都满了，则放到全局队列

##### GMP调度的问题

##### 1. 本地队列饥饿

当有一个长任务一直执行不结束时，本地队列中的协程将长时间无法执行。

解决方法：触发切换

![](../imgs/goroutine-trigger-switch.png)

##### 2. 全局队列饥饿

在解决方法一中，本地队列内部小循环，可能造成大任务一直在本地队列中轮换，导致全局队列中的任务长时间进不到本地队列中。

解决方法：定时从全局队列中取一些到本地队列中，让尽可能多的协程任务参与本地小循环。

```go
func findRunnable() {
	...
  // Check the global runnable queue once in a while to ensure fairness.
	// Otherwise two goroutines can completely occupy the local runqueue
	// by constantly respawning each other.
  // 每执行61次会从全局队列中拿一个上来
	if pp.schedtick%61 == 0 && sched.runqsize > 0 {
		lock(&sched.lock)
		gp := globrunqget(pp, 1)
		unlock(&sched.lock)
	}
}
```

##### 3. 触发切换长任务的时机

- 主动挂起 runtime.gopark()

  ```go
  // gopark 主动挂起
  func gopark() {
  	...
  	mcall(park_m)	// 切换线程栈至g0
  }
  
  func park_m(gp *g) {
  	// ...维护g的状态
  	schedule()
  }
  ```

无法主动调用，通过别的方式，比如：time.Sleep()隐式调用，gopark后协程进入waiting状态

- 系统调用完成时

​	```entersyscall()   ``` ```exitsyscall()```时会重新调用```schedule()```

##### 4. runtime.morestack()

如果一个协程永远都不主动挂起，并且永远都不进行系统调用怎么办？

> 在每一次函数跳转时，编译器会自动插入一条runtime.morestack()语句
>
> 标记抢占：go运行时监控到Goroutine运行超过10ms时，会认为该协程可能会造成其他协程饥饿。将```g .stackguard0```置为0xfffffade标记为抢占

系统在执行```morestack()```时判断是否被抢占，如果```g.stackguard0```被标记为抢占，直接回到```schedule()```

##### 5. 基于信号的抢占调度

如果永远都不调用```runtime.morestack()```怎么办？

```go
// 不主动挂起 不进行系统调用 没有函数切换
func do {
	i := 0
	for {
		i ++
	}
}

func main() {
	go do()
}
```

借助操作系统底层基于信号的通信，注册SIGURG信号的处理函数，在GC工作时，向目标线程发送信号，线程收到信号时触发```schedule()```

![](../imgs/goroutine-sig-schedule.png)

##### 协程太多的问题

1. 调度所需要的时间和资源大于任务本身时，系统会panic
2. 内存限制
3. 文件打开数限制

如何解决：

1. channel缓冲区

   ```go
   func do(i int, ch chan struct{}) {
     fmt.Println(i)
     time.Sleep(time.Second)
     <- ch
   }
   
   func main() {
     c := make(chan struct{}, 3000)
     for i:= 0; i < math.MaxInt32; i ++ {
       c <- struct{}{}
       go do(i, c)
     }
     time.Sleep(time.Hour)
   }
   ```



## Channel

声明并使用：

```go
func main() {
  ch := make(chan int, 0);
  go func() {
    <- ch
  }()
  ch <- 1
}
```

不要通过共享内存的方式通信，而是通过通信的方式来共享内存。避免协程竞争访问内存和数据冲突的问题。

#### 数据结构：

```go
type hchan struct {
	// circular buffer
  qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	elemtype *_type 				// element type
  send     uint index		  // send index
	recvx    uint   				// receive index
	recvq    waitq  				// list of recv waiters
	sendq    waitq  				// list of send waiters
  // lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock 		 mutex
  closed   uint32
}

// waitq 链表 指示等待队列中的头尾协程
type waitq struct {
	first   *sudog
	last    *sudog
}

type sudog struct {
	g *g
	next *sudog
	prev *sudog
}
```

![](../imgs/channel-buffer-queue.png)

#### channel如何发送数据

```c <-```语法糖，编译时会把```c <-```转化为```runtime.chansend()```

##### 1. 直接发送

- 发送数据前已经有G在休眠等待
- 此时缓冲区一定为空
- 从等到队列中取出一个等待接收的G，将数据直接拷贝给G的接收变量，并唤醒G

```go
func chansend() {
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
}
```

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	...
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
  // 唤醒协程G
	goready(gp)
}

// 直接将内存拷贝到等待接收的变量中
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src is on our stack, dst is a slot on another stack.

	// Once we read sg.elem out of sg, it will no longer
	// be updated if the destination's stack gets copied (shrunk).
	// So make sure that no preemption points can happen between read & use.
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	// No need for cgo write barrier checks because dst is always
	// Go memory.
	memmove(dst, src, t.size)
}
```

##### 2. 放入缓存

- 没有G在休眠等待，并且有缓存空间。
- 获取可存入的缓存地址，将数据放入缓存，维护缓存索引。

```go
func chansend() {
	...
  lock(&c.lock)
  // 判断缓冲区还有空间可用
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
    // 移动到对应内存区域
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
}
```

##### 3. 休眠等待

- 没有G在休眠等待，并且没有缓存或缓存满了
- 把自己包装为sudog，将sudog放入sendq发送等待队列，休眠等待并解锁

```go
func chansend() {
	...
	// Block on the channel. Some receiver will complete our operation for us.
	// 获取自己的协程g
	gp := getg()
	// 将自己封装为sudog
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// 填充数据
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	// 入队等待发送队列
	c.sendq.enqueue(mysg)
	// 阻塞
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
}
```

#### channel如何接收数据

``` <-c```语法糖，编译时会把```c <-```转化为```runtime.chanrecv()```

##### 1. 有等待的G，从G接收

- 接收数据前，有g1在发送队列阻塞等待
- channel没有缓存或者缓存空了
- 将数据直接从g1拷贝过来，唤醒g1

```go
func chanrecv() {
	...
  if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
}

func recv() {
  // 判断缓冲区为空
  if c.dataqsiz == 0 {
    // 直接接收
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	}
  goready(gp, skip+1)
}

// 直接拷贝
func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	// dst is on our stack or the heap, src is on another stack.
	// The channel is locked, so src will not move during this
	// operation.
	src := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}
```

##### 2. 有等待的G，从缓存接收

- 接收数据前有G在发送队列阻塞等待发送
- 且channel有缓存，缓存的数据一定比sendq中的数据来的早
- 从缓存取走一个数据，并**将sendq的队头出队，将一个休眠的G的数据放入缓存，并唤醒G**

##### 3. 接收缓存数据

- 没有G在发送队列阻塞等待，但是缓存有数据
- 从缓存中取走数据

##### 4. 阻塞接收

- 没有G在发送队列阻塞等待，且缓存也没有数据
- 将自己包装成sudo给，将自己放入接收队列，休眠

## Lock

#### Atomic

硬件CPU层面加锁机制，数据类型和操作类型有限

#### sema锁

sema锁也叫信号量锁，核心是一个uint32值，代表可并发的数量，每一个sema锁都对应一个semaRoot结构体。当sema对应内存的值为0时，被当作休眠队列使用。

```go
// Asynchronous semaphore for sync.Mutex.

// A semaRoot holds a balanced tree of sudog with distinct addresses (s.elem).
// Each of those sudog may in turn point (through s.waitlink) to a list
// of other sudogs waiting on the same address.
// The operations on the inner lists of sudogs with the same address
// are all O(1). The scanning of the top-level semaRoot list is O(log n),
// where n is the number of distinct addresses with goroutines blocked
// on them that hash to the given semaRoot.
type semaRoot struct {
	lock  mutex
	treap *sudog // root of balanced tree of unique waiters.
	nwait uint32 // Number of waiters. Read w/o the lock.
}
```

在runtime.sema中给表面上每一个uint32数字对应一个SemaRoot结构体，sema代表了可并发获取锁的协程数，如果sema>0，则获取锁时将sema减一。

如果sema初始为0或被多个协程竞争时减为0

![](../imgs/lock-sema-root.png)

获取锁：

```go
func cansemacquire(addr *uint32) bool {
	for {
		v := atomic.Load(addr)
		if v == 0 {
			return false
		}
    // 如果不是0 就减一
		if atomic.Cas(addr, v, v-1) {
			return true
		}
    // 否则将该协程入阻塞队列treap
    root := semroot(addr)
    root.queue(addr)
    // 阻塞 直到另一个协程释放锁
    goparkunlock(&root.lock)
	}
}
```

释放锁：

```go
func semrelease1(addr *uint32) {
	atomic.Xadd(addr, 1)
  root := semroot(addr)
	// 如果treap上没有等待的协程 直接返回
	if atomic.Load(&root.nwait) == 0 {
		return
	}
  // 否则从treap中取出一个阻塞的协程并唤醒
  root.dequeue(addr)
}
```

### Mutex互斥锁

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

Mutex的结构：

![](../imgs/lock-mutex.png)

#### 为什么CAS不能替代Mutex

- 可能会有ABA问题
- 多个线程竞争时，可能导致协程无法从线程中轮换，因为无法触发gopark()、抢占式调度、协作式调度

#### Mutex正常模式

有多个协程竞争Locked位时，会使用```atomic.CompareAndSwap()```进行加锁，必然有一个协程加锁成功，其余协程加锁失败。若加锁成功会继续执行业务，若加锁失败先会自旋尝试一段时间，若仍失败则尝试获取sema信号量，此时sema为0必然获取失败，则**将该协程加入SemaRoot结构体的treap中休眠等待**，并在```sync.Mutex```的```state```的```WaiterShift```变量中记录有一个休眠的协程。

```sync.Mutex```结合两种方案的使用场景，反映了面对并发环境通过持续试探逐渐由乐观转为悲观的态度。

- 自旋累计达到4次仍未取得锁时转为阻塞模式
- CPU**单核**或**仅有单个P调度器**时直接转为阻塞模式
- 当前P的执行队列中仍有待执行的G（避免因自旋影响到GMP调度效率）

##### 加锁

```go
// Lock 加锁
func (m *Mutex) Lock() {
	// 当前未上锁且锁内不存在阻塞协程，直接尝试CAS获取锁
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	// 开始尝试自选
	m.lockSlow()
}
```

```go
func (m *Mutex) lockSlow() {
	...
	for {
		// 判断这把锁仍在在锁着，并且处于正常模式，且仍然符合自旋条件
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// 进入if分支：当前锁阻塞队列有协程，但还未被唤醒，因此需要将mutexWoken置为1
      // 避免再有其他协程被唤醒和自己抢锁
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}
		new := old
		// 如果没有在饥饿模式下 可以继续尝试加锁
		if old&mutexStarving == 0 {
			new |= mutexLocked	// 先将新状态置为1
		}
    // 如果被锁并且在处于饥饿模式，直接将该协程放入SemaRoot的等待队列中
		if old&(mutexLocked|mutexStarving) != 0 {
      // 修改状态：等待数量+1
			new += 1 << mutexWaiterShift
		}
    // 如果上一轮循环判断为饥饿模式，则在下一轮循环的这个位置写入这一位
    if starving && old&mutexLocked != 0 {
			new |= mutexStarving
		}
    // 将最新的状态写到state的lock位中
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
      // 饥饿模式下 加锁成功
			if old&(mutexLocked|mutexStarving) == 0 {
				break // locked the mutex with CAS
			}
			// If we were already waiting before, queue at the front of the queue.
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}
      // 获取sema锁 记录在SemaRoot的treap中
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
      // 被唤醒后：
      // 计算是否因为自己等待时间过长进入饥饿模式
      starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
      // 如果被唤醒之前已经是饥饿模式
      if old&mutexStarving != 0 {
        // 直接退出等待队列 等待协程数减一
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
        // 判断是不是阻塞队列中最后一个出队的协程 或者等锁时间小于1ms 如果是：关闭饥饿模式
				if !starving || old>>mutexWaiterShift == 1 {
					delta -= mutexStarving
				}
        // 直接获取这把锁
				atomic.AddInt32(&m.state, delta)
				break
			}
			...
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}
}
```

##### 解锁

```go
// Unlock 解锁
func (m *Mutex) Unlock() {
	// 直接令最后一位减一：解锁
	new := atomic.AddInt32(&m.state, -mutexLocked)
	// 如果还有在该锁上阻塞的协程 尝试唤醒一个treap中的协程
	if new != 0 {
		m.unlockSlow(new)
	}
}
```

```go
func (m *Mutex) unlockSlow(new int32) {
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
			// 判断是否需要执行唤醒动作
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// Grab the right to wake someone.
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			// 唤醒
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
    // 饥饿模式
		// Starving mode: handoff mutex ownership to the next waiter, and yield
		// our time slice so that the next waiter can start to run immediately.
		// Note: mutexLocked is not set, the waiter will set it after wakeup.
		// But mutex is still considered locked if mutexStarving is set,
		// so new coming goroutines won't acquire it.
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```

#### Mutex饥饿模式

- 当前协程等待锁的时间超过1ms，切换到饥饿模式，Starving位置1。

- 在饥饿模式下，新参与竞争锁的协程不自旋，直接进入SemaRoot休眠等待。

- 被唤醒的协程直接获取锁，**与其竞争的其他协程直接进入SemaRoot休眠。**
- 没有协程在SemaRoot的队列中继续等待时，或者协程等待锁的时间小于1ms。回到正常模式。

饥饿模式可以避免大量的协程自旋，是一种公平锁。

### RWMutex读写锁

#### 一般读写锁

读写锁内部分为读锁和写锁，没有加写锁时，多个协程都可以加读锁。加了写锁时，无法加读锁，读协程排队等待，同理也无法加写锁。

#### 数据结构

```go
type RWMutex struct {
	w           Mutex  // held if there are pending writers 作为写锁
	writerSem   uint32 // semaphore for writers to wait for completing readers
	readerSem   uint32 // semaphore for readers to wait for completing writers
	readerCount int32  // number of pending readers 正在读的协程数，若为负数则为加了写锁
	readerWait  int32  // number of departing readers 写锁应该等待读协程的个数
}
const rwmutexMaxReaders = 1 << 30
```

![](../imgs/lock-rwmutex.png)

#### 如何加写锁？

- 当前没有读协程在前面等待，```readerCount=0```

  先竞争RWMutex的互斥锁w，锁上w后，设置```readerCount=-rwmutexMaxReaders```，变为负数标志着当前有写协程在排队。加锁完成。

- 前面有读协程在读取，假设```readerCount=3```

  先竞争RWMutex的互斥锁w，锁上w后，将```readerCount=3-rwmutexMaxReaders```，一是标志着当前有写协程在排队，二是标志着前面有三个读协程在读。设置```readerWait=3```，记录三个读协程释放后唤醒写协程，陷入writerSem

```go
// Lock locks rw for writing.
// If the lock is already locked for reading or writing,
// Lock blocks until the lock is available.
func (rw *RWMutex) Lock() {
	// First, resolve competition with other writers.
	rw.w.Lock()
	// Announce to readers there is a pending writer. 获取减之前的数
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	// Wait for active readers.
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
  // if r == 0, lock successfully
}
```

#### 如何解写锁？

设置```readerCount+=rwmutexMaxReaders```，变为正值，允许读锁的获取。然后释放readerSem中等待的所有读协程，解锁mutex。

```go
// Unlock unlocks rw for writing. It is a run-time error if rw is
// not locked for writing on entry to Unlock.
//
// As with Mutexes, a locked RWMutex is not associated with a particular
// goroutine. One goroutine may RLock (Lock) a RWMutex and then
// arrange for another goroutine to RUnlock (Unlock) it.
func (rw *RWMutex) Unlock() {
	// Announce to readers there is no active writer.
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	// Unblock blocked readers, if any.
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
	// Allow other writers to proceed.
	rw.w.Unlock()
}
```

#### 如何加读锁？

- ```readerCount>=0```时

说明没有写锁在我们之前，```readerCount+=1```，加锁成功。

- ```readerCount < 0```时

说明在我们之前被加了写锁，此时有两种情况，一种是```readerWait > 0```，写锁正在排队等待读锁完成。另一种是```readerWait=0```，写锁正在工作中。无论哪一种情况，先```readerCount+=1```表示新进入了一个读协程，如果readerCount是负数，说明被加了写锁，则陷入readerSem中排队。

```go
// It should not be used for recursive read locking; a blocked Lock
// call excludes new readers from acquiring the lock. See the
// documentation on the RWMutex type.
func (rw *RWMutex) RLock() {
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		// A writer is pending, wait for it.
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}
```

#### 如何解读锁？

1. ```readerCount -=1```
2. 如果```readerCount > 0```，解锁成功
3. 如果```readerCount < 0```，说明有写锁在排队；判断自己是不是readerWait的最后一个，如果是则唤醒写协程。

```go
// RUnlock undoes a single RLock call;
// it does not affect other simultaneous readers.
// It is a run-time error if rw is not locked for reading
// on entry to RUnlock.
func (rw *RWMutex) RUnlock() {
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		// Outlined slow-path to allow the fast-path to be inlined
		rw.rUnlockSlow(r)
	}
}

func (rw *RWMutex) rUnlockSlow(r int32) {
	// A writer is pending.
	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		// The last reader unblocks the writer.
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```

### WaitGroup

![](../imgs/go-lock-waitgroup.png)

#### Wait()

- 如果被等待的协程没有了：```counter=0```，直接返回不阻塞
- 否则，```waiter+=1```，将该协程陷入sema中阻塞

```go
// Wait blocks until the WaitGroup counter is zero.
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	for {
		state := atomic.LoadUint64(statep)
		counter := int32(state >> 32)
		w := uint32(state)
		if counter == 0 {
			// Counter is 0, no need to wait.
			return
		}
		// Increment waiters count.
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			runtime_Semacquire(semap)
			return
		}
	}
}
```

#### Done()

```counter-=1```，通过```Add(-1)```实现。

```go
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32)
	w := uint32(state)
	if v > 0 || w == 0 {
		return
	}
	// This goroutine has set counter to 0 when waiters > 0.
	// Now there can't be concurrent mutations of state:
	// - Adds must not happen concurrently with Wait,
	// - Wait does not increment waiters if it sees counter == 0.
	// Still do a cheap sanity check to detect WaitGroup misuse.
	// Reset waiters count to 0.
	*statep = 0
	// 被等待的协程都做完，且有人在等待，唤醒所有sema中的协程
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)
	}
}
```

### sync.Once

一段代码只能执行一次。

```go
// A Once must not be copied after first use.
type Once struct {
	done uint32		// 标识任务有无被触发过
	m    Mutex		// 保护done变量
}
```

先判断变量```done```是否已经被修改为1，如果没有，则尝试获取锁。获取到锁后协程执行业务代码，修改```don e```为1，解锁。

```go
func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	//
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the atomic.StoreUint32 must be delayed until after f returns.

	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
  // 双重检查锁
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```



## MM&GC

### 栈内存（协程栈、调用栈）

协程栈记录了协程的**执行路径、局部变量、和函数传参和返回值**。

协程栈位于堆内存上，普通方法栈底是用于协程调度回退的```goexit()```方法，main方法执行前先调用```runtime.main()```方法，```runtime.main()```再调用```main.main()```

#### 协程栈编译时分配的不够大怎么办

栈帧回收后，需要继续使用的变量或者太大的变量可能会内存逃逸到堆上。

指针逃逸：方法返回的指针在上一级方法中继续使用，该指针会逃逸到堆上。

空接口逃逸：如果函数的形参类型为interface{}，实参很可能会逃逸，因为interface{}类型的函数往往会使用反射来查看传入的值实际的类型，反射的对象要求往往是在堆上而不是栈上。

大变量逃逸：64位机器中，一般超过64KB的变量会逃逸到堆上。

调用层数过多：**栈初始空间位2KB**，在函数调用前会使用```morestack()```判断栈空间是否还充足，```golang1.13```以后使用连续栈扩容，**当空间不足时变为原来的2倍**，将原来的栈拷贝到新栈中。当空间使用率不足1/4时缩容，变为原来的1/2

### 堆内存

位于操作系统的虚拟内存上，Go进程每次申请**虚拟内存单元**【**heapArena**】为64MB，最多有```2^20```个虚拟内存单元，所有的heapArena组成了mheap（堆内存）

![](../imgs/golang-mm-heaparena.png)

#### heapArena的分配--分级分配

根据隔离适应策略，使用内存时的最小单位为mspan，每个mspan为N个相同大小的格子，Go中一共有68级span

![](../imgs/golang-heaparena-mspan.png)

### GC



## TCP网络编程

### Socket

操作系统提供了Socket作为TCP通信的抽象

通信过程：

![](../imgs/golang-web-socket.png)

IO模型指的是操作Socket的方案。

### 阻塞IO

每一个线程服务于一个客户端，当客户端没有消息时，线程会陷入内核态。当读写成功后，切换回用户态，继续执行。**内核态切换的开销是极大的。**

### 非阻塞IO

轮询所有的Socket，直到某一个Socket可以读写。不会陷入内核态，自由度高，但需要自旋。

### 多路复用

将监听多个Socket的任务从业务转移到了操作系统，由操作系统帮我们负责。将多个Socket事件注册到event poll中，当有事件发生时，操作系统会把事件列表返回。

### 结合阻塞模型和多路复用

在底层使用操作系统的多路复用IO，在协程层次使用阻塞模型，阻塞协程将协程休眠。

在Go中，network poller抽象了底层的epoll

```go
func netpollinit() {
	// 拿到epoll file descriptor
	epfd = epollcreate1(_EPOLL_CLOEXEC)
	// 新建Epoll
	if epfd < 0 {
		epfd = epollcreate(1024)
		if epfd < 0 {
			println("runtime: epollcreate failed with", -epfd)
			throw("runtime: netpollinit failed")
		}
		closeonexec(epfd)
	}
	// 新建一个linux的pipe管道用于中断Epoll
	r, w, errno := nonblockingPipe()
	ev := epollevent{
		events: _EPOLLIN,
	}
	*(**uintptr)(unsafe.Pointer(&ev.data)) = &netpollBreakRd
	// 将 “管道有数据到达” 事件注册到Epoll中
	errno = epollctl(epfd, _EPOLL_CTL_ADD, r, &ev)
	netpollBreakRd = uintptr(r)
	netpollBreakWr = uintptr(w)
}
```

```go
// 插入事件
func netpollopen(fd uintptr, pd *pollDesc) int32 {
	var ev epollevent
  // pollDesc指针是Socket相关详细信息 记录了哪个协程休眠在等待此Socket
	ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
	*(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
	return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

