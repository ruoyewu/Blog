---
title: Android URI 权限
date: 2018-03-22 19:30
tags:
	- android
---

## 背景

自从 Android 7.0(M) 版本之后，Android 的资源文件访问方式出现了变化。在开发的过程中，有些时候我们需要使用 Uri 和 File 互相转化，比如调用系统的相册选取图片的时候， 就是通过传递文件的 Uri 传递选取的图片的，我们在得到这个 Uri 之后一般并不能直接使用，还要通过一定的方法，得到文件的路径名，进而就可以构建一个 File 对象对文件进行操作，不过在 7.0 之后，这个返回的 Uri 跟之前的 Uri 不一样了，直接通过之前通过 Uri 获取文件路径的方法在这里不适用了，自然也要寻找新的解决办法。

## 新的 Uri

```java
Uri uri = Uri.fromFile(File)

uri : "file:///storage/emulated/0/DCIM/Camera/IMG_20180322_194038.jpg"

↓

Uri uri = FileProvider.getUriForFile(Context, String, File)

uri : "content://com.android.providers.media.documents/document/image%3A767"

```

以上就是通过两种方法分别得到的 Uri ，可以看到，旧版本的方法得到的 Uri ，直接就是`file://`后面加上文件的路径名，而新版本的方法得到的 Uri ，则是以`content://`开头的，后面跟上对应的参数的一串代码，就光看这个就能感觉到，Android 本身的安全级别确实是提高了不少（文件路径终于不是直接显示出来了），但是安全系数上去了，对应的操作也就复杂了，如果要使用 FileProvider 这个类的话，需要进行一系列的配置才行，见最后。另外，通过旧的方式获取的 Uri 是不能在 7.0 以后正常使用的，因为 Android 对 Uri 的使用加上了一些安全机制，得是后面的那种格式才行。获取的 Uri 不同了，那么通过 Uri 得到文件路径的方式肯定也有变化，我们通过`Uri.getPath()`方法直接对后者使用的时候，会发现得到的 path 是`/document/image:767`，这显然不是文件对应的路径，所以，与之配套的还有一个通过 Uri 得到路径的方法，源码见最后。

有这么一个有趣的过程，首先通过调用系统的相册选取图片，返回了图片的 Uri ，然后再调用系统的图片剪裁对图片进行剪裁工作，传给这个负责剪裁的 Activity 的是相册返回的 Uri 。然后代码写好，放上去运行，发现每次进行剪裁的时候，剪裁的结果都是`resultCode == RESULT_CANCEL`，查阅资料后得知，需要在构建 Intent 的时候给目标应用一个授权，即使用`intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION) | Intent.FLAG_GRANT_WRITE_URI_PERMISSION)`给目标应用操作传入的 Uri 的权限，然后再运行，发现可以对图片进行剪裁了，但是点击保存之后，返回的结果仍然是`resultCode == RESULT_CANCEL`，并且附加一个错误输出：

```java
Writing exception to parcel
java.lang.SecurityException: Permission Denial: writing android.support.v4.content.FileProvider uri content://com.wuruoye.demo.fileprovider/external_files/com.wuruoye.demo/image/1521722273255.jpg from pid=19096, uid=10016 requires the provider be exported, or grantUriPermission()
    ...
    ...
```

这里，可以看到说是这个负责剪裁的应用没有对这个 Uri 写的权限，同时还给了提示，那么按照这个提示来，通过`grantUriPermission()`方法对应用授权，

## FileProvider 使用

## 通过 Uri 得到路径