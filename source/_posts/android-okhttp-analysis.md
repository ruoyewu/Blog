---
title: Android OkHttp 原理分析
date: 2019-03-10 17:00
tags:
	- android
---

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

## 请求过程

### OkHttpClient

OkHttpClient 是每一个网络请求都要使用到的相当于入口类的一个类，构建实例的时候使用了 Builder ，一般会选择将 OkHttpClient 作为一个单例使用。OkHttpClient 的作用就是配置所有网络请求的基本配置，如设置 timeout 这些，它有自己的 Builder 类，提供了比较多的设置选项，添加 Interceptor ，设置 Cache 、 Proxy 、 Dispatch 等等，没有设置的话就会使用默认的。

OkhttpClient 主要是作为生产 Call 的工厂类出现，利用`newCall(Request)`方法：

```java
@Override public Call newCall(Request request) {
  return RealCall.newRealCall(this, request, false /* for web socket */);
}
```

这里利用了 Request 构建 Call 实例。

### Request

Request 也需要使用 Builder 构建，存放了关于一个网络请求的全部数据，URL 、Method 、Body 等等。

### RequestBody

RequestBody 包含了请求使用的参数，他还有两个子类，FormBody 和 MultipartBody ，前者用于一般的表单数据，后者用于混合类型的数据，如同时使用了多个 MIME 类型传递数据（同时使用到了字符串和文件等），大致的实现就是 MultipartBody 内部包含了多个 RequestBody 实例。

### RealCall

OkHttpClient 根据 Request 构建出 Call ，具体的实现类是 RealCall ，值得注意的是这里创建 RealCall 实例的时候不是直接调用构造方法，而是通过一个静态方法构造实例：

```java
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket)
  // Safely publish the Call instance to the EventListener.
  RealCall call = new RealCall(client, originalRequest, forWebSocket);
  call.eventListener = client.eventListenerFactory().create(call);
  return call;
}
```

RealCall 提供了两种网络请求的方式，分别对应着两个方法：`execute()`和`enqueue(Callback)`。

在 Call 中调用`execute()`方法时，在`execute()`中会提交给 Dispatcher 一个同步任务并立刻调用`getResponseWithInterceptorChain()`方法得到 Response 并将其返回。

```java
@Override public Response execute() throws IOException {
  synchronized (this) {
    if (executed) throw new IllegalStateException("Already Executed");
    executed = true;
  }
  captureCallStackTrace();
  timeout.enter();
  eventListener.callStart(this);
  try {
    client.dispatcher().executed(this);
    Response result = getResponseWithInterceptorChain();
    if (result == null) throw new IOException("Canceled");
    return result;
  } catch (IOException e) {
    e = timeoutExit(e);
    eventListener.callFailed(this, e);
    throw e;
  } finally {
    client.dispatcher().finished(this);
  }
}
```

当调用`enqueue(Callback)`方法时，因为是异步请求，首先会将 RealCall 和 Callback 一起构建出一个 AsyncCall ，在 AsyncCall 中利用 Callback 完成了结果的回调：

```java
@Override protected void execute() {
  boolean signalledCallback = false;
  timeout.enter();
  try {
    Response response = getResponseWithInterceptorChain();
    if (retryAndFollowUpInterceptor.isCanceled()) {
      signalledCallback = true;
      responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
    } else {
      signalledCallback = true;
      responseCallback.onResponse(RealCall.this, response);
    }
  } catch (IOException e) {
    e = timeoutExit(e);
    if (signalledCallback) {
      // Do not signal the callback twice!
      Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
    } else {
      eventListener.callFailed(RealCall.this, e);
      responseCallback.onFailure(RealCall.this, e);
    }
  } finally {
    client.dispatcher().finished(this);
  }
}
```

### Dispatcher

在 Dispatcher 中使用三个队列保存它收到的任务，分别是：

-   readyAsyncCalls，待执行的异步任务
-   runningAsyncCalls，正在执行的异步任务
-   runningSyncCalls，正在执行的同步任务

当在 RealCall 中调用`execute()`时，会直接开始网络请求，并将这个 Call 加入 runningSyncCalls 。

当在 RealCall 中调用`enqueue()`时，会首先将 AsyncCalls 加入到 readyAsyncCalls 中等待调度，并在`promoteAndExecute()`方法中根据 maxRequests 和 maxRequestsPerHost 决定是否将某个 AsyncCalls 提交给线程池。

```java
private boolean promoteAndExecute() {
  assert (!Thread.holdsLock(this));
  List<AsyncCall> executableCalls = new ArrayList<>();
  boolean isRunning;
  synchronized (this) {
    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall asyncCall = i.next();
      if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
      if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.
      i.remove();
      asyncCall.callsPerHost().incrementAndGet();
      executableCalls.add(asyncCall);
      runningAsyncCalls.add(asyncCall);
    }
    isRunning = runningCallsCount() > 0;
  }
  for (int i = 0, size = executableCalls.size(); i < size; i++) {
    AsyncCall asyncCall = executableCalls.get(i);
    asyncCall.executeOn(executorService());
  }
  return isRunning;
}
```

在 Dispatcher 中有四个地方会调用这个方法：

-   加入一个待执行的 AsyncCalls 
-   某个 Call 任务结束，将其移除运行队列时
-   更改 maxRequests 值
-   更改 maxRequestsPerHost 值

### Interceptor

无论是同步请求还是异步请求，最终都会调用`getResponseWithInterceptorChain()`方法完成网络请求并得到 Response ，OkHttp 的网络请求过程可谓是比较具有特色的，使用到了责任链模式，其中的好处就是低耦合、高内聚等，也给开发者扩展其功能提供了很好的入口，在使用 OkHttpClient.Builder 构建 OkHttpClient 实例的时候，就可以看到 Builder 有`addInterceptor(Interceptor)`和`addNetworkInterceptor(Interceptor)`方法供开发者加入自定义的拦截器，从下面的代码中可以看到，这两类代码分别选择了不同的位置加入：前者最先加入，后者在 ConnectInterceptor 之后加入。

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

上面的方法中加入了许多的 Interceptor 的子类，Interceptor 中只有一个方法：

```java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;
}
```

方法的参数是 Chain ，返回参数是 Response ，Chain 也是一个接口：

```java
interface Chain {
  Request request();
  Response proceed(Request request) throws IOException;
  /**
   * Returns the connection the request will be executed on. This is only available in the chains
   * of network interceptors; for application interceptors this is always null.
   */
  @Nullable Connection connection();
  Call call();
  int connectTimeoutMillis();
  Chain withConnectTimeout(int timeout, TimeUnit unit);
  int readTimeoutMillis();
  Chain withReadTimeout(int timeout, TimeUnit unit);
  int writeTimeoutMillis();
  Chain withWriteTimeout(int timeout, TimeUnit unit);
}
```

主要的有两个方法：`request()`方法能够得到一个 Request 实例，`proceed(Request)`能够得到 Response 实例，所以一般情况下，自定义 Interceptor 的步骤如下：

```java
public MyInterceptor implements Interceptor {
    Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        // do something with request
        Response response = chain.proceed(request);
        // do something with response
        return response;
    }
}
```

也就是说，通过一个自定义的 Interceptor ，我们可以直接对所有的网络请求的 Request 和 Response 进行修改，如给所有的 Request 都增加一个 header 等，所有在代码中需要自定义的一些设置，都可以放到这里进行。

上面说到，这是一个责任链模式，各个 Interceptor 的实现类构成责任链的结点，结点之间的串联是如何完成的？再看上面的`getResponseWithInterceptorChain()`方法，将所有的拦截器整合到一起之后，创建了一个 Interceptor.Chain 对象，并调用了它的`proceed(Request)`方法。Chain 的实现类是 RealInterceptroChain ，下面是关于它的`proceed(Request)`的实现：

```java
@Override public Response proceed(Request request) throws IOException {
  return proceed(request, streamAllocation, httpCodec, connection);
}
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

在这个方法中，首先创建了一个新的 Chain 对象，只有一个参数 index 作了 +1 改变，然后接着取出了索引值为 index 的 Interceptor ，并将这个新的 Chain 对象作为参数调用其`intercept(Chain)`方法，上面的关于自定义 Interceptor 的使用上可以知道，在 Interceptor 也是调用了 Chain 的`proceed()`方法得到 Response 的，此时就会调用到 index+1 位置的 Interceptor 的`intercept()`方法......由此便形成了一个类似于递归调用的调用链：每一层对 Request 进行一些处理，直到最后一层通过 Request 得到 Response ，再原路返回对 Response 进行处理。

除了开发者自定义添加的 Interceptor 之外，OkHttp 至少需要 5 个拦截器，分别是 RetryAndFollowUpInterceptor、BridgeInterceptor、CacheInterceptor、ConnectInterceptor 和 CallServerInterceptor 。在 RealCall 中初始化 Interceptor.Chain 的时候传入了很多参数，如下：

```java
Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
    originalRequest, this, eventListener, client.connectTimeoutMillis(),
    client.readTimeoutMillis(), client.writeTimeoutMillis());
```

其中的三个 null 分别是 StreamAllocation、HttpCodec 和 RealConnection ，这些参数在最开始构造 Chain 的时候是 null ，不过会在后续的一些拦截器调用的过程中不断将其完善。

#### RetryAndFollowUpInterceptor

这个拦截器从名字就可以看出个大概：重新和接续请求。整个流程大致如下：

```java
public Response intercept() {
    while(true) {
    	try {
    	    response = chain.proceed();
    	} catch (Exception e) {
    	    if (canRecover)
    	    	continue;
    	}
    
    	if (!followUp) {
        	return response;
    	}
    }
}
```

内部有一个死循环，首先通过 chain 调用下一层的 Interceptor 得到一个 Response ，如果这个过程中发生了错误，判断是否可以恢复请求，如果可以，那就重新再请求一次，否则抛出异常。然后根据 response 判断是否有下一步请求(`followUpRequest(Response, Route)`)，如果有就继续请求，否则返回这个 Response 。这就是 RetryAndFollowUpInterceptor 类的大致功能，类名确实已经概括地差不多了。

完整代码：

```java
@Override public Response intercept(Chain chain) throws IOException {
  Request request = chain.request();
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Call call = realChain.call();
  EventListener eventListener = realChain.eventListener();
  StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
      createAddress(request.url()), call, eventListener, callStackTrace);
  this.streamAllocation = streamAllocation;
  int followUpCount = 0;
  Response priorResponse = null;
  while (true) {
    if (canceled) {
      streamAllocation.release(true);
      throw new IOException("Canceled");
    }
    Response response;
    boolean releaseConnection = true;
    try {
      response = realChain.proceed(request, streamAllocation, null, null);
      releaseConnection = false;
    } catch (RouteException e) {
      // The attempt to connect via a route failed. The request will not have been sent.
      if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
        throw e.getFirstConnectException();
      }
      releaseConnection = false;
      continue;
    } catch (IOException e) {
      // An attempt to communicate with a server failed. The request may have been sent.
      boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
      if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
      releaseConnection = false;
      continue;
    } finally {
      // We're throwing an unchecked exception. Release any resources.
      if (releaseConnection) {
        streamAllocation.streamFailed(null);
        streamAllocation.release(true);
      }
    }
    // Attach the prior response if it exists. Such responses never have a body.
    if (priorResponse != null) {
      response = response.newBuilder()
          .priorResponse(priorResponse.newBuilder()
                  .body(null)
                  .build())
          .build();
    }
    Request followUp;
    try {
      followUp = followUpRequest(response, streamAllocation.route());
    } catch (IOException e) {
      streamAllocation.release(true);
      throw e;
    }
    if (followUp == null) {
      streamAllocation.release(true);
      return response;
    }
    closeQuietly(response.body());
    if (++followUpCount > MAX_FOLLOW_UPS) {
      streamAllocation.release(true);
      throw new ProtocolException("Too many follow-up requests: " + followUpCount);
    }
    if (followUp.body() instanceof UnrepeatableRequestBody) {
      streamAllocation.release(true);
      throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
    }
    if (!sameConnection(response, followUp.url())) {
      streamAllocation.release(false);
      streamAllocation = new StreamAllocation(client.connectionPool(),
          createAddress(followUp.url()), call, eventListener, callStackTrace);
      this.streamAllocation = streamAllocation;
    } else if (streamAllocation.codec() != null) {
      throw new IllegalStateException("Closing the body of " + response
          + " didn't close its backing stream. Bad interceptor?");
    }
    request = followUp;
    priorResponse = response;
  }
}
```

首先，在`intercept()`方法中创建了一个 StreamAllocation 对象，并在调用`proceed()`方法的时候将其传递给了下层的 Interceptor 。

对于异常的处理，先调用`recover(IOException, Request)`判断是否可以恢复请求，方法的内部大致判断了四个方面：

-   应用层是否拒绝重试
-   请求体不能二次使用
-   根据异常类型判断是否可恢复
-   是否有可用路由

然后就是判断是否需要继续请求，调用了`followUpRequest(Response, Route)`方法，这个方法内部判断了响应码，如鉴权失败、重定向等进行了判断，如果是重定向的话，就会重新返回一个新的 Request 以供下一次请求。

#### BridgeInterceptor

这个类完成了从应用层到网络层的联系，它根据一个 user request 构建出 network request ，然后再通过 network response 构建 user response 。代码上的表现就是给 Request 和 Response 加了一些 header ：

```java
@Override public Response intercept(Chain chain) throws IOException {
  Request userRequest = chain.request();
  Request.Builder requestBuilder = userRequest.newBuilder();
  RequestBody body = userRequest.body();
  if (body != null) {
    MediaType contentType = body.contentType();
    if (contentType != null) {
      requestBuilder.header("Content-Type", contentType.toString());
    }
    long contentLength = body.contentLength();
    if (contentLength != -1) {
      requestBuilder.header("Content-Length", Long.toString(contentLength));
      requestBuilder.removeHeader("Transfer-Encoding");
    } else {
      requestBuilder.header("Transfer-Encoding", "chunked");
      requestBuilder.removeHeader("Content-Length");
    }
  }
  if (userRequest.header("Host") == null) {
    requestBuilder.header("Host", hostHeader(userRequest.url(), false));
  }
  if (userRequest.header("Connection") == null) {
    requestBuilder.header("Connection", "Keep-Alive");
  }
  // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
  // the transfer stream.
  boolean transparentGzip = false;
  if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
    transparentGzip = true;
    requestBuilder.header("Accept-Encoding", "gzip");
  }
  List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
  if (!cookies.isEmpty()) {
    requestBuilder.header("Cookie", cookieHeader(cookies));
  }
  if (userRequest.header("User-Agent") == null) {
    requestBuilder.header("User-Agent", Version.userAgent());
  }
  Response networkResponse = chain.proceed(requestBuilder.build());
  HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());
  Response.Builder responseBuilder = networkResponse.newBuilder()
      .request(userRequest);
  if (transparentGzip
      && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
      && HttpHeaders.hasBody(networkResponse)) {
    GzipSource responseBody = new GzipSource(networkResponse.body().source());
    Headers strippedHeaders = networkResponse.headers().newBuilder()
        .removeAll("Content-Encoding")
        .removeAll("Content-Length")
        .build();
    responseBuilder.headers(strippedHeaders);
    String contentType = networkResponse.header("Content-Type");
    responseBuilder.body(new RealResponseBody(contentType, -1L, Okio.buffer(responseBody)));
  }
  return responseBuilder.build();
}
```

这里的应用层和网络层是如何区分的？了解过 HTTP 协议的知道，不论是 Request 还是 Response ，实质上都可以看作是一串字符，[详见相关介绍](../https-introduction)，简单来说就是配置信息如 Content-Type、Content-Length 等是以字符串的方式存储在请求头中的，这就是这里所说的 network request ，而在此之前是通过 Request 对象保存的，诸如 Content-Type 这些参数是以成员变量的方式存储在 RequestBody 中的，这就是所谓的 user request ，所以在 BridgeInterceptor 中完成的就是这样一个转变，还有一些对请求头的完善。

当这个类的工作完成之后，Request 的请求头参数趋于完善，就可以进行下一步操作了。

#### CacheInterceptor

看名字就可以知道，这是一个与缓存相关的类，它能够通过对获得的 Response 进行缓存，可以在某些情况下直接读取缓存而避免网络请求。完整代码如下：

```java
@Override public Response intercept(Chain chain) throws IOException {
  Response cacheCandidate = cache != null
      ? cache.get(chain.request())
      : null;
  long now = System.currentTimeMillis();
  CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
  Request networkRequest = strategy.networkRequest;
  Response cacheResponse = strategy.cacheResponse;
  if (cache != null) {
    cache.trackResponse(strategy);
  }
  if (cacheCandidate != null && cacheResponse == null) {
    closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
  }
  // If we're forbidden from using the network and the cache is insufficient, fail.
  if (networkRequest == null && cacheResponse == null) {
    return new Response.Builder()
        .request(chain.request())
        .protocol(Protocol.HTTP_1_1)
        .code(504)
        .message("Unsatisfiable Request (only-if-cached)")
        .body(Util.EMPTY_RESPONSE)
        .sentRequestAtMillis(-1L)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
  }
  // If we don't need the network, we're done.
  if (networkRequest == null) {
    return cacheResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .build();
  }
  Response networkResponse = null;
  try {
    networkResponse = chain.proceed(networkRequest);
  } finally {
    // If we're crashing on I/O or otherwise, don't leak the cache body.
    if (networkResponse == null && cacheCandidate != null) {
      closeQuietly(cacheCandidate.body());
    }
  }
  // If we have a cache response too, then we're doing a conditional get.
  if (cacheResponse != null) {
    if (networkResponse.code() == HTTP_NOT_MODIFIED) {
      Response response = cacheResponse.newBuilder()
          .headers(combine(cacheResponse.headers(), networkResponse.headers()))
          .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
          .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
          .cacheResponse(stripBody(cacheResponse))
          .networkResponse(stripBody(networkResponse))
          .build();
      networkResponse.body().close();
      // Update the cache after combining headers but before stripping the
      // Content-Encoding header (as performed by initContentStream()).
      cache.trackConditionalCacheHit();
      cache.update(cacheResponse, response);
      return response;
    } else {
      closeQuietly(cacheResponse.body());
    }
  }
  Response response = networkResponse.newBuilder()
      .cacheResponse(stripBody(cacheResponse))
      .networkResponse(stripBody(networkResponse))
      .build();
  if (cache != null) {
    if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
      // Offer this request to the cache.
      CacheRequest cacheRequest = cache.put(response);
      return cacheWritingResponse(cacheRequest, response);
    }
    if (HttpMethod.invalidatesCache(networkRequest.method())) {
      try {
        cache.remove(networkRequest);
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
    }
  }
  return response;
}
```

首先从 InternalCache 中取一个与 Request 相对应的缓存的 Response ，然后利用 Request 和 Response 生成一个 CacheStrategy 对象，具体步骤在`CacheStrategy.Factory.getCandidate()`方法中，判断是否有可用的 Response 和 Request ，并将筛选结果返回，然后再在`CacheInterceptor.intercept()`方法中根据结果判断是否需要进行网络请求，有四种情况：

-   networkRequest == null，表示当前不能进行网络请求
    -   cacheResponse == null，如果也没有缓存的 Response ，则返回一个失败响应
    -   cacheResponse != null，有缓存，则将缓存内容返回
-   networkRequest != null，表示需要进行网络请求
    -   cacheResponse == null，如果没有缓存，那么请求结果缓存并返回
    -   cacheResponse != null，如果有缓存，则需要将请求结果与缓存结果结合之后，更新缓存并返回

#### ConnectInterceptor

在这个方法中完成了建立连接的工作，先后创建了 RealConnection 和 HttpCodec 对象。

```java
@Override public Response intercept(Chain chain) throws IOException {
  RealInterceptorChain realChain = (RealInterceptorChain) chain;
  Request request = realChain.request();
  StreamAllocation streamAllocation = realChain.streamAllocation();
  // We need the network to satisfy this request. Possibly for validating a conditional GET.
  boolean doExtensiveHealthChecks = !request.method().equals("GET");
  HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
  RealConnection connection = streamAllocation.connection();
  return realChain.proceed(request, streamAllocation, httpCodec, connection);
}
```

由上面的代码可以看到，与之前的一些 Interceptor 不同，ConnectInterceptor 中没有对 Request 和 Response 的操作，它的作用就是用于创建客户端与服务端之间的连接，通过`StreamAllocation.newStream()`方法。这也就更好地理解了为什么 OkHttp 要将自定义 Interceptor 分为普通 Interceptor 和 networkInterceptor 并放在不同位置来执行了，普通的 Interceptor 更多的是对 Response 和 Request 的一些修饰性操作，所以放在了最外层执行，而 networkInterceptor 放在了 ConnectInterceptor 下层，此时主机之间的 HTTP 连接已经建立完成，所以此时的 networkInterceptor 更多的是做一些关于网络连接方面的操作。

下面再看一下`newStream()`方法：

```java
public HttpCodec newStream(
    OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
  int connectTimeout = chain.connectTimeoutMillis();
  int readTimeout = chain.readTimeoutMillis();
  int writeTimeout = chain.writeTimeoutMillis();
  int pingIntervalMillis = client.pingIntervalMillis();
  boolean connectionRetryEnabled = client.retryOnConnectionFailure();
  try {
    RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
        writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
    HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);
    synchronized (connectionPool) {
      codec = resultCodec;
      return resultCodec;
    }
  } catch (IOException e) {
    throw new RouteException(e);
  }
}
```

它调用`findHealthyConnection()`方法建立连接，并利用连接创建 HttpCodec 对象，将其返回。找到一个「健康」的连接，这个方法的注释为：

>   找到一个连接，如果它是健康的，将其返回，否则重复执行这个步骤。

所以在这个方法中有一个循环，两步操作：

```java
private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
    int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
    boolean doExtensiveHealthChecks) throws IOException {
  while (true) {
    RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
        pingIntervalMillis, connectionRetryEnabled);
    // If this is a brand new connection, we can skip the extensive health checks.
    synchronized (connectionPool) {
      if (candidate.successCount == 0) {
        return candidate;
      }
    }
    // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
    // isn't, take it out of the pool and start again.
    if (!candidate.isHealthy(doExtensiveHealthChecks)) {
      noNewStreams();
      continue;
    }
    return candidate;
  }
}
```

先调用`findConnection()`找到一个连接，然后利用`isHealthy()`判断它是否「健康」。

关于寻找连接，分为三个步骤：

1.  是否已经存在，是则取用，否则转2
2.  连接池中是否存在，是则取用，否则转3
3.  创建新连接

在连接创建的过程中，Okio 利用 Socket 获得了 BufferedSource 和 BufferedSink 对象，这些都作为 HttpCodec 的成员变量存在，这两个类分别实现自 Source 和 Sink ，用于数据流的读和写。

关于判断是否健康，在`isHealthy()`中判断了 Socket 是否关闭、Http 连接是否关闭等。

#### CallServerInterceptor

CallServerInterceptor 位于整个调用链的最底层，在上层 Interceptor 的充足准备下，CallServerInterceptor 发起网络请求，并得到服务端返回的结果，由于 BufferedSource 和 BufferedSink 都集成在 HttpCodec 中了，所以在 CallServerInterceptor 中网络数据的读写操作都是通过 HttpCodec 完成的。大致四个方法：

-   `writeRequestHeaders()`，写请求头
-   `createRequestBody()`，写请求体
-   `readResponseHeaders()`，读响应头
-   `openResponseBody()`，读响应体

其内部具体的实现就是结合 Okio、Source、Sink、Buffer 这些完成的，对数据流不了解不作过多解释。

## 要点

### 线程池的使用

在构建好了一个 RealCall 之后，可以使用同步或者异步的方式发起网络请求，分别对应着方法`execute()`和`enqueue(Callback)`，对于同步执行来说，会直接在当前线程调用`getResponseWithInterceptorChain()`方法执行一系列网络请求的操作。对于异步执行来说，Okhttp 使用到了线程池。`enqueue(Callback)`会调用`Dispatcher.enqueue(AsyncCall)`：

```java
void enqueue(AsyncCall call) {
  synchronized (this) {
    readyAsyncCalls.add(call);
    // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call t
    // the same host.
    if (!call.get().forWebSocket) {
      AsyncCall existingCall = findExistingCallWithHost(call.host());
      if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
    }
  }
  promoteAndExecute();
}
```

在这个方法中首先将当前的 Call 加入 readyAsyncCalls ，然后调用`promoteAndExecute()`方法，根据最大执行任务策略选择一部分 Call 并将其提交给线程池。

线程池的加入，使得 OkHttp 处理大量网络请求的时候也能游刃有余，同时也不会过多浪费系统资源。

### 责任链模式

借由拦截链的存在，OkHttp 在数据处理方面的流程十分简单，只是将不同的操作放在不同的拦截器里面，再通过拦截链将所有的拦截器连通起来：每一层拦截器的 Request 来自于上一层处理过的 Request ，每一层拦截器处理过的 Response 都会交给上一层拦截器继续处理。所以，在达到拦截器的最底层的时候，Request 所有的处理已经完善，将其用于网络请求，得到原始的 Response ，然后一层层回到最外层，Response 经过各层拦截器的处理也将变得完善。

并且 OkHttp 为开发者开放了两种拦截器的自定义，分别在最外层和建立连接之后，使得开发者需要实现某些不一样的功能的时候不需要更改 OkHttp 的源码就可以实现比较复杂的功能，对于有特殊要求的用户也是比较友好的。

## 总结

OkHttp 是 Android 上的一个网络请求框架，相对来说，OkHttp 这个框架对于网络请求这个功能的实现还是比较彻底的，从高层到底层都是由自己实现，不过得益于责任链模式的使用，整个请求过程十分清晰，每个阶段所做的事分给了不同的拦截器，同时也提供了不错的扩展能力。