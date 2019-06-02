---
title: Android-Studio解决依赖冲突
date: 2019-04-14 19:40:23
tags:
- AndroidStudio
categories:
- Android
- AndroidStudio
---

# Android-Studio解决依赖冲突

我们做android项目通常会引入很多第三方库， 有时候不同的第三方库会出现依赖冲突，导致添加依赖后就android-studio就报错。
做项目是要添加glide库，直接添加最新版本 4.8，《第一行代码》中介绍使用的版本是3.7。 结果就是4.8的版本一添加依赖就报错，在build.gradle文件中报错buildtool版本冲突。我在网上找到了如下的文章，解决方法是一样的，做一个笔记。**需要说明的 glide4.+ 的版本和 3.+ 的版本，提供的api接口的操作形式发生了变化，而且google上搜到说版本升级之后其实性能没有多大提升**

原文：
<https://blog.csdn.net/victor888886/article/details/73714141>

**以下是网上的文章内容：**
最近刚接手一个项目，里面模块有三四个，引入的第三方包更多了。但是问题来了，新配置的studio一运行就报了错。

    Error:Execution failed for task ':app:processDebugManifest'.
    

> Manifest merger failed : Attribute meta-data#android.support.VERSION@value value=(25.3.1) from \[com.android.support:design:25.3.1\] AndroidManifest.xml:27:9-31  
> is also present at \[com.android.support:support-v4:26.0.0-alpha1\] AndroidManifest.xml:27:9-38 value=(26.0.0-alpha1).  
> Suggestion: add ‘tools:replace=”android:value”’ to element at AndroidManifest.xml:25:5-27:34 to override.  

可以看到，studio已经明确的指出了错误，在清单文件中Android support 库版本冲突了，而且，studio还很“人性”地给出了suggestion：清单文件25行——27行添加：

    tools:replace="android:value"

坑就坑在这里，给出的建议完全误导人了。咳咳，下面看我详（如）细（何）解（装）释（逼）：

#### 问题分析：

    看到com.android.support:design:25.3.1 和
    com.android.support:support-v4:26.0.0-alpha1，
    说明这个Android support库版本冲突了，解决的思想很简单，就是统一使用同一个版本的support库，比如修改掉26.0.0-alpha1的依赖，统一换成25.3.1的版本。
    

### 解决

    既然有了思路，那就动手试一试，全局搜索26.0.0-alpha1，统一替换为 25.3.1
    正常情况下，这个是能解决问题的，但只能解决gradle里面自己引入的依赖版本问题。然而，今天碰到的坑还没完呢，同步代码以后，还是原来的错误信息！
    

### 再次分析：

    问题就出在第三方库的依赖了，好多第三方库默认引用当前最新的support库，现在最新的就是26.0.0-alpha1这个版本。所以，要解决问题，就要从引入的三方库里面入手了！那么问题来了，挖掘机哪家......哦不，怎么知道哪个依赖包有冲突？下面出杀手锏了：
    打开Android studio下面的terminal，输入命令：`gradle -q app:dependencies`,惊喜出现了：没有配置gradle环境变量的同学赶快去配一个吧！..
    **（这里不需要配置gradle环境变量也可以，在terminal中输入命令：./gradlew -q app:dependencies效果是一样的）**
    配过之后可以看到类似一下的输出：
    

+— project :base-util  
| +— com.android.support:recyclerview-v7:25.3.1 (*)  
| +— cn.qqtheme.framework:WheelPicker:1.5.1  
| | +— cn.qqtheme.framework:Common:1.5.1  
| | | +— com.android.support:support-v4:latest.release -> 26.0.0-alpha1 (*)  
| | | — com.android.support:support-annotations:latest.release -> 26.0.0-alpha1  
| | +— com.android.support:support-v4:latest.release -> 26.0.0-alpha1 (*)  
| | — com.android.support:support-annotations:latest.release -> 26.0.0-alpha1  
| +— com.github.CymChad:BaseRecyclerViewAdapterHelper:v1.9.8  
| +— io.reactivex:rxjava:1.1.8  
| +— io.reactivex:rxandroid:1.1.0  
| | — io.reactivex:rxjava:1.1.0 -> 1.1.8  
| +— com.squareup.okhttp3:okhttp:3.2.0 -> 3.4.1  
| | — com.squareup.okio:okio:1.9.0  
| +— com.squareup.retrofit2:retrofit:2.0.2  
| | — com.squareup.okhttp3:okhttp:3.2.0 -> 3.4.1 (*)  
| +— com.squareup.retrofit2:adapter-rxjava:2.0.2  
| | +— com.squareup.retrofit2:retrofit:2.0.2 (*)  
| | — io.reactivex:rxjava:1.1.1 -> 1.1.8  
| +— com.squareup.retrofit2:converter-gson:2.0.2  
| | +— com.squareup.retrofit2:retrofit:2.0.2 (*)  
| | — com.google.code.gson:gson:2.6.1  
| +— com.squareup.okhttp3:logging-interceptor:3.4.1  
| | — com.squareup.okhttp3:okhttp:3.4.1 (*)  
| +— com.github.zhaokaiqiang.klog:library:1.5.0  
| | — com.android.support:support-annotations:23.4.0 -> 26.0.0-alpha1  
| +— com.squareup.retrofit2:converter-simplexml:2.0.2  
| | +— com.squareup.retrofit2:retrofit:2.0.2 (*)  
| | — org.simpleframework:simple-xml:2.7.1  
| +— com.github.bumptech.glide:glide:3.7.0  
| +— project :base-res (*)  
| — com.jakewharton:butterknife:7.0.1

很明显cn.qqtheme.framework:WheelPicker这个包默认引用了最新的support库！

### 最终解决：

找到依赖的库，修改为下面的方式引入：

    compile ('cn.qqtheme.framework:WheelPicker:1.5.1'){
            exclude group:'com.android.support'
        }

