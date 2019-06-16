---
title: 'scrollBy(),scrollTo()和Scroller'
date: 2019-06-16 20:26:14
tags: 
- View
categories:
- Android
---
# scrollBy(),scrollTo()和Scroller

## 作用
用于对View进行滚动
<br>
注意：
1. **滚动的是View的内容，而不是View本身（对viewd的视窗进行移动）**
比如：TextView滚动的是内部的text而不是整个view
2. **滚动的方向与坐标轴正方向相反**
比如：scrollBy(20,0)
最后显示，view会向左移动
因为是视窗的移动，所以视窗右移，view相对的向左移动(可以这么理解，具体看源码)

## scrollBy() 和 scrollTo()的区别
scrollBy()方法是让View相对于当前的位置滚动某段距离，而scrollTo()方法则是让View相对于初始的位置滚动某段距离。

## Scroller
利用Scroller可以实现有过渡动画的平滑移动，而不是突兀的瞬移
### 使用步骤
Scroller的基本用法其实还是比较简单的，主要可以分为以下几个步骤：

1. 创建Scroller的实例
2. 调用startScroll()方法来初始化滚动数据并刷新界面
3. 重写computeScroll()方法，并在其内部完成平滑滚动的逻辑



### 代码：实现自定义的简单ViewPager
```java
package com.example.work3;

import android.content.Context;
import android.support.v4.view.ViewConfigurationCompat;
import android.util.AttributeSet;
import android.util.Log;
import android.view.MotionEvent;
import android.view.View;
import android.view.ViewConfiguration;
import android.view.ViewGroup;
import android.widget.Scroller;

public class ScrollLayout extends ViewGroup {

    private final String TAG = "test_scroll";
    private Scroller mScroller;
    // 可以判定为拖动的最小滑动距离
    private int mTouchSlop;
    // 落下的屏幕坐标
    private float mXDown;
    // 当前的屏幕坐标
    private float mXMove;
    // 上一次Action_MMOVE的屏幕坐标
    private float mLastMove;
    // 界面可滑动的左边界
    private int mLeftBorder;
    // 界面可滑动的右边界
    private int mRightBorder;

    public ScrollLayout(Context context) {
        super(context);
        init(context);
    }

    public ScrollLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        init(context);
    }

    public ScrollLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init(context);
    }

    private void init(Context context) {
        mScroller = new Scroller(context);
        // 获取系统定义的mTouchSlop值
        mTouchSlop = ViewConfigurationCompat.getScaledPagingTouchSlop(ViewConfiguration.get(context));
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            measureChild(getChildAt(i), widthMeasureSpec, heightMeasureSpec);
        }
    }

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        if (!changed)
            return;
        int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            View childView = getChildAt(i);
            childView.layout(i * childView.getMeasuredWidth(), 0, (i + 1) * childView.getMeasuredWidth(), childView.getMeasuredHeight());
        }
        // 初始化左右边界
        mLeftBorder = getChildAt(0).getLeft();
        mRightBorder = getChildAt(childCount - 1).getRight();
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        switch (ev.getAction()) {
            case MotionEvent.ACTION_DOWN:
                mXDown = ev.getRawX();
                mLastMove = mXDown;
                break;
            case MotionEvent.ACTION_MOVE:
                mXMove = ev.getRawX();
                float diff = Math.abs(mXMove - mXDown);
                mLastMove = mXMove;
                // 手指拖动值大于TouchSlop，认为应该进行滚动，拦截事件
                if (diff > mTouchSlop) {
                    return true;
                }
                break;
            case MotionEvent.ACTION_UP:
                break;
            default:
                break;
        }
        return super.onInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        switch (event.getAction()) {
            case MotionEvent.ACTION_DOWN:
                break;
            case MotionEvent.ACTION_MOVE: {
                mXMove = event.getRawX();
                int scrolledX = (int) (mLastMove - mXMove);
                if (getScrollX() + scrolledX < mLeftBorder) {
                    scrollTo(mLeftBorder, 0);
                    return true;
                } else if (getScrollX() + scrolledX + getWidth() > mRightBorder) {
                    scrollTo(mRightBorder - getWidth(), 0);
                    return true;
                }
                // view随着手指的拖动进行滚动
                scrollBy(scrolledX, 0);
                Log.i(TAG, "onTouchEvent: " + getChildAt(1).getLeft());
                mLastMove = mXMove;
                break;
            }
            case MotionEvent.ACTION_UP: {
                // 当手指抬起时，根据当前的滚动值来判定应该滚动到哪个子控件的界面
                int targetIndex = (getScrollX() + getWidth() / 2) / getWidth();
                int dx = targetIndex * getWidth() - getScrollX();
                // 第二步，调用startScroll()方法来初始化滚动数据并刷新界面
                mScroller.startScroll(getScrollX(), 0, dx, 0);
                // 对view重绘
                invalidate();
                break;
            }
        }
        return super.onTouchEvent(event);
    }

    @Override
    public void computeScroll() {
        // computeScroll方法重写的模版代码， 如果是子View需要调用父布局的scrollTo方法
        if (mScroller.computeScrollOffset()) {
            scrollTo(mScroller.getCurrX(), mScroller.getCurrY());
            // 对view重绘
            invalidate();
        }
    }
}
```

## getScrollX()方法
返回当前滑动View左边界的位置，其实获取的值是画布在窗口左边界时的x坐标。
原点（0，0）是初始化时内容显示的位置。

## 参考
[Android getScrollX()详解
](https://blog.csdn.net/znouy/article/details/51338256)
[Android Scroller完全解析，关于Scroller你所需知道的一切
](https://blog.csdn.net/guolin_blog/article/details/48719871)