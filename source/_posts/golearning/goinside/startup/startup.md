### 进程启动

------

在linux系统下，像c程序一样，进程启动有一段引导过程，而并非直接调转到main函数接口并执行。通过readelf命令可以查看可执行程序入口函数，如下：

```shell
$ readelf -a main
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x45bfa0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          456 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         7
  Size of section headers:           64 (bytes)
  Number of section headers:         23
  Section header string table index: 3

...

Symbol table '.symtab' contains 2054 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
  ...
  
  1094: 000000000045bfa0     5 FUNC    GLOBAL DEFAULT    1 _rt0_amd64_linux
  ...
```

从可执行程序的elf信息看，该程序的入口函数`Entry point address: 0x45bfa0`地址是`0x45bfa0`，即`_rt0_amd64_linux`函数。

```assembly
TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
   JMP    _rt0_amd64(SB)
```

`_rt0_amd64_linux`函数将调转到`_rt0_amd64`函数。

```assembly
TEXT _rt0_amd64(SB),NOSPLIT,$-8
   MOVQ   0(SP), DI  // argc
   LEAQ   8(SP), SI  // argv
   JMP    runtime·rt0_go(SB)
```

`_rt0_amd64`函数分别拷贝入参`argc`，`argv`到DI，SI寄存器中，并跳转到`runtime·rt0_go`函数。

```assembly
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
	// copy arguments forward on an even stack
	MOVQ	DI, AX		// 拷贝入参argc到AX寄存器
	MOVQ	SI, BX		// 拷贝入参argv到BX寄存器
	SUBQ	$(5*8), SP	// 开辟5*8字节堆栈空间，用于3个参数+2个auto（是什么？）
	ANDQ	$~15, SP    // 将SP值对齐16字节（将SP值低4bit清零，SP值保持不变或者减小，栈空间扩大）   
	MOVQ	AX, 24(SP)  // 将AX值(argc)存入SP+24栈地址上
	MOVQ	BX, 32(SP)  // 将BX值(argv)存入SP+32栈地址上

	// create istack out of the given (operating system) stack.
	// _cgo_init may update stackguard.
	MOVQ	$runtime·g0(SB), DI // 将runtime·g0全局变量（地址）存入DI寄存器
	LEAQ	(-64*1024+104)(SP), BX // 将SP-64*1024+104值存入BX寄存器
	MOVQ	BX, g_stackguard0(DI)  // 将BX值（SP-64*1024+104）赋值给runtime·g0.stackguard0
	MOVQ	BX, g_stackguard1(DI)  // 将BX值（SP-64*1024+104）赋值给runtime·g0.stackguard1
	MOVQ	BX, (g_stack+stack_lo)(DI) // 将BX值（SP-64*1024+104）赋值给runtime·g0.stack.stack_lo
	MOVQ	SP, (g_stack+stack_hi)(DI) // 将当前SP值赋值给runtime·g0.stack.stack_hi

	// find out information about the processor we're on
	MOVL	$0, AX // 将0赋值给AX寄存器，CPUID指令将根据AX值打印不同的CPU信息
	CPUID          // 打印CPU信息，并将CPU信息存于BX，DX，CX寄存器，并将处理结果存于AX寄存器
	CMPL	AX, $0 // 比较AX值与0
	JE	nocpuinfo  // 如果AX值等于0，表示无CPU信息，则调转到nocpuinfo

	CMPL	BX, $0x756E6547  // "Genu" 
	JNE	notintel             // 如果非intel CPU，则跳转到notintel
	CMPL	DX, $0x49656E69  // "ineI"
	JNE	notintel             // 如果非intel CPU，则跳转到notintel
	CMPL	CX, $0x6C65746E  // "ntel"
	JNE	notintel             // 如果非intel CPU，则跳转到notintel
	MOVB	$1, runtime·isIntel(SB) // 如果是intel CPU，则设置全局变量runtime·isIntel为1

notintel: // 不展开说明
	// Load EAX=1 cpuid flags
	MOVL	$1, AX
	CPUID
	MOVL	AX, runtime·processorVersionInfo(SB)

nocpuinfo: 
	// if there is an _cgo_init, call it.
	MOVQ	_cgo_init(SB), AX // 将_cgo_init（unsafe.Pointer）存入AX寄存器
	TESTQ	AX, AX // 检测AX值（_cgo_init）是否为空
	JZ	needtls // 如果AX值为0（没有使能cgo），则跳转到needtls
	// arg 1: g0, already in DI
	MOVQ	$setg_gcc<>(SB), SI // arg 2: setg_gcc //将setg_gcc（函数地址）存入SI寄存器
#ifdef GOOS_android // 不展开说明
	MOVQ	$runtime·tls_g(SB), DX 	// arg 3: &tls_g
	// arg 4: TLS base, stored in slot 0 (Android's TLS_SLOT_SELF).
	// Compensate for tls_g (+16).
	MOVQ	-16(TLS), CX
#else
	MOVQ	$0, DX	// arg 3, 4: not used when using platform's TLS // 将0赋值给DX值
	MOVQ	$0, CX // 将0赋值给CX值
#endif
#ifdef GOOS_windows // 不展开说明
	// Adjust for the Win64 calling convention.
	MOVQ	CX, R9 // arg 4
	MOVQ	DX, R8 // arg 3
	MOVQ	SI, DX // arg 2
	MOVQ	DI, CX // arg 1
#endif
	CALL	AX // 调用AX值表示的函数（_cgo_init指针指向的函数）

	// update stackguard after _cgo_init
	MOVQ	$runtime·g0(SB), CX         // 将runtime·g0（全局变量）存入CX寄存器
	MOVQ	(g_stack+stack_lo)(CX), AX  // 将runtime·g0.stack.stack_lo存入AX寄存器
	ADDQ	$const__StackGuard, AX      // 将AX值（runtime·g0.stack.stack_lo）+ StackGuard 存入AX寄存器
	MOVQ	AX, g_stackguard0(CX) // 将AX值赋值runtime·g0.stackguard0
	MOVQ	AX, g_stackguard1(CX) // 将AX值赋值runtime·g0.stackguard1

#ifndef GOOS_windows
	JMP ok // 非Windows平台，跳转到ok处
#endif
needtls:
#ifdef GOOS_plan9 // 不展开
	// skip TLS setup on Plan 9
	JMP ok
#endif
#ifdef GOOS_solaris // 不展开
	// skip TLS setup on Solaris
	JMP ok
#endif
#ifdef GOOS_illumos // 不展开
	// skip TLS setup on illumos
	JMP ok
#endif
#ifdef GOOS_darwin // 不展开
	// skip TLS setup on Darwin
	JMP ok
#endif
#ifdef GOOS_openbsd // 不展开
	// skip TLS setup on OpenBSD
	JMP ok
#endif

	LEAQ	runtime·m0+m_tls(SB), DI // 将runtime·m0.tls地址保存到DI寄存器
	CALL	runtime·settls(SB)       // 调用runtime·settls函数，将runtime·m0.tls[0]设置成线程TLS

	// store through it, to make sure it works
	get_tls(BX)                      // 将TLS寄存器值保存到BX寄存器
	MOVQ	$0x123, g(BX)            // 将0x123赋值给BX值指向的地址
	MOVQ	runtime·m0+m_tls(SB), AX // 将runtime·m0.tls[0]值保存到AX
	CMPQ	AX, $0x123               // 比较AX值和0x123是否相等
	JEQ 2(PC)                        // 相等则调到ok
	CALL	runtime·abort(SB)        // 不相等，则调用runtime·abort退出
ok:
	// set the per-goroutine and per-mach "registers"
	get_tls(BX)                      // 将TLS寄存器（-8(FS)）值保存到BX寄存器 
	LEAQ	runtime·g0(SB), CX       // 将runtime·g0的地址保存到CX寄存器
	MOVQ	CX, g(BX)                // 将CX值（runtime·g0的地址）保存到TLS（即runtime·m0.tls[0]）
	LEAQ	runtime·m0(SB), AX       // 将runtime·m0的地址保存到AX寄存器

	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)             // 将CX值（runtime·g0的地址）赋值给runtime·m0.g0
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)              // 将AX值（runtime·m0的地址）赋值给runtime·g0.m

	CLD				// convention is D is always left cleared

	// Check GOAMD64 reqirements
	// We need to do this after setting up TLS, so that
	// we can report an error if there is a failure. See issue 49586.
#ifdef NEED_FEATURES_CX // 不展开
	MOVL	$0, AX
	CPUID
	CMPL	AX, $0
	JE	bad_cpu
	MOVL	$1, AX
	CPUID
	ANDL	$NEED_FEATURES_CX, CX
	CMPL	CX, $NEED_FEATURES_CX
	JNE	bad_cpu
#endif

#ifdef NEED_MAX_CPUID // 不展开
	MOVL	$0x80000000, AX
	CPUID
	CMPL	AX, $NEED_MAX_CPUID
	JL	bad_cpu
#endif

#ifdef NEED_EXT_FEATURES_BX // 不展开
	MOVL	$7, AX
	MOVL	$0, CX
	CPUID
	ANDL	$NEED_EXT_FEATURES_BX, BX
	CMPL	BX, $NEED_EXT_FEATURES_BX
	JNE	bad_cpu
#endif

#ifdef NEED_EXT_FEATURES_CX // 不展开
	MOVL	$0x80000001, AX
	CPUID
	ANDL	$NEED_EXT_FEATURES_CX, CX
	CMPL	CX, $NEED_EXT_FEATURES_CX
	JNE	bad_cpu
#endif

#ifdef NEED_OS_SUPPORT_AX // 不展开
	XORL    CX, CX
	XGETBV
	ANDL	$NEED_OS_SUPPORT_AX, AX
	CMPL	AX, $NEED_OS_SUPPORT_AX
	JNE	bad_cpu
#endif

	CALL	runtime·check(SB) // 调用runtime·check函数，检查是否一切正常

	MOVL	24(SP), AX		// 将SP+24值（argc）赋值给AX寄存器
	MOVL	AX, 0(SP)       // 将AX值（argc）保存到SP栈地址上
	MOVQ	32(SP), AX		// 将SP+32值（argv）赋值给AX寄存器
	MOVQ	AX, 8(SP)       // 将AX值（argv）保存到SP+8栈地址上
	CALL	runtime·args(SB)      // 调用runtime·args函数
	CALL	runtime·osinit(SB)    // 调用runtime·osinit函数
	CALL	runtime·schedinit(SB) // 调用runtime·schedinit函数

	// create a new goroutine to start program
	MOVQ	$runtime·mainPC(SB), AX		// 将runtime·main函数地址赋值给AX寄存器
	PUSHQ	AX                          // 将AX值（runtime·main）压栈
	CALL	runtime·newproc(SB)         // 调用runtime·newproc函数创建协程
	POPQ	AX                          // 将栈顶SP弹出保存到AX寄存器

	// start this M
	CALL	runtime·mstart(SB)          // 调用mstart，将当前启动线程作为m启动

	CALL	runtime·abort(SB)	        // 一般情况下mstart不会退出（循环调度g），否则退出进程
	RET
```

`runtime·rt0_go`主要完成了如下流程：

1. 创建并初始化`g0`栈
2. 将`g0`地址保存到线程变量
3. 相关模块初始化
4. 创建协程执行`runtime·main`函数（主协程）
5. 将当前线程作为`m0`启动

`runtime·main`函数实现如下：

```go
func main() {
   g := getg()

   // Racectx of m0->g0 is used only as the parent of the main goroutine.
   // It must not be used for anything else.
   g.m.g0.racectx = 0

   // Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
   // Using decimal instead of binary GB and MB because
   // they look nicer in the stack overflow failure message.
   if goarch.PtrSize == 8 {
      maxstacksize = 1000000000
   } else {
      maxstacksize = 250000000
   }

   // An upper limit for max stack size. Used to avoid random crashes
   // after calling SetMaxStack and trying to allocate a stack that is too big,
   // since stackalloc works with 32-bit sizes.
   maxstackceiling = 2 * maxstacksize

   // Allow newproc to start new Ms.
   mainStarted = true

   if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
      systemstack(func() {
         newm(sysmon, nil, -1)
      })
   }

   // Lock the main goroutine onto this, the main OS thread,
   // during initialization. Most programs won't care, but a few
   // do require certain calls to be made by the main thread.
   // Those can arrange for main.main to run in the main thread
   // by calling runtime.LockOSThread during initialization
   // to preserve the lock.
   lockOSThread()

   if g.m != &m0 {
      throw("runtime.main not on m0")
   }

   // Record when the world started.
   // Must be before doInit for tracing init.
   runtimeInitTime = nanotime()
   if runtimeInitTime == 0 {
      throw("nanotime returning zero")
   }

   if debug.inittrace != 0 {
      inittrace.id = getg().goid
      inittrace.active = true
   }

   doInit(&runtime_inittask) // Must be before defer.

   // Defer unlock so that runtime.Goexit during init does the unlock too.
   needUnlock := true
   defer func() {
      if needUnlock {
         unlockOSThread()
      }
   }()

   gcenable() // 使能gc

   main_init_done = make(chan bool)
   if iscgo {
      if _cgo_thread_start == nil {
         throw("_cgo_thread_start missing")
      }
      if GOOS != "windows" {
         if _cgo_setenv == nil {
            throw("_cgo_setenv missing")
         }
         if _cgo_unsetenv == nil {
            throw("_cgo_unsetenv missing")
         }
      }
      if _cgo_notify_runtime_init_done == nil {
         throw("_cgo_notify_runtime_init_done missing")
      }
      // Start the template thread in case we enter Go from
      // a C-created thread and need to create a new thread.
      startTemplateThread()
      cgocall(_cgo_notify_runtime_init_done, nil)
   }

   doInit(&main_inittask)

   // Disable init tracing after main init done to avoid overhead
   // of collecting statistics in malloc and newproc
   inittrace.active = false

   close(main_init_done)

   needUnlock = false
   unlockOSThread()

   if isarchive || islibrary {
      // A program compiled with -buildmode=c-archive or c-shared
      // has a main, but it is not executed.
      return
   }
   fn := main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
   fn()
   if raceenabled {
      racefini()
   }

   // Make racy client program work: if panicking on
   // another goroutine at the same time as main returns,
   // let the other goroutine finish printing the panic trace.
   // Once it does, it will exit. See issues 3934 and 20018.
   if atomic.Load(&runningPanicDefers) != 0 {
      // Running deferred functions should not take long.
      for c := 0; c < 1000; c++ {
         if atomic.Load(&runningPanicDefers) == 0 {
            break
         }
         Gosched()
      }
   }
   if atomic.Load(&panicking) != 0 {
      gopark(nil, nil, waitReasonPanicWait, traceEvGoStop, 1)
   }

   exit(0)
   for {
      var x *int32
      *x = 0
   }
}
```

其中

```go
   fn := main_main 
   fn()
```

将执行用户定义的main函数。