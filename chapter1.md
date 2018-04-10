<!-- Copyright © 2018 Clement Rey <cr.rey.clement@gmail.com>. -->
<!-- Licensed under the BY-NC-SA Creative Commons 4.0 International Public License. -->

```Bash
$ go version
go version go1.10 linux/amd64
```

# 第一章：Go的汇编入门

在我们开始研究go语言运行时和标准库的实现之前，有必要先熟悉一些Go语言的抽象汇编开发知识。
希望这份快速指南对您有所帮助。

---

**目录**
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- ["伪汇编"](#%E4%BC%AA%E6%B1%87%E7%BC%96)
- [分解一个简单的程序](#%E5%88%86%E8%A7%A3%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84%E7%A8%8B%E5%BA%8F)
  - [剖析`add`](#%E5%89%96%E6%9E%90add)
  - [剖析`main`](#%E5%89%96%E6%9E%90main)
- [简述goroutines，stacks和splits](#%E7%AE%80%E8%BF%B0goroutinesstacks%E5%92%8Csplits)
  - [Stacks](#stacks)
  - [Splits](#splits)
  - [一些不足之处](#%E4%B8%80%E4%BA%9B%E4%B8%8D%E8%B6%B3%E4%B9%8B%E5%A4%84)
- [结论](#%E7%BB%93%E8%AE%BA)
- [链接](#%E9%93%BE%E6%8E%A5)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

---

- *本章假设有汇编程序的基本知识。*
- *如果遇到特定操作系统的问题，请使用`linux/amd64`。*
- *我们将一直**开启**编译器优化。*
- *除非另有说明，引用的文本和注释请参考官方文档或代码。*

## "伪汇编"

Go编译器输出一个抽象的，可移植的汇编形式，它实际上并不映射到任何真实的硬件。Go汇编程序之后使用这个伪汇编输出来生成针对目标硬件的具体的，特定于机器的指令。
这个额外的层有许多好处，很容易移植到新的体系结构就是主要的好处之一。有关更多信息，请参阅本章末尾链接中列出的Rob Pike的Go编译器设计(*The Design of the Go Assembler*)。

>了解Go的汇编程序不是底层机器的直接表示是很最重要的事情。一些细节精确地映射到机器，但有些则不。这是因为编译器套件在通常的管道中不需要汇编程序。相反，编译器在一种半抽象指令集上运行，指令选择部分在代码生成后发生。汇编程序工作在半抽象表单上，所以当你看到像MOV这样的指令时，工具链实际上为该操作产生的内容可能根本不是移动指令，可能是清除或加载。或者它可能完全对应于具有该名称的机器指令。
一般来说，特定于机器的指令往往表现为自己，但更多常见指令例如：内存移动，子程序调用，返回等指令则更为抽象。细节因架构而异，我们对不精确性表示歉意;情况并不明确。

>汇编程序是一种剖析半抽象指令集的描述并将其转化为输入到链接器的指令的方法。

## 分解一个简单的程序

考虑下面的Go代码([direct_topfunc_call.go](./direct_topfunc_call.go)):
```Go
//go:noinline
func add(a, b int32) (int32, bool) { return a + b, true }

func main() { add(10, 32) }
```
*（注意`//go:noinline`编译器指令在这里... 不要忽视掉。）*

我们来编译这个程序集：
```
$ GOOS=linux GOARCH=amd64 go tool compile -S direct_topfunc_call.go
```
```Assembly
0x0000 TEXT		"".add(SB), NOSPLIT, $0-16
  0x0000 FUNCDATA	$0, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
  0x0000 FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x0000 MOVL		"".b+12(SP), AX
  0x0004 MOVL		"".a+8(SP), CX
  0x0008 ADDL		CX, AX
  0x000a MOVL		AX, "".~r2+16(SP)
  0x000e MOVB		$1, "".~r3+20(SP)
  0x0013 RET

0x0000 TEXT		"".main(SB), $24-0
  ;; ...omitted stack-split prologue...
  0x000f SUBQ		$24, SP
  0x0013 MOVQ		BP, 16(SP)
  0x0018 LEAQ		16(SP), BP
  0x001d FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x001d FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x001d MOVQ		$137438953482, AX
  0x0027 MOVQ		AX, (SP)
  0x002b PCDATA		$0, $0
  0x002b CALL		"".add(SB)
  0x0030 MOVQ		16(SP), BP
  0x0035 ADDQ		$24, SP
  0x0039 RET
  ;; ...omitted stack-split epilogue...
```

为了更好地理解编译器正在做什么，我们将逐行剖析这两个函数。

### 剖析`add`

```Assembly
0x0000 TEXT "".add(SB), NOSPLIT, $0-16
```

-  0x0000`：当前指令的偏移，相对于函数的开始。

- 'TEXT'".add`：`TEXT`指令声明`"".add`符号作为`.text`节的一部分（即可运行代码），并且指出后面的指令是函数的主体。在连接时，空字符串`""`将被当前包的名称替换：也就是，执行过链接后到我们的最终二进制文件，``".add`将变成`main.add`。

- `(SB)`：`SB`是保存"静态库"指针的虚拟寄存器，即程序地址空间开始的地址。
`"".add(SB)`声明我们的符号位于从地址空间开始的某个常量偏移量处（由链接器计算）。换句话说，它有一个绝对寻址(direct address)：它是一个全局函数符号。
'objdump'将为我们确认所有这些：
```
$ objdump -j .text -t direct_topfunc_call | grep 'main.add'
000000000044d980 g     F .text	000000000000000f main.add
```
> 所有用户定义的符号都被写为伪寄存器FP（参数和本地变量）和SB（全局变量）的偏移量。
> 伪寄存器SB可以被认为是内存的起始地址，所以符号foo(SB)是名称foo作为内存中的地址。

- `NOSPLIT`：指示编译器*不*应该插入* stack-split *前导码，它检查当前堆栈是否需要增长。
在我们的`add`函数中，编译器十分聪明的设置了标志：因为`add`没有本地变量，也没有自己的栈帧，所以它不能超过当前的栈;因此检查每个调用点完全是浪费CPU周期。
> "NOSPLIT"：不要插入前导码来检查堆栈是否必须拆分。例程的栈帧，加上它调用的任何东西，都必须放在堆栈顶部的空闲空间中。用于保护例程，如堆栈分割代码本身。
本章末尾，我们将简单介绍一下goroutines和分段栈(stack-split)。

- '$0-16`：`$0`表示将被分配的栈帧的大小（以字节为单位）而`$16`指定调用者传入的参数的大小。
>在一般情况下，帧大小由后面跟着一个由减号分隔的参数决定。（这不是一个减法，只是特殊的语法。）帧大小 $24-8 表示该函数有一个24字节的帧，并被调用8字节的参数，它们位于调用者的帧上。如果NOSPLIT没有为TEXT指定，则必须提供参数大小。对于使用Go原型的汇编函数，go vet会检查参数大小是否正确。

```Assembly
0x0000 FUNCDATA $0, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
0x0000 FUNCDATA $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
```

> 指令FUNCDATA和PCDATA包含供垃圾收集器使用的信息;它们由编译器引入。

现在不要担心这个;我们稍后会在书中回顾垃圾收集时再回到它。

```Assembly
0x0000 MOVL "".b+12(SP), AX
0x0004 MOVL "".a+8(SP), CX
```

Go调用约定要求每个参数必须在堆栈上传递，预留在调用者的堆栈帧上的空间。调用者有责任适当增大（并缩小）栈，以便将参数传递给被调用者，并将潜在的返回值传递给调用者。

Go编译器不会从PUSH / POP系列产生指令：通过分别递减或递增虚拟堆栈指针"SP"来增加或缩小堆栈。
> SP伪寄存器是一个虚拟堆栈指针，用于引用帧本地变量和为函数调用准备的参数。它指向本地堆栈帧的顶部，因此引用应使用范围[-framesize，0）中的负偏移量：x-8(SP)，y-4(SP)等等。

虽然官方文档指出："*所有用户定义的符号都被写为伪寄存器FP（参数和本地）*的偏移量"，这对于手写代码来说只会是真的。
像最新的编译器一样，Go工具套件总是直接在其生成的代码中使用来自堆栈指针的偏移量来引用参数和本地值。这允许帧指针在具有较少寄存器（例如x86）的平台上用作额外的通用寄存器。如果您喜欢这种细节，请参阅本章末尾的链接中的x86-64 *上的*框架布局。
* [更新：我们已在[问题＃2：帧指针]（https://github.com/teh-cmc/go-internals/issues/2）中讨论了此问题。]*

`"".b + 12(SP)`和`"".a + 8(SP)`分别表示堆栈顶部12字节和8字节的地址（记住：它向下增长！）。
`.a`和`.b`是指定位置的任意别名;尽管*它们绝对没有语义含义*无论如何，它们在虚拟寄存器上使用相对寻址时是强制性的。
有关虚拟帧指针的文档可以这样说：
> FP伪寄存器是用于引用函数参数的虚拟帧指针。编译器维护一个虚拟帧指针，并将堆栈中的参数作为该伪寄存器的偏移量。因此，0（FP）是函数的第一个参数，8（FP）是第二个参数（在64位机器上），依此类推。但是，以这种方式引用函数参数时，必须在first_arg + 0（FP）和second_arg + 8（FP）中放置一个名称。（来自帧指针的偏移量 - 偏移量的含义 - 与其在SB中的使用截然不同，其中它是与符号的偏移量。）汇编程序强制执行这个约定，拒绝纯0（FP）和8（FP）。实际名称在语义上不相关，但应该用来记录参数的名称。

最后，这里有两件重要的事情需要注意：
1. 第一个参数`a`不在`0(SP)`，而是`8(SP)`;这是因为调用者通过`CALL`伪指令将返回地址存储在`0(SP)`中。
2. 参数以相反的顺序传递;即第一个参数是最靠近栈顶的。

```Assembly
0x0008 ADDL CX, AX
0x000a MOVL AX, "".~r2+16(SP)
0x000e MOVB $1, "".~r3+20(SP)
```

`ADDL`实际添加了两个存储在"AX"和"CX"中的** L **字（即4字节值），然后将最终结果存储在"AX"中。然后将结果移到`"".(SP)r2 + 16(SP)`，调用者之前预留了一些堆栈空间并希望找到它的返回值。再一次，`"".(SP)r2`在这里没有语义含义。

为了演示Go如何处理多个返回值，我们还返回一个常量"true"布尔值。
汇编代码中与我们的第一个返回值完全相同;只有相对于SP的偏移量发生变化。
```Assembly
0x0013 RET
```

最后的`RET`伪指令告诉Go汇编器插入目标平台调用约定所需的任何指令，以便正确地从子程序调用中返回。
这很可能会导致代码弹出存储在"0(SP)"处的返回地址，然后跳转回去。
> TEXT块中的最后一条指令必须是某种跳转，通常是RET（伪）指令。
>（如果不是这样，链接器会附加一条跳到自身的指令; TEXT中不会出现跳转。）
这是很多语法和语义来一次摄取的。以下是我们刚刚介绍的内容摘要：

```Assembly
;; 声明全局函数符号"".add（实际上是main.add执行过链接后）
;; 不要插入堆栈分割前导码
;; 0字节的栈帧，16字节的参数传入
;; func add(a, b int32) (int32, bool)
0x0000 TEXT "".add(SB), NOSPLIT, $0-16
  ;; ...omitted FUNCDATA stuff...
  0x0000 MOVL "".b + 12(SP)，AX         ;;  将来自调用者的栈帧的第二个长字(4B)参数移入AX
  0x0004 MOVL "".a + 8(SP)，CX          ;;  将来自调用者的堆栈帧的第一个长字(4B)参数移入CX
  0x0008 ADDL CX，AX                    ;;  计算AX = CX + AX
  0x000a MOVL AX，"".(SP)r2 + 16(SP)    ;;  将加法结果(AX)移入调用者的栈帧
  0x000e MOVB $1，"".(SP)r3 + 20(SP)    ;;  将`true`布尔(常量)移入调用者的堆栈帧
  0x0013 RET                            ;;  跳转到存储在0(SP)的返回地址
```

总而言之，下面是main.add执行完成后堆栈看起来像什么的可视化表示：
```
   |    +-------------------------+ <-- 32(SP)
   |    |                         |
 G |    |                         |
 R |    |                         |
 O |    | main.main's saved       |
 W |    |     frame-pointer (BP)  |
 S |    |-------------------------| <-- 24(SP)
   |    |      [alignment]        |
 D |    | "".~r3 (bool) = 1/true  | <-- 21(SP)
 O |    |-------------------------| <-- 20(SP)
 W |    |                         |
 N |    | "".~r2 (int32) = 42     |
 W |    |-------------------------| <-- 16(SP)
 A |    |                         |
 R |    | "".b (int32) = 32       |
 D |    |-------------------------| <-- 12(SP)
 S |    |                         |
   |    | "".a (int32) = 10       |
   |    |-------------------------| <-- 8(SP)
   |    |                         |
   |    |                         |
   |    |                         |
 \ | /  | return address to       |
  \|/   |     main.main + 0x30    |
   -    +-------------------------+ <-- 0(SP) (TOP OF STACK)

(diagram made with https://textik.com)
```
<!-- https://textik.com/#af55d3485eaa56f2 -->

### 剖析`main`

我们会让你省去一些不必要的滚动，这里提醒我们main函数是什么样的：

```Assembly
0x0000 TEXT     "".main(SB), $24-0
  ;; ...omitted stack-split prologue...
  0x000f SUBQ       $24, SP
  0x0013 MOVQ       BP, 16(SP)
  0x0018 LEAQ       16(SP), BP
  ;; ...omitted FUNCDATA stuff...
  0x001d MOVQ       $137438953482, AX
  0x0027 MOVQ       AX, (SP)
  ;; ...omitted PCDATA stuff...
  0x002b CALL       "".add(SB)
  0x0030 MOVQ       16(SP), BP
  0x0035 ADDQ       $24, SP
  0x0039 RET
  ;; ...omitted stack-split epilogue...
```

```Assembly
0x0000 TEXT"".main(SB)，$24-0
```

这里没有新东西：
-`"".main`（`main.main`执行过链接后）是一个全局函数符号`.text`部分，其地址是从我们的地址空间开始的一些不变的偏移量。
- 它分配一个24字节的栈帧，不会收到任何参数，也不返回任何值。

```Assembly
0x000f SUBQ     $24, SP
0x0013 MOVQ     BP, 16(SP)
0x0018 LEAQ     16(SP), BP
```

正如我们上面提到的，Go调用约定要求每个参数都必须在堆栈上传递。

我们的调用者`main`，通过减少虚拟堆栈指针来增加堆栈帧24字节（*记住堆栈向下增长，所以`SUBQ`实际上使堆栈帧变大*）。
在这24字节中：
- 8字节（`16(SP)`-`24(SP)`）用于存储帧指针"BP"的当前值（*真实值*）允许堆栈展开并且方便调试
- 1+3字节（`12(SP)`-`16(SP)`）为第二个返回值（`bool`）加上3字节为`amd64`做必要对齐
- 4字节（`8(SP)`-`12(SP)`）保留为参数`b（int32）`的值
- 4字节（`0(SP)`-`4(SP)`）被保留用于参数`a（int32）'的值

最后，随着堆栈的增长，`LEAQ`重新计算帧指针的地址并将其存储在`BP`中。


```Assembly
0x001d MOVQ     $137438953482, AX
0x0027 MOVQ     AX, (SP)
```

调用程序将被调用者的参数作为四倍字（**q**uad word 即一个8字节的值）推送到刚刚增长的堆栈顶部。
尽管起初它可能看起来像随机垃圾，但实际上，"137438953482"对应于连接成一个8字节值的"10"和"32"4字节值：
```
$ echo 'obase=2;137438953482' | bc
10000000000000000000000000000000001010
\_____/\_____________________________/
   32                             10
```

```Assembly
0x002b CALL     "".add(SB)
```

`CALL`的`add`函数作为相对于静态指针的偏移量：这是直接跳转到直接地址的简单方法。
请注意，`CALL`也会将堆栈顶部的返回地址（8字节值）所以从'add'函数中对`SP`的每个引用最终都会被8字节抵消！例如`"".a`不再是'0(SP)'，而是'8(SP)`。
```Assembly
0x0030 MOVQ     16(SP), BP
0x0035 ADDQ     $24, SP
0x0039 RET
```

最终：
1. 将帧指针展开一个栈帧（即我们"向下"一级）
2. 将堆栈缩小24字节以回收先前分配的堆栈空间
3. 请Go编译器插入子例程返回相关的东西

## 简述goroutines，stacks和splits

稍后我们再去研究goroutines的内部，但是当我们接触越来越多的汇编栈，与栈管理相关的指令将迅速成为非常熟悉的景象。

### Stacks

由于Go程序中的goroutine数量并不确定，并且在运行中可能高达数百万，运行时必须采用保守的方式为goroutines分配栈空间以避免耗尽所有可用内存。
因此，每个新的goroutine在运行时都会被赋予一个最初的小型2kB栈（实际上栈是分配在堆上）。
随着goroutine一直在做其工作，它可能最终超过了它最初的栈空间（即栈溢出）。
为了防止这种情况发生，运行时确保当goroutine的栈用完时，将分配一个新的更大的栈，其大小是旧栈的两倍，并且将原栈的内容复制到新栈上。
这个过程被称为* 分段栈(stack-split) *，并有效地使得goroutine栈动态调整大小。

### Splits

为了配合栈分离，编译器在每个栈可能溢出函数的开始和结尾处插入一些指令。
正如我们在本章前面所看到的，为了避免不必要的开销，不可能栈溢出的函数被标记为'NOSPLIT`，提示编译器不插入这些检查。

让我们看看上面的主函数，这次不需要省略栈分离前导码：
```Assembly
0x0000 TEXT "".main(SB), $24-0
  ;; stack-split prologue
  0x0000 MOVQ   (TLS), CX
  0x0009 CMPQ   SP, 16(CX)
  0x000d JLS    58

  0x000f SUBQ   $24, SP
  0x0013 MOVQ   BP, 16(SP)
  0x0018 LEAQ   16(SP), BP
  ;; ...omitted FUNCDATA stuff...
  0x001d MOVQ   $137438953482, AX
  0x0027 MOVQ   AX, (SP)
  ;; ...omitted PCDATA stuff...
  0x002b CALL   "".add(SB)
  0x0030 MOVQ   16(SP), BP
  0x0035 ADDQ   $24, SP
  0x0039 RET

  ;; stack-split epilogue
  0x003a NOP
  ;; ...omitted PCDATA stuff...
  0x003a CALL   runtime.morestack_noctxt(SB)
  0x003f JMP    0
```
您可以看到，栈分离前导码被分为序言(prologue)和后记(epilogue)：
- 序言检查是否存在空间不足，如果是的情况下，跳到结尾。
- 另一方面，后记则触发堆叠机械，然后跳回序幕。

这会创建一个反馈循环，只要没有为goroutine分配足够大的堆栈，就会继续循环。

**序言**
```Assembly
0x0000 MOVQ	(TLS), CX   ;; 将 *g 存储到寄存器CX
0x0009 CMPQ	SP, 16(CX)  ;; 比较SP 和 g.stackguard0
0x000d JLS	58	    ;; 调转到 0x3a if SP <= g.stackguard0
```

TLS是由运行时维护的虚拟寄存器，它保存指向当前g的指针，即指向一个goroutine的所有状态的数据结构。
从`runtime`的源代码看`g`的定义：
```Go
type g struct {
    stack       stack   // 16 bytes
    // stackguard0 is the stack pointer compared in the Go stack growth prologue.
    // It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
    stackguard0 uintptr
    stackguard1 uintptr

    // ...omitted dozens of fields...
}
```
我们可以看到`16(CX)`对应`g.stackguard0`，它是由运行时维护的阈值，与堆栈指针相比较时，它指示goroutine是否即将用完了空间。
因此，序言检查当前的'SP'值是否小于或等于`stackguard0'阈值（也就是说它更大），然后在碰巧情况下跳到结尾。

**后记**
```Assembly
0x003a NOP
0x003a CALL	runtime.morestack_noctxt(SB)
0x003f JMP	0
```

后记的主体非常简单：调用运行时，它将执行堆栈增长的实际工作，然后跳回到函数的第一条指令（即序言）。

`CALL`之前的`NOP`指令使得序言不会直接跳到'CALL`指令上。在某些平台上，这样做会出现归一的结果;在实际调用之前设置无指令(nop)并登陆该`NOP`是一种普遍的做法。
* [更新：我们已经在[问题＃4：澄清"nop before call"段落]（https://github.com/teh-cmc/go-internals/issues/4）中讨论了此事。]*

### 一些不足之处

我们仅仅在这里介绍了go汇编的冰山一角。栈增长还有很多巧妙的处理机制，在这里很难一一提到：整个过程是一个相当复杂的机器总体，需要一个自己的章节。
我们会及时回到这些问题上。

## 结论

Go的汇编程序的这个快速入门应该给你足够的材料来开始讨论。

随着我们对本书其余部分的Go内部深入挖掘，Go汇编将成为我们最依赖的工具之一，它可以理解幕后发生的事情，可能乍一看并不明显。
如果您有任何问题或建议，请不要犹豫，用`chapter1：`前缀开启一个问题！

## 链接

- [[Official] A Quick Guide to Go's Assembler](https://golang.org/doc/asm)
- [[Official] Go Compiler Directives](https://golang.org/cmd/compile/#hdr-Compiler_Directives)
- [[Official] The design of the Go Assembler](https://www.youtube.com/watch?v=KINIAgRpkDA)
- [[Official] Contiguous stacks Design Document](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub)
- [[Official] The `_StackMin` constant](https://github.com/golang/go/blob/bf86aec25972f3a100c3aa58a6abcbcc35bdea49/src/runtime/stack.go#L70-L71)
- [[Discussion] Issue #2: *Frame pointer*](https://github.com/teh-cmc/go-internals/issues/2)
- [[Discussion] Issue #4: *Clarify "nop before call" paragraph*](https://github.com/teh-cmc/go-internals/issues/4)
- [A Foray Into Go Assembly Programming](https://blog.sgmansfield.com/2017/04/a-foray-into-go-assembly-programming/)
- [Dropping Down Go Functions in Assembly](https://www.youtube.com/watch?v=9jpnFmJr2PE)
- [What is the purpose of the EBP frame pointer register?](https://stackoverflow.com/questions/579262/what-is-the-purpose-of-the-ebp-frame-pointer-register)
- [Stack frame layout on x86-64](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64)
- [How Stacks are Handled in Go](https://blog.cloudflare.com/how-stacks-are-handled-in-go/)
- [Why stack grows down](https://gist.github.com/cpq/8598782)
