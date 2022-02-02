# 0x04 Page tables

## 0 准备工作

阅读【1】中第3章；阅读memlayout.h【2】；阅读vm.c【3】；阅读kalloc【4】；阅读riscv.h【5】；阅读exec.c【6】

【1】https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf

【2】https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/memlayout.h

【3】https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/vm.c

【4】https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/kalloc.c

【5】https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/riscv.h

【6】https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/exec.c

## 1 Outline

1. 讨论地址空间（Address Spaces）
2. 支持虚拟内存的硬件
3. xv6 中虚拟内存代码，内核地址空间和用户地址空间的结构

## 2 地址空间 Address Spaces

> 不同进程有自己的地址空间，且只能在自己的地址空间上操作内存，以此实现内存隔离。

## 3 页表 Page Table

页表是实现地址空间的一种方式（在一个物理内存上创建不同的地址空间。）页表在硬件中通过处理器和内存管理单元实现。对于一条指令，其中的地址都是虚拟内存地址，将会被转到内存管理单元（MMU, Memory Management Unit）翻译成物理地址，这个物理地址将被用来索引物理内存，并从物理内存中加载或向物理内存存储数据。

MMU 中存在一个表单实现虚拟内存向物理内存的翻译。这个地址翻译表单也存储在物理内存中，因此CPU中有一个特殊的寄存器用于存放表单在物理内存中的地址，在RISC-V中通过寄存器SATP存储这个表单的物理内存地址。

**注意**

> 这里表单存储的粒度显然不能是地址。因为对于64bit的寄存器，可以表示的地址范围是 2^64 个，这个表单之大物理内存都存不下

这里分两步介绍RISC-V中page table是如何工作的。

第一步：不为每个地址创建一个表单条目，而是为每个page创建一个表单条目，每次地址翻译都是针对一个page。在RISC-V中，一个page大小4KB，也就是4096Bytes。

首先对于虚拟内存地址，可以被分成index和offset两个部分。index用于查找page，offset对应于page中哪个字节。当MMU在做地址翻译的时候，通过读取虚拟内存地址中的index可以知道物理内存中的page号，这个page号对应了物理内存中的4096个字节。之后虚拟内存地址中的offset指向了page中的4096个字节中的某一个，假设offset是12，那么page中的第12个字节被使用了。将offset加上page的起始地址，就可以得到物理内存地址。

**注意**

> 有关RISC-V的一件有意思的事情是，虚拟内存地址都是64bit，这也说的通，因为RISC-V的寄存器是64bit的。但是实际上，在我们使用的RSIC-V处理器上，并不是所有的64bit都被使用了，也就是说高25bit并没有被使用。这样的结果是限制了虚拟内存地址的数量，虚拟内存地址的数量现在只有2^39个，大概是512GB。当然，如果必要的话，最新的处理器或许可以支持更大的地址空间，只需要将未使用的25bit拿出来做为虚拟内存地址的一部分即可。
>
> 在RISC-V中，物理内存地址是56bit。
>
> 物理内存地址是56bit，其中44bit是物理page号（PPN，Physical Page Number），剩下12bit是offset完全继承自虚拟内存地址（也就是地址转换时，只需要将虚拟内存中的27bit翻译成物理内存中的44bit的page号，剩下的12bitoffset直接拷贝过来即可）。



通过前面的第一步，我们现在是的地址转换表是以page为粒度，而不是以单个内存地址为粒度，现在这个地址转换表已经可以被称为page table了。但是目前的设计还不能满足实际的需求。

**Q**：如果每个进程都有自己的page table，那么每个page table表会有多大呢？

**A**：这个page table最多会有2^27个条目（虚拟内存地址中的index长度为27），这是个非常大的数字。如果每个进程都使用这么大的page table，进程需要为page table消耗大量的内存，并且很快物理内存就会耗尽。

**从概念上来说，你可以认为page table是从0到2^27**。**实际中，page table是一个多级的结构**。下图是一个真正的RISC-V page table结构和硬件实现。

![image-20220201144826116](D:\linux\v2free\6.S081\01_note\0x04 Page tables.assets\image-20220201144826116.png)

之前提到的虚拟内存地址中的27bit的index，实际上是由3个9bit的数字组成（L2，L1，L0）。前9个bit被用来索引最高级的page directory（注：通常page directory是用来索引page table或者其他page directory物理地址的表单，但是在课程中，page table，page directory， page directory table区分并不明显，可以都认为是有相同结构的地址对应表单）。

一个directory是4096Bytes，就跟page的大小是一样的。Directory中的一个条目被称为PTE（Page Table Entry）是64bits，就像寄存器的大小一样，也就是8Bytes。所以一个Directory page有512个条目。

- 所以实际上，SATP寄存器会指向最高一级的page directory的物理内存地址，之后我们用虚拟内存中index的高9bit用来索引最高一级的page directory，这样我们就能得到一个PPN，也就是物理page号。这个PPN指向了中间级的page directory。

- 当我们在使用中间级的page directory时，我们通过虚拟内存地址中的L1部分完成索引。
- 接下来会走到最低级的page directory，我们通过虚拟内存地址中的L0部分完成索引。在最低级的page directory中，我们可以得到对应于虚拟内存地址的物理内存地址。

![image-20220201145001356](D:\linux\v2free\6.S081\01_note\0x04 Page tables.assets\image-20220201145001356.png)

PTE中有10bits是做flag的用于用来控制地址权限：

- 第一个标志位是Valid。如果Valid bit位为1，那么表明这是一条合法的PTE，你可以用它来做地址翻译。对于刚刚举得那个小例子（应用程序只用了1个page的例子），我们只使用了3个page directory，每个page directory中只有第0个PTE被使用了，所以只有第0个PTE的Valid bit位会被设置成1，其他的511个PTE的Valid bit为0。这个标志位告诉MMU，你不能使用这条PTE，因为这条PTE并不包含有用的信息。
- Readable和Writable 表明你是否可以读/写这个page。
- Executable表明你可以从这个page执行指令。
- User表明这个page可以被运行在用户空间的进程访问。
- 其他标志位并不是那么重要，他们偶尔会出现，前面5个是重要的标志位。

**Q**：PPN是如何合并成最终的物理内存地址？

**A**：在最高级的page directory中的PPN，包含了下一级page directory的物理内存地址，依次类推。在最低级page directory，我们还是可以得到44bit的PPN，这里包含了我们实际上想要翻译的物理page地址，然后再加上虚拟内存地址的12bit offset，就得到了56bit物理内存地址。

**Q**：我想知道我们是怎么计算page table的物理地址，是不是这样，我们从最高级的page table得到44bit的PPN，然后再加上虚拟地址中的12bit offset，就得到了完整的56bit page table物理地址？

**A**：我们不会加上虚拟地址中的offset，这里只是使用了12bit的0。所以我们用44bit的PPN，再加上12bit的0，这样就得到了下一级page directory的56bit物理地址。这里要求每个page directory都与物理page对齐（也就是page directory的起始地址就是某个page的起始地址，所以低12bit都为0）。

**Q**：3级的page table是由操作系统实现的还是由硬件自己实现的？

**A**：是由硬件实现的，所以3级 page table的查找都发生在硬件中。MMU是硬件的一部分而不是操作系统的一部分。在XV6中，有一个函数也实现了page table的查找，因为时不时的XV6也需要完成硬件的工作，所**以XV6有这个叫做walk的函数，它在软件中实现了MMU硬件相同的功能。**

## 4 页表缓存（Translation Lookside Buffer）

由page table的设计结构我们可以知道：当处理器从内存加载或者存储数据时，基本上都要做3次内存查找，第一次在最高级的page directory，第二次在中间级的page directory，最后一次在最低级的page directory。对于一个虚拟内存地址的寻址，需要读三次内存，这里代价有点高。所以实际中，几乎所有的处理器都会对于最近使用过的虚拟地址的翻译结果有缓存。这个缓存被称为：Translation Lookside Buffer，TLB（通常翻译成**页表缓存**）。

> 当处理器第一次查找一个虚拟地址时，硬件通过3级page table得到最终的PPN，TLB会保存虚拟地址到物理地址的映射关系。这样下一次当你访问同一个虚拟地址时，处理器可以查看TLB，TLB会直接返回物理地址，而不需要通过page table得到结果。

- RISC-V中清空TLB的指令时sfence_vma

**Q**：在这个机制中，TLB发生在哪一步，是在地址翻译之前还是之后？

**A**：整个CPU和MMU都在处理器芯片中，所以在一个RISC-V芯片中，有多个CPU核，MMU和TLB存在于每一个CPU核里面。RISC-V处理器有L1 cache，L2 Cache，有些cache是根据物理地址索引的，有些cache是根据虚拟地址索引的，由虚拟地址索引的cache位于MMU之前，由物理地址索引的cache位于MMU之后。

## 5 kernel page table

首先我们来看一下kernel page的分布。下图就是内核中地址的对应关系，左边是内核的虚拟地址空间，右边上半部分是物理内存或者说是DRAM，右边下半部分是I/O设备。接下来我会首先介绍右半部分，然后再介绍左半部分。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaY9xY8MaH5XTiwuBm%2Fimage.png?alt=media&token=3adbe628-da78-472f-8e7b-3d0b1d3177b5)

图中的右半部分的结构完全由硬件设计者决定。当操作系统启动时，会从地址0x80000000开始运行，这个地址其实也是由硬件设计者决定的。主板的设计人员决定了，在完成了虚拟到物理地址的翻译之后，如果得到的物理地址大于0x80000000会走向DRAM芯片，如果得到的物理地址低于0x80000000会走向不同的I/O设备。这是由这个主板的设计人员决定的物理结构。如果想要查看这里的物理结构，可以阅读主板的手册，手册中会一一介绍物理地址对应关系。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaa841bcyITdSeXgSv%2Fimage.png?alt=media&token=722903cf-faa3-4e99-8bd3-3198d3a7cf59)

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaaGcY8o95G9RhrbP9%2Fimage.png?alt=media&token=eb60fc6f-da2d-4a8d-9364-c17f5c0f9ac5)

地址0是保留的，地址0x10090000对应以太网，地址0x80000000对应DDR内存，处理器外的易失存储（Off-Chip Volatile Memory），也就是主板上的DRAM芯片。

回到最初那张图的右侧：物理地址的分布。可以看到最下面是未被使用的地址，这与主板文档内容是一致的（地址为0）。地址0x1000是boot ROM的物理地址，当你对主板上电，主板做的第一件事情就是运行存储在boot ROM中的代码，当boot完成之后，会跳转到地址0x80000000，操作系统需要确保那个地址有一些数据能够接着启动操作系统。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKaeaT3eXOyG4jfKKU7%2Fimage.png?alt=media&token=a04af08d-3c8d-4c61-a63d-6376dec252ea)

这里还有一些其他的I/O设备：

- PLIC是中断控制器（Platform-Level Interrupt Controller）
- CLINT（Core Local Interruptor）也是中断的一部分。所以多个设备都能产生中断，需要中断控制器来将这些中断路由到合适的处理函数。
- UART0（Universal Asynchronous Receiver/Transmitter）负责与Console和显示器交互。
- VIRTIO disk，与磁盘进行交互。

地址0x02000000对应CLINT，当你向这个地址执行读写指令，你是向实现了CLINT的芯片执行读写。这里你可以认为你直接在与设备交互，而不是读写物理内存。

当机器刚刚启动时，还没有可用的page，XV6操作系统会设置好内核使用的虚拟地址空间，也就是这张图左边的地址分布。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKalBg8KcCxk0HNNgOr%2Fimage.png?alt=media&token=17dfd137-31cc-4262-8f7b-d00df4d3b9f1)

为了使xv6保持精简，这里的虚拟地址到物理地址的映射，大部分是相等的关系。比如说内核会按照这种方式设置page table，虚拟地址0x02000000对应物理地址0x02000000。这意味着左侧低于PHYSTOP的虚拟地址，与右侧使用的物理地址是一样的。

除此之外，这里还有两件重要的事情：

- 有一些page在虚拟内存中的地址很靠后，比如kernel stack在虚拟内存中的地址就很靠后。这是因为在它之下有一个未被映射的Guard page，这个Guard page对应的PTE的Valid 标志位没有设置，这样，如果kernel stack耗尽了，它会溢出到Guard page，但是因为Guard page的PTE中Valid标志位未设置，会导致立即触发page fault，这样的结果好过内存越界之后造成的数据混乱。立即触发一个panic（也就是page fault），你就知道kernel stack出错了。同时我们也又**不想浪费物理内存给Guard page，所以Guard page不会映射到任何物理内存，它只是占据了虚拟地址空间的一段靠后的地址。**立即触发一个panic（也就是page fault），你就知道kernel stack出错了。同时我们也又不想浪费物理内存给Guard page，所以Guard page不会映射到任何物理内存，它只是占据了虚拟地址空间的一段靠后的地址。

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MK_UbCc81Y4Idzn55t8%2F-MKbZEkzbzbKYgRRedXU%2Fimage.png?alt=media&token=2167acef-e76c-4d0c-81b0-f5175475793f)

> 这是众多你可以通过page table实现的有意思的事情之一。你可以向同一个物理地址映射两个虚拟地址，你可以不将一个虚拟地址映射	到物理地址。可以是一对一的映射，一对多映射，多对一映射。XV6至少在1-2个地方用到类似的技巧。这的kernel stack和Guard     page就是XV6基于page table使用的有趣技巧的一个例子

- 权限。例如Kernel text page被标位R-X，意味着你可以读它，也可以在这个地址段执行指令，但是你不能向Kernel text写数据。通过设置权限我们可以尽早的发现Bug从而避免Bug。对于Kernel data需要能被写入，所以它的标志位是RW-，但是你不能在这个地址段运行指令，所以它的X标志位未被设置。（注，所以，kernel text用来存代码，代码可以读，可以运行，但是不能篡改，kernel data用来存数据，数据可以读写，但是不能通过数据伪装代码在kernel中运行）

用户进程的虚拟地址空间分布：

![img](https://906337931-files.gitbook.io/~/files/v0/b/gitbook-28427.appspot.com/o/assets%2F-MHZoT2b_bcLghjAOPsJ%2F-MKlssQnZeSx7lgksqSn%2F-MKopGK-JjubGvX84-qy%2Fimage.png?alt=media&token=0084006f-eedf-44ac-b93e-a12c936e0cc0)

## 6 kvminit 函数



## 7 kvminithart 函数



## 8 walk 函数



