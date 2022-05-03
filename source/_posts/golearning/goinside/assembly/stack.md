#### 调用栈

------

在程序运行的过程中，会涉及函数的调用，函数返回后继续执行本函数剩下的代码。程序是如何组织这一调用过程的呢，就是靠栈。

##### 栈

栈是程序执行单元（线程或者协程）在执行过程中记录程序上下文的数据结构，用于保存函数调用过程中的各种数据（包含本地变量，参数，返回值等）。每一个程序执行单元都有一个独立的栈，伴随其整个生命周期，这样保证了每个执行单元能够并行执行。

栈由栈帧组成，每一个调用函数对应一个栈帧。栈空间从高地址向低地址生长，栈底指向栈起始地址（高地址），栈顶执行栈终止地址（低地址）

![image-20220501155623356](/assets/images/golearing/image-20220501155623356.png)

当调用函数时，则从当前栈顶往下扩一个栈帧；当从函数执行完返回，则往上缩一个栈帧。

##### 栈帧

每个函数在栈中对应的片段叫做栈帧，也就是stack frame。栈帧记录了函数调用需要的上下文信息。在1.18版本，Go的栈帧如下：

![image-20220501220333663](/assets/images/golearing/image-20220501220333663.png)

栈帧有以下5部分组成：

Caller BP：保存调用函数的栈基地址（BP），用于函数返回后获得调用函数的栈帧基地址。

Local Var：保存函数内部本地变量。

temporarily unused space：保存在函数运行过程中产生的临时变量。

Return Temp Var：保存函数返回值临时变量。

Callee Arg：保存被调用函数的返回值。

Return Address：保存被调用函数返回后的程序地址，即本函数调用被调用函数的下一条指令地址。

注意：Return Address实际并不是在创建函数栈的时候生成，而是在调用Call函数时生成，函数RET时释放。

栈帧的创建和释放过程如下伪汇编代码：

```assembly
TEXT funcb, , $framesize-argumentsize
    SUBQ $framesize, SP //将SP减去栈帧大小，即开辟栈帧
    MOVQ BP, $framesize-8(SP)//将调用函数的BP地址保存到Caller BP栈地址上
    LEAQ $framesize-8(SP), BP//将当前栈基地址SP+framesize-8存入BP寄存器
    ...
    CALL .funca //调用函数funca，将SP减去8，并向栈中压入下一条指令地址即Return Address
    ...
    MOVQ $framesize-8(SP), BP//将SP+framesize-8保存的Caller BP存入BP寄存器
    ADDQ $framesize, SP //将SP加栈帧大小，即释放栈帧
    RET//返回函数，将栈顶保存的返回地址弹出到IP寄存器，并将SP加8，释放Return Address
```

每个函数都遵循以上的栈处理过程，其中省略部分为函数体执行过程中，其中涉及栈空间存取过程。

##### 代码分析

下面以一个简单的函数调用关系来说明栈空间的处理过程，源代码如下：

```go
package math

//go:nosplit
func add(a, b int) int {
	return a + b
}

//go:nosplit
func calc(a, b int) int {
	c := add(a, b)

	return c
}
```

通过编译生成汇编代码如下：

```assembly
TEXT    "".add(SB), NOSPLIT|ABIInternal, $16-16
	SUBQ    $16, SP
	MOVQ    BP, 8(SP)
	LEAQ    8(SP), BP
	FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	FUNCDATA        $5, "".add.arginfo1(SB)
	PCDATA  $0, $-2
	MOVQ    AX, "".a+24(SP)
	MOVQ    BX, "".b+32(SP)
	MOVQ    $0, "".~r0(SP)
	MOVQ    "".a+24(SP), AX
	ADDQ    "".b+32(SP), AX
	MOVQ    AX, "".~r0(SP)
	MOVQ    8(SP), BP
	ADDQ    $16, SP
	RET

TEXT    "".calc(SB), NOSPLIT|ABIInternal, $40-16
	SUBQ    $40, SP
	MOVQ    BP, 32(SP)
	LEAQ    32(SP), BP
	FUNCDATA        $0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	FUNCDATA        $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
	FUNCDATA        $5, "".calc.arginfo1(SB)
	PCDATA  $0, $-2
	MOVQ    AX, "".a+48(SP)
	MOVQ    BX, "".b+56(SP)
	MOVQ    $0, "".~r0+16(SP)
	MOVQ    "".b+56(SP), BX
	MOVQ    "".a+48(SP), AX
	PCDATA  $1, $0
	CALL    "".add(SB)
	MOVQ    AX, "".c+24(SP)
	MOVQ    AX, "".~r0+16(SP)
	MOVQ    32(SP), BP
	ADDQ    $40, SP
	RET
```

相信在学习了前面知识的基础上，大家基本上能看懂上述代码的意思。下面仅对栈空间的处理过程进行描述。

1. 开辟calc函数栈空间，栈帧长度为40字节；将BP保存到Caller BP栈地址上，并将BP寄存器设置成当前栈帧BP地址。

   ```assembly
       SUBQ    $40, SP
       MOVQ    BP, 32(SP)
       LEAQ    32(SP), BP
   ```

   ![image-20220501210656748](/assets/images/golearing/image-20220501210656748.png)

2. 将参数保存到调用函数的Callee Arg栈地址上。

   ```assembly
   	MOVQ    AX, "".a+48(SP)
   	MOVQ    BX, "".b+56(SP)
   ```

   ![image-20220501211002563](/assets/images/golearing/image-20220501211002563.png)

3. 将返回值Callee Ret栈地址上的值清零，并将入参分别赋值给AX，BX寄存器。

   ```assembly
   	MOVQ    $0, "".~r0+16(SP)
   	MOVQ    "".b+56(SP), BX
   	MOVQ    "".a+48(SP), AX
   ```

   ![image-20220501211535383](/assets/images/golearing/image-20220501211535383.png)

4. 调用函数add，将`CALL    "".add(SB)`的下一条指令地址压入栈中。

   ```assembly
   	CALL    "".add(SB)
   ```

   ![image-20220501211920086](/assets/images/golearing/image-20220501211920086.png)
   
5. 执行add函数，开辟add函数栈帧，大小为16字节；同样保存Caller BP和更新BP寄存器指向地址。

   ```assembly
   	SUBQ    $16, SP
   	MOVQ    BP, 8(SP)
   	LEAQ    8(SP), BP
   ```

   ![image-20220501212436303](/assets/images/golearing/image-20220501212436303.png)

6. 将参数保存到调用函数的Callee Arg栈地址上。

   ```assembly
   	MOVQ    AX, "".a+24(SP)
   	MOVQ    BX, "".b+32(SP)
   ```

   ![image-20220501212819453](/assets/images/golearing/image-20220501212819453.png)

7. 将返回值栈地址的值清零；并将参数相加，并将结果保存到AX寄存器；将计算结果存入Callee Ret栈地址上。

   ```assembly
   MOVQ    $0, "".~r0(SP)
   MOVQ    "".a+24(SP), AX
   ADDQ    "".b+32(SP), AX
   MOVQ    AX, "".~r0(SP)
   ```

   ![image-20220501214302219](/assets/images/golearing/image-20220501214302219.png)

8. 将calc's BP保存到BP，并释放add函数栈帧。

   ```assembly
   MOVQ    8(SP), BP
   ADDQ    $16, SP
   ```

   ![image-20220501213539086](/assets/images/golearing/image-20220501213539086.png)

9. add函数执行完返回，将当前栈顶保存的返回地址存入IP寄存器。

   ```assembly
   	RET
   ```

   ![image-20220501213737186](/assets/images/golearing/image-20220501213737186.png)

10. add执行完返回后继续执行。将add函数执行结果分别保存到本地变量c和返回值栈地址上。

    ```assembly
    	MOVQ    AX, "".c+24(SP)
    	MOVQ    AX, "".~r0+16(SP)
    ```

    ![image-20220501214057420](/assets/images/golearing/image-20220501214057420.png)

11. 将调用calc的函数栈帧的BP保存到BP寄存器，并释放calc栈帧。

    ```assembly
        MOVQ    32(SP), BP
        ADDQ    $40, SP
    ```

    ![image-20220501214715398](/assets/images/golearing/image-20220501214715398.png)

12. calc函数行完返回，将当前栈顶保存的返回地址存入IP寄存器。

    ```assembly
    	RET
    ```

    ![image-20220501214843593](/assets/images/golearing/image-20220501214843593.png)

上述详细描述了函数调用过程中，函数栈帧的变化。但其中还有如下疑问：

> 问题1. 栈帧大小计算规则？
>
> 问题2. 参数栈地址和返回值栈地址上的值赋值规则？
>
> 问题3. 超多入参或者超多返回值的处理规则？

这些将在后面的章节中讲到。

