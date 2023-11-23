## context

参考视频：[Bilibili-小徐先生1212-解说Golang context实现原理](https://www.bilibili.com/video/BV1EA41127Q3/)

小徐先生1212公众号文章：[Golang context 实现原理](https://mp.weixin.qq.com/s/AavRL-xezwsiQLQ1OpLKmA)



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
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
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
		if raceenabled {
			racesync(c, sg)
		}
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

#### sema锁

sema锁也叫信号量锁，核心是一个uint32值，代表可并发的数量，每一个sema锁都对应一个semaRoot结构体

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

在runtime.sema中给表面上每一个uint32数字对应一个SemaRoot结构体

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



