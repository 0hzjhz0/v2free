# Lab 0: Preparation

## 环境搭建

参考官方配置教程：[tools](https://pdos.csail.mit.edu/6.828/2020/tools.html)

环境： wsl2+Ubuntu20.04

```
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu 

# fix qemu
sudo apt-get remove qemu-system-misc
sudo apt-get install qemu-system-misc=1:4.2-3ubuntu6
```

## Testing your Installation

这里如果不是手动build toolchain的话，可能没有 `riscv64-unknown-elf-gcc`而是其他版本，具体可以去`/usr/bin`或`/usr/local/bin`中查看，比如我的是`riscv64-linux-gnu-gcc`

```
$ riscv64-linux-gnu-gcc --version
riscv64-linux-gnu-gcc (Ubuntu 9.4.0-1ubuntu1~20.04) 9.4.0
Copyright (C) 2019 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

$ qemu-system-riscv64 --version
QEMU emulator version 4.2.0 (Debian 1:4.2-3ubuntu6)
Copyright (c) 2003-2019 Fabrice Bellard and the QEMU Project developers
```

接下来能够启动 `QEMU`，并在 `QEMU` 里面运行 `xv6`，环境就算准备完了

```sh
$ git clone git://g.csail.mit.edu/xv6-labs-2020
Cloning into 'xv6-labs-2020'...
...
$ cd xv6-labs-2020
$ git checkout util
Branch 'util' set up to track remote branch 'util' from 'origin'.
Switched to a new branch 'util'

$ make qemu
···
xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2059
xargstest.sh   2 3 93
cat            2 4 24256
echo           2 5 23080
(...other files)
```

退出 `QEMU` 快捷键是 `Ctrl-a + x` 【先按ctrl-a 然后x】