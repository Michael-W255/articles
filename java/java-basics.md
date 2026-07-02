# 一、Java 版本与演进
 
## 1. 最常用版本 + LTS 节奏
 
**简易版：**
- Java 8 还在用但在退，大概只剩两成多。
- 主力是两个 LTS：Java 17 约三成半，Java 21 约三成。
- Java 25 是 2025 年 9 月刚出的最新 LTS。
- 节奏是半年一版、每两年一个 LTS，即 8、11、17、21、25；中间非 LTS 版本是预览试验田，生产不用。
**细节版：**
- Oracle 2024.9 停了 17 的免费商用支持，导致 17 从 48% 掉到 34%，21 补上到约 31%。
- 21 靠虚拟线程成为史上采用最快的 LTS 之一。
- 交易所核心求稳，一定选 LTS；新项目倾向 21（虚拟线程红利）；25 太新，核心系统会等成熟再上。
- 被问"你们用哪个"时，绑到业务判断答：交易系统求稳选 LTS，新项目 17 或 21，别赌具体数字。
## 2. Java 8 → 11 新特性
 
**简易版：**
- 这段不是语法大革命，主要是生态打磨。
- Java 9（2017）：模块系统、集合工厂 `List.of`、Stream 增强（`takeWhile`）。
- Java 10（2018）：核心就一个 `var` 局部变量类型推断。
- Java 11（2019，LTS）：String 强化（`strip`/`isBlank`/`repeat`）、HttpClient 转正、单文件直接运行。
**细节版：**
- `List.of`/`Map.of` 返回**不可变**集合，add/put 抛 UnsupportedOperationException。
- Stream 增强：`takeWhile`（取到不满足为止）、`dropWhile`（丢到不满足为止）、带终止条件的 `iterate`。
- Optional 增强：`ifPresentOrElse`、`or`。
- `var` 只用于局部变量，类型仍是静态确定的，不能用于字段、参数、返回值。
- 11 是第一个"能替代 8"的 LTS；真正的语法革命（Record/Sealed/模式匹配/虚拟线程）在 17、21。
## 3. Record（Java 16 正式）
 
**简易版：**
- 不可变数据载体，只声明字段，自动生成构造器、访问器、`equals`、`hashCode`、`toString`。
- 访问器是 `x()` 不是 `getX()`。
- 隐式 final，不能继承类，可实现接口。
- 典型用途：DTO、多返回值、Map 的 key。
**细节版：**
- 字段是 private final；可写紧凑构造器做校验。
- 是"浅不可变"：字段若是可变容器（List），内容仍可改，需在紧凑构造器里 `List.copyOf()` 做防御性拷贝才真正不可变。
- 隐式继承 `java.lang.Record`，所以不能再继承别的类。
- 不适合 JPA 实体（JPA 需可变字段 + 无参构造器）。
## 4. Sealed 密封类（Java 17）
 
**简易版：**
- 用 `permits` 限制哪些类能继承，形成封闭类型层级。
- 被允许的子类必须是 final、sealed 或 non-sealed 之一。
- 核心价值：配合 switch 做穷尽性检查——覆盖所有子类就不用 default，加新子类会编译报错强制处理。
**细节版：**
- 和 Record、模式匹配凑齐就是代数数据类型（ADT）：Record = 积类型（数据封闭），Sealed = 和类型（类型封闭），模式匹配 = 解构消费。
- 引入价值：补数据导向编程这条腿、把穷尽性检查交给编译器（漏处理编译期就报错）、减样板。
- 对标 Kotlin 的 sealed class + when、Scala 的 case class + match、Rust 的 enum + match。
## 5. 模式匹配（instanceof: Java 16 / switch: Java 21）
 
**简易版：**
- instanceof 模式匹配（16）把类型判断和强转合成一步：`if (o instanceof String s)`，s 直接可用。
- switch 模式匹配（21）能按类型分支、解构 record、用 `when` 加守卫、带穷尽检查。
**细节版：**
- 旧写法 `if (o instanceof String) { String s = (String)o; }` → 新写法 `if (o instanceof String s && s.length() > 5)`。
- record 解构：`case Point(int x, int y) -> x + y`。
- 守卫条件：`case Integer i when i > 100 -> ...`。
- 配合 sealed 覆盖所有子类时，不需要 default 分支。
## 6. 虚拟线程（Java 21）⭐ 重点
 
**简易版：**
- JVM 管理的轻量级线程，不是 OS 线程，可起几百万个。
- 遇到阻塞调用时被挂起，释放底层 OS 载体线程，而不是阻塞它。
- 好处：用朴素的阻塞式代码拿到接近异步的吞吐，不用 reactive 复杂度。
- 适合 I/O 密集；模型类似 Go 的 goroutine（M:N 调度）。
**细节版：**
- 平台线程与 OS 线程 1:1（约 1MB 栈），只能上千个；虚拟线程栈在堆上，由 JVM 调度到少量载体线程（底层是 ForkJoinPool）。
- 阻塞 I/O 时 JVM 把虚拟线程从载体卸载、挂上另一个，载体永不阻塞。创建：`Executors.newVirtualThreadPerTaskExecutor()`。
- **单任务不会更快**（一次 HTTP 还是 100ms），快在并发规模 → 总吞吐。
- 类比：虚拟线程 ≈ goroutine，载体线程 ≈ Go 调度器的 P/M。
- 三要点：只用于 I/O 密集（CPU 密集无益还增开销）、不要池化（很便宜，每任务一个）、临界区用 ReentrantLock 防 pinning。
- 注意对下游限流：海量并发会压垮下游，需用 Semaphore 或有界池控制。
## 7. Pinning 钉住（Java 21 的限制，Java 24 修复）
 
**简易版：**
- 虚拟线程在 synchronized 块内或 native 调用里阻塞时，无法卸载，会把载体线程钉住，失去虚拟线程意义。
- 解法：I/O 密集的临界区改用 ReentrantLock，而非 synchronized。
**细节版：**
- 原因：synchronized 的 monitor 持有信息记在 native 栈、和载体 OS 线程绑定，阻塞时无法安全卸载（卸载后锁归属会乱），只能钉住载体。
- ReentrantLock 基于 AQS，锁状态是纯 Java 堆里的 state（int + CLH 队列 Node），不绑任何 OS 线程，虚拟线程可连栈一起正常卸载。
- JDK 24（JEP 491）已修复 synchronized pinning；但 JDK 21 上仍建议 I/O 临界区用 ReentrantLock。
## 8. 结构化并发（Java 21 起多轮预览）
 
**简易版：**
- 把一组相关并发任务当成一个整体管理。
- 一个失败或取消，兄弟任务一起取消，父任务等所有子任务完成。
- 好处：防线程泄漏，并发代码读起来像顺序代码。用 `StructuredTaskScope`。
**细节版：**
- 流程：fork 子任务 → join() 等全部完成 → throwIfFailed() 传播失败。
- 对比往 executor 乱 submit：一个任务失败，其他会变成孤儿线程继续跑浪费资源。
- 本质是给并发加了 try-with-resources 式的清晰生命周期边界。
## 9. 接口默认方法（Java 8）+ 多 default 冲突
 
**简易版：**
- default 方法让接口带默认实现，目的是不破坏已有实现类地给接口加新方法（典型：8 给 Collection 加 stream）。
- 单接口多 default 没问题。
- 多接口同名 default 是钻石问题，编译器无法裁决时报错，必须在实现类重写。
**细节版：**
- 冲突解决三规则（优先级从高到低）：
  1. 类优先于接口（能从父类继承的方法赢过接口 default）
  2. 子接口优先于父接口（更具体的赢）
  3. 平级无法裁决 → 实现类必须重写，用 `接口名.super.方法()` 指定用哪个父接口的实现。
## 10. 模块系统（Java 9）+ 友好 NPE（Java 14）
 
**简易版：**
- 模块系统（9）：用 `module-info.java` 声明依赖（requires）和暴露的包（exports），解决 JAR hell、实现强封装；企业普及度不高，知道概念即可。
- 友好 NPE（14）：精确报出链式调用里哪个变量、哪个方法返回值为 null，大幅缩短排查。
**细节版：**
- 模块系统三好处：强封装（没 exports 的包外部绝对访问不到，即使 public）、显式依赖（缺依赖启动就报错）、可裁剪 JDK（jlink）。
- 友好 NPE：`user.getAddress().getCity()` 里 getCity 返回 null 时，旧 NPE 只报行号，新 NPE 直接说"因为 Address.getCity() 返回 null"。
- 友好 NPE：JDK 15 起默认开启，14 需加 `-XX:+ShowCodeDetailsInExceptionMessages`。
---
 
# 二、并发
 
## 11. 线程池
 
**简易版：**
- 七参数核心：核心线程数、最大线程数、空闲存活时间、阻塞队列、拒绝策略。
- 执行流程四步：核心没满建核心线程 → 核心满进队列 → 队列满建临时线程 → 到 max 且队列满触发拒绝。
- 反直觉点：先进队列，队列满才建临时线程。
- 生产一定用有界队列，否则任务堆积 OOM，且 max 失效。
**细节版：**
- 无界队列（如不指定容量的 LinkedBlockingQueue）→ 队列永不满 → max 永不生效 → 退化成固定 core 大小。
- **4 拒绝策略：** AbortPolicy（抛异常，默认）、CallerRunsPolicy（提交线程自己跑 = 天然背压）、DiscardPolicy（静默丢）、DiscardOldestPolicy（丢队列最老的）。
- **队列选择：** ArrayBlockingQueue（数组有界，一把锁）、LinkedBlockingQueue（链表，不指定容量则无界，读写两把锁吞吐高）、SynchronousQueue（不存储、直接交接，Cached 池用它）、PriorityBlockingQueue（带优先级）。
- **禁用 Executors 原因：** Fixed/Single 用无界队列 → OOM；Cached/Scheduled 的 max 是 Integer.MAX_VALUE → 线程无限 → OOM。
- **线程数：** CPU 密集 ≈ 核数 + 1；I/O 密集 ≈ 核数 × 2 或 核数 ×（1 + 等待时间/计算时间）；公式只是起点，实际靠压测。
- **交易场景：** 订单任务不能丢用 AbortPolicy 或 CallerRunsPolicy；行情推送可用 DiscardOldest（旧价格作废）。
## 12. synchronized 原理
 
**简易版：**
- JVM 内置锁，靠对象头 Mark Word 记录锁状态，底层是 monitor。
- 代码块用 monitorenter/monitorexit，方法用 ACC_SYNCHRONIZED 标志。
- 可重入、非公平。
**细节版：**
- monitor（HotSpot 的 ObjectMonitor）四部分：`_owner`（持有线程）、`_count`（重入次数）、`_EntryList`（抢锁阻塞队列）、`_WaitSet`（wait 的线程）。
- 可重入靠 `_count`：同线程重入 +1，退出 -1，减到 0 才释放。
- wait/notify 操作的就是 `_WaitSet`，所以必须在 synchronized 块内调用（得先持有 monitor）。
- monitorexit 在异常时也会执行，保证锁一定释放。
- 重量级锁时 monitor 依赖 OS mutex，有用户态/内核态切换开销。
## 13. 锁升级
 
**简易版：**
- JDK 6 后 synchronized 有锁升级：无锁 → 偏向锁 → 轻量级锁 → 重量级锁，只升不降。
- 目的：低竞争时避免直接用重量级锁的开销。
- JDK 15 后偏向锁默认禁用并废弃，后彻底移除，现在是无锁 → 轻量 → 重量三级。
**细节版：**
- **偏向锁：** Mark Word 记住线程 ID，同线程再进直接放行，几乎零开销，适合无竞争。
- **轻量级锁：** 出现第二个线程竞争，撤销偏向锁升级；用 CAS + 自旋尝试，不阻塞，适合竞争不激烈、持锁短。
- **重量级锁：** 自旋超限还拿不到，升级；竞争线程阻塞挂起（进内核），适合竞争激烈。
- **移除偏向锁原因：** 实现和 HotSpot 耦合重、维护成本高；现代"名义同步实际单线程"场景变少收益下降；有竞争时撤销代价大反成负优化。
## 14. ReentrantLock
 
**简易版：**
- JUC 里基于 AQS 的显式锁。
- 比 synchronized 多四个能力：可中断、可超时、可公平、多条件变量。
- 代价：手动 lock/unlock，unlock 必须放 finally，否则异常时死锁。
**细节版：**
- 可中断：`lockInterruptibly()`，等锁时能响应中断，不死等。
- 可超时：`tryLock(3, SECONDS)`，拿不到就放弃，避免死锁。
- 可公平：构造传 true 变公平（按排队顺序给锁），默认非公平（吞吐高）。
- 多条件：一个锁 `newCondition()` 出多个 Condition，精准唤醒某组线程（生产者/消费者分开等待）。
- 虚拟线程 I/O 临界区首选它（不 pinning）。
## 15. AQS ⭐ 核心
 
**简易版：**
- AbstractQueuedSynchronizer，JUC 大部分同步工具的底层框架。
- 核心两样：一个 volatile int 状态 state + 一个 CLH 双向等待队列。
- 拿锁是 CAS 改 state，拿不到进队列排队并挂起。
- ReentrantLock、Semaphore、CountDownLatch 都基于它。
**细节版：**
- state 含义：ReentrantLock 里 0 = 无锁、>0 = 持有（重入几次就是几）；Semaphore 里是许可数。
- CLH 队列：抢不到锁的线程包成 Node 入 FIFO 双向链表，`LockSupport.park()` 挂起；前驱释放时 `unpark()` 唤醒后继。
- 非公平锁：上来先 CAS 抢一把（不管队列有没有人），抢不到才排队 → 可能插队、吞吐高。
- 公平锁：先检查队列有没有人排队，有就排队尾。
- 模板方法模式：AQS 定框架（入队、挂起、唤醒），tryAcquire/tryRelease 交子类实现。
## 16. CAS
 
**简易版：**
- Compare-And-Swap，一条 CPU 原子指令：比较内存值和预期值，相等才更新，不等就失败。
- 无锁并发基础，失败自旋重试；AtomicInteger、AQS 都基于它。
- 我在日志平台 bloom 过滤器用过：bit 数组底层用 int 数组，置位用 AtomicInteger 的 CAS，锁粒度降到单个 int，且分词后 key 重复率高竞争小。
- 三缺点：ABA、自旋占 CPU、只保证单变量原子。
**细节版：**
- 三操作数：内存位置 V、预期旧值 A、新值 B，V==A 才把 V 改成 B；x86 底层是 cmpxchg 指令。
- 自旋写法：`do { old = get(); } while(!compareAndSet(old, old+1));`。
- ABA：值 A→B→A，CAS 以为没变过；解法 `AtomicStampedReference` 加版本号。
- 自旋开销：竞争激烈时一直失败空转占 CPU。
- 只保证单变量：多变量复合操作管不了；解法 `AtomicReference` 把多个变量包成一个对象，或退回加锁。
## 17. ConcurrentHashMap
 
**简易版：**
- 1.7 用分段锁 Segment，默认 16 段每段一把锁，并发度 = 段数。
- 1.8 抛弃 Segment，改数组 + CAS + synchronized：桶为空 CAS 直接放入，桶非空 synchronized 只锁那个桶的头节点，粒度细化到单桶，并发度更高。
**细节版：**
- 1.8 为什么改：分段锁粒度粗、Segment 结构占内存；此时 synchronized 已有锁升级优化，不比 ReentrantLock 差。
- 不允许 null key/value：并发下无法区分"值是 null"和"key 不存在"，会歧义。
- size() 弱一致：用 baseCount + CounterCell 分散计数减少竞争。
- **并发扩容（亮点）：** 有全局 transferIndex 指针，多线程用 CAS 各领一段桶（默认 16 个/段，即 stride）并行迁移；迁移完的桶放 ForwardingNode（hash=-1）标记；其他线程 put 时碰到 ForwardingNode 就加入帮忙迁移，读碰到就顺着 fwd 去新表查。
## 18. volatile + JMM
 
**简易版：**
- 保证可见性和有序性，不保证原子性。
- 可见性：写完立即刷回主存 + 让其他线程缓存失效。
- 有序性：内存屏障禁止指令重排。
- 不保证原子性：i++ 是读、改、写三步，volatile 挡不住中间被打断。
**细节版：**
- **JMM 背景：** 每个线程有工作内存（对应 CPU 缓存），共享变量在主存，"拷到工作内存→改→写回"的模型导致可见性问题。
- **有序性经典应用 DCL 单例：** `instance = new Singleton()` 底层是"分配内存→初始化对象→赋引用"三步，不加 volatile 可能重排成先赋引用（instance 非 null 但对象是半成品），别的线程在锁外第一个 if 读到非 null 直接用 → 拿到半成品。volatile 禁止此重排。
- **不保证原子性举例：** 两个线程同时对 volatile i 做 i++，都读到同一旧值各自 +1，结果只 +1。要原子用 AtomicInteger 或加锁。
- **happens-before 关键规则：** 程序顺序、volatile 写 hb 后续 volatile 读、unlock hb 后续 lock、传递性（A hb B, B hb C ⇒ A hb C）。
## 19. volatile 实现层次（深挖）
 
**简易版：**
- 实现横跨三层。
- 源码层：加 volatile 关键字。
- 字节码层：只体现为字段的 ACC_VOLATILE 标志，读写指令和普通变量完全一样（都是 getfield/putfield），没有屏障。
- 机器码层：JIT 编译时才按 ACC_VOLATILE 标志插入内存屏障——所以屏障不是字节码，是机器码层面的东西。
**细节版：**
 
**四种屏障：** LoadLoad、StoreStore、LoadStore、StoreLoad（最重）。volatile 写前插 StoreStore、后插 StoreLoad；volatile 读后插 LoadLoad + LoadStore。
 
**写前读后规则（配对才成立）：** 被保护的普通变量，写侧必须放在 volatile 写**之前**，读侧必须放在 volatile 读**之后**，且读到 volatile = true，才保证读到最新值。代码例子：
 
```java
class Worker {
    private int data = 0;                     // 普通变量
    private volatile boolean ready = false;   // volatile 标志位
 
    // 写侧：普通变量 data 写在 volatile 写 ready 【之前】
    void producer() {
        data = 42;        // (1) 普通写
        ready = true;     // (2) volatile 写（前面的 StoreStore 屏障保证 data=42 先完成并刷主存）
    }
 
    // 读侧：普通变量 data 读在 volatile 读 ready 【之后】
    void consumer() {
        if (ready) {                  // (3) volatile 读
            System.out.println(data); // (4) 读到 ready=true 后再读 data，保证是 42
        }
    }
}
```
 
- 若把 `data = 42` 写在 `ready = true` **后面** → 不受保护，可能重排/延迟刷出。
- 若读侧在读 `ready` **之前**就读了 `data` → 那次读不在屏障保护内，可能是旧值 0。
- 本质（happens-before）：volatile 写之前的所有写，必须在该 volatile 写生效前完成并可见；读到该 volatile 的线程，能看到这些写。
**底层落地（x86）：**
- x86 是强内存模型，LoadLoad/StoreStore/LoadStore 硬件天然保证，基本是空操作。
- 唯一需真正落地的是 StoreLoad：用 `lock` 前缀指令（如 `lock addl $0,(%rsp)`）实现——① 排空整个 store buffer（把之前的普通写连同 volatile 写一起刷到缓存/主存）② 触发 MESI 让其他核缓存行失效 ③ 作为全屏障禁止重排。
- 所以 x86 上 volatile 写贵、读几乎免费；ARM 弱内存模型读写两侧都要真实屏障指令（dmb）。
**为什么 ready 写时 data 已写：** ① StoreStore 保证 data 的写指令先于 ready 执行、先进 store buffer；② store buffer 按 FIFO 刷出 + lock 排空，data 先于 ready 到主存。关键认知：指令"执行顺序"对 ≠ 对其他线程"可见顺序"对，元凶是 store buffer 异步刷出，屏障管的是可见顺序。
 
## 20. ThreadLocal
 
**简易版：**
- 给每个线程一份独立变量副本，实现线程隔离。
- 每个 Thread 内有 ThreadLocalMap，key 是 ThreadLocal 对象（弱引用），value 是值。
- key 用弱引用防 ThreadLocal 无法回收；但 value 是强引用仍可能泄漏，用完必须 remove（线程池尤甚）。
**细节版：**
- 数据不存在 ThreadLocal 里，存在每个 Thread 自己的 ThreadLocalMap；`get()` 是"拿当前线程的 map，以这个 threadLocal 为 key 取值"，天然隔离。
- 内存泄漏根源：key 被回收后变 null，但 value 强引用还在，线程（如池化线程）一直活着 value 就一直泄漏；get/set 会顺带清理 key=null 的 entry 但不彻底 → 必须 remove。
- **数据库连接场景：** Connection 有事务和内部状态、非线程安全、不能多线程共享；ThreadLocal 保证每线程一个独立连接，且让同一线程整条调用链共用一个连接——这是 Spring 声明式事务能"一个请求一个事务"的底层基础。连接本身来自连接池，用完归还。
---
 
# 三、集合与 JVM
 
## 21. HashMap
 
**简易版：**
- 数组 + 链表 + 红黑树。
- 链表长 > 8 且数组容量 ≥ 64 转红黑树，节点数 < 6 退回链表。
- 默认容量 16，负载因子 0.75，超阈值扩容为两倍。
- 线程不安全：1.7 头插并发扩容会死循环，1.8 尾插解决死循环但仍可能丢数据。
**细节版：**
- put：hash 定位桶 → 空则直接放 → 非空遍历，key 相同覆盖、不同则尾插（1.8）；超阈值扩容。
- 扩容优化（1.8）：元素要么留原位置，要么移到"原位置 + 旧容量"，靠 hash 与旧容量做与运算判断，不用重新算 hash。
- 容量取 2 的幂：让 `hash & (n-1)` 等价于取模，又快又分布均匀。
- 线程不安全细节：1.7 头插 + 并发扩容 → 链表成环 → 死循环；1.8 尾插解决死循环，但并发 put 仍可能覆盖丢数据。并发用 ConcurrentHashMap。
## 22. JVM 内存区域 + 类加载
 
**简易版：**
- 内存五块：堆（对象）、方法区/元空间（类信息）、虚拟机栈（栈帧）、本地方法栈（native）、程序计数器（执行位置）。
- 堆和方法区线程共享，栈和程序计数器线程私有。
- 类加载五步：加载 → 验证 → 准备 → 解析 → 初始化，双亲委派保证核心类不被篡改。
**细节版：**
- 堆：GC 主战场，分新生代（Eden + 2 Survivor）和老年代。
- 虚拟机栈：每个方法调用一个栈帧，栈溢出报 StackOverflowError；程序计数器是唯一不会 OOM 的区域。
- 准备阶段：静态变量赋**默认值**（int → 0，不是代码里的初值）；初始化阶段才执行静态代码块、赋真正的值。
- **元空间：** JDK 8 把类元信息从"永久代（堆的一部分、有固定上限 MaxPermSize、易 OOM PermGen）"移到"本地内存"。
- **本地内存：** 直接向 OS 申请、不归 JVM 堆管、不受 -Xmx 限制的堆外内存，上限约为物理内存。
- **永久代/元空间也会 GC：** 回收废弃常量和无用类；卸载类条件极严（该类所有实例已回收 + 加载它的 ClassLoader 已回收 + Class 对象无引用），平时几乎不回收。
- **双亲委派：** 请求先交父加载器（Bootstrap → Extension → Application），父加载不了才自己加载；防止核心类被篡改（自己写的 java.lang.String 加载不了）。
## 23. JVM GC
 
**简易版：**
- 分代回收：新生代复制算法（Minor GC），老年代标记-整理（Major/Full GC）。
- 存活判断用可达性分析：从 GC Root 出发找引用链，到不了的回收。
- 收集器：G1 现代默认（分 Region、可设停顿目标），ZGC/Shenandoah 低延迟（亚毫秒）。
- 交易系统对延迟敏感，核心链路优先用 ZGC 压低 GC 抖动。
**细节版：**
- 可达性分析不用引用计数（解决不了循环引用）；GC Root 包括栈引用、静态变量、常量等。
- 新生代：对象朝生夕死用复制算法（Eden + 两 Survivor，存活的复制到另一块）；熬过约 15 次 GC 晋升老年代，大对象直接进老年代。
- 收集器演进：CMS（并发标记清除，有碎片、Full GC 兜底问题，JDK 14 移除）→ G1（JDK 9+ 默认）→ ZGC / Shenandoah（超低停顿）。
**G1 原理与调优：**
- 原理：把堆分成很多大小相等的 Region（不再是物理连续的分代，逻辑上分 Eden/Survivor/Old）；每次优先回收垃圾最多的 Region，所以叫 Garbage First；用可预测停顿模型，按停顿目标挑本次回收哪些 Region。
- 大对象放专门的 Humongous Region。
- 调优：核心是设停顿目标 `-XX:MaxGCPauseMillis`（默认 200ms），G1 自己权衡回收多少 Region 来逼近这个目标；`-XX:G1HeapRegionSize` 调 Region 大小；停顿目标设太小会导致每次回收太少、GC 更频繁、吞吐下降，需权衡。
- 定位：适合大堆、要兼顾吞吐和停顿的通用场景。
**ZGC 原理与调优：**
- 原理：目标停顿 < 1ms 且几乎不随堆增大而增加；靠**染色指针**（把标记信息存在指针的空闲位里）+ **读屏障**（应用读引用时顺带做重定位/标记），把标记和转移几乎全部并发化，STW 只剩极短的根扫描。
- 版本：JDK 11 实验、15 转正生产可用、21 引入分代 ZGC（`-XX:+ZGenerational`，大幅提升）；**至今默认仍是 G1**，ZGC 需手动 `-XX:+UseZGC`。
- 调优哲学：少调参。核心是给足 `-Xmx`——ZGC 并发回收靠"堆里有富余空间"支撑边分配边回收，堆太小会触发分配停顿（allocation stall）；`-XX:SoftMaxHeapSize` 设软上限；回收跟不上分配可调 `-XX:ConcGCThreads`（但抢应用 CPU 要权衡）。
- 没有 MaxGCPauseMillis 可调，因为停顿本就恒定亚毫秒、与堆大小无关。
- 定位：大堆、低延迟场景（如交易撮合/下单核心）。
**交易场景选型：** 核心低延迟链路用 JDK 21 分代 ZGC（堆给足、避免分配停顿），吞吐优先的离线任务用 G1。
 
---
 
# 四、Rust（简历 Solana 支撑，够用即可）
 
## 24. 何时选 Rust
 
**简易版：**
- 核心价值：用编译期所有权检查，换来没有 GC、性能接近 C++、又不会有 C++ 那类内存安全漏洞。
- 三类场景：延迟敏感不能有 GC 停顿的系统（高频交易、数据库引擎）；系统级编程（OS、驱动、嵌入式）；区块链智能合约和 WebAssembly。
- 代价：学习曲线陡、开发慢，迭代快的 CRUD 后端不选它。
**细节版：**
- 空白定位：C/C++ 性能极致但手动管内存易出漏洞；Java/Go 有 GC 安全但有停顿和运行时开销；Rust 要"C++ 性能 + 内存安全"，把检查放编译期。
- 其他典型场景：给 Python/Node 写高性能 native 扩展（如 ruff、pydantic-core）。
- 不该选：性能无极致要求（过度工程）、团队无 Rust 经验招人难。
## 25. 为什么无 GC
 
**简易版：**
- 把内存回收时机在编译期就确定。
- 所有权规则：每块内存有且只有一个所有者，所有者离开作用域内存立即自动释放（编译器插入释放代码，类似 RAII）。
- 所有权关系编译期就清晰，编译器精确知道释放点，运行时不需要 GC 扫描追踪。
**细节版：**
- 对比：C++ 手动 free、Java/Go 运行时 GC 扫描、Rust 编译期所有权。
- GC 是"运行时不知道何时释放，派后台管家去查"；Rust 是"编译期算清释放点，把释放代码编译进去"。
- 例外机制：需共享所有权/运行时才知生命周期时，用 `Rc`（引用计数）/ `Arc`（原子引用计数）——是引用计数不是追踪式 GC，且显式选用，所以说"默认不需要 GC"精确。
## 26. 借用检查器
 
**简易版：**
- 编译期静态分析，强制两条规则：一是读写互斥（要么多个只读引用，要么一个可写引用）；二是引用不能比它指向的数据活得久。
- 在编译期就证明没有数据竞争和悬垂引用，这就是 fearless concurrency。
**细节版：**
- 规则一杜绝数据竞争：写时必然独占（无别人在读），且检查在编译期做，不靠运行时加锁。
- 规则二杜绝 use-after-free：靠生命周期分析确认引用使用时数据还活着。
- 底层机制：生命周期标注（'a）+ NLL（基于控制流看引用的实际最后使用点，而非大括号）+ move 检查（值 move 走后再用原变量报错）。
- 代价：很多别的语言随手能写的模式（如两处持有同一可变引用）在 Rust 编译不过，得换 `Rc<RefCell>`、`Arc<Mutex>`；这是学习曲线陡的主因。

### 阿里巴巴 java开发手册注意事项
* 如果使用 tab 缩进，必须设置 缩进，必须设置 缩进，必须设置 缩进，必须设置 缩进，必须设置 缩进，必须设置 1个 tab 为 4个空格。 IDEA 设置 tab 为 4个空格时， 请勿勾选 Use tab character ；而在 eclipse 中，必须勾选 insert spaces for tabs 。
* 【强制】POJO类中布尔类型的变量，都不要加is，否则部分框架解析会引起序列化错误。
* 【强制】所有的覆写方法，必须加@Override注解。
* 【强制】Object的equals方法容易抛空指针异常，应使用常量或确定有值的对象来调用equals。推荐使用java.util.Objects#equals （JDK7引入的工具类）
* 【强制】所有的相同类型的包装类对象之间值的比较，全部使用equals方法比较。
* 【强制】关于基本数据类型与包装数据类型的使用标准如下： 1） 所有的POJO类属性必须使用包装数据类型。 2） RPC方法的返回值和参数必须使用包装数据类型。 3） 所有的局部变量【推荐】使用基本数据类型。
* 【强制】POJO类必须写toString方法。使用IDE的中工具：source> generate toString时，如果继承了另一个POJO类，注意在前面加一下super.toString。 说明：在方法执行抛出异常时，可以直接调用POJO的toString()方法打印其属性值，便于排查问题。
* 【推荐】 类内方法定义顺序依次是：公有方法或保护方法 > 私有方法 > getter/setter方法。
* 【强制】使用工具类Arrays.asList()把数组转换成集合时，不能使用其修改集合相关的方法，它的add/remove/clear方法会抛出UnsupportedOperationException异常。可以使用 List list = new ArrayList(Arrays.asList(array));
* 【强制】不要在foreach循环里进行元素的remove/add操作。remove元素请使用Iterator方式，如果并发操作，需要对Iterator对象加锁。
* 【推荐】使用entrySet遍历Map类集合KV，而不是keySet方式进行遍历。
* 【强制】线程池不允许使用 【强制】线程池不允许使用 Executors ,而是通过ThreadPoolExecutor
* 【推荐】使用CountDownLatch进行异步转同步操作，每个线程退出前必须调用countDown方法，线程执行代码注意catch异常，确保countDown方法可以执行，避免主线程无法执行至countDown方法，直到超时才返回结果。 说明：注意，子线程抛出异常堆栈，不能在主线程try-catch到。
* 【强制】在表查询中，一律不要使用 * 作为查询的字段列表，需要哪些字段必须明确写明。 说明：1）增加查询分析器解析成本。2）增减字段容易与resultMap配置不一致。
* 【强制】使用ISNULL()来判断是否为NULL值。注意：NULL与任何值的直接比较都为NULL。 说明： 1） NULL<>NULL的返回结果是NULL，而不是false。 2） NULL=NULL的返回结果是NULL，而不是true。 3） NULL<>1的返回结果是NULL，而不是true。
* 【推荐】所有pom文件中的依赖声明放在<dependencies>语句块中，所有版本仲裁放在<dependencyManagement>语句块中。意思是父pom在dependencyManagement中声明依赖和对应的版本, 子pom只需要引入各自实际需要的dependency而不需要再写版本了. 说明：<dependencyManagement>里只是声明版本，并不实现引入，因此子项目需要显式的声明依赖，version和scope都读取自父pom。而<dependencies>所有声明在主pom的<dependencies>里的依赖都会自动引入，并默认被所有的子项目继承。


### java集合
#### List
* 为什么例如 ArrayList, 单线程在for 循环remove 的时候会报ConcurrentModifcationEx
因为 remove 的时候会把remove 节点之后的元素都全部arrayCopy 前移一位, 导致下次for 循环会少遍历一个节点, 传统的remove(index) 和remove(obj) 只用于remove 单个节点的情况, 所以需要iterator 去remove, itr 有两个指针, a 一个指向上一次返回的节点index, b 一个指向下一个返回的index, 所以remove arraycopy 之后, 将b 往前移一位就可以了. 

* ArrayList与LinkedList区别
前者更适合随机下标遍历, O(1)的时间复杂度; 后者更适合增删,因为前者需要数组的拷贝,效率比较低,可以当作堆栈、队列和双向队列使用。
* ArrayList与Vector的区别
Vector是线程安全的, 进而效率比较低, 在方法上加了对象锁
* Stack
继承了Vector, 添加了Pop和Push等几个方法, 是线程安全的

* LinkedHashSet 可以说是 LinkedHashMap按照插入顺序排序的特例, 通过维护一个双端链表, 保证插入顺序和遍历顺序一致. 
* TreeSet 是 TreeMap value都相同的特例. 内部是一个二叉搜索树, 是排序的
* HashMap
只能有一个null的key, 可以有多个null的value; hashMap的不是有序的, 

* LinkedHashMap是有序的,可以按照元素插入和访问的顺序排序, LinkedHashMap 的访问序可以方便地用来实现一个 LRU(least and recently used) Cache。在访问序模式下，尾部节点是最近一次被访问的节点 (least-recently)，而头部节点则是最远访问 (most-recently) 的节点。因而在决定失效缓存的时候，将头部节点移除即可。
* TreeMap基于Compartor的排序. 红黑树








### 方法重载与重写

* 重载(overload)

  * 指的是方法同名,但形参列表不同
* 重写(override)

  * 指的父子类间,子类重写父类的方法,并覆盖.(除非显示声明super.methodName(),否则调用子类的方法)
  * 子类的访问修饰权限不能小于父类
  * 要使父类的方法不被override,声明为final或者private

## concurrency
* 方法

  * extends Thread
  * implements Runnable
  * Executors
     
    Executors allow you to manage the execution of asynchronous tasks without having to explicitly manage the lifecycle of threads. Executors are the preferred method for starting tasks in Java SE5/6.
* 从task返回值
    
    If you want the task to produce a value when it’s done, you can implement the **Callable** interface rather than the Runnable interface. Callable, introduced in Java SE5, is a generic with a type parameter representing the return value from the method call( ) (instead of run( )), and must be invoked using an ExecutorService submit( ) method

* daemon
    
    A "daemon" thread is intended to provide a general service in the background as long as the program is running, but is not part of the essence of the program. Thus, when all of the non-daemon threads complete, the program is terminated, killing all daemon threads in the process. Conversely(相反的), if there are any non-daemon threads still running, the program doesn’t terminate. There is, for instance, a non-daemon thread that runs main( ).

    A daemon thread is a thread that does not prevent the JVM from exiting when the program finishes but the thread is still running. An example for a daemon thread is the garbage collection.
* join

    One thread may call join( ) on another thread to wait for the second thread to complete before proceeding. If a thread calls t.join( ) on another thread t, then the calling thread is suspended until the target thread t finishes (when t.isAlive( ) is false).
    
### memory
* Difference between “on-heap” and “off-heap”
    
    The on-heap store refers to objects that will be present in the Java heap (and also subject to GC). On the other hand, the off-heap store refers to (serialized) objects that are managed by EHCache, but stored outside the heap (and also not subject to GC). As the off-heap store continues to be managed in memory, it is slightly slower than the on-heap store, but still faster than the disk store

* heap and stack
    * stack is a data structure. LIFO.
    * stack is used to store primitive data type and function call.but heap is to store object.
    * If there is no memory left in the stack for storing function call or local variable, JVM will throw java.lang.StackOverFlowError, while if there is no more heap space for creating an object, JVM will throw java.lang.OutOfMemoryError: Java Heap Space
    *　Variables stored in stacks are only visible to the owner Thread while objects created in the heap are visible to all thread. In other words, stack memory is kind of private memory of Java Threads while heap memory is shared among all thread


### initialize and cleanup
*  constructor

    一旦自定义了构造函数,则默认的构造函数不会被隐式调用.
    ```
     new ClassName()
    ```
    compiler会不懂调用的是哪一个构造函数,必须显示调用.
    

* constructor is atomic or not
    
    构造函数是不具有synchronized的性质,在构造函数执行过程中,对象是对其他线程可见的,而且因为
```
objectA = new ClassA();
```
在不使用volatile的情况下,会被reorder,所以构造函数应该需要手动同步.

* static construct
    * static修饰的field属于某个类(class),而不是类的某个实例(instance),类的所有实例共用一个static 变量,在类初始化之前被初始化(类被调用的时候),只被初始化一次,如果没有明显的赋值,则会被赋予默认值.
    * The JVM won't execute a class's static initializer until you actually touch something in the class
    * Static variables are initialized only once , at the start of the execution 
    * 顺序: static field -> non static field -> class constructor,若有父类则父类优先

**To summarize the process of creating an object, consider a class called Dog:**

* 代码分析一


```
class Foo {
  private volatile Helper helper = null;
  public Helper getHelper() {
    if (helper == null) {
      synchronized(this) {
        if (helper == null) {
          helper = new Helper();
        }
      }
    }
  return helper;
}
```
 * 要加volatile, 因为第一个判断是否为空在synchronize的外面, 若不加volatile,各自线程有自己的缓存,可能导致数据不一致,没初始化完成就被使用的情况.
 * 第一个判断是否为空,是为了提升性能,如果不加,每次多线程调用的时候都会有可能产生线程间的等待,降低性能.
 * 第二个判断是否为空,是因为一开始instance为空,有多个线程会进入synchronize块,但是只有一个线程会完成instance的实例化,当这个线程完成实例化后,其他线程不能在对instance进行初始化,所以需要加第二个判断非空.
 * 较好的方法(lazy initialization)
```
public class Something {
    private Something() {}

    private static class LazyHolder {
        private static final Something INSTANCE = new Something();
    }

    public static Something getInstance() {
        return LazyHolder.INSTANCE;
    }
}

```
  Since the class initialization phase is guaranteed by the JLS to be serial, i.e., non-concurrent, so no further synchronization is required in the static getInstance method during loading and initialization
* 代码分析二
```
class MyClass {
  private static MyClass myClass = new MyClass();
  private static MyClass myClass2 = new MyClass();
  public MyClass() {
    System.out.println(myClass);
    System.out.println(myClass2);
  }
}
```
That will print:
```
null
null
myClassObject
null
```
    
  because it divide into two steps,first is initializaion,second is assignment.when the first initialzation is ended,the myClass is not null,but not until the  assignment to myClass2 is done,the myClass2 is  instantiated.

* this
    
    * Suppose you’re inside a method and you’d like to get the reference to the current object. Since that reference is passed secretly by the compiler, there’s no identifier for it. However, for this purpose there’s a keyword: this. The this keyword—which can be used only inside a non-static method—produces the reference to the object that the method has been called for
    * The this keyword is used only for those special cases in which you need to explicitly use the reference to the current object. For example, it’s often used in return statements when you want to return the reference to the current object


### null pointer exception
* 当使用一个reference的时候,但是没有指向任何object,或者指向的object出错,实际为null的时候.
> The best way to avoid this type of exception is to always check for null when you did not create the object yourself." If the caller passes null, but null is not a valid argument for the method, then it's correct to throw the exception back at the caller because it's the caller's fault

### hashmap

![enter image description here](https://drive.google.com/uc?id=1CliBbv1YMdfPtT7NwlZwoxL6MUf_MRmj)
* 一些基础概念
有一个table 数组, 数组的每个槽作为一个bucket, 然后数组的元素可能是一个链表的node ,也可能是一个 红黑树的node. 取决于jdk 的版本和一个bucket 里面entry的数量; 每个key 和val 组成一个entry. 

初始容量 和 负载因子，这两个参数是影响HashMap性能的重要参数。其中，容量表示哈希表中桶的数量 (table 数组的大小)，初始容量是创建哈希表时桶的数量；负载因子是哈希表在其容量自动增加之前可以达到多满的一种尺度，它衡量的是一个散列表的空间的使用程度，负载因子越大表示散列表的装填程度越高，整个hashmap 空间需要的更少, 但是查找时间会增加, 反之愈小。默认的, 当初始容量 capacity(默认16 ) * load factor (0.75 )>  enrty的数量的时候, 会认为需要进行table 数组的扩容了. 当初始容量不是2的n 次方的时候, 会选择比它大的, 但是最小的2的n 次方作为数组的初始容量. 所以当entry 的最大数量小于容量 * 负责因子的时候, 就永远不会进行rehash 

* 负载因子是怎么算出0.75 的
根据这个问题的第三个答案 https://stackoverflow.com/questions/10901752/what-is-the-significance-of-load-factor-in-hashmap, 
公式的原理应该是:  size 为s , 已经有了n 个entry, 当下一个entry 进来的时候, 如何能最小化碰撞的概率, 并且此时空间利用率最大, 然后那个公式给出是n/s 小于 log2 的时候, 碰撞的概率都很小, 然后近似取了个比较容易算的值: 0.75

* 为什么当链表长度大于等于8 的时候, 链表转化为红黑树
这种情况用于hashcode 分布性很差的时候, 出现了很多个hash 冲突的极端情况, 因为当负载因子等于0.75 的时候, list 中元素的个数符合泊松分布: 
https://www.zhihu.com/question/26441147
参见代码注释, 所以出现list size 为8 的情况是极为少见的, 就算出现这个情况, 将链表转化为红黑树, 查找的平均时间复杂度由O(n) 变为O(logn) , 但是需要近似两倍的空间. 

* HashMap 的底层数组长度总是2的n次方的原因有两个

一是当 length=2^n 时：h&(length - 1)  在随机hash 的情况下, 能均匀分布到各个index , 
https://blog.csdn.net/claram/article/details/77750899

假定 length = 50 （非 2 的整数次幂），二进制值为 0011 0010，这里我们使用 8 位二进制数来进行计算。length - 1 = 49，二进制值为 0011 0001。我们计算任何整数与 49 进行与运算的可能的结果如下：
```
0000  0000  //0  
0000  0001  //1  
0001  0000  //16  
0001  0001  //17 
0010  0000  //32  
0010  0001  //33  
0011  0000  //48  
0011  0001  //49
相当于为1的各个位置的排列组合, 
```
可能的结果值为：0、1、16、17、32、33、48、49，对于一个长度为 50 的数组，我们只命中了其中的 8 个index. 

假定 length = 16，length - 1 = 15，二进制值为 0000 1111。我们计算任何整数与 31 进行与运算的可能的结果如下：

```
0000  0000  //0  
0000  0001  //1  
0000  0010  //2  
0000  0011  //3  
0000  0100  //4  
0000  0101  //5  
0000  0110  //6  
0000  0111  //7  
0000  1000  //8  
0000  1001  //9  
0000  1010  //10  
0000  1011  //11  
0000  1100  //12  
0000  1101  //13  
0000  1110  //14  
0000  1111  //15
```
每个 index 都用到了, 均匀分布. 


* null 的key 放在数组的第一位
hashmap key和value都可以为null, 因为key 为null, 则hash值为0, hash& table.length -1 均为0, 所以key 为null的object 放在table[0], val覆盖.



* 解决hash冲突的方法
    * 开放地址法(例如 ThreadLocalMap), 冲突时,往下遍历, 直到找到null的位置; 缺点: 容易产生元素的堆聚, 因为较长的链总是更容易产生冲突, 从而元素会落到链的末尾.
    * 拉链法(HashMap)
    
    
    * 两个方法的优劣, 为什么ThreadLocalMap 采用开放地址法

    因为threadLocal的应用场景决定了数据量并不大, 采用开放地址法, 并采用Fibonacci hashing, 使hash 分布均匀, 在小数据量的时候存取会很快.
  
* hashcode 
作为一个native 的方法, 有以下三点规范, 来自代码注释
1. 当对一个java application 的一次调用中, 必须返回同一个整型, 提供的信息也用于对象的相等比较，且不会被修改,但是在两次调用中, 不必返回相同的值
2. 如果两个对象使用equal 方法区判断是否相等, 则调用hashCode 必须返回相同的结果
3. 如果两个对象使用equal 方法比较是不相等的, 则调用hashcode 方法必须返回不同的结果. 开发者需要意识到对unequal 的对象产生不同的hashcode 有利于提高hashtable 的性能. 

* jdk1.8 hash 方法
```
static final int hash(Object key) {  
    int h;  
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  
}
```
讲原 int32 的 hashcode 无符号右移 16 位在与原 hashcode 异或, 等于说高 16 位不变, 低 16 位是低十六位与高十六位的异或的结果, 因为某些 size 比较小的 map, 例如 size=8, size-1=7, 二进制: 0000 0111, hashcode & (size-1), 只用了低三位, 大大增加了冲突的概率, 

* java 7 和java 8的hashmap 的主要优化在于
https://juejin.im/post/5aa5d8d26fb9a028d2079264#heading-20

1. 定位到数组的位置之后, 在链表往后进行一个个查找的时候, 当链表长度大于8 时, java 8会转化为红黑树, 将查找的时间复杂度O(n), 降低为O(logn)
2. hashcode 函数不同, java 7 中进行了多次与和异或运算, 8 均降低为1次, 边际递减. 
3. resize 的时候, 8 不会重新计算hash 值, 根据hash & oldCapacity == 0 ? 放在元数组位置: 否则放在oldIndex+oldCapacity 的位置. 


* put 大致过程
根据hash, 找到需要存放的entry 数组的index, 如果该位置没元素, 就直接放入, 如果有, 则判断该位置是不是红黑树的节点, 是的话就调用红黑树的插入方法(默认链表长度大于8, 则会变为红黑树), 否则遍历链表, 放到链表末尾, 如果插入后, 链表长度大于等于9, 则将该链表转化为红黑树, 最后如果插入后, hashMap size大于threshold, 则进行resize. 

* resize 的大致过程
resize 成原来容量的两倍, 如果原先数组的位置只有一个元素, 就直接取模迁移到新数组的位置, 如果是红黑树则是调用红黑树的分裂方法, 如果是链表, 则分成两个子链表, 一个放在原位置, 另一个放在 (原位置+oldCapacity).

1. 返回bucket的index
 ``` 
int indexFor(int hash,int length)
  return hash & (length-1)
```
比取余数(hash % length)的方法更加的快,

* hashmap线程不安全的表现
> 多线程put的情况下, 同时进入了rehash, 出现死循环; 同时在遍历map的时候, 如果有其他线程改变了map的结构, 会抛出ConcurrentModificationException, 是fail-fast的策略, 目的是为了提醒线程安全的问题. 

* 为什么String, Interger这样的wrapper类适合作为键
> 保证key是不可变的, 具体说来就是hashCode不能变, 作为键的对象, 需要重写hashCode和equal方法, 保证能正常获取到之前的对象. 

* HashMap, ConcurrentHashMap,HashTable 的结构，在JDK 1.7 和1.8 中有什么不同

* put时，是加到链表头还是链表尾
在jdk1.8之前是插入头部的，在jdk1.8中是插入尾部的


## 泛型
1.使用原生态类型
> 为了兼容老的jdk1.5之前的Collection的代码,仅有这个不是类型安全的,编译期不能保证类型安全,可能在运行期出现,例如可能将java.util.Date插入到java.sql.Date中,很难发现

## inner class
* inner class cannot be inited outside the parent class scope, except inner class is static


### difference between static class and singleton
* singleton can 实现多态,继承或者实现接口,但是static不能
* 在spring框架中, singleton可以被DI, 但是static不能autowire
```
// 实在要用, 也可以这样注入static field
private static Class a;
@Autowired
public void setClass(A a){
    TheClass.a = a;
}
```
反正最好用singleton

### compression 压缩
* lz4
[lz4 stream example](https://stackoverflow.com/questions/36012183/java-lz4-compression-using-input-output-streams)

#### String 注意事项

![enter image description here](https://drive.google.com/uc?id=1Vj5wcAnfdZUdF-p9lH-BScBprXBJM0_7)
https://juejin.im/entry/5a4ed02a51882573541c29d5

>  String 为什么是Immutable 不可变的
1. 为了安全性, String 作为网络连接和数据库连接的参数, 防止被修改
2. 用作hasheSet 等的key, 保证了key 的唯一性

* 什么是Immutable  类
简而言之对象的状态一旦初始化之后就是不可变的, 由以下几个直接的现象: 一是final 不能被继承, 不能被子类所修改; 二是每次都返回一个新的对象, 三是无需要多线程的同步 
> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbNjI5MjkxNDQ0LDIwNjgzNTY1NCwxODQ2Mj
IxMzUwLDM0NzA5Nzg0NywtMTQ1OTMzOTIwNCwyMTMyNzI1NSwt
MTg2NTkxMTc4MywxMzk5Mzc1NzgsMTI1NTY4MTMxMSwtNTc1ND
kxNjQ5LC05NzU5NjQzOTksLTExNzkzMTIwODQsLTIwMTM5Mjc0
MzldfQ==
-->
