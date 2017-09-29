## 核心的类

1. **Request**对请求的封装，实现了Comparable接口，便于优先队列根据请求的优先级进行排序。还承担一定的解析和分发响应的任务。
2. **NetworkResponse**对响应的封装，包括返回的状态码、请求头、数据**Response**比**NetworkResponse**简单**，**封装了响应的结果、缓存、异常以及响应分发回调接口。
3. **Network**处理网络请求的接口，只有一个方法：NetworkResponse performRequest(Request<?> request);接受Request对象返回NetworkResponse对象。**BasicNetwork**是它的实现类。
4. **RequestQueue**包含几个队列：**Set<Request<?>> mCurrentRequests** ：正在被RequestQueue 处理的请求的集合，所有被add()添加到RequestQueue中的请求都会放到这。在finish()中将处理完成的Request剔除，在cancelAll中遍历集合，调用特定的Request的cancel()方法。
5. **PriorityBlockingQueue<Request<?>> mNetworkQueue  ** 走得网络访问的的请求，根据优先级来安排处理次序。
6. **PriorityBlockingQueue<Request<?>> mCacheQueue** 用缓存处理的请求队列
7. **Map<String, Queue<Request<?>>> mWaitingRequests**  一个key对应一个队列（同一个队列存同一种请求），给那些在前面已经有一个相同的请求在处理的请求提供待命区域（staging area），等前面这个Request处理完后再添加到缓存队列，那么这些Request肯定能用Cache处理了。这是减小访问网络次数的一种优化。
8. **NetworkDispatcher[] mDispatchers**  每个NetworkDispatcher实际是一个线程，在run()里面从mNetworkQueue中取出Request来处理。一个RequestQueue里面默                           认有4个这样的网络请求处理线程。
9. **CacheDispatcher mCacheDispatcher** 缓存处理线程，从mCacheQueue中取出Request处理。一个RequestQueue只有一个缓存处理线程。
10. **ResponseDelivery** 分发响应的结果或异常到主线程的接口，**ExecutorDelivery**内部用Handler传递数据，在主线程中执行了：mRequest.deliverResponse(mResponse.result);

## 核心流程

### 静态工厂方法创建RequestQueue

```java
mRequestQueue = Volley.newRequestQueue(this);
-----
public static RequestQueue newRequestQueue(Context context, HttpStack stack) {
    File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);
    String userAgent = "volley/0";
    try {
        String packageName = context.getPackageName();
        PackageInfo info = context.getPackageManager().getPackageInfo(packageName, 0);
        userAgent = packageName + "/" + info.versionCode;
    } catch (NameNotFoundException e) {
    }
    if (stack == null) {
        if (Build.VERSION.SDK_INT >= 9) {
            stack = new HurlStack();
        } else {
            // Prior to Gingerbread, HttpUrlConnection was unreliable.
            // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
            stack = new HttpClientStack(AndroidHttpClient.newInstance(userAgent));
        }
    }

    Network network = new BasicNetwork(stack);
    RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
    queue.start();
    return queue;
}
```

### 启动并初始化RequestQueue

```java
queue.start();
----
public void start() {
    stop();  // Make sure any currently running dispatchers are stopped.
    // Create the cache dispatcher and start it. 创建缓存处理线程
    mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
    mCacheDispatcher.start();

    // Create network dispatchers (and corresponding threads) up to the pool size.
     //创建并启动网络请求处理线程，默认是开辟4个处理线程
    for (int i = 0; i < mDispatchers.length; i++) {
        NetworkDispatcher networkDispatcher = new NetworkDispatcher(mNetworkQueue, mNetwork,
                mCache, mDelivery);
        mDispatchers[i] = networkDispatcher;
        networkDispatcher.start();
    }
}
```

### CacheDispatcher 缓存工作处理线程

```java
@Override
  public void run() {
    //初始化Cache
    mCache.initialize();
    Request<?> request;
    while (true) {
            //阻塞  获取一个Cache任务
            request = mCacheQueue.take();
        try {
            //已经被取消
            if (request.isCanceled()) {
                request.finish("cache-discard-canceled");
                continue;
            }
            //如果拿cache未果，放入网络请求队列
            Cache.Entry entry = mCache.get(request.getCacheKey());
            if (entry == null) {
                request.addMarker("cache-miss");
                mNetworkQueue.put(request);
                continue;
            }
            //缓存超时，放入网络请求队列
            if (entry.isExpired()) {
                request.addMarker("cache-hit-expired");
                request.setCacheEntry(entry);
                mNetworkQueue.put(request);
                continue;
            }
            //根据Cache构造Response
            Response<?> response = request.parseNetworkResponse(
                    new NetworkResponse(entry.data, entry.responseHeaders));
            //是否超过软过期（需要刷新，但真正没过期）
            if (!entry.refreshNeeded()) {
                // 直接返回Cache即可
                mDelivery.postResponse(request, response);
            } else {
                request.setCacheEntry(entry);
                //设置缓存数据作为中间结果
                response.intermediate = true;
                //发送缓存数据作为中间结果
                final Request<?> finalRequest = request;
                mDelivery.postResponse(request, response, new Runnable() {
                    @Override
                    public void run() {
                        try {
                            //将请求放入网络队列
                            mNetworkQueue.put(finalRequest);
                        } catch (InterruptedException e) {
                            // Not much we can do about this.
                        }
                    }
                });
            }
        } catch (Exception e) {
        }
    }
}
```

### NetworkDispatcher网络处理线程

```java
public void run() {
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    Request<?> request;
    while (true) {
        long startTimeMs = SystemClock.elapsedRealtime();
        // release previous request object to avoid leaking request object when mQueue is drained.
        request = null;
        try {
            request = mQueue.take();
        } catch (InterruptedException e) {
            if (mQuit) {
                return;
            }
            continue;
        }

        try {
            request.addMarker("network-queue-take");
            //请求已取消
            if (request.isCanceled()) {
                request.finish("network-discard-cancelled");
                continue;
            }
            //通过Http栈实现客户端发送网络请求
            NetworkResponse networkResponse = mNetwork.performRequest(request);
            request.addMarker("network-http-complete");

            // 如果缓存软过期，那么会重新走网络；如果server返回304，表示上次之后请求结果数据本地并没有过期，所以可以直接用本地的，因为之前Volley已经发过一次Response了，所以这里就不需要再发送Response结果了。
            if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                request.finish("not-modified");
                continue;
            }
               //解析响应
            Response<?> response = request.parseNetworkResponse(networkResponse);
            request.addMarker("network-parse-complete");
            //更新缓存
            if (request.shouldCache() && response.cacheEntry != null) {
                mCache.put(request.getCacheKey(), response.cacheEntry);
                request.addMarker("network-cache-written");
            }
            //发送响应
            request.markDelivered();
            mDelivery.postResponse(request, response);
        } catch (VolleyError volleyError) {
            volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
            parseAndDeliverNetworkError(request, volleyError);
        } catch (Exception e) {
            VolleyLog.e(e, "Unhandled exception %s", e.toString());
            VolleyError volleyError = new VolleyError(e);
            volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
            mDelivery.postError(request, volleyError);
        }
    }
}

```

### 添加请求到队列

```java
mQueue.add(request);
-----
public <T> Request<T> add(Request<T> request) {
    // Tag the request as belonging to this queue and add it to the set of current requests.
    request.setRequestQueue(this);
    synchronized (mCurrentRequests) {
        mCurrentRequests.add(request);
    }

    // Process requests in the order they are added.
    request.setSequence(getSequenceNumber());
    request.addMarker("add-to-queue");

    // If the request is uncacheable, skip the cache queue and go straight to the network.
     //如果请求不要缓存，直接添加到网络请求队列
    if (!request.shouldCache()) {
        mNetworkQueue.add(request);
        return request;
    }

    // Insert request into stage if there's already a request with the same cache key in flight.
    synchronized (mWaitingRequests) {
        String cacheKey = request.getCacheKey();
          //如果已经有一个相同的请求正在处理，待命！
        if (mWaitingRequests.containsKey(cacheKey)) {
            // There is already a request in flight. Queue up.
            Queue<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
            if (stagedRequests == null) {
                stagedRequests = new LinkedList<Request<?>>();
            }
            stagedRequests.add(request);
            mWaitingRequests.put(cacheKey, stagedRequests);
            if (VolleyLog.DEBUG) {
                VolleyLog.v("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
            }
        } else {
            // Insert 'null' queue for this cacheKey, indicating there is now a request in
            // flight.
            mWaitingRequests.put(cacheKey, null);
            mCacheQueue.add(request);
        }
        return request;
    }
}
```

## 总结

首先，Volley有三个核心类，一个是Request，封装了网络请求的参数，请求头，URL等信息，也负责一定的解析和分发网络响应的职能；一个是NetWorkResponse，封装了网络响应的相关参数。还有就是RequestQueue了。RequestQueue提供一个静态工厂方法newRequestQueue，新建缓存路径和实例化处理网络请求的对象，然后调用start方法启动队列。RequestQueue里有两个4个数据结构，分别是mCurrentRequests，是一个存放正在被处理的集合；mWaitingRequests，是一个存放正在等待的Requests的队列映射表；CacheQueue是缓存处理队列；NetWorkRequestQueue是网络请求队列。当调用RequestQueue的add方法添加一个Request对象时，首先把它添加到mCurrentRequests；接着判断是否需要缓存，如果不需要缓存直接添加到网络请求队列中；接着判断如果mWaitingRequests已经有相同的Request对对象正在处理，把request对象append到对应request队列的队尾，否则呢，将这个请求添加到缓存处理队列当中。当然，RequestQueue还有一个CacheDispatcher对象，本质上是一个缓存处理线程。从缓存处理队列当中取出request对象，检查如果缓存未过期，那么直接将缓存数据解析成响应对象，分发给调用线程；如果缓存过期了，那么将request对象添加到网络请求队列当中；此外，RequestQueue还有一个NetWorkDispatcher对象数组，本质上就是网络处理线程，负责从网络处理队列中取request对象，进行真正的网络请求，得到响应后调用Request的parseNetworkResponse，然后将解析好的Response对象通过ResponseDelivery对象分发到调用线程。