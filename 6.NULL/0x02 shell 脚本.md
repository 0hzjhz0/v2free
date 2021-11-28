# 0x02 shell 工具和脚本

> 重要的是你要知道有些问题使用合适的工具就会迎刃而解，而具体选择哪个工具则不是那么重要

## Shell 脚本

terminal、shell、bash的区别？

- terminal 是终端，即命令窗口
- shell 是命令行解释器，用于解释命令，根据命令执行相应的程序等
- bash 是shell 的一种，还有 zsh, fish 等等

赋值语法`foo=bar`

注意不能使用空格分割，bash 会以空格为单位进行解释。如果以空格分隔 `foo = bar`，bash 会调用 `foo` 程序将 `=` 和 `bar` 作为参数

访问变量`$foo`

单引号和双引号不同，单引号内部的内容会原样输出，双引号则会进行相应的替换，例如

```shell
hz@PC:/mnt/d/linux/v2free$ foo=bar
hz@PC:/mnt/d/linux/v2free$ echo "$foo"
bar
hz@PC:/mnt/d/linux/v2free$ echo '$foo'
$foo
```

> 和其他多数编程语言一样，bash 也支持 `if`, `case`, `while` 和 `for` 这些流程控制语句

bash 支持函数

```shell
mcd() {
	mkdir -p "$1"
	cd "$1"
}
```

- `$0` - 脚本名
- `$1` 到 `$9` - 脚本的参数。 `$1` 是第一个参数，依此类推
- `$@` - 所有参数
- `$#` - 参数个数
- `$?` - 前一个命令的返回值
- `$$` - 当前脚本的进程识别码
- `!!` - 完整的上一条命令，包括参数。常见应用：当你因为权限不足执行命令失败时，可以使用 `sudo !!`再尝试一次
- `$_` - 上一条命令的最后一个参数。如果你正在使用的是交互式shell，你可以通过按下 `Esc` 之后键入 . 来获取这个值

`STDOUT `返回输出值， `STDERR` 返回错误及错误码。

返回值 0 表示正常执行，非零则不正常。根据返回值可以判断程序是否正常执行。

当通过`$(cmd)`的方式执行命令`cmd`时，它的输出结果会替换`$(cmd)`，例如：

```shell
hz@PC:/mnt/d/linux/v2free$ echo "Starting program at $(date)"
Starting program at Sun Nov 28 16:07:23 CST 2021
```

还有个不常用的类似用法是**进程替换**，`<(cmd)`会执行`cmd`并将结果输出到一个临时文件中，并将`<(cmd)`替换成临时文件名，这在我们希望返回值通过文件而不是stdin 传递时很有用。例如，`diff <(ls foo) <(ls bar)` 会显示文件夹`foo` 和 `bar` 中文件的区别。

下面这个例子展示了一部分上面提到的特性。这段脚本会遍历我们提供的参数，使用`grep` 搜索字符串 `foobar`，如果没有找到，则将其作为注释追加到文件中

```shell
#!/bin/bash

echo  "Starting program at $(date)" # date会被替换成日期和时间

echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
	grep foobar "$file" > /dev/null 2> /dev/null
	# 如果模式没有找到，则grep退出状态为 1
    # 我们将标准输出流和标准错误流重定向到Null，因为我们并不关心这些信息
     if [[ $? -ne 0 ]]; then
     	echo "File $file does not have any foobar, adding one"
     	echo "# foobar" >> "$file"
	fi
done
```

- 判断语句尽量使用双方括号 `[[]]`，降低犯错概率

- `$?` 是上一个命令的返回值，用于判断命令的执行情况，if 中的内容则是判断返回值是否等 0 
- `ne` 代表 no equal ，Bash实现了许多类似的比较操作，可以查看 [`test 手册`](https://man7.org/linux/man-pages/man1/test.1.html)

执行脚本的时候需要提供相应的参数，脚本会匹配相应的参数，这叫做 shell 的通配：

-  `?` 和 `*` 匹配一个或任意个字符。例如，对`foo`, `foo1`, `foo2`, `foo10` 和 `bar`，`rm foo?`会删除 `foo1` 和 `foo2`, 而 `rm foo*` 会删除除了 `bar` 之外的所有文件
- 花括号`{}` - 当你有一系列的指令，其中包含一段公共子串时，可以用花括号来自动展开这些命令。这在批量移动或转换文件时非常方便

```shell
convert image.{png,jpg}
# 会展开为
convert image.png image.jpg

cp /path/to/project/{foo,bar,baz}.sh /newpath
# 会展开为
cp /path/to/project/foo.sh /path/to/project/bar.sh /path/to/project/baz.sh /newpath

# 也可以结合通配使用
mv *{.py,.sh} folder
# 会移动所有 *.py 和 *.sh 文件

mkdir foo bar

# 下面命令会创建foo/a, foo/b, ... foo/h, bar/a, bar/b, ... bar/h这些文件
touch {foo,bar}/{a..h}
touch foo/x bar/y
# 比较文件夹 foo 和 bar 中包含文件的不同
diff <(ls foo) <(ls bar)
# 输出
# < x
# ---
# > y
```

脚本不一定非得用 shell 写，也可以采用 python 写脚本。 但是需要在文件的头部写上 [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) `#!/usr/local/bin/python` 用于告诉 shell 通过 python 来执行后续内容。



## Shell 工具

### 查看命令如何使用

最常用的方法是为对应的命令行添加`-h` 或 `--help` 标记。另外一个更详细的方法则是使用`man` 命令。[`man`](https://man7.org/linux/man-pages/man1/man.1.html) 命令是手册（manual）的缩写，它提供了命令的用户手册。

有时候手册内容太过详实，让人难以在其中查找哪些最常用的标记和语法。 [TLDR pages](https://tldr.sh/) 是一个很不错的替代品，它提供了一些案例，可以帮助你快速找到正确的选项。

### 查找文件

类 Unix 系统一般都包含命令 find 用于文件查找

```shell
# 查找所有名称为src的文件夹
find . -name src -type d
# 查找所有文件夹路径中包含test的python文件
find . -path '**/test/**/*.py' -type f
# 查找前一天修改的所有文件
find . -mtime -1
# 查找所有大小在500k至10M的tar.gz文件
find . -size +500k -size -10M -name '*.tar.gz'
```

对文件进行操作：

```shell
# Delete all files with .tmp extension
find . -name '*.tmp' -exec rm {} \;
# Find all PNG files and convert them to JPG
find . -name '*.png' -exec convert {} {}.jpg \;
```

### 查找代码

查找文件是很有用的技能，但是很多时候你的目标其实是查看文件的内容。一个最常见的场景是你希望查找具有某种模式的全部文件，并找它们的位置。

为了实现这一点，很多类UNIX的系统都提供了[`grep`](https://man7.org/linux/man-pages/man1/grep.1.html)命令，它是用于对输入文本进行匹配的通用工具。

`grep` 有很多选项，这也使它成为一个非常全能的工具。其中，常用的有：

- `-C`：获取查找结果的上下文（Context），`grep -C 5` 会输出匹配结果的前后五行
- `-v`：对结果进行反选（invert），输出不匹配的结果
- `-R`：递归地进入字目录并搜索所有的文本文件

当前，社区也出现了很多`grep`的替代品，如`ag`, `rg` 和 `ack`。以`rg`为例，它速度快，而且用法非常符合直觉。例子如下：

```shell
# 查找所有使用了 requests 库的文件
rg -t py 'import requests'
# 查找所有没有写 shebang 的文件（包含隐藏文件）
rg -u --files-without-match "^#!"
# 查找所有的foo字符串，并打印其之后的5行
rg foo -A 5
# 打印匹配的统计信息（匹配的行和文件的数量）
rg --stats PATTERN
```

### 查找 shell 命令

随着你使用shell的时间越来越久，您可能想要找到之前输入过的某条命令。首先，按向上的方向键会显示你使用过的上一条命令，继续按上键则会遍历整个历史记录。

`history` 命令会在标准输出中打印shell中的里面命令，可以通过`|`和`grep`搜索特定命令， `history | grep find` 会打印包含find子串的命令。

对于大多数的shell来说，可以使用 `Ctrl+R` 对命令历史记录进行回溯搜索。敲 `Ctrl+R` 后您可以输入子串来进行匹配，查找历史命令行。

### 文件夹导航

可以使用[`fasd`](https://github.com/clvv/fasd)和[autojump](https://github.com/wting/autojump)这两个工具来查找最常用或最近使用的文件和目录

Fasd 基于 [*frecency*](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/Places/Frecency_algorithm)对文件和文件排序，也就是说它会同时针对频率（*frequency* ）和时效（ *recency*）进行排序。默认情况下，`fasd`使用命令 `z` 帮助我们快速切换到最常访问的目录。例如， 如果您经常访问`/home/user/files/cool_project` 目录，那么可以直接使用 `z cool` 跳转到该目录。对于 autojump，则使用`j cool`代替即可。

还有一些更复杂的工具可以用来概览目录结构，例如 [`tree`](https://linux.die.net/man/1/tree), [`broot`](https://github.com/Canop/broot) 或更加完整的文件管理器，例如 [`nnn`](https://github.com/jarun/nnn) 或 [`ranger`](https://github.com/ranger/ranger)。



## 补充

[`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html) 命令，它可以使用标准输入中的内容作为参数。 例如 `ls | xargs rm` 会删除当前目录中的所有文件



## 参考

- [Lecture 2: Shell Tools and Scripting (2020) - YouTube](https://www.youtube.com/watch?v=kgII-YWo3Zw)

