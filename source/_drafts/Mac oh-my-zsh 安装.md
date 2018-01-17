---
title: Mac oh-my-zsh 使用
date: 2018-01-11 00:00
tags: 
	- linux
	- mac
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

## 安装 zsh

如果系统是 ubuntu ，可以在终端中输入下面代码安装 zsh ，如果是 mac ，系统一般会自带，可以使用上面的`/etc/shells`查看。

```bash
apt-get install zsh
apt-get install git-core
```

