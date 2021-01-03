---
title: Fragment可见性总结
date: 2020-10-04 18:28:37
tags:
- Fragment
---

## onResume()/onPause()

**不要用这两个方法为可见性依据做操作。** 触发onResume()/onPause()的场景有很多

* **Activity执行onResume()/onPause()**，其所有的 Fragment 都会的 onResume()/onPause() 方法，即使Fragment是状态是隐藏的
* **在当前Activity中添加一个Fragment时**，被添加的Fragment都会调用onResume，这包括通过add和replace的方式添加；在Fragment销毁的情况下都会触发onPause，如果是add到回退栈的情况，back键就会触发onPause，如果是replace的情况，则被替换掉的Fragment会触发onPause。
* **ViewPager 预加载：** 被预加载的Fragment也会执行 onResume()/onPause() 

## isVisible()

注释说是可以用来判断其可见性，**但是如果你是在Fragment的onResume/onPause方法中调用这两个方法判断是不准确的，**onResume的方法中isVisible()有可能为false, 而onPause的方法中isVisible()也有可能为true。

## onHiddenChanged()

就是对Fragment进行show和hide的时候会回调onHiddenChanged方法。除此之外的操作，如add/replace/remove都不会触发这个方法。

## setUserVisibleHint()/getUserVisibleHint()

getUserVisibleHint() 默认返回true， 只有和 ViewPager 配合使用有效，ViewPager 选中的 fragment 调用 getUserVisibleHint() 会返回true。 **在ViewPager2无效**

## 判断fragment是否可见的保险操作？？ maybe

```java
final boolean parentVisible = parentFragment == null ? true : parentFragment.isVisible()=
final boolean visible = isVisible() && parentVisible && getUserVisibleHint()
```

