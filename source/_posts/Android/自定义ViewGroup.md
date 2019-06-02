---
title: 自定义ViewGroup
date: 2019-04-13 13:12:35
tags:
- view
categories:
- Android
- view
---

# 自定义ViewGroup

### 参考文章
[自定义LayoutParams](https://blog.csdn.net/xmxkf/article/details/51500304#3-%E6%94%AF%E6%8C%81layoutmargin%E5%B1%9E%E6%80%A7)
[关于onMeasure过程的理解](https://blog.csdn.net/xmxkf/article/details/51490283)
[Measure测量流程全解析（简洁）](https://juejin.im/post/5ad37c476fb9a028bc2e32af)
### 下面是继承自ViewGroup的FlowLayout标签流式布局
```java
package com.example.test;

import android.content.Context;
import android.util.AttributeSet;
import android.util.Log;
import android.view.View;
import android.view.ViewGroup;

public class FlowLayout extends ViewGroup {
    public FlowLayout(Context context) {
        super(context);
    }

    public FlowLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public FlowLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
        int specWidth = MeasureSpec.getSize(widthMeasureSpec);
        int specHeight = MeasureSpec.getSize(heightMeasureSpec);
        int specWidthMode = MeasureSpec.getMode(widthMeasureSpec);
        int specHeightMode = MeasureSpec.getMode(heightMeasureSpec);

        int count = getChildCount();

        //计算child的大小
        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            //measureChildWithMargins方法在计算时已经考虑到了padding, 所以这里widthUsed参数设置为0
            //这里为了支持margin，所以用measureChildWithMargins方法
            measureChildWithMargins(child,
                    widthMeasureSpec, 0,
                    heightMeasureSpec, 0);
        }
        //计算ViewGroup自身的大小
        //这里只要处理height的wrap_content情况就可以了
        int height = specHeight;
        int width = specWidth;
        if (specHeightMode == MeasureSpec.AT_MOST) {
            height = getPaddingBottom() + getPaddingTop();
            int used_width = 0;
            int line_max_height = 0;
            for (int i = 0; i < count; i++) {
                View child = getChildAt(i);
                MarginLayoutParams mlp = (MarginLayoutParams) child.getLayoutParams();
                int child_width = child.getMeasuredWidth() + mlp.leftMargin + mlp.rightMargin;
                int child_height = child.getMeasuredHeight() + mlp.topMargin + mlp.bottomMargin;
                //在这一行可以容纳
                if (used_width + child_width <= width - getPaddingStart() - getPaddingEnd()) {
                    line_max_height = Math.max(line_max_height, child_height);
                    used_width += child_width;
                } else {
                    //切换到下一行
                    height += line_max_height;
                    used_width = child_width;
                    line_max_height = child_height;
                }
            }
            //加上最后一行的最大height
            height += line_max_height;
        }
        setMeasuredDimension(width, height);
    }

    private final String TAG = "test_log";

    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int width = getMeasuredWidth() - getPaddingStart() - getPaddingEnd();
        int used_width = 0;
        int used_height = 0;

        int count = getChildCount();
        int last_line_max_height = 0;
        for (int i = 0; i < count; i++) {
            View child_view = getChildAt(i);
            MarginLayoutParams mlp = (MarginLayoutParams) child_view.getLayoutParams();
            //用于计算是否越界时需要包含margin
            int child_width = child_view.getMeasuredWidth() + mlp.leftMargin + mlp.rightMargin;
            int child_height = child_view.getMeasuredHeight() + mlp.topMargin + mlp.bottomMargin;
            //layout时的位置，必须考虑padding
            int layout_l, layout_t, layout_r, layout_b;
            if (used_width + child_width <= width) {
                layout_l = getPaddingStart() + used_width + mlp.leftMargin;
                layout_t = getPaddingTop() + used_height + mlp.topMargin;
                layout_r = layout_l + child_view.getMeasuredWidth();
                layout_b = layout_t + child_view.getMeasuredHeight();

                used_width += child_width;
                //记录该行height的最大值
                last_line_max_height = Math.max(last_line_max_height, child_height);
            } else {
                layout_l = getPaddingStart() + mlp.leftMargin;
                layout_t = getPaddingTop() + used_height + last_line_max_height + mlp.topMargin;
                layout_r = layout_l + child_view.getMeasuredWidth();
                layout_b = layout_t + child_view.getMeasuredHeight();

                used_height += last_line_max_height;
                used_width = child_width;

                last_line_max_height = child_height;
            }
            child_view.layout(layout_l, layout_t, layout_r, layout_b);
        }
    }

    @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new MarginLayoutParams(getContext(), attrs);
    }

    @Override
    protected LayoutParams generateLayoutParams(LayoutParams p) {
        return new MarginLayoutParams(p);
    }

    @Override
    protected LayoutParams generateDefaultLayoutParams() {
        return new MarginLayoutParams(super.generateDefaultLayoutParams());
    }
}

```

### 关于MeasureSpec

父控件告诉子控件可获得的空间以及关于这个空间的约束条件

#### SpecMode

* EXACTLY
  * 设置了精确的宽高。如width、height设置了具体值或者设置为 match_parent，都属于这种模式
* AT_MOST
  * width、height设置为wrap_content则属于这种模式。表示父容器只是限制了子view的最大值
* UNSPECIFIED
  * 这种模式一般用于系统， 父容器不对View有任何限制。 一般很少用

### 关于view的Measure过程
我们知道，整个绘制流程是从ViewRootImpl类中performTraversals()开始的，这里面分别执行performMeasure、performLayout、performDraw来完成整个绘制的三大流程。而三大流程都是至顶向下，今天这里只说measure的过程。
    这里以DecorView(根View)面放着一个ViewGroup(ViewGroupA)ViewGroup里面放着一个View(ViewB)为例来说明整个测量的流程：
* **ViewRootImpl.performTraversals()->performMeasure():**
> 这里面会调getRootMeasureSpec（）根据手机屏幕的宽高和DecorView的LayoutParams生成DecorView的MeasureSpec,然后调用DecorView的measure()开始DecorView的测量
* **DecorView.measure()->onMeasure():**
> DecorView继承自FrameLayout，所以会走到FrameLayout的onMeasure(),onMeasure()里调measureChild()来根据上面说的规则为ViewGroupA生成MeasureSpec，并通过ViewGroupA.measure（）开始ViewGroupA的测量
* **ViewGroupA.measure()->onMeasure():**
>这是我们自定义的一个ViewGroup(继承自ViewGroup)
假如我们没有重写onMeasure()的话，则默认调的是View.onMeasure()，则不会发起对子View的measure,它里面的子View也就不会被测量(0),而这个ViewGroup如果没有设置具体宽高的话，（wrap_content）则ViewGroup展示的就是父容器的宽高（根据上面说的MeasureSpec生成规则)。
    所以如果我们继承自ViewGroup来自定义一个ViewGroup的话，是肯定要重写onMeasure()的，**里面要调用measureChild()来为子View生成MeasureSpec并调child.measure()开始对child的测量(getChildMeasureSpec()方法)，这样子View才能被测量显示。而如果我们要使设置的wrap_content生效，还要根据子View测量结果进行计算从而得到自己的宽高，最后通过调setMeasuredDimension(int measuredWidth, int measuredHeight)来设置自己的宽高，从而达到wrap_content的效果。**
* **ViewB.measure()->onMeasure():**
> View的测量相对于ViewGroup要简单点，因为不用去Measure child,但是一样的，如果要使wrap_conten生效需自己重写onMeasure()计算。

### 测量子view时MeasureSpec的生成规则
1. **当子View的宽高设置的是具体数值时**
> 显然我们可以直接拿到子View的宽高，则子View宽高就确定了，不用再去考虑父容器的SpecMode了,**此时子View的SpecMode为EXACTLY，SpecSize就是设置的宽高。**
2. **当子View的宽高设置的是match_parent**
> 则**不管父容器的SpecMode是什么模式，子View的SpecSize就等于父容器的宽高，而子View的SpecMode随父容器的SpecMode。**（这里没有考虑UNSPECIFIED模式，如果父容器是UNSPECIFIED模式，则子View SpecSize为0，SpecMode为UNSPECIFIED）
3. **当子View的宽高设置的是wrap_content,**
> 因为这种情况父容器实在不知道子View应该多宽多高，**所以子View的SpecSize给的是父容器的宽高，也就是说只是给子View限制了一个最大宽高，而子View的SpecMode是AT_MOST模式。**（这里没有考虑UNSPECIFIED模式，如果父容器是UNSPECIFIED模式，则子View SpecSize为0，SpecMode为UNSPECIFIED）。
    
* 通过上面的解析我们可以知道，当你给一个View/ViewGroup设置宽高为具体数值或者match_parent，它都能正确的显示，但是如果你设置的是wrap_content，则默认显示出来是其父容器的大小，如果你想要它正常的显示为wrap_content，则你就要自己重写onMeasure()来自己计算它的宽高度并设置。**所以我们平常自定义View/ViewGroup的时候之所以要重写onMeasure()，就是为了能让wrap_content达到效果。**

### 关于LayoutParams
在上面的FlowLayout代码中，为了支持margin属性，使用了MarginLayoutParams。这个MarginLayoutParams继承自LayoutParams。**在使用中必须重写所有的generateLayoutParams()方法**
尝试了一下，RelativeLayoutParams等都是继承自MarginLayoutParams
