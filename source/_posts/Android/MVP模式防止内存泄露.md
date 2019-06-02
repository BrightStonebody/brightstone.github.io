---
title: MVP模式防止内存泄露
date: 2019-04-15 08:54:27
tags:
- MVP
- 内存泄露
categories:
- Android
---

# MVP模式防止内存泄露
### 参考链接
https://jocherch.github.io/mvp/mvp-memory-leak/
https://blog.csdn.net/Xiong_IT/article/details/52610729

### 发生内存泄露的原因
由于Presenter经常性地需要执行一些耗时的操作，例如，我们经常使用的网络请求数据。当 Presenter 持有了 Activity 的强引用，如果在请求结束之前，Activity 被销毁了，那么由于网络请求还没有返回，导致 Presenter 一直持有 Activity 对象的引用，使得该对象无法被系统回收，此时就发生了内存泄露。

**解决方案：通过弱引用和 Activity / Fragment 的生命周期来解决这个问题。**

**Model**
```java
interface BaseMvpModel{
    public void mvpCancleTasks();
        // TODO 终止线程池ThreadPool.shutDown()，AsyncTask.cancle()，或者调用框架的取消任务api
    
}
```
**View**
```java
interface BaseMvpView{
    public void mvpDetachView();
    /*
        例如
        @Override
        public void onDestroy() {
            super.onDestroy();
            mPresenter.mvpDestroy();
            mPresenter = null;
        }
    */
}
```

**Presenter**
```java
interface BaseMvpPresenter{
    public void mvpDestory();
    /*例如:
        public void mvpDestory() {
            view = null;
            if(modle != null) {
                modle.mvpaCncleTasks();
                modle = null;
            }
        }
    */
}
```

这里只是创建了一个BaseInterface用于让mvp的接口去继承，没有做更加详细的封装，仅仅由于提示。主要是不同的模块处理可能会有不同的操作

**注意要使用WeakReference**
并不是在任何情况下Activity的onDestroy都会被调用（其它原因导致Activity对象还在被引用，就不会回调onDestroy方法），一旦这种情况发生，弱引用也能够保证不会造成内存泄露。

