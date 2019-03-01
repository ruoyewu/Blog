---
title: Android OkHttp 分析
date: 2018-05-28 12:30
tags:
	- android
---

## OkHttp

OkHttp 是 Android 上的一个第三方网络请求库，提供了一系列关于网络请求的方法，现今一般的 APP 都会涉及到网络请求，而且一般是比较频繁的网络请求，所以会用一个网络请求库，以及了解其内部实现原理，是有必要的。

对于 OkHttp 网络请求库来说，发起一个网络请求需要用到类有这么几个：

1.  OkHttpClient：OkHttp 用来构建 Call 的工厂类，也是使用 OkHttp 首先需要用到的类，通过对 OkHttpCilcent 配置一些参数，来影响使用它构建出来的 Call 类的属性。
2.  Request：代表着一个 Http 的请求，包含一个请求的 URL 、方法、参数等。
3.  RequestBody：请求体，存储着 Http 请求的参数，可以是字符串、多媒体数据等。
4.  Call：代表一次 Http 请求，包括对应的 Request 和 Response 。
5.  Response：代表一个 Http 的响应。

然后一个网络请求的代码如下：

`Responsse = OkHttpClicent.newCall(Request).execute()`

下面就上述这个过程分别讲述各个阶段做了哪些事情。

### OkHttpClient

OkHttpClient 是每一个网络请求都要使用到的相当于入口类的一个类，一般书写的时候会选择将 OkHttpClient 作为一个单例使用。OkHttpClient 的作用就是配置所有网络请求的基本配置，如设置 timeout 这些，它有自己的 Builder 类，提供了比较多的设置选项，添加 Interceptor ，设置 Cache 、 Proxy 、 Dispatch 等等，没有设置的话就会使用默认的。

### RequestBody

RequestBody 包含了请求使用的参数，他还有两个子类，FormBody 和 MultipartBody ，前者用于一般的表单数据，后者用于混合类型的数据，如同时使用了多个 MIME 类型传递数据（同时使用到了字符串和文件等），大致的实现就是 MultipartBody 内部包含了多个 RequestBody 实例。

### Request

Request 存放了关于一个网络请求的全部数据，URL 、Method 、Body 等等，它也是使用 Builder 构建的。

### Call

OkHttpClient 根据 Request 构建出 Call ，具体的实现类是 RealCall ，它负责完成的工作比较多。它提供了两种网络请求的方式，分别对应着两个方法：

1.  execute
2.  enqueue

两个方法分别对应着同步网络请求和异步网络请求，异步请求的时候需要使用它的一个子类 AsyncCall ，这个子类实现了 NameRunnable 接口，并且在这个过程中使用到了线程池，具体在下面的 Dispatcher 中完成。

如果使用 execute 方法发起请求的话，就会直接调用它的 getResponseWithInterceptorChain 方法，完成整个网络请求的过程，所有的如封装 header 、 缓存处理 、 网络请求等等，都是在这个方法里面完成。

OkHttp 处理网络请求的过程的时候使用到了一个个人觉得比较有趣的方法，那就是 Interceptor Chain，Interceptor 意为“拦截器”，在 OkHttp 中，这个 Interceptor 完成了几乎所有的网络请求的工作，它通过一种“拦截链”措施，在每个拦截器中完成了不同的工作，同时还支持用户添加自己的拦截器，从代码层面上将整个网络请求的过程明显地区分开来，不得不说，很美，让人一眼就能看透它的处理过程。

具体见下面 Interceptor 。

### Dispatcher

当 Call 调用 enqueue 方法的时候，Dispatcher 会将这个请求加入线程池等待完成，这个 Dispatch 是可以在构建 OkHttpClient 的时候自己设置的，可以自定义选取哪个线程池。完了之后最后还是会调用 AsyncCall 的 execute 方法，然后调用到 getResponseWithInterceptorChain 方法。

### Interceptor

首先列出`RealCall.getResponseWithInterceptorChain()`方法：

```java
Response getResponseWithInterceptorChain() throws IOException {
  // Build a full stack of interceptors.
  List<Interceptor> interceptors = new ArrayList<>();
  interceptors.addAll(client.interceptors());
  interceptors.add(retryAndFollowUpInterceptor);
  interceptors.add(new BridgeInterceptor(client.cookieJar()));
  interceptors.add(new CacheInterceptor(client.internalCache()));
  interceptors.add(new ConnectInterceptor(client));
  if (!forWebSocket) {
    interceptors.addAll(client.networkInterceptors());
  }
  interceptors.add(new CallServerInterceptor(forWebSocket));
    
  Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null,null,0,originalRequest,this,eventListener,client.connectTimeoutMillis(),client.readTimeoutMillis(),client.writeTimeoutMillis());
    
  return chain.proceed(originalRequest);
}
```

这里可以看到所有的 Interceptor 以及它们的层次，用户定义的 Interceptor 处于最外层，然后依次是`retryAndFollowUpIntercetror` `BridgeInterceptor` `CacheInterceptor` `ConnectInterceptor` `client.netWorkInterceptors` `CallServerInterceptor`，最后构建了 Interceptor.Chain ，并开始请求。

上面的每个拦截器完成了一件工作，拦截链内部调用的方式是，首先调用最外层的 Interceptor ，然后这个 Interceptor 会继续调用下一层 Interceptor 的方法，对于一个 Interceptor 来说，它可以在下一层 Interceptor 执行之前做一些预处理的工作，也可以在下一层 Interceptor 执行之后再做一些后续的处理事宜。

如下是`Chain.proceed()`方法：

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
    RealConnection connection) throws IOException {
  if (index >= interceptors.size()) throw new AssertionError();
  calls++;
  // If we already have a stream, confirm that the incoming request will use it.
  if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
    throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
        + " must retain the same host and port");
  }
  // If we already have a stream, confirm that this is the only call to chain.proceed().
  if (this.httpCodec != null && calls > 1) {
    throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
        + " must call proceed() exactly once");
  }
  // Call the next interceptor in the chain.
  RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
      connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
      writeTimeout);
  Interceptor interceptor = interceptors.get(index);
  Response response = interceptor.intercept(next);
  // Confirm that the next interceptor made its required call to chain.proceed().
  if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
    throw new IllegalStateException("network interceptor " + interceptor
        + " must call proceed() exactly once");
  }
  // Confirm that the intercepted response isn't null.
  if (response == null) {
    throw new NullPointerException("interceptor " + interceptor + " returned null");
  }
  if (response.body() == null) {
    throw new IllegalStateException(
        "interceptor " + interceptor + " returned a response with no body");
  }
  return response;
}
```

这个方法最终会调用`Interceptor.intercept(Chain)`方法，index 参数标识了当前要处理的 interceptor 所在的位置，然后每次 interceptor 调用下一层 interceptor 的时候，会直接调用`Chain.proceed(Request)`方法，可以看到这里的传参是 Request ，也就是说，每过一层 interceptor ，就可以生成一个新的 Request ，然后它的返回参数是 Response ，这也就意味着，每当一个 interceptor 获取到下一层 interceptor 生成的 Response 之后，也可以使用自己的规则处理一遍 Response ，然后再将 Response 传给自己的上一层。

#### client.interceptors()

从最外层往里一层一层看 Interceptor ，如果用户需要在所有的网络请求上加上同一个参数，或者设置同一个请求头的话，就可以使用一个自定义的拦截器，然后网络请求开始之前统一地处理初始的 Request ，就可以比较轻松地实现这个功能，而对于外部的构建 Request 的过程来说，完全不需要再考虑这种问题。

#### RetryAndFollowUpInterceptor

内部是一个 While 无限循环，然后循环里面有是否取消请求判断，如果是因为非正常因素导致的请求失败，就会重新请求，直到重传次数超限等。

#### BridgeInterceptor

会为 Request 设置默认的请求头，以及通过 CookieJar 处理一些 Cookie 相关的工作（自动获取 Cookie 、自动添加 Cookie）等，同时还会给获取到的 Response 添加一些缺失的响应头字段。

#### CacheInterceptor

Cache 相关，处理上一层请求的时候，可能会将缓存的 Response 返回，并且将下层得到的 Response 加入缓存等操作。

#### ConnectInterceptor

开启网络连接，从网络获取初始数据。具体工作由 StreamAllocation 完成，如 Socket 的连接，数据获取，SSL/TLS 握手等等。

#### client.networkInterceptors()

这里也是可以供用户自定义的 interceptor ，相对于上面的可以工作在网络请求的更底层。

#### CallServerInterceptor

处理请求数据，将请求到的数据格式化成 Response ，根据响应码做一些对应的操作，然后还有一些关于 StreamAllocation 的操作，如关闭等。最后将得到的 Response 返回，经过一层层的返回和处理之后，就会从 getResponseWithInterceptorChain 方法返回出去，由 Call 得到这个 Response ，再根据发起请求时的同步或者异步方式，将 Response 送出。

### Response

Response 包含网络请求之后的结果，这些结果是经过处理过的，可以直接调用相应的方法获取相应头的某些参数或者是响应体。

### 总结

经过上面的一整个流程之后，一个网络请求就算是结束了，我们由最初的 Reques 得到了 Response ，无数的这些网络请求，就构成了 APP 内部一条完成的数据链，连接着客户端和服务端，只要在有网的地方，用户就可以获取他想要的数据。

## 重难点

### 线程池的使用

### 拦截链的使用

### Socket 连接过程