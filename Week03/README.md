### Go Concurrency

Go使用Goroutine、Chan、Sync包以及Context来解决Go并发中的问题

go并发使用通讯来解决内存共享
Do not communicate by sharing memory; instead, share memory by communicating

goroutine方便进行任务拆分

chan用来进行消息通讯

sync包来解决同步问题

context使用COW来解决并发goroutine可能引起的变量共享同步问题

#### goroutine使用实践

对所有的goroutine的生命周期都要管住
1.goroutine能不能安全退出
2.如何让goroutine安全退出，几种做法：1.通过shutdown安全退出，2.通过Context超时退出，3.通过chan发个信号close也可以退出--------如果没搞清楚goroutine什么时候会结束，否则就不要启动goroutine
3.把并行调用交给调用者

log.Fatal调用os.Exit,会无条件终止程序，defers都不会被调用到，所以一般在main和init函数执行

一定要做代码超时控制

#### go内存模型
并发的问题：
1.有序性  -------  编译器和内存重排续，导致有序性问题
2.原子性  -------  线程切换导致的原子性问题
3.可见性  -------  缓存导致可见性问题

Happen-Before规则

#### Sync包
Mutex和Atomic包
写入单个machine word是原子的
interface内部是两个machine word

COW: Copy-On-Write
全量复制一个新的对象，添加上新写的数据，再使用原子替换(atomic.Value)，更新调用者的变量，来完成无锁访问共享数据

Mutex实现有点像管程的实现--- 后续研究Mutex源码
Mutex的几种实现：
1.Bargin
2.Handsoff
3.Spinning自旋锁

#### Context
Context类似Thread Local storage
Context传递两种方式：1.调用函数的首参数，2.使用WithContext包装到对象

Context传递元数据，一般不传修改的

#### chan
类型安全的消息队列，充当两个goroutine之间传递的通道，通过chan进行资源交换
chan有无缓冲通道和带缓冲的通道


#### go工具
go profiler
go build -race
go test -race
to tool compile
Benchmark  --- go test -bench=.
