# 0x06 Isolation & system call entry/exit

## 准备工作

阅读【1】中第4章，除了4.6；阅读RISCV.h【2】；阅读trampoline.S【3】；阅读trap.c【4】

【1】https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf

【2】https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/riscv.h

【3】https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trampoline.S

【4】https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/trap.c

## Outliner

## Trap 机制

程序运行时如何完成用户空间和内核空间的切换：

- 程序执行系统调用
- 程序出现类似page fault、运行时除以0等错误
- 一个设备触发了终端使得档期啊那程序运行需要相应内核设备驱动

都会发生切换。用户空间和内核空间的切换通常被称为trap，而trap涉及了许多小心的设计和重要的细节，这些细节对于实现安全隔离和性能来说非常重要。很多应用程序，要么因为系统调用，要么因为page fault，**都会频繁的切换到内核中，所以trap机制要尽可能的简单**

我们需要清楚如何让程序的运行，从只拥有user权限并且位于用户空间的程序，切换到拥有supervisor权限的内核。**在这个过程中，硬件的状态将会非常重要**，因为我们很多的工作都是将硬件从适合运行用户应用程序的状态，改变到适合运行内核代码的状态。其中值得关注的是32个寄存器的状态变化。这里的很多寄存器都有特殊的作用，其中一个特别有意思的寄存器是stack pointer（也叫做堆栈寄存器 stack register）。此外：

- 在硬件中还有一个寄存器叫做程序计数器（Program Counter Register）。
- 表明当前mode的标志位，这个标志位表明了当前是supervisor mode还是user mode。当我们在运行Shell的时候，自然是在user mode。
- 还有一堆控制CPU工作方式的寄存器，比如SATP（Supervisor Address Translation and Protection）寄存器，它包含了指向page table的物理内存地址
- STVEC（Supervisor Trap Vector Base Address Register）寄存器，它指向了内核中处理trap的指令的起始地址。
- SEPC（Supervisor Exception Program Counter）寄存器，在trap的过程中保存程序计数器的值。
- SSRATCH（Supervisor Scratch Register）寄存器，这也是个非常重要的寄存器

**这些寄存器表明了执行系统调用时计算机的状态。这些寄存器表明了执行系统调用时计算机的状态。**在trap处理的过程中，我们实际上需要更改一些这里的状态，或者对状态做一些操作。这样我们才可以运行系统内核中普通的C程序。

- 首先，我们需要**保存32个用户寄存器**。因为很显然我们需要恢复用户应用程序的执行，尤其是当用户程序随机的被设备中断所打断时。我们希望内核能够响应中断，之后在用户程序完全无感知的情况下再恢复用户代码的执行。所以这意味着32个用户寄存器不能被内核弄乱。但是这些寄存器又要被内核代码所使用，所以在trap之前，你必须先在某处保存这32个用户寄存器。
- **程序计数器也需要在某个地方保存**，它几乎跟一个用户寄存器的地位是一样的，**我们需要能够在用户程序运行中断的位置继续执行用户程序。**
- 需要将mode改成supervisor mode，以便使用内核中的各种各样的特权指令。
- SATP寄存器现在正指向user page table，而user page table只包含了用户程序所需要的内存映射和一两个其他的映射，它并没有包含整个内核数据的内存映射。所以**在运行内核代码之前，需要将SATP指向kernel page table。**
- 需要**将堆栈寄存器指向位于内核的一个地址**，因为我们需要一个堆栈来调用内核的C函数。
- 当所有的硬件状态都适合在内核中使用， 我们需要跳入内核的C代码。

我们不想让用户代码介入到这里的user/kernel切换，否则有可能会破坏安全性。所以这意味着，**trap中涉及到的硬件和内核机制不能依赖任何来自用户空间东西**。比如说我们不能依赖32个用户寄存器，它们可能保存的是恶意的数据，所以，XV6的trap机制不会查看这些寄存器，而只是将它们保存起来。

对于mode标志位，mode表明当前是user mode还是supervisor mode。但是有一点很重要：当这个标志位从user mode变更到supervisor mode时，我们能得到什么样的权限。实际上，这里获得的额外权限实在是有限：

- 其中的一件事情是，你现在可以读写控制寄存器了。

  - 读写SATP寄存器，也就是page table的指针
  - STVEC，也就是处理trap的内核指令地址
  - SEPC，保存当发生trap时的程序计数器
  - SSCRATCH等等

  > 在supervisor mode你可以读写这些寄存器，而用户代码不能做这样的操作。

- 它可以使用PTE_U标志位为0的PTE。

  > 当PTE_U标志位为1的时候，表明用户代码可以使用这个页表；如果这个标志位为0，则只有supervisor mode可以使用这个页表。

> 这两点就是supervisor mode可以做的事情，除此之外就不能再干别的事情了。

supervisor mode中的代码并不能读写任意物理地址。在supervisor mode中，就像普通的用户代码一样，也需要通过page table来访问内存。如果一个虚拟地址并不在当前由SATP指向的page table中，又或者SATP指向的page table中PTE_U=1，那么supervisor mode不能使用那个地址。

## Trap 代码执行流程

demo：跟踪如何在Shell中调用write系统调用。从Shell的角度来说，这就是个Shell代码中的C函数调用，但是实际上，write通过执行ECALL指令来执行系统调用。ECALL指令会切换到具有supervisor mode的内核中。

- 在这个过程中，内核中执行的第一个指令是一个由汇编语言写的函数，叫做uservec。这个函数是内核代码trampoline.s文件的一部分。所以执行的第一个代码就是这个uservec汇编函数。
- 之后，在这个汇编函数中，代码执行跳转到了由C语言实现的函数usertrap中，这个函数在trap.c中。
- 在usertrap这个C函数中，我们执行了一个叫做syscall的函数。
- 这个函数会在一个表单中，根据传入的代表系统调用的数字进行查找，并在内核中执行具体实现了系统调用功能的函数。对于我们来说，这个函数就是sys_write。
- sys_write会将要显示数据输出到console上，当它完成了之后，它会返回给syscall函数
- 因为我们现在相当于在ECALL之后中断了用户代码的执行，为了用户空间的代码恢复执行，需要做一系列的事情。在syscall函数中，会调用一个函数叫做**usertrapret**，它也位于trap.c中，这个函数完成了部分方便在C代码中实现的返回到用户空间的工作。
- 除此之外，最终还有一些工作只能在汇编语言中完成。这部分工作通过汇编语言实现，并且存在于trampoline.s文件中的userret函数中。
- 最终，在这个汇编函数中会调用机器指令返回到用户空间，并且恢复ECALL之后的用户程序的执行。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKFsfImgYCtnwA1d2hO%2F-MKHxleUqYy-y0mrS48w%2Fimage.png?alt=media&token=ab7c66bc-cf61-4af4-90fd-1fefc96c7b5f)

> 学生提问：这个问题或许并不完全相关，read和write系统调用，相比内存的读写，他们的代价都高的多，因为它们需要切换模式，并来回捣腾。有没有可能当你执行打开一个文件的系统调用时， 直接得到一个page table映射，而不是返回一个文件描述符？这样只需要向对应于设备的特定的地址写数据，程序就能通过page table访问特定的设备。你可以设置好限制，就像文件描述符只允许修改特定文件一样，这样就不用像系统调用一样在用户空间和内核空间来回捣腾了。
>
> Robert教授：这是个很好的想法。实际上很多操作系统都提供这种叫做内存映射文件（Memory-mapped file access）的机制，在这个机制里面通过page table，可以将用户空间的虚拟地址空间，对应到文件内容，这样你就可以通过内存地址直接读写文件。实际上，你们将在mmap 实验中完成这个机制。对于许多程序来说，这个机制的确会比直接调用read/write系统调用要快的多。

## ECALL 指令之前的状态

这里开始跟踪一个xv6的系统调用，即shell将它的提时信息通过write系统调用走到O/S然后再输出到console的过程。用户代码`sh.c`初始了这一切。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5OrMSu1sZV8JrCG4C%2F-ML5Q7oMs3EmQSXOpHZw%2Fimage.png?alt=media&token=d465e9a8-bb7b-4847-bf8e-89cc1e12c3ba)

上图中选中的行，是一个write系统调用，它将“$ ”写入到文件描述符2。接下来我将打开gdb并启动XV6。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5OrMSu1sZV8JrCG4C%2F-ML5QkklZGS46F7Ulkzx%2Fimage.png?alt=media&token=e1dfd288-9b00-4fe9-8502-408e59f7ca15)

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5OrMSu1sZV8JrCG4C%2F-ML5QkklZGS46F7Ulkzx%2Fimage.png?alt=media&token=e1dfd288-9b00-4fe9-8502-408e59f7ca15)

作为用户代码的Shell调用write时，实际上调用的是关联到Shell的一个库函数。你可以查看这个库函数的源代码，在usys.s。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5OrMSu1sZV8JrCG4C%2F-ML5RUaTjsUrdw2GrcYT%2Fimage.png?alt=media&token=54b07586-6e4d-4304-b399-3696cc0152f6)

上面这几行代码就是实际被调用的write函数的实现。这是个非常短的函数，它首先将SYS_write加载到a7寄存器，SYS_write是常量16。这里告诉内核，我想要运行第16个系统调用，而这个系统调用正好是write。之后这个函数中执行了ecall指令，从这里开始代码执行跳转到了内核。内核完成它的工作之后，代码执行会返回到用户空间，继续执行ecall之后的指令，也就是ret，最终返回到Shell中。所以ret从write库函数返回到了Shell中。

为了展示这里的系统调用，我会在ecall指令处放置一个断点，为了能放置断点，我们需要知道ecall指令的地址，我们可以通过查看由XV6编译过程产生的sh.asm找出这个地址。sh.asm是带有指令地址的汇编代码（注，asm文件3.7有介绍）。我这里会在ecall指令处放置一个断点，这条指令的地址是0xde6。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5OrMSu1sZV8JrCG4C%2F-ML5Tv0Tt2I5mrnm9wiZ%2Fimage.png?alt=media&token=4eb82e08-991e-4fb5-9841-b8cc78e0f4ba)

现在，我要让XV6开始运行。我期望的是XV6在Shell代码中正好在执行ecall之前就会停住。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5UF6bH4CcXBo4EXjF%2F-MLAYCSG9U40OjnDWtTs%2Fimage.png?alt=media&token=0c1cfe8f-d7c5-4311-8e13-15b178c787a4)

完美，从gdb可以看出，我们下一条要执行的指令就是ecall。我们来检验一下我们真的在我们以为自己在的位置，让我们来打印程序计数器（Program Counter），正好我们期望在的位置0xde6。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5UF6bH4CcXBo4EXjF%2F-MLAYi_F9KULTLcqdfJV%2Fimage.png?alt=media&token=1218fe66-5487-4119-8a0e-d427e521a680)

我们还可以输入*info reg*打印全部32个用户寄存器，

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5UF6bH4CcXBo4EXjF%2F-MLAZAzDe6phr2w8cY5e%2Fimage.png?alt=media&token=613b4b60-3c06-4c5d-b2e2-d1b83a4d941b)

这里有一些数值我们还不知道，也不关心，但是这里的a0，a1，a2是Shell传递给write系统调用的参数。所以a0是文件描述符2；a1是Shell想要写入字符串的指针；a2是想要写入的字符数。我们还可以通过打印Shell想要写入的字符串内容，来证明断点停在我们认为它应该停在的位置。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5UF6bH4CcXBo4EXjF%2F-MLA_SsIyQObazOHpYd9%2Fimage.png?alt=media&token=d0610673-48ba-477d-a537-fc278e565237)

可以看出，输出的确是美元符（$）和一个空格。所以，我们现在位于我们期望所在的write系统调用函数中。

有一件事情需要注意，上图的寄存器中，程序计数器（pc）和堆栈指针（sp）的地址现在都在距离0比较近的地址，这进一步印证了当前代码运行在用户空间，因为用户空间中所有的地址都比较小。但是一旦我们进入到了内核，内核会使用大得多的内存地址。

系统调用的时间点会有大量状态的变更，其中一个最重要的需要变更的状态，并且在它变更之前我们对它还有依赖的，就是是当前的page table。我们可以查看STAP寄存器。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5UF6bH4CcXBo4EXjF%2F-MLAbVqnXu4s5M_wDTKm%2Fimage.png?alt=media&token=9d38fcce-bec6-450d-9c70-6ebfc4648422)

这里输出的是物理内存地址，它并没有告诉我们有关page table中的映射关系是什么，page table长什么样。但是幸运的是，在QEMU中有一个方法可以打印当前的page table。从QEMU界面，输入*ctrl a + c*可以进入到QEMU的console，之后输入*info mem*，QEMU会打印完整的page table。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-ML5UF6bH4CcXBo4EXjF%2F-MLAcP8ULSwHAM90b_AM%2Fimage.png?alt=media&token=a02cc317-c767-46f4-9190-553ef4b5a072)

这是个非常小的page table，它只包含了6条映射关系。这是用户程序Shell的page table，而Shell是一个非常小的程序，这6条映射关系是有关Shell的指令和数据，以及一个无效的page用来作为guard page，以防止Shell尝试使用过多的stack page。我们可以看出这个page是无效的，因为在attr这一列它并没有设置u标志位（第三行）。attr这一列是PTE的标志位，第三行的标志位是rwx表明这个page可以读，可以写，也可以执行指令。之后的是u标志位，它表明PTE_u标志位是否被设置，用户代码只能访问u标志位设置了的PTE。再下一个标志位我也不记得是什么了（注，从4.3可以看出，这个标志位是Global）。再下一个标志位是a（Accessed），表明这条PTE是不是被使用过。再下一个标志位d（Dirty）表明这条PTE是不是被写过。

现在，我们有了这个小小的page table。顺便说一下，最后两条PTE的虚拟地址非常大，非常接近虚拟地址的顶端，如果你读过了XV6的书，你就知道这两个page分别是trapframe page和trampoline page。你可以看到，它们都没有设置u标志，所以用户代码不能访问这两条PTE。一旦我们进入到了supervisor mode，我们就可以访问这两条PTE了。

对于这里page table，有一件事情需要注意：它并没有包含任何内核部分的地址映射，这里既没有对于kernel data的映射，也没有对于kernel指令的映射。除了最后两条PTE，这个page table几乎是完全为用户代码执行而创建，所以它对于在内核执行代码并没有直接特殊的作用。

> 学生提问：PTE中a标志位是什么意思？
>
> Robert教授：这表示这条PTE是不是被代码访问过，是不是曾经有一个被访问过的地址包含在这个PTE的范围内。d标志位表明是否曾经有写指令使用过这条PTE。这些标志位由硬件维护以方便操作系统使用。对于比XV6更复杂的操作系统，当物理内存吃紧的时候，可能会通过将一些内存写入到磁盘来，同时将相应的PTE设置成无效，来释放物理内存page。你可以想到，这里有很多策略可以让操作系统来挑选哪些page可以释放。我们可以查看a标志位来判断这条PTE是否被使用过，如果它没有被使用或者最近没有被使用，那么这条PTE对应的page适合用来保存到磁盘中。类似的，d标志位告诉内核，这个page最近被修改过。
>
> 不过XV6没有这样的策略。

接下来，我会在Shell中打印出write函数的内容。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MLFmR-ec8XwDd9RO7RJ%2F-MLIg7ff-PZuCSvqWMhG%2Fimage.png?alt=media&token=36f63570-7fb0-4f9f-9353-4361efada276)

程序计数器现在指向ecall指令，我们接下来要执行ecall指令。现在我们还在用户空间，但是马上我们就要进入内核空间了。

## ECALL 指令之后的状态

现在我执行ecall指令，

## usertrap 函数

## usertrapret 函数

## userret 函数

