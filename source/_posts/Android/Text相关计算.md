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
![图片](/images/Text相关计算.webp)

Baseline上方的值为正，下方的值为负

## TextSize和TextView大小的转换

相关参数包括：

* 基准点是baseline

* Ascent是baseline之上至字符最高处的距离

* Descent是baseline之下至字符最低处的距离

* 其实是上一行字符的descent到下一行的ascent之间的距离

* Top指的是指的是最高字符到baseline的值，即ascent的最大值

* 同上，bottom指的是最下字符到baseline的值，即descent的最大值

## 参考
[Paint 绘制文字属性](https://www.jianshu.com/p/1728b725b4a6)
[TextView文字实际高度分析](https://neutra.github.io/2016/TextView%E6%96%87%E5%AD%97%E5%AE%9E%E9%99%85%E9%AB%98%E5%BA%A6%E5%88%86%E6%9E%90/)
