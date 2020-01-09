---
title: 外部View随着RecyclerView的滚动而显示或隐藏
date: 2020-01-09 23:01:39
tags:
- RecyclerView
categories:
- Android
---


外部的View上滑显示, 下滑隐藏
```java
mOutsideOnScrollListener = new OnScrollListener() {
    boolean controlsVisible = false;
    int scrolledDistance = 0;

    @Override
    public void onScrollStateChanged(int newState) {
        if (newState == RecyclerView.SCROLL_STATE_IDLE) {
            scrolledDistance = 0;
        }
    }

    @Override
    public void onScrolled(int dx, int dy) {
        if ((controlsVisible && dy < 0) || (!controlsVisible && dy > 0){
            scrolledDistance += dy;
        }
        if (scrolledDistance < thresholdToShow && controlsVisible) {
            //UP
            scrolledDistance = 0;
            controlsVisible = false;
            view.onShow();
        } else if (scrolledDistance >= thresholdToHide && !controlsVisible) {
            //DOWN
            scrolledDistance = 0;
            controlsVisible = true;
            view.onPause();
        }
    }
};
```

* 我们计算总的滚动距离（每一次滚动的总和）。但是，我们只关心View隐藏时的向上滑动或者View显示时的向下滑动，因为这些是我们所关心的情况。
```java
if ((controlsVisible && dy < 0) || (!controlsVisible && dy > 0)) {
    scrolledDistance += dy;
}
```

* 如果当滚动值超过某个阈值（你可以设置阈值，值越大，需要滚动滚动更多的距离，才能看到显示/隐藏View的效果）。我们根据滚动的方向来显示/隐藏View（DY＞0意味着我们向下滚动，Dy＜0意味着我们向上滚动）。

```java
if (scrolledDistance < thresholdToShow && controlsVisible) {
    //UP
    scrolledDistance = 0;
    controlsVisible = false;
    view.onResume();
} else if (scrolledDistance >= thresholdToHide && !controlsVisible) {
    //DOWN
    scrolledDistance = 0;
    controlsVisible = true;
    view.onPause();
}
```