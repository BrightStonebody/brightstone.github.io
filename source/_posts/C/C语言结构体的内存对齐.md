---
title: C语言结构体的内存对齐
date: 2019-04-13 13:11:14
categories: 
- C/C++
tags:
---

# C语言结构体的内存对齐

## 内存对齐原则
* 数据成员对齐规则：结构（struct或联合union）的数据成员，第一个数据成员放在offset为0的地方，以后每个数据成员存储的起始位置: min(#pragma pack()指定的数,这个数据%成员的自身长度)的倍数

* 结构体作为成员：如果一个结构里有某些结构体成员，则结构体成员要从min(#pragram pack() , 内部长度最长的数据成员)的整数倍地址开始存储。（struct a里存有struct b，b里有char，int，double等元素，那b应该从min(#pragram pack(), 8)的整数倍开始存储。）

* 结构体的总大小，也就是sizeof的结果，必须是 min(#pragram pack() , 长度最长的数据成员) 的整数倍

## pragram pack(4)
设置内存对齐的字节数， 默认为系统字长，64位系统为8字节，32位系统为4字节

## 例子：
```c++
# pragram pack(8)

struct S3
{
    double d;
    char c;
    int i;
};
struct S4
{
    char c1;
    struct S3 s3;
    double d;
};
printf("%d\n", sizeof(struct S4));
```

最后的输出为 32

## 参考：
[[C/C++] 结构体内存对齐用法 - 我自逍遥笑 - 博客园](https://www.cnblogs.com/zwh0214/p/8833314.html)

## C语言联合体union的sizeof

**分配给union的实际大小不仅要满足是对齐大小的整数倍，同时要满足实际大小不能小于最大成员的大小。**

