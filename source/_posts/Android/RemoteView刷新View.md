---
title: RemoteView刷新View
date: 2019-09-15 16:09:50
tags:
- Android
- View
categories:
- Android
- View
---

# RemoteView刷新View

## 通知栏
在Notification通知栏的相关开发中，RemoteView刷新view只要重新发送一遍Notification，保证message_id是相同的就可以了
但是由于在实际的需求中出现了异步刷新的场景。即，当要刷新时，先直接刷新textview，调起线程刷新获取image资源，在回调里调用RemoteView的setxxx方法，刷新ImageView。 在这种情况下，出现了刷新不及时的情况，图片的刷新有时成功有时不成功。

在这种情况下，更改了刷新逻辑，当数据更新时直接刷新textview，并发送notification。在回调里调用RemoteView的相关方法，刷新image，并在回调里再次发送notification，以确保通知的view是最新的。
其中，虽然是发送了两次notification，但是并不用每次都new一个新的RemoteView出来， 即RemoteView可以重复利用，setContent的时候设置回去就好了

## 关于Intent
由于通知栏的按钮的响应操作都很复杂，所以采取的方法时每次都发送一个广播，添加指定的action和extra，在广播接收器中进行指定的操作。

在操作的过程出出现了一个问题。通知栏有多个按钮，但是一开始，不同按钮的intent的action和extra的key都是一样的，extra的value不同，然后在广播接收器内判断extra来区分不同的按钮。
这样做并没达到想要的效果。所有的按钮接收到的intent中的extra值 都是得到的代码中添加的最后一个value值。

也就是说，**Intent中，设置了相同的action值，相同的extra的key，之后设置的value值是会覆盖之前的value的。即使是不同的Intent对象**

