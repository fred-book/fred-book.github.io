### 基础知识

------

先来看一段代码

```go
package math

func add(a, b int) int {
	return a + b
}
```

使用`go tool compile -N -L -S math.go`生成汇编代码如下：

```assembly
0x0000 00000 (math.go:4)        TEXT    "".add(SB), NOSPLIT|ABIInternal, $16-16
0x0000 00000 (math.go:4)        SUBQ    $16, SP
0x0004 00004 (math.go:4)        MOVQ    BP, 8(SP)
0x0009 00009 (math.go:4)        LEAQ    8(SP), BP
0x000e 00014 (math.go:4)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
0x000e 00014 (math.go:4)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
0x000e 00014 (math.go:4)        FUNCDATA        $5, "".add.arginfo1(SB)
0x000e 00014 (math.go:4)        PCDATA  $0, $-2
0x000e 00014 (math.go:4)        MOVQ    AX, "".a+24(SP)
0x0013 00019 (math.go:4)        MOVQ    BX, "".b+32(SP)
0x0018 00024 (math.go:4)        MOVQ    $0, "".~r0(SP)
0x0020 00032 (math.go:5)        MOVQ    "".a+24(SP), AX
0x0025 00037 (math.go:5)        ADDQ    "".b+32(SP), AX
0x002a 00042 (math.go:5)        MOVQ    AX, "".~r0(SP)
0x002e 00046 (math.go:5)        MOVQ    8(SP), BP
0x0033 00051 (math.go:5)        ADDQ    $16, SP
0x0037 00055 (math.go:5)        RET
```

从上面的代码可以看出，汇编代码由指令，寄存器，常量以及一些关键字组成。

#### 指令

指令也称为机器码，是CPU的最小执行单元。指令是由CPU类型决定的，比如x86架构CPU支持的指令集和ARM架构的不一样，64位CPU和32位CPU也不一样。而汇编语言只是为指令起了一个别名，最终会转换成CPU支持的指令。Plan 9共有几百个指令（详见Plan 9指令集），其中常用的指令有如下几类：

**移动类指令**

移动类指令一般是将常量，栈地址，寄存器中保存的值赋值给寄存器或者栈地址，指令形式一般是`MOVX SRC, DST`，其中MOVX中的X表示数据类型，SRC表示源数据，DST表示目的寄存器或者栈地址。如下表列出常用的几个移动类指令：

![image-20220430215832785](/assets/images/golearing/image-20220430215832785.png)

**计算类指令**

计算类指令包含了整型加减乘除，位移，位运算，地址运算等运算功能。如下表列出常用的几个计算类指令：

![image-20220430215844880](/assets/images/golearing/image-20220430215844880.png)

**比较类指令**

比较类运算用来比较两个整型是否相等。如下表列出常用的几个比较类指令：

![image-20220430215857613](/assets/images/golearing/image-20220430215857613.png)

**跳转类指令**

跳转指令包含无条件跳转和有条件跳转两类，表示在无条件或者条件成立下PC跳转到另外一个地址。如下表列出常用的几个跳转类指令：

![image-20220430215905429](/assets/images/golearing/image-20220430215905429.png)

**操作类指令**

操作类指令是指各种基础操作功能的指令。如下表列出常用的几个操作类指令：

![image-20220430215913780](/assets/images/golearing/image-20220430215913780.png)

**调用类指令**

调用类指令是指汇编中调用其它函数过程中使用的指令，包括CALL和RET两个，如下表所示：

![image-20220430215921491](/assets/images/golearing/image-20220430215921491.png)

**伪指令**

同时，为了配合运行时优化，Plan 9定了一些伪指令。如下表列出常用的几个伪指令：

![image-20220430215928151](/assets/images/golearing/image-20220430215928151.png)

#### 寄存器

寄存器是CPU片上存储单元，空间极小但访问速率极高，用来辅助完成指令操作。x86-64系统有18个通用寄存器，它们的作用及在Plan 9中的名称如下表所示：

![image-20220430222019084](/assets/images/golearing/image-20220430222019084.png)

另外为了编程便利，Plan 9引入如下4个伪寄存器：

FP: Frame Pointer: arguments

> 指向函数的入参起始位子，使用形式为`symbol+offset(FP)`，如函数的第1个参数表示为`arg1+0(FP)`，第2个参数为`arg2+8(FP)`等。其中symbol是必须有的，在汇编角度来看，无实际意义仅仅为了提升代码可读性。
>
> FP指向的地址并不在本栈帧内，而是指向调用本函数的调用函数的栈帧中。

PC: Program Counter: jumps and branches

> 在x86-64系统上，PC实际是IP寄存器，指向当前执行的指令地址。

SB: Static Base Pointer: global symbols

> 全局静态基指针，一般用来声明函数和全局变量。

SP: Stack Pointer: locals

> 指向当前栈帧的局部变量的起始位置，使用形式为`symbol+offset(SP)`，如第1个本地变量表示为`localvar1-8(SP)`，第2个变量为`localvar2-16(SP)`（注：假设变量长度为8字节）。
>
> 该寄存器与硬件SP（栈顶）指向的地址不一样。如果在手写代码中使用硬件寄存器SP，则需要写成`offset(SP)`形式。
>
> **特别注意：对于编译（如go tool compile -S或者go tool objdump）输出的汇编代码，其中的SP均表示硬件寄存器SP，不论代码中是否制定symbol。**

#### 关键字

**声明变量（DATA，GLOBL）**

汇编中的变量，一般是指保存在.rodata和.data段中的只读值，对应于代码，则是已经初始化过的全局的const，var，static变量。

使用格式如下：

```assembly
DATA symbol+offset(SB)/width, value
GLOBL symbol(SB), flag, $width
```

其中：

offset：为相对symbol地址的偏移

flag：为常量类型，其取值如下：

> - `NOPROF` = 1
>   (For `TEXT` items.) Don't profile the marked function. This flag is deprecated.
> - `DUPOK` = 2
>   It is legal to have multiple instances of this symbol in a single binary. The linker will choose one of the duplicates to use.
> - `NOSPLIT` = 4
>   (For `TEXT` items.) Don't insert the preamble to check if the stack must be split. The frame for the routine, plus anything it calls, must fit in the spare space remaining in the current stack segment. Used to protect routines such as the stack splitting code itself.
> - `RODATA` = 8
>   (For `DATA` and `GLOBL` items.) Put this data in a read-only section.
> - `NOPTR` = 16
>   (For `DATA` and `GLOBL` items.) This data contains no pointers and therefore does not need to be scanned by the garbage collector.
> - `WRAPPER` = 32
>   (For `TEXT` items.) This is a wrapper function and should not count as disabling `recover`.
> - `NEEDCTXT` = 64
>   (For `TEXT` items.) This function is a closure so it uses its incoming context register.
> - `LOCAL` = 128
>   This symbol is local to the dynamic shared object.
> - `TLSBSS` = 256
>   (For `DATA` and `GLOBL` items.) Put this data in thread local storage.
> - `NOFRAME` = 512
>   (For `TEXT` items.) Do not insert instructions to allocate a stack frame and save/restore the return address, even if this is not a leaf function. Only valid on functions that declare a frame size of 0.
> - `TOPFRAME` = 2048
>   (For `TEXT` items.) Function is the outermost frame of the call stack. Traceback should stop at this function.

例如声明一个pi常量：

```assembly
DATA pi+0(SB)/8 3.1415926
GLOBL pi(SB), RODATA, $8
```

例如声明一个字符串常量：

```assembly
DATA apple+0(SB)/8 "this is "
DATA apple+8(SB)/8 "an apple"
GLOBL apple(SB), RODATA, $16
```

**声明函数（TEXT）**

使用TEXT声明函数，主要是因为函数存储在TEXT段中，格式如下：

```assembly
                                     栈帧大小（局部变量+可能需要的额外调用函数的参数空间的总大小）
                                        |
TEXT pkgname·funcname(SB), flags, $framesize-argumentsize
        |        |                                |
       包名      函数名                            参数和返回值总大小
```

其中：

pkgname为包名，可以省略，为了代码可读性，建议写上。

funcname为函数名，与包名之间用Unicode的中点连接。

framesize为栈帧大小

argumentssize为入参元素总大小

#### 代码分析

在学习完上述基础知识后，再回到节首的汇编代码就比较容易理解了。这里只解释字面意思，其中原理在下一节介绍。

```assembly
0x0000 00000 (math.go:4)        TEXT    "".add(SB), NOSPLIT|ABIInternal, $16-16
```

声明函数add，其中pkgname用“”代替，在链接时将替换成math包名。函数栈帧大小为16字节，参数大小为16字节（2个int大小）

```assembly
0x0000 00000 (math.go:4)        SUBQ    $16, SP
0x0004 00004 (math.go:4)        MOVQ    BP, 8(SP)
0x0009 00009 (math.go:4)        LEAQ    8(SP), BP
```

开辟16字节的栈空间，将当前BP寄存器地址存入SP+8地址上，并将当前SP+8地址保存到BP寄存器。

```assembly
0x000e 00014 (math.go:4)        FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
0x000e 00014 (math.go:4)        FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
0x000e 00014 (math.go:4)        FUNCDATA        $5, "".add.arginfo1(SB)
0x000e 00014 (math.go:4)        PCDATA  $0, $-2
```

伪指令不做过多解释。

```assembly
0x000e 00014 (math.go:4)        MOVQ    AX, "".a+24(SP)
0x0013 00019 (math.go:4)        MOVQ    BX, "".b+32(SP)
0x0018 00024 (math.go:4)        MOVQ    $0, "".~r0(SP)
```

将AX保存的值存入SP+24地址上，将BX保存的值存入SP+32地址上，将0存入SP地址上。

```assembly
0x0020 00032 (math.go:5)        MOVQ    "".a+24(SP), AX
0x0025 00037 (math.go:5)        ADDQ    "".b+32(SP), AX
0x002a 00042 (math.go:5)        MOVQ    AX, "".~r0(SP)
```

将SP+24地址上的值赋值给AX，将AX的值与SP+32地址上的值相加并保存到AX，将AX的值存入SP地址上。

```assembly
0x002e 00046 (math.go:5)        MOVQ    8(SP), BP
0x0033 00051 (math.go:5)        ADDQ    $16, SP
```

将SP+8地址上的值存入BP寄存器，将SP加上16。

```assembly
0x0037 00055 (math.go:5)        RET
```

函数返回。