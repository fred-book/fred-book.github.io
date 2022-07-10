### 入参传递

------

当执行可执行程序的时候，经常会在执行命令中携带参数如下所示：

```shell
main para1 para2
```

这部分参数是怎样在启动过程中传递的呢？

1. 在启动函数`_rt0_amd64`中：

```assembly
TEXT _rt0_amd64(SB),NOSPLIT,$-8
   MOVQ   0(SP), DI  // argc
   LEAQ   8(SP), SI  // argv
   JMP    runtime·rt0_go(SB)
```

将栈上的入参保存到DI和SI寄存器（至于在此之前入参是怎么保存到栈上的不展开说明）。

2. 在`rt0_go`函数中，

```assembly
TEXT runtime·rt0_go(SB),NOSPLIT|TOPFRAME,$0
	// copy arguments forward on an even stack
	MOVQ	DI, AX		// 拷贝入参argc到AX寄存器
	MOVQ	SI, BX		// 拷贝入参argv到BX寄存器
    ...  
	MOVQ	AX, 24(SP)  // 将AX值(argc)存入SP+24栈地址上
	MOVQ	BX, 32(SP)  // 将BX值(argv)存入SP+32栈地址上
	...
	MOVL	24(SP), AX		// 将SP+24值（argc）赋值给AX寄存器
	MOVL	AX, 0(SP)       // 将AX值（argc）保存到SP栈地址上
	MOVQ	32(SP), AX		// 将SP+32值（argv）赋值给AX寄存器
	MOVQ	AX, 8(SP)       // 将AX值（argv）保存到SP+8栈地址上
    CALL	runtime·args(SB)      // 调用runtime·args函数
    CALL	runtime·osinit(SB)    // 调用runtime·osinit函数
	CALL	runtime·schedinit(SB) // 调用runtime·schedinit函数
	...
```

将入参存入栈`SP+0`和`SP+8`地址上（构造栈上的`Callee arg`），并调用`runtime·args`函数，将入参保存到全局变量argc和argv上

```go
func args(c int32, v **byte) {
   argc = c
   argv = v
   sysargs(c, v)
}
```

3. 在`runtime·schedinit`函数中调用`goargs()`函数，将argv以字符串的形式保存到全局切片变量argslice中。

```go
func goargs() {
   if GOOS == "windows" {
      return
   }
   argslice = make([]string, argc)
   for i := int32(0); i < argc; i++ {
      argslice[i] = gostringnocopy(argv_index(argv, i))
   }
}
```

4. 在`os`包初始化的时候调用`runtime_args`（即`runtime`包中`os_runtime_args`函数），将全局变量argslice拷贝一份赋值给`os`包全局变量`Args`

```go
var Args []string

func init() {
   if runtime.GOOS == "windows" {
      // Initialized in exec_windows.go.
      return
   }
   Args = runtime_args()
}
```

```go
//go:linkname os_runtime_args os.runtime_args
func os_runtime_args() []string { return append([]string{}, argslice...) }
```

那么，使用者可以在代码中访问`os.Args`获得程序入参。
