---
title: Android apk 编译流程
date: 2018-03-13 09:30
tags:
	- android
---

## Apk 编译

Android 软件开发，一般情况下是指将一个空白的项目，一点一点变得丰满，使得这份代码生成的 Android 应用能够完成越来越多的功能的过程，所以大部分人的工作就是，写代码，然后点击运行，就可以看到代码变现成了一个一个的功能，所以大部分的软件开发也就到写代码为止了。不过从一段代码变成一个 Android 应用的 .apk 安装包是有一个过程的，即为编译过程，需要将一个 Android 项目所包含的内容经过转化、打包变成一个统一的文件，供 Android 系统使用。

一般时候，一个`.apk`文件就是一个`.zip`压缩文件，可以通过解压一个`.zip`的方式对一个`.apk`进行解压，解压之后就可以看到，`.apk`文件中会包含以下几个文件（夹）：

1.  AndroidManifest.xml
2.  asserts/
3.  classes.dex
4.  lib/
5.  res/
6.  resources.arsc

可以看到，打包之后还会有几个主要的文件，

### AndroidManifest.xml

应用的基本配置文件，包括应用包名、应用需要权限、应用包含的四大组件以及一些其他的第三方库需要的配置等等。用户在安装一个应用的时候，系统会首先读取这个文件，了解到应用的基本信息，以此来判断系统是否可安装这个软件并向用户展示这里面列出的需要的权限，另外当系统安装了一个软件之后，这个文件包含的信息就会被系统记录在案，例如当其他软件需要通过隐式 Intent 打开一个 Activity 的时候，系统就会扫描所有的这些 AndroidManifest.xml 中列出的 Activity 中是否有适合此 Intent 的 Activity 并打开。

### asserts/

打开这个文件夹就会看到，里面都是一些`.png`文件，对应的是项目中`drawable/`文件夹中的 Bitmap 类型的文件，如果应用需要用的资源文件还有视频或者音频的话，这些资源文件也会放到这个文件夹下。

### classes.dex

classes.dex 是应用的 Java 代码的一个打包文件，是 Dalvik 虚拟机的可执行文件，包括应用所有的逻辑处理，也就是我们编写的`.java`文件的归所。Android 操作系统是基于 Linux 操作系统的，在桌面端我们使用 Jvm 运行 Java 代码，Dalvik 的功能也跟 Jvm 差不多，每当打开一个 App ，系统就会为应用分配一个 Dalvik 虚拟机，Dalvik 就会加载应用的`classes.dex`文件完成应用的开启。关于 classes.dex ，有一个方法数不能超过 64k 的限制，这个是因为 Dex 本身格式的原因所限，不过可以通过一定的方法避免这个问题，如打包成多个 Dex 文件，这样就不会超过限制了，具体的方法可查看[http://yifeng.studio/2016/10/26/android-64k-methods-count/](http://yifeng.studio/2016/10/26/android-64k-methods-count/)。

### lib/

lib/ 文件夹里包含的一般是一些 .so 库，如果在项目中使用到了这些库，一般就会原封不动地转移到这里。

### res/

这个文件夹对应的是项目中的`res/`文件夹，包含的是应用中需要使用的各种`.xml`资源文件，不过这里不是直接移动，而是进行了一定的处理，如将文件变为二进制编码等。 

### resources.arsc

应用中使用到的资源，如图片、`.xml`文件等在打包之后，会生成`R.java`与`resource.arsc`这两个文件，前者存放着资源的ID，与其他`.java`文件一同打包进`.dex`文件，供在代码里索引资源，后者则是相应的资源索引表，能够根据当前设备的信息，得到不同的资源，如对于使用不同语言、尺寸不同的设备，会索引到不同的字符串资源或者图片资源。

## 编译流程

首先看 Android 官网的流程图：

![](https://developer.android.com/images/tools/studio/build-process_2x.png?hl=zh-cn)

再来一张官网老版的：

![](http://blog-1251826226.coscd.myqcloud.com/build.png)

这两张图共同描述了一个过程，那就是从一个 Android Studio 里面的项目生成一个能够供手机安装的`.apk`安装包的所要经过的步骤，不管是通过 Android Studio 或者是 Eclipse 一键生成安装包，还是在终端里使用各种工具一步步生成，原理还都是这么个原理，使用到的工具，有 aapt 、 aidl 、 Java Complier 、 apkbuilder 、 Jarsigner 、 zipalign 。下面就看看这些工具到底都是干什么用。

### aapt

**The Android Asset Packaing Tool（Android 资源打包工具）**，用于为资源文件打包，`.xml`文件会被转化为二进制文件，图片文件保持不变，并生成资源对照表`R.java`文件。

### aidl

**Android Interface Definition Language（Android 接口定义语言）**，处理应用中使用到的`.aidl`文件，并生成对应的 Java 接口文件。

### Java Complier

对上述两个过程生成的`R.java`文件、Java 接口文件以及程序的源代码（各种`.java`）文件进行编译，生成`.class`文件。

### dex

对程序的`.class`文件，以及项目中使用到的第三方库的文件进行整合处理，得到`.dex`文件，这个文件能够被 Dalvik 虚拟机识别、运行。

### apkbuilder

将上述步骤生成的`.dex`文件、`.xml`文件、`resources.arsc`文件以及其他资源文件一同打包，生成`.apk`文件。

### Jarsigner

要生成一个安装包，光是得到一个`.apk`安装包还不行，还需要对这个安装包进行签名，用来建立应用与开发者之间的联系，同时 Android 系统不允许安装未经签名的安装包。签名的时候需要一个`keystore`文件，在日常使用 Android Studio 进行调试的时候，如果没有手动进行签名的相关配置，AS 就会自动添加一个默认的 Debug 类型的签名的。

### zipalign

zipalign 会对安装包进行简单的优化，使之运行效率更高。

签名完成之后，安装包就可以被系统安装并运行了，不过通过 zipalign 工具，能够将安装包进行对齐处理，具体来说就是将安装包中所有的资源文件距离文件起始位置的偏移量为 4 的整数倍，这样通过内存映射访问应用的`.apk`文件时速度能够更快，即减少运行时内存的使用。

## 参考

[http://shinelw.com/2016/04/27/android-make-apk/](http://shinelw.com/2016/04/27/android-make-apk/)

[http://blog.csdn.net/shagoo/article/details/7485958](http://blog.csdn.net/shagoo/article/details/7485958)

[http://www.iloveandroid.net/2016/08/16/android_resource_2/](http://www.iloveandroid.net/2016/08/16/android_resource_2/)

[http://shinelw.com/2016/04/27/android-make-apk/](http://shinelw.com/2016/04/27/android-make-apk/)

[http://www.cnblogs.com/xirihanlin/archive/2010/04/12/1710164.html](http://www.cnblogs.com/xirihanlin/archive/2010/04/12/1710164.html)

## 附录

### 1 详细流程

![](https://user-gold-cdn.xitu.io/2017/3/2/31991e580b91cdc71280bcc0d7159314.png?imageView2/0/w/1280/h/960/ignore-error/1)