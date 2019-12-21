---
title: 'ViewModel,LiveData和Lifecycle'
date: 2019-12-21 20:19:41
tags:
---

# ViewModel

ViewModel通常在Activity、Fragment 的生命周期内被创建并且与之关联在一起直到Activity 被 finish 掉。(也就是说使用ViewModel并不会像MVP模式的Presenter一样会内存泄漏的风险)**当 Activity、Fragment 被重新创建的时候，通过 ViewModelProvides 可以重新获得与之关联的 ViewModel，从而确保数据不会丢失。**

### 从Activity和Fragment中获取ViewModel实例:

```java
// 有可能是创建ViewModel有可能是重新连接已有的ViewModel
XxxxViewModel XxxxViewModel = ViewModelProviders.of(this).get(XxxxViewModel.class);
```

### ViewModelProviders: (不同于ViewModelProvider)
```java
@NonNull
@MainThread
public static ViewModelProvider of(@NonNull FragmentActivity activity,
        @Nullable Factory factory) {
    Application application = checkApplication(activity);
    if (factory == null) {
        factory = ViewModelProvider.AndroidViewModelFactory.getInstance(application);
    }
    return new ViewModelProvider(ViewModelStores.of(activity), factory);
}
```

### ViewModelStores: (不同于ViewModelStore)

**ViewModelStores.of:**
```java
@NonNull
@MainThread
public static ViewModelStore of(@NonNull FragmentActivity activity) {
    if (activity instanceof ViewModelStoreOwner) {
        // ViewModelStoreOwner是一个接口，只有一个方法，在 27.1.0 的 Fragment和Activity 已经实现了该接口
        return ((ViewModelStoreOwner) activity).getViewModelStore();
    }
    // 兼容27.1.0以下版本的 support 库. 
    return holderFragmentFor(activity).getViewModelStore(); 
}
```

```java
// 27.1.0 以下的版本 google 则通过创建一个不可见的实现 ViewModelStoreOwner接口的 fragment 去做兼容
HolderFragment holderFragmentFor(FragmentActivity activity) {
    FragmentManager fm = activity.getSupportFragmentManager();
    HolderFragment holder = findHolderFragment(fm);
    if (holder != null) {
        return holder;
    }
    holder = mNotCommittedActivityHolders.get(activity);
    if (holder != null) {
        return holder;
    }

    if (!mActivityCallbacksIsAdded) {
        mActivityCallbacksIsAdded = true;
        activity.getApplication().registerActivityLifecycleCallbacks(mActivityCallbacks); //注册生命周期，activity 销毁时将 HolderFragment 从 mNotCommittedActivityHolders 移除
    }
    holder = createHolderFragment(fm);
    mNotCommittedActivityHolders.put(activity, holder);
    return holder;
}
```

### ViewModelProvider

**ViewModelProvider.get:**
```java
@MainThread
public <T extends ViewModel> T get(@NonNull String key, @NonNull Class<T> modelClass) {
    ViewModel viewModel = mViewModelStore.get(key);
    // 可以看出ViewModelStore是存储ViewModel的地方

    if (modelClass.isInstance(viewModel)) {
        //noinspection unchecked
        return (T) viewModel;
    } else {
        //noinspection StatementWithEmptyBody
        if (viewModel != null) {
            // TODO: log a warning.
        }
    }

    viewModel = mFactory.create(modelClass);
    mViewModelStore.put(key, viewModel);
    //noinspection unchecked
    return (T) viewModel;
}
```

### ViewModel的销毁

**FragmentActivity:(API27+, 实现了ViewModelStoreOwner接口)**
```java
protected void onDestroy() {
    super.onDestroy();
    // isChangingConfigurations()判断应用是否处于参数改变的情况, 比如横竖屏切换
    // 所以横竖屏切换并不会清楚掉ViewModelStore中的ViewModel
    if (this.mViewModelStore != null && !this.isChangingConfigurations()) {
        this.mViewModelStore.clear();
    }

    this.mFragments.dispatchDestroy();
}
```

**HolderFragment: (API27以下, 通过添加HolderFragment)**
```java
@Override
public void onDestroy() {
    super.onDestroy();
    mViewModelStore.clear();
}
```

### 在Fragment间共享数据(共享ViewModel)
```java
ViewModel viewModel = ViewModelProviders.of(getActivity()).get(UserViewModel::class.java)
```

### 总结: ViewModel的存储

* ViewModel存储在ViewModelStore中. Activity和Fragment会持有ViewModelStore实例, 存储ViewModel集合(MVVM中一个View中可能对应多个ViewModel)
* 在Ativity和Framgent销毁重建时, 会通过某种方式保存ViewModelStore(还不清楚时怎么保存的...)
* ViewModelProviders, 从Fragment和Activity中获取ViewModelStore对象, 并new实例化出一个ViewModelProvider对象

## Lifecycle

### Lifecycle的STATE和Activity生命周期的对应关系

```java
public enum State
{       
    DESTROYED,
    INITIALIZED,
    CREATED,
    STARTED,
    RESUMED;
}
```
* DESTROYED，在组件的onDestroy调用前，会变成该状态，变成此状态后将不会再出现任何状态改变，也不会发送任何生命周期事件
* INITIALIZED，构造函数执行完成后但onCreate未执行时为此状态，是最开始时的状态
* CREATED，在onCreate调用之后，以及onStop调用前会变成此状态
* STARTED，在onStart调用之后，以及onPause调用前会变成此状态
* RESUMED，再onResume调用之后会变成此状态

### LifecycleOwner接口
LifecycleOwner是Lifecycle组件包中的一个接口，所有需要管理生命周期的类型都必须实现这个接口。

只要继承这些类型，可以轻松的通过LifecycleOwner#getLifecycle()获取到Lifecycle实例.这是一种解耦实现,LifecycleOwner不包含任何有关生命周期管理的逻辑,实际的逻辑都在Lifecycle实例中,我们可以通过传递Lifecycle实例而非LifecycleOwner来防止内存泄漏.

需要注意的是**并不是所有的google官方的Actiivyt和Fragment类都实现了LifecycleOwner接口, 实现了这个接口有v4包下的Fragment, FragmentActivity和AppCompatActivity**

### SupportActivity类中对Lifecycle的使用
```java
// 这个类继承自Lifecycle类
private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
```
```java
// SupportActiity类中只有onSaveInstanceState调用了Lifecycle的相关方法
protected void onSaveInstanceState(Bundle outState) {
    this.mLifecycleRegistry.markState(State.CREATED);
    super.onSaveInstanceState(outState);
}
```
我们在查看源码时发现 Lifecycle 只在 Activity 的 onSaveInstanceState 中 调用了 mLifecycleRegistry.markState(Lifecycle.State.CREATED) 方法，在其他生命周期中并没有 Lifecycle 的相关代码，那 support 库中是怎么做到 disptch event 的呢？

### ReportFragment

在SupportActivity的onCreate方法中有一行代码, 它创建了一个不可见的 fragment 去分发 event
```java
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
}
```

```java
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    dispatchCreate(mProcessListener);
    dispatch(Lifecycle.Event.ON_CREATE);
}

private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }

    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

## LiveData
提供一个Observer接口给客户端，客户端实现这个接口，并将引用传递给它，当我们调用 LiveData 的 setValue 或 postValue 的时候，LiveData 就会去回调这个接口的方法，其实也就是LiveData 自己实现了观察者模式。不过 LiveData 搞定了一个让我们长期以来头疼的事情，那就是当 Activity、Fragment 不可见的时候 UI 不要更新的问题，其内部其实也是监听了 LifecycleOwner 的可见和不可见的方法，**当 Activity、Fragment 不可见的时候就不回调 Observer 的方法，当 Activity、Fragment 变成可见的时候，就对比最新的值和最后发出去的值是否是同一个，不同的话就发送最新的值，确保 Activity、Fragment 可见的时候用户看到的一定是最新的数据。**


### observe

```java
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    // 讲LifecycleOwner和Observer组装起来
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    // 讲Observer加入lifecycle的监听之中
    owner.getLifecycle().addObserver(wrapper);
}
```

```java
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED);
    }

    // 当Lifecycle的state的改变时会回调这个方法
    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);
            return;
        }
        activeStateChanged(shouldBeActive());
    }

    @Override
    boolean isAttachedTo(LifecycleOwner owner) {
        return mOwner == owner;
    }

    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this);
    }
}
```

```java
void activeStateChanged(boolean newActive) {
    if (newActive == mActive) {
        return;
    }
    // immediately set active state, so we'd never dispatch anything to inactive
    // owner
    mActive = newActive;
    boolean wasInactive = LiveData.this.mActiveCount == 0;
    LiveData.this.mActiveCount += mActive ? 1 : -1;
    if (wasInactive && mActive) {
        onActive();
    }
    if (LiveData.this.mActiveCount == 0 && !mActive) {
        onInactive();
    }
    // 当Lifecycle的状态由不可见变为可见, 重新分发value
    if (mActive) {
        dispatchingValue(this);
    }
}
```

### postValue
```java
protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}

private final Runnable mPostValueRunnable = new Runnable() {
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        //noinspection unchecked
        setValue((T) newValue);
    }
};

@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```
postValue方法和setValue最终都会保证是在主线程执行的. 所以LiveData自动实现了线程的跳转.


