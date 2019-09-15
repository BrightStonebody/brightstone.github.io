---
title: startService()和bindService()
date: 2019-07-04 20:37:58
tags: 
- Service
categories:
- Android
- Service
---

# startService()和bindService()的区别

![](/images/startService\ and\ bindService)

### 生命周期上的差别
#### startService()
执行startService时，Service经历onCreate->onStartCommand。当执行stopService时，直接调用onDestroy方法。调用者如果没有stopService，Service会一直在后台运行，下次调用者再起来仍然可以stopService。

多次调用startService，该Service只能被创建一次，即该Service的onCreate方法只会被调用一次。**但是每次调用startService，onStartCommand方法都会被调用。**无论startService调用多少次，stopService只需要调用一次就能够终止Service

#### BindService()
bindService开启服务时，根据生命周期里onBind方法的返回值是否为空，有两种情况。
1. **onBind返回值是null**
调用bindService开启服务，生命周期执行的方法依次是：
onCreate() ==> onBind();
**调用多次bindService，onCreate和onBind也只在第一次会被执行。调用unbindService结束服务，生命周期执行onDestroy方法，并且unbindService方法只能调用一次，多次调用应用会抛出异常。** 使用时也要注意调用unbindService一定要确保服务已经开启，否则应用会抛出异常。
2. **onBind返回值不为null**
这时候调用bindService开启服务，生命周期执行的方法依次是：
onCreate() ==> onBind() ==> onServiceConnected();
可以发现我们自己写的Connection类里的onServiceConnected方法被调用了。**调用多次bindService，onCreate和onBind都只在第一次会被执行，onServiceConnected会执行多次。**
并且我们注意到onServiceConnected方法的第二个参数也是IBinder类型的，不难猜测onBind()方法返回的对象被传递到了这里。打印一下两个对象的地址可以证明猜测是正确的。
也就是说我们可以在onServiceConnected方法里拿到了Service服务的内部类Binder的对象，通过这个内部类对象，只要强转一下，我们可以调用这个内部类的非私有成员对象和方法。
调用unbindService结束服务和上面相同，unbindService只能调用一次，onDestroy也只执行一次，多次调用会抛出异常。
<br>
**总结**
第一次执行bindService时，onCreate和onBind方法会被调用，但是多次执行bindService时，onCreate和onBind方法并不会被多次调用，即并不会多次创建服务和绑定服务。

### 既使用startService又使用bindService的情况
如果一个Service又被启动又被绑定，则该Service会一直在后台运行。首先不管如何调用，onCreate始终只会调用一次。对应startService调用多少次，Service的onStart方法便会调用多少次。Service的终止，需要unbindService和stopService同时调用才行。不管startService与bindService的调用顺序，如果先调用unbindService，此时服务不会自动终止，再调用stopService之后，服务才会终止；如果先调用stopService，此时服务也不会终止，而再调用unbindService或者之前调用bindService的Context不存在了（如Activity被finish的时候）之后，服务才会自动停止。

参考链接:
[https://my.oschina.net/tingzi/blog/376545]()
[https://www.jianshu.com/p/d870f99b675c]()
