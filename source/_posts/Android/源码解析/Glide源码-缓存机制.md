---
title: Glide源码-缓存机制
date: 2019-11-18 16:52:17
tags:
- glide
- 源码解析
categories:
- Android
---

## Glide的配置

在实例化Glide的时候, 对很多重要的部分做了初始化.

```java

// 起始入口
public DrawableTypeRequest<String> load(String string) {
    return (DrawableTypeRequest<String>) fromString().load(string);
}

... // 省略中间步骤
```

一直向下追溯, 可以找到Glide类的这个方法

```java
Glide createGlide() {
    if (sourceService == null) {
        // 初始化加载网络图片的线程池
        final int cores = Math.max(1, Runtime.getRuntime().availableProcessors());
        sourceService = new FifoPriorityThreadPoolExecutor(cores);
    }
    if (diskCacheService == null) {
        // 初始化加载磁盘缓存的线程池
        diskCacheService = new FifoPriorityThreadPoolExecutor(1);
    }

    // 初始化内存缓存池, 可以看到, 是一个lrucache
    MemorySizeCalculator calculator = new MemorySizeCalculator(context);
    if (bitmapPool == null) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
            int size = calculator.getBitmapPoolSize();
            bitmapPool = new LruBitmapPool(size);
        } else {
            bitmapPool = new BitmapPoolAdapter();
        }
    }

    if (memoryCache == null) {
        memoryCache = new LruResourceCache(calculator.getMemoryCacheSize());
    }

    if (diskCacheFactory == null) {
        diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }

    if (engine == null) {
        // 实例化Glide Engine
        engine = new Engine(memoryCache, diskCacheFactory, diskCacheService, sourceService);
    }

    if (decodeFormat == null) {
        decodeFormat = DecodeFormat.DEFAULT;
    }

    return new Glide(engine, memoryCache, bitmapPool, context, decodeFormat);
}
```

## 内存缓存

### Engine-load

```java
private final Map<Key, WeakReference<EngineResource<?>>> activeResources;

...

public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
        DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
        Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
    Util.assertMainThread();
    long startTime = LogTime.getLogTime();

    // 组装key
    final String id = fetcher.getId();
    EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
            loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
            transcoder, loadProvider.getSourceEncoder());

    // 先从内存cache中获取
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
        cb.onResourceReady(cached);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Loaded resource from cache", startTime, key);
        }
        return null;
    }

    // 从ActiveResources获取
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
        cb.onResourceReady(active);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Loaded resource from active resources", startTime, key);
        }
        return null;
    }

    // 后面的内容暂时和内存缓存无关

    EngineJob current = jobs.get(key);
    if (current != null) {
        // 当已经存在相同的图片加载job, 直接添加新的callback即可
        current.addCallback(cb);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Added to existing load", startTime, key);
        }
        return new LoadStatus(cb, current);
    }

    EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
    DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
            transcoder, diskCacheProvider, diskCacheStrategy, priority);
    // 具体的图片请求工作在这个runnable中
    EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
    jobs.put(key, engineJob);
    engineJob.addCallback(cb);
    engineJob.start(runnable);
    
    return new LoadStatus(cb, engineJob);
}
```

可以出来一个策略就是先从内存cache中获取, 然后从activeResource中获取. 
ActivityResource是一个Map, 用弱引用存储正在使用的图片资源. 关于这个集合:
* 为什么需要这样一个集合, 而不是把所有的图片资源都加入到cache中:
使用activeResources来缓存正在使用中的图片，可以保护这些图片不会被LruCache算法回收掉。
* 存储的正在使用的图片资源的**弱引用**: 官方的解释是对于ActiveResources中的资源的释放并没有严格约束, 为了防止内存泄漏, 使用弱引用

来看看对lrucache和ActiveResources的操作

### 内存cache的获取

```java
// 从cache中获取
private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    // 检查内存缓存是否被禁用
    if (!isMemoryCacheable) {
        return null;
    }

    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
        // cache命中, 从lrucache中取出后放入ActiveResources
        cached.acquire();
        activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
    }
    return cached;
}
// 从cache中获取资源
private EngineResource<?> getEngineResourceFromCache(Key key) {
    // 注意这里是直接remove的
    Resource<?> cached = cache.remove(key);

    final EngineResource result;
    if (cached == null) {
        result = null;
    } else if (cached instanceof EngineResource) {
        // Save an object allocation if we've cached an EngineResource (the typical case).
        result = (EngineResource) cached;
    } else {
        result = new EngineResource(cached, true /*isCacheable*/);
    }
    return result;
}

private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
    // 检查内存cache是否被禁用
    if (!isMemoryCacheable) {
        return null;
    }

    EngineResource<?> active = null;
    WeakReference<EngineResource<?>> activeRef = activeResources.get(key);
    if (activeRef != null) {
        active = activeRef.get();
        if (active != null) {
            // ActiveResource命中
            active.acquire();
        } else {
            // 弱引用的对象被释放了, 直接remove
            activeResources.remove(key);
        }
    }

    return active;
}
```

### 内存cache的释放

先来看看resource中如何表示图片需要被释放
```java
class EngineResource<Z> implements Resource<Z> {
    private int acquired;
    ...

    void acquire() {
        ... // 省略异常处理
        ++acquired;
    }

    void release() {
        ... // 省略异常处理
        if (--acquired == 0) {
            // 当acquired为0的使用, 表示当前图片没有被引用
            listener.onResourceReleased(key, this);
        }
    }
}
```

resource中listener的回调是在EngineJob类中实现的

```java
// 图片的引用为0, 图片没有在使用了, 所以从ActiveResource中移除, 添加进lrucache
@Override
public void onResourceReleased(Key cacheKey, EngineResource resource) {
    Util.assertMainThread();
    activeResources.remove(cacheKey);
    if (resource.isCacheable()) {
        cache.put(cacheKey, resource);
    } else {
        resourceRecycler.recycle(resource);
    }
}
```

## 磁盘缓存

首先要明确一点, 就是磁盘缓存不止一种. 在配置Glide的磁盘缓存的时候, 可以配置单独缓存原始图片, 还是单独缓存转化后的图片. 所以Glide的内部, 肯定对原始图片的缓存 和 转化后图片的缓存都做了处理.

### 入口

内存cache的工作在Engine的load方法中已经做完了. 磁盘cache在Runnable的decode()方法中

```java
private Resource<?> decode() throws Exception {
    if (isDecodingFromCache()) {
        return decodeFromCache();
    } else {
        return decodeFromSource();
    }
}

// 从磁盘缓存中获取图片, 先尝试获取处理后的图片, 再尝试获取原始图片
private Resource<?> decodeFromCache() throws Exception {
    Resource<?> result = null;
    try {
        result = decodeJob.decodeResultFromCache();
    } catch (Exception e) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
            Log.d(TAG, "Exception decoding result from cache: " + e);
        }
    }

    if (result == null) {
        result = decodeJob.decodeSourceFromCache();
    }
    return result;
    }
```

### 磁盘cache读取

```java
public Resource<Z> decodeResultFromCache() throws Exception {
    if (!diskCacheStrategy.cacheResult()) {
        return null;
    }
    long startTime = LogTime.getLogTime();
    Resource<T> transformed = loadFromCache(resultKey);
    startTime = LogTime.getLogTime();
    Resource<Z> result = transcode(transformed);
    return result;
}

public Resource<Z> decodeSourceFromCache() throws Exception {
    if (!diskCacheStrategy.cacheSource()) {
        return null;
    }
    long startTime = LogTime.getLogTime();
    Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
    return transformEncodeAndTranscode(decoded);
}

private Resource<T> loadFromCache(Key key) throws IOException {
    File cacheFile = diskCacheProvider.getDiskCache().get(key);
    if (cacheFile == null) {
        return null;
    }
    Resource<T> result = null;
    try {
        result = loadProvider.getCacheDecoder().decode(cacheFile, width, height);
    } finally {
        if (result == null) {
            diskCacheProvider.getDiskCache().delete(key);
        }
    }
    return result;
}
```

磁盘缓存读取这里挺好懂的

### 磁盘cache写入

#### 缓存原始图片

直接贴调用过程吧

EngineRunnable.decodeFromSource()
-> EngineRunnable.decodeFromSource()
-> DecodeJob.decodeFromSource()
-> DecodeJob.decodeSource()
-> DecodeJob.decodeFromSourceData()
-> DecodeJob.cacheAndDecodeSourceData

```java
private Resource<T> decodeSource() throws Exception {
    Resource<T> decoded = null;
    try {
        long startTime = LogTime.getLogTime();
        // 从网络获取图片
        final A data = fetcher.loadData(priority);
        if (isCancelled) {
            return null;
        }
        decoded = decodeFromSourceData(data);
    } finally {
        fetcher.cleanup();
    }
    return decoded;
}

private Resource<T> cacheAndDecodeSourceData(A data) throws IOException {
    long startTime = LogTime.getLogTime();
    SourceWriter<A> writer = new SourceWriter<A>(loadProvider.getSourceEncoder(), data);
    diskCacheProvider.getDiskCache().put(resultKey.getOriginalKey(), writer);

    startTime = LogTime.getLogTime();
    // ?? 才刚放进去之后又拿出来? 为什么不直接用放进去的那个?
    Resource<T> result = loadFromCache(resultKey.getOriginalKey());
    return result;
}
```

#### 缓存处理过后的图片

EngineRunnable.decodeFromSource()
-> EngineRunnable.decodeFromSource()
-> DecodeJob.decodeFromSource()
-> DecodeJob.transformEncodeAndTranscode
-> DecodeJob.writeTransformedToCache

```java
private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
    long startTime = LogTime.getLogTime();
    // 先转换图片
    Resource<T> transformed = transform(decoded);
    // 写入磁盘
    writeTransformedToCache(transformed);

    startTime = LogTime.getLogTime();
    // 转码
    Resource<Z> result = transcode(transformed);

    return result;
}

private void writeTransformedToCache(Resource<T> transformed) {
    if (transformed == null || !diskCacheStrategy.cacheResult()) {
        return;
    }
    long startTime = LogTime.getLogTime();
    SourceWriter<Resource<T>> writer = new SourceWriter<Resource<T>>(loadProvider.getEncoder(), transformed);
    diskCacheProvider.getDiskCache().put(resultKey, writer);
}
```
