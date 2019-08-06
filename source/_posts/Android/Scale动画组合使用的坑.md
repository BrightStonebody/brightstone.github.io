---
title: Scale动画组合使用的坑
date: 2019-08-02 09:43:37
tags:
- Animation
- View
categories:
- Android
- View
---

# Scale动画组合使用的坑

## API
```java
        Animation scale = new ScaleAnimation(0.6f, 1.03f, 0.6f, 1.03f, Animation.RELATIVE_TO_SELF, 1f, Animation.RELATIVE_TO_SELF, 0);
```
参数1: X的初始值
参数2: X的最终值
参数3: Y的初始值
参数4: Y的最终值
参数5: X轴，Animation.RELATIVE_TO_SELF表示根据view大小的百分比进行缩放
参数6: X轴缩放轴点，1f表示以View的最右端为轴点
参数7: Y轴，同参数5
参数8: Y轴，同参数6

## 问题
一个补间动画AnimationSet中有两个ScaleAnimation，先放大后缩小
```java
// start animation
        AnimationSet animationSet = new AnimationSet(false);
        Animation alpha = new AlphaAnimation(0, 100);
        alpha.setDuration(80);
        Animation scale = new ScaleAnimation(0.6f, 1.03f, 0.6f, 1.03f, Animation.RELATIVE_TO_SELF, 1f, Animation.RELATIVE_TO_SELF, 0);
        scale.setDuration(160);
        scale.setInterpolator(PathInterpolatorCompat.create(0.32f, 0.66f, 0.6f, 1f));
        Animation scale2 = new ScaleAnimation(1.03f, 1f, 1.03f, 1f, Animation.RELATIVE_TO_SELF, 1f, Animation.RELATIVE_TO_SELF, 0);
        scale2.setDuration(70);
        scale2.setStartOffset(160);
        animationSet.addAnimation(scale);
        animationSet.addAnimation(scale2);
        animationSet.addAnimation(alpha);
        tv.startAnimation(animationSet);
```

但是最后的动画效果却非常不流畅，通过加长动画效果，发现动画在放大后缩回的过程中，缩一定程度后直接突变到了最初始的大小

## 原因
scale动画是根据当前的值进行缩放的，所以scale2应该改成这个样子
```java
Animation scale2 = new ScaleAnimation(1.03f, 1f, 1.03f, 1f, Animation.RELATIVE_TO_SELF, 1f, Animation.RELATIVE_TO_SELF, 0);
scale2.setDuration(70);
scale2.setStartOffset(160);
```