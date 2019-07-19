---
title: Text相关计算
date: 2019-07-19 11:09:42
tags:
- View
categories:
- Android
---

# Text相关计算

## Text的相关属性
![图片](/images/Text相关计算)

Baseline上方的值为正，下方的值为负

## TextSize和TextView大小的转换

相关参数包括：

* 所使用字体(fallback的话不影响)的UPM(Units Per EM)

* ascent/descent属性

* top/bottom参数

**禁止includePadding时, TextView实际占据高度是 (ascent - descent) / UPM * textSize**

**开启includePadding时, TextView实际占据高度是 (top - bottom) / UPM * textSize**

## 参考
[Paint 绘制文字属性](https://www.jianshu.com/p/1728b725b4a6)
[TextView文字实际高度分析](https://neutra.github.io/2016/TextView%E6%96%87%E5%AD%97%E5%AE%9E%E9%99%85%E9%AB%98%E5%BA%A6%E5%88%86%E6%9E%90/)
