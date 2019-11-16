---
title: OkHttp3源码-RetryAndInterceptor和BridgeInterceptor
date: 2019-11-14 16:54:17
tags:
  - okhttp3
  - 源码解析
categories:
  - Android
---

# OkHttp3 源码-RetryAndInterceptor 和 BridgeInterceptor

## RetryAndInterceptor

**主要功能:**
失败重连, 重定向

```java
@Override 
public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;

    int followUpCount = 0;
    // priorResponse表示在重定向时, 上一次request的response
    Response priorResponse = null;
    // while死循环, 在请求失败或者重定向之后重新发起请求
    while (true) {
        if (canceled) {
            streamAllocation.release();
            throw new IOException("Canceled");
        }

        Response response;
        boolean releaseConnection = true;
        try {
            // 进入责任链的下一个结点
            response = realChain.proceed(request, streamAllocation, null, null);
            releaseConnection = false;
        } catch (RouteException e) {
            // The attempt to connect via a route failed. The request will not have been sent.
            // recover方法判断这个request是否可以失败重连
            // 注意: client的Builder可以配置拒绝失败重连, 这样的话, recover方法会直接返回false
            if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
                throw e.getFirstConnectException();
            }
            releaseConnection = false;
            // 进入下一个while迭代, 开始失败重连
            continue;
        } catch (IOException e) {
            // An attempt to communicate with a server failed. The request may have been sent.
            boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
            if (!recover(e, streamAllocation, requestSendStarted, request))
                throw e;
            releaseConnection = false;
            continue;
        } finally {
            // We're throwing an unchecked exception. Release any resources.
            if (releaseConnection) {
                streamAllocation.streamFailed(null);
                streamAllocation.release();
            }
        }

        // Attach the prior response if it exists. Such responses never have a body.
        // 如果priorResponse不为null, 将其加入到当前response中
        if (priorResponse != null) {
            response = response.newBuilder()
                .priorResponse(priorResponse.newBuilder()
                        .body(null)
                        .build())
                .build();
        }

        // followUp意思是重定向
        Request followUp;
        try {
            // followUpRequest方法: 重定向时根据response构建新的request
            followUp = followUpRequest(response, streamAllocation.route());
        } catch (IOException e) {
            streamAllocation.release();
            throw e;
        }

        // followUp为空, 表示没有重定向了, 当前response为最终结果, return
        if (followUp == null) {
            streamAllocation.release();
            return response;
        }

        // 有重定向, 关闭响应流
        closeQuietly(response.body());

        // 重定向次数+1, 当超过MAX_FOLLOW_UPS时, 抛出异常
        if (++followUpCount > MAX_FOLLOW_UPS) {
            streamAllocation.release();
            throw new ProtocolException("Too many follow-up requests: " + followUpCount);
        }

        // 判断是否是不可重定向的类型
        if (followUp.body() instanceof UnrepeatableRequestBody) {
            streamAllocation.release();
            throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
        }

        if (!sameConnection(response, followUp.url())) {
            streamAllocation.release();
            streamAllocation = new StreamAllocation(client.connectionPool(),
                createAddress(followUp.url()), call, eventListener, callStackTrace);
            this.streamAllocation = streamAllocation;
        } else if (streamAllocation.codec() != null) {
            throw new IllegalStateException("Closing the body of " + response
                + " didn't close its backing stream. Bad interceptor?");
        }

        // 更新request 和 priorResponse
        request = followUp;
        priorResponse = response;
    }
}
```

## RetryAndInterceptor伪代码
```java
@Override 
public Response intercept(Chain chain) throws IOException {
    Response response;
    Request request = chain.request;
    StreamAllocation streamAllocation = new StreamAllocation();

    int followUpCount = 0;
    while(true){
        try{
            response = realChain.proceed(request, streamAllocation, null, null);
        }catch(OkhttpException e){
            if (!recover())
                throw e;
            continue;
        }catch(OtherException e){
            throw e;
        }

        Request followUp;
        try {
            followUp = followUpRequest(response, streamAllocation.route());
        } catch (IOException e) {
            throw e;
        }

        if(followUp == null)
            return response;
        
        if (++followUpCount > MAX_FOLLOW_UPS) {
            throw new Exception("Too many follow-up requests: " + followUpCount);
        }

        request = followUp;
    }
}
```

## BridgeInterceptor

这个拦截器比较简单, 提一下源代码上的注释吧

```java
/**
* Bridges from application code to network code. First it builds a network request from a user
* request. Then it proceeds to call the network. Finally it builds a user response from the network
* response.
*/

/**
* 从应用程序代码到网络代码的桥梁。首先，它建立用户的网络请求
* 请求，然后继续呼叫网络，最后建立来自网络的用户响应回应。
*/
```
