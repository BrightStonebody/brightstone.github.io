---
title: RecyclerView全局刷新的问题
date: 2019-09-15 15:33:22
tags:
- Android
- View
- RecyclerView
categories:
- Android
- View
---

## 1. RecyclerView全局刷新避免出现闪烁

闪烁是因为RecyclerView有默认的动画, 把默认动画去掉就可以了

```java
mTabListRecycler.getItemAnimator().setChangeDuration(0);
((SimpleItemAnimator) mTabListRecycler.getItemAnimator()).setSupportsChangeAnimations(false);
```

## 2. RecyclerView全局刷新避免出现滚动

```java
mTabListRecycler.setHasFixedSize(true);
```
setHasFixedSize(true)的意思是表示, RecyclerView的大小是不收Adapter的内容影响的, 这样RecyclerView绘制的时候会有很多优化. 同时也能够解决刷新的滚动问题

## 3. RecyclerView全局刷新View避免卡顿

### 3.1 onBindViewHolder(VH holder, int position， List<Object> payloads)

这个方法默认是调用普通的onBindViewHolder方法。
我们可以重写这个方法，在payloads参数为空时，执行默认的onBindViewHolder，在其不为空时，刷新每个item特定的view
在调用notifyxxxx类似方法的时候，调用包含payload参数的重载方法

#### 问题：
实践了上面的方法，发现卡顿的问题得到了缓解，但是并没有完全解决

### 3.2 创建并操作自己的ViewHolder缓存

#### 还是有卡顿的原因：

在调用notifyxxxx相关方法时，即使添加了payload参数，依然会有很多item刷新时走onCreateViewHolder方法，在这里进行inflate操作会非常耗时。所以要想办法跳过onCreateViewHolder方法

##### 创建自己的viewholder缓存

1. 创建缓存集合
```
private List<ViewHolder> holderList = new ArrayList<>();
```

2. 添加到集合
```
holder.itemView.setTag(itemData));
if (!holderList.contains(holder)) {
    holderList.add(holder);
}
```
因为在添加之前做了是否包含的判断，所以集合中按道理之后包含显示的ViewHolder以及缓存的ViewHolder，集合的size并不会无限制的增长。

3. 刷新
```
public void update(){
    for (ViewHolder holder : holderList) { 
        ItemData itemData = (ItemData) holder.itemView.getTag();
        // update view
    }
}
```
卡顿问题解决