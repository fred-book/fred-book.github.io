### 线程本地存储

------

线程本地存储（TLS: thread local storage）为每个线程有独立的存储空间来存储线程变量。当一个线程修改线程变量的值时，不影响另外一个线程读取线程变量的值。因为线程变量在每个线程的TLS都有一份，访问时只能读取所在TLS中的线程变量值。

在x86-64系统下，与线程本地存储有关的寄存器是FS寄存器：

- FS寄存器：用户态使用FS寄存器保存线程本地存储的基址

可用通过调用系统函数设置或者查询TLS基址：

```c
int syscall(SYS_arch_prctl, int code, unsigned long addr);
int syscall(SYS_arch_prctl, int code, unsigned long *addr);
```

其中code的取值：

> **ARCH_SET_FS**
>
> Set the 64-bit base for the *FS* register to *addr*.
>
> **ARCH_GET_FS**
>
> Return the 64-bit base value for the *FS* register of the current thread in the *unsigned* long pointed to by *addr*.
>
> **ARCH_SET_GS**
>
> Set the 64-bit base for the *GS* register to *addr*.
>
> **ARCH_GET_GS**
>
> Return the 64-bit base value for the *GS* register of the current thread in the *unsigned* long pointed to by *addr*.

Go本身不提供线程本地存储能力（对于Go编程人员来说，直接面对的是协程），而是在内核使用了TLS机制来协助完成协程的调度。

Go内核使用到LTS的有以下几处地方：

1. 在启动函数`runtime·rt0_go`中

```assembly
LEAQ    runtime·m0+m_tls(SB), DI // 将runtime·m0.tls地址存入DI寄存器
CALL    runtime·settls(SB)       // 调用runtime·settls函数
...
get_tls(BX) // 将LTS地址（下限地址）保存到BX寄存器
LEAQ	runtime·g0(SB), CX // 将runtime·g0地址存入CX寄存器
MOVQ	CX, g(BX) // 将CX值（runtime·g0地址）存入LTS地址
...
```

```assembly
TEXT runtime·settls(SB),NOSPLIT,$32
#ifdef GOOS_android
   // Android stores the TLS offset in runtime·tls_g.
   SUBQ   runtime·tls_g(SB), DI
#else
   ADDQ   $8, DI // ELF wants to use -8(FS) // 将DI值+8（runtime·m0.tls+8）并存入DI寄存器
#endif
   MOVQ   DI, SI         // 将DI值存入SI寄存器
   MOVQ   $0x1002, DI    // 将0x1002（ARCH_SET_FS）存入DI寄存器
   MOVQ   $SYS_arch_prctl, AX // 将SYS_arch_prctl函数地址存入AX寄存器
   SYSCALL                    // 调用SYSCALL
   CMPQ   AX, $0xfffffffffffff001 // 比较AX值和-1比较，判断系统调用范围值是否为-1
   JLS    2(PC)                   // 小于-1，则跳转到PC+2指令即RET
   MOVL   $0xf1, 0xf1  // crash   // 等于-1，则crash
   RET
```

```c
#define get_tls(r) MOVQ TLS, r
```

以上代码的目的是将`m0.tls`保存到FS寄存器，作为LTS的基址。其中m结构体中的tls为长度为6的uintptr数组，即每个m可以用来作为LTS的空间大小为48字节。因为FS寄存器存储的TLS基址是TLS的上限地址，所以上述汇编`runtime·settls`要将`runtime·m0.tls+8`保存到FS寄存器，即TLS空间大小为8字节，即`runtime·m0.tls[0]`内存空间。

然后将`runtime·g0`地址保存到LTS中。

```
type m struct {
   ...
   tls           [tlsSlots]uintptr // thread-local storage (for x86 extern register)
   ...
}
```

2. 在创建m的过程中，调用`newosproc`创建系统线程，`newosproc`会调用`clone`函数，在`clone`函数中将调用`SYS_clone`，同样将`m.tls[0]`作为LTS，并将`m.g0`保存到LTS中。

```assembly
// int32 clone(int32 flags, void *stk, M *mp, G *gp, void (*fn)(void));
TEXT runtime·clone(SB),NOSPLIT,$0
   MOVL   flags+0(FP), DI
   MOVQ   stk+8(FP), SI
   MOVQ   $0, DX
   MOVQ   $0, R10
   MOVQ    $0, R8
   // Copy mp, gp, fn off parent stack for use by child.
   // Careful: Linux system call clobbers CX and R11.
   MOVQ   mp+16(FP), R13
   MOVQ   gp+24(FP), R9
   MOVQ   fn+32(FP), R12
   CMPQ   R13, $0    // m
   JEQ    nog1
   CMPQ   R9, $0    // g
   JEQ    nog1
   LEAQ   m_tls(R13), R8
#ifdef GOOS_android
   // Android stores the TLS offset in runtime·tls_g.
   SUBQ   runtime·tls_g(SB), R8
#else
   ADDQ   $8, R8 // ELF wants to use -8(FS)
#endif
   ORQ    $0x00080000, DI //add flag CLONE_SETTLS(0x00080000) to call clone
nog1:
   MOVL   $SYS_clone, AX
   SYSCALL
   ...
   // In child, set up new stack
   get_tls(CX)
   MOVQ	R13, g_m(R9)
   MOVQ	R9, g(CX)
```

3. 在调度器选择可运行g后，使用gogo执行g的过程中将执行的g地址保存到LTS中。

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

4. 从LTS中读取的关键函数是`getg`，`getg`是内建函数，从注释可以看出是从TLS中读取保存的g对象。

```go
// getg returns the pointer to the current g.
// The compiler rewrites calls to this function into instructions
// that fetch the g directly (from TLS or from the dedicated register).
func getg() *g
```

当然还有其它地方使用到了LTS，概括起来就是，在创建m时，将g0地址存放在线程的LTS中；在切换协程时，将切换后的协程地址存放在线程的LTS中；在需要获取当前协程的地方，从LTS中读取当前协程的g地址。
