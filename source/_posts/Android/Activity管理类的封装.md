---
title: Activity管理类的封装
date: 2019-04-14 19:41:50
tags:
- Activity
categories:
- Android
---

### 使用场景
有的时候我们需要在一个activity1中销毁另一个制定的activity2。或者是在程序的某个位置销毁所有的activity，达到退出整个app的目的

我在网上找到一个做法，考虑的很严谨，在stack里存的不是activity，而是WeakReference。作者的愿意是担心出现有activity被finish之后ActivityMannager却忘了通知的情况，然后就是内存泄露 。。
**这里主要是说在系统内存不足杀死Activity时onDestory方法不会被调用**

```java
public class FinishActivityManager extends BaseActivity {
    private FinishActivityManager() {
    }
    private static FinishActivityManager sManager;
    private Stack<WeakReference<Activity>> mActivityStack;
    public static FinishActivityManager getManager() {
        if (sManager == null) {
            synchronized (FinishActivityManager.class) {
                if (sManager == null) {
                    sManager = new FinishActivityManager();
                }
            }
        }
        return sManager;
    }
    /**
     * 添加Activity到栈
     * @param activity
     */
    public void addActivity(Activity activity) {
        if (mActivityStack == null) {
            mActivityStack = new Stack<>();
        }
        mActivityStack.add(new WeakReference<>(activity));
    }
    /**
     * 检查弱引用是否释放，若释放，则从栈中清理掉该元素
     */
    public void checkWeakReference() {
        if (mActivityStack != null) {
            // 使用迭代器进行安全删除
            for (Iterator<WeakReference<Activity>> it = mActivityStack.iterator(); it.hasNext(); ) {
                WeakReference<Activity> activityReference = it.next();
                Activity temp = activityReference.get();
                if (temp == null) {
                    it.remove();
                }
            }
        }
    }
    /**
     * 获取当前Activity（栈中最后一个压入的）
     * @return
     */
    public Activity currentActivity() {
        checkWeakReference();
        if (mActivityStack != null && !mActivityStack.isEmpty()) {
            return mActivityStack.lastElement().get();
        }
        return null;
    }
    /**
     * 关闭当前Activity（栈中最后一个压入的）
     */
    public void finishActivity() {
        Activity activity = currentActivity();
        if (activity != null) {
            finishActivity(activity);
        }
    }
    /**
     * 关闭指定的Activity
     * @param activity
     */
    public void finishActivity(Activity activity) {
        if (activity != null && mActivityStack != null) {
            // 使用迭代器进行安全删除
            for (Iterator<WeakReference<Activity>> it = mActivityStack.iterator(); it.hasNext(); ) {
                WeakReference<Activity> activityReference = it.next();
                Activity temp = activityReference.get();
                // 清理掉已经释放的activity
                if (temp == null) {
                    it.remove();
                    continue;
                }
                if (temp == activity) {
                    it.remove();
                }
            }
            activity.finish();
        }
    }
    /**
     * 关闭指定类名的所有Activity
     * @param cls
     */
    public void finishActivity(Class<?> cls) {
        if (mActivityStack != null) {
            // 使用迭代器进行安全删除
            for (Iterator<WeakReference<Activity>> it = mActivityStack.iterator(); it.hasNext(); ) {
                WeakReference<Activity> activityReference = it.next();
                Activity activity = activityReference.get();
                // 清理掉已经释放的activity
                if (activity == null) {
                    it.remove();
                    continue;
                }
                if (activity.getClass().equals(cls)) {
                    it.remove();
                    activity.finish();
                }
            }
        }
    }
    /**
     * 结束所有Activity
     */
    public void finishAllActivity() {
        if (mActivityStack != null) {
            for (WeakReference<Activity> activityReference : mActivityStack) {
                Activity activity = activityReference.get();
                if (activity != null) {
                    activity.finish();
                }
            }
            mActivityStack.clear();
        }
    }
    /**
     * 退出应用程序
     */
    public void exitApp() {
        try {
            finishAllActivity();
            // 退出JVM,释放所占内存资源,0表示正常退出
            System.exit(0);
            // 从系统中kill掉应用程序
            android.os.Process.killProcess(android.os.Process.myPid());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```


然后是一个BaseActivity类， 重写onCreate和onDestory方法

```java
package com.example.chenlei.test;

import android.os.Bundle;
import android.support.annotation.Nullable;
import android.support.v7.app.AppCompatActivity;
import android.util.Log;

public class BaseActivity extends AppCompatActivity {


    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        MyActivityManager.addActivity(this);
        Log.i("有activity新添加", "onCreate: ");
    }



    @Override
    protected void onDestroy() {
        Log.i("有activity被销毁", "onDestroy: "+ MyActivityManager.getSize());
        MyActivityManager.finishActivity(this);
        super.onDestroy();
    }
}

```

接下来所有的activity类都继承自BaseActivity， 然后就可以在制定的activity类中对ActivityManager类进行操作


网上原文：
<http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2017/0629/8124.html>
