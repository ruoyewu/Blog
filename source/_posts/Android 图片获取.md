---
title: Android 拍照、打开相册
date: 2017-05-06 00:00:00
categories: android
tags: android
---

## 获取照片方法

1.  打开相册方法

    ```java
    static final int OPEN_ALBUM = 1;
    Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
    intent.setType("image/*");
    intent.addCategory(Intent.CATEGORY_OPENABLE);
    startActivityForResult(intent,OPEN_ALBUM);
    ```

    这里`Intent.ACTION_GET_CONTENT`表明了要打开的activity的类型，`setType()`标示了要选取的文件类型。

2. 打开相机方法

    ```java
    Uri uri;
    static final int OPEN_CAMERA = 2;
    Intent intent = new Intent();
    intent.putExtra(MediaStore.EXTRA_OUTPUT,uri);
    startActivityForResult(intent,OPEN_CAMERA);
    ```

    这里`Intent.EXTRA_OUTPUT`指定了拍照之后将照片存储的Uri路径，这里需要提供一个Uri，下面是由file得到Uri的方法：

    ```java
    File file = new File(filePath);
    Uri uri;
    if (Build.VERSION.SDK_INI < 21){
      uri = Uri.fromFile(file);
    }else {
      uri = FileProveder.getUriForFile(context,authority,file);
    }
    ```

    filePath是图片保存的路径，其中可以看到，uri的获取分为sdk版本21以上和以下，这是因为在api 21以上之后，android更改了文件系统，就是说app不回再把`file://`Uri暴露给其他app，包括通过Intent方法或ClipData方法，原因是使用`file://`Uri会存在某些风险。所以，google提供了FileProvider，使用它可以生成`content://`Uri来代替`file://`Uri。

    解决方案如下，

    首先在`AndroidManifest.xml`文件中添加provider：

    ```xml
    <provider
              android:name="android.support.v4.content.FileProvider"
              android:authorities="appPackge.fileprovider"
              android:exported="false"
              android:grantUriPermissions="true">
      	<meta-data
                   android:name="android.support.FILE_PROVIDER_PATHS"
                   android:resource="@xml/provider_paths"/>
    </provider>
    ```

    同时在新建一个文件 `res/xml/provider_paths.xml`

    ```xml
    <?xml version="1.0" encoding="utf-8"?>
    <paths
          xmlns:android="http://schemas.android.com/apk/res/android">
      <external-path name="external_files" path="."/>
    </paths>
    ```

    由此便可以解释上述的代码

    ```java
    uri = FileProveder.getUriForFile(context,authority,file);
    ```

    这里的`authority`就是在provider中定义的`authorities`。

3.  调用系统图片剪裁方法

    ```java
    static final int CROP_PHOTO = 3;
    Uri uri;
    Uri outUri;
    Intent intent = new Intent("com.android.camera.action.CROP");
    intent.setDataAndType(uri, "image/*");
    intent.putExtra("crop", true);
    intent.putExtra("aspectX", 1);
    intent.putExtra("aspectY", 1);
    intent.putExtra("outputX", 500);
    intent.putExtra("outputY", 500);
    intent.putExtra("return-data", false);
    intent.putExtra(MediaStore.EXTRA_OUTPUT, outUri);
    intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);
    intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION);

    List<ResolveInfo> resInfoList = getPackageManager().queryIntentActivities(intent, PackageManager.MATCH_DEFAULT_ONLY);
    for (ResolverInfo info : resInfoList){
      String packageName = info.activityInfo.packageName;
      grantUriPermission(packageName, uri, Intent.FLAG_GRANT_WRITE_URI_PERMISSION | Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    startActivityForResult(intent, CROP_PHOTO);
    ```

    上面列出了很多剪裁时需要用到的参数，大致就是一些剪裁比例，剪裁之后输出的像素大小，输入uri，还有就是申请操作uri的权限之类的。

    其中，uri 为待剪裁的原图片的uri， outUri 为剪裁之后输出的uri， 两个uri都可以由对应的文件路径获得。

    在实际使用过程中发现，在`cropPhoto`中使用到的这两个uri，获取途径如下：

    1.  outUri，一般都是输出到指定的文件夹，即在当前app中使用FileProvider获得某文件的uri。
    2.  uri，图片来源uri，来源有两种，一种是从相册中选取的时候相册app返回的uri，一种是调用相机app时app输出的uri，前一种uri为相册app提供，后一种则是在当前app内根据文件路径获取uri并作为输出uri，用法与剪裁图片类似，只是没有输入uri。

    而在调用系统剪裁功能的时候出现了下面的问题，进入剪裁app的时候会报错：

    ```java
    Caused by: java.lang.SecurityException: Uid 10384 does not have permission to uri 0 @ content://com.android.providers.media.documents/document/image%3A215598
    ```

    出现这种错误的原因是我直接将相册app返回的uri，即在`onActivityResult`中得到的` data.getData()`作为`cropPhoto`的输入uri，系统就会报错，意思大概是说相应的权限不允许，不过后来我又换了一种方式，通过相册app返回的uri，得到对应这个uri的文件路径，再由文件路径得到文件的uri，将这个uri作为`cropPhoto`的输入uri，就不会报错了。

    ```java
    String filePath = FilePathUtil.getPathFromUri(this, data.getData);
    Uri uri = FileProvider.getUriForFile(this, FILE_AUTHORITY, new File(filePath));
    cropPhoto(uri);
    ```

    其中`FILE_AUTHORITY`是下面所说的authorities，`FilePathUril.getPathFormUri()` 是自己定义的一个工具类，就是由uri得到文件路径。

    说来真的很奇怪，按理说应该是同一个uri，由两种方法获得的结果却不一样。因为第二种uri是在当前app内自己获取的，有可能会得到以下结论：一个uri不能在多个应用中多次传递。

    再后面就是关于uri的权限操作：

    ```java
    List<ResolveInfo> resInfoList = getPackageManager().queryIntentActivities(intent, PackageManager.MATCH_DEFAULT_ONLY);
    for (ResolverInfo info : resInfoList){
      String packageName = info.activityInfo.packageName;
      grantUriPermission(packageName, uri, Intent.FLAG_GRANT_WRITE_URI_PERMISSION | Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    ```

    第一句`getPackageManager().queryIntentActivities(intent, Packagemanager.MATCH_DEFAULT_ONLY)`是得到能够接收当前intent的所有acitivity，并给对应的app都申请了操作输入输出uri的权限，这样就可以确保在剪裁过程中不会出现权限不够的问题。

## FileProvider

### 定义FileProvider

就是在AndroidManifest.xml的`<application>`节点中添加`<provider>`。

```java
<provider
          android:name="android.support.v4.content.FileProvider"
          android:authorities="appPackge.fileprovider"
          android:exported="false"
          android:grantUriPermissions="true">
  	<meta-data
               android:name="android.support.FILE_PROVIDER_PATHS"
               android:resource="@xml/provider_paths"/>
</provider>
```

1.  `android:authorities`是用来标识provider的唯一标识，在同一个手机上一个“authority”串只能被一个app使用，冲突会导致app无法安装，所以我们一般使用`包名.fileprovider`来作authority。
2.  `android:exported`必须设置成false，否则运行时会报错`java.lang.SecurityException: Provider must not be exported`。
3.  `android:grantUriPermissions`用来控制共享文件的访问权限，也可以在java代码中设置。

### 总结

打开相册，拍照，剪裁等，是我在做一个需要选取用户头像功能的时候涉及到的，因为android7.0之后严格的文件读取权限，以及uri的改变，给这些需要操作系统文件及uri的地方带来的很多麻烦，看了很多博客，在stackOverFlow上也看了很多相关问题的解决办法，最后总算完成了这个功能，所有相关的代码都会在下面列出。