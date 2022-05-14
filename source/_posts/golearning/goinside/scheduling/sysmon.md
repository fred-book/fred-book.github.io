### 系统监控

------

在进程启动过程中，将创建一个专门用来监控系统的m，其入口函数为`sysmon`：

```go
func sysmon() {
   lock(&sched.lock)
   sched.nmsys++
   checkdead()
   unlock(&sched.lock)

   lasttrace := int64(0)
   idle := 0 // how many cycles in succession we had not wokeup somebody
   delay := uint32(0)

   for {
      // 计算休眠时间，如果所有的p处于休眠状态越长，休眠的时间越长
      if idle == 0 { // start with 20us sleep...
         delay = 20
      } else if idle > 50 { // start doubling the sleep after 1ms...
         delay *= 2
      }
      if delay > 10*1000 { // up to 10ms
         delay = 10 * 1000
      }
      usleep(delay) // 休眠一段时间

      // sysmon should not enter deep sleep if schedtrace is enabled so that
      // it can print that information at the right time.
      //
      // It should also not enter deep sleep if there are any active P's so
      // that it can retake P's from syscalls, preempt long running G's, and
      // poll the network if all P's are busy for long stretches.
      //
      // It should wakeup from deep sleep if any P's become active either due
      // to exiting a syscall or waking up due to a timer expiring so that it
      // can resume performing those duties. If it wakes from a syscall it
      // resets idle and delay as a bet that since it had retaken a P from a
      // syscall before, it may need to do it again shortly after the
      // application starts work again. It does not reset idle when waking
      // from a timer to avoid adding system load to applications that spend
      // most of their time sleeping.
      // 当进程处于gc等待阶段或所有的p都处于空闲状态，则休眠到第一个定时器超时。
      now := nanotime()
      if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs)) {
         lock(&sched.lock)
         if atomic.Load(&sched.gcwaiting) != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs) {
            syscallWake := false
            next, _ := timeSleepUntil()
            if next > now {
               atomic.Store(&sched.sysmonwait, 1)
               unlock(&sched.lock)
               // Make wake-up period small enough
               // for the sampling to be correct.
               sleep := forcegcperiod / 2
               if next-now < sleep {
                  sleep = next - now
               }
               shouldRelax := sleep >= osRelaxMinNS
               if shouldRelax {
                  osRelax(true)
               }
               syscallWake = notetsleep(&sched.sysmonnote, sleep)
               if shouldRelax {
                  osRelax(false)
               }
               lock(&sched.lock)
               atomic.Store(&sched.sysmonwait, 0)
               noteclear(&sched.sysmonnote)
            }
            if syscallWake {
               idle = 0
               delay = 20
            }
         }
         unlock(&sched.lock)
      }

      lock(&sched.sysmonlock)
      // Update now in case we blocked on sysmonnote or spent a long time
      // blocked on schedlock or sysmonlock above.
      now = nanotime()

      // trigger libc interceptors if needed
      if *cgo_yield != nil {
         asmcgocall(*cgo_yield, nil)
      }
      // poll network if not polled for more than 10ms
      // 如果距离上一次轮询网络超过10ms，则再次轮询网络
      lastpoll := int64(atomic.Load64(&sched.lastpoll))
      if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
         atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
         list := netpoll(0) // non-blocking - returns list of goroutines
         // 如果当前存在不处于阻塞的g，则将g加入到本地可运行队列或者全局可运行队列
         if !list.empty() {
            // Need to decrement number of idle locked M's
            // (pretending that one more is running) before injectglist.
            // Otherwise it can lead to the following situation:
            // injectglist grabs all P's but before it starts M's to run the P's,
            // another M returns from syscall, finishes running its G,
            // observes that there is no work to do and no other running M's
            // and reports deadlock.
            incidlelocked(-1)
            injectglist(&list)
            incidlelocked(1)
         }
      }
      if GOOS == "netbsd" && needSysmonWorkaround {
         // netpoll is responsible for waiting for timer
         // expiration, so we typically don't have to worry
         // about starting an M to service timers. (Note that
         // sleep for timeSleepUntil above simply ensures sysmon
         // starts running again when that timer expiration may
         // cause Go code to run again).
         //
         // However, netbsd has a kernel bug that sometimes
         // misses netpollBreak wake-ups, which can lead to
         // unbounded delays servicing timers. If we detect this
         // overrun, then startm to get something to handle the
         // timer.
         //
         // See issue 42515 and
         // https://gnats.netbsd.org/cgi-bin/query-pr-single.pl?number=50094.
         if next, _ := timeSleepUntil(); next < now {
            startm(nil, false)
         }
      }
      if atomic.Load(&scavenge.sysmonWake) != 0 {
         // Kick the scavenger awake if someone requested it.
         wakeScavenger()
      }
      // retake P's blocked in syscalls
      // and preempt long running G's
      // 重新获取阻塞在系统调用的p，并抢占长时间运行的g
      if retake(now) != 0 {
         idle = 0
      } else {
         idle++
      }
      // check if we need to force a GC
      // 如果距离上次gc超过2分钟则启动强制gc
      if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {
         lock(&forcegc.lock)
         forcegc.idle = 0
         var list gList
         list.push(forcegc.g)
         injectglist(&list)
         unlock(&forcegc.lock)
      }
      if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
         lasttrace = now
         schedtrace(debug.scheddetail > 0)
      }
      unlock(&sched.sysmonlock)
   }
}
```

从上述代码中可以看出，系统监控m除了休眠外主要做了如下3件事务：

1. 周期性（距离上一次轮询超过10ms）轮询网络，并将处于就绪状态的g对象插入到可运行队列中
2. 重新获取阻塞在系统的p，并抢占长时间运行的g
3. 如果距离上次gc超过2分钟则启动强制gc

其中，事务2的处理规则如下：

```go
func retake(now int64) uint32 {
   n := 0
   // Prevent allp slice changes. This lock will be completely
   // uncontended unless we're already stopping the world.
   lock(&allpLock)
   // We can't use a range loop over allp because we may
   // temporarily drop the allpLock. Hence, we need to re-fetch
   // allp each time around the loop.
   for i := 0; i < len(allp); i++ {// 遍历所有的p
      _p_ := allp[i]
      if _p_ == nil {
         // This can happen if procresize has grown
         // allp but not yet created new Ps.
         continue
      }
      pd := &_p_.sysmontick
      s := _p_.status
      sysretake := false
      if s == _Prunning || s == _Psyscall { // 如果当前p的状态是_Prunning或者_Psyscall
         // Preempt G if it's running for too long.
         t := int64(_p_.schedtick) 
         if int64(pd.schedtick) != t { 
          // 如果p.sysmontick.schedtick与p.schedtick不相等，则更新p.sysmontick.schedtick并且更新schedwhen为当前时间
            pd.schedtick = uint32(t)
            pd.schedwhen = now
         } else if pd.schedwhen+forcePreemptNS <= now {// 如果p单次运行时间超过10ms，则抢占当前p正在执行的g。
            preemptone(_p_)
            // In case of syscall, preemptone() doesn't
            // work, because there is no M wired to P.
            sysretake = true
         }
      }
      if s == _Psyscall {
         // Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
         t := int64(_p_.syscalltick)
         if !sysretake && int64(pd.syscalltick) != t {
            pd.syscalltick = uint32(t)
            pd.syscallwhen = now
            continue
         }
         // On the one hand we don't want to retake Ps if there is no other work to do,
         // but on the other hand we want to retake them eventually
         // because they can prevent the sysmon thread from deep sleep.
         if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
            continue
         }
         // Drop allpLock so we can take sched.lock.
         unlock(&allpLock)
         // Need to decrement number of idle locked M's
         // (pretending that one more is running) before the CAS.
         // Otherwise the M from which we retake can exit the syscall,
         // increment nmidle and report deadlock.
         incidlelocked(-1)
         if atomic.Cas(&_p_.status, s, _Pidle) { // 将p的状态设置成_Pidle
            if trace.enabled {
               traceGoSysBlock(_p_)
               traceProcStop(_p_)
            }
            n++
            _p_.syscalltick++
            handoffp(_p_) // 从当前执行的m中移出p对象
         }
         incidlelocked(1)
         lock(&allpLock)
      }
   }
   unlock(&allpLock)
   return uint32(n)
}
```

从上述代码看出，如果p单次调度超过10ms，则将当前执行的协程抢占，抢占的实现如下：

```go
func preemptone(_p_ *p) bool {
   mp := _p_.m.ptr()
   if mp == nil || mp == getg().m {
      return false
   }
   gp := mp.curg
   if gp == nil || gp == mp.g0 {
      return false
   }

   gp.preempt = true

   // Every call in a goroutine checks for stack overflow by
   // comparing the current stack pointer to gp->stackguard0.
   // Setting gp->stackguard0 to StackPreempt folds
   // preemption into the normal stack overflow check.
   gp.stackguard0 = stackPreempt

   // Request an async preemption of this P.
   if preemptMSupported && debug.asyncpreemptoff == 0 {
      _p_.preempt = true
      preemptM(mp)
   }

   return true
}
```

其中关键处理为`gp.stackguard0 = stackPreempt`，在goroutine每次调用函数时，默认都会进行栈检查即判断`SP > gp.stackguard0`（栈是从高地址向低地址生长，正常情况下SP都应该大于栈保护线`stackguard0`），当`gp.stackguard0 = stackPreempt`时（`stackPreempt`是一个非常大的值），将大于SP，导致栈检查失败，切换到另外一个协程执行。

另外，如果p单次系统调用时间超过10ms，则将p移交处理其它协程。p移交后，调度p的m则继续系统调用。移交过程如下代码：

```go
func handoffp(_p_ *p) {
   // handoffp must start an M in any situation where
   // findrunnable would return a G to run on _p_.

   // if it has local work, start it straight away
   if !runqempty(_p_) || sched.runqsize != 0 {
      startm(_p_, false)
      return
   }
   // if it has GC work, start it straight away
   if gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) {
      startm(_p_, false)
      return
   }
   // no local work, check that there are no spinning/idle M's,
   // otherwise our help is not required
   if atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) == 0 && atomic.Cas(&sched.nmspinning, 0, 1) { // TODO: fast atomic
      startm(_p_, true)
      return
   }
   lock(&sched.lock)
   if sched.gcwaiting != 0 {
      _p_.status = _Pgcstop
      sched.stopwait--
      if sched.stopwait == 0 {
         notewakeup(&sched.stopnote)
      }
      unlock(&sched.lock)
      return
   }
   if _p_.runSafePointFn != 0 && atomic.Cas(&_p_.runSafePointFn, 1, 0) {
      sched.safePointFn(_p_)
      sched.safePointWait--
      if sched.safePointWait == 0 {
         notewakeup(&sched.safePointNote)
      }
   }
   if sched.runqsize != 0 {
      unlock(&sched.lock)
      startm(_p_, false)
      return
   }
   // If this is the last running P and nobody is polling network,
   // need to wakeup another M to poll network.
   if sched.npidle == uint32(gomaxprocs-1) && atomic.Load64(&sched.lastpoll) != 0 {
      unlock(&sched.lock)
      startm(_p_, false)
      return
   }

   // The scheduler lock cannot be held when calling wakeNetPoller below
   // because wakeNetPoller may call wakep which may call startm.
   when := nobarrierWakeTime(_p_)
   pidleput(_p_)
   unlock(&sched.lock)

   if when != 0 {
      wakeNetPoller(when)
   }
}
```

从代码中可以看出，移交过程优先唤醒m来调度p，如果可运行队列中没有可运行的g、或者网络轮询中无事件等，则将p放入sched.pidle链表中。
