# 0x05 Calling conventions and stack frames RISC-V

## 准备工作

阅读【1】

【1】https://pdos.csail.mit.edu/6.828/2020/readings/riscv-calling.pdf

## Outliners

- C->asm /processors
- RISV-V 
- Registers
- Stack + calling conventions
- statck layout in memory

调用约定（Calling Convention）是规定子过程如何获取参数以及如何返回的方案，其通常与架构、编译器等相关。具体来说，调用约定一般规定了

- 参数、返回值、返回地址等放置的位置（寄存器、栈或存储器等）
- 如何将调用子过程的准备工作与恢复现场的工作划分到调用者（Caller）与被调用者（Callee）身上

## C 程序到汇编程序的转换

首先来简单看一下C语言是如何转换成汇编语言的。 通常来说，我们的C语言程序会有一个main函数，假设在这个函数内你执行了一些打印然后退出了。

```c
// c -> asm
int main() {
    print();
    exit();
}
```

但是处理器并不能理解C语言。处理器能够理解的是汇编语言，或者更具体的说，**处理器能够理解的是二进制编码之后的汇编代码。**当我们说到一个RISC-V处理器时，意味着这个处理器能够理解RISC-V的指令集。所以，任何一个处理器都有一个关联的ISA（Instruction Sets Architecture）。每一条指令都有一个对应的二进制编码或者一个**Opcode**。

所以通常来说，要让C语言能够运行在你的处理器之上。我们首先要写出C程序，之后这个C程序需要被编译成汇编语言。之后汇编语言会被翻译成二进制文件也就是.obj或者.o文件。

## RISC-V vs x86

- 首先是指令的数量。实际上，创造RISC-V的一个非常大的初衷就是因为Intel手册中指令数量太多了。x86-64指令介绍由3个文档组成，并且新的指令以每个月3条的速度在增加。因为x86-64是在1970年代发布的，所以我认为现在有多于15000条指令。RISC-V指令介绍由两个文档组成。在这节课中，不需要你们记住每一个RISC-V指令，但是如果你感兴趣或者你发现你不能理解某个具体的指令的话，在课程网站的参考页面有RISC-V指令的两个文档链接。这两个文档包含了RISC-V的指令集的所有信息，分别是240页和135页，相比x86的指令集文档要小得多的多。这是有关RISC-V比较好的一个方面。所以在RISC-V中，我们有更少的指令数量。

- 除此之外，RISC-V指令也更加简单。在x86-64中，很多指令都做了不止一件事情。这些指令中的每一条都执行了一系列复杂的操作并返回结果。但是RISC-V不会这样做，RISC-V的指令趋向于完成更简单的工作，相应的也消耗更少的CPU执行时间。这其实是设计人员的在底层设计时的取舍。并没有一些非常确定的原因说RISC比CISC更好。它们各自有各自的使用场景。

- 相比x86来说，RISC另一件有意思的事情是它是开源的。这是市场上唯一的一款开源指令集，这意味着任何人都可以为RISC-V开发主板。RISC-V是来自于UC-Berkly的一个研究项目，之后被大量的公司选中并做了支持，网上有这些公司的名单，许多大公司对于支持一个开源指令集都感兴趣。

## gdb 和汇编代码执行



## RISC-V 寄存器

这部分内容来自准备材料。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MM-XjiGboAFe-3YvZfT%2F-MM0rYc4eVnR9nOesAAv%2Fimage.png?alt=media&token=f30ebac8-8dc0-4b5d-8aa7-b241a10b43b3)

这个表里面是RISC-V寄存器。寄存器是CPU或者处理器上，预先定义的可以用来存储数据的位置。寄存器之所以重要是因为汇编代码并不是在内存上执行，而是在寄存器上执行，也就是说，当我们在做add，sub时，我们是对寄存器进行操作。所以你们通常看到的汇编代码中的模式是，我们通过load将数据存放在寄存器中，这里的数据源可以是来自内存，也可以来自另一个寄存器。之后我们在寄存器上执行一些操作。如果我们对操作的结果关心的话，我们会将操作的结果store在某个地方。这里的目的地可能是内存中的某个地址，也可能是另一个寄存器。这就是通常使用寄存器的方法。

**寄存器是用来进行任何运算和数据读取的最快的方式。**当我们调用函数时，你可以看到这里有a0 - a7寄存器。通常我们在谈到寄存器的时候，我们会用它们的ABI名字。不仅是因为这样描述更清晰和标准，同时也因为在写汇编代码的时候使用的也是ABI名字。第一列中的寄存器名字并不是超级重要，它唯一重要的场景是在RISC-V的Compressed Instruction中。基本上来说，RISC-V中通常的指令是64bit，但是在Compressed Instruction中指令是16bit。在Compressed Instruction中我们使用更少的寄存器，也就是x8 - x15寄存器。我猜你们可能会有疑问，为什么s1寄存器和其他的s寄存器是分开的**，因为s1在Compressed Instruction是有效的，而s2-11却不是。**除了Compressed Instruction，寄存器都是通过它们的ABI名字来引用。

a0到a7寄存器是用来作为函数的参数。如果一个函数有超过8个参数，我们就需要用内存了。从这里也可以看出，当可以使用寄存器的时候，我们不会使用内存，我们只在不得不使用内存的场景才使用它。

表单中的第4列，Saver列，当我们在讨论寄存器的时候也非常重要。它有两个可能的值Caller，Callee：

- Caller Saved寄存器在函数调用的时候不会保存
- Callee Saved寄存器在函数调用的时候会保存

**一个Caller Saved寄存器可能被其他函数重写。**假设我们在函数a中调用函数b，任何被函数a使用的并且是Caller Saved寄存器，调用函数b可能重写这些寄存器。一个比较好的例子就是Return address寄存器（注，保存的是函数返回的地址），你可以看到ra寄存器是Caller Saved，这一点很重要，**当函数a调用函数b的时侯，b会重写Return address。**所以基本上来说，任何一个Caller Saved寄存器，作为调用方的函数要小心可能的数据可能的变化；任何一个Callee Saved寄存器，作为被调用方的函数要小心寄存器的值不会相应的变化。

所有的寄存器都是64bit，各种各样的数据类型都会被改造的可以放进这64bit中。比如说我们有一个32bit的整数，取决于整数是不是有符号的，会通过在前面补32个0或者1来使得这个整数变成64bit并存在这些寄存器中。

> **Q**：除了Stack Pointer和Frame Pointer，我不认为我们需要更多的Callee Saved寄存器。
>
> **A**：0 - s11都是Callee寄存器，我认为它们是提供给编译器而不是程序员使用。在一些特定的场景下，你会想要确保一些数据在函数调用之后仍然能够保存，这个时候编译器可以选择使用s寄存器。

## Stack

**栈**之所以很重要的原因是，它**使得我们的函数变得有组织，且能够正常返回**。下面是一个非常简单的栈的结构图，其中每一个区域都是一个Stack Frame，**每执行一次函数调用就会产生一个Stack Frame**。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MM3Hk7Gv6ibvM2lxjCc%2F-MM4D2J3t3ajqkngxRPC%2Fimage.png?alt=media&token=1f78ffd1-9322-4666-85f2-8aa831ced49e)

每次调用一个函数，函数都会为自己创建一个Stack Frame，并且只给自己用。函数通过移动Stack Pointer来完成Stack Frame的空间分配。

对于Stack来说，是从高地址开始向低地址使用。所以栈总是向下增长。当我们想要创建一个新的Stack Frame的时候，总是对当前的Stack Pointer做减法。一个函数的Stack Frame包含了保存的寄存器，本地变量，并且，如果函数的参数多于8个，额外的参数会出现在Stack中。所以Stack Frame大小并不总是一样，即使在这个图里面看起来是一样大的。不同的函数有不同数量的本地变量，不同的寄存器，所以Stack Frame的大小是不一样的。但是有关Stack Frame有两件事情是确定的：

- Return address总是会出现在Stack Frame的第一位
- 指向前一个Stack Frame的指针也会出现在栈中的固定位置

有关Stack Frame中有两个重要的寄存器：

- 第一个是SP（Stack Pointer），**它指向Stack的底部并代表了当前Stack Frame的位置**。
- 第二个是FP（Frame Pointer），**它指向当前Stack Frame的顶部**。因为Return address和指向前一个Stack Frame的的指针都在当前Stack Frame的固定位置，所以可以通过当前的FP寄存器寻址到这两个数据。

> **保存前一个Stack Frame的指针的原因是为了让我们能跳转回去。**所以当前函数返回时，我们可以将前一个Frame Pointer存储到FP寄存器中。所以我们使用Frame Pointer来操纵我们的Stack Frames，并确保我们总是指向正确的函数。

Stack Frame必须要被汇编代码创建，因此是编译器生成了汇编代码，进而创建了Stack Frame。

在汇编代码中，函数的最开始你们可以看到Function prologue，之后是函数的本体，最后是Epollgue。这就是一个汇编函数通常的样子。



## Struct

struct在内存中是一段连续的地址，如果我们有一个struct，并且有f1，f2，f3三个字段。

```c
struct {
    f1
    f2
    f3
}
```

当我们创建这样一个struct时，内存中相应的字段会彼此相邻。可以认为**struct像是一个数组，但是里面的不同字段的类型可以不一样。**

