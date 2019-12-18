---
title: Glide源码-主要流程
date: 2019-11-18 02:00:05
tags:
- glide
- 源码解析
categories:
- Android
---

## API调用

```java
Glide.with(fragment)
    .load(myUrl)
    .into(imageView);
```

## with过程

### 主要工作

将图片加载和对应的生命周期绑定(如activity, fragment等)
绑定生命周期的优点:
* 在activity, fragment等销毁的时候, 停止对应的图片加载. 
* 避免消耗资源
* 防止空指针问题的出现

### Glide
```java
    public static RequestManager with(Context context) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }

    public static RequestManager with(Activity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }
    
    public static RequestManager with(FragmentActivity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }
    
    public static RequestManager with(android.app.Fragment fragment) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(fragment);
    }
    
    public static RequestManager with(Fragment fragment) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(fragment);
    }
```

### RequestManagerRetriever

前面说了要监控生命周期的变化, 那么Glide如何监控activity和fragment的生命周期呢? 做法是:
默认添加一个自定义的fragment, 因为fragment和activity或者父fragment的生命周期是绑定的, 所以只要监控Glide自定义的那个fragment就可以了.

```java
public RequestManager get(Context context) {
    if (context == null) {
        throw new IllegalArgumentException("You cannot start a load on a null Context");
    } else if (Util.isOnMainThread() && !(context instanceof Application)) {
        if (context instanceof FragmentActivity) {
            return get((FragmentActivity) context);
        } else if (context instanceof Activity) {
            return get((Activity) context);
        } else if (context instanceof ContextWrapper) {
            return get(((ContextWrapper) context).getBaseContext());
        }
    }

    return getApplicationManager(context);
}

// 若调用的get方法, 传入的参数是一个ApplicationContext, 则将图片加载和整个APP的生命周期绑定
private RequestManager getApplicationManager(Context context) {
    // Either an application context or we're on a background thread.
    if (applicationManager == null) {
        synchronized (this) {
            if (applicationManager == null) {
                // Normally pause/resume is taken care of by the fragment we add to the fragment or activity.
                // However, in this case since the manager attached to the application will not receive lifecycle
                // events, we must force the manager to start resumed using ApplicationLifecycle.

                // 双校验锁 懒汉单例
                // 将资源加载与整个APP的生命周期绑定
                applicationManager = new RequestManager(context.getApplicationContext(),
                        new ApplicationLifecycle(), new EmptyRequestManagerTreeNode());
            }
        }
    }

    return applicationManager;
}

// 获取RequestManager
public RequestManager get(Activity activity) {
    if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB) {
        return get(activity.getApplicationContext());
    } else {
        assertNotDestroyed(activity);
        android.app.FragmentManager fm = activity.getFragmentManager();
        return fragmentGet(activity, fm);
    }
}

.... // 省略其他的重载方法

// 添加监控生命周期的fragment, 并将RequestManager创建并和fragment绑定
RequestManager fragmentGet(Context context, android.app.FragmentManager fm) {
    RequestManagerFragment current = getRequestManagerFragment(fm);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
        requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
        current.setRequestManager(requestManager);
    }
    return requestManager;
}

// 创建并添加RequestManagerFragment
RequestManagerFragment getRequestManagerFragment(final android.app.FragmentManager fm) {
    RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
    if (current == null) {
        // pendingRequestManagerFragments是fragment的一个map集合, 没懂用来干什么的
        current = pendingRequestManagerFragments.get(fm);
        if (current == null) {
            // 使用FragmentManager添加fragment
            current = new RequestManagerFragment();
            pendingRequestManagerFragments.put(fm, current);
            fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
            handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
        }
    }
    return current;
}

```

## load过程

### RequestManager-load

```java
public DrawableTypeRequest<String> load(String string) {
    return (DrawableTypeRequest<String>) fromString().load(string);
}
```

### GenericRequestBuilder

RequestManager的load方法跟踪之后来到了GenericRequestBuilder的load方法. 
跳过中间的过程, GenericRequestBuilder中有placeHolder, centerCrop的Glide加载的配置方法, 知道就好了

```java    
// ModelType是一个泛型通配符, model是加载的参数
public GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> load(ModelType model) {
    this.model = model;
    isModelSet = true;
    return this;
}

```

## into过程

### GenericRequestBuilder-into

```java
public Target<TranscodeType> into(ImageView view) {
    Util.assertMainThread();
    if (view == null) {
        throw new IllegalArgumentException("You must pass in a non null View");
    }

    // ...省略代码

    return into(glide.buildImageViewTarget(view, transcodeClass));
}

public <Y extends Target<TranscodeType>> Y into(Y target) {
    Util.assertMainThread();
    
    ... // 省略代码

    Request previous = target.getRequest();

    // 清楚target上旧的图片请求
    if (previous != null) {
        previous.clear();
        requestTracker.removeRequest(previous);
        previous.recycle();
    }

    Request request = buildRequest(target);
    target.setRequest(request);
    lifecycle.addListener(target);
    // 开始请求图片
    requestTracker.runRequest(request);

    return target;
}
```

**为什么要清楚target上旧的图片请求:**

由于ListView中Item的复用机制，会导致网络图片加载的错位或者闪烁。那我们解决这个问题的办法也很简单，就是给当前的ImageView设置tag，这个tag可以是图片的URL等等。当从网络中获取到图片时判断这个ImageVIew中的tag是否是这个图片的URL，如果是就加载图片，如果不是则跳过。

在有了Glide之后，我们处理ListView或者Recyclerview中的图片加载就很无脑了，根本不需要作任何多余的操作，直接正常使用就行了。这其中的原理是Glide给我们处理了这些判断. 判断的方法和我们平常的思路相同, 同样是添加tag, Glide中给view添加的tag是Request对象

### GenericRequestBuilder-runRequest

```java
public void runRequest(Request request) {
    requests.add(request);
    if (!isPaused) {
        // 请求加载没有暂停, 则开始请求
        request.begin();
    } else {
        // 请求加载暂停了, 则放入到等待队列中去
        pendingRequests.add(request);
    }
}
```

```java
@Override
public void begin() {
    startTime = LogTime.getLogTime();
    if (model == null) {
        onException(null);
        return;
    }

    status = Status.WAITING_FOR_SIZE;
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        // 如果用户在调用API时配置了图片的大小, 直接下一步
        // onSizeReady这个方法很重要
        onSizeReady(overrideWidth, overrideHeight);
    } else {
        // 如果没有, 则获取ImageView的大小
        // 这个方法最终也会走到onSizeReady方法
        target.getSize(this);
    }

    if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
        target.onLoadStarted(getPlaceholderDrawable());
    }
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    }
}
```

```java
@Override
public void onSizeReady(int width, int height) {
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
    if (status != Status.WAITING_FOR_SIZE) {
        return;
    }
    status = Status.RUNNING;

    width = Math.round(sizeMultiplier * width);
    height = Math.round(sizeMultiplier * height);

    ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
    final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);

    if (dataFetcher == null) {
        onException(new Exception("Failed to load model: \'" + model + "\'"));
        return;
    }
    ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
    }
    loadedFromMemoryCache = true;
    // 前面的代码都不知道在讲啥, 反正这个engine.load是重点
    loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
            priority, isMemoryCacheable, diskCacheStrategy, this);
    loadedFromMemoryCache = resource != null;
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
}
```

### Engine-load

获取资源图片, 这里要考虑到缓存, 具体的策略之后再说吧. 

```java
public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
        DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
        Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
    Util.assertMainThread();
    long startTime = LogTime.getLogTime();

    final String id = fetcher.getId();
    EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
            loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
            transcoder, loadProvider.getSourceEncoder());

    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
        cb.onResourceReady(cached);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Loaded resource from cache", startTime, key);
        }
        return null;
    }

    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
        cb.onResourceReady(active);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Loaded resource from active resources", startTime, key);
        }
        return null;
    }

    EngineJob current = jobs.get(key);
    if (current != null) {
        current.addCallback(cb);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Added to existing load", startTime, key);
        }
        return new LoadStatus(cb, current);
    }

    // 前面的缓存策略, 只有再说

    EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
    DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
            transcoder, diskCacheProvider, diskCacheStrategy, priority);
    EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
    jobs.put(key, engineJob);
    engineJob.addCallback(cb);
    engineJob.start(runnable);

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
}
```

### EngineRunnable

获取网络图片, 并编解码

```java
@Override
public void run() {
    if (isCancelled) {
        return;
    }

    Exception exception = null;
    Resource<?> resource = null;
    try {
        // decode()方法是重点
        // decode()中会有 访问网络, 编码, 解码, 根据缓存策略决定是否缓存等
        // 这里就不分析了, 太麻烦
        resource = decode();
    } catch (OutOfMemoryError e) {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Out Of Memory Error decoding", e);
        }
        exception = new ErrorWrappingGlideException(e);
    } catch (Exception e) {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Exception decoding", e);
        }
        exception = e;
    }

    if (isCancelled) {
        if (resource != null) {
            resource.recycle();
        }
        return;
    }

    if (resource == null) {
        onLoadFailed(exception);
    } else {
        onLoadComplete(resource);
    }
}

private void onLoadComplete(Resource resource) {
    // manager是一个EngineJob对象
    manager.onResourceReady(resource);
}
```

### EngineJob

```java
@Override
public void onResourceReady(final Resource<?> resource) {
    this.resource = resource;
    // 这个handler最终会执行到handleResultOnMainThread方法
    MAIN_THREAD_HANDLER.obtainMessage(MSG_COMPLETE, this).sendToTarget();
}

private void handleResultOnMainThread() {
    if (isCancelled) {
        resource.recycle();
        return;
    } else if (cbs.isEmpty()) {
        throw new IllegalStateException("Received a resource without any callbacks to notify");
    }
    engineResource = engineResourceFactory.build(resource, isCacheable);
    hasResource = true;

    // Hold on to resource for duration of request so we don't recycle it in the middle of notifying if it
    // synchronously released by one of the callbacks.
    engineResource.acquire();
    listener.onEngineJobComplete(key, engineResource);

    for (ResourceCallback cb : cbs) {
        if (!isInIgnoredCallbacks(cb)) {
            engineResource.acquire();
            cb.onResourceReady(engineResource);
        }
    }
    // Our request is complete, so we can release the resource.
    engineResource.release();
}
```

就是简单的通知每个调用每个callback.onResourceReady, 后面的逻辑都很简单了




