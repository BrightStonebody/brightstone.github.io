---
title: RecyclerView缓存机制总结
date: 2019-08-15 00:31:17
tags:
- Android
- View
categories:
- Android
- View
---

# RecyclerView缓存机制总结

## 基本概念
**scrapped:** 
A "scrapped" view is a view that is still attached to its parent RecyclerView but that has been marked for removal or reuse

## RecyclerView中涉及到缓存的集合

* mAttachedScrap
    * 显示在屏幕中，未与RecyclerView分离但被标记移除的Holder。 实际上是从屏幕上分离出来的ViewHolder，但是又即将添加到屏幕上去的ViewHolder。
* mChangedScrap
    * 显示在屏幕中，数据已经发生改变的Holder。notifxxx方法时产生
* mCachedViews 
    * 在屏幕外的Holder。缓存，幕刃大小为2。
* mRecyclerPool
    * 在屏幕外的Holder。当mCachedViews满时，存储至此。按照ViewType进行分类存储。默认大小为5。从中取出的Holder需要调用onBindViewHolder方法

mAttachedScrap、mChangedScrap、mCachedViews中取出的Holder是直接可用的，不需要调用onCreatedViewHolder和onBindViewHolder方法。
mRecyclerPool中取出的Holder是无效的，需要调用onBindViewHolder方法
**很奇怪，按照网上mChangedScrap的解释，其中的Holder明显是需要调用onBindViewHolder方法的。但是网上的解释同时又都说不需要调用。。**

## RecyclerView获取Holder的顺序(sdk 28)

1. getChangedScrapViewForPosition
2. getScrapOrHiddenOrCachedHolderForPosition
3. getScrapOrCachedViewForId
4. getChildViewHolder
5. mViewCacheExtension.getViewForPositionAndType
6. getRecycledViewPool().getRecycledView
7. mAdapter.createViewHolder

## 四级缓存
1. mAttachedScrap  mChangedScrap
2. mCacheView
3. mViewCacheExtension
4. mRecyclerPool

## ListView的缓存机制

### 缓存的集合
* mActiveViews 
    * 屏幕内的view，可直接重用
* mScrapViews
    * 屏幕外的view，需要调用bind

### 与RecyclerView的不同
1. 缓存不同： RecyclerView缓存的是ViewHolder，避免了每次的findViewByid，ListView缓存的是View。
2. RecyclerView中mCacheViews(屏幕外)获取缓存时，是通过匹配pos获取目标位置的缓存，这样做的好处是，当数据源数据不变的情况下，无须重新bindView。 
离屏缓存，ListView从mScrapViews根据pos获取相应的缓存，但是并没有直接使用，而是重新getView（即必定会重新bindView）
3. RecyclerView可以实现局部刷新， ListView不行


## 参考：
[RecyclerView源码分析缓存机制](https://blog.jiahuan.me/2018/07/27/RecyclerView%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%BC%93%E5%AD%98%E6%9C%BA%E5%88%B6/)

[RecyclerView的缓存机制](https://www.jianshu.com/p/efe81969f69d)

[Android ListView 与 RecyclerView 对比浅析--缓存机制](https://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=2653578065&idx=2&sn=25e64a8bb7b5934cf0ce2e49549a80d6&chksm=84b3b156b3c43840061c28869671da915a25cf3be54891f040a3532e1bb17f9d32e244b79e3f&scene=21#wechat_redirect)
