### 协程

------

#### 协程的意义

一般来说，线程的进程的最小执行单元，被操作系统统一调度。为了提升进程的并发处理能力，可以采用多线程的方式。但是随着线程数量的增加，线程本身带来的开销变得不可忽视，特别是在大规模并发系统下线程频繁切换场景。线程的开销大体现在如下两点：

- 创建和切换开销大：线程的创建和切换需要进入内核，而进入内核所消耗的性能代价比较高
- 内存开销大：内核在创建线程时默认为其分配一个比较大的栈空间（一般为2 M），并且栈空间一旦创建和初始化后，其大小就不能再变化，所以在某些场景下会出现栈溢出故障。

为了解决以上线程的弊端，同时满足大规模并发的要求，协程（coroutine）的概念应运而生，相比于线程，协程则轻量得多：

- 协程是用户态，其创建和切换都在用户代码中完成，而无需进入内核态，其开销远小于线程的创建和切换。
- 协程栈空间默认为2 K，在大多数情况下已经够用。即使不够，Go协程也能自动进行扩栈，同时如果多余的栈空间不再使用，还会自动缩栈。这样既提升了内存利用率，同时也避免了栈溢出风险。

#### 协程的原理

协程（Goroutine）依靠线程调度，而这些线程则称为协程调度器。在一个进程中会同时存在多个协程，而调度线程也有多个，协程和线程形成多对多（M : N）的关系模型。每个调度线程都是在周而复始的调度协程，其工作过程可以用以下伪代码来简单概括：

```go
//线程执行函数
func schedule() {
    for { //循环调度
        g := find_runable_goroutine_from_m_goroutines() //根据某种算法找出一个可用的goroutine
        run_g(g) //运行该goroutine, 直到调度其它goroutine才返回
        save_status_of_g(g) //保存goroutine状态
    }
}
```

每一个调度线程都负责调度一组协程，在每一个调度循环中，线程获取一个可运行的协程，并调度该协程。当协程调度完（调度时间片到达，长时间阻塞，协程退出等），则保存协程状态，并执行下一个调度过程。

线程被CPU调度，相当于将CPU执行时间切分成若干时间片分配给不同的线程；协程被线程调度，则进一步切分线程的执行时间。不管是线程还是协程，都是一种逻辑概念，本质上是记录程序运行上下文的一种结构体，通过一种调度机制，被挂载到CPU上执行，并在执行过程中更新上下文。

![image-20220504182707764](/assets/images/golearing/image-20220504182707764.png)
