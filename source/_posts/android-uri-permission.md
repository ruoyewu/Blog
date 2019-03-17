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

这里，可以看到说是这个负责剪裁的应用没有对这个 Uri 写的权限，同时还给了提示，那么按照这个提示来，通过`grantUriPermission()`方法对应用授权。也就是说这个剪裁图片的这个 Activity 还是没有得到这个 Uri 的授权，到那时我已经在代码中给了授权，这是为什么呢？有两种可能：

1.  虽然我写了给这个 Activity 授权，但是在执行的过程中没有正确授权
2.  我没有这个 Uri 的授权权，所以我给这个 Activity 的授权是无效的

我是否有这个 Uri 的授权权？先看这个 Uri 的出处，我是先利用相册选择了一个图片，然后相册返回给了我这个图片的 Uri ，所以这个 Uri 应该是相册生成的，那么按照一般的道理，如果这个 Uri 不是我生成的，我应该是没有这个 Uri 的使用权的，那么这个一想，如果我利用这个 Uri ，找到它对应的图片文件，然后再利用文件路径生成一个新的 Uri 是否就可以使用了呢？于是我就这么做了，在要调用剪裁 Activity 之前，我通过某种方法（见下面）得到了这个图片的文件路径，然后再利用`FileProvider.getUriForFile()`生成了一个 uri ，以此作为剪裁 Activity 的参数，再次调用剪裁 Activity 的时候，果然可以使用了。

也就是说，我不使用相册 Activity 提供的 Uri ，而是自己通过文件路径生成的 Uri ，就可以给剪裁 Activity 正常使用。这虽然是一个解决问题的办法，但是这个问题出现的根本原因我还是不了解，希望之后会有机会真正了解 Uri 的这个权限机制。

## 通过 Uri 得到路径

要使用 Uri 得到对应文件的路径，由于 7.0 之后 Uri 机制的改变，获取文件路径的方式也有改变：

```java
public static String getFilePathByUri2(Context context, Uri uri) {
    try {
        String[] proj = { MediaStore.Images.Media.DATA };
    	CursorLoader loader = new CursorLoader(context, uri, proj, null, null, null);
    	Cursor cursor = loader.loadInBackground();
    	int column_index = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.DATA);
    	cursor.moveToFirst();
    	return cursor.getString(column_index);
    } catch (Exception e) {
        
    } finally {
        if (cursor != null) {
            cursor.close();
        }
    }
}
public static String getFilePathByUri(Context context, Uri uri) {
    String path = null;
    // 以 file:// 开头的
    if (ContentResolver.SCHEME_FILE.equals(uri.getScheme())) {
        path = uri.getPath();
        return path;
    }
    // 以 content:// 开头的，比如 content://media/extenral/images/media/17766
    if (ContentResolver.SCHEME_CONTENT.equals(uri.getScheme()) && Build.VERSION.SDK_INT < Build.VERSION_CODES.KITKAT) {
        Cursor cursor = context.getContentResolver().query(uri, new String[]{MediaStore.Images.Media.DATA}, null, null, null);
        if (cursor != null) {
            if (cursor.moveToFirst()) {
                int columnIndex = cursor.getColumnIndexOrThrow(MediaStore.Images.Media.DATA);
                if (columnIndex > -1) {
                    path = cursor.getString(columnIndex);
                }
            }
            cursor.close();
        }
        return path;
    }
    // 4.4及之后的 是以 content:// 开头的，比如 content://com.android.providers.media.documents/document/image%3A235700
    if (ContentResolver.SCHEME_CONTENT.equals(uri.getScheme()) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        if (DocumentsContract.isDocumentUri(context, uri)) {
            if (isExternalStorageDocument(uri)) {
                // ExternalStorageProvider
                final String docId = DocumentsContract.getDocumentId(uri);
                final String[] split = docId.split(":");
                final String type = split[0];
                if ("primary".equalsIgnoreCase(type)) {
                    path = Environment.getExternalStorageDirectory() + "/" + split[1];
                    return path;
                }
            } else if (isDownloadsDocument(uri)) {
                // DownloadsProvider
                final String id = DocumentsContract.getDocumentId(uri);
                final Uri contentUri = ContentUris.withAppendedId(Uri.parse("content://downloads/public_downloads"),
                        Long.valueOf(id));
                path = getDataColumn(context, contentUri, null, null);
                return path;
            } else if (isMediaDocument(uri)) {
                // MediaProvider
                final String docId = DocumentsContract.getDocumentId(uri);
                final String[] split = docId.split(":");
                final String type = split[0];
                Uri contentUri = null;
                if ("image".equals(type)) {
                    contentUri = MediaStore.Images.Media.EXTERNAL_CONTENT_URI;
                } else if ("video".equals(type)) {
                    contentUri = MediaStore.Video.Media.EXTERNAL_CONTENT_URI;
                } else if ("audio".equals(type)) {
                    contentUri = MediaStore.Audio.Media.EXTERNAL_CONTENT_URI;
                }
                final String selection = "_id=?";
                final String[] selectionArgs = new String[]{split[1]};
                path = getDataColumn(context, contentUri, selection, selectionArgs);
                return path;
            }
        }
    }
    return null;
}
private static String getDataColumn(Context context, Uri uri, String selection, String[] selectionArgs) {
    Cursor cursor = null;
    final String column = "_data";
    final String[] projection = {column};
    try {
        cursor = context.getContentResolver().query(uri, projection, selection, selectionArgs, null);
        if (cursor != null && cursor.moveToFirst()) {
            final int column_index = cursor.getColumnIndexOrThrow(column);
            return cursor.getString(column_index);
        }
    } finally {
        if (cursor != null)
            cursor.close();
    }
    return null;
}
private static boolean isExternalStorageDocument(Uri uri) {
    return "com.android.externalstorage.documents".equals(uri.getAuthority());
}
private static boolean isDownloadsDocument(Uri uri) {
    return "com.android.providers.downloads.documents".equals(uri.getAuthority());
}
private static boolean isMediaDocument(Uri uri) {
    return "com.android.providers.media.documents".equals(uri.getAuthority());
}
```

以上是两种通过 Uri 求路径的方法。