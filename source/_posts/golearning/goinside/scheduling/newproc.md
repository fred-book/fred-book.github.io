### 创建协程

------

在汇编代码里，使用`call newproc`创建协程；在go代码中，使用`go`关键字创建协程，实际`go`的汇编代码也是`call newproc`。

newproc代码如下：

```go
func newproc(fn *funcval) {
   gp := getg()
   pc := getcallerpc()
   systemstack(func() {
      newg := newproc1(fn, gp, pc)

      _p_ := getg().m.p.ptr()
      runqput(_p_, newg, true)

      if mainStarted {
         wakep()
      }
   })
}
```

从代码中，可以看出创建协程分为下面三个阶段：

1. 创建g对象
2. 存入可运行队列
3. 唤醒p对象

#### 创建g对象

创建g对象的代码如下：

```go
//入参fn：为协程的入口函数
//入参callergp：调用newproc函数的g
//入参callerpc：调用newproc的函数中go/call newproc语句的指令地址
func newproc1(fn *funcval, callergp *g, callerpc uintptr) *g {
   _g_ := getg()//因为newproc1是通过systemstack调用的，所以getg获取的是当前g所属的g0（后续详细介绍系统调用）

   if fn == nil {
      _g_.m.throwing = -1 // do not dump full stacks
      throw("go of nil func value")
   }
   acquirem() // disable preemption because it can be holding p in a local var

   _p_ := _g_.m.p.ptr()//获取当前m所绑定的p
   newg := gfget(_p_)//获取空闲状态的g对象，优先从p对象的gFree队列中取，如果p中gFree为空，则向sched对象的全局gFree中获取
   if newg == nil {
      newg = malg(_StackMin)//如果没有空闲的g对象，则创建一个新的g对象，并创建最小栈空间大小的栈
      casgstatus(newg, _Gidle, _Gdead)//设置g对象状态为_Gdead，并存入allg切片中
      allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
   }
   if newg.stack.hi == 0 {
      throw("newproc1: newg missing stack")
   }

   if readgstatus(newg) != _Gdead {
      throw("newproc1: new g is not Gdead")
   }
   //初始化栈空间
   totalSize := uintptr(4*goarch.PtrSize + sys.MinFrameSize) // extra space in case of reads slightly beyond frame
   totalSize = alignUp(totalSize, sys.StackAlign)
   sp := newg.stack.hi - totalSize
   spArg := sp
   if usesLR {
      // caller's LR
      *(*uintptr)(unsafe.Pointer(sp)) = 0
      prepGoExitFrame(sp)
      spArg += sys.MinFrameSize
   }
   //以下赋值newg.sched非常关键
   memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
   newg.sched.sp = sp//将当前栈顶地址保存
   newg.stktopsp = sp//将当前栈顶地址保存
   //设置pc值为goexit函数的地址+1
   newg.sched.pc = abi.FuncPCABI0(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
   newg.sched.g = guintptr(unsafe.Pointer(newg))//保存newg对象地址
   gostartcallfn(&newg.sched, fn)//这行代码非常关键，其作用是模拟goexit函数入口调用了fn
   newg.gopc = callerpc
   newg.ancestors = saveAncestors(callergp)
   newg.startpc = fn.fn
   if isSystemGoroutine(newg, false) {
      atomic.Xadd(&sched.ngsys, +1)
   } else {
      // Only user goroutines inherit pprof labels.
      if _g_.m.curg != nil {
         newg.labels = _g_.m.curg.labels
      }
   }
   // Track initial transition?
   newg.trackingSeq = uint8(fastrand())
   if newg.trackingSeq%gTrackingPeriod == 0 {
      newg.tracking = true
   }
   casgstatus(newg, _Gdead, _Grunnable)//将g对象的状态设置成_Grunnable
   gcController.addScannableStack(_p_, int64(newg.stack.hi-newg.stack.lo))
   //分配goroutine id
   if _p_.goidcache == _p_.goidcacheend {
      // Sched.goidgen is the last allocated id,
      // this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
      // At startup sched.goidgen=0, so main goroutine receives goid=1.
      _p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
      _p_.goidcache -= _GoidCacheBatch - 1
      _p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
   }
   newg.goid = int64(_p_.goidcache)
   _p_.goidcache++
   if raceenabled {
      newg.racectx = racegostart(callerpc)
   }
   if trace.enabled {
      traceGoCreate(newg, newg.startpc)
   }
   releasem(_g_.m)

   return newg
}
```

其中最关键的是赋值newg.sched的那段代码，其作用主要是设置sched，模拟goexit调用协程函数fn，这个将在循环调度中起到至关重要的作用。

在调用gostartcallfn函数前，新建协程的栈如下：

![image-20220512213916418](/assets/images/golearing/image-20220512213916418.png)

调用gostartcallfn函数后，栈变成：

```go
func gostartcallfn(gobuf *gobuf, fv *funcval) {
	var fn unsafe.Pointer
	if fv != nil {
		fn = unsafe.Pointer(fv.fn)
	} else {
		fn = unsafe.Pointer(abi.FuncPCABIInternal(nilfunc))
	}
	gostartcall(gobuf, fn, unsafe.Pointer(fv))
}

func gostartcall(buf *gobuf, fn, ctxt unsafe.Pointer) {
	sp := buf.sp
	sp -= goarch.PtrSize
	*(*uintptr)(unsafe.Pointer(sp)) = buf.pc
	buf.sp = sp
	buf.pc = uintptr(fn)
	buf.ctxt = ctxt
}
```

![image-20220512213933729](/assets/images/golearing/image-20220512213933729.png)

在栈顶减去8字节空间，并将`PC(goexit)+1`存入栈顶地址上，将PC寄存器（当前还未真实设置PC寄存器，在执行新协程时，会将g.sched.pc的值存入PC寄存器）的值设置成fn地址，这个过程就是汇编代码`call fn`的过程。



这个过程至关重要，最后的效果是模拟goexit调用fn，当协程执行完后，将退栈并继续执行goexit函数。

#### 存入可运行队列

```go
//入参_p_：当前所在的m绑定的p对象
//入参gp：新建的g对象
//入参next：是否优先放置到runnext
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrandn(2) == 0 {//如果是随机调度模式，则一半概率是不放置到runnext
		next = false
	}

	if next {//如果优先放置到runnext，则尝试将gp放置到runnext位置，如果runnext本身为空则返回，否则需要将原runnext放到队尾
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}

retry://如果可运行队列未满，则将gp放置到队尾，否则将一半的gp从本地可运行队列放至全局可运行队列中
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}
```

默认情况下，新建的g对象优先放置到本地可运行队列的runnext上，并将runnext上的g对象放至本地可运行队列队尾。如果本地可运行队列满，则将队列中前一半的g对象放至全局可运行队列中。

#### 唤醒P对象

如果主协程已经启动，需要唤醒p对象来调度goroutine。唤醒p对象实际是startm过程（上一节已经描述）。为什么需要唤醒p呢？是因为新建的g对象肯定是没有被调度的（因为创建该g对象的goroutine正在被所在的m调度），所以为了尽最快速度调度新建的g对象，可以唤醒p对象来执行新建的g对象。

```go
func wakep() {
   if atomic.Load(&sched.npidle) == 0 {
      return
   }
   // be conservative about spinning threads
   if atomic.Load(&sched.nmspinning) != 0 || !atomic.Cas(&sched.nmspinning, 0, 1) {
      return
   }
   startm(nil, true)
}
```
