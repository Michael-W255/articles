# goroutine 泄漏的常见原因

核心:**goroutine 进入永久阻塞或永久运行,没有退出路径。**

1. **channel 发送阻塞** —— 没有接收方,卡在 `ch <- v`。
2. **channel 接收阻塞** —— 没有发送方(发送方提前退出或忘了关 channel),卡在 `<-ch`。
3. **context 没用** —— 没传取消信号,或 goroutine 没监听 `ctx.Done()`,父任务结束子 goroutine 还在跑。
4. **select 全分支阻塞** —— 所有 channel 都不就绪,又没有 default 或 `ctx.Done()` 退出分支。
5. **WaitGroup 计数不匹配** —— Add/Done 对不上,Wait 永不归零。
6. **无限循环无退出条件** —— `for {}` 里没有 break/return/ctx 检查,空转。
7. **锁/IO 阻塞无超时** —— 锁没释放、网络或 DB 调用没设超时,对端不响应就一直挂着。

**排查**:pprof 看 goroutine 数量是否持续增长、卡在哪一行, 两次pprof 对比
**预防**:凡是会阻塞的操作,都要问一句"能否正常退出?"——用 context、select + ctx.Done()、设超时、defer Unlock、保证 channel 有人收发

# 函数传参是引用值类型还是引用类型
都是引用类型, 如果是指针就是拷贝新的指针指向值类型; 如果是值就是值拷贝.

# 一个对象是分配在栈上还是堆上？
go 得局部变量如果在离开作用域后没有被引用, 会分配在栈上. 逃逸得可能情况是被外部引用, 超过栈空间, 变量大小不确定等

# 理解golang happen before
* 博客

  https://studygolang.com/articles/34209
  
* 官方链接

https://go.dev/ref/mem?utm_source=chatgpt.com
# 数组和切片的区别
数组是固定长度, 长度是数组类型的一部分 适用固定长度的场景, 值传递(方法参数传递时会拷贝) , 
```
初始化: 
var array [10]int

var array = [5]int{1,2,3,4,5}
```
</br>

切片它有三个属性：指针、长度(slice 的长度)和容量(底层数组的容量)。
切片不需要指定大小, 地址传递, 底层是数组, 当数组容量不够时会扩容.
```
sli:=make([]int, 0, 3) // 如果第一个参数不是 0 , 例如是 2, 会默认填充 2 个 0 在slice 开头, 所以一般情况下, len指定0, cap 指定一个预定的容量, 减少扩容次数

sli:=make([]int, 1) // 如果只有一个参数, 那么len 和cap 都是1, 默认有一个0
```
# golang 经典编程问题
## mutex 是不可重入的
```
package main
import (
	"fmt"
	"sync"
)
var mu sync.Mutex
var chain string
func main() {
	chain = "main"
	A()
	fmt.Println(chain)
}
func A() {
	mu.Lock()
	defer mu.Unlock()
	chain = chain + " --> A"
	B()
}
func B() {
	chain = chain + " --> B"
	C()
}
func C() {
	mu.Lock()
	defer mu.Unlock()
	chain = chain + " --> C"
}
A: 不能编译
B: 输出 main --> A --> B --> C
C: 输出 main
D: panic
```
选D, mutex 不能重入, 不能同一个go 调用两次lock, 

## RWMutex read lock must wait write lock to acquire first
```
package main
import (
	"fmt"
	"sync"
	"time"
)
var mu sync.RWMutex
var count int
func main() {
	go A()
	time.Sleep(2 * time.Second)
	mu.Lock()
	defer mu.Unlock()
	count++
	fmt.Println(count)
}
func A() {
	mu.RLock()
	defer mu.RUnlock()
	B()
}
func B() {
	time.Sleep(5 * time.Second)
	C()
}
func C() {
	mu.RLock()
	defer mu.RUnlock()
}
A: 不能编译
B: 输出 1
C: 程序hang住
D: panic
```
选D, 因为go 1 一开始先获取读锁, 然后sleep 5s, 此时main go 获取写锁, 等待 go1 释放读锁, 此时 go 1 继续获取读锁, 因为之前有写锁等待, 所以go 1 等待main 释放写锁, 互相等待, 导致死锁. 
</br>
官方原话: If any goroutine calls RWMutex.Lock while the lock is already held by one or more readers, concurrent calls to RWMutex.RLock will block until the writer has acquired (and released) the lock, to ensure that the lock eventually becomes available to the writer
https://pkg.go.dev/sync#RWMutex.Lock

## 双检查实现单例, 内存可见性问题
```
package doublecheck
import (
	"sync"
)
type Once struct {
	m    sync.Mutex
	done uint32
}
func (o *Once) Do(f func()) {
	if o.done == 1 {
		return
	}
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		o.done = 1
		f()
	}
}
A: 不能编译
B: 可以编译，正确实现了单例
C: 可以编译，有并发问题，f函数可能会被执行多次
D: 可以编译，但是程序运行会panic
```

选C, o.done 不能保证多个线程间的可见性, 除非使用atomic package 类似java 的volatile 关键字, mutex不像java 的synchronize 能保证内存可见性.

## mutex 在使用之后不能复制
```
package main

import (
	"fmt"
	"sync"
)

type MyMutex struct {
	count int
	sync.Mutex
}

func main() {
	var mu MyMutex
	mu.Lock()
	var mu2 = mu
	mu.count++
	mu.Unlock()
	mu2.Lock()
	mu2.count++
	mu2.Unlock()
	fmt.Println(mu.count, mu2.count)
}
A: 不能编译
B: 输出 1, 1
C: 输出 1, 2
D: panic
```
选D, var mu2 = mu 会做值拷贝, mutex 在使用之后不能复制, 

* 解决办法

var mu2 = &mu 这样就不会做值拷贝, 指向同一个对象 
## 内存池Pool
```
package main
import (
	"bytes"
	"fmt"
	"runtime"
	"sync"
	"time"
)
var pool = sync.Pool{New: func() interface{} { return new(bytes.Buffer) }}
func main() {
	go func() {
		for {
			processRequest(1 << 28) // 256MiB
		}
	}()
	for i := 0; i < 1000; i++ {
		go func() {
			for {
				processRequest(1 << 10) // 1KiB
			}
		}()
	}
	var stats runtime.MemStats
	for i := 0; ; i++ {
		runtime.ReadMemStats(&stats)
		fmt.Printf("Cycle %d: %dB\n", i, stats.Alloc)
		time.Sleep(time.Second)
		runtime.GC()
	}
}
func processRequest(size int) {
	b := pool.Get().(*bytes.Buffer)
	time.Sleep(500 * time.Millisecond)
	b.Grow(size)
	pool.Put(b)
	time.Sleep(1 * time.Millisecond)
}

A: 不能编译
B: 可以编译，运行时正常，内存稳定
C: 可以编译，运行时内存可能暴涨
D: 可以编译，运行时内存先暴涨，但是过一会会回收掉

```
选C, 因为Pool 的size 只增不减, 只要单个buffer 申请过 256MB 就算后续grow 到 1KB 实际也会占用 256MB, 并且因为在 processRequest 函数中, 每个goroutine 都会sleep 500ms 就算此时GC 也没法回收goroutine 持有的buffer, 所以大量buffer 是无法在GC 时回收的, 经过测试很快就OOM. 

### 解决办法
#### Pool 对象大小只增不减
所以通常大对象是不会进入pool 的, 或者大对象进入大pool , 小对象进入小pool

#### 限制并发
```
sem := make(chan struct{}, 1) // 并发1
func processLimitConcurrency() {
    sem <- struct{}{}
    defer func() { <-sem }()

    processRequest(1 << 28)
}
```

## 读nil chan 和并发问题
```
package main
import (
	"fmt"
	"runtime"
	"time"
)
func main() {
	var ch chan int
	go func() {
		ch = make(chan int, 1)
		ch <- 1
	}()
	go func(ch chan int) {
		time.Sleep(time.Second)
		<-ch
	}(ch)
	c := time.Tick(1 * time.Second)
	for range c {
		fmt.Printf("#goroutines: %d\n", runtime.NumGoroutine())
	}
}

A: 不能编译
B: 一段时间后总是输出 #goroutines: 1
C: 一段时间后总是输出 #goroutines: 2
D: panic
```

选C, 有main 和第二个goroutine, 第一个go 在发送完成后就退出了, 由于第二个go 初始化的时候并没有等待第一个go 对ch 的初始化, 并且goroutine 的参数是值传递, 所以就算等待第一个go 初始化完成, 第二个go 拿到的chan 还是nil, 所以一直会block
### 解决办法
一开始就初始化 ch 或者ch 不要作为参数传递到goroutine, 作为全局变量传递

## close nil chan
```
package main
import "fmt"
func main() {
	var ch chan int
	var count int
	go func() {
		ch <- 1
	}()
	go func() {
		count++
		close(ch)
	}()
	<-ch
	fmt.Println(count)
}
A: 不能编译
B: 输出 1
C: 输出 0
D: panic
```
选d, close nil chan 会直接panic, send 和receive nil chan 会block. 

## chan happen before
```
package main
var c = make(chan int)
var a int
func f() {
	a = 1
	<-c
}
func main() {
	go f()
	c <- 0
	print(a)
}
A: 不能编译
B: 输出 1
C: 输出 0
D: panic

```
选B, 无缓冲chan 的HB 原则是receive hb send 所以时序是 a = 1, <-c, c<- 0, print(1), 如果是有缓冲chan, send HB receive; 如果是有缓冲chan, The closing of a channel is synchronized before a receive that returns a zero value; 在单个goroutine 内, 代码顺序在前的代码 HB 在后的代码; 
HB 有传递原则, a HB b, b HB c, 那么 a HB c.

# GMP model 
![image](https://github.com/Knight-Wu/articles/assets/20329409/1051e364-079a-4cda-8e9b-c2e384f0286f)

## 为什么需要 gmp 模型
### 概况
总而言之为了提高 cpu 的使用率, 当一个任务需要cpu 等待时立马切换到其他任务上, 一种思路是遇到阻塞时, 新起一个线程去工作, 这样任务和操作系统线程之间是 1:1 的关系, 但是会造成切换开销过大, 线程太多, 消耗太多内存; 
那么就需要任务和线程之前是N:M 的关系, 由一种调度关系去把任务调度到能执行的线程上, 线程的数量不能太多, 任务要足够轻量化来降低切换开销和内存消耗, 那么就有了GMP 模型. 
### 高效的并发模型

Goroutine 比系统线程更轻量，创建和销毁的开销更小。使用 GMP 模型，可以在单个进程内创建数十万甚至数百万个 Goroutine，而不会像线程那样引起过多的资源消耗和调度开销。
### 调度灵活性

GMP 模型允许在 P 上运行多个 Goroutine，并在不同的 M 线程之间切换。这样，Go 运行时可以更好地利用多核处理器的性能，最大限度地提高并发执行效率。
### 阻塞操作处理

当 Goroutine 执行阻塞操作（如 I/O 操作）时，Go 运行时会将其从当前的 M 线程中剥离出来，并分配另一个 Goroutine 到该 M 线程上继续执行。这种机制确保了阻塞操作不会阻塞整个线程，提高了程序的响应性和吞吐量。
### 自动扩展和收：

GMP 模型可以根据负载自动调整 M 线程的数量，以适应当前的工作量。它可以在需要时创建更多的 M 线程，或者在空闲时销毁不必要的 M 线程，从而高效地管理系统资源。

## goroutine 的资源占用
每个协程初始化栈大小是2KB, 会随调用深度增加而扩大, 栈会保存函数的参数以及调用信息, 栈是每个协程独有的, 当变量很大的时候会保存在堆上, 当多个协程共享变量的时候, 这个变量就需要放在堆上了. 
## 原理
### GMP 是什么
![image](https://github.com/user-attachments/assets/e5605f8a-4fd3-4a94-b273-736f11414416)
![image](https://github.com/user-attachments/assets/03f5ce90-4691-4e12-91cd-ad6f82f24d39)

#### G(goroutine)
每次 go func 就会创建一个 G, 保存函数、栈和上下文, 如果 G 里面工作很简单, 数量很多也没关系, 如果是需要网络连接和创建文件, 则太多的 G 会导致too many files open or Resource temporarily unavailable 

G 需要绑定 M 来跑, M 需要绑定 P 来跑. 
G is bound to M to run, and M needs to be bound to P to run, so theoretically the number of running G at the same time is equal to the number of P

* G占用得内存

 每个占用内存 2KB(初始栈大小), 连续内存, 按需增长（2KB → 4KB → 8KB → …）, 通过 栈拷贝 扩容, 1 万个 goroutine ≈ 20 MB 栈空间
 
* G的内部结构中重要字段如下，完全结构参见源码

```
type g struct {
    stack       stack   // g自己的栈
    m            *m      // 隶属于哪个M
    sched        gobuf   // 保存了g的现场，goroutine切换时通过它来恢复
    atomicstatus uint32  // G的运行状态
    goid         int64
    schedlink    guintptr // 下一个g, g链表
    preempt      bool //抢占标记
    lockedm      muintptr // 锁定的M,g中断恢复指定M执行
    gopc          uintptr  // 创建该goroutine的指令地址
    startpc       uintptr  // goroutine 函数的指令地址
}
```
##### 发生G 切换时
在 Go 语言的协程（Goroutine）中，切换上下文时，状态的保存和管理是由 Go 运行时系统负责的。具体来说，状态保存在 Goroutine 自身的数据结构中，确保在上下文切换时，Goroutine 可以恢复到正确的执行点继续运行。
##### 调度过程中阻塞 
GMP模型的阻塞可能发生在下面几种情况：

1. I/O，select
2. block on syscall
3. channel
4. 等待锁
5. runtime.Gosched()
* 用户态阻塞
   
当goroutine因为channel操作或者network I/O而阻塞时（实际上golang已经用netpoller实现了goroutine网络I/O阻塞不会导致M被阻塞，仅阻塞G），对应的G会被放置到某个wait队列(如channel的waitq)，该G的状态由_Gruning变为_Gwaitting，而M会跳过该G尝试获取并执行下一个G，如果此时没有runnable的G供M运行，那么M将解绑P，并进入sleep状态；当阻塞的G被另一端的G2唤醒时（比如channel的可读/写通知），G被标记为runnable，尝试加入G2所在P的runnext，然后再是P的Local队列和Global队列。

* 系统调用阻塞
  
当G被阻塞在某个系统调用上时，此时G会阻塞在_Gsyscall状态，M也处于 block on syscall 状态，此时的M可被抢占调度：执行该G的M会与P解绑，而P则尝试与其它idle的M绑定，继续执行其它G。如果没有其它idle的M，但P的Local队列中仍然有G需要执行，则创建一个新的M；当系统调用完成后，G会重新尝试获取一个idle的P进入它的Local队列恢复执行，如果没有idle的P，G会被标记为runnable加入到Global队列。
#### M（Machine）
每个M 代表一个操作系统线程，负责真正执行指令. 每个M 占用 8MB 内存(栈空间), 由操作系统分配.  一个 M 阻塞了，会创建新的 M。
</br>
线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。



##### M 的定义
M is an object in runtime that represents a thread. Each M object created creates a thread bound to M. New threads are created by executing the clone() system call. runtime defines the maximum number of M to be 10000. The maximum number of M is defined in runtime as 10000, which can be adjusted by debug.SetMaxThreads(n).

* M 的创建

第一种是主线程 : M0, The Golang program creates the main thread when it starts, and the main thread is the first M i.e. M0.
另一种是当有 G 要创建或运行时, 并且有空闲的 P, 就会去找空闲的 M, 没有的话就创建.
When a new G is created or a G goes from _Gwaiting to _Grunning and there is a free P, startm() will be called, first getting an M from the global queue (sched.midle) and binding the free P to execute the G. If there is no free M, M will be created by newm().

如果 G (当做一个 function), 触发了系统调用, M 会释放 P; 如果结束了系统调用, M 会找空闲的 P,找不到就进入 sleep. 并且会记录 old P , 结束系统调用的时候倾向于找 old P, 因为之前的内存可以用, 减少拷贝. 
When the G associated with M enters the system call, M will actively unbind with the associated P. When the G associated with M executes the exitsyscall() function to exit the system call, M will find a free P to bind, if no free P is found then M will call stopm() to enter the sleep state.

* thread info


/proc/sys/kernel/threads-max: indicates the maximum number of threads supported by the system.
/proc/sys/kernel/pid_max: indicates the limit of the system global PID number value, every process or thread has an ID, the process or thread will fail to be created if the value of the ID exceeds this number.
/proc/sys/vm/max_map_count: indicates a limit on the number of VMAs (virtual memory areas) a process can have.

* M 中的 G0(g 零)
Every time an M is started, the first goroutine created is g0. Each M will have its own g0. g0 is mainly used to record the stack information used by the worker thread, and is only used to be responsible for scheduling, which needs to be used when executing the scheduling code. When executing the user goroutine code, the stack of the user goroutine is used, and the stack switch occurs when scheduling.

* M的内部结构中重要字段如下，完全结构参见源码

```

type m struct {
    g0      *g     // g0, 每个M都有自己独有的g0

    curg          *g       // 当前正在运行的g
    p             puintptr // 隶属于哪个P
    nextp         puintptr // 当m被唤醒时，首先拥有这个p
    id            int64
    spinning      bool // 是否处于自旋

    park          note
    alllink       *m // on allm
    schedlink     muintptr // 下一个m, m链表
    mcache        *mcache  // 内存分配
    lockedg       guintptr // 和 G 的lockedm对应
    freelink      *m // on sched.freem
}
```




#### P（Processor）
调度上下文，代表执行 Go 代码的“CPU 执行权” , 数量通常是 CPU 核心数. It can be set at program startup with the environment variable GOMAXPROCS or the runtime.GOMAXPROCS() method, and the number of P’s is fixed for the duration of the program.
</br>
g 全局队列（Global Queue）：存放等待运行的 G。
</br>
P 的本地队列(用于存放本地的G)：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。当某个正在运行的G 新建 G’时，G’优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。

##### P 的定义
* IO 密集型

io 密集型系统中 P 的数量可以多于逻辑核心, 因为 M 会被 system call block, 此时 P 会被阻塞一会, 等待周期性 check (10MS) 去释放 P 和 G 到新的 M. 

In the IO-intensive scenario, the number of P can be adjusted appropriately, because M needs to be bound to P to run, and M will fall into the system call when executing G. At this time, P associated with M is in the waiting state, if the system call never returns, then the CPU resources are actually wasted during this time waiting for the system call, although there is a sysmon monitoring thread in runtime can seize G, here is to seize P associated with G, let P rebind a M to run G, but sysmon is periodic execution of seizure, after sysmon stable operation every 10ms to check whether to seize P, 

The open source database project https://github.com/dgraph-io/dgraph adjusts GOMAXPROCS to 128 to increase the IO processing power.

* p 状态

<img width="695" alt="image" src="https://user-images.githubusercontent.com/20329409/234467418-f25a923c-6b61-498d-9147-e7f317d48db7.png">

* P 的字段, 完整见源码

```
  type p struct {
    id          int32
    status      uint32 // P的状态
    link        puintptr // 下一个P, P链表
    m           muintptr // 拥有这个P的M
    mcache      *mcache  

    // P本地runnable状态的G队列，无锁访问
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr
    
    runnext guintptr // 一个比runq优先级更高的runnable G

    // 状态为dead的G链表，在获取G时会从这里面获取
    gFree struct {
        gList
        n int32
    }

    gcBgMarkWorker       guintptr // (atomic)
    gcw gcWork

}
```
##### 为什么需要 P

Go 1.1 之后才引入P, 
Before Golang 1.1, there was no P component in the scheduler. The performance of the scheduler was still poor at this time. Dmitry Vyukov of the community summarized the problems in the current scheduler and designed to introduce the P component to solve the current problems ([Scalable Go Scheduler Design Doc](https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#heading=h.mmq8lm48qfcw)), and introduced the P component in Go 1.1. The introduction of the P component not only solves several problems listed in the documentation, but also introduces some good mechanisms.

* global queue lock

之前没有p 的时候, 需要 global queue lock, 因为所有的 g 都在全局队列里, 引入了 p, 就可以大多数情况无锁访问 p 的 local G queue.</br>

* 为什么不直接把本地队列挂在M 上呢 ?
  
一般来讲，M 的数量都会多于 P。像在 Go 中，M(线程) 的数量一般没有上限, 需要就创建，P 的默认数量的 CPU 核数。另外由于 M 的属性，也就是如果存在系统阻塞调用，阻塞了 M，又不够用的情况下，M 会不断增加。
M 不断增加的话，如果本地队列挂载在 M 上，那就意味着本地队列会随着M 得创建和销毁频繁被拷贝, 这显然是不合理的

* G 切换问题

切换G 带来的开销, 如果没有P, 一个goroutine 里面创建的g' 会先放到全局g 的队列, 而不是直接被执行, 现在有了P 就直接放到P 的本地队列直接被执行

* M’s memory cache (M.mcache) problem

mcache 是一个 M object local cache 存放 G 的对象, 但是 M 有可能被 block by 系统调用, 所以 cache 就浪费了. 所以引入 P 之后 mcache 搬到了 P, 只有在运行的时候才会被占用, 不会造成空间浪费, 也避免了锁, 因为是没有其他线程去竞争的. 

* Frequent thread blocking and wake-up problems ?(不说)

In the original scheduler, the number of system threads is limited by runtime.GOMAXPROCS(). Only one system thread is opened by default. And since M performs operations such as system calls, when M blocks, it does not create a new M to perform other tasks, but waits for M to wake up, and M switches between blocking and waking frequently, which causes additional overhead. In the new scheduler, when M is in the system scheduling state, it will be disassociated from the bound P and will wake up the existing or create a new M to run other G bound to P.


### 可视化 GMP
有 2 种方式可以查看一个程序的 GMP 的数据。

方式 1：go tool trace

trace 记录了运行时的信息，能提供可视化的 Web 页面。

### 调度器的设计策略

1）work stealing 机制

​ 当本线程无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，避免频繁的创建、销毁线程，而是对线程的复用

2）hand off 机制

​ 当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程执行。充分利用cpu

### GMP 简要介绍
* G(Goroutine):协程,Go 的并发执行单元,栈很小(2KB 起,可伸缩),创建和切换成本低。

* M(Machine):系统线程,真正执行代码的实体,由 OS 调度。

* P(Processor):逻辑处理器,持有可运行 G 的本地队列,M 必须绑定 P 才能执行 G。P 的数量由 GOMAXPROCS 决定,默认等于 CPU 核数。

* 核心关系

P 是 G 和 M 之间的桥梁。M 要执行 G,必须先拿到一个 P,从 P 的本地队列里取 G 来跑。所以真正的并行度由 P 的数量决定

#### 为什么这样设计(优势)

* 早期是 GM 模型(只有全局队列 + 全局锁),锁竞争严重;引入 P 后有了本地队列,大幅减少锁竞争。

* goroutine 轻量 + 用户态调度,切换不陷入内核,几十万 goroutine 也能轻松跑。

* work stealing + hand off 保证了负载均衡和阻塞不阻塞整体。

### 创建一个goroutine 如何执行
![image](https://github.com/user-attachments/assets/408947c5-598b-4fd3-8fc4-a4fcb00a2828)
1. 创建一个G 时先放P 的本地队列, 本地队列放不下, 放全局队列
2. 每个P和一个M绑定，M是真正的执行P中goroutine的实体(流程3)，M从绑定的P中的局部队列获取G来执行
3. 当M绑定的P的局部队列为空时，M会从全局队列获取到本地队列来执行G(流程3.1)，当从全局队列中没有获取到可执行的G时候，M会从其他P的局部队列中偷取G来执行(流程3.2)，这种从其他P偷的方式称为work stealing
4. 当G阻塞(因系统调用 syscall)时会阻塞M，此时P会和M解绑即hand off，并寻找新的idle的M 来继续执行P 中剩余得G ，若没有idle的M就会新建一个M(流程5.1)。当阻塞的G恢复后会重新进入runnable进入P队列等待执行(流程5.3)


# debug
### debug binary file in local machine
1. download a small utility program named gops, available at [https://github.com/google/gops](https://github.com/google/gops). This program helps the IDE find Go process running on your machine. Then invoke the _Attach to Process…_ feature again.
2. If you are running with Go 1.10 or newer, you need to add `-gcflags="all=-N -l"` to the `go build` command. 然后运行. 
3. 然后选择goland Run 菜单下的 attach , 选择自己的进程, 就可以断点调试了

### debug binary file in remote machine
1. If you are running with Go 1.10 or newer, you need to add `-gcflags="all=-N -l"` to the `go build` command.  然后上传服务器.
2. 在远程服务器下载 delve. 
```
go get -u github.com/go-delve/delve/cmd/dlv
```
[https://github.com/go-delve/delve/blob/master/Documentation/installation/linux/install.md](https://github.com/go-delve/delve/blob/master/Documentation/installation/linux/install.md)

3. 然后运行应用. 
有两种方式: 
a. let the debugger run the process for you: if you choose this option, you need to run `dlv --listen=:2345 --headless=true --api-version=2 exec ./application`
b. attaching to the process: you need to run  `dlv --listen=:2345 --headless=true --api-version=2 attach <pid>` where  _<pid>_  is the process id of your application.
都需要暴露服务器上的应用的端口给 local machine
4. _Run | Edit Configurations… | + | Go Remote_ and configuring the host and port your remote debugger is listening on.


[jetbrains 的完整文档](https://blog.jetbrains.com/go/2019/02/06/debugging-with-goland-getting-started/#debugging-a-running-application-on-a-remote-machine)

### float 运算
若某个浮点数, 没有经过强制保留两位小数, 则有可能再经过运算后, 出现尾数, 
例如
> float a = 0.02, float b= 1-0.02= 0.9800001 , 

* 解决办法
1. 计算结果强制保留两位小数: 
>c, _ := strconv.ParseFloat(fmt.Sprintf("%.2f", 1-a), 32), 则c = 0.98

2. 浮点数计算框架: 
[https://github.com/shopspring/decimal](https://github.com/shopspring/decimal)

3. 将浮点数转化为整数, 乘 10 的 n 次方


# value or pointer
### https://jordanorelli.com/post/32665860244/how-to-use-interfaces-in-go 总结
func (t T)MyMethod(s string) {
    // ...
}

is a function of type  `func(T, string)`; method receivers are passed into the function by value just like any other parameter. by value means copy.

* Any changes to the receiver made inside of a method defined on a value type (e.g., `func (d Dog) Speak() { ... }`) will not be seen by the caller because the caller is scoping a completely separate `Dog` value.

* This works because a pointer type can access the methods of its associated value type, but not vice versa.( a pointer type may call the methods of its associated value type, but not vice versa)

* Since everything is passed by value, it should be obvious why a `*Cat` method is not usable by a `Cat` value; any one `Cat` value may have any number of `*Cat` pointers that point to it. If we try to call a `*Cat` method by using a `Cat` value, we never had a `*Cat` pointer to begin with. Conversely, if we have a method on the `Dog` type, and we have a `*Dog` pointer, we know exactly which `Dog` value to use when calling this method, because the `*Dog` pointer points to exactly one `Dog` value
( 所以 method receiver 最好是 pointer ? )


* Remember: everything is pass-by-value in Go. That means that inside of the `UnmarshalJSON` method, the pointer `t` is not the same pointer as the pointer in its calling context; it is a copy. If you were to assign `t` to another value directly, you would just be reassigning a function-local pointer; the change would not be seen by the caller.
(包括指针也是, 所以在函数内把指针重新指向, 对函数外也不生效)

* as a rule of thumb you can just remember that it’s typically better to take in an `interface{}` value as a parameter than it is to return an `interface{}` value 
(interface 最好作为参数而不是返回值) 

-   an  `interface{}`  value is not of any type; it is of  `interface{}`  type
-   interfaces are two words wide; schematically they look like  `(type, value)`

*  `var u User` will automatically [zero](https://href.li/?http://golang.org/ref/spec#The_zero_value "http://golang.org/ref/spec#The_zero_value") the `User` struct. Go is not like some other languages in that declaration and initialization occur separately,

# protobuf
### 好处
1. 用Base 128 Varints 编码数字

例如300 , int, 本来需要四个字节, 用这个编码之后只需要两个字节. 
让比较小的int 用更少的字节, 但是较大的int 就需要五个的字节, 从统计学意义上是能节省空间的, 

具体原理: 
用每一个字节的第一位(是否是1 ) 来表示是否还有下一个字节, 所以每个字节的有效位数只有7 位. 

如果用不到 1 个字节，那么最高有效位设为 0 ，如下面这个例子，1 用一个字节就可以表示，0000 0001. 

#  Calling a method on a nil struct pointer doesn't panic.

because [Michael Jones explained this well](https://groups.google.com/d/msg/golang-nuts/wcrZ3P1zeAk/WI88iQgFMvwJ)
摘录如下
> I think the question is really "but how would you inspect the object pointer to find and dispatch the function, as in C++ vtable." We might answer this from that angle:

  

> Method dispatch, as is used in some other languages with something like objectpointer->methodName() involves inspection and indirection via the indicated pointer (or implicitly taking a pointer to the object) to determine what function to invoke using the named method. Some of these languages use pointer dereferencing syntax to access the named function. In any such cases, calling a member function or method on a zero pointer understandably has no valid meaning.
> 
>   
> 
> In Go, however, the function to be called by the Expression.Name() syntax is entirely determined by the type of Expression and not by the particular run-time value of that expression, including nil. In this manner, the invocation of a method on a nil pointer of a specific type has a clear and logical meaning. Those familiar with vtable[] implementations will find this odd at first, yet, when thinking of methods this way, it is even simpler and makes sense. Think of:
> 
>   
> 
> func (p *Sometype) Somemethod (firstArg int) {}
> 
>   
> 
> as having the literal meaning:
> 
>   
> 
> func SometypeSomemethod(p *Sometype, firstArg int) {}
> 
>   
> 
> and in this view, the body of SometypeSomemethod() is certainly free to test it's (actual) first argument (p *Sometype) for a value of nil. Note though that the calling site invoking on a nil value must have a context of the expected type. An effort to invoke an unadorned nil.Somemethod would not work in Go because there is be no implicit "Sometype" for the typeless value nil to expand the Somemethod() call into "SometypeSomemethod()"

大致意思就是如果 object 是 nil , 那是如何找到这个 function 的呢? 在 golang 中与其他语言不同,  [Expression.method](http://expression.name/)() 是由 expression 的 type 决定的, 而不是 expression 的 value 决定的. 但是有些情况却不一样, 因为没有隐式的 type 转换. 

**所以当出现 nil 报错的时候需要看清是哪一行的报错, nil.field 是肯定报错的, 但是 nil.method 却不一定报错 !**

# concurrent


## channel
* close
 send on a close channel will cause panic, receive on close channel 接收所有已经发送的 val, 直到 val 接收完, 若 val 已经接收完, 则后续的 receive 会立马返回, 并且返回 val 的 zero value

* unbufferer channel
>A send operation on an unbuffered channel blocks the sending goroutine until another goroutine executes a corresponding receive on the same channel, at which point the value is transmitted and both goroutines may continue. Conversely, if the receive operation was attempted first, the receiving goroutine is blocked until another goroutine performs a send on the same channel.

send 和 receive 在一个 unbuffered channel, 会互相阻塞, 例如若没有 goroutine 在 receive , send 会阻塞, 反之亦然

### channel 实现原理
简要介绍(说重点)
channel 底层是一个 hchan 结构体,核心由三部分组成:

* 环形缓冲区(ring buffer):有缓冲 channel 用的循环队列,存数据。

* 两个等待队列(sendq / recvq):存放被阻塞的发送方和接收方 goroutine(用 sudog 封装)。

* 一把互斥锁(lock):保护整个 channel 的并发操作。

收发数据时先抢锁,再决定是直接传递、放进缓冲、还是阻塞入队。channel 不是无锁的,而是加锁的队列 + goroutine 阻塞唤醒机制。

#### 可能被追问的细节
1. 无缓冲 channel 怎么传数据?
无缓冲(dataqsiz=0)没有 buf,发送和接收必须同步配对:一方先到就阻塞入队,另一方到了直接内存拷贝 + 唤醒。所以是强同步通信。

2. 直接拷贝优化(关键点)
有接收方在等时,发送方直接把数据拷到接收方的栈上,不经过缓冲区,省一次拷贝。这是 channel 高效的原因之一。

3. close 做了什么?
关闭时加锁,设置 closed 标志,然后唤醒 sendq 和 recvq 里所有阻塞的 goroutine。接收方被唤醒后读到零值(ok=false),发送方被唤醒会 panic。

4. close 的几个 panic 点

关闭已关闭的 channel → panic。
向已关闭的 channel 发送 → panic。
关闭 nil channel → panic。
(读已关闭 channel 是安全的,返回零值。)

5. 读写 nil channel 会怎样?
永久阻塞。常用于 select 中"动态禁用"某个 case。

6. goroutine 阻塞唤醒的本质
阻塞时调用 gopark 让出 M,goroutine 状态变 waiting,M 去跑别的 G(不会阻塞线程);被唤醒时 goready 把它重新放回运行队列。channel 阻塞不会阻塞底层线程,这是和锁的重要区别。

7. sudog 是什么?
封装一个等待中的 goroutine 及其要收发的数据元素,挂在 sendq/recvq 上,是连接 channel 和 goroutine 调度的桥梁。

8. select 怎么实现的?
多个 case 会随机打乱顺序避免饥饿,逐个尝试加锁检查能否立即收发;都不行就把当前 goroutine 同时挂到所有相关 channel 的等待队列上,任一就绪即被唤醒,再从其他队列摘除。

9. channel 是 happens-before 的吗?
是。发送 happens-before 对应的接收完成;close happens-before 因 close 而返回的接收。这是 channel 能安全传递数据的内存模型保证。

## sync map
sync.Map 是 Go 提供的并发安全 map,核心设计是读写分离 + 空间换时间,用两个 map 来减少锁竞争:

read(只读 map):atomic.Value 存储,读操作走这里完全无锁(原子读),性能极高。
dirty(脏 map):普通 map + Mutex 保护,存放新写入的、read 里还没有的 key。

读的时候先无锁查 read,命中直接返回;没命中(且可能在 dirty 里)才加锁去查 dirty。这样读多写少场景下,绝大多数读都不用加锁。
适用场景:读多写少、key 集合相对稳定。写多或 key 频繁变化时,性能反而不如 map + RWMutex

1. 为什么用两个 map?
为了让读尽量无锁。如果只有一个 map 加锁,读写都要抢锁。拆成 read/dirty 后,稳定的 key 都在 read 里,读直接原子操作,锁只在涉及 dirty 时才用。
2. read 和 dirty 怎么同步的(misses 机制)?
read 里查不到 key 时,会去 dirty 查,同时 misses 计数 +1。当 misses 累积到 ≥ len(dirty) 时,触发提升(promote):把整个 dirty 直接升级为新的 read,dirty 置空。这样高频访问的 key 最终都会沉淀到无锁的 read 里。
3. entry 的三种状态(关键点)
每个 value 用 entry 指针封装,有三种状态:

正常值:指向真实数据。
nil:key 被删除,但 dirty 已重建,此 entry 还留在 read 里(软删除)。
expunged(已删除标记):特殊指针,表示该 key 在 read 中被删且不在 dirty 里。重建 dirty 时,nil 会被标记成 expunged,避免被复制进新 dirty。

4. 删除为什么用"标记删除"而不是真删?
如果 key 在 read 里,删除只是把 entry 的值原子置为 nil(无锁软删除),不立即从 map 摘除。真正清理发生在下次 dirty 重建时。这样删除也能走无锁路径。
5. dirty 是怎么重建的?
当一个新 key 第一次写入(read 和 dirty 都没有)时,如果 dirty 为 nil,会把 read 中所有未删除的 entry 拷贝到新 dirty,然后再写入新 key。这个拷贝是 O(n) 的,是 sync.Map 写入的主要开销来源。
6. 为什么写多场景性能差?

每次写新 key 可能触发 dirty 重建(全量拷贝)。
写操作要加锁。
频繁 promote 也有成本。
所以写密集时不如直接 RWMutex + map。

7. 和 RWMutex+map 的本质区别
RWMutex 读也要加读锁(虽然多读可并发,但有锁开销和写饥饿问题);sync.Map 的读在命中 read 时是纯原子操作,零锁,这是它在读多场景的核心优势

* 读取过程

先读read, 读到直接返回, 读不到且dirty 有更多的数据, 则从dirty 读, 并且报告一次miss ,因为后续是一个slow path. 

```
// Load returns the value stored in the map for a key, or nil if no
// value is present.
// The ok result indicates whether value was found in the map.
func (m *Map) Load(key any) (value any, ok bool) {
	read, _ := m.read.Load().(readOnly)
	e, ok := read.m[key]
	if !ok && read.amended {
		m.mu.Lock()
		// Avoid reporting a spurious miss if m.dirty got promoted while we were
		// blocked on m.mu. (If further loads of the same key will not miss, it's
		// not worth copying the dirty map for this key.)
		read, _ = m.read.Load().(readOnly)
		e, ok = read.m[key]
		if !ok && read.amended {
			e, ok = m.dirty[key]
			// Regardless of whether the entry was present, record a miss: this key
			// will take the slow path until the dirty map is promoted to the read
			// map.
			m.missLocked()
		}
		m.mu.Unlock()
	}
	if !ok {
		return nil, false
	}
	return e.load()
}
```

* 修改过程

## sync.waitgroup

*  waitgroup add 必须要在 go func() 的外面, 保证 happen before waitgroup wait().
* break in select means break out in one loop, 如果外面还有 for loop, 并不能跳出, 需要一个标志位. 
排查bug 的两点总结: 
1. waitgroup 要传指针
2. add 要在主线程做, 否则可能主线程都wait 了, 还没add , 就会出现和预期不符合的结果. 
>Typically this means the calls to Add should execute before the statement  creating the goroutine or other event to be waited for.

铭记一点: 越离奇, 越不符合之前认知的错误, 越复杂, 越要静下心来, 细细分析, 用最理性的思维, 判断, 很有可能需要推翻之前的很多认知, 这样才有可能越快找到问题. 


# type conversion
用 built-in 的 string(),int() 为什么有时候不行, 有时候 string([]byte) 又可以
### int to string	
[https://stackoverflow.com/questions/39442167/convert-int32-to-string-in-golang](https://stackoverflow.com/questions/39442167/convert-int32-to-string-in-golang)

### int array to string
[https://stackoverflow.com/questions/37532255/one-liner-to-transform-int-into-string](https://stackoverflow.com/questions/37532255/one-liner-to-transform-int-into-string)

### fmt 用法
[https://blog.csdn.net/chenbaoke/article/details/39932845](https://blog.csdn.net/chenbaoke/article/details/39932845)

# time format
[https://ichon.me/post/998.html](https://ichon.me/post/998.html)


而Go语言的方式则略显奇葩，采用的是`"2006-01-02 15:04:05"`这样的layout string:

> time.Now().Format("2006-01-02 15:04:05")

# 当go 中引用了cgo, cgo 不能在mac 编译linux , 所以需要在本地的linux docker 中编译, 本地编译方便, 

1. 启动docker container
 docker run --rm -it -v ~/.ssh/:/root/.ssh/ -v /usr/local/go/src/:/go/src -v /Users/xxx/go:/go/code  golang:1.12.5 /bin/bash
 
本地命令: 
docker run --rm -it -v ~/.ssh/:/root/.ssh/ -v /usr/local/go/src/:/go/src -v /Users/tongwu/go:/go/code  golang:1.12.5 /bin/bash

2. 在container 里面指定GOPATH, 否则找不到依赖. 
默认的依赖在 $GOPATH/src  目录下, 根据报错提示更改 $GOPATH 到指定的目录. 即可

本地命令: export GOPATH=/go/code


# static link confluent-kafka-go
背景: 
golang 的 之前的 kafka 库都不太好用, sarama-kafka 已经不维护了, 而且自动提交 offset 的时候有 bug, 会导致重复消费; segmentio-kafka 不支持消费 topic 的 pattern, 并且设置了 groupid 之后就不能设置 auto.offset.reset 了, 真是够了. 然后就选用 [confluent-kafka]([https://github.com/confluentinc/confluent-kafka-go](https://github.com/confluentinc/confluent-kafka-go)) , 但是这个是用 go 封装了 c++的代码, 核心代码是 C++, 所以需要 build 的时候依赖 [librdkafka](https://github.com/edenhill/librdkafka),  这个 c++的库, 本来在物理机上很简单的, 但是需要通过 Jenkins 在 docker 上面 build, 问题可能就多了.  

依赖的方法很多, 当机器上面有这个包了之后, 接下来分为dynamic link 和 static link. 
依赖的方法: 
1. 直接用各个系统的包管理器, 例如Ubuntu apt-get install, 但是有些源下载下来的版本过低, 所以采用 confluent 的 repository. 但是! apt-get update 又出现 some indices update failed, 没有更详细的报错, 故作罢, 本来这个是最简单的方式, 而且 docker container 重启后会新建一个, 不会影响其他的环境. 
```
"wget -qO - https://packages.confluent.io/deb/5.2/archive.key |apt-key add -",
"echo \"deb [arch=amd64] https://packages.confluent.io/deb/5.2 stable main\" >> /etc/apt/sources.list",
"apt-get update",
"apt-get install -y librdkafka-dev=1.1.0~1confluent5.3.0-1",
```

2. 在一台 Ubuntu 上面编译好放到代码路径中, 但是有可能因为编译的系统环境和运行的系统环境不同而有风险, 公司的环境基本一致, 故觉得这个方法最好, 不需要依赖外部的网络, 速度又快. 但是编译好之后需要将包含rdkafka.pc 的文件夹添加到这个环境变量 PKG_CONFIG, 但是始终无法让这个环境变量生效, 莫非是嵌套 shell 的问题? 
3.  最后在这个无法找到 PKG_CONFIG 的报错中, 发现confluent-kafka-go 有个 issue : [Static build failing with missing rdkafka-static.pc #316](https://github.com/confluentinc/confluent-kafka-go/issues/316) , 介绍了一个分支是预先将 build 之后的包放到了confluent-kafka 这个的包路径中, 然后终于成功了, 可以看下他是如何预先编译的, 因为还没有正式合到 release , 还有些担心用到生产. 


### 问题
1. methods 和function 的区别
2. 短赋值的场景
3. 为什么要有指针
* Choosing a value or pointer receiver

There are two reasons to use a pointer receiver.

The first is so that the method can modify the value that its receiver points to.

The second is to avoid copying the value on each method call. This can be more efficient if the receiver is a large struct, for example.
4. 为什么要用make
5. interface 在 go里面怎么理解
6.   evaluation of `f`, `x`, `y`, and `z` happens in the current goroutine and the execution of `f` happens in the new goroutine ？ two gorountine ？
7. 如何理解channel
8. sync.Mutex 是控制程序执行的顺序还是内存可见性
9. go 编译和运行？ 如何引用其他文件的代码
10. go get 如何在 ide get 的时候自动指定版本 
11. go build 为什么需要 sudo , 然后再服务器执行就 exec format error 了, go install 是可以的
12. ssh 端口转发
