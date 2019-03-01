---
title: Android 源码下载、编译、查看流程记录
date: 2018-01-13 01:35
tags:
	- mac
	- android
	- android 源码
	- 工具
---

## 概述

作为一名 android 开发工作者，想要在 android 开发的道路上走得更远，阅读源码肯定是一种必要。然而在日常写代码的时候我们使用的是 android SDK 提供的一系列的接口，使用 SDK 的过程中总会不可避免地遇到各种各样的问题，这时候阅读源码则能够帮助我们更好地解决遇到的问题，对于一些比较常用的类，如`View.java`等，我们可以通过查找直接看到它的源代码，但是对于大多数的类来说，我们还是看不到它们更多的代码，更多时候我们会看到的是这样的：

## 环境搭建

对于 Mac 来说，编译 android 源代码需要以下一些条件。

### 大小写敏感

Mac 与 Window 的文件系统是对大小写不敏感的，而 Linux 是敏感的，由于 Android 基于 Linux ，所以 android 的系统也是大小写敏感的，所以为了在 Mac 上构造一个适应 android 编译的环境，需要先创建一个大小写敏感的磁盘映像，然后在这个磁盘里面放入 android 源码等文件。

Android 的源码一般为几十个 G ，编译之后整体会上涨到一百多个 G ，所以为了能够应对各种环境，最好能设置 150G 以上。创建大小写敏感的磁盘映像可以通过两种方式，磁盘工具和终端。

#### 磁盘工具

使用 `command + space` 直接输入“磁盘工具”，回车就会进入磁盘工具界面，点击最上面的菜单栏，文件 -> 新建映像 -> 磁盘映像，或者直接键入 `command + N`新建一个映像，然后输入映像名称、大小、格式等，格式需要设置为 `Mac OS 扩展（区分大小写，日志式）`，点击存储等待创建完成即可。

#### 终端

创建磁盘空间：

```shell
hdiutil create -type SPARSE -fs 'Case-sensitive Journaled HFS+' -size 100g ~/android.dmg
# 显示以下输出表示成功
created: /Users/wuruoye/android.dmg.sparseimage
```

上面的代码会在当前目录创建一个 100g 大小的名为 android.dmg.sparseimage 的磁盘映像。同时也可以调整一个磁盘的大小，

```shell
hdiutil resize -size 200g ~/android.dmg.sparseimage
```

这时磁盘的大小就会变为 200g 。

生成了磁盘之后，可以在 finder 中直接找到对应的磁盘文件，双击就可以挂载到计算机上。可以看到在`/Volumes/`目录下有一个名为 android 的磁盘。

### jdk & XCode

编译不同的 Android 版本的时候，需要的 jdk 和 Xcode 版本也都是不一样的，对于 jdk 来说，可以到[这里](https://source.android.com/source/requirements)具体查看所需要的版本。版本确定之后，再根据里面给出的链接下载不同版本的 jdk 就可以了，对于 Xcode ，一般下载到的都是最新的版本，一般情况下这样是可以的，不过有一点需要做的，首先要到`/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/`这个文件夹下查看当前的`MacOSX.sdk`的版本，比如说我的目前来说是这样的：

![](http://blog-1251826226.coscd.myqcloud.com/Snip20180112_17.png)

在编译 android 源码的时候不同的版本用到的这个 sdk 的版本是不同的，可能会引起编译时候的报错，所以为了安全起见，可以到[这里](https://github.com/phracker/MacOSX-SDKs/releases)下载不同版本的 sdk 然后解压到这里。

### MacPort

编译 android 源码的时候还需要其他的一些工具，如 git 、gmake 、libsdl 、git 、gnupg 等。

可以点击[这里](https://www.macports.org/install.php)下载 MacPort ，下载完成之后，执行下面命令：

```shell
POSIXLY_CORRECT=1 sudo port install gmake libsdl git gnupg
```

等待一段时间下载完成就可以了。

### 其他

由于 Mac 下系统默认只能同时打开1024个文件，但是在 android 源码编译的过程中可能会超出这个上限，所以我们要现在就出这个限制：

```shell
# set the number of open files to be 1024
ulimit -S -n 1024
```

将上面段内容添加到`~/.bash_profile`中就可以了。

另外，如果当前系统使用的 Shell 不是 bash ，如正在使用 zsh ，那么在编译之前需要将系统默认 Shell 设置为 bash ，因为 android 源码编译的时候需要在 bash 下执行，可以使用以下代码切换默认 shell ：

```shell
chsh -s /bin/bash
```

至此，在 Mac 上编译 Android 源码的环境已经基本配置完成，下面进入下载源码阶段。

## 下载

### 下载工具

源码下载需要用到工具`repo`，可以输入下面命令下载：

```shell
curl https://mirrors.tuna.tsinghua.edu.cn/git/git-repo -o repo
chmod +x repo
```

第一句话的意思是下载 repo 到当前文件夹，但它本身还只是一个文件，第二句话为 repo 提供了直接运行的权限，是我们可以直接输入`repo`来运行这段脚本程序。repo 本身只是一个工具，下载 android 源代码也不是 repo 执行下载操作，repo 只是 git 的一个封装工具，它将 git 与 android 代码联系起来，是我们可以仅输入一行命令就完成整个的 android 代码下载到本地的过程，这个过程单纯使用 git 命令也是能够完成的，只不过需要的命令数量比较多而已。

### 下载源码

由于墙的原因，我们无法直接从 google 官方提供的链接中下载 android 的源码，所以只能使用国内的镜像下载，国内的镜像中[清华镜像](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)和[科大镜像](https://lug.ustc.edu.cn/wiki/mirrors/help/aosp)都是挺好的，可以直接进去根据他们的教程下载。

android 源码的下载有两种方式，一种是直接下载一个初始化包，然后使用 repo 直接从这个包里 checkout 出来，另一种则是传统的方法，在执行 repo 命令的时候下载。

#### 使用初始化包

首先需要下载一个本地包，清华镜像的链接为[https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar](https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar)，我下载的时候包的大小为46g，之后这个包大小肯定还会继续增加。下载完成之后解压，得到的是一个`.repo`文件夹，然后使用`repo sync`执行一遍，就可以得到一遍完整的目录了。命令如下：

```shell
# 下载初始化包 aosp-latest.tar
wget -c https://mirrors.tuna.tsinghua.edu.cn/aosp-monthly/aosp-latest.tar
# 解压初始化包，得到 aosp 文件夹
tar xf aosp-latest.tar
# 进入 aosp 文件夹，里面只有一个 .repo 文件夹，执行 repo sync
repo sync
```

这时就得到了当前的所有的源代码，之后也可以使用`repo sync`保持同步。

#### 传统方法

首先进入你需要下载源代码的目录里面去，然后执行下面的代码初始化仓库：

```shell
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest
```

使用时可能会报错说无法连接到 gerrit.googlesource.com ，这时需要将下面的一段内容复制到`~/.bash_profile`文件里面去，

```shell
export REPO_URL='https://mirrors.tuna.tsinghua.edu.cn/git/git-repo/'
```

使用上面的`repo init`命令的时候没有指定某一个版本，默认会加载最新的一个版本进来，如果要选择某个特定的 Android 版本，可以使用下面这种方法：

```shell
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b android-4.0.1.r1
```

-b 后面跟的是 Android 的版本号，可以点击[这里](https://source.android.com/source/build-numbers.html#source-code-tags-and-builds)查看不同的 Android 版本对应的版本号。

上面的命令执行完成之后，同样使用`repo sync`来同步源码树。

执行`repo sync`的过程中可能会出现各种各样的错误导致同步中断，这时只需要再次执行`repo sync`命令即可，同时可以编写一个脚本来做这种事情，脚本内容如下：

```shell
#!/bin/bash

repo sync
while [ $? = 1 ]; do
echo "=======sync failed, try again=========="
sleep 3
repo sync
done
```

将其保存为`doRepo.sh`文件，然后在终端中输入命令：

```shell
# 赋给 doRespo.sh 可执行的权限
chmod +x doRepo.sh
# 执行 doRepo.sh 中的命令
./doRepo.sh
```

这时脚本就会自动执行`repo sync`命令，并在出错中断的时候自动重新执行，直到同步完成就可以了。

完成之后的目录如下：

![](http://blog-1251826226.coscd.myqcloud.com/Snip20180112_18.png)

## 编译

源代码下载完成之后，就可以对源码进行编译处理了，相关的命令有以下几个：

```shell
make clobber
```

这个命令的作用是清空当前的编译缓存文件，可以在编译失败之后执行此文件清空一下再重新开始新的一次编译。

```shell
source build/envsetup.sh
```

这个命令用来引入`build/envsetup.sh`脚本，初始化编译环境，并将一些辅助的 Shell 函数引进来，比如说下面的 lunch 函数。

```shell
lunch

# 输出为：
You're building on Darwin
Lunch menu... pick a combo:
     1. aosp_arm-eng
     2. aosp_arm64-eng
     3. aosp_mips-eng
     4. aosp_mips64-eng
     5. aosp_x86-eng
     6. aosp_x86_64-eng
     7. full_fugu-userdebug
     8. aosp_fugu-userdebug
     9. mini_emulator_arm64-userdebug
     10. m_e_arm-userdebug
     11. m_e_mips-userdebug
     12. m_e_mips64-eng
     13. mini_emulator_x86-userdebug
     14. mini_emulator_x86_64-userdebug
     15. aosp_dragon-userdebug
     16. aosp_dragon-eng
     17. aosp_marlin-userdebug
     18. aosp_sailfish-userdebug
     19. aosp_flounder-userdebug
     20. aosp_angler-userdebug
     21. aosp_bullhead-userdebug
     22. hikey-userdebug
     23. aosp_shamu-userdebug
Which would you like? [aosp_arm-eng]
```

这个命令是让我们选择编译目标，每个版本对应着不同的应用场景，具体说明可以到官网查看。对于我来说我只是为了查看代码，所以选择了`6. aosp_x86_64-eng`，选择之后会在执行一段时间的操作，完成之后就可以开始编译了：

```shell
make -j4
```

后面的`-j4`表示编译代码是同时编译的数量，一般为当前计算机 CPU 核心数的2倍。

这个执行过程中可能会遇到一些错误，如`ninja: build stopped: subcommand failed.make: *** [ninja_wrapper] Error 1`，解决方案是找到`AOSP/prebuilts/sdk/tools/jack-admin`这个文件中的`start-server)`函数，然后将这后面的一段代码改为：

```shell
isServerRunning
   RUNNING=$?
   if [ "$RUNNING" = 0 ]; then
     echo "Server is already running"
   else
     #JACK_SERVER_COMMAND="java -XX:MaxJavaStackTraceDepth=-1 -Djava.io.tmpdir=$TMPDIR $JACK_SERVER_VM_ARGUMENTS -cp $LAUNCHER_JAR $LAUNCHER_NAME" #原有的隐藏
     JACK_SERVER_COMMAND="java -Djava.io.tmpdir=$TMPDIR $JACK_SERVER_VM_ARGUMENTS-Xmx2048M -cp $LAUNCHER_JAR $LAUNCHER_NAME"	#新增的
     echo "Launching Jack server" $JACK_SERVER_COMMAND
```

源码编译会花费大量的时间，几个小时不等，这时候只能耐心的等待。

## 查看

经过漫长的等待之后，源代码编译就完成了。为了将源代码导入 Android Studio 进行查看，还要执行一些操作：

```shell
mmm development/tools/idegen/
development/tools/idegen/idegen.sh
```

第一条命令为编译 idegen 模块，如果系统提示`command not found`，那就重新执行一遍`source build/envsetup.sh`。

第二条命令是生成 Android Studio 需要的工程配置文件`android.ipr`和`android.iml`，以使得 Android Studio 能够打开这个项目。

然后就可以打开 AS ，选择打开文件，选取这个`android.ipr`文件，选择之后就又是一段漫长的 AS 导入源码的过程，一般为1到2小时左右。

Android 源码导入成功以后，还需要进行一些配置，首先，打开`Project Structure`选项，进入`SDKs`选项，新建一个名为 android_aosp_jdk 的空 jdk ，作为项目的依赖 jdk ，如图：

![](http://blog-1251826226.coscd.myqcloud.com/screen2018-01-13%2001.01.08.png)

在这个 jdk 里面，其中所有标签的依赖，都被删除了，然后进入那个`Moudles`栏中的`Dependencies`标签，将`Moudle SDK`更改为刚刚新建的`android_aosp_jdk`这个空的 jdk ，并且删除除了 `Moudle source`和`android_aosp_jdk`这两项以外的所有以来项，结果如图：

![](http://blog-1251826226.coscd.myqcloud.com/screen2018-01-13%2001.08.32.png)

然后转至这里面的`Source`标签，找到`out/target/comment/R`文件夹，右键将其设置为`Source`，完成之后点击`Apply`然后退出，AS 就会执行以上配置，然后就可以轻轻松松地在各个 Android 源码之间跳转了。

## 总结

以上就是一个从下载 Android 源码到能够适用 Android Studio 阅读源码的完整流程。我从开始下载源码，一直到能够正常地查看，使用了三天的时间，当然三天里面大部分的时间都是等待下载完成、等待编译完成、等待加载完成等等。这样一套流程下来，总共需要占用的空间要200g左右，所以我选择的是使用外接的移动硬盘来存放源码，这也导致了各种加载慢，更慢。大部分的 Mac 应该是不足以空出这么多空间来放置源码的，使用移动硬盘的方法，就是可以直接将移动硬盘格式化为大小写敏感的磁盘，然后就可以直接挂载这个硬盘在硬盘里面操作，或者也可以在移动硬盘里面新建一个磁盘映像，方法与在本地新建类似。

## 参考

[http://szysky.com/2016/07/12/mac系统android编译源码](http://szysky.com/2016/07/12/mac系统android编译源码/)

[https://agehua.github.io/2017/08/01/aosp-compile](https://agehua.github.io/2017/08/01/aosp-compile/)

[https://www.jianshu.com/p/1513fc9e1a74](https://www.jianshu.com/p/1513fc9e1a74)

[https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/](https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/)

