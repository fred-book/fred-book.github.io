### 循环调度

------

调度器m启动后，将循环调度goroutine，

```go
func schedule() {
   _g_ := getg()

   if _g_.m.locks != 0 {
      throw("schedule: holding locks")
   }

   if _g_.m.lockedg != 0 {
      stoplockedm()
      execute(_g_.m.lockedg.ptr(), false) // Never returns.
   }

   // We should not schedule away from a g that is executing a cgo call,
   // since the cgo call is using the m's g0 stack.
   if _g_.m.incgo {
      throw("schedule: in cgo")
   }

top:
   pp := _g_.m.p.ptr()
   pp.preempt = false

   if sched.gcwaiting != 0 {
      gcstopm()
      goto top
   }
   if pp.runSafePointFn != 0 {
      runSafePointFn()
   }

   // Sanity check: if we are spinning, the run queue should be empty.
   // Check this before calling checkTimers, as that might call
   // goready to put a ready goroutine on the local run queue.
   if _g_.m.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
      throw("schedule: spinning with local work")
   }

   checkTimers(pp, 0)

   var gp *g
   var inheritTime bool

   // Normal goroutines will check for need to wakeP in ready,
   // but GCworkers and tracereaders will not, so the check must
   // be done here instead.
   tryWakeP := false
   if trace.enabled || trace.shutdown {
      gp = traceReader()
      if gp != nil {
         casgstatus(gp, _Gwaiting, _Grunnable)
         traceGoUnpark(gp, 0)
         tryWakeP = true
      }
   }
   if gp == nil && gcBlackenEnabled != 0 {
      gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
      if gp != nil {
         tryWakeP = true
      }
   }
   if gp == nil {
      // Check the global runnable queue once in a while to ensure fairness.
      // Otherwise two goroutines can completely occupy the local runqueue
      // by constantly respawning each other.
      // 每隔一段时间检查一次（1/61）全局可运行队列以确保公平，否则，goroutine可以通过不断地创建新协程完全占据本地运行队列。
      if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
         lock(&sched.lock)
         gp = globrunqget(_g_.m.p.ptr(), 1)
         unlock(&sched.lock)
      }
   }
   //从当前调度m所绑定的p的本地可运行队列中获取g
   if gp == nil {
      gp, inheritTime = runqget(_g_.m.p.ptr())
      // We can see gp != nil here even if the M is spinning,
      // if checkTimers added a local goroutine via goready.
   }
   //如果本地不存在可运行的g，则继续寻找可运行的g
   if gp == nil {
      gp, inheritTime = findrunnable() // blocks until work is available
   }

   // This thread is going to run a goroutine and is not spinning anymore,
   // so if it was marked as spinning we need to reset it now and potentially
   // start a new spinning M.
   if _g_.m.spinning {
      resetspinning()
   }

   if sched.disable.user && !schedEnabled(gp) {
      // Scheduling of this goroutine is disabled. Put it on
      // the list of pending runnable goroutines for when we
      // re-enable user scheduling and look again.
      lock(&sched.lock)
      if schedEnabled(gp) {
         // Something re-enabled scheduling while we
         // were acquiring the lock.
         unlock(&sched.lock)
      } else {
         sched.disable.runnable.pushBack(gp)
         sched.disable.n++
         unlock(&sched.lock)
         goto top
      }
   }

   // If about to schedule a not-normal goroutine (a GCworker or tracereader),
   // wake a P if there is one.
   if tryWakeP {
      wakep()
   }
   if gp.lockedm != 0 {
      // Hands off own p to the locked m,
      // then blocks waiting for a new p.
      startlockedm(gp)
      goto top
   }
   //执行可运行的g
   execute(gp, inheritTime)
}
```

调度过程包含以下3步：

1. 选择协程：选择可运行的goroutine
2. 执行协程：执行选择的goroutine
3. 下一周期调度：goroutine执行结束后，重新下一周期调度（从代码中看，这一步是隐式的）

下面将对这3步进行详细说明。

#### 选择协程

在开头的代码中，选择协程的代码片段如下：

```go
   if gp == nil {
      // Check the global runnable queue once in a while to ensure fairness.
      // Otherwise two goroutines can completely occupy the local runqueue
      // by constantly respawning each other.
      // 每隔一段时间检查一次（1/61）全局可运行队列以确保公平，否则，goroutine可以通过不断地创建新协程完全占据本地运行队列。
      if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
         lock(&sched.lock)
         gp = globrunqget(_g_.m.p.ptr(), 1)
         unlock(&sched.lock)
      }
   }
   //从当前调度m所绑定的p的本地可运行队列中获取g
   if gp == nil {
      gp, inheritTime = runqget(_g_.m.p.ptr())
      // We can see gp != nil here even if the M is spinning,
      // if checkTimers added a local goroutine via goready.
   }
   //如果本地不存在可运行的g，则继续寻找可运行的g
   if gp == nil {
      gp, inheritTime = findrunnable() // blocks until work is available
   }
```

首先每隔段时间（1/61调度周期）从全局可运行队列获取g对象，目的是为了调度公平。假设有这样一种场景：

1. 协程a的处理函数中，调用`go funb`创建协程b
2. 协程a的处理函数中，调用`go funa`创建协程a

如果只是优先从p的本地可运行队列获取g对象调度，则会出现不停的调度协程a和协程b，而全局可运行队列得不到调度。

其次从p的本地可运行队列中获取g对象，如果本地可运行队列不存在g对象，则继续寻找可运行的g对象。寻找过程代码如下：

```go
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()

	// The conditions here and in handoffp must agree: if
	// findrunnable would return a G to run, handoffp must start
	// an M.

top:
	_p_ := _g_.m.p.ptr()
	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}
	if _p_.runSafePointFn != 0 {
		runSafePointFn()
	}

	now, pollUntil, _ := checkTimers(_p_, 0)

	if fingwait && fingwake {
		if gp := wakefing(); gp != nil {
			ready(gp, 0, true)
		}
	}
	if *cgo_yield != nil {
		asmcgocall(*cgo_yield, nil)
	}

    // 1. 从本地可运行队列中获取g对象
	// local runq
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}

    // 2. 从全局可运行队列中获取g对象
	// global runq
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}

    // 3. 从netpoll中获取处于非阻塞状态的g对象
	// Poll network.
	// This netpoll is only an optimization before we resort to stealing.
	// We can safely skip it if there are no waiters or a thread is blocked
	// in netpoll already. If there is any kind of logical race with that
	// blocked thread (e.g. it has already returned from netpoll, but does
	// not set lastpoll yet), this thread will do blocking netpoll below
	// anyway.
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(0); !list.empty() { // non-blocking
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}

    // 4. 从其它p中盗取g对象
	// Spinning Ms: steal work from other Ps.
	//
	// Limit the number of spinning Ms to half the number of busy Ps.
	// This is necessary to prevent excessive CPU consumption when
	// GOMAXPROCS>>1 but the program parallelism is low.
	procs := uint32(gomaxprocs)
	if _g_.m.spinning || 2*atomic.Load(&sched.nmspinning) < procs-atomic.Load(&sched.npidle) {
		if !_g_.m.spinning {
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)
		}

		gp, inheritTime, tnow, w, newWork := stealWork(now)
		now = tnow
		if gp != nil {
			// Successfully stole.
			return gp, inheritTime
		}
		if newWork {
			// There may be new timer or GC work; restart to
			// discover.
			goto top
		}
		if w != 0 && (pollUntil == 0 || w < pollUntil) {
			// Earlier timer to wait for.
			pollUntil = w
		}
	}

    // 5. 如果确实没有课调度的g，则处于gc标记阶段，则获取背景标记g对象
	// We have nothing to do.
	//
	// If we're in the GC mark phase, can safely scan and blacken objects,
	// and have work to do, run idle-time marking rather than give up the
	// P.
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) {
		node := (*gcBgMarkWorkerNode)(gcBgMarkWorkerPool.pop())
		if node != nil {
			_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
			gp := node.gp.ptr()
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}

	// wasm only:
	// If a callback returned and no other goroutine is awake,
	// then wake event handler goroutine which pauses execution
	// until a callback was triggered.
	gp, otherReady := beforeIdle(now, pollUntil)
	if gp != nil {
		casgstatus(gp, _Gwaiting, _Grunnable)
		if trace.enabled {
			traceGoUnpark(gp, 0)
		}
		return gp, false
	}
	if otherReady {
		goto top
	}

	// Before we drop our P, make a snapshot of the allp slice,
	// which can change underfoot once we no longer block
	// safe-points. We don't need to snapshot the contents because
	// everything up to cap(allp) is immutable.
	allpSnapshot := allp
	// Also snapshot masks. Value changes are OK, but we can't allow
	// len to change out from under us.
	idlepMaskSnapshot := idlepMask
	timerpMaskSnapshot := timerpMask

	// return P and block
	lock(&sched.lock)
	if sched.gcwaiting != 0 || _p_.runSafePointFn != 0 {
		unlock(&sched.lock)
		goto top
	}
    // 6. 再次从全局可运行队列中获取g对象
	if sched.runqsize != 0 {
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		return gp, false
	}
	if releasep() != _p_ {
		throw("findrunnable: wrong p")
	}
    
    // 将p休眠
	pidleput(_p_)
	unlock(&sched.lock)

	// Delicate dance: thread transitions from spinning to non-spinning
	// state, potentially concurrently with submission of new work. We must
	// drop nmspinning first and then check all sources again (with
	// #StoreLoad memory barrier in between). If we do it the other way
	// around, another thread can submit work after we've checked all
	// sources but before we drop nmspinning; as a result nobody will
	// unpark a thread to run the work.
	//
	// This applies to the following sources of work:
	//
	// * Goroutines added to a per-P run queue.
	// * New/modified-earlier timers on a per-P timer heap.
	// * Idle-priority GC work (barring golang.org/issue/19112).
	//
	// If we discover new work below, we need to restore m.spinning as a signal
	// for resetspinning to unpark a new worker thread (because there can be more
	// than one starving goroutine). However, if after discovering new work
	// we also observe no idle Ps it is OK to skip unparking a new worker
	// thread: the system is fully loaded so no spinning threads are required.
	// Also see "Worker thread parking/unparking" comment at the top of the file.
    // 7. 如果当前m自旋使能，则从netpoll中等待事件到达的g对象
	wasSpinning := _g_.m.spinning
	if _g_.m.spinning {
		_g_.m.spinning = false
		if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
			throw("findrunnable: negative nmspinning")
		}

		// Note the for correctness, only the last M transitioning from
		// spinning to non-spinning must perform these rechecks to
		// ensure no missed work. We are performing it on every M that
		// transitions as a conservative change to monitor effects on
		// latency. See golang.org/issue/43997.

		// Check all runqueues once again.
		_p_ = checkRunqsNoP(allpSnapshot, idlepMaskSnapshot)
		if _p_ != nil {
			acquirep(_p_)
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)
			goto top
		}

		// Check for idle-priority GC work again.
		_p_, gp = checkIdleGCNoP()
		if _p_ != nil {
			acquirep(_p_)
			_g_.m.spinning = true
			atomic.Xadd(&sched.nmspinning, 1)

			// Run the idle worker.
			_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}

		// Finally, check for timer creation or expiry concurrently with
		// transitioning from spinning to non-spinning.
		//
		// Note that we cannot use checkTimers here because it calls
		// adjusttimers which may need to allocate memory, and that isn't
		// allowed when we don't have an active P.
		pollUntil = checkTimersNoP(allpSnapshot, timerpMaskSnapshot, pollUntil)
	}

	// Poll network until next timer.
	if netpollinited() && (atomic.Load(&netpollWaiters) > 0 || pollUntil != 0) && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
		atomic.Store64(&sched.pollUntil, uint64(pollUntil))
		if _g_.m.p != 0 {
			throw("findrunnable: netpoll with p")
		}
		if _g_.m.spinning {
			throw("findrunnable: netpoll with spinning")
		}
		delay := int64(-1)
		if pollUntil != 0 {
			if now == 0 {
				now = nanotime()
			}
			delay = pollUntil - now
			if delay < 0 {
				delay = 0
			}
		}
		if faketime != 0 {
			// When using fake time, just poll.
			delay = 0
		}
		list := netpoll(delay) // block until new work is available
		atomic.Store64(&sched.pollUntil, 0)
		atomic.Store64(&sched.lastpoll, uint64(nanotime()))
		if faketime != 0 && list.empty() {
			// Using fake time and nothing is ready; stop M.
			// When all M's stop, checkdead will call timejump.
			stopm()
			goto top
		}
		lock(&sched.lock)
		_p_ = pidleget()
		unlock(&sched.lock)
		if _p_ == nil {
			injectglist(&list)
		} else {
			acquirep(_p_)
			if !list.empty() {
				gp := list.pop()
				injectglist(&list)
				casgstatus(gp, _Gwaiting, _Grunnable)
				if trace.enabled {
					traceGoUnpark(gp, 0)
				}
				return gp, false
			}
			if wasSpinning {
				_g_.m.spinning = true
				atomic.Xadd(&sched.nmspinning, 1)
			}
			goto top
		}
	} else if pollUntil != 0 && netpollinited() {
		pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
		if pollerPollUntil == 0 || pollerPollUntil > pollUntil {
			netpollBreak()
		}
	}
    // 将m停止
	stopm()
	goto top
}
```

从上述代码中可以看出，选择协程按照以下顺序：

1. 从本地可运行队列中获取g对象
2. 从全局可运行队列中获取g对象
3. 从netpoll中获取处于非阻塞状态的g对象
4. 从其它p中盗取g对象
5. 如果确实没有课调度的g，则处于gc标记阶段，则获取背景标记g对象
6. 再次从全局可运行队列中获取g对象
7. 如果当前m自旋使能，则从netpoll中等待事件到达的g对象

如果实在没有可以调度的g对象，则停止m，并再次按照上述顺序寻找可运行的g对象。

#### 执行协程

当前获取到可运行的g对象后，则执行该g对象。

```go
func execute(gp *g, inheritTime bool) {
   _g_ := getg()

   // Assign gp.m before entering _Grunning so running Gs have an
   // M.
   _g_.m.curg = gp
   gp.m = _g_.m
   casgstatus(gp, _Grunnable, _Grunning)
   gp.waitsince = 0
   gp.preempt = false
   gp.stackguard0 = gp.stack.lo + _StackGuard
   if !inheritTime {
      _g_.m.p.ptr().schedtick++
   }

   // Check whether the profiler needs to be turned on or off.
   hz := sched.profilehz
   if _g_.m.profilehz != hz {
      setThreadCPUProfiler(hz)
   }

   if trace.enabled {
      // GoSysExit has to happen when we have a P, but before GoStart.
      // So we emit it here.
      if gp.syscallsp != 0 && gp.sysblocktraced {
         traceGoSysExit(gp.sysexitticks)
      }
      traceGoStart()
   }

   gogo(&gp.sched)
}
```

其中最关键的是调用gogo函数。

```assembly
TEXT runtime·gogo(SB), NOSPLIT, $0-8
   MOVQ   buf+0(FP), BX   // FP指向入参起始位置，所以FP+0指向第一个入参，即gobuf
   MOVQ   gobuf_g(BX), DX // 将gobuf.g存入DX
   MOVQ   0(DX), CX       // 将gobuf.g存入CX
   JMP    gogo<>(SB)      // 跳转到gogo

TEXT gogo<>(SB), NOSPLIT, $0
   get_tls(CX)          // 将TLS地址存入CX
   MOVQ   DX, g(CX)     // 将DX的值（gobuf.g）存入CX指向的地址（TLS）
   MOVQ   DX, R14       // 将DX的值（gobuf.g）存入R14
   MOVQ   gobuf_sp(BX), SP    // 将gobuf.sp存入SP
   MOVQ   gobuf_ret(BX), AX   // 将gobuf.ret存入AX
   MOVQ   gobuf_ctxt(BX), DX  // 将gobuf.ctxt存入DX
   MOVQ   gobuf_bp(BX), BP    // 将gobuf.bp存入BP
   MOVQ   $0, gobuf_sp(BX)    // 将gobuf.sp清零
   MOVQ   $0, gobuf_ret(BX)   // 将gobuf.ret清零 
   MOVQ   $0, gobuf_ctxt(BX)  // 将gobuf.ctxt清零 
   MOVQ   $0, gobuf_bp(BX)    // 将gobuf.bp清零 
   MOVQ   gobuf_pc(BX), BX    // 将gobuf.pc存入BX
   JMP    BX                  // 跳转到BX指向的地址
```

上述汇编代码主要是将gobuf中存的上下文恢复到寄存器上，从上一节讲到的g.sched的内容可知，此时，当前栈和寄存器即恢复到如下状态：

![image-20220512214018818](/assets/images/golearing/image-20220512214018818.png)

最后`JMP    BX` 指令则跳转到协程的入口函数`fn`，即开始执行`fn`函数，此时的栈则如下：

![image-20220512214458959](/assets/images/golearing/image-20220512214458959.png)



#### 下一周期调度

当fn函数执行完成返回后，根据第1章讲到函数调用返回的过程（先释放当前栈帧，将SP地址保存的值弹出到PC寄存器，并跳转到该地址）可知，程序将跳转到`PC(goexit)+1`指令地址，即执行`CALL   runtime·goexit1(SB)` 。

```go
TEXT runtime·goexit(SB),NOSPLIT|TOPFRAME,$0-0
   BYTE   $0x90  // NOP
   CALL   runtime·goexit1(SB)    // does not return
   // traceback from goexit1 must hit code range of goexit
   BYTE   $0x90  // NOP
```

```go
func goexit1() {
   if raceenabled {
      racegoend()
   }
   if trace.enabled {
      traceGoEnd()
   }
   mcall(goexit0) // 切换到g0，并执行goexit0
}

// goexit continuation on g0.
// 入参gp：退出的协程
func goexit0(gp *g) {
   _g_ := getg()
   _p_ := _g_.m.p.ptr()

   casgstatus(gp, _Grunning, _Gdead) // 设置gp为_Gdead状态
   gcController.addScannableStack(_p_, -int64(gp.stack.hi-gp.stack.lo)) // 将退出协程栈空间加入可扫描栈
   if isSystemGoroutine(gp, false) {
      atomic.Xadd(&sched.ngsys, -1)
   }
   gp.m = nil // 以下为清理退出协程的上下文
   locked := gp.lockedm != 0
   gp.lockedm = 0
   _g_.m.lockedg = 0
   gp.preemptStop = false
   gp.paniconfault = false
   gp._defer = nil // should be true already but just in case.
   gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
   gp.writebuf = nil
   gp.waitreason = 0
   gp.param = nil
   gp.labels = nil
   gp.timer = nil

   if gcBlackenEnabled != 0 && gp.gcAssistBytes > 0 {
      // Flush assist credit to the global pool. This gives
      // better information to pacing if the application is
      // rapidly creating an exiting goroutines.
      assistWorkPerByte := gcController.assistWorkPerByte.Load()
      scanCredit := int64(assistWorkPerByte * float64(gp.gcAssistBytes))
      atomic.Xaddint64(&gcController.bgScanCredit, scanCredit)
      gp.gcAssistBytes = 0
   }

   dropg() // 解绑定当前m和执行g（退出的协程）的关系

   if GOARCH == "wasm" { // no threads yet on wasm
      gfput(_p_, gp)
      schedule() // never returns
   }

   if _g_.m.lockedInt != 0 {
      print("invalid m->lockedInt = ", _g_.m.lockedInt, "\n")
      throw("internal lockOSThread error")
   }
   gfput(_p_, gp) // 将g对象存入p的gfree队列
   if locked {
      // The goroutine may have locked this thread because
      // it put it in an unusual kernel state. Kill it
      // rather than returning it to the thread pool.

      // Return to mstart, which will release the P and exit
      // the thread.
      if GOOS != "plan9" { // See golang.org/issue/22227.
         gogo(&_g_.m.g0.sched)
      } else {
         // Clear lockedExt on plan9 since we may end up re-using
         // this thread.
         _g_.m.lockedExt = 0
      }
   }
   schedule()  // 重新开始下一周期调度
}
```

从以上汇编代码中可以看下，在清理完退出协程的上下文后，将协程对象放入p.gfree队列，并进行下一周期调度。

至此调度框架实现了对协程的周期调度。
