---
title: Android P 网络请求失败
date: 2019-02-13 16:00
tags:
	- android
---

很长一段时间没有碰过 Android 开发了，最近重拾之前开发过的软件，先更新了一下版本，将 targetSdkVersion 更改为了 28 ，重新编译安装之后发现怎么也加载不出来东西，并且有报错，java.net.UnknownServiceException: CLEARTEXT communication to all.wuruoye.com not permitted by network security policy报错内容为：

```
java.net.UnknownServiceException: CLEARTEXT communication to all.wuruoye.com not permitted by network security policy
```

我使用的是网络请求库是 Okhttp ，第一眼看到这个错误我还以为是我忘了给 App 添加网络权限了，立马翻到 Manifests 中明明看到所有权限都在的，故而转求于 Google 。一番搜索找到了原因，大致就是在 Android P Google 禁用了 Http 请求，只能使用 Https ，意在营造一个更加安全的网络环境。

那么这个问题如何解决？大致有以下三个：

### 1. 改用 Https

既然推荐使用 Https ，那么将所有的网络请求都换作 Https 自然而言就能够解决这个问题，这对一个新系统而言可能比较简单，但是对于一个老系统来说，可能因为各种各样的问题，不是那么容易就能够换成 Https （比如原先的服务器并没有做成 Https 的，那么这个改动可能需要一段不太短的时间，新版本的 APP 显然等不了）。那么就需要一定的办法使得软件可以使用 Http 请求。

### 2. 更改 targetSdkVersion

为了使 Android P 兼容之前版本的 App（应该有不少是直接使用 Http 的），所有的 targetSdkVersion 在 28 以下的软件都不受新系统的这些规则的限制，所以为了使软件能够立马在 Android P 系统中使用，直接将targetSdkVersion 改为 27 是一个不错的办法，但总归不是长久之计（最好能在这段时间内将所有的 Http 都转变为 Https ，以便下一个版本的更新）。

### 3. 设置允许使用 Http

或许有的 App 因为某些原因不得不使用 Http （Google 不得不考虑），所以还有另外一种办法，即在原来的基础上增加一个网络设置，使其允许使用 Http 的网络请求。

首先需要一个配置文件，xml 格式的，添加在`src/xml/`文件夹下，假设名字为`network_config.xml`，内容为：

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config xmlns:android="http://schemas.android.com/apk/res/android">
    <base-config cleartextTrafficPermitted="true" />
</network-security-config>
```

然后还要更改`AndroidManifest.xml`文件中`application`结点的属性：

```xml
<application
             ...
             android:networkSecurityConfig="@xml/network_config"
             ...>
</application>
```

大致就是给 application 添加一个网络安全方面的配置，然后在配置文件中允许进行 Http 请求，这样一来 App 就可以在 Android P 系统中使用 Http 了。

个人感觉，虽然 Android 的一大优点是开源，并且 Android 对它的软件有很大的包容性（相比较 IOS 的强硬管理而言），这是一个优点，但也是一大缺点，导致了 Android 系统被各种流氓软件荼毒得越来越卡顿等，并且可以看到，在 Android 系统的近几个版本中，Android 对系统的安全性也越来越看重，个人感觉是随着互联网的发展，手机已经远不止是一个工具这么简单了，手机支付这些东西的出现，导致手机的安全所占的比重越来越大，也有越来越多的人更看重的是手机是否安全，或者是系统是否安全，所以提高手机系统的安全程度也越来越重要，所以，提供一个更加可靠的网络环境，大概就是驱使 Android P 作出这样一个改动的原因。

### 参考

[https://android-developers.googleblog.com/2018/04/protecting-users-with-tls-by-default-in.html](https://android-developers.googleblog.com/2018/04/protecting-users-with-tls-by-default-in.html)