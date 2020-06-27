---
title: Android自定义注解
date: 2020-06-27 11:06:02
tags:
- 注解
categories:
- Android
---

因为要自定义一个页面导航工具，需要使用自定义注解

## 1. 创建Java Library

创建两个新的module，**创建时module一定要选择Java Library**

两个module分别是:
* libnavannotation 注解
* libnavcompile 注解处理器



## 2. 定义注解

定义两个注解ActivityDestination, FragmentDestination

```kotlin
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.BINARY)
annotation class ActivityDestination(
    val pageUrl: String,
    val needLogin: Boolean = false,
    val asStarter: Boolean = false
)
```

```kotlin
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.BINARY)
annotation class FragmentDestination(
    val pageUrl: String,
    val needLogin: Boolean = false,
    val asStarter: Boolean = false
)
```

## 3. 定义注解处理器（关键）

### 3.1 配置build.gradle

```groovy
apply plugin: 'java-library'
apply plugin: 'kotlin'
apply plugin: 'kotlin-kapt'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"

    implementation 'com.alibaba:fastjson:1.2.59'

    implementation project(':libnavannotation')
    implementation 'com.google.auto.service:auto-service:1.0-rc6'
//    annotationProcessor 'com.google.auto.service:auto-service:1.0-rc6'
    kapt 'com.google.auto.service:auto-service:1.0-rc6'
}

sourceCompatibility = "8"
targetCompatibility = "8"
```

需要注意以下几点：
* 如果注解处理器是使用kotlin编写的，那么，一定要添加`kotlin-kapt`插件
* implementation 导入annotation注解模块
* 导入auto-service注解处理器依赖，如果是纯Java代码，可以使用`annotationProcessor 'com.google.auto.service:auto-service:1.0-rc6'`， **如果是kotlin代码，必须使用`    kapt 'com.google.auto.service:auto-service:1.0-rc6'`。** 我这里还implementation了auto-service

### 3.2 编写注解处理器

```kotlin

@AutoService(Processor::class)
@SupportedSourceVersion(SourceVersion.RELEASE_8)
@SupportedAnnotationTypes(
    "com.example.libnavannotation.ActivityDestination",
    "com.example.libnavannotation.FragmentDestination"
)
class NavProcessor : AbstractProcessor() {

    override fun init(processingEnv: ProcessingEnvironment) {
        super.init(processingEnv)
        
        ...
        
    }

    override fun process(annotations: Set<TypeElement>, roundEnv: RoundEnvironment): Boolean {

        ...
        
    }
}
```

**注意注解处理器类上面的几个注解**

### 3.3 创建processor configuration file

这里取决于gradle的版本，**高版本必须创建processor配置文件，否则不会执行注解处理器的代码**

* 需要在注解处理器所在module的 main 底下新建一个package，名称为 resources
* 在 `resources` 底下新建文件 `META-INF/services/javax.annotation.processing.Processor`
* 在 `javax.annotation.processing.Processor` 下写入 注解处理器的全名称  eg: `com.example.libnavcompiler.NavProcessor`

## 4. 使用注解

在Android工程module中配置build.gradle

```groovy
apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply plugin: 'kotlin-android-extensions'
apply plugin: 'kotlin-kapt'

...

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"

    ...

    implementation project(":libnavannotation")
//    annotationProcessor project(":libnavcompiler")
    kapt project(":libnavcompiler")
}
```

导入 注解module、注解处理器module
kotlin相关的注意事项和 注解处理器module 中的一样

```kotlin
@FragmentDestination(pageUrl = "main/tabs/home", asStarter = true)
class HomeFragment : Fragment() {
    
    ...

}
```

在build中点击`make project`，即可执行直接处理器中的代码。如果遇到不成功，可以`rebuild`再试一次

## 参考

[教你如何完全解析Kotlin中的注解](https://juejin.im/post/5cb7ebeee51d456e8833394b#heading-0)

[Android 开发之 自定义注解处理器](https://juejin.im/post/5e58dca26fb9a07cb24aa1d6#heading-4)









