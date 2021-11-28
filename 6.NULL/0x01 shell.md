# 0x01 shell

当你打开终端时，它通常是这样子的：

```shell
hz@PC:~$
```

这是shell最主要的文本接口，它包含三点信息：

- `PC`表示你的主机名
- `hz`表示当前用户名

- `~` 表示当前位于 `home` 目录下
- `$` 表示当前的身份不是 `root` 用户

shell 是命令行解释器，用于解释命令。例如输入`date`会输出当前日期

```shell
hz@PC:~$ date
Sun Nov 28 13:57:20 CST 2021
```

shell 通过空格分割命令行以进行解析。例如 `echo` 命令，以空格分割

```shell
hz@PC:~$ echo hello
hello
```

如果空格作为命令的一部分，需要采用转移符号`\`进行转移。

如果执行的命令不是 shell 内置的，shell 会到环境变量`$PATH`中查找。

## 在 shell 中导航

在 Linux/macOS 中路径以斜杠 `/` 分割，根目录 `/` 代表系统所有文件都在该路径下。

在 windows 中路径以反斜杠 `\` 分割。分为多个盘，例如 (C/D/E/F)等，每个盘都有一个根目录，例如`C:\`。

- 绝对路径表示以根目录起始的路径
- 相对路径则是相对于当前工作目录的路径。绝对路径外的都是相对路径

 `pwd` 命令可以查看当前的工作目录

```shell
hz@PC:~$ pwd
/home/hz
```

`cd`(change directory) 命令用于切换目录：

- `..` 表示父目录

  ```shell
  hz@PC:/home$ cd ..
  hz@PC:/$
  ```

- `.` 表示当前工作目录

  ```shell
  hz@PC:/$ cd ./home
  hz@PC:/home$
  ```

- `~`表示当前用户的主(家)目录

  - 对于 `root` 用户，`cd ~` 会切换到`/root`目录下

    ```
    root@PC:/# pwd
    /
    root@PC:/# cd ~
    root@PC:~# pwd
    /root
    ```

  - 非 `root` 用户，`cd ~`会切换到当前用户的`home`目录下，即`/home/{username}`目录

    ```shell
    hz@PC:/$ pwd
    /
    hz@PC:/$ cd ~
    hz@PC:~$ pwd
    /home/hz
    ```

- `/`表示根目录

- `cd -` 撤销目录切换，比如从`A`目录切换到`B`，想回退到`A`目录，可以用这条命令

  ```shell
  hz@PC:~$ pwd
  /home/hz
  hz@PC:~$ cd /
  hz@PC:/$ cd -
  /home/hz
  ```

`ls` 命令用于查看当前目录下的文件

```shell
hz@PC:/home$ ls
hz
```

`ls` 命令有很多参数，如果忘记参数含义，可以采用 `ls --help` 进行查看

例如`ls -l` 可以打印更详细的文件信息

```shell
hz@PC:/home$ ls -l
total 4
drwxr-xr-x 4 hz hz 4096 Nov 27 23:49 hz
```

第三行中开头的`d`表示 `hz` 是一个目录，接下来9个字符每三个字符一组`d(rwx)(r-x)(r-x)`

这三组分别代表了**文件所有者**，**用户组**和**其他所有人**所拥有的权限

- `r`(read) - 可读
- `w`(write) - 可写
- `x`(execute) - 可执行
- `-` 表示该用户不具备相应的权限

> 注意，`/bin`目录下的可执行文件在最后一组，即其他所有人的用户组中，均包含`x`权限，也就是任何人都可以执行这些程序

现阶段，还需要掌握如下几个文件操作命令：

- `mv` 重命名或移动文件
- `cp` 拷贝文件
- `mkdir` 新建文件

## 在程序间创建连接

信息的流动成为**流**，程序中存在两个流：

- 输入程序的**输入流**
- 输出程序的**输出流**

程序读取信息时会从输入流中读入，打印信息时会输出到输出流中

> 最简单的重定向`< file`和`> file`会将程序的输入流和输出流分别重定向文件中

```shell
hz@PC:/mnt/d/linux$ echo hello > hello.txt
hz@PC:/mnt/d/linux$ cat hello.txt
hello
hz@PC:/mnt/d/linux$ cat < hello.txt > hello2.txt
hz@PC:/mnt/d/linux$ cat hello2.txt
hello
```

`>>` 表示追加内容

```shell
hz@PC:/mnt/d/linux$ cat hello.txt
hello
hz@PC:/mnt/d/linux$ echo hello >> hello.txt
hz@PC:/mnt/d/linux$ cat hello.txt
hello
hello
```

> 使用管道`pipe`，我们能更好的利用重定向。`|`操作符允许我们将一个程序的输出和另一个程序的输入连接起来

`ls -l /` 表示查看`/`下所有文件详细信息

```shell
hz@PC:/mnt/d/linux$ ls -l /
total 768
lrwxrwxrwx   1 root root       7 Apr 23  2020 bin -> usr/bin
drwxr-xr-x   2 root root    4096 Apr 23  2020 boot
drwxr-xr-x   8 root root    2800 Nov 28 13:49 dev
drwxr-xr-x  94 root root    4096 Nov 28 13:49 etc
drwxr-xr-x   3 root root    4096 Nov 24 21:23 home
-rwxr-xr-x   3 root root 1392928 Nov 23 21:58 init
lrwxrwxrwx   1 root root       7 Apr 23  2020 lib -> usr/lib
lrwxrwxrwx   1 root root       9 Apr 23  2020 lib32 -> usr/lib32
lrwxrwxrwx   1 root root       9 Apr 23  2020 lib64 -> usr/lib64
lrwxrwxrwx   1 root root      10 Apr 23  2020 libx32 -> usr/libx32
drwx------   2 root root   16384 Apr 11  2019 lost+found
drwxr-xr-x   2 root root    4096 Apr 23  2020 media
drwxr-xr-x   6 root root    4096 Nov 24 21:20 mnt
drwxr-xr-x   2 root root    4096 Apr 23  2020 opt
dr-xr-xr-x 227 root root       0 Nov 28 13:49 proc
drwx------   2 root root    4096 Nov 27 23:09 root
drwxr-xr-x   6 root root     120 Nov 28 13:49 run
lrwxrwxrwx   1 root root       8 Apr 23  2020 sbin -> usr/sbin
drwxr-xr-x   2 root root    4096 Apr 10  2020 snap
drwxr-xr-x   2 root root    4096 Apr 23  2020 srv
dr-xr-xr-x  11 root root       0 Nov 28 13:49 sys
drwxrwxrwt   2 root root    4096 Nov 28 13:49 tmp
drwxr-xr-x  14 root root    4096 Apr 23  2020 usr
drwxr-xr-x  13 root root    4096 Apr 23  2020 var
```

如果我们指向查看最后一个目录的相关信息，可以通过管道进行过滤

```shell
hz@PC:/mnt/d/linux$ ls -l / | tail -n1
drwxr-xr-x  13 root root    4096 Apr 23  2020 var
```

## su 命令——一个功能全面又强大的工具

根用户(`root`)在类 Unix 系统中时非常强大的，拥有整个系统的所有权限，可以通过 `su`(super user) 命令切换到根用户

根用户权限比较高， 通常不建议处于根用户的状态避免误操作。但是执行某些命令是权限不够，`sudo` 命令用于解决这个问题，也就是可以以 su 的身份来 do 一些事情。

## 补充

- shell 命名怎么用？

  1. 查看帮助信息 `-h`，`-help`

  2. `man [command]`
  3. [command-not-found.com ](https://command-not-found.com)
  4. [tldr pages](https://tldr.sh/)

  只是为了快速了解该命令怎么用推荐使用3和4

- 常用shell 命令请参考：
  -  https://www.geeksforgeeks.org/basic-shell-commands-in-linux/
  - https://swcarpentry.github.io/shell-novice/reference.html

## 参考

- [Lecture 1: Course Overview + The Shell (2020) - YouTube](https://www.youtube.com/watch?v=Z56Jmr9Z34Q)

- [IPADS新人培训第一讲：Shell_bilibili](https://www.bilibili.com/video/BV1y44y1v7c3?spm_id_from=333.999.0.0)













