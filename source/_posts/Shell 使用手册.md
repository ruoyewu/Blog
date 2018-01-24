---
title: Shell 使用手册
date: 2018-01-24 15:25
tags: 
	- linux
	- shell
	- 工具
---

## Shell

### 介绍

Shell 是一个 C 语言编写的程序，以方便用户对系统进行操作。它既是一种命令语言，又是一种程序设计语言。我们可以直接在系统的终端中输入一行一行的命令，每一个命令都会得到它的反馈，同时我们也可以编写一个脚本文件，在里面写上比较复杂的命令与操作，并且可以使用结构化的程序设计方式，完成一个比较复杂的功能。

Shell 不是 Linux 系统内核的一部分，但是它调用了系统内核的大部分功能，利用 Shell 程序设计语言可以编写出来功能强大且代码简单的 shell 脚本程序，还能将 linux 命令与 shell 相结合，在脚本程序里面不仅可以调用 Shell 本身提供的命令，还能执行其他所有可以直接在 linux 终端里面执行的命令，从而为我们提供了特别丰富的功能。

### 种类

常见的 Shell 包括以下几个：

|  名称  |           作者           | 命令数量 | 介绍                                       |
| :--: | :--------------------: | :--: | :--------------------------------------- |
| ash  |    Kenneth Almquist    |  24  | Linux 中占用系统资源最小的 Shell，使用起来并不方便          |
| bash | Brian Fox 和 Chet Ramey |  40  | Linux 系统默认 Shell ，BourneAgain Shell 的缩写，特色：<br />1. 可以使用 DOS 里的 doskey 功能，用上下方向键查阅和快速输入并修改命令<br />2. 自动通过查找匹配方式，给出以某字串开头的命令<br />3. 包含自身的帮助功能，能够通过键入 help 来获得相关帮助信息 |
| ksh  |       Eric Gisin       |  42  | Korn Shell 的缩写，最大的优点就是与商业发行版的 ksh 完全相容，无需购买商业版本就能尝试商业版本性能 |
| csh  |  William Joy 为首的47位作者  |  52  | Linux 比较大的 Shell                         |
| zsh  |      Paul Falstad      |  84  | Linux 最大的 Shell 之一，同时功能也很丰富，可以使用很多插件来很大程度地简化使用过程 |

可以通过`/etc/shells`文件中的内容查看：

```shell
cat /etc/shells
# List of acceptable shells for chpass(1).
# Ftpd will not allow users to connect who are not using
# one of these shells.

/bin/bash
/bin/csh
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
```

下面列出的这些就是当前电脑上有的 Shell 类型，然后还可以使用`echo $SHELL`命令查看当前使用的 Shell 类型：

```shell
echo $SHELL
/bin/zsh
```

因为我当前使用的是 zsh ，所以显示的结果为`/bin/zsh`，如果已经安装了某个 Shell 的话，可以直接键入 Shell 的名字来使用它：

```shell
MacBook-Pro:~ wuruoye$ echo $SHELL
/bin/bash
MacBook-Pro:~ wuruoye$ zsh
⬢  ~  exit
MacBook-Pro:~ wuruoye$
```

比如上面的一段示例，系统使用的是 bash ，然后我输入了`zsh`，就会进入 zsh 的 Shell 环境中，同时可以使用`exit`退出 zsh ，但是这种方式只是由一种 Shell 进入另一种 Shell 中去，如果想要改变系统默认的 Shell 类型，也就是打开终端的时候默认使用的 Shell 种类，可以使用下面的方式：

```shell
chsh -s /bin/zsh
```

这时就把系统默认的 Shell 换成了 zsh ，如果要改成其他的 Shell ，可以将上面的`/bin/zsh`换成`/etc/shells`文件中的任意一项。

### 命令形式

1.  内部命令

    内部命令一般置于 Shell 源码中，在打开 Shell 的时候，这些命令会随着程序一同加载到内存中，这些命令一般都是比较简单同时使用又比较多的命令，执行起来比较快。

2.  外部命令

    外部命令一般是存放在文件系统中，以文件的形式存在，这些命令一般功能比强大，逻辑也比较复杂，使用比较少，所以为了节省资源不放在内存。

可以使用`type`命令查看命令的类型：

```shell
type cd
cd is a shell builtin	# 内部命令
type cp
cp is /bin/cp	# 外部命令
```

#### 编写 Shell 脚本

最简单的一个脚本：

```shell
#!/bin/bash
echo 'hello world'
```

这是一个很简单的 Shell 脚本，用来在显示器上打印一行`hello world`，其中第一行的`#!/bin/bash`是一个约定行标记，告诉系统我们需要哪个 Shell 来执行这个脚本文件，如果在系统中找不到这个 Shell ，就会报错。如下面的例子，有四个脚本文件：

```shell
# test_bash.sh
#!/bin/bash
echo "$-"

# test_zsh.sh
#!/bin/zsh
echo "$-"

# test_ersh.sh
#!/bin/ersh
echo "$-"

# test_nosh.sh
echo "$-"
```

这四个脚本文件的功能一样，都是会打印出来执行当前命令的 Shell 的选项标志，然后在终端中执行这个脚本：

```shell
# 首先在 zsh 环境中执行以下命令
⬢  shell  echo $-
569JNRXZghikls
⬢  shell  ./test_bash.sh
hB
⬢  shell  ./test_zsh.sh
569X
⬢  shell  ./test_ersh.sh
zsh: ./test_ersh.sh: bad interpreter: /bin/ersh: no such file or directory
⬢  shell  ./test_nosh.sh
hB

# 然后在 bash 环境中执行命令
bash-3.2$ echo $-
himBH
bash-3.2$ ./test_bash.sh
hB
bash-3.2$ ./test_zsh.sh
569X
bash-3.2$ ./test_ersho.sh
bash: ./test_ersho.sh: No such file or directory
bash-3.2$ ./test_nosh.sh
hB
```

有上面几个例子可以看出，当我们在脚本文件的第一行以`#!/bin/${shell}`的方式指定了需要执行当前脚本的 Shell 之后，执行文件命令的时候会首先加载这个 Shell 然后用来执行脚本中的命令，如果加载不到就会报错。同时如果在脚本文件中不指定 Shell 的类型，系统就会使用系统默认的 Shell 来执行，而与当前正在使用的 Shell 类型无关，在 MacOS 上，bash 是系统默认的 Shell 。

#### 更改 Shell 类型

一般情况下，MacOS 与 Linux 都是使用 bash 作为默认的 Shell 类型，但是如果我们想要功能更加丰富的Shell 如 zsh 的话，不仅要安装好 zsh ，还要更改系统的 Shell 类型才能使用，而以直接输入`zsh`的方式虽然能够开启一个 zsh Shell 供我们使用，但是当我们重新打开终端的时候，默认加载的还是 bash ，如何改变系统默认的 Shell ，需要使用的是`chsh`命令，顾名思义，就是`change shell`的意思：

```shell
chsh -s /bin/zsh
```

执行这条命令就可以改变系统默认加载的 Shell 了，一般情况下执行这条命令的时候都需要输入当前用户的密码。然后就是，刚执行完这条命令的时候并不会立刻更改当前的 Shell ，需要使用`exit`命令退出后重新打开就可以了。

另外，并不是新建一个脚本文件就可以直接运行的，比如：

```shell
# 首先新建一个文件 test_chmnod
#!/bin/bash
echo "$SHELL"

# 然后在在终端中输入
⬢  shell  ./test_chmod
zsh: permission denied: ./test_chmod

# 首先给 test_chmod 赋予执行权限
chmod +x test_chmod
⬢  shell  test_chmod
zsh: command not found: test_chmod
⬢  shell  ./test_chmod
/bin/zsh

# 把当前文件夹加入环境变量
⬢  shell  PATH=/Users/wuruoye/Documents/shell:$PATH
⬢  shell  test_chmod
/bin/zsh
```

可以看到，直接创建之后输入会报错`permission denied`，所以还需要使用`chmod +x ${文件名}`来给对应的文件赋予执行权限，同时也不能直接使用`test_chmod`，因为不加上`./`，系统会从系统环境变量里寻找，也就是说，只有输入的命令所处的文件夹在系统的环境配置文件中，系统才能根据这个路径找到对应的脚本文件来执行，而我们没有把当前的文件夹加入到环境变量中去，所以系统自然找不到这个命令。而加上`./`则是显示地指定了当前脚本的文件夹，就可以执行了。如果这个命令用到的比较频繁，可以使用`PATH=${文件夹路径}:$PATH`的方式将当前文件夹加入环境变量，这时，在任何地方就都可以直接使用`test_chmod`来执行这个脚本文件了。但是使用这种方法并不能持久化，在本次登录退出以后再次登录就会发现，依然会报错`command not found`，如果想要永久化的使一个文件夹加入到环境变量中去，需要手动编辑文件`.bash_profile`，在里面加入一行`export PATH=${文件夹}:$PATH`，这样每次 Shell 初始化的时候就会加载这个文件里面的内容。

### Shell 初始化

Shell 在启动的时候，会加载一系列的配置文件，这些配置文件包括系统必须的一些资源，以及用户自定义的一些数据，比如用户可以在配置文件中添加一些自定义的环境变量，方便使用 Shell 时的操作。不过由于每次 Shell 启动的时候都需要加载配置文件，所以应该尽可能保证配置文件中所做的操作比较简单，使 Shell 打开的时候不至于太过缓慢。当然，大部分的一些 Shell 配置都是比较简单的，不会对使用造成什么影响，不过像使用 zsh 的时候，因为 zsh 提供了很多各种功能的插件，而这些插件也都是在配置文件中加入的，每次 Shell 初始化的时候都要从磁盘中加载到内存中，所以如果使用的插件比较多，就能够感觉到明显的卡顿了。

Shell 的初始化一般就是指根据一些配置文件执行一系列命令或者加载一些数据，这些配置文件又分为系统级的与用户级。

#### 系统级配置文件

系统级的 Shell 配置文件一般放在`/etc`目录下面，如`profiles`, `bashrc`, `bash.bashrc`等。d

#### 用户级配置文件

用户级 Shell 配置文件一般放在用户的根目录下，如`.profiles`, `.bash_profile`, `.bashrc`, `.bash_login`等文件，并且用户级的配置能够覆盖系统级配置。

### shell 类型

#### 交互式与非交互式

**交互式：Shell 等待用户输入命令，并且用户每输入一条命令就会立即予以响应。**

**非交互式：Shell读取放在文件中的命令然后执行，直到出错或者执行完毕才会推出执行状态。**

#### 登录式与非登录式

**登录式**

当一个用户成功登录一个系统的时候会调用该 Shell ，当启动一个 Shell 的时候，会读取一些配置文件，进行 Shell 的初始化。如这种情况：

```shell
Last login: Wed Jan 24 15:12:53 on ttys001
⬢  ~
```

第一行就是显示了此次登录系统的信息。

**非登录式**

当用户从终端中执行`bash`或`zsh`命令手动切换了 Shell 的时候，就会调用一个非登录式 Shell ，或者在打开`xterm`等图形化终端的时候，也会调用一个非登录式 Shell 。如：

```shell
⬢  ~  bash
bash-3.2$ zsh
⬢  ~  exit
bash-3.2$ exit
exit
⬢  ~
```

用户本身处于一个 zsh  Shell 中，然后执行 bash 命令之后，就会开启一个 bash Shell ，进入 bash 的环境，然后再输入 zsh ，就会再在 bash 中开启一个 zsh Shell ，这时进入的这个 bash 或者 zsh 都是非登录式的，当调用`exit`命令的时候，就会退出当前 Shell 返回到上一层，然后再调用`exit`的时候，就会返回到最初的那个 zsh Shell 中去。

## 参考

[http://blog.csdn.net/u011026329/article/details/50952897](http://blog.csdn.net/u011026329/article/details/50952897)

[http://www.runoob.com/linux/linux-shell.html](http://www.runoob.com/linux/linux-shell.html)

[https://dotbbq.com/logs/Understanding-Linux-Shell-Initialization-Files-and-User-Profiles.html](https://dotbbq.com/logs/Understanding-Linux-Shell-Initialization-Files-and-User-Profiles.html)