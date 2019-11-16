---
title: OkHttp3源码-CallServerInterceptor
date: 2019-11-17 01:10:03
tags:
- okhttp3
- 源码解析
categories:
- Android
---

```java
/** This is the last interceptor in the chain. It makes a network call to the server. */

  @Override 
  public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    realChain.eventListener().requestHeadersStart(realChain.call());
    // HttpCodec相当于流, 将请求header写入流
    httpCodec.writeRequestHeaders(request);
    realChain.eventListener().requestHeadersEnd(realChain.call(), request);

    Response.Builder responseBuilder = null;
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
      // Continue" response before transmitting the request body. If we don't get that, return
      // what we did get (such as a 4xx response) without ever transmitting the request body.
      // 当请求头包含"Expect: 100-continue"字段, 先发送请求头, 若服务端返回code 100, 继续发送请求体. 若返回code 4xx, 关闭流.
      // "Expect: 100-continue": http协议里的, 表示先发送请求头, 若服务端同意, 再发送请求体
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        // 发送请求头
        httpCodec.flushRequest();
        realChain.eventListener().responseHeadersStart(realChain.call());
        // 读取response
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      // 当responseBuilder为null, 则表示没有"Expect: 100-continue"头, 
      // 或者有"Expect: 100-continue", 但是服务端同意继续发送请求体
      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        // 将请求体写入流
        realChain.eventListener().requestBodyStart(realChain.call());
        long contentLength = request.body().contentLength();
        CountingSink requestBodyOut =
            new CountingSink(httpCodec.createRequestBody(request, contentLength));
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);

        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
        realChain.eventListener()
            .requestBodyEnd(realChain.call(), requestBodyOut.successfulCount);
      } else if (!connection.isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
        // from being reused. Otherwise we're still obligated to transmit the request body to
        // leave the connection in a consistent state.

        // 包含"Expect: 100-continue"的请求没有得到请求头允许, 如果是http1.x, 则关闭流(http1.1 不允许多个流同时复用一个http连接)
        streamAllocation.noNewStreams();
      }
    }

    // 结束发送请求
    httpCodec.finishRequest();

    if (responseBuilder == null) {
      realChain.eventListener().responseHeadersStart(realChain.call());
      // 读取response的头
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    // 构建带响应头的响应体
    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (code == 100) {
      // server sent a 100-continue even though we did not request one.
      // try again to read the actual response
      responseBuilder = httpCodec.readResponseHeaders(false);

      response = responseBuilder
              .request(request)
              .handshake(streamAllocation.connection().handshake())
              .sentRequestAtMillis(sentRequestMillis)
              .receivedResponseAtMillis(System.currentTimeMillis())
              .build();

      code = response.code();
    }

    realChain.eventListener()
            .responseHeadersEnd(realChain.call(), response);

    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      // 构建响应体
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }

    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
      streamAllocation.noNewStreams();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
      throw new ProtocolException(
          "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
  }
```