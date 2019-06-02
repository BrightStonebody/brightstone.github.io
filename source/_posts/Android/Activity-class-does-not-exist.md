---
title: 'Activity class {} does not exist'
date: 2019-04-16 19:23:35
tags:
- AndroidStudio
categories:
- Android
- AndroidStudio
---

# Activity class {} does not exist

有时候真机调试，在手机上卸载调试的APP就会出现上述的问题。

**解决办法：**
使用adb命令
```shell
adb uninstall [包名]
```

估计可能是调试的APP没有彻底卸载造成的。