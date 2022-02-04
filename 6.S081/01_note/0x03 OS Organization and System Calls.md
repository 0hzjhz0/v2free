# 0x03 OS Organization and System Calls

## 准备工作

阅读【1】中的chap.2

【1】https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf

## Outline

讨论 4 个话题：

- isolation
- kernel & user mode - 隔离操作系统内核和用户应用程序
- system calls - 应用程序能够切换到内核执行的基本方法
- 上述主题在 xv6 中的实现

## Isolation

> 操作系统的隔离性。

如果没有隔离性：

- 程序直接跟硬件接触，程序A发生异常（死循环）占用CPU资源，导致其他程序无法使用CPU
- 直接访问物理内存，出现踩内存的情况
- ...

> 操作系统提供了了**硬件资源的复用**（multiplexing）和**强隔离**

从隔离的角度来稍微看看Unix接口，那么我们可以发现，接口被精心设计以实现资源的强隔离，也就是multiplexing和物理内存的隔离。**接口通过抽象硬件资源**，从而使得提供强隔离性成为可能。例如：

1. **进程-CPU**。之前通过fork创建了进程。进程本身不是CPU，但是它们对应了CPU，它们使得你可以在CPU上运行计算任务。所以你懂的，应用程序不能直接与CPU交互，只能与进程交互。操作系统内核会完成不同进程在CPU上的切换。所以，操作系统不是直接将CPU提供给应用程序，而是向应用程序提供“进程”，进程抽象了CPU，这样操作系统才能在多个应用程序之间复用一个或者多个CPU。
2. **exec - 内存** 。可以认为exec抽象了内存。当我们在执行exec系统调用的时候，我们会传入一个文件名，而这个文件名对应了一个应用程序的内存镜像。内存镜像里面包括了程序对应的指令，全局的数据。应用程序可以逐渐扩展自己的内存，但是应用程序并没有直接访问物理内存的权限，例如应用程序不能直接访问物理内存的1000-2000这段地址。不能直接访问的原因是，操作系统会提供内存隔离并控制内存，操作系统会在应用程序和硬件资源之间提供一个中间层。exec是这样一种系统调用，它表明了应用程序不能直接访问物理内存。
3. **file - 磁盘**。files基本上来说抽象了磁盘。应用程序不会直接读写挂在计算机上的磁盘本身，并且在Unix中这也是不被允许的。在Unix中，与存储系统交互的唯一方式就是通过files。Files提供了非常方便的磁盘抽象，你可以对文件命名，读写文件等等。之后，操作系统会决定如何将文件与磁盘中的块对应，确保一个磁盘块只出现在一个文件中，并且确保用户A不能操作用户B的文件。通过files的抽象，可以实现不同用户之间和同一个用户的不同进程之间的文件强隔离。

## Defensive

> 操作系统防御性

- 应用程序不能使得O/S崩溃
- 应用程序不能打破对它的隔离

实际应用程序或许是恶意的，这意味着需要在应用程序和O/S之间提供强隔离性。

> 通常需要通过硬件来实现强隔离性，这里的硬件支持包括：**user/kernel mode**（kernel mode 在 RSIC-V 中被称为 supervisor mode）；**page table**（或者说虚拟内存）

所有的处理器，如果需要运行能够支持多个应用程序的O/S，需要同时支持user/kernel mode 和虚拟内存。

## 硬件对于强隔离的支持

> 硬件对强隔离的支持包括：**`user/kernel mode`** 和**`虚拟内存`**。

### user/kernel mode

为了支持user/kernel mode，**处理器会有两种操作模式**，第一种是user mode，第二种是kernel mode。当运行在kernel mode时，CPU可以运行特定权限的指令（privileged instructions）；当运行在user mode时，CPU只能运行普通权限的指令（unprivileged instructions）。

- 普通权限指令：例如将两个寄存器相加的指令ADD、将两个寄存器相减的指令SUB、跳转指令JRC、BRANCH指令等等

  > 所有的应用程序都允许执行这些指令

- 特殊权限指令：主要是一些**直接操纵硬件的指令**和**设置保护的指令**，例如**设置page table寄存器、关闭时钟中断**

  > 应用程序不应该执行这些指令，这些指令只能被内核执行

  > 当一个应用程序尝试执行一条特殊权限指令，因为不允许在user mode执行特殊权限指令，处理器会拒绝执行这条指令。通常来说，这时会将控制权限从user mode切换到kernel mode，当操作系统拿到控制权之后，或许会杀掉进程，因为应用程序执行了不该执行的指令。

> 实际上RISC-V还有第三种模式称为machine mode。在大多数场景下，**会忽略这种模式**

**Q**：如果kernel mode允许一些指令的执行，user mode不允许一些指令的执行，那么是谁在检查当前的mode并实际运行这些指令，并且怎么知道当前是不是kernel mode？是有什么标志位吗？

**A**：在处理器里面有一个flag。在处理器的一个bit，当它为1的时候是user mode，当它为0时是kernel mode。当处理器在解析指令时，如果指令是特殊权限指令，并且该bit被设置为1，处理器会拒绝执行这条指令，就像在运算时不能除以0一样。

**Q**：设置处理器中kernel mode的bit位的指令是一条特殊权限指令，那么一个用户程序怎么才能让内核执行任何内核指令？因为现在切换到kernel mode的指令都是一条特殊权限指令了，对于用户程序来说也没法修改那个bit位

**A**：在RISC-V中，**如果你在用户空间（user space）尝试执行一条特殊权限指令，用户程序会通过系统调用来切换到kernel mode**。当用户程序执行系统调用，会通过ECALL触发一个软中断（software interrupt），软中断会查询操作系统预先设定的中断向量表，并执行中断向量表中包含的中断处理程序。中断处理程序在内核中，这样就完成了user mode到kernel mode的切换，并执行用户程序想要执行的特殊权限指令。

### 虚拟内存

基本上所有的CPU都支持虚拟内存。处理器包含了page table，而**page table将虚拟内存地址与物理内存地址做了对应**。

**每一个进程都会有自己独立的page table**，每一个进程只能访问出现在自己page table中的物理内存。操作系统会设置page table，使得每一个进程都有不重合的物理内存，这样一个进程就不能访问其他进程的物理内存，因为其他进程的物理内存都不在它的page table中。一个进程甚至都不能随意编造一个内存地址，然后通过这个内存地址来访问其他进程的物理内存。这样就给了我们内存的强隔离性。

## user/kernel mode 切换

可以将user/kernel mode 看成分割用户空间和内核空间的边界。用户空间运行的程序运行在user mode，内核空间的程序运行在kernel mode。操作系统位于内核空间。

需要有一种方式能够让应用程序可以将控制权转移给内核（Entering Kernel），在RISC-V中，有一个专门的指令用来实现这个功能，叫做**ECALL**。

> ECALL接收一个数字参数，当一个用户程序想要**将程序执行的控制权转移到内核**，它**只需要执行ECALL指令**，并传入一个数字。这里的**数字参数代表了应用程序想要调用的System Call**。

`这里有图。`

ECALL会跳转到内核中一个特定，由内核控制的位置。(xv6 中存在一个唯一的系统调用接入点，每次应用程序执行ECALL，都会通过这个接入点进入内核)

> 举个例子，不论是Shell还是其他的应用程序，当它在用户空间执行fork时，它并不是直接调用操作系统中对应的函数，而是调用ECALL指令，并将fork对应的数字作为参数传给ECALL。之后再通过ECALL跳转到内核。

`这里有图。`

**注意**

> 用户空间和内核空间的界限是一个硬性的界限，用户不能直接调用fork，**应用程序执行系统调用的唯一方法就是通过这里的ECALL指令。**

假设我现在要执行另一个系统调用write，相应的流程是类似的，write系统调用不能直接调用内核中的write代码，而是由封装好的系统调用函数执行ECALL指令。所以write函数实际上调用的是ECALL指令，指令的参数是代表了write系统调用的数字。之后控制权到了syscall函数，syscall会实际调用write系统调用。

`这里有图。`

**Q**：操作系统在什么时候检查是否允许执行fork或者write？现在看起来应用程序只需要执行ECALL再加上系统调用对应的数字就能完成调用，但是内核在什么时候决定这个应用程序是否有权限执行特定的系统调用？

**A**：原则上来说，在内核侧实现fork的位置可以实现任何的检查，例如检查系统调用的参数，并决定应用程序是否被允许执行fork系统调用。在Unix中，任何应用程序都能调用fork，我们以write为例吧，write的实现需要检查传递给write的地址（需要写入数据的指针）属于用户应用程序，这样内核才不会被欺骗从别的不属于应用程序的位置写入数据

**Q**：当应用程序表现的恶意或者就是在一个死循环中，内核是如何夺回控制权限的？

**A**：内核会通过硬件设置一个定时器，定时器到期之后会将控制权限从用户空间转移到内核空间，之后内核就有了控制能力并可以重新调度CPU到另一个进程中。我们接下来会看一些更加详细的细节。

## Monolithic Kernel vs Micro Kernel 宏内核 vs 微内核

现在，我们有了一种方法，可以**通过系统调用或者说ECALL指令，将控制权从应用程序转到操作系统中**。之后内核负责实现具体的功能并检查参数以确保不会被一些坏的参数所欺骗，所以内核有时候也被称为**可被信任的计算空间**（`TCB, Trusted Computing Base`）。

要被称为TCB需要尽力满足两个条件：

1. 内核正确且没有Bug

   > 假设内核中有Bug，攻击者可能会利用那个Bug，并将这个Bug转变成漏洞，这个漏洞使得攻击者可以打破操作系统的隔离性并接管内核。

2. 内核必须要将用户应用程序或者进程当做是恶意的

什么程序应该运行在kernel mode？

- 敏感的代码肯定是运行在kernel mode，因为这是Trusted Computing Base。

### 宏内核

> 整个操作系统代码都运行在kernel mode。大多数的Unix操作系统实现都运行在kernel mode，比如，XV6中，所有的操作系统服务都在kernel mode中

特点：

- 如果考虑Bug的话，这种方式不太好。在一个宏内核中，任何一个操作系统的Bug都有可能成为漏洞。因为我们现在在内核中运行了一个巨大的操作系统，出现Bug的可能性更大了。你们可以去查一些统计信息，平均每3000行代码都会有几个Bug，所以如果有许多行代码运行在内核中，那么出现严重Bug的可能性也变得更大。所以从安全的角度来说，在内核中有大量的代码是宏内核的缺点。
- 如果你去看一个操作系统，它包含了各种各样的组成部分，比如说文件系统，虚拟内存，进程管理，这些都是操作系统内实现了特定功能的子模块。**宏内核的优势在于，因为这些子模块现在都位于同一个程序中，它们可以紧密的集成在一起，这样的集成提供很好的性能。**例如Linux，它就有很不错的性能。

### 微内核

> 主要关注点是减少内核中的代码。在这种模式下，希望在kernel mode中运行尽可能少的代码。所以这种设计下还是有内核，但是内核只有非常少的几个模块，例如，内核通常会有一些IPC的实现或者是Message passing；非常少的虚拟内存的支持，可能只支持了page table；以及分时复用CPU的一些支持。

特点：

- 将大部分的操作系统运行在内核之外。

  >  还是会有user mode以及user/kernel mode的边界，但是会将原来在内核中的其他部分，作为普通的用户程序来运行。比如文件系统可能就是个常规的用户空间程序。

- 因为在内核中的代码的数量较小，更少的代码意味着更少的Bug。

- 性能变差。

  > 假设我们需要让Shell能与文件系统交互，比如Shell调用了exec，必须有种方式可以接入到文件系统中。通常来说，这里工作的方式是，Shell会通过内核中的IPC系统发送一条消息，内核会查看这条消息并发现这是给文件系统的消息，之后内核会把消息发送给文件系统。文件系统会完成它的工作之后会向IPC系统发送回一条消息说，这是你的exec系统调用的结果，之后IPC系统再将这条消息发送给Shell。
  >
  > 里是典型的**通过消息来实现传统的系统调用**。现在，对于任何文件系统的交互，都需要分别完成2次用户空间<->内核空间的跳转。与宏内核对比，在宏内核中如果一个应用程序需要与文件系统交互，只需要完成1次用户空间<->内核空间的跳转，所以微内核的的跳转是宏内核的两倍。

**微内核的挑战在于性能更差**，这里有两个方面需要考虑：

- 在user/kernel mode反复跳转带来的性能损耗。
- 在一个类似宏内核的紧耦合系统，各个组成部分，例如文件系统和虚拟内存系统，可以很容易的共享page cache。**而在微内核中，每个部分之间都很好的隔离开了，这种共享更难实现。进而导致更难在微内核中得到更高的性能。**

## 编译运行kernel

`xv6`代码主要分为三部分：

- `kernel` - 包含了基本上所有的内核文件。`xv6` 是宏内核的，这里所有的文件会被编译成一个叫做 `kernel` 的二进制文件，然后这个二进制文件会被运行在 `kernle mode` 中。

- `user` - 基本是运行在 `user mode` 的程序。
- `mkfs`  - 会创建一个空的文件镜像，这个镜像会存在磁盘上，这样就有一个可以直接使用的空的文件系统。

**内核编译**

`Makefile` 会创建 `kernel.asm`（包含了内核的完整汇编），打开查看可以知道第一个指令地址位于 `0x80000000`，内核入口为 `_entry`

`make qemu` 指令中传递给qemu的几个参数：

- `-kernel`：这里传递的是内核文件（kernel目录下的kernel文件），这是将在QEMU中运行的程序文件
- `-m`：这里传递的是RISC-V虚拟机将会使用的内存容量
- `-smp`：这里传递的是虚拟机可以使用的CPU核数
- `-drive`：传递的是虚拟机使用的磁盘驱动，这里传入的是fs.img文件

## xv6 启动过程

1. 启动 `QEMU`，并打开 `gdb` - `make CPUS=1 qemu-gdb`

   >  `QEMU` 内部有一个 `gdb server` 支持 `gdb`，当我们启动之后，``QEMU` 会等待 `gdb` 客户端连接。

   ![image-20220204205724860](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204205724860.png)

2. 另起一个 `gdb` 客户端（这里是一个RISC-V 64位Linux的 `gdb`， 我的环境里是 `gdb-multiarch`），并在程序入口处打断点

   ![image-20220204211656566](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204211656566.png)

   这里提示了 `gdb` 内核的方式，在`~/.gdbinit`中加入 `add-auto-load-safe path /mng/d/linux/xv6-labs-2020/.gdbinit`，然后重新执行 `gdb-multiarch` 即可调试 `kernel`

   ![image-20220204213816545](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204213816545.png)

   > 设置完断点之后，我运行程序，可以发现代码并没有停在 `0x8000000`（见 `kernel.asm` ，`0x80000000` 是程序的起始位置），而是停在了 `0x80000004`

   **注意：** `0x80000000`  是一个被QEMU认可的地址，如果你想使用 `QEMU`，那么第一个指令地址必须是它。`xv6` 的内核加载器 `kernel.ld` 定义了内核如何被加载

   ![image-20220204214014702](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204214014702.png)

   这里可以看到内核使用的起始地址是 `QEMU` 指定的 `0x80000000` 这个地址。从这里也可以看到 `xv6` 是从 `entry.s` 开始启动，见下图

   ![image-20220204214123489](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204214123489.png)

   此时没有内存分页，隔离性，并且运行于 `M-mode`(machine mode)。`kernel` 会尽快跳转到 `kernel mode`（riscv 中也叫 `supervisor mode`）

3. 进入 `kernel mode`

   > 这里断住 `main` 函数（运行于 `kernel mode`）

   ![image-20220204214230542](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204214230542.png)

   `gdb` 下执行 `layout split` 可以看到具体执行到哪条语句，断点的位置。

   ![image-20220204214350903](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204214350903.png)

4. `main()` 做的初始化工作

   ![image-20220204214757026](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204214757026.png)

   `main` 完成操作系统的初始化工作，包括：

   - `consoleinit` 初始化 `console` 并向 `console` 打印一些内容
   - `kinit` 设置好页表分配器（page allocator）
   - `kvminit` 设置好(内核)虚拟内存
   - `kvminithart` 打开页表
   - `procinit` 设置好进程表单（或者说设置好初始进程）
   - `trapinit/trapinithart`  设置好 `user/kernel mode` 转换代码
   - `plicinit/plicinithart` 设置好中断控制器PLIC（Platform Level Interrupt Controller）
   - `binit `分配buffer cache
   - `iinit` 初始化inode缓存
   - `fileinit` 初始化文件系统
   - `virtio_disk_init` 初始化磁盘
   - `userinit` 最后当所有的设置都完成了，操作系统也运行起来了，会通过userinit运行第一个进程

5. `userinit`

   通过 `gdb` 的 `s` 指令跳转到 `userinit` 内部

   ![image-20220204220033013](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204220033013.png)

   ![image-20220204220113840](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204220113840.png)

   上图是 `gdb` 试图，下图是 `userinit` 源码，`userinit` 有点像 `Glue code`(胶水代码，不实现具体功能，只为了适配不同部分)，它利用了 `xv6` 的特性，并启动了第一个进程。

   > 我们总是需要有一个用户进程在运行才能实现与操作系统的交互，所以这里需要有一个小程序来初始化第一个用户进程，这个小程序定义在 `initcode` 中，它已经被二进制编码到程序中了，它会链接或者在内核中静态定义。

   ![image-20220204220442123](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204220442123.png)

   实际上这段代码对应了如下汇编程序

   ![image-20220204220658489](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204220658489.png)

   这段汇编程序首先将 `init` 的地址加载到 `a0` - `la a0, init`，`argv` 中的地址加载到 `a1` `la a1, argv`，`exec`  系统调用的数据加载到 `a7` - `li a7, SYS_exec`， 最后调用 `ecall` 将控制权限交给操作系统


6. 系统调用 `syscall` 

   在 `syscall` 设置断点，并运行。`userinit` 会创建初始进程，返回到用户空间，执行`5` 里说的 3 条指令，然后再回到内核空间。这里是 `xv6` 中任何用户都会使用到的第一个系统调用。

   ![image-20220204221256885](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204221256885.png)

   当前进入到 `syscall` 中，对应的代码如下

   ![image-20220204221403498](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204221403498.png)

   `num = p->trapframe->a7` 会读取使用的系统调用对应的整数，当代码执行完这行后可以在 `gdb` 中打印 `num`，得到结果如下

   ![image-20220204221533491](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204221533491.png)

   如果查看 `syscall.h` 可以看到对应是 `exec`

   ![image-20220204221623573](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204221623573.png)

   > 这里本事上是告诉内核，某个用户程序执行了 `ecall` 指令，并且想要调用 `exec` 系统调用

   `p->trapframe->a0 = syscall[num]()` 执行了系统调用，可以预期`syscall[7]`对应了`exec`的入口函数，接下来就跳转到这个函数中

   ![image-20220204221955356](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204221955356.png)

   ![image-20220204222050569](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204222050569.png)

   `sys_exec` 做的第一件事是从用户空间读取参数，他会读取 `path` 也就是要执行程序的文件名（这里先为参数分配空间，然后从用户空间将参数拷贝到内核空间），打印 `path`

   ![image-20220204222420053](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204222420053.png)

   可以看到，传入的是 `init` 程序。

   综合来看，`initcode` 完成了通过 `exec` 调用 `init` 程序。

7. `user/init.c`

   ![image-20220204222907314](C:\Users\hzji2\AppData\Roaming\Typora\typora-user-images\image-20220204222907314.png)

   `init`  会为用户空间设置好一些东西，比如配置好 `console`，调用 `fork`，并在 `fork` 出的子进程中执行 `shell`。

 

上图是userinit函数，右边是源码，左边是gdb视图。userinit有点像是胶水代码/Glue code（胶水代码不实现具体的功能，只是为了适配不同的部分而存在），它利用了XV6的特性，并启动了第一个进程。**我们总是需要有一个用户进程在运行，这样才能实现与操作系统的交互，所以这里需要一个小程序来初始化第一个用户进程**。这个小程序定义在initcode中。