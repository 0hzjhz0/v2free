# 0x01 Intro & Examples

## 准备工作

阅读【1】中的chap.1

【1】https://pdos.csail.mit.edu/6.828/2020/xv6/book-riscv-rev1.pdf

## 课程目标

- 理解O/S的『设计』和『实现』
- 通过 xv6 获得实际动手经验：通过研究现有操作系统，并结合课程配套的实验，获得扩展O/S，修改并提升O/S的相关经验，并能够通过操作系统接口，编写系统软件

## 操作系统简介

### O/S 的目标

1. `抽象硬件`
2. `复用硬件资源`
3. `提供隔离性`
4. `数据共享`
5. `Security/Permission System/Access Control System`
6. `高性能`
7. `多任务`：支持不同类型的任务，如文本编辑器、游戏、数据库...

### O/S 结构

在 CS 领域，分层思想无处不在，经典的 O/S 组织结构也是分层的，如下图。

`分层的O/S结构`

- 这个结构的最底层是一些硬件资源，如 CPU、内存、磁盘、网卡

- 在这个框架的最上层，运行着各种各样的应用程序，如文本编辑器 `VIM`、C 编译器 `CC` 等等，这些程序都运行在『用户态』`userspace`

- 区别于用户态程序，有一个特殊的程序总是运行 - `kernel`

  > `kernel` 是计算机资源的守护者，当打开计算机时，kernel 总是第一个被启动。
  >
  > kernel 程序只有一个，负责管理每一个用户进程；
  >
  > kernel 维护了大量的数据结构来帮助它管理各种各样的硬件资源，以供用户空间的程序使用；
  >
  > kernel 内置了大量的服务，例如，kernel 通常回有文件系统实现类似文件名，文件内容，目录的东西，并理解如何将文件存储在磁盘中。因此用户态的程序会与kernel的文件系统交互，文件系统再与磁盘交互。

这门课主要关注 kernel、连接kernel和用户态程序的接口、kernel的架构。kernel中的服务：『文件系统』，『进程管理系统』。每个用户态程序都被称为一个进程，它有自己的内存和共享的 CPU 时间。同时 kernel 会『管理内存分配』...

在一个真实完备的O/S中，会有许多其他服务：不同进程之间通信的进程间通信『IPC』、一票和网络关联的软件『TCP/IP协议栈』、支持声卡的软件、支持各种磁盘、网卡的驱动...

`kernel API` 决定了用户态程序如何访问 kernel，也即系统调用。



## 系统调用

### `read` `write` `exit`

```c
// copy.c: copy input to output
#include "kernel/types.h"
#include "user/user.h"

int
main()
{
    char buf[64];
    
    while(1) {
        int n = read(0, buf, sizeof(buf));
        if (n <= 0)
            break;
        write(1, buf, n);
    }
    
    exit(0);
}
```

这个程序包含三个系统调用：`read`、`write`、`exit`

`read` 接收三个参数：

- 第一个参数是文件描述符，默认情况下文件描述符 `0 - stdin` 连接 console 的输入，文件描述符 `1-stdout` 连接 console 的输出
- 第二个参数是一个指向某段内存的指针`缓冲`，保存 `read` 读取的数据
- 第三个参数是读取的最大长度

`read` 的返回值可以是读取的长度，也可能是异常`负数`



### `open`

`copy.c`  的文件描述符是已经设置好的，通常，我们需要手动创建文件描述符，最直接的创建文件描述符的方式是 **`open` 系统调用**

```c
// open.c: create a file, write to it.
#include "kernel/types.h"
#include "user/user.h"
#include "kernel/fcntl.h"

int
main()
{
    int fd = open("output.txt", O_WRONLY | O_CREATE);
    write(fd, "ooo\n", 4);
    
    exit(0);
}
```

`open.c` 会创建一个叫 `output.txt` 的文件并向其中写入一些数据。`open` 系统调用的第二个参数是一些标志位：这里是创建并写入一个文件。`open` 系统调用会返回一个新分配的文件描述符。

之后，这个文件描述符作为第一个参数被传到 `write`，`write` 的第二个参数是数据的指针，第三个参数是要写入的。数据被写入到了文件描述符对应的文件中。

文件描述符的本质对应内核中的一个表单数据。内核维护了每个运行进程的状态，内核会为每个运行进程保存一个表单，表单的 key 是文件描述符，这个表单让内核知道，每个文件描述符对应的实际内容是什么。值得注意的是，**每个进程都有自己独立的文件描述符空间**。如果运行了两个不同的进程，它们都打开了一个文件，它们或许可以得到相同的文件描述符，但这些相同数字的文件描述符可能会对应不同的文件。

### shell

shell 通常是人们说的命令行接口，是一种对unix系统管理来说非常有用的接口，它提供了很多工具来管理文件，编写程序，编写脚本。

当输入 `ls`，其实是让 shell 去运行一个名为 `ls` 的程序。

`ls > out` 将 `ls` 的输出重定向至 `out` 文件；`grep x`  搜索输入中包含 `x`的行，入 `grep x < out`

**问题**

- 编译器如何处理系统调用？生成的汇编语言是不是会调用一些由操作系统定义的代码段？

  > 对于 RISC-V，有一个特殊的指令，程序可以调用这个指令，并将控制权交给内核。实际当运行 C 语言并执行例如 `open` 等系统调用时，从技术上说，`open` 是一个 C 函数，但这个函数内的指令实际上是机器指令。也就是说，我们调用的 `open` 函数并不是一个 C 语言函数，而是由汇编语言实现，组成这个系统调用的汇编语言实际上在 RISC-V 中被称为 `ecall`。这个特殊的指令将控制权转给内核，之后内核检查进程的内存和寄存器，并确定相应的参数。

### `fork`

`fork` 会创建一个新的进程。

```c
//fork.c: create a new process
#include "kernel/types.h"
#include "user/user.h"

int
main()
{
    int pid;
    
    pid = fork();
    
    printf("fork() return %d\n", pid);
    
    if (pid == 0) {
        printf("child\n");
    } else {
        printf("parent\n");
    }
    
    exit(0);
}
```

`fork` 会拷贝当前进程的内存，并创建一个新的进程，这里的内存包含了进程的指令和数据。`fork` 系统调用在两个进程中都会返回，在原始进程中，`fork` 系统调用会返回大于 `0` 的整数，这是新创建进程的 ID。在新创建的进程中，`fork` 系统调用会返回 `0`。因此，即使两个进程的内存完全一样，还是可以通过 `fork` 的返回值区分新旧新城。

当在 `shell` 中运行东西时，`shell` 实际上会创建一个新的进程来运行输入的指令。所以当输入`ls`时，我们需要`shell`通过`fork`创建一个进程来运行`ls`，这里需要某种方式让这个进程来运行`ls`程序中的指令，加载名为`ls`的指令(也就是后面提到的`exec`系统调用)

### `exec` `wait`

这个例子会使用 `echo` ，它接收输入，并将输入写道输出

```shell
$ echo a b c d
a b c d
```

```c
// exec.c: replace a process with an executable file
#include "kernel/types.h"
#include "user/user.h"

int
main()
{
    char *argv[] = {"echo", "this", "is", "echo", 0};
    exec("echo", argv);
    printf("exec failed!\n");
    exit(0);
}
```

`exec`系统调用会从指定的文件中读取并加载指令，并代替当前调用进程的指令。从某种程度上说，这样相当于丢弃了调用进程的内存，并开始执行新加载的指令。

上述代码相当于：操作系统从一个名为 `echo` 的文件中加载指令到当前进程，并替换了当前进程的内存，之后开始执行这些新加载的指令。同时，可以传入命令行参数，`exec` 允许传入一个命令行参数的数组。

所以这里等价运行 `echo` 命令，并带上 `this is echo` 三个参数。

关于 `exec` 系统调用，有一些重要的事情：

- `exec` 系统调用会保留当前的文件描述表单。所以任何在exec 系统调用之前的文件描述符，例如 `0`，`1`，`2`等，它们在新的程序中表示相同的东西。
- 通常，`exec` 不会返回，因为 `exec` 完全替换了当前进程的内存，相当于当前进程不存在了，所以`exec` 系统调用没有地方能返回了

- `exec` 系统调用只有在出错时才会返回，比如程序文件不存在

实际上在shell运行类似`echo a b c` 指令时，我们不期望 `shell` 进程被替换。实际上，`fork` 会被执行，之后 `fork` 出的子进程再调用`exec` 系统调用。

```c
// forkexec.c: fork then exec
int
main()
{
    int pid, status;
    
    pid = fork();
    if (pid == 0) {
        char *argv[] = {"echo", "this", "is", "echo", 0};
        exec("echo", argv);
        printf("exec failed!\n");
        exit(1);
    } else {
        printf("parent waiting\n");
        wait(&status);
        printf("the child exited with status %d\n", status);
    }
    exit(0);
}
```

如 `15` 行所示，unix 提供了系统调用 `wait` ，它会等待之前创建的子进程退出。当我们在命令行执行一个指令时，我们一般会系统shell等待指令执行完成。这里 `wait` 的参数是 `status`，是一种让退出的紫禁城以一个整数 `32bit` 的格式与等待的父进程进行通信的方式。

unix的风格是，如果一个程序成功退出，那么 `exit` 的参数会是 `0`，如果出现了错误，那么会向 `exit` 传递 `1`。

**注意**

> 这里的写法先fork子进程然后执行exec替换子进程内存的方式有些浪费。某种程度上说这拷贝操作浪费了。在大型程序中，这样的操作影响明显，如果你运行了一个几G的程序，并调用fork，那么实际上会拷贝所有内存，可能会消耗将近1s中来完成拷贝，这可能会是个问题。
>
> 后面会介绍一些优化，如`copy-on-write-fork`，这种方式会消除fork几乎所有的明显的抵消，而只拷贝执行exec所需要的内存，这里需要很多涉及虚拟内存的系统的技巧。
>
> 你可以构建一个fork，对于内存实行lazy-copy，通常来说fork之后立即是exec，这样你就不用进行实际的拷贝，因为子进程实际并没有使用大部分的内存

**问题**

- unix没有直接的方式让子进程等待父进程，wait系统调用只能等待进程的子进程。wait的工作原理：如果当前进程由任何子进程，并且其中一个已经退出，那么wait会马上返回。

### I/ORedirect

结合上述系统调用实现I/O重定向。

```c
// redirect.c: run a command with output redirected
int
main()
{
    int pid;
    
    pid = fork();
    if (pid == 0) {
        close(1);
        open("output.txt", O_WRONLY | O_CREATE);
        
        char *argv[] = {"echo", "this", "is", "redirected", "echo", 0};
        exec("echo", argv);
        printf("exec failed!\n");
        exit(1);
    } else {
        wait((int *) 0);
    }
    exit(0);
}
```

这里 `close(1)` 是希望文件描述符`1`指向其他位置。`10` 行一定会返回`1`，因为`open`会返回当前进程未使用的最小文件描述符。

