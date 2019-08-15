---
title: LayoutInflater解析
date: 2019-08-13 20:31:21
tags:

* Android
* View

Categories:

* Android
* View

---

# LayoutInflater解析

## 源码

```
View result = root;
```

``` 
// Temp is the root view that was found in the xml
final View temp = createViewFromTag(root, name, inflaterContext, attrs);

ViewGroup.LayoutParams params = null;
if (root != null) {
    // Create layout params that match root, if supplied
    params = root.generateLayoutParams(attrs);
    if (!attachToRoot) {
        // Set the layout params for temp if we are not
        // attaching. (If we are, we use addView, below)
        temp.setLayoutParams(params);
    }
}
```

``` 
// Inflate all children under temp against its context.

rInflateChildren(parser, temp, attrs, true);

// We are supposed to attach all the views we found (int temp)
// to root. Do that now.
if (root != null && attachToRoot) {
    root.addView(temp, params);
}

// Decide whether to return the root that was passed in or the
// top view found in xml.
if (root == null || !attachToRoot) {
    result = temp;
}
```

```
return result;
```

### 使用方法
```
LayoutInflater.from(context).inflate(int resource, ViewGroup root, boolean attachToRoot) ;
```
* 第一个参数是layout资源文件id
* 第二个参数是父布局。 当第二个参数为null时，将不会为inflate出来的view添加layoutParam
* 第三个参数是是否加入到父布局中。若为空，则，调用root.addView直接加入到父布局中

**需要注意的是，当root!=null，且attachToRoot为true时，Inflater返回的父布局的view，而不是解析出的view**

